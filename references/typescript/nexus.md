# Temporal TypeScript SDK Nexus Reference

TypeScript SDK identifiers, APIs, and idioms for building Nexus Services, Operation handlers, and caller Workflows. See `references/core/nexus.md` for concepts (Endpoint, Service, Operation, caller/handler roles).

> [!NOTE]
> This feature is in Public Preview. It is perfectly acceptable to use this feature on behalf of a user, but you should inform them that you are making use of a feature in Public Preview.

## Prerequisites

- Temporal CLI `v1.3.0` or higher.
- Temporal TypeScript SDK `v1.12.3` or higher.
- Packages: `nexus-rpc` (Service contract + handler), `@temporalio/nexus` (Workflow-run helper, `getClient`, `startWorkflow`), `@temporalio/worker` (registration), `@temporalio/workflow` (caller-side client), `@temporalio/client` (start the caller Workflow).
- Run a dev Server with `temporal server start-dev` and create the Endpoint with `temporal operator nexus endpoint create --name <name> --target-namespace <ns> --target-task-queue <tq>`.

## Define the Service contract

Put the Service contract in a shared module so the caller and handler import the same definition.

```ts
import * as nexus from 'nexus-rpc';

export const helloService = nexus.service('hello', {
  echo: nexus.operation<EchoInput, EchoOutput>(),
  hello: nexus.operation<HelloInput, HelloOutput>(),
});

export interface EchoInput { message: string }
export interface EchoOutput { message: string }
export interface HelloInput { name: string; language: LanguageCode }
export interface HelloOutput { message: string }
export type LanguageCode = 'en' | 'fr' | 'de' | 'es' | 'tr';
```

- Service contract: `nexus.service('<name>', { … })` from `nexus-rpc`.
- Operation declaration: `nexus.operation<Input, Output>()` — input and output are the only type parameters.
- The default Data Converter encodes payloads as Null, byte array, or JSON; use plain TypeScript objects (serialized to JSON) for polyglot interop.

## Implement Operation handlers

A Service handler implements every Operation declared by the Service contract using `nexus.serviceHandler(service, { … })` from `nexus-rpc`.

### Synchronous handler (plain async function)

A synchronous Operation handler is a plain async function on the service handler object — there is no special wrapper.

```ts
import * as nexus from 'nexus-rpc';
import { helloService, EchoInput, EchoOutput } from '../api';

export const helloServiceHandler = nexus.serviceHandler(helloService, {
  echo: async (ctx, input: EchoInput): Promise<EchoOutput> => {
    return input;
  },
  // ...
});
```

- Signature: `async (ctx, input: I): Promise<O> => { … }`.
- Use sync handlers for short RPC-style work (lookups, computations, Signals/Queries/Updates). Handlers must be reliable: the circuit breaker trips after 5 consecutive retryable errors.

### Sync handler that calls Temporal (Signal / Query / Update)

```ts
import * as nexus from 'nexus-rpc';
import * as temporalNexus from '@temporalio/nexus';

export const nexusGreetingServiceHandler = nexus.serviceHandler(nexusGreetingService, {
  getLanguages: async (ctx, input: GetLanguagesInput) => {
    const client = temporalNexus.getClient();
    const handle = client.workflow.getHandle(workflowIdForUser(input.userId));
    return await handle.query(getLanguagesQuery);
  },
});
```

- `temporalNexus.getClient()` returns a Temporal Client connected via the same `NativeConnection` as the host Worker.
- All work inside a sync handler must complete within the Nexus request timeout.
- `ctx.abortSignal` is an `AbortSignal` that fires when the deadline is exceeded — pass it to Temporal Client calls so they are canceled on timeout.
- `ctx.requestDeadline` is an optional `Date` for the current request's deadline; use it to decide whether to start work or to set downstream timeouts.
- Keep Updates inside sync handlers short-lived to stay within the request deadline.

### Asynchronous (Workflow-run) handler

To back an Operation with a Workflow, use `new WorkflowRunOperationHandler<I, O>(fn)` from `@temporalio/nexus` and start the Workflow with `temporalNexus.startWorkflow(ctx, workflow, options)`.

```ts
import { randomUUID } from 'crypto';
import * as nexus from 'nexus-rpc';
import * as temporalNexus from '@temporalio/nexus';
import { helloService, HelloInput, HelloOutput } from '../api';
import { helloWorkflow } from './workflows';

export const helloServiceHandler = nexus.serviceHandler(helloService, {
  hello: new temporalNexus.WorkflowRunOperationHandler<HelloInput, HelloOutput>(
    async (ctx, input: HelloInput) => {
      return await temporalNexus.startWorkflow(ctx, helloWorkflow, {
        args: [input],
        workflowId: ctx.requestId ?? randomUUID(),
        // Task queue defaults to the task queue this Operation is handled on.
      });
    },
  ),
});
```

- The delegate `fn` receives `(ctx, input)` and is the place to validate/transform input and customize Workflow start options.
- `ctx.requestId` is allocated by Temporal when the caller schedules the Operation and is stable across retries — use it as the `workflowId` to dedupe starts.
- Workflow IDs should be business-meaningful and are used to dedupe Workflow starts; pass the ID through the Operation input as part of the Service contract when possible.
- The Task Queue defaults to the queue the Operation handler runs on — set it explicitly only when targeting a different Worker fleet.

#### Multi-argument Workflows

A Nexus Operation takes a single input, so wrap multiple Workflow arguments in one input object and spread them into the `args` array on `startWorkflow`.

