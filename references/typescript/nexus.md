# Temporal TypeScript SDK Nexus Reference

TypeScript SDK identifiers, APIs, and idioms for building Nexus Services, Operation handlers, and caller Workflows. See `references/core/nexus.md` for concepts (Endpoint, Service, Operation, caller/handler roles).

> [!NOTE]
> This feature is in Public Preview. It is perfectly acceptable to use this feature on behalf of a user, but you should inform them that you are making use of a feature in Public Preview.

<!-- docs/develop/typescript/nexus/feature-guide.mdx:21-23 -->

## Prerequisites

- Temporal CLI `v1.3.0` or higher. <!-- docs/develop/typescript/nexus/feature-guide.mdx:51 -->
- Temporal TypeScript SDK `v1.12.3` or higher. <!-- docs/develop/typescript/nexus/feature-guide.mdx:52 -->
- Packages: `nexus-rpc` (Service contract + handler), `@temporalio/nexus` (Workflow-run helper, `getClient`, `startWorkflow`), `@temporalio/worker` (registration), `@temporalio/workflow` (caller-side client), `@temporalio/client` (start the caller Workflow). <!-- Sources: docs/develop/typescript/nexus/feature-guide.mdx:105,151,209,240,285,303 -->
- Run a dev Server with `temporal server start-dev` and create the Endpoint with `temporal operator nexus endpoint create --name <name> --target-namespace <ns> --target-task-queue <tq>`. <!-- docs/develop/typescript/nexus/feature-guide.mdx:57,81-85 -->

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

<!-- Sources: docs/develop/typescript/nexus/feature-guide.mdx:105-139 -->

- Service contract: `nexus.service('<name>', { … })` from `nexus-rpc`. <!-- docs/develop/typescript/nexus/feature-guide.mdx:107 -->
- Operation declaration: `nexus.operation<Input, Output>()` — input and output are the only type parameters. <!-- docs/develop/typescript/nexus/feature-guide.mdx:112,118 -->
- The default Data Converter encodes payloads as Null, byte array, or JSON; use plain TypeScript objects (serialized to JSON) for polyglot interop. <!-- docs/develop/typescript/nexus/feature-guide.mdx:95-98 -->

## Implement Operation handlers

A Service handler implements every Operation declared by the Service contract using `nexus.serviceHandler(service, { … })` from `nexus-rpc`. <!-- docs/develop/typescript/nexus/feature-guide.mdx:144,146,172 -->

### Synchronous handler (plain async function)

A synchronous Operation handler is a plain async function on the service handler object — there is no special wrapper. <!-- docs/develop/typescript/nexus/feature-guide.mdx:159 -->

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

<!-- Sources: docs/develop/typescript/nexus/feature-guide.mdx:167-184 -->

- Signature: `async (ctx, input: I): Promise<O> => { … }`. <!-- docs/develop/typescript/nexus/feature-guide.mdx:173 -->
- Use sync handlers for short RPC-style work (lookups, computations, Signals/Queries/Updates). Handlers must be reliable: the circuit breaker trips after 5 consecutive retryable errors. <!-- docs/develop/typescript/nexus/feature-guide.mdx:149,161 -->

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

<!-- Sources: docs/develop/typescript/nexus/feature-guide.mdx:209-222 -->

- `temporalNexus.getClient()` returns a Temporal Client connected via the same `NativeConnection` as the host Worker. <!-- docs/develop/typescript/nexus/feature-guide.mdx:154,160,217 -->
- All work inside a sync handler must complete within the Nexus request timeout. <!-- docs/develop/typescript/nexus/feature-guide.mdx:192 -->
- `ctx.abortSignal` is an `AbortSignal` that fires when the deadline is exceeded — pass it to Temporal Client calls so they are canceled on timeout. <!-- docs/develop/typescript/nexus/feature-guide.mdx:193-194 -->
- `ctx.requestDeadline` is an optional `Date` for the current request's deadline; use it to decide whether to start work or to set downstream timeouts. <!-- docs/develop/typescript/nexus/feature-guide.mdx:197-199 -->
- Keep Updates inside sync handlers short-lived to stay within the request deadline. <!-- docs/develop/typescript/nexus/feature-guide.mdx:195 -->

### Asynchronous (Workflow-run) handler

To back an Operation with a Workflow, use `new WorkflowRunOperationHandler<I, O>(fn)` from `@temporalio/nexus` and start the Workflow with `temporalNexus.startWorkflow(ctx, workflow, options)`. <!-- docs/develop/typescript/nexus/feature-guide.mdx:230,246,253 -->

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

