# Python SDK Nexus Reference

## Overview

This file documents the Python SDK APIs for Temporal Nexus: decorators, classes, methods, and Worker/caller wiring. For Nexus concepts (Endpoints, Services, Operations, lifecycle, cross-Namespace routing), see `references/core/nexus.md`.

Temporal Python SDK support for Nexus is Generally Available. <!-- docs/develop/python/nexus/feature-guide.mdx:20-24 -->

## Prerequisites

- Temporal CLI `v1.3.0` or higher. <!-- docs/develop/python/nexus/feature-guide.mdx:52 -->
- Temporal Python SDK `v1.14.1` or higher. <!-- docs/develop/python/nexus/feature-guide.mdx:53 -->
- The `temporalio` package (provides `temporalio.nexus`, `temporalio.workflow`, `temporalio.exceptions.NexusOperationError`). <!-- docs/develop/python/nexus/feature-guide.mdx:173, 336 -->
- The `nexusrpc` package (provides `@nexusrpc.service`, `nexusrpc.Operation`, `nexusrpc.handler.*`, `nexusrpc.OperationError`, `nexusrpc.HandlerError`). <!-- docs/develop/python/nexus/feature-guide.mdx:106, 144, 334-335 -->

## Define the Service contract

Declare the Service as a class decorated with `@nexusrpc.service`. Each Operation is a class attribute annotated with `nexusrpc.Operation[InputType, OutputType]`. <!-- docs/develop/python/nexus/feature-guide.mdx:119-123 -->

```python
from dataclasses import dataclass
import nexusrpc

@dataclass
class MyInput:
    name: str

@dataclass
class MyOutput:
    message: str

@nexusrpc.service
class MyNexusService:
    my_sync_operation: nexusrpc.Operation[MyInput, MyOutput]
    my_workflow_run_operation: nexusrpc.Operation[MyInput, MyOutput]
```

<!-- Sources: docs/develop/python/nexus/feature-guide.mdx:103-123 -->

- A Nexus Operation takes exactly one input parameter; map multiple Workflow arguments via the input dataclass and unpack with `args=[...]` on the handler side. <!-- docs/develop/python/nexus/feature-guide.mdx:230 -->
- The default Data Converter encodes payloads as Null, Byte array, Protobuf JSON, then JSON; dataclasses serialize to JSON. <!-- docs/develop/python/nexus/feature-guide.mdx:96-99 -->

## Implement Operation handlers

A handler class is decorated with `@nexusrpc.handler.service_handler(service=MyNexusService)` — the `service=` kwarg binds the handler to its contract. <!-- docs/develop/python/nexus/feature-guide.mdx:146-147 -->

Two utility decorators define Operations: <!-- docs/develop/python/nexus/feature-guide.mdx:132-135 -->

- `nexusrpc.handler.sync_operation` — synchronous handler.
- `nexus.workflow_run_operation` — asynchronous handler that starts a Workflow.

### Synchronous Operation handler

Decorate an `async def` method whose signature is `(self, ctx: nexusrpc.handler.StartOperationContext, input: I) -> O`. <!-- docs/develop/python/nexus/feature-guide.mdx:148-152 -->

```python
import nexusrpc

@nexusrpc.handler.service_handler(service=MyNexusService)
class MyNexusServiceHandler:
    @nexusrpc.handler.sync_operation
    async def my_sync_operation(
        self, ctx: nexusrpc.handler.StartOperationContext, input: MyInput
    ) -> MyOutput:
        return MyOutput(message=f"Hello {input.name} from sync operation!")
```

<!-- Sources: docs/develop/python/nexus/feature-guide.mdx:143-153 -->

- A sync handler must return in under `10s`; for longer work use `@nexus.workflow_run_operation` below. <!-- docs/develop/python/nexus/feature-guide.mdx:156 -->
- Handlers should be reliable: 5 consecutive retryable errors trip the circuit breaker and block all Operations from the caller to that Endpoint. <!-- docs/develop/python/nexus/feature-guide.mdx:130, 157 -->

### Sync handler accessing Temporal primitives

From inside a sync handler, get the Worker's Temporal Client via `nexus.client()` to Signal, Query, or Update an existing Workflow (or Signal-With-Start / Update-With-Start). <!-- docs/develop/python/nexus/feature-guide.mdx:161-167 -->

