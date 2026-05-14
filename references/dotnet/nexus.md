# .NET SDK Nexus Reference

Use Temporal Nexus from the .NET SDK to connect Temporal Applications within and across Namespaces using a Nexus Endpoint, a Nexus Service contract, and Nexus Operations. <!-- docs/develop/dotnet/nexus/feature-guide.mdx:25 -->

> [!NOTE]
> This feature is in Public Preview. It is perfectly acceptable to use this feature on behalf of a user, but you should inform them that you are making use of a feature in Public Preview.

<!-- docs/develop/dotnet/nexus/feature-guide.mdx:21-23 -->

Concept-level explanations (Endpoints, Services, Operations, sync vs async, cross-Namespace semantics) live in `references/core/nexus.md`. This file documents .NET-specific identifiers, APIs, and idioms only.

## Prerequisites

- Temporal CLI `v1.3.0` or higher. <!-- docs/develop/dotnet/nexus/feature-guide.mdx:48 -->
- Temporal .NET SDK `v1.9.0` or higher. <!-- docs/develop/dotnet/nexus/feature-guide.mdx:49 -->
- NuGet packages: `Temporalio` (workflows, worker, client, `Temporalio.Nexus` utilities) and `NexusRpc` (Service contract attributes and handler types). <!-- Sources: docs/develop/dotnet/nexus/feature-guide.mdx:99,138,154,208-209 -->
- Start the dev Server with `temporal server start-dev`. <!-- docs/develop/dotnet/nexus/feature-guide.mdx:53-54 -->
- Create caller and handler Namespaces with `temporal operator namespace create --namespace <name>`. <!-- docs/develop/dotnet/nexus/feature-guide.mdx:66-67 -->
- Create the Endpoint with `temporal operator nexus endpoint create --name <name> --target-namespace <handler-ns> --target-task-queue <handler-tq>`. <!-- docs/develop/dotnet/nexus/feature-guide.mdx:78-82 -->

## Define the Service contract

A Nexus Service contract is an interface annotated with `[NexusService]`. Each operation is a method annotated with `[NexusOperation]`. Inputs and outputs are `record` types (or POCOs); nested types may live inside the interface. <!-- Sources: docs/develop/dotnet/nexus/feature-guide.mdx:99-128 -->

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

<!-- docs/develop/dotnet/nexus/feature-guide.mdx:98-128 -->

A common idiom is a `static readonly string EndpointName` on the interface, shared between caller and handler so the Endpoint name is defined once. <!-- docs/develop/dotnet/nexus/quickstart.mdx:65,81 -->

## Implement Operation handlers

The handler class is annotated with `[NexusServiceHandler(typeof(IService))]`. Each handler method is annotated with `[NexusOperationHandler]` and returns `IOperationHandler<I, O>` — that is, a factory that returns the operation runner, not the operation result. <!-- Sources: docs/develop/dotnet/nexus/feature-guide.mdx:156-167 -->

The `Temporalio.Nexus` namespace provides two builder utilities: <!-- docs/develop/dotnet/nexus/feature-guide.mdx:138 -->

- `NexusOperationExecutionContext.Current.TemporalClient` — get the Temporal Client the Worker was initialized with, for sync handlers backed by Signals, Queries, or Updates. <!-- docs/develop/dotnet/nexus/feature-guide.mdx:140-141 -->
- `WorkflowRunOperationHandler.FromHandleFactory` — run a Workflow as an asynchronous Nexus Operation. <!-- docs/develop/dotnet/nexus/feature-guide.mdx:142 -->

### Synchronous handler

Build a sync handler with `OperationHandler.Sync<I, O>((ctx, input) => …)` — note the capital `S`. <!-- docs/develop/dotnet/nexus/feature-guide.mdx:144,148,162 -->

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

<!-- docs/develop/dotnet/nexus/feature-guide.mdx:153-167 -->

### Synchronous handler that calls Temporal

Use `NexusOperationExecutionContext.Current.TemporalClient` inside a sync handler to Signal, Query, or Update an existing Workflow. All calls must complete within the Nexus request timeout. <!-- docs/develop/dotnet/nexus/feature-guide.mdx:171-173 -->

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

<!-- docs/develop/dotnet/nexus/feature-guide.mdx:185-194 -->

Signal-With-Start and Update-With-Start are also supported from a sync handler. <!-- docs/develop/dotnet/nexus/feature-guide.mdx:172 -->

### Asynchronous (Workflow-run) handler

