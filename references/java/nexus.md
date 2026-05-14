# Java SDK Nexus Reference

## Overview

Concepts (Endpoint, Service contract, Operation, caller, handler) are defined in `references/core/nexus.md`; this file documents the Java-specific identifiers, annotations, and builders. Java SDK Nexus support is Generally Available. <!-- docs/develop/java/nexus/feature-guide.mdx:20-25 -->

## Prerequisites

- Temporal CLI `v1.3.0` or higher. <!-- docs/develop/java/nexus/feature-guide.mdx:52-53 -->
- Temporal Java SDK `v1.28.0` or higher. <!-- docs/develop/java/nexus/feature-guide.mdx:54-55 -->
- Same `io.temporal:temporal-sdk` Gradle/Maven coordinate as the rest of the Java SDK. Nexus annotations live in `io.nexusrpc.*`; Nexus runtime utilities live in `io.temporal.nexus.*`. <!-- docs/develop/java/nexus/feature-guide.mdx:203 --> <!-- docs/develop/java/nexus/quickstart.mdx:58-59 -->

Default Data Converter encodes payloads as Null, Byte array, Protobuf JSON, then JSON; this reference uses Java classes serialized to JSON. <!-- docs/develop/java/nexus/feature-guide.mdx:101-104 -->

## Define the Service contract

A Service contract is a Java interface annotated with `@Service`. Each Operation is an interface method annotated with `@Operation` that takes exactly one input parameter and returns one output type. Input and output types should be JSON-serializable; use Jackson `@JsonCreator` / `@JsonProperty` on nested DTO classes. <!-- docs/develop/java/nexus/feature-guide.mdx:111-191 -->

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

<!-- Sources: docs/develop/java/nexus/feature-guide.mdx:111-191 -->

## Implement Operation handlers

The implementation class is annotated `@ServiceImpl(service = SampleNexusService.class)`. Each method annotated `@OperationImpl` is a **factory** that returns an `OperationHandler<Input, Output>` — it does not return the Operation's `Output` directly. <!-- docs/develop/java/nexus/feature-guide.mdx:224-242 -->

### Synchronous handler

Use `OperationHandler.sync` (lowercase `sync`) for simple RPC-style handlers. The lambda receives `(ctx, details, input)` and returns the output value directly. <!-- docs/develop/java/nexus/feature-guide.mdx:209-242 -->

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

<!-- Sources: docs/develop/java/nexus/feature-guide.mdx:227-242 -->

### Synchronous handler with Signals, Queries, or Updates

Inside a sync handler, obtain the Worker's Client via `Nexus.getOperationContext().getWorkflowClient()` and construct stubs for Signal / Query / Update calls. All calls must complete within the Nexus request timeout; Updates should be short-lived. <!-- docs/develop/java/nexus/feature-guide.mdx:205-272 -->

```java
import io.temporal.nexus.Nexus;

private GreetingWorkflow getWorkflowStub(String userId) {
  return Nexus.getOperationContext()
      .getWorkflowClient()
      .newWorkflowStub(GreetingWorkflow.class, getWorkflowId(userId));
}
```

<!-- Sources: docs/develop/java/nexus/feature-guide.mdx:255-272 -->

### Asynchronous (Workflow-run) handler

Use `WorkflowRunOperation.fromWorkflowMethod` to expose a Workflow as an async Operation when the Workflow takes one argument matching the Operation input. The lambda returns a method reference to the Workflow stub (`stub::method`). Default Task Queue is the queue this operation is handled on. Use `details.getRequestId()` for the Workflow Id when no business-meaningful Id is available — the request Id is stable across retries. <!-- docs/develop/java/nexus/feature-guide.mdx:279-320 -->

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

<!-- Sources: docs/develop/java/nexus/feature-guide.mdx:293-315 -->

### Multi-argument handler Workflows

When the handler Workflow takes multiple arguments, use `WorkflowRunOperation.fromWorkflowHandle` and `WorkflowHandle.fromWorkflowMethod(stub::method, arg1, arg2, ...)` to map the single Nexus input to multiple Workflow arguments. <!-- docs/develop/java/nexus/feature-guide.mdx:329-388 -->

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

<!-- Sources: docs/develop/java/nexus/feature-guide.mdx:361-387 -->

## Register handlers on the handler Worker

Register the Nexus Service implementation alongside any Workflow types it starts. <!-- docs/develop/java/nexus/feature-guide.mdx:417-422 -->

```java
import io.temporal.worker.Worker;
import io.temporal.worker.WorkerFactory;

Worker worker = factory.newWorker("my-handler-task-queue");
worker.registerWorkflowImplementationTypes(HelloHandlerWorkflowImpl.class);
worker.registerNexusServiceImplementation(new SampleNexusServiceImpl());
factory.start();
```

<!-- Sources: docs/develop/java/nexus/feature-guide.mdx:409-423 -->

## Caller Workflow

Inside a caller Workflow, create a Nexus Service stub via `Workflow.newNexusServiceStub(ServiceClass.class, NexusServiceOptions)`. Calling a method on the stub invokes the Operation and blocks until it completes. <!-- docs/develop/java/nexus/feature-guide.mdx:446-461 -->

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

<!-- Sources: docs/develop/java/nexus/feature-guide.mdx:446-462 -->

For access to the started state or final Promise, use `Workflow.startNexusOperation(stub::method, input)` which returns `NexusOperationHandle<Output>`. Call `handle.getExecution().get()` to wait until the Operation is started (the `NexusOperationExecution` carries the Operation token for async Operations), and `handle.getResult().get()` to wait for the final result. <!-- docs/develop/java/nexus/feature-guide.mdx:480-501 -->

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

<!-- Sources: docs/develop/java/nexus/feature-guide.mdx:493-501 -->

