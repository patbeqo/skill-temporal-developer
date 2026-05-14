# .NET SDK Nexus Reference

Use Temporal Nexus from the .NET SDK to connect Temporal Applications within and across Namespaces using a Nexus Endpoint, a Nexus Service contract, and Nexus Operations.

> [!NOTE]
> This feature is in Public Preview. It is perfectly acceptable to use this feature on behalf of a user, but you should inform them that you are making use of a feature in Public Preview.

Concept-level explanations (Endpoints, Services, Operations, sync vs async, cross-Namespace semantics) live in `references/core/nexus.md`. This file documents .NET-specific identifiers, APIs, and idioms only.

## Prerequisites

- Temporal CLI `v1.3.0` or higher.
- Temporal .NET SDK `v1.9.0` or higher.
- NuGet packages: `Temporalio` (workflows, worker, client, `Temporalio.Nexus` utilities) and `NexusRpc` (Service contract attributes and handler types).
- Start the dev Server with `temporal server start-dev`.
- Create caller and handler Namespaces with `temporal operator namespace create --namespace <name>`.
- Create the Endpoint with `temporal operator nexus endpoint create --name <name> --target-namespace <handler-ns> --target-task-queue <handler-tq>`.

## Define the Service contract

A Nexus Service contract is an interface annotated with `[NexusService]`. Each operation is a method annotated with `[NexusOperation]`. Inputs and outputs are `record` types (or POCOs); nested types may live inside the interface.

```csharp
using NexusRpc;

[NexusService]
public interface IHelloService
{
    static readonly string EndpointName = "nexus-simple-endpoint";

    [NexusOperation]
    EchoOutput Echo(EchoInput input);

    [NexusOperation]
    HelloOutput SayHello(HelloInput input);

    public record EchoInput(string Message);

    public record EchoOutput(string Message);

    public record HelloInput(string Name, HelloLanguage Language);

    public record HelloOutput(string Message);

    public enum HelloLanguage { En, Fr, De, Es, Tr }
}
```

A common idiom is a `static readonly string EndpointName` on the interface, shared between caller and handler so the Endpoint name is defined once.

## Implement Operation handlers

The handler class is annotated with `[NexusServiceHandler(typeof(IService))]`. Each handler method is annotated with `[NexusOperationHandler]` and returns `IOperationHandler<I, O>` — that is, a factory that returns the operation runner, not the operation result.

The `Temporalio.Nexus` namespace provides two builder utilities:

- `NexusOperationExecutionContext.Current.TemporalClient` — get the Temporal Client the Worker was initialized with, for sync handlers backed by Signals, Queries, or Updates.
- `WorkflowRunOperationHandler.FromHandleFactory` — run a Workflow as an asynchronous Nexus Operation.

### Synchronous handler

Build a sync handler with `OperationHandler.Sync<I, O>((ctx, input) => …)` — note the capital `S`.

```csharp
using NexusRpc.Handlers;

[NexusServiceHandler(typeof(IHelloService))]
public class HelloService
{
    [NexusOperationHandler]
    public IOperationHandler<IHelloService.EchoInput, IHelloService.EchoOutput> Echo() =>
        OperationHandler.Sync<IHelloService.EchoInput, IHelloService.EchoOutput>(
            (ctx, input) => new(input.Message));
}
```

### Synchronous handler that calls Temporal

Use `NexusOperationExecutionContext.Current.TemporalClient` inside a sync handler to Signal, Query, or Update an existing Workflow. All calls must complete within the Nexus request timeout.

```csharp
[NexusOperationHandler]
public IOperationHandler<INexusGreetingService.GetLanguagesInput, INexusGreetingService.GetLanguagesOutput> GetLanguages() =>
    OperationHandler.Sync<INexusGreetingService.GetLanguagesInput, INexusGreetingService.GetLanguagesOutput>(
        async (ctx, input) =>
        {
            var client = NexusOperationExecutionContext.Current.TemporalClient;
            var handle = client.GetWorkflowHandle<GreetingWorkflow>(WorkflowIdForUser(input.UserId));
            return await handle.QueryAsync(wf => wf.QueryLanguages(input.IncludeUnsupported));
        });
```

