# Java SDK Nexus Reference

## Overview

Concepts (Endpoint, Service contract, Operation, caller, handler) are defined in `references/core/nexus.md`; this file documents the Java-specific identifiers, annotations, and builders. Java SDK Nexus support is Generally Available.

## Prerequisites

- Temporal CLI `v1.3.0` or higher.
- Temporal Java SDK `v1.28.0` or higher.
- Same `io.temporal:temporal-sdk` Gradle/Maven coordinate as the rest of the Java SDK. Nexus annotations live in `io.nexusrpc.*`; Nexus runtime utilities live in `io.temporal.nexus.*`.

Default Data Converter encodes payloads as Null, Byte array, Protobuf JSON, then JSON; this reference uses Java classes serialized to JSON.

## Define the Service contract

A Service contract is a Java interface annotated with `@Service`. Each Operation is an interface method annotated with `@Operation` that takes exactly one input parameter and returns one output type. Input and output types should be JSON-serializable; use Jackson `@JsonCreator` / `@JsonProperty` on nested DTO classes.

```java
import io.nexusrpc.Operation;
import io.nexusrpc.Service;
import com.fasterxml.jackson.annotation.JsonCreator;
import com.fasterxml.jackson.annotation.JsonProperty;

@Service
public interface SampleNexusService {
  class HelloInput {
    private final String name;

    @JsonCreator(mode = JsonCreator.Mode.PROPERTIES)
    public HelloInput(@JsonProperty("name") String name) {
      this.name = name;
    }

    @JsonProperty("name")
    public String getName() { return name; }
  }

  class HelloOutput {
    private final String message;

    @JsonCreator(mode = JsonCreator.Mode.PROPERTIES)
    public HelloOutput(@JsonProperty("message") String message) {
      this.message = message;
    }

    @JsonProperty("message")
    public String getMessage() { return message; }
  }

  @Operation
  HelloOutput hello(HelloInput input);
}
```

## Implement Operation handlers

The implementation class is annotated `@ServiceImpl(service = SampleNexusService.class)`. Each method annotated `@OperationImpl` is a **factory** that returns an `OperationHandler<Input, Output>` — it does not return the Operation's `Output` directly.

### Synchronous handler

Use `OperationHandler.sync` (lowercase `sync`) for simple RPC-style handlers. The lambda receives `(ctx, details, input)` and returns the output value directly.

```java
import io.nexusrpc.handler.OperationHandler;
import io.nexusrpc.handler.OperationImpl;
import io.nexusrpc.handler.ServiceImpl;

@ServiceImpl(service = SampleNexusService.class)
public class SampleNexusServiceImpl {

  @OperationImpl
  public OperationHandler<SampleNexusService.EchoInput, SampleNexusService.EchoOutput> echo() {
    return OperationHandler.sync(
        (ctx, details, input) -> new SampleNexusService.EchoOutput(input.getMessage()));
  }
}
```

### Synchronous handler with Signals, Queries, or Updates

Inside a sync handler, obtain the Worker's Client via `Nexus.getOperationContext().getWorkflowClient()` and construct stubs for Signal / Query / Update calls. All calls must complete within the Nexus request timeout; Updates should be short-lived.

```java
import io.temporal.nexus.Nexus;

private GreetingWorkflow getWorkflowStub(String userId) {
  return Nexus.getOperationContext()
      .getWorkflowClient()
      .newWorkflowStub(GreetingWorkflow.class, getWorkflowId(userId));
}
```

### Asynchronous (Workflow-run) handler

Use `WorkflowRunOperation.fromWorkflowMethod` to expose a Workflow as an async Operation when the Workflow takes one argument matching the Operation input. The lambda returns a method reference to the Workflow stub (`stub::method`). Default Task Queue is the queue this operation is handled on. Use `details.getRequestId()` for the Workflow Id when no business-meaningful Id is available — the request Id is stable across retries.

