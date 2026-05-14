# Go SDK Nexus Reference

## Overview

Temporal Go SDK support for Nexus is Generally Available. <!-- docs/develop/go/nexus/feature-guide.mdx:21-25 -->

This file documents Go SDK identifiers, APIs, and idioms for Nexus. Concept-level material (Endpoints, Services, Operations, sync vs. async, timeout semantics, error categories) lives in `references/core/nexus.md` and is not redefined here.

## Prerequisites

- Temporal CLI v1.3.0 or higher. <!-- docs/develop/go/nexus/feature-guide.mdx:50 -->
- Temporal Go SDK v1.33.0 or higher. <!-- docs/develop/go/nexus/feature-guide.mdx:51-52 -->
- Run the dev server with `temporal server start-dev` (Nexus enabled). <!-- docs/develop/go/nexus/feature-guide.mdx:56-58 -->

Go modules involved:

- `go.temporal.io/sdk/client` <!-- docs/develop/go/nexus/feature-guide.mdx:151 -->
- `go.temporal.io/sdk/temporalnexus` <!-- docs/develop/go/nexus/feature-guide.mdx:152 -->
- `go.temporal.io/sdk/worker` <!-- docs/develop/go/nexus/feature-guide.mdx:273 -->
- `go.temporal.io/sdk/workflow` <!-- docs/develop/go/nexus/feature-guide.mdx:153 -->
- `github.com/nexus-rpc/sdk-go/nexus` <!-- docs/develop/go/nexus/feature-guide.mdx:149 -->

Create a Nexus Endpoint to route requests from a caller Namespace to a handler Namespace and Task Queue: <!-- docs/develop/go/nexus/feature-guide.mdx:76-85 -->

```
temporal operator nexus endpoint create \
  --name my-nexus-endpoint-name \
  --target-namespace my-target-namespace \
  --target-task-queue my-handler-task-queue
```

## Define the Service contract

Go has no decorator-based Service contract. Use string constants for the Service and Operation names plus Go types for inputs and outputs, typically in a shared package imported by both caller and handler. <!-- docs/develop/go/nexus/feature-guide.mdx:89-117 -->

```go
package service

const HelloServiceName = "my-hello-service"
const EchoOperationName = "echo"

type EchoInput struct {
    Message string
}

type EchoOutput EchoInput
```

<!-- Sources: docs/develop/go/nexus/feature-guide.mdx:100-116 -->

## Implement Operation handlers

The `temporalnexus` package provides builders for Operation handlers. `NewWorkflowRunOperation` runs a Workflow as an asynchronous Nexus Operation. `GetClient` returns the Temporal Client that the Worker was initialized with for use in synchronous handlers. <!-- docs/develop/go/nexus/feature-guide.mdx:126-130 -->

### Synchronous handler

Use `nexus.NewSyncOperation` for simple RPC handlers. The handler receives `ctx context.Context`, the typed input, and `nexus.StartOperationOptions`, and returns the typed output and an error. <!-- docs/develop/go/nexus/feature-guide.mdx:136-167 -->

```go
import (
    "context"

    "github.com/nexus-rpc/sdk-go/nexus"
    "go.temporal.io/sdk/temporalnexus"
)

var EchoOperation = nexus.NewSyncOperation(
    service.EchoOperationName,
    func(ctx context.Context, input service.EchoInput, options nexus.StartOperationOptions) (service.EchoOutput, error) {
        return service.EchoOutput(input), nil
    },
)
```

<!-- Sources: docs/develop/go/nexus/feature-guide.mdx:159-164 -->

The handler `ctx` is set with the Nexus request deadline; pass it directly to Temporal Client calls to propagate the timeout. <!-- docs/develop/go/nexus/feature-guide.mdx:173-175 --> Handlers should be reliable since the circuit breaker trips after 5 consecutive retryable errors, blocking all Operations from the caller to that Endpoint. <!-- docs/develop/go/nexus/feature-guide.mdx:124 -->

### Synchronous handler that uses the Temporal Client

Call `temporalnexus.GetClient(ctx)` inside a sync handler to get the Worker's Temporal Client. Use it to Signal, Query, Update, Signal-With-Start, or Update-With-Start a Workflow. <!-- docs/develop/go/nexus/feature-guide.mdx:169-193 -->