Signal-With-Start and Update-With-Start are also supported from a sync handler.

### Asynchronous (Workflow-run) handler

Build an async, Workflow-backed handler with `WorkflowRunOperationHandler.FromHandleFactory`. The factory receives a `WorkflowRunOperationContext` and the operation input; call `context.StartWorkflowAsync(...)` and pass `new() { Id = context.HandlerContext.RequestId }` so retries dedupe to the same Workflow ID.

```csharp
using NexusRpc.Handlers;
using Temporalio.Nexus;

[NexusServiceHandler(typeof(IHelloService))]
public class HelloService
{
    [NexusOperationHandler]
    public IOperationHandler<IHelloService.HelloInput, IHelloService.HelloOutput> SayHello() =>
        WorkflowRunOperationHandler.FromHandleFactory(
            (WorkflowRunOperationContext context, IHelloService.HelloInput input) =>
                context.StartWorkflowAsync(
                    (HelloHandlerWorkflow wf) => wf.RunAsync(input),
                    new() { Id = context.HandlerContext.RequestId }));
}
```

Prefer business-meaningful Workflow IDs in production — typically derived from the operation input. `context.HandlerContext.RequestId` is shown for examples because it is stable across retries of a single operation.

### Mapping one Nexus input to multiple Workflow arguments

A Nexus Operation has a single input parameter. To start a Workflow that takes multiple arguments, deconstruct the Nexus input in the call to `RunAsync`.

```csharp
[NexusOperationHandler]
public IOperationHandler<IHelloService.HelloInput, IHelloService.HelloOutput> SayHello() =>
    WorkflowRunOperationHandler.FromHandleFactory(
        (WorkflowRunOperationContext context, IHelloService.HelloInput input) =>
            context.StartWorkflowAsync(
                (HelloHandlerWorkflow wf) => wf.RunAsync(input.Language, input.Name),
                new() { Id = context.HandlerContext.RequestId }));
```

## Register handlers on a Worker

Register a Nexus Service handler instance with `.AddNexusService(new MyService())` on `TemporalWorkerOptions`, alongside `.AddWorkflow<T>()` and any activity registrations. The handler Worker's Task Queue must match `--target-task-queue` on the Endpoint.

```csharp
using var worker = new TemporalWorker(
    await ConnectClientAsync("nexus-simple-handler-namespace"),
    new TemporalWorkerOptions(taskQueue: "nexus-simple-handler-sample").
        AddNexusService(new HelloService()).
        AddWorkflow<HelloHandlerWorkflow>());

await worker.ExecuteAsync(tokenSource.Token);
```

`.AddNexusService(...)` takes a concrete handler instance, the same way `.AddActivity(...)` takes an activity instance.

## Caller Workflow

Inside a Workflow, create a typed Nexus client with `Workflow.CreateNexusWorkflowClient<TService>(endpointName)` and invoke operations through the proxy with `.ExecuteNexusOperationAsync(svc => svc.OperationName(input))`.

```csharp
using Temporalio.Workflows;

[Workflow]
public class HelloCallerWorkflow
{
    [WorkflowRun]
    public async Task<string> RunAsync(string name, IHelloService.HelloLanguage language)
    {
        var output = await Workflow.CreateNexusWorkflowClient<IHelloService>(IHelloService.EndpointName).
            ExecuteNexusOperationAsync(svc => svc.SayHello(new(name, language)));
        return output.Message;
    }
}
```

The caller depends only on the Service interface, not the handler implementation, which is what allows caller and handler to live in different Namespaces or different codebases.

The caller Worker is registered as usual; it only needs `.AddWorkflow<CallerWorkflow>()` and does not register the Nexus Service.

