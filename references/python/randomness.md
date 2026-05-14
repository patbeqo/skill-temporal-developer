# Python Workflow Randomness

Guidance for generating random numbers and UUIDs inside a `@workflow.defn` class. Time (`workflow.now()`) and replay detection (`workflow.unsafe.is_replaying`) are separate topics — do not absorb them here. Activities are unaffected and may use any stdlib API.

## The constraint

Workflow code must be deterministic to support replay.

The list of forbidden behaviors inside a workflow explicitly includes "no randomness".

Inline random branching is an example of intrinsic non-determinism: a workflow that branches on a random number can emit a different Command sequence on re-execution and will fail replay.

The SDK's replay-safe APIs store their results in Event History so that re-executed workflow code issues the same sequence of Commands even when branching is involved.

## The two replay-safe APIs

### `workflow.random()`

Returns a deterministic `random.Random` instance seeded per Workflow Execution.

The return value is an instance — call methods on it (`randint`, `choice`, `random`, etc.); calling `workflow.random()` by itself does not produce a number.

The seed is stable across replays of the same Workflow Execution; do not describe it as regenerated on replay.

```python
from temporalio import workflow

@workflow.defn
class PickWorkflow:
    @workflow.run
    async def run(self) -> int:
        value = workflow.random().randint(1, 100)  # single draw
        rng = workflow.random()                    # reuse for multiple draws
        choice = rng.choice(["a", "b", "c"])
        return value
```

Never use `random.random()` or other `random` module functions directly inside a workflow.

### `workflow.uuid4()`

Returns a deterministic UUID4 for use inside a workflow.

This is the only documented UUID helper; deterministic versions of `uuid1`, `uuid3`, and `uuid5` are not provided.

```python
from temporalio import workflow

@workflow.defn
class IdWorkflow:
    @workflow.run
    async def run(self) -> str:
        unique_id = workflow.uuid4()
        return str(unique_id)
```

Never use `uuid.uuid4()` directly inside a workflow.

## Don't do X, do Y

Inside a `@workflow.defn` class:

| Don't | Do |
|---|---|
| `import random; random.random()` | `workflow.random().random()`  |
| `random.randint(1, 100)` | `workflow.random().randint(1, 100)`  |
| `random.choice([...])` | `workflow.random().choice([...])`  |
| `random.seed(...)` to make stdlib `random` "deterministic" | use `workflow.random()`  |
| `import uuid; uuid.uuid4()` | `workflow.uuid4()`  |
| `uuid.uuid1()` / `uuid.uuid3()` / `uuid.uuid5()` inside a workflow | move the call to an Activity; only `workflow.uuid4()` is documented |

These rules apply only inside `@workflow.defn` classes. Activities can use any stdlib randomness or UUID API.

## Cryptographic randomness

Cryptographically secure randomness (`secrets`, `random.SystemRandom`, `os.urandom`) is not available inside a workflow. Move the call to an Activity and return the result to the workflow.

## Interceptors

Workflow inbound and outbound interceptor methods also execute during replay; they must use the same replay-safe APIs (`workflow.random()`, `workflow.uuid4()`) for randomness and UUIDs.