```python
from temporalio import nexus

@nexusrpc.handler.service_handler(service=NexusGreetingService)
class NexusGreetingServiceHandler:
    def _get_workflow_handle(self, user_id: str):
        return nexus.client().get_workflow_handle_for(
            GreetingWorkflow.run, get_workflow_id(user_id)
        )
```

<!-- Sources: docs/develop/python/nexus/feature-guide.mdx:172-189 -->

- All calls inside a sync handler must complete within the Nexus request timeout; keep Updates short-lived. <!-- docs/develop/python/nexus/feature-guide.mdx:163 -->
- Use `nexus.info()` to access information about the currently-executing Operation, including its Task Queue. <!-- docs/develop/python/nexus/feature-guide.mdx:194 -->

### Async (Workflow-run) Operation handler

Decorate an `async def` whose signature is `(self, ctx: nexus.WorkflowRunOperationContext, input: I) -> nexus.WorkflowHandle[O]`, and return the handle from `await ctx.start_workflow(...)`. <!-- docs/develop/python/nexus/feature-guide.mdx:199, 209-218 -->

```python
import uuid
import nexusrpc
from temporalio import nexus

@nexusrpc.handler.service_handler(service=MyNexusService)
class MyNexusServiceHandler:
    @nexus.workflow_run_operation
    async def my_workflow_run_operation(
        self, ctx: nexus.WorkflowRunOperationContext, input: MyInput
    ) -> nexus.WorkflowHandle[MyOutput]:
        return await ctx.start_workflow(
            WorkflowStartedByNexusOperation.run,
            input,
            id=str(uuid.uuid4()),
        )
```

<!-- Sources: docs/develop/python/nexus/feature-guide.mdx:203-218 -->

- Start the underlying Workflow with `await ctx.start_workflow(...)` — do **not** use a `Client` from inside an async handler. <!-- docs/develop/python/nexus/feature-guide.mdx:213-217 -->
- Use a business-meaningful Workflow `id` (passed in via the Operation input where possible); the ID dedupes Workflow starts. <!-- docs/develop/python/nexus/feature-guide.mdx:220 -->
- The Workflow's Task Queue defaults to the Task Queue the Operation is handled on. <!-- docs/develop/python/nexus/quickstart.mdx:108 -->
- Attach multiple Nexus callers to one handler Workflow with Conflict-Policy `Use-Existing`. <!-- docs/develop/python/nexus/feature-guide.mdx:224 -->

#### Multi-argument Workflows

A Nexus Operation has one input, but the started Workflow can take many. Unpack the input via `args=[...]`: <!-- docs/develop/python/nexus/feature-guide.mdx:228-258 -->

```python
@nexus.workflow_run_operation
async def hello(
    self, ctx: nexus.WorkflowRunOperationContext, input: HelloInput
) -> nexus.WorkflowHandle[HelloOutput]:
    return await ctx.start_workflow(
        HelloHandlerWorkflow.run,
        args=[input.name, input.language],
        id=f"hello-multi-args-{input.name}-{input.language}",
    )
```

<!-- Sources: docs/develop/python/nexus/feature-guide.mdx:244-258 -->

## Register handlers on a Worker

Pass instantiated handlers to the Worker via the `nexus_service_handlers=[...]` kwarg. Constructor arguments needed by your handler go into its `__init__`. <!-- docs/develop/python/nexus/feature-guide.mdx:267-282 -->

```python
from temporalio.client import Client
from temporalio.worker import Worker

async def main():
    client = await Client.connect("localhost:7233", namespace=NAMESPACE)
    worker = Worker(
        client,
        task_queue=TASK_QUEUE,
        workflows=[WorkflowStartedByNexusOperation],
        nexus_service_handlers=[MyNexusServiceHandler()],
    )
    await worker.run()
```

<!-- Sources: docs/develop/python/nexus/feature-guide.mdx:272-282 -->

- A Worker only polls for Nexus tasks when handlers are registered via `nexus_service_handlers`. <!-- docs/develop/python/nexus/quickstart.mdx:165-167 -->
- Operation handlers typically live in the same Worker as the underlying Temporal primitives they abstract. <!-- docs/develop/python/nexus/feature-guide.mdx:127 -->

