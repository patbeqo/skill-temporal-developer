<!-- Reference for determinism-review skill. Scan diffs against these patterns to determine if a change affects the workflow command sequence. -->

## Command-Producing Operations

Every operation in this table produces a Command that is recorded in the workflow event history. Adding, removing, reordering, or conditionally skipping any of these changes the command sequence and risks NondeterminismError on replay.

| Operation | Command | What Gets Recorded |
|-----------|---------|-------------------|
| Execute activity | ScheduleActivityTask | Activity type, input, options |
| Execute local activity | ScheduleActivityTask (local variant) | Activity type, input, options |
| Start child workflow | StartChildWorkflowExecution | Workflow type, input, options |
| Signal external workflow | SignalExternalWorkflowExecution | Target workflow, signal name, input |
| Cancel external workflow | RequestCancelExternalWorkflowExecution | Target workflow |
| Sleep / timer | StartTimer | Duration |
| SideEffect | RecordMarker | Recorded value |
| MutableSideEffect | RecordMarker | Recorded value + ID |
| UpsertSearchAttributes | UpsertWorkflowSearchAttributes | Attribute key/values |
| UpsertMemo | ModifyWorkflowProperties | Memo key/values |
| ContinueAsNew | ContinueAsNewWorkflowExecution | Workflow type, input, options |
| Complete workflow | CompleteWorkflowExecution | Result |
| Fail workflow | FailWorkflowExecution | Error |
| Cancel workflow | CancelWorkflowExecution | - |
| `workflow.Go()` (Go) | (internal) | Goroutine scheduling order |

---

## Changes That BREAK Replay (Need Versioning)

### 1. Adding a new activity call

```python
# BEFORE
@workflow.defn
class OrderWorkflow:
    @workflow.run
    async def run(self, order: Order) -> str:
        result = await workflow.execute_activity(
            validate_order, order, start_to_close_timeout=timedelta(seconds=30)
        )
        return await workflow.execute_activity(
            charge_payment, order, start_to_close_timeout=timedelta(seconds=30)
        )

# AFTER
@workflow.defn
class OrderWorkflow:
    @workflow.run
    async def run(self, order: Order) -> str:
        result = await workflow.execute_activity(
            validate_order, order, start_to_close_timeout=timedelta(seconds=30)
        )
        await workflow.execute_activity(              # <-- NEW COMMAND inserted before charge_payment
            check_inventory, order, start_to_close_timeout=timedelta(seconds=10)
        )
        return await workflow.execute_activity(
            charge_payment, order, start_to_close_timeout=timedelta(seconds=30)
        )
```

Replay expects command 2 to be `ScheduleActivityTask(charge_payment)`. It now sees `ScheduleActivityTask(check_inventory)`.

### 2. Removing an activity call

```python
# BEFORE
await workflow.execute_activity(validate_order, order, start_to_close_timeout=timedelta(seconds=30))
await workflow.execute_activity(check_inventory, order, start_to_close_timeout=timedelta(seconds=10))
await workflow.execute_activity(charge_payment, order, start_to_close_timeout=timedelta(seconds=30))

# AFTER - check_inventory removed
await workflow.execute_activity(validate_order, order, start_to_close_timeout=timedelta(seconds=30))
await workflow.execute_activity(charge_payment, order, start_to_close_timeout=timedelta(seconds=30))  # <-- now command 2, was command 3
```

Replay expects command 2 to be `ScheduleActivityTask(check_inventory)`. It now sees `ScheduleActivityTask(charge_payment)`.

### 3. Reordering activity calls

```python
# BEFORE
await workflow.execute_activity(send_email, data, start_to_close_timeout=timedelta(seconds=30))
await workflow.execute_activity(update_database, data, start_to_close_timeout=timedelta(seconds=30))

# AFTER - swapped order
await workflow.execute_activity(update_database, data, start_to_close_timeout=timedelta(seconds=30))  # <-- was command 2, now command 1
await workflow.execute_activity(send_email, data, start_to_close_timeout=timedelta(seconds=30))
```

### 4. Wrapping existing command in conditional on mutable state

```python
# BEFORE
await workflow.execute_activity(send_notification, data, start_to_close_timeout=timedelta(seconds=30))

# AFTER
if self.should_notify:  # <-- mutable workflow state (e.g., set by signal)
    await workflow.execute_activity(send_notification, data, start_to_close_timeout=timedelta(seconds=30))
```

If `self.should_notify` was `True` during original execution but `False` during replay (because signal timing differs), the command is skipped and the sequence breaks.