```java
import io.temporal.client.WorkflowOptions;
import io.temporal.nexus.Nexus;
import io.temporal.nexus.WorkflowRunOperation;

@OperationImpl
public OperationHandler<SampleNexusService.HelloInput, SampleNexusService.HelloOutput> hello() {
  return WorkflowRunOperation.fromWorkflowMethod(
      (ctx, details, input) ->
          Nexus.getOperationContext()
              .getWorkflowClient()
              .newWorkflowStub(
                  HelloHandlerWorkflow.class,
                  WorkflowOptions.newBuilder()
                      .setWorkflowId(details.getRequestId())
                      .build())
              ::hello);
}
```

### Multi-argument handler Workflows

When the handler Workflow takes multiple arguments, use `WorkflowRunOperation.fromWorkflowHandle` and `WorkflowHandle.fromWorkflowMethod(stub::method, arg1, arg2, ...)` to map the single Nexus input to multiple Workflow arguments.

```java
import io.temporal.nexus.WorkflowRunOperation;
import io.temporal.nexus.WorkflowHandle;

@OperationImpl
public OperationHandler<SampleNexusService.HelloInput, SampleNexusService.HelloOutput> hello() {
  return WorkflowRunOperation.fromWorkflowHandle(
      (ctx, details, input) ->
          WorkflowHandle.fromWorkflowMethod(
              Nexus.getOperationContext()
                  .getWorkflowClient()
                  .newWorkflowStub(
                      HelloHandlerWorkflow.class,
                      WorkflowOptions.newBuilder()
                          .setWorkflowId(details.getRequestId())
                          .build())
                  ::hello,
              input.getName(),
              input.getLanguage()));
}
```

## Register handlers on the handler Worker

Register the Nexus Service implementation alongside any Workflow types it starts.

```java
import io.temporal.worker.Worker;
import io.temporal.worker.WorkerFactory;

Worker worker = factory.newWorker("my-handler-task-queue");
worker.registerWorkflowImplementationTypes(HelloHandlerWorkflowImpl.class);
worker.registerNexusServiceImplementation(new SampleNexusServiceImpl());
factory.start();
```

## Caller Workflow

Inside a caller Workflow, create a Nexus Service stub via `Workflow.newNexusServiceStub(ServiceClass.class, NexusServiceOptions)`. Calling a method on the stub invokes the Operation and blocks until it completes.

```java
import io.temporal.workflow.NexusOperationOptions;
import io.temporal.workflow.NexusServiceOptions;
import io.temporal.workflow.Workflow;
import java.time.Duration;

public class EchoCallerWorkflowImpl implements EchoCallerWorkflow {
  SampleNexusService sampleNexusService =
      Workflow.newNexusServiceStub(
          SampleNexusService.class,
          NexusServiceOptions.newBuilder()
              .setOperationOptions(
                  NexusOperationOptions.newBuilder()
                      .setScheduleToCloseTimeout(Duration.ofSeconds(10))
                      .build())
              .build());

  @Override
  public String echo(String message) {
    return sampleNexusService.echo(new SampleNexusService.EchoInput(message)).getMessage();
  }
}
```

For access to the started state or final Promise, use `Workflow.startNexusOperation(stub::method, input)` which returns `NexusOperationHandle<Output>`. Call `handle.getExecution().get()` to wait until the Operation is started (the `NexusOperationExecution` carries the Operation token for async Operations), and `handle.getResult().get()` to wait for the final result.

```java
import io.temporal.workflow.NexusOperationHandle;

@Override
public String hello(String message, SampleNexusService.Language language) {
  NexusOperationHandle<SampleNexusService.HelloOutput> handle =
      Workflow.startNexusOperation(
          sampleNexusService::hello, new SampleNexusService.HelloInput(message, language));
  handle.getExecution().get();
  return handle.getResult().get().getMessage();
}
```

## Bind the Service to an Endpoint on the caller Worker