```go
var GetLanguagesOperation = nexus.NewSyncOperation(
    service.GetLanguagesOperationName,
    func(ctx context.Context, input service.GetLanguagesInput, options nexus.StartOperationOptions) (service.GetLanguagesOutput, error) {
        c := temporalnexus.GetClient(ctx)
        workflowID := GetWorkflowID(input.UserID)
        // ... use c to signal/query/update ...
    },
)
```

<!-- Sources: docs/develop/go/nexus/feature-guide.mdx:189-193 -->

All calls inside a sync handler must complete within the Nexus request timeout. Updates should be short-lived to stay within this deadline. <!-- docs/develop/go/nexus/feature-guide.mdx:173-175 -->

### Asynchronous (Workflow-run) handler

Use `temporalnexus.NewWorkflowRunOperation(name, workflow, optionsFn)` to expose a Workflow as an asynchronous Operation. The `optionsFn` returns `client.StartWorkflowOptions`. Use `options.RequestID` as the Workflow ID for stability across retries of the Operation. <!-- docs/develop/go/nexus/feature-guide.mdx:198-218 -->

```go
import (
    "context"

    "github.com/nexus-rpc/sdk-go/nexus"
    "go.temporal.io/sdk/client"
    "go.temporal.io/sdk/temporalnexus"
)

var HelloOperation = temporalnexus.NewWorkflowRunOperation(
    service.HelloOperationName,
    HelloHandlerWorkflow,
    func(ctx context.Context, input service.HelloInput, options nexus.StartOperationOptions) (client.StartWorkflowOptions, error) {
        return client.StartWorkflowOptions{
            ID: options.RequestID,
        }, nil
    },
)
```

<!-- Sources: docs/develop/go/nexus/feature-guide.mdx:207-215 -->

If `TaskQueue` is omitted from `client.StartWorkflowOptions`, the handler Workflow runs on the same Task Queue as the Operation handler. <!-- docs/develop/go/nexus/feature-guide.mdx:213 -->

### Multi-argument handler Workflows

A Nexus Operation takes exactly one input. To start a Workflow that takes multiple arguments, use `temporalnexus.NewWorkflowRunOperationWithOptions` or `temporalnexus.MustNewWorkflowRunOperationWithOptions` with `temporalnexus.WorkflowRunOperationOptions[I, O]`, and call `temporalnexus.ExecuteUntypedWorkflow[O](ctx, options, startOpts, workflow, args…)` inside the handler. <!-- docs/develop/go/nexus/feature-guide.mdx:229-257 -->

```go
var HelloOperation = temporalnexus.MustNewWorkflowRunOperationWithOptions(
    temporalnexus.WorkflowRunOperationOptions[service.HelloInput, service.HelloOutput]{
        Name: service.HelloOperationName,
        Handler: func(ctx context.Context, input service.HelloInput, options nexus.StartOperationOptions) (temporalnexus.WorkflowHandle[service.HelloOutput], error) {
            return temporalnexus.ExecuteUntypedWorkflow[service.HelloOutput](
                ctx,
                options,
                client.StartWorkflowOptions{
                    ID: options.RequestID,
                },
                HelloHandlerWorkflow,
                input.Name,
                input.Language,
            )
        },
    },
)
```

<!-- Sources: docs/develop/go/nexus/feature-guide.mdx:237-255 -->

The generic type parameters `[I, O]` on `WorkflowRunOperationOptions` are the Operation input and output types, not the Workflow's argument types. <!-- docs/develop/go/nexus/feature-guide.mdx:237-239 -->

## Register the Nexus Service in a Worker

Build a `nexus.Service` with `nexus.NewService(name)`, register Operation handlers on it with `service.Register(ops…)`, then attach the Service to the Worker with `w.RegisterNexusService(service)`. Also register any handler Workflow with `w.RegisterWorkflow` so the same Worker can execute it. <!-- docs/develop/go/nexus/feature-guide.mdx:259-312 -->

```go
c, err := client.Dial(client.Options{ /* ... */ })
if err != nil {
    log.Fatalln("Unable to create client", err)
}
defer c.Close()

w := worker.New(c, "my-handler-task-queue", worker.Options{})

service := nexus.NewService(service.HelloServiceName)
err = service.Register(handler.EchoOperation, handler.HelloOperation)
if err != nil {
    log.Fatalln("Unable to register operations", err)
}
w.RegisterNexusService(service)
w.RegisterWorkflow(handler.HelloHandlerWorkflow)

if err := w.Run(worker.InterruptCh()); err != nil {
    log.Fatalln("Unable to start worker", err)
}
```