Build an async, Workflow-backed handler with `WorkflowRunOperationHandler.FromHandleFactory`. The factory receives a `WorkflowRunOperationContext` and the operation input; call `context.StartWorkflowAsync(...)` and pass `new() { Id = context.HandlerContext.RequestId }` so retries dedupe to the same Workflow ID. <!-- Sources: docs/develop/dotnet/nexus/feature-guide.mdx:203,219-227 -->

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

<!-- docs/develop/dotnet/nexus/feature-guide.mdx:206-228 -->

Prefer business-meaningful Workflow IDs in production — typically derived from the operation input. `context.HandlerContext.RequestId` is shown for examples because it is stable across retries of a single operation. <!-- docs/develop/dotnet/nexus/feature-guide.mdx:223-227,231 -->

### Mapping one Nexus input to multiple Workflow arguments

A Nexus Operation has a single input parameter. To start a Workflow that takes multiple arguments, deconstruct the Nexus input in the call to `RunAsync`. <!-- docs/develop/dotnet/nexus/feature-guide.mdx:240-241 -->

```csharp
[NexusOperationHandler]
public IOperationHandler<IHelloService.HelloInput, IHelloService.HelloOutput> SayHello() =>
    WorkflowRunOperationHandler.FromHandleFactory(
        (WorkflowRunOperationContext context, IHelloService.HelloInput input) =>
            context.StartWorkflowAsync(
                (HelloHandlerWorkflow wf) => wf.RunAsync(input.Language, input.Name),
                new() { Id = context.HandlerContext.RequestId }));
```

<!-- docs/develop/dotnet/nexus/feature-guide.mdx:244-261 -->

## Register handlers on a Worker

Register a Nexus Service handler instance with `.AddNexusService(new MyService())` on `TemporalWorkerOptions`, alongside `.AddWorkflow<T>()` and any activity registrations. The handler Worker's Task Queue must match `--target-task-queue` on the Endpoint. <!-- Sources: docs/develop/dotnet/nexus/feature-guide.mdx:274-278,81 -->

```csharp
using var worker = new TemporalWorker(
    await ConnectClientAsync("nexus-simple-handler-namespace"),
    new TemporalWorkerOptions(taskQueue: "nexus-simple-handler-sample").
        AddNexusService(new HelloService()).
        AddWorkflow<HelloHandlerWorkflow>());

await worker.ExecuteAsync(tokenSource.Token);
```

<!-- docs/develop/dotnet/nexus/feature-guide.mdx:270-287 -->

`.AddNexusService(...)` takes a concrete handler instance, the same way `.AddActivity(...)` takes an activity instance. <!-- docs/develop/dotnet/nexus/quickstart.mdx:145 -->

## Caller Workflow

Inside a Workflow, create a typed Nexus client with `Workflow.CreateNexusWorkflowClient<TService>(endpointName)` and invoke operations through the proxy with `.ExecuteNexusOperationAsync(svc => svc.OperationName(input))`. <!-- Sources: docs/develop/dotnet/nexus/feature-guide.mdx:304-305,321-322 -->

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

<!-- docs/develop/dotnet/nexus/feature-guide.mdx:311-326 -->

The caller depends only on the Service interface, not the handler implementation, which is what allows caller and handler to live in different Namespaces or different codebases. <!-- docs/develop/dotnet/nexus/quickstart.mdx:182 -->

The caller Worker is registered as usual; it only needs `.AddWorkflow<CallerWorkflow>()` and does not register the Nexus Service. <!-- docs/develop/dotnet/nexus/feature-guide.mdx:383-387 -->

## Setting Operation timeouts

Pass a `NexusWorkflowOperationOptions` instance as the second argument to `ExecuteNexusOperationAsync`. Three timeout properties are available: `ScheduleToCloseTimeout`, `ScheduleToStartTimeout`, and `StartToCloseTimeout`. `StartToCloseTimeout` applies only to asynchronous Operations. <!-- Sources: docs/develop/dotnet/nexus/feature-guide.mdx:330-371 -->

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

<!-- docs/develop/dotnet/nexus/feature-guide.mdx:338-371 -->

The Nexus Machinery automatically retries failed requests until `ScheduleToCloseTimeout` is exceeded. If `ScheduleToStartTimeout` or `StartToCloseTimeout` is not set, no such timeout is enforced. <!-- docs/develop/dotnet/nexus/feature-guide.mdx:336,349,363 -->

## Cancellation