## Caller Workflow

Import the Service contract through the determinism sandbox, then build a Nexus client via `workflow.create_nexus_client(service=..., endpoint=...)`. <!-- docs/develop/python/nexus/feature-guide.mdx:293-303 -->

```python
from temporalio import workflow

with workflow.unsafe.imports_passed_through():
    from hello_nexus.service import MyInput, MyNexusService, MyOutput

@workflow.defn
class CallerWorkflow:
    @workflow.run
    async def run(self, name: str) -> tuple[MyOutput, MyOutput]:
        nexus_client = workflow.create_nexus_client(
            service=MyNexusService,
            endpoint=NEXUS_ENDPOINT,
        )
        # One-shot: start and wait in a single await.
        wf_result = await nexus_client.execute_operation(
            MyNexusService.my_workflow_run_operation,
            MyInput(name),
        )
        # Two-phase: obtain a handle, then await it for the result.
        sync_operation_handle = await nexus_client.start_operation(
            MyNexusService.my_sync_operation,
            MyInput(name),
        )
        sync_result = await sync_operation_handle
        return sync_result, wf_result
```

<!-- Sources: docs/develop/python/nexus/feature-guide.mdx:290-317 -->

- `execute_operation(op, input)` starts the Operation and waits for the result in one `await`. <!-- docs/develop/python/nexus/feature-guide.mdx:304-308 -->
- `start_operation(op, input)` returns the started state (including the operation token for async Operations); `await` the handle to obtain the result. <!-- docs/develop/python/nexus/feature-guide.mdx:309-315 -->
- Register and start the caller Workflow exactly like any other Workflow (`client.start_workflow` / `client.execute_workflow`). <!-- docs/develop/python/nexus/feature-guide.mdx:319-327 -->

## Setting Operation timeouts

Pass timeout kwargs on `execute_operation` (or `start_operation`). The Python docs show `schedule_to_close_timeout`: <!-- docs/develop/python/nexus/quickstart.mdx:190-194 -->

```python
return await nexus_client.execute_operation(
    SayHelloNexusService.say_hello,
    MyInput(name=name),
    schedule_to_close_timeout=timedelta(seconds=10),
)
```

<!-- Sources: docs/develop/python/nexus/quickstart.mdx:190-194 -->

<!-- VERIFY: schedule_to_start_timeout and start_to_close_timeout are not shown in the Python Nexus docs; only schedule_to_close_timeout is documented. -->

## Cancellation

Only asynchronous Operations can be canceled; cancellation flows via the operation token. Call `handle.cancel()` on the Operation handle returned by `start_operation`. <!-- docs/develop/python/nexus/feature-guide.mdx:340 -->

- The Workflow or resources backing the Operation may ignore the cancellation request, in which case the Operation can still enter a terminal state. <!-- docs/develop/python/nexus/feature-guide.mdx:341-342 -->

Set the `cancellation_type` kwarg when starting/executing an Operation. The four values: <!-- docs/develop/python/nexus/feature-guide.mdx:344-351 -->

- `ABANDON` — do not request cancellation of the Operation. <!-- docs/develop/python/nexus/feature-guide.mdx:346 -->
- `TRY_CANCEL` — initiate the cancellation request and immediately report cancellation to the caller; delivery is not guaranteed if the caller exits first. <!-- docs/develop/python/nexus/feature-guide.mdx:347 -->
- `WAIT_REQUESTED` — request cancellation and wait for confirmation the request was received; does not wait for actual cancellation. <!-- docs/develop/python/nexus/feature-guide.mdx:348 -->
- `WAIT_COMPLETED` — wait for Operation completion (may or may not complete as cancelled). <!-- docs/develop/python/nexus/feature-guide.mdx:349 -->

Default is `WAIT_COMPLETED`. <!-- docs/develop/python/nexus/feature-guide.mdx:351 -->

Once the caller Workflow completes, the caller's Nexus Machinery makes no further attempts to cancel running Operations; to ensure delivery, await all pending Operations before exiting. <!-- docs/develop/python/nexus/feature-guide.mdx:353-355 -->

## Exceptions

Three Nexus-specific exception classes: <!-- docs/develop/python/nexus/feature-guide.mdx:332-336 -->