### 5. Changing which activity is called

```python
# BEFORE
await workflow.execute_activity(
    send_email_v1, data, start_to_close_timeout=timedelta(seconds=30)
)

# AFTER
await workflow.execute_activity(
    send_email_v2, data, start_to_close_timeout=timedelta(seconds=30)  # <-- different activity type
)
```

The command records the activity type name. `send_email_v1` != `send_email_v2`.

### 6. Loop iteration count change

```python
# BEFORE
for item in order.items[:10]:  # fixed upper bound
    await workflow.execute_activity(
        process_item, item, start_to_close_timeout=timedelta(seconds=30)
    )

# AFTER
for item in order.items:  # <-- no limit; list may have grown since original execution
    await workflow.execute_activity(
        process_item, item, start_to_close_timeout=timedelta(seconds=30)
    )
```

If `order.items` had 10 items originally, replay expects exactly 10 ScheduleActivityTask commands. A code change that processes a different number of items breaks the sequence.

### 7. Sequential to parallel conversion

```python
# BEFORE (sequential - commands 1, 2 issued one at a time)
result_a = await workflow.execute_activity(
    step_a, data, start_to_close_timeout=timedelta(seconds=30)
)
result_b = await workflow.execute_activity(
    step_b, data, start_to_close_timeout=timedelta(seconds=30)
)

# AFTER (parallel - both commands issued in the same workflow task)
result_a, result_b = await asyncio.gather(
    workflow.execute_activity(step_a, data, start_to_close_timeout=timedelta(seconds=30)),
    workflow.execute_activity(step_b, data, start_to_close_timeout=timedelta(seconds=30)),
)
```

Sequential execution issues one command per workflow task. Parallel issues both in the same task. The event history structure differs.

**TypeScript equivalent:**

```typescript
// BEFORE
const a = await executeActivity(stepA, data);
const b = await executeActivity(stepB, data);

// AFTER
const [a, b] = await Promise.all([
  executeActivity(stepA, data),
  executeActivity(stepB, data),  // <-- both commands in same WFT
]);
```

### 8. Timer elision

```python
# BEFORE
await workflow.execute_activity(step_one, data, start_to_close_timeout=timedelta(seconds=30))
await workflow.sleep(timedelta(minutes=5))  # <-- StartTimer command
await workflow.execute_activity(step_two, data, start_to_close_timeout=timedelta(seconds=30))

# AFTER - sleep removed
await workflow.execute_activity(step_one, data, start_to_close_timeout=timedelta(seconds=30))
await workflow.execute_activity(step_two, data, start_to_close_timeout=timedelta(seconds=30))  # <-- now command 2, was command 3
```

`workflow.sleep()` produces a StartTimer command. Removing it shifts all subsequent commands.

### 9. Error handling path change

```python
# BEFORE
await workflow.execute_activity(process_payment, order, start_to_close_timeout=timedelta(seconds=30))

# AFTER
try:
    await workflow.execute_activity(process_payment, order, start_to_close_timeout=timedelta(seconds=30))
except ActivityError:
    await workflow.execute_activity(
        rollback_payment, order, start_to_close_timeout=timedelta(seconds=30)  # <-- new command on failure path
    )
    await workflow.execute_activity(
        notify_failure, order, start_to_close_timeout=timedelta(seconds=30)    # <-- new command on failure path
    )
```

If the original execution saw `process_payment` succeed (no error path), replay follows the same path and this is fine. But if the original execution saw `process_payment` fail and had no catch block (workflow failed), this changes what happens on that failure path.

### 10. ContinueAsNew condition change

```python
# BEFORE
if self.iterations > 1000:
    workflow.continue_as_new(self.state)

# AFTER
if self.iterations > 500:  # <-- triggers earlier
    workflow.continue_as_new(self.state, memo={"migrated": True})  # <-- different args
```

Changes when ContinueAsNew fires and what arguments it carries. In-flight workflows past iteration 500 but below 1000 will now ContinueAsNew where they previously would have continued processing, changing the command sequence.

### 11. Go goroutine creation reorder