<!-- Sources: docs/develop/go/nexus/feature-guide.mdx:291-310 -->

A Worker will only poll for and process incoming Nexus requests if the Nexus Service Handlers are registered. <!-- docs/develop/go/nexus/quickstart.mdx:186 -->

## Caller Workflow

Inside a Workflow function, call `workflow.NewNexusClient(endpointName, serviceName)` to get a client, then `c.ExecuteOperation(ctx, operationName, input, workflow.NexusOperationOptions{})` to schedule the Operation. The return value is a future. <!-- docs/develop/go/nexus/feature-guide.mdx:318-367 -->

```go
package caller

import (
    "github.com/temporalio/samples-go/nexus/service"
    "go.temporal.io/sdk/workflow"
)

const (
    TaskQueue    = "my-caller-workflow-task-queue"
    endpointName = "my-nexus-endpoint-name"
)

func EchoCallerWorkflow(ctx workflow.Context, message string) (string, error) {
    c := workflow.NewNexusClient(endpointName, service.HelloServiceName)

    fut := c.ExecuteOperation(ctx, service.EchoOperationName, service.EchoInput{Message: message}, workflow.NexusOperationOptions{})

    var res service.EchoOutput
    if err := fut.Get(ctx, &res); err != nil {
        return "", err
    }
    return res.Message, nil
}
```

<!-- Sources: docs/develop/go/nexus/feature-guide.mdx:333-344 -->

`fut.Get(ctx, &result)` blocks the Workflow until the Operation completes and decodes the result. <!-- docs/develop/go/nexus/feature-guide.mdx:339-341 -->

### Wait for Operation start and read the token

`fut.GetNexusOperationExecution()` returns a secondary future that resolves once the Operation has started. It yields a `workflow.NexusOperationExecution` whose `OperationToken` field is the handle used for additional actions such as cancelation on asynchronous Operations. <!-- docs/develop/go/nexus/feature-guide.mdx:352-358 -->

```go
fut := c.ExecuteOperation(ctx, service.HelloOperationName, service.HelloInput{Name: name, Language: language}, workflow.NexusOperationOptions{})

var exec workflow.NexusOperationExecution
if err := fut.GetNexusOperationExecution().Get(ctx, &exec); err != nil {
    return "", err
}

var res service.HelloOutput
if err := fut.Get(ctx, &res); err != nil {
    return "", err
}
```

<!-- Sources: docs/develop/go/nexus/feature-guide.mdx:346-363 -->

Register the caller Workflow on its own Worker with `w.RegisterWorkflow(caller.EchoCallerWorkflow)`. The caller Worker runs in the caller Namespace; the handler Worker runs in the target Namespace. <!-- docs/develop/go/nexus/feature-guide.mdx:412-451 -->

## Operation timeouts

Set timeouts on `workflow.NexusOperationOptions` when calling `ExecuteOperation`. <!-- docs/develop/go/nexus/feature-guide.mdx:369-372 --> See `references/core/nexus.md` for the lifecycle semantics of each.

```go
fut := c.ExecuteOperation(ctx, opName, input, workflow.NexusOperationOptions{
    ScheduleToCloseTimeout: 10 * time.Minute,
    ScheduleToStartTimeout: 2 * time.Minute,
    StartToCloseTimeout:    5 * time.Minute,
})
```

- `ScheduleToCloseTimeout` limits the total duration from scheduling to completion. The Nexus Machinery automatically retries failed requests until this timeout is exceeded. <!-- docs/develop/go/nexus/feature-guide.mdx:374-383 -->
- `ScheduleToStartTimeout` limits how long the caller will wait for the Operation to be started by the handler; if not set, no Schedule-to-Start timeout is enforced. <!-- docs/develop/go/nexus/feature-guide.mdx:385-394 -->
- `StartToCloseTimeout` limits how long the caller will wait for an asynchronous Operation to complete after it has been started; only applies to asynchronous Operations; if not set, no Start-to-Close timeout is enforced. <!-- docs/develop/go/nexus/feature-guide.mdx:396-406 -->

## Cancellation