<!-- Sources: docs/develop/typescript/nexus/feature-guide.mdx:238-265 -->

- The delegate `fn` receives `(ctx, input)` and is the place to validate/transform input and customize Workflow start options. <!-- docs/develop/typescript/nexus/feature-guide.mdx:247-249 -->
- `ctx.requestId` is allocated by Temporal when the caller schedules the Operation and is stable across retries — use it as the `workflowId` to dedupe starts. <!-- docs/develop/typescript/nexus/feature-guide.mdx:256-259 -->
- Workflow IDs should be business-meaningful and are used to dedupe Workflow starts; pass the ID through the Operation input as part of the Service contract when possible. <!-- docs/develop/typescript/nexus/feature-guide.mdx:269-270 -->
- The Task Queue defaults to the queue the Operation handler runs on — set it explicitly only when targeting a different Worker fleet. <!-- docs/develop/typescript/nexus/feature-guide.mdx:261 -->

#### Multi-argument Workflows

A Nexus Operation takes a single input, so wrap multiple Workflow arguments in one input object and spread them into the `args` array on `startWorkflow`. <!-- docs/develop/typescript/nexus/feature-guide.mdx:231-233 -->

```ts
return await temporalNexus.startWorkflow(ctx, myWorkflow, {
  args: [input.x, input.y],
  workflowId: ctx.requestId ?? randomUUID(),
});
```

## Register handlers on a Worker

Pass `nexusServices: [...]` to `Worker.create({ … })`. <!-- docs/develop/typescript/nexus/feature-guide.mdx:296 -->

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

<!-- Sources: docs/develop/typescript/nexus/feature-guide.mdx:285-297 -->

- A Worker only polls for Nexus tasks if at least one handler is in `nexusServices`. <!-- docs/develop/typescript/nexus/quickstart.mdx:158,161 -->
- Service handlers are typically hosted on the same Worker as the Workflows and Activities they wrap. <!-- docs/develop/typescript/nexus/feature-guide.mdx:145 -->

## Caller Workflow

Inside a Workflow, build a Nexus client with `createNexusServiceClient` from `@temporalio/workflow` and invoke `executeOperation(opName, input, options)`. <!-- docs/develop/typescript/nexus/feature-guide.mdx:303,317,322 -->

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

<!-- Sources: docs/develop/typescript/nexus/feature-guide.mdx:310-329 -->

- `createNexusServiceClient({ service, endpoint })` — the `service` is the imported Service contract and `endpoint` is the Nexus Endpoint name. <!-- docs/develop/typescript/nexus/feature-guide.mdx:317-319 -->
- `executeOperation` starts the Operation and awaits its result. <!-- docs/develop/typescript/nexus/quickstart.mdx:207 -->
- The caller depends only on the Service contract, not on handler code — keep the contract in a shared module. <!-- docs/develop/typescript/nexus/quickstart.mdx:202-204 -->

## Operation timeouts

Set timeouts in the third argument of `executeOperation`. The documented option in TypeScript is `scheduleToCloseTimeout`. <!-- docs/develop/typescript/nexus/feature-guide.mdx:325 --> <!-- docs/develop/typescript/nexus/quickstart.mdx:190 -->

```ts
await nexusClient.executeOperation('hello', input, { scheduleToCloseTimeout: '10s' });
```

<!-- VERIFY: TypeScript docs only show scheduleToCloseTimeout for Nexus Operations; scheduleToStartTimeout and startToCloseTimeout are not mentioned in the Nexus pages. -->

## Cancellation

Nexus Operations execute inside Cancellation Scopes provided by `@temporalio/workflow`. <!-- docs/develop/typescript/nexus/feature-guide.mdx:353 -->

- Cancelling a Cancellation Scope cancels every cancellable operation owned by it, including Nexus Operations. <!-- docs/develop/typescript/nexus/feature-guide.mdx:354 -->
- The Workflow itself is the root Cancellation Scope: cancelling the caller Workflow propagates cancellation to all Nexus Operations it started. <!-- docs/develop/typescript/nexus/feature-guide.mdx:355-356 -->
- For per-Operation control, create a new Cancellation Scope and start the Operation inside it. <!-- docs/develop/typescript/nexus/feature-guide.mdx:358 -->
- Only asynchronous Operations can be canceled — cancellation is delivered via the operation token. Synchronous Operations cannot be canceled. <!-- docs/develop/typescript/nexus/feature-guide.mdx:361 -->
- The Workflow or resource backing the Operation may ignore the cancellation request. <!-- docs/develop/typescript/nexus/feature-guide.mdx:362 -->
- Once the caller Workflow completes, the Nexus Machinery makes no further attempts to cancel still-running Operations — `await` pending Operations before returning if you need cancellation to be delivered. <!-- docs/develop/typescript/nexus/feature-guide.mdx:364-366 -->