## Setting Operation timeouts

Pass a `NexusWorkflowOperationOptions` instance as the second argument to `ExecuteNexusOperationAsync`. Three timeout properties are available: `ScheduleToCloseTimeout`, `ScheduleToStartTimeout`, and `StartToCloseTimeout`. `StartToCloseTimeout` applies only to asynchronous Operations.

```csharp
var output = await Workflow.CreateNexusWorkflowClient<IHelloService>(IHelloService.EndpointName).
    ExecuteNexusOperationAsync(
        svc => svc.SayHello(new(name, language)),
        new NexusWorkflowOperationOptions
        {
            ScheduleToCloseTimeout = TimeSpan.FromMinutes(10),
            ScheduleToStartTimeout = TimeSpan.FromMinutes(2),
            StartToCloseTimeout = TimeSpan.FromMinutes(5),
        });
```

The Nexus Machinery automatically retries failed requests until `ScheduleToCloseTimeout` is exceeded. If `ScheduleToStartTimeout` or `StartToCloseTimeout` is not set, no such timeout is enforced.

## Cancellation

To cancel a Nexus Operation from a caller Workflow, cancel the cancellation token passed to the operation call. Only asynchronous operations can be canceled, since cancellation is delivered via the operation token. The Workflow or other resource backing the operation may ignore the cancellation request.

Set the cancellation type via `CancellationType` on `NexusWorkflowOperationOptions`. The four values are:

- `Abandon` — do not request cancellation of the operation.
- `TryCancel` — initiate a cancellation request and immediately report cancellation to the caller. Does not guarantee delivery if the caller exits first.
- `WaitCancellationRequested` — request cancellation and wait for confirmation the request was received; does not wait for actual cancellation.
- `WaitCancellationCompleted` — wait for operation completion. Operation may or may not complete as cancelled. This is the default.

To ensure cancellations are delivered, wait for all pending operations to finish before exiting the caller Workflow. Once the caller Workflow completes, the caller's Nexus Machinery will not make further cancellation attempts.

## Observability

Caller Workflow history events:

- Synchronous Operations: `NexusOperationScheduled`, `NexusOperationCompleted`. `NexusOperationStarted` is not emitted for sync operations.
- Asynchronous Operations: `NexusOperationScheduled`, `NexusOperationStarted`, `NexusOperationCompleted`.

Use `temporal workflow describe -w <ID>` to show pending Nexus Operations and attached callbacks; use `temporal workflow show -w <ID>` to see Nexus events in the caller's history.

## Common mistakes

1. Writing `OperationHandler.sync` (lowercase). In .NET it is `OperationHandler.Sync<I, O>(...)` with a capital `S`.
2. Omitting `[NexusServiceHandler(typeof(IService))]` on the handler implementation class. The attribute links the handler class to its Service interface.
3. Returning an `O` (the output type) directly from a `[NexusOperationHandler]` method. The method must return `IOperationHandler<I, O>` produced by `OperationHandler.Sync<I, O>(...)` or `WorkflowRunOperationHandler.FromHandleFactory(...)`.
4. Forgetting `.AddNexusService(new MyService())` on `TemporalWorkerOptions`. A Worker only handles incoming Nexus requests when its Nexus Service handlers are registered.
5. Forgetting the endpoint name argument to `Workflow.CreateNexusWorkflowClient<T>(endpointName)`. Standard practice is to expose it as a `static readonly string EndpointName` on the Service interface and pass `IService.EndpointName`.
6. Putting `[NexusOperation]` on handler methods. `[NexusOperation]` belongs on the **interface** methods; the **implementation** uses `[NexusOperationHandler]`.
7. Mixing up the client factory name. It is `Workflow.CreateNexusWorkflowClient<T>(endpointName)` — not `CreateNexusClient`.
8. Using `NexusOperationOptions` instead of `NexusWorkflowOperationOptions` when setting timeouts or `CancellationType` from a caller Workflow.