```go
// BEFORE
workflow.Go(ctx, func(ctx workflow.Context) {
    _ = workflow.ExecuteActivity(ctx, ActivityA, data).Get(ctx, nil)
})
workflow.Go(ctx, func(ctx workflow.Context) {
    _ = workflow.ExecuteActivity(ctx, ActivityB, data).Get(ctx, nil)
})

// AFTER - swapped
workflow.Go(ctx, func(ctx workflow.Context) {
    _ = workflow.ExecuteActivity(ctx, ActivityB, data).Get(ctx, nil)  // <-- was second goroutine
})
workflow.Go(ctx, func(ctx workflow.Context) {
    _ = workflow.ExecuteActivity(ctx, ActivityA, data).Get(ctx, nil)
})
```

Go SDK schedules goroutines in creation order. Swapping `workflow.Go()` calls changes which command is issued first.

---

## Changes That Are SAFE (No Versioning Needed)

### 1. Activity implementation code

```python
# Changing what happens INSIDE an activity is always safe.
# Activities are not replayed - only their recorded results are used.

@activity.defn
async def send_email(data: EmailData) -> str:
    # BEFORE: used SMTP
    # AFTER: switched to SendGrid API  <-- safe, no workflow commands affected
    async with httpx.AsyncClient() as client:
        resp = await client.post("https://api.sendgrid.com/v3/mail/send", json=payload)
    return resp.json()["id"]
```

### 2. Adding a query handler

```python
# BEFORE
@workflow.defn
class OrderWorkflow:
    @workflow.run
    async def run(self, order: Order) -> str:
        ...

# AFTER
@workflow.defn
class OrderWorkflow:
    @workflow.run
    async def run(self, order: Order) -> str:
        ...

    @workflow.query                           # <-- safe, queries don't produce commands
    def get_status(self) -> str:
        return self.status
```

### 3. Adding a signal handler

```python
# Adding a new signal handler is safe. Signal handlers respond to incoming signals
# but don't change the existing command sequence. The handler itself may call
# command-producing operations - those are appended, not inserted.

@workflow.signal
async def cancel_order(self) -> None:
    self.cancelled = True
```

### 4. Adding an update handler

```python
@workflow.update
async def modify_quantity(self, new_qty: int) -> str:
    self.quantity = new_qty
    return f"Updated to {new_qty}"
```

Additive handler registration. Does not change the command sequence of the main workflow run method.

### 5. Changing local variables

```python
# BEFORE
discount = price * 0.1

# AFTER
discount = price * 0.15  # <-- safe, no commands involved
```

### 6. Changing activity retry policy

```python
# BEFORE
await workflow.execute_activity(
    send_email, data,
    start_to_close_timeout=timedelta(seconds=30),
    retry_policy=RetryPolicy(maximum_attempts=3),
)

# AFTER
await workflow.execute_activity(
    send_email, data,
    start_to_close_timeout=timedelta(seconds=30),
    retry_policy=RetryPolicy(maximum_attempts=5, backoff_coefficient=3.0),  # <-- safe
)
```

The server stores the original retry policy for in-flight workflows. New policy applies only to new executions.

### 7. Changing activity timeout values

```python
# BEFORE
await workflow.execute_activity(
    process_order, order, start_to_close_timeout=timedelta(seconds=30)
)

# AFTER
await workflow.execute_activity(
    process_order, order, start_to_close_timeout=timedelta(seconds=60)  # <-- safe
)
```

Timeout changes don't affect command matching. The original timeout is recorded in history for in-flight workflows.

### 8. Changing timer duration

```python
# BEFORE
await workflow.sleep(timedelta(minutes=5))

# AFTER
await workflow.sleep(timedelta(minutes=10))  # <-- safe (may surprise developer)
```

The timer command exists in both versions. Duration is a parameter, not a sequence-affecting change. In-flight workflows use the originally recorded duration.

### 9. Changing workflow logging

```python
# Safe ONLY when using Temporal's replay-aware logger:
workflow.logger.info("Processing order %s", order.id)  # <-- suppressed during replay

# UNSAFE if using standard logging in a way that has side effects,
# but the logging call itself doesn't produce commands.
```

### 10. Changing arguments passed to activities

```python
# BEFORE
await workflow.execute_activity(
    send_email, EmailData(to=user.email, subject="Order confirmed"),
    start_to_close_timeout=timedelta(seconds=30),
)

# AFTER
await workflow.execute_activity(
    send_email, EmailData(to=user.email, subject="Your order is confirmed!", cc=user.manager),  # <-- safe
    start_to_close_timeout=timedelta(seconds=30),
)
```

Activity input is part of the command, but command matching uses activity type and position, not input content. Changed arguments apply only to new executions.

### 11. Bug fixes that don't change command flow