<!-- VERIFY: The TypeScript Nexus docs do not document a CancellationType enum (ABANDON / TRY_CANCEL / WAIT_REQUESTED / WAIT_COMPLETED) the way the Python and Go docs do. Cancellation in TypeScript is expressed through Cancellation Scopes from @temporalio/workflow. -->

## Exceptions

TypeScript surfaces three Nexus-specific exception types. <!-- docs/develop/typescript/nexus/feature-guide.mdx:345 -->

- `OperationError` (from `nexus-rpc`) — throw inside a handler to signal application-level failure that should not be retried. <!-- docs/develop/typescript/nexus/feature-guide.mdx:347 -->
- `HandlerError` (from `nexus-rpc`) — throw with a `HandlerErrorType`. Non-retryable types: `BAD_REQUEST`, `UNAUTHENTICATED`, `UNAUTHORIZED`, `NOT_FOUND`, `NOT_IMPLEMENTED`. Retryable types: `RESOURCE_EXHAUSTED`, `INTERNAL`, `UNAVAILABLE`, `UPSTREAM_TIMEOUT`. <!-- docs/develop/typescript/nexus/feature-guide.mdx:348 -->
- `NexusOperationFailure` (from `@temporalio/nexus`) — thrown inside the caller Workflow when an Operation fails for any reason; inspect `cause` to walk the cause chain. <!-- docs/develop/typescript/nexus/feature-guide.mdx:349 -->

## Observability

Caller Workflow history for synchronous Operations contains `NexusOperationScheduled` and `NexusOperationCompleted`. For asynchronous Operations it also contains `NexusOperationStarted`. <!-- docs/develop/typescript/nexus/feature-guide.mdx:457-466 -->

For tracing, register `OpenTelemetryPlugin` from `@temporalio/interceptors-opentelemetry` via the Worker's `plugins` option — it auto-registers Nexus, Activity, and Workflow interceptors. <!-- docs/develop/typescript/nexus/feature-guide.mdx:476-493 -->

## Common mistakes

1. Forgetting `nexusServices: [...]` on `Worker.create` — the Worker silently never polls for Nexus tasks. <!-- docs/develop/typescript/nexus/quickstart.mdx:158,161 -->
2. Using a wrapper type for sync handlers — there is no wrapper; a sync handler is a plain `async (ctx, input) => …` property on the `serviceHandler` object. <!-- docs/develop/typescript/nexus/feature-guide.mdx:159,173 -->
3. Building the Temporal Client manually inside a sync handler — use `temporalNexus.getClient()` so the call uses the Worker's existing `NativeConnection`. <!-- docs/develop/typescript/nexus/feature-guide.mdx:154,217 -->
4. Ignoring `ctx.abortSignal` for downstream calls — long Client calls will run past the Nexus request deadline if you don't pass it through. <!-- docs/develop/typescript/nexus/feature-guide.mdx:193-194 -->
5. Putting long-running work in a sync handler — sync handlers must complete inside the Nexus request timeout; back long work with a Workflow via `WorkflowRunOperationHandler`. <!-- docs/develop/typescript/nexus/feature-guide.mdx:192,230 -->
6. Generating a random `workflowId` for every retry of a Workflow-run Operation — use `ctx.requestId ?? randomUUID()` so retries dedupe to the same Workflow. <!-- docs/develop/typescript/nexus/feature-guide.mdx:256-259 -->
7. Passing multiple positional Workflow arguments through a Nexus Operation — Operations take a single input; wrap arguments in one object and spread into `args: [...]`. <!-- docs/develop/typescript/nexus/feature-guide.mdx:231-233 -->
8. Expecting to cancel a synchronous Operation — only asynchronous Operations support cancellation. <!-- docs/develop/typescript/nexus/feature-guide.mdx:361 -->
9. Letting the caller Workflow return while Operations are still in flight — pending cancellations are not delivered once the caller completes; `await` outstanding Operations first. <!-- docs/develop/typescript/nexus/feature-guide.mdx:364-366 -->
10. Mismatching the Endpoint name between the caller's `createNexusServiceClient({ endpoint })` and the Endpoint created via `temporal operator nexus endpoint create --name` — calls fail to route. <!-- docs/develop/typescript/nexus/feature-guide.mdx:81-85,317-319 -->