The Nexus stub cannot dispatch without an Endpoint binding. On the caller Worker, pass `WorkflowImplementationOptions` that map each Nexus Service class name to a `NexusServiceOptions` carrying the Endpoint name when registering Workflow types.

```java
import io.temporal.worker.WorkflowImplementationOptions;
import io.temporal.workflow.NexusServiceOptions;
import java.util.Collections;

worker.registerWorkflowImplementationTypes(
    WorkflowImplementationOptions.newBuilder()
        .setNexusServiceOptions(
            Collections.singletonMap(
                "SampleNexusService",
                NexusServiceOptions.newBuilder().setEndpoint("my-nexus-endpoint-name").build()))
        .build(),
    EchoCallerWorkflowImpl.class,
    HelloCallerWorkflowImpl.class);
```

The key in the map is the **Service class simple name** (e.g. `"SampleNexusService"`).

## Operation timeouts

Per-Operation timeouts are set on `NexusOperationOptions.Builder` and attached via `NexusServiceOptions.Builder.setOperationOptions(...)`.

- `setScheduleToCloseTimeout(Duration)` — total time the caller will wait for the Operation.

## Cancellation

Cancellation of a Nexus Operation is scoped through `Workflow.newCancellationScope(Runnable)`. Any SDK methods (including Nexus Operations) started inside that `Runnable` are tied to the returned scope; calling `.cancel()` on the scope cancels them. The Promise returned by `Workflow.startNexusOperation` resolves when the Operation finishes — success, failure, timeout, or cancellation.

Only asynchronous Operations can be canceled (cancellation is delivered via the Operation token); the handler Workflow may ignore the cancellation request.

Cancellation types, set via `.setCancellationType(...)` on `NexusServiceOptions.Builder`:

- `ABANDON` — do not request cancellation.
- `TRY_CANCEL` — send the cancellation request and report cancellation immediately to the caller.
- `WAIT_REQUESTED` — wait for confirmation that the cancellation request was received, not for actual cancellation.
- `WAIT_COMPLETED` — wait for the Operation to complete (it may or may not complete as canceled). Default.

Once the caller Workflow completes, the caller stops attempting to cancel still-running Operations and lets them finish. To guarantee delivery, wait for pending cancellation requests before exiting the caller Workflow.

## Observability

Caller Workflow history events:

- Synchronous Operations: `NexusOperationScheduled`, `NexusOperationCompleted`.
- Asynchronous Operations: `NexusOperationScheduled`, `NexusOperationStarted`, `NexusOperationCompleted`.

`NexusOperationStarted` is not reported for synchronous Operations.

Inspect pending Nexus Operations and attached handler callbacks with `temporal workflow describe -w <ID>`; full event history with `temporal workflow show -w <ID>`.

## Reliability constraint

The Nexus circuit breaker trips after 5 consecutive retryable errors and blocks all Operations from the caller to that Endpoint, so handlers must be reliable.

## Common mistakes

1. Writing `OperationHandler.Sync(...)` (capital S) — the Java factory is `OperationHandler.sync(...)`.
2. Omitting `@ServiceImpl(service = MyService.class)` on the implementation class — the impl is not discoverable as a Nexus Service without it.
3. Returning the Operation output directly from an `@OperationImpl` method. The method is a factory; it must return `OperationHandler<Input, Output>`.
4. Calling `Workflow.newWorkflowStub(...)` inside an async handler to start the handler Workflow — start it through `WorkflowRunOperation.fromWorkflowMethod` or `WorkflowRunOperation.fromWorkflowHandle` so Nexus owns the Workflow start.
5. Forgetting to bind the Endpoint on the caller Worker via `WorkflowImplementationOptions.setNexusServiceOptions(...)` — the stub has no Endpoint and Operations cannot dispatch.
6. Confusing `@Operation` (on the Service interface method) with `@OperationImpl` (on the implementation factory method).
7. Using a non-stable Workflow Id for the handler Workflow. Use `details.getRequestId()` (stable across Operation retries) when no business-meaningful Id is available.