To cancel a Nexus Operation from within a Workflow, create a cancellable context with `workflow.WithCancel`. This returns a new context and a function that, when called, cancels the context and any SDK method that was passed it, including the Nexus Operation future. The future is resolved when the Operation finishes, whether it succeeds, fails, times out, or is canceled. <!-- docs/develop/go/nexus/feature-guide.mdx:560-562 -->

```go
cancelCtx, cancel := workflow.WithCancel(ctx)
fut := c.ExecuteOperation(cancelCtx, opName, input, workflow.NexusOperationOptions{})
// ... later ...
cancel()
```

Only asynchronous Operations can be canceled in Nexus, as cancelation is sent using an operation token. The Workflow or other resources backing the Operation may choose to ignore the cancelation request; if ignored, the Operation may enter a terminal state. <!-- docs/develop/go/nexus/feature-guide.mdx:564-566 -->

Once the caller Workflow completes, the caller's Nexus Machinery will not make any further attempts to cancel Operations that are still running. To ensure cancelations are delivered, wait for all pending Operations to finish before exiting the Workflow. <!-- docs/develop/go/nexus/feature-guide.mdx:568-570 -->

## Errors

`fut.Get(ctx, &result)` returns a non-nil error if the Operation fails, is canceled, or exceeds a configured timeout. See `references/core/nexus.md` for the Nexus error categories and retry semantics. The future returned by `NexusClient.ExecuteOperation` is resolved when the Operation finishes, whether it succeeds, fails, times out, or is canceled. <!-- docs/develop/go/nexus/feature-guide.mdx:562 -->

Sync handlers feeding the circuit breaker should be reliable: 5 consecutive retryable errors trip the breaker for that Endpoint. <!-- docs/develop/go/nexus/feature-guide.mdx:124 -->

## Observability

For asynchronous Nexus Operations the caller's history records `NexusOperationScheduled`, `NexusOperationStarted`, and `NexusOperationCompleted`. For synchronous Operations the caller's history records only `NexusOperationScheduled` and `NexusOperationCompleted` — `NexusOperationStarted` is not reported. <!-- docs/develop/go/nexus/feature-guide.mdx:755-770 -->

Use the Temporal CLI to inspect a caller Workflow: <!-- docs/develop/go/nexus/feature-guide.mdx:741-753 -->

```
temporal workflow describe -w <ID>
temporal workflow show -w <ID>
```

## Common mistakes

1. Calling a non-existent `NewAsyncOperation` instead of `temporalnexus.NewWorkflowRunOperation` for async (Workflow-run) Operations. <!-- docs/develop/go/nexus/feature-guide.mdx:200 -->
2. Building the `nexus.Service` and registering Operations but forgetting `w.RegisterNexusService(service)`, so the Worker does not poll Nexus tasks. <!-- docs/develop/go/nexus/quickstart.mdx:186 -->
3. Registering the Nexus Service on the handler Worker but forgetting `w.RegisterWorkflow(handler.HelloHandlerWorkflow)` for the Workflow that an async Operation starts. <!-- docs/develop/go/nexus/feature-guide.mdx:303-304 -->
4. Using `client.Dial` to open a new Client inside a sync handler instead of calling `temporalnexus.GetClient(ctx)` to reuse the Worker's existing Client. <!-- docs/develop/go/nexus/feature-guide.mdx:137 -->
5. Mismatching the generic parameters on `WorkflowRunOperationOptions[I, O]` — `I` and `O` are the Operation input and output types, and the handler must return `temporalnexus.WorkflowHandle[O]`. <!-- docs/develop/go/nexus/feature-guide.mdx:237-239 -->
6. Calling `c.ExecuteOperation` outside a Workflow function — `workflow.NewNexusClient` must be called with a Workflow `ctx`, and `ExecuteOperation` is a Workflow-side API. <!-- docs/develop/go/nexus/feature-guide.mdx:333-336 -->
7. Forgetting to pass `options.RequestID` (or another stable, business-meaningful ID) as the Workflow ID in the async handler's options function, defeating dedupe on Operation retries. <!-- docs/develop/go/nexus/feature-guide.mdx:209-212 -->
8. Letting the caller Workflow exit while async Operations are still running and expecting cancelations to be delivered — they will not be. <!-- docs/develop/go/nexus/feature-guide.mdx:568-570 -->