- `nexusrpc.OperationError` — raise from inside an Operation handler to indicate the Operation failed per its own application logic and should not be retried. <!-- docs/develop/python/nexus/feature-guide.mdx:334 -->
- `nexusrpc.HandlerError` — raise with a specific `nexusrpc.HandlerErrorType`; retryability follows the type per the Nexus spec. <!-- docs/develop/python/nexus/feature-guide.mdx:335 -->
- `temporalio.exceptions.NexusOperationError` — raised inside the caller Workflow when a Nexus Operation fails for any reason; walk the cause chain via the `__cause__` attribute. <!-- docs/develop/python/nexus/feature-guide.mdx:336 -->

`HandlerErrorType` categories: <!-- docs/develop/python/nexus/feature-guide.mdx:335 -->

- Non-retryable: `BAD_REQUEST`, `UNAUTHENTICATED`, `UNAUTHORIZED`, `NOT_FOUND`, `NOT_IMPLEMENTED`.
- Retryable: `RESOURCE_EXHAUSTED`, `INTERNAL`, `UNAVAILABLE`, `UPSTREAM_TIMEOUT`.

## Observability

Workflow history events surfaced in the caller's history: <!-- docs/develop/python/nexus/feature-guide.mdx:454-467 -->

- Async Operations: `NexusOperationScheduled`, `NexusOperationStarted`, `NexusOperationCompleted`.
- Sync Operations: `NexusOperationScheduled`, `NexusOperationCompleted` (no `NexusOperationStarted`).

Use `temporal workflow describe -w <ID>` to see pending Nexus Operations and attached callbacks; `temporal workflow show -w <ID>` to see the full history. <!-- docs/develop/python/nexus/feature-guide.mdx:442-452 -->

## Common mistakes

1. Using `@nexus.sync_operation` (wrong) — the sync decorator is `@nexusrpc.handler.sync_operation`. <!-- docs/develop/python/nexus/feature-guide.mdx:139, 148 -->
2. Using `@nexusrpc.handler.workflow_run_operation` (wrong) — the Workflow-run decorator is `@nexus.workflow_run_operation` from `temporalio.nexus`. <!-- docs/develop/python/nexus/feature-guide.mdx:199, 209 -->
3. Omitting the `service=` kwarg from `@nexusrpc.handler.service_handler(service=MyNexusService)`. <!-- docs/develop/python/nexus/feature-guide.mdx:146 -->
4. Omitting `service=` or `endpoint=` from `workflow.create_nexus_client(...)`. <!-- docs/develop/python/nexus/feature-guide.mdx:300-303 -->
5. Running long work in a sync handler — the deadline is `10s`; use `@nexus.workflow_run_operation` for long-running work. <!-- docs/develop/python/nexus/feature-guide.mdx:156 -->
6. Calling `Client.start_workflow` from inside an async handler — use `await ctx.start_workflow(...)` instead. <!-- docs/develop/python/nexus/feature-guide.mdx:213 -->
7. Forgetting `args=[...]` for multi-arg Workflows when mapping a single Operation input. <!-- docs/develop/python/nexus/feature-guide.mdx:230, 253-256 -->
8. Importing the Service module inside a Workflow without `with workflow.unsafe.imports_passed_through():`. <!-- docs/develop/python/nexus/feature-guide.mdx:293-294 -->
9. Omitting `nexus_service_handlers=[...]` on `Worker(...)` — without it the Worker will not poll for Nexus tasks. <!-- docs/develop/python/nexus/quickstart.mdx:147, 165-167 -->
10. Confusing `nexusrpc.OperationError` (Operation-level, non-retryable application failure) with `nexusrpc.HandlerError` (handler-level failure whose retryability is determined by `HandlerErrorType`). <!-- docs/develop/python/nexus/feature-guide.mdx:334-335 -->
11. Trying to cancel a synchronous Operation — only async Operations can be canceled. <!-- docs/develop/python/nexus/feature-guide.mdx:340 -->
12. Exiting the caller Workflow with pending Operations expecting cancels to be delivered — the Nexus Machinery stops delivering cancels once the caller completes. <!-- docs/develop/python/nexus/feature-guide.mdx:353-355 -->