## Bind the Service to an Endpoint on the caller Worker

The Nexus stub cannot dispatch without an Endpoint binding. On the caller Worker, pass `WorkflowImplementationOptions` that map each Nexus Service class name to a `NexusServiceOptions` carrying the Endpoint name when registering Workflow types. <!-- docs/develop/java/nexus/feature-guide.mdx:525-547 -->

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

<!-- Sources: docs/develop/java/nexus/feature-guide.mdx:533-543 -->

The key in the map is the **Service class simple name** (e.g. `"SampleNexusService"`). <!-- docs/develop/java/nexus/feature-guide.mdx:537-539 -->

## Operation timeouts

Per-Operation timeouts are set on `NexusOperationOptions.Builder` and attached via `NexusServiceOptions.Builder.setOperationOptions(...)`. <!-- docs/develop/java/nexus/feature-guide.mdx:450-455 -->

- `setScheduleToCloseTimeout(Duration)` — total time the caller will wait for the Operation. <!-- docs/develop/java/nexus/feature-guide.mdx:453 -->

<!-- VERIFY: setScheduleToStartTimeout and setStartToCloseTimeout on NexusOperationOptions.Builder not shown in these docs files -->

## Cancellation

Cancellation of a Nexus Operation is scoped through `Workflow.newCancellationScope(Runnable)`. Any SDK methods (including Nexus Operations) started inside that `Runnable` are tied to the returned scope; calling `.cancel()` on the scope cancels them. The Promise returned by `Workflow.startNexusOperation` resolves when the Operation finishes — success, failure, timeout, or cancellation. <!-- docs/develop/java/nexus/feature-guide.mdx:642-648 -->

Only asynchronous Operations can be canceled (cancellation is delivered via the Operation token); the handler Workflow may ignore the cancellation request. <!-- docs/develop/java/nexus/feature-guide.mdx:649-651 -->

Cancellation types, set via `.setCancellationType(...)` on `NexusServiceOptions.Builder`: <!-- docs/develop/java/nexus/feature-guide.mdx:656-665 -->

- `ABANDON` — do not request cancellation. <!-- docs/develop/java/nexus/feature-guide.mdx:656 -->
- `TRY_CANCEL` — send the cancellation request and report cancellation immediately to the caller. <!-- docs/develop/java/nexus/feature-guide.mdx:657-659 -->
- `WAIT_REQUESTED` — wait for confirmation that the cancellation request was received, not for actual cancellation. <!-- docs/develop/java/nexus/feature-guide.mdx:660-661 -->
- `WAIT_COMPLETED` — wait for the Operation to complete (it may or may not complete as canceled). Default. <!-- docs/develop/java/nexus/feature-guide.mdx:662-664 -->

Once the caller Workflow completes, the caller stops attempting to cancel still-running Operations and lets them finish. To guarantee delivery, wait for pending cancellation requests before exiting the caller Workflow. <!-- docs/develop/java/nexus/feature-guide.mdx:667-671 -->

## Observability

Caller Workflow history events:

- Synchronous Operations: `NexusOperationScheduled`, `NexusOperationCompleted`. <!-- docs/develop/java/nexus/feature-guide.mdx:823-826 -->
- Asynchronous Operations: `NexusOperationScheduled`, `NexusOperationStarted`, `NexusOperationCompleted`. <!-- docs/develop/java/nexus/feature-guide.mdx:817-821 -->

`NexusOperationStarted` is not reported for synchronous Operations. <!-- docs/develop/java/nexus/feature-guide.mdx:828-832 -->

Inspect pending Nexus Operations and attached handler callbacks with `temporal workflow describe -w <ID>`; full event history with `temporal workflow show -w <ID>`. <!-- docs/develop/java/nexus/feature-guide.mdx:802-815 -->

## Reliability constraint

The Nexus circuit breaker trips after 5 consecutive retryable errors and blocks all Operations from the caller to that Endpoint, so handlers must be reliable. <!-- docs/develop/java/nexus/feature-guide.mdx:198-202 -->

## Common mistakes

1. Writing `OperationHandler.Sync(...)` (capital S) — the Java factory is `OperationHandler.sync(...)`. <!-- docs/develop/java/nexus/feature-guide.mdx:209-215 -->
2. Omitting `@ServiceImpl(service = MyService.class)` on the implementation class — the impl is not discoverable as a Nexus Service without it. <!-- docs/develop/java/nexus/feature-guide.mdx:227-228 -->
3. Returning the Operation output directly from an `@OperationImpl` method. The method is a factory; it must return `OperationHandler<Input, Output>`. <!-- docs/develop/java/nexus/feature-guide.mdx:224-242 -->
4. Calling `Workflow.newWorkflowStub(...)` inside an async handler to start the handler Workflow — start it through `WorkflowRunOperation.fromWorkflowMethod` or `WorkflowRunOperation.fromWorkflowHandle` so Nexus owns the Workflow start. <!-- docs/develop/java/nexus/feature-guide.mdx:279-388 -->
5. Forgetting to bind the Endpoint on the caller Worker via `WorkflowImplementationOptions.setNexusServiceOptions(...)` — the stub has no Endpoint and Operations cannot dispatch. <!-- docs/develop/java/nexus/feature-guide.mdx:533-543 -->
6. Confusing `@Operation` (on the Service interface method) with `@OperationImpl` (on the implementation factory method). <!-- docs/develop/java/nexus/feature-guide.mdx:185-189 --> <!-- docs/develop/java/nexus/feature-guide.mdx:229-230 -->
7. Using a non-stable Workflow Id for the handler Workflow. Use `details.getRequestId()` (stable across Operation retries) when no business-meaningful Id is available. <!-- docs/develop/java/nexus/feature-guide.mdx:306-319 -->