```python
# BEFORE
total = sum(item.price for item in order.items)  # bug: missing tax

# AFTER
total = sum(item.price * (1 + item.tax_rate) for item in order.items)  # <-- safe, used in return value only
```

If the computed value is only used as a workflow return value or passed to an activity, the command sequence is unchanged.

---

## Edge Cases (Requires Careful Analysis)

### 1. Condition on immutable workflow input

```python
# This is SAFE because workflow input never changes between execution and replay:
@workflow.run
async def run(self, order: Order) -> str:
    if order.premium:  # <-- immutable input, same on replay
        await workflow.execute_activity(
            priority_processing, order, start_to_close_timeout=timedelta(seconds=30)
        )
    await workflow.execute_activity(
        standard_processing, order, start_to_close_timeout=timedelta(seconds=30)
    )
```

**Safe** if the condition depends on workflow input (frozen at start). **Unsafe** if the condition depends on mutable workflow state (signals, update handlers, counters that accumulate differently).

Key question for the reviewer: is the branching variable set from workflow input, or from runtime state?

### 2. Refactored helper hiding a reorder

```python
# BEFORE (inline)
await workflow.execute_activity(validate, data, start_to_close_timeout=timedelta(seconds=30))
await workflow.execute_activity(enrich, data, start_to_close_timeout=timedelta(seconds=30))
await workflow.execute_activity(store, data, start_to_close_timeout=timedelta(seconds=30))

# AFTER (extracted to helper)
async def _process_pipeline(self, data):
    await workflow.execute_activity(validate, data, start_to_close_timeout=timedelta(seconds=30))
    await workflow.execute_activity(enrich, data, start_to_close_timeout=timedelta(seconds=30))
    await workflow.execute_activity(store, data, start_to_close_timeout=timedelta(seconds=30))

# In workflow.run:
await self._process_pipeline(data)
```

The diff shows deleted activity calls and a new method call. Must verify the helper preserves the same command order. Pure extraction with identical ordering is safe.

### 3. Changing data structures that drive command loops

```python
# BEFORE
steps = ["validate", "enrich", "store"]
for step_name in steps:
    await workflow.execute_activity(
        dispatch_step, step_name, start_to_close_timeout=timedelta(seconds=30)
    )

# AFTER
steps = ["validate", "transform", "enrich", "store"]  # <-- "transform" inserted
for step_name in steps:
    await workflow.execute_activity(
        dispatch_step, step_name, start_to_close_timeout=timedelta(seconds=30)
    )
```

No activity call was added or removed in the code. But the list drives the loop, so the command sequence changes. Look for changes to any collection that controls iteration over command-producing operations.

### 4. Search attribute mutations

```python
# BEFORE
workflow.upsert_search_attributes({"OrderStatus": ["processing"]})
await workflow.execute_activity(process, data, start_to_close_timeout=timedelta(seconds=30))

# AFTER - upsert moved after activity
await workflow.execute_activity(process, data, start_to_close_timeout=timedelta(seconds=30))
workflow.upsert_search_attributes({"OrderStatus": ["processing"]})  # <-- reordered, breaks replay
```

`UpsertSearchAttributes` produces a command. Developers often think of it as metadata-only, but reordering or conditionally skipping upserts changes the sequence.

### 5. SideEffect / MutableSideEffect changes

```python
# BEFORE
token = workflow.side_effect(lambda: generate_token())
await workflow.execute_activity(call_api, token, start_to_close_timeout=timedelta(seconds=30))

# AFTER - added another side_effect before the existing one
request_id = workflow.side_effect(lambda: str(uuid4()))  # <-- new RecordMarker command
token = workflow.side_effect(lambda: generate_token())
await workflow.execute_activity(call_api, token, start_to_close_timeout=timedelta(seconds=30))
```

SideEffect and MutableSideEffect produce RecordMarker commands. Adding, removing, or reordering them is a sequence change.

### 6. Changing activity options beyond retry/timeout

```python
# BEFORE
await workflow.execute_activity(
    process, data,
    start_to_close_timeout=timedelta(seconds=30),
    task_queue="default",
)

# AFTER
await workflow.execute_activity(
    process, data,
    start_to_close_timeout=timedelta(seconds=30),
    task_queue="priority",  # <-- different task queue
)
```

Some option changes (task queue, activity ID) may affect command matching depending on SDK version and server behavior. Changing `task_queue` on an in-flight workflow can cause the replayed command to target a different queue than the one recorded in history. Treat non-retry, non-timeout option changes as potentially unsafe and verify against the specific SDK version.