To cancel a Nexus Operation from a caller Workflow, cancel the cancellation token passed to the operation call. Only asynchronous operations can be canceled, since cancellation is delivered via the operation token. The Workflow or other resource backing the operation may ignore the cancellation request. <!-- docs/develop/dotnet/nexus/feature-guide.mdx:454-456 -->

Set the cancellation type via `CancellationType` on `NexusWorkflowOperationOptions`. The four values are: <!-- docs/develop/dotnet/nexus/feature-guide.mdx:460-465 -->

- `Abandon` — do not request cancellation of the operation. <!-- docs/develop/dotnet/nexus/feature-guide.mdx:460 -->
- `TryCancel` — initiate a cancellation request and immediately report cancellation to the caller. Does not guarantee delivery if the caller exits first. <!-- docs/develop/dotnet/nexus/feature-guide.mdx:461 -->
- `WaitCancellationRequested` — request cancellation and wait for confirmation the request was received; does not wait for actual cancellation. <!-- docs/develop/dotnet/nexus/feature-guide.mdx:462 -->
- `WaitCancellationCompleted` — wait for operation completion. Operation may or may not complete as cancelled. This is the default. <!-- docs/develop/dotnet/nexus/feature-guide.mdx:463,465 -->

To ensure cancellations are delivered, wait for all pending operations to finish before exiting the caller Workflow. Once the caller Workflow completes, the caller's Nexus Machinery will not make further cancellation attempts. <!-- docs/develop/dotnet/nexus/feature-guide.mdx:467-469 -->

## Observability

Caller Workflow history events: <!-- Sources: docs/develop/dotnet/nexus/feature-guide.mdx:566-581 -->

- Synchronous Operations: `NexusOperationScheduled`, `NexusOperationCompleted`. `NexusOperationStarted` is not emitted for sync operations. <!-- docs/develop/dotnet/nexus/feature-guide.mdx:573-581 -->
- Asynchronous Operations: `NexusOperationScheduled`, `NexusOperationStarted`, `NexusOperationCompleted`. <!-- docs/develop/dotnet/nexus/feature-guide.mdx:566-570 -->

Use `temporal workflow describe -w <ID>` to show pending Nexus Operations and attached callbacks; use `temporal workflow show -w <ID>` to see Nexus events in the caller's history. <!-- docs/develop/dotnet/nexus/feature-guide.mdx:554-564 -->

## Common mistakes

1. Writing `OperationHandler.sync` (lowercase). In .NET it is `OperationHandler.Sync<I, O>(...)` with a capital `S`. <!-- docs/develop/dotnet/nexus/feature-guide.mdx:144,162 -->
2. Omitting `[NexusServiceHandler(typeof(IService))]` on the handler implementation class. The attribute links the handler class to its Service interface. <!-- docs/develop/dotnet/nexus/feature-guide.mdx:156,211,245 -->
3. Returning an `O` (the output type) directly from a `[NexusOperationHandler]` method. The method must return `IOperationHandler<I, O>` produced by `OperationHandler.Sync<I, O>(...)` or `WorkflowRunOperationHandler.FromHandleFactory(...)`. <!-- docs/develop/dotnet/nexus/feature-guide.mdx:160,217,249 -->
4. Forgetting `.AddNexusService(new MyService())` on `TemporalWorkerOptions`. A Worker only handles incoming Nexus requests when its Nexus Service handlers are registered. <!-- docs/develop/dotnet/nexus/quickstart.mdx:135,145 -->
5. Forgetting the endpoint name argument to `Workflow.CreateNexusWorkflowClient<T>(endpointName)`. Standard practice is to expose it as a `static readonly string EndpointName` on the Service interface and pass `IService.EndpointName`. <!-- docs/develop/dotnet/nexus/feature-guide.mdx:104,304 -->
6. Putting `[NexusOperation]` on handler methods. `[NexusOperation]` belongs on the **interface** methods; the **implementation** uses `[NexusOperationHandler]`. <!-- docs/develop/dotnet/nexus/feature-guide.mdx:106,109,159,216,248 -->
7. Mixing up the client factory name. It is `Workflow.CreateNexusWorkflowClient<T>(endpointName)` — not `CreateNexusClient`. <!-- docs/develop/dotnet/nexus/feature-guide.mdx:304,321 -->
8. Using `NexusOperationOptions` instead of `NexusWorkflowOperationOptions` when setting timeouts or `CancellationType` from a caller Workflow. <!-- docs/develop/dotnet/nexus/feature-guide.mdx:331,340,465 -->