```ts
return await temporalNexus.startWorkflow(ctx, myWorkflow, {
  args: [input.x, input.y],
  workflowId: ctx.requestId ?? randomUUID(),
});
```

## Register handlers on a Worker

Pass `nexusServices: [...]` to `Worker.create({ … })`.

```ts
import { Worker, NativeConnection } from '@temporalio/worker';
import { helloServiceHandler } from './handler';

const connection = await NativeConnection.connect({ address: 'localhost:7233' });
const worker = await Worker.create({
  connection,
  namespace: 'my-target-namespace',
  taskQueue: 'my-handler-task-queue',
  workflowsPath: require.resolve('./workflows'),
  nexusServices: [helloServiceHandler],
});
```

- A Worker only polls for Nexus tasks if at least one handler is in `nexusServices`.
- Service handlers are typically hosted on the same Worker as the Workflows and Activities they wrap.

## Caller Workflow

Inside a Workflow, build a Nexus client with `createNexusServiceClient` from `@temporalio/workflow` and invoke `executeOperation(opName, input, options)`.

```ts
import * as wf from '@temporalio/workflow';
import { helloService, LanguageCode } from '../service/api';

const HELLO_SERVICE_ENDPOINT = 'hello-service-endpoint-name';

export async function helloCallerWorkflow(name: string, language: LanguageCode): Promise<string> {
  const nexusClient = wf.createNexusServiceClient({
    service: helloService,
    endpoint: HELLO_SERVICE_ENDPOINT,
  });

  const helloResult = await nexusClient.executeOperation(
    'hello',
    { name, language },
    { scheduleToCloseTimeout: '10s' },
  );

  return helloResult.message;
}
```

- `createNexusServiceClient({ service, endpoint })` — the `service` is the imported Service contract and `endpoint` is the Nexus Endpoint name.
- `executeOperation` starts the Operation and awaits its result.
- The caller depends only on the Service contract, not on handler code — keep the contract in a shared module.

## Operation timeouts

Set timeouts in the third argument of `executeOperation`. The documented option in TypeScript is `scheduleToCloseTimeout`.

```ts
await nexusClient.executeOperation('hello', input, { scheduleToCloseTimeout: '10s' });
```

## Cancellation

Nexus Operations execute inside Cancellation Scopes provided by `@temporalio/workflow`.

- Cancelling a Cancellation Scope cancels every cancellable operation owned by it, including Nexus Operations.
- The Workflow itself is the root Cancellation Scope: cancelling the caller Workflow propagates cancellation to all Nexus Operations it started.
- For per-Operation control, create a new Cancellation Scope and start the Operation inside it.
- Only asynchronous Operations can be canceled — cancellation is delivered via the operation token. Synchronous Operations cannot be canceled.
- The Workflow or resource backing the Operation may ignore the cancellation request.
- Once the caller Workflow completes, the Nexus Machinery makes no further attempts to cancel still-running Operations — `await` pending Operations before returning if you need cancellation to be delivered.

## Exceptions

TypeScript surfaces three Nexus-specific exception types.

- `OperationError` (from `nexus-rpc`) — throw inside a handler to signal application-level failure that should not be retried.
- `HandlerError` (from `nexus-rpc`) — throw with a `HandlerErrorType`. Non-retryable types: `BAD_REQUEST`, `UNAUTHENTICATED`, `UNAUTHORIZED`, `NOT_FOUND`, `NOT_IMPLEMENTED`. Retryable types: `RESOURCE_EXHAUSTED`, `INTERNAL`, `UNAVAILABLE`, `UPSTREAM_TIMEOUT`.
- `NexusOperationFailure` (from `@temporalio/nexus`) — thrown inside the caller Workflow when an Operation fails for any reason; inspect `cause` to walk the cause chain.

## Observability

Caller Workflow history for synchronous Operations contains `NexusOperationScheduled` and `NexusOperationCompleted`. For asynchronous Operations it also contains `NexusOperationStarted`.

For tracing, register `OpenTelemetryPlugin` from `@temporalio/interceptors-opentelemetry` via the Worker's `plugins` option — it auto-registers Nexus, Activity, and Workflow interceptors.

## Common mistakes

1. Forgetting `nexusServices: [...]` on `Worker.create` — the Worker silently never polls for Nexus tasks.
2. Using a wrapper type for sync handlers — there is no wrapper; a sync handler is a plain `async (ctx, input) => …` property on the `serviceHandler` object.
3. Building the Temporal Client manually inside a sync handler — use `temporalNexus.getClient()` so the call uses the Worker's existing `NativeConnection`.
4. Ignoring `ctx.abortSignal` for downstream calls — long Client calls will run past the Nexus request deadline if you don't pass it through.
5. Putting long-running work in a sync handler — sync handlers must complete inside the Nexus request timeout; back long work with a Workflow via `WorkflowRunOperationHandler`.
6. Generating a random `workflowId` for every retry of a Workflow-run Operation — use `ctx.requestId ?? randomUUID()` so retries dedupe to the same Workflow.
7. Passing multiple positional Workflow arguments through a Nexus Operation — Operations take a single input; wrap arguments in one object and spread into `args: [...]`.
8. Expecting to cancel a synchronous Operation — only asynchronous Operations support cancellation.
9. Letting the caller Workflow return while Operations are still in flight — pending cancellations are not delivered once the caller completes; `await` outstanding Operations first.
10. Mismatching the Endpoint name between the caller's `createNexusServiceClient({ endpoint })` and the Endpoint created via `temporal operator nexus endpoint create --name` — calls fail to route.
