# Temporal Nexus

This document is the cross-language conceptual reference for Temporal Nexus. After reading it, see `references/{your_language}/nexus.md` for SDK-specific APIs.

## Overview

Nexus connects Temporal Applications across (and within) isolated Namespaces through typed Service contracts and a managed reverse-proxy Endpoint.  Each team owns its own Namespace for security and fault isolation and exposes only a stable contract via a Nexus Endpoint.  Nexus is peer-to-peer, not hierarchical: caller and handler Workflows are siblings communicating across Namespace boundaries.  The Nexus platform is Generally Available for Temporal Cloud and self-hosted deployments.

## When to use Nexus

- Cross-team or cross-Namespace orchestration where caller and handler are owned and deployed independently.
- Exposing reusable functionality behind a stable Service contract so callers do not depend on internal Workflow IDs, Signals, Queries, or Task Queues.
- Composing functionality across multiple Services and teams via multi-level calls (Workflow A -> Nexus Op -> Workflow B -> Nexus Op -> Workflow C).
- Connecting Namespaces across regions or clouds without requiring direct connectivity or shared configuration.

## Core vocabulary

- **Nexus Service**: A named collection of Nexus Operations exposed as a contract for sharing across team boundaries.  Multiple Services can run in the same Worker.
- **Nexus Operation**: A unit of work within a Service; can be synchronous or asynchronous, with an Operation token used to re-attach to long-running asynchronous Operations.
- **Nexus Endpoint**: A fully managed reverse proxy that routes requests from a caller Workflow to a single target Namespace and Task Queue.  Callers only know the Endpoint name; the target Namespace, Task Queue, and implementation are encapsulated.
- **Nexus Registry**: The catalog that manages Endpoints; in Temporal Cloud it is global across an Account, in self-hosted deployments it is scoped to a Cluster.
- **Nexus Machinery**: The built-in delivery machinery that handles at-least-once execution, automatic retries, rate limiting, concurrency limiting, circuit breaking, and load balancing.
- **Nexus Task**: The task type handler Workers poll from the Endpoint's target Task Queue to process Nexus Operation requests.

## Operation lifecycle modes

Operations are defined using SDK builder functions: **New-Workflow-Run-Operation** for asynchronous Operations (starts a Workflow) and **New-Sync-Operation** for synchronous Operations (invokes a Query/Signal/Update or runs other reliable code via the SDK Client).

### Synchronous

Synchronous Operations must complete within the 10-second handler deadline, measured from the caller's Nexus Machinery.  They complete as part of the start request, so they do **not** have a `NexusOperationStarted` event in the caller's history.  Canonical caller-side event sequence:

1. `ScheduleNexusOperation` command issued by the caller Worker.
2. `NexusOperationScheduled` event recorded.
3. Handler processes the request via New-Sync-Operation and responds with the result.
4. `NexusOperationCompleted` or `NexusOperationFailed` event recorded.

For longer work, use New-Workflow-Run-Operation.

### Asynchronous

Asynchronous Operations start a Workflow and can run up to 60 days (the maximum Schedule-to-Close in Temporal Cloud).  Canonical caller-side event sequence:

1. `ScheduleNexusOperation` command issued by the caller Worker.
2. `NexusOperationScheduled` event recorded.
3. Handler processes the request via New-Workflow-Run-Operation and responds with the start Operation response.
4. `NexusOperationStarted` event recorded.
5. Handler Workflow completes and a Nexus completion Callback is delivered to the caller's Nexus Machinery.
6. `NexusOperationCompleted` or `NexusOperationFailed` event recorded.

Terminal events on the caller side are one of: `NexusOperationStarted`, `NexusOperationCompleted`, `NexusOperationFailed`, `NexusOperationCanceled`, or `NexusOperationTimedOut`.

## The three timeouts

Set timeouts on the caller when scheduling the Operation.

- **Schedule-to-Close**: Total end-to-end cap from schedule to completion. The Nexus Machinery automatically retries internally until this timeout expires, at which point the Operation fails with a `NexusOperationTimedOut` event.  Maximum in Temporal Cloud is 60 days.
- **Schedule-to-Start**: How long the caller will wait for the Operation to be started (or completed, for sync). Fails with `TIMEOUT_TYPE_SCHEDULE_TO_START`. No enforcement if zero/unset. Requires Temporal Server 1.31.0 or later.
- **Start-to-Close**: How long the caller will wait after an asynchronous Operation has started. Fails with `TIMEOUT_TYPE_START_TO_CLOSE`. **Applies only to asynchronous Operations; synchronous Operations ignore this timeout.** No enforcement if zero/unset. Requires Temporal Server 1.31.0 or later.

## Automatic retries and circuit breaking

The Nexus Machinery retries on retryable Nexus errors and upstream timeouts up to the default Retry Policy's max attempts and expiration interval, until Schedule-to-Start or Schedule-to-Close is exceeded.  To stop retries, the handler returns a non-retryable Nexus error.

Circuit breaking is per caller-Namespace/Endpoint destination pair; each pair trips and resets independently.  The breaker trips by default after **5 consecutive retryable errors**, opens for **60 seconds**, then transitions to half-open and allows a single probe request; success returns it to closed, failure reopens for another 60 seconds.  Consecutive request timeouts (e.g., no Workers polling the handler Task Queue) count as retryable errors and trip the breaker.  Different Operations within the same destination pair share the trip count, so a single Operation may have fewer than 5 attempts when the breaker opens.

Circuit breaker state surfaces in Pending Nexus Operations and Pending Callbacks; when open, pending Operations show `State: Blocked` with a `BlockedReason: The circuit breaker is open.`

## Execution semantics and idempotency

The Nexus Machinery provides **at-least-once** execution: handlers may be invoked multiple times for the same Operation until Schedule-to-Close expires.  Handlers should be idempotent (highly recommended, similar to Activities).  To upgrade to exactly-once, back the Operation with a Workflow that uses a `WorkflowIDReusePolicy` of `RejectDuplicates`, which permits only one Execution per Workflow ID within a Namespace for the Retention Period.

## Cancellation vs termination

- **Cancellation**: Cancelling a caller Workflow automatically propagates to all pending Nexus Operations and their underlying handler Workflows; a canceled handler Workflow reports a Canceled Failure to the caller.
- **Termination**: Terminating a caller Workflow **abandons** all pending Nexus Operations; no cancel request is sent to the handler Namespace, so handler Workflows keep running until they time out or are manually stopped.  Termination also prevents compensation logic from running.  **Prefer cancellation over termination.**

## Attaching multiple callers to a handler Workflow

Operations started with New-Workflow-Run-Operation automatically attach a completion Callback to the handler Workflow.  Additional callers can attach to the same handler Workflow using a Workflow-ID-Conflict-Policy of `Use-Existing`.  Each handler Workflow has a per-Workflow Callback limit (2000 total Callbacks per Workflow Execution in Temporal Cloud); callers that exceed the limit receive an error.  A single Workflow Execution can have a maximum of 30 in-flight Nexus Operations.  When a handler Workflow uses Continue-As-New, existing completion Callbacks are copied to the new Execution; the previous Execution's Callbacks remain in `Standby` state indefinitely.

## Errors

By default, handler errors are retryable unless they are one of the following:

- Application Failures explicitly marked as non-retryable.
- Nexus Operation errors that resolve the Operation as failed or canceled.
- Non-retryable Nexus errors.

When the caller's Nexus Machinery receives an error:

- **Non-retryable** -> `NexusOperationFailed` event is added to the caller's history.
- **Retryable** -> automatically retried; surfaces in Pending Operations.

Caller-side error shape: a Nexus Operation Failure containing the operation name, token, and failure reason; the `cause` field indicates the type (for example, Application Error or Canceled Error).

Observed handler error category strings in the encyclopedia include `INTERNAL` and `UPSTREAM_TIMEOUT`, surfaced as `handler error (CATEGORY): message` with `applicationFailureInfo.type: "NexusHandlerError"`.

## Deployment patterns

Two deployment patterns:

- **Collocated (default)**: Operation handlers run in the same Worker and on the same Task Queue as the underlying Workflows; the Endpoint targets that Task Queue.  Supports Eager Workflow Start when the handler starts a Workflow in the same Worker, executing the first Workflow Task locally while still recording durable state.  Use by default.
- **Router-queue**: A dedicated Nexus Worker polls a "router" Task Queue and starts Workflows on different Task Queues in the same Namespace.  Use when you need independent scaling of Nexus routing from Workflow execution, different IAM permissions per Worker fleet, or to add Nexus without modifying existing Workers.

## Endpoints and Registry

- One Endpoint targets one Namespace plus one Task Queue; the supported `EndpointSpec` target type is `Worker`.  Endpoints are **not** general-purpose proxies and do not route to multiple backends.
- Multiple Endpoints can target different Task Queues in the same Namespace.
- Endpoint names must be unique within the Registry.  Adding an Endpoint deploys it immediately for runtime use.
- Access is **deny by default**: the Access Policy is an explicit allowlist of caller Namespaces, and no callers are allowed by default even if in the same Namespace as the target.
- Everything except the Endpoint name can be edited; new Operations route to the updated target immediately.  Changing the target Namespace is permitted but: in-flight async completion callbacks still point to the original handler Namespace, Cancel requests route to the new target, and Workflow ID uniqueness is per-Namespace (Signal-With-Start can create duplicates in the new target). **Drain existing Operations before changing the target Namespace.**
- The Registry is global across the Account in Temporal Cloud, Cluster-scoped self-hosted.
- Manage via the Temporal UI, CLI, Terraform provider, or Cloud Ops API; the Operator API is available for self-hosted.

### RBAC

In Temporal Cloud the Registry enforces RBAC: viewing/searching Endpoints requires the Read-only role (or higher) at the Account level; managing Endpoints requires the Developer role (or higher) **plus** Namespace Admin on the target Namespace.  Self-hosted deployments can implement a custom Authorizer plugin.

## Security and payload encryption

- Temporal Cloud has built-in mTLS for all cross-Namespace Nexus traffic (start, cancel, and completion callbacks) across cells and regions; self-hosted relies on Cluster security.
- Workers authenticate to their Namespace using mTLS or API key.
- On each Operation, Temporal Cloud verifies the caller's Namespace is in the Endpoint's allowlist before routing the request.
- Endpoints are only accessible from within a Temporal Cloud Account through the Temporal SDK and are not externally accessible.
- Nexus uses the **same Data Converter** as Workflows and Activities. A Codec used for encryption also encrypts Nexus payloads. Caller and handler Workers must have compatible Data Converters. The sender encrypts: the caller encrypts the input, the handler encrypts the result.

Three approaches for cross-Namespace payload encryption:

| Approach | When to pick |
|---|---|
| Same encryption key on both Namespaces.  | Simplest; no additional configuration.  |
| Per-Namespace key with the KMS key ID in payload metadata.  | Each Namespace keeps its own key; the Codec Server needs KMS decrypt permissions for all relevant keys.  |
| Wrapper types (for example, `EndpointValue`) for endpoint-specific encryption keys.  | Teams that do not want to share Namespace encryption keys across teams.  |

Options 1 and 2 work with the standard Data Converter; option 3 is advanced.

## Observability

- `temporal workflow describe` surfaces **Pending Nexus Operations** with fields including `Endpoint`, `Service`, `Operation`, `OperationToken`, `State`, `Attempt`, `ScheduleToCloseTimeout`, `NextAttemptScheduleTime`, `LastAttemptCompleteTime`, `LastAttemptFailure`, and `BlockedReason`.
- Cancellation requests on async Operations surface the same pattern with `CancelationState`, `CancelationAttempt`, `CancelationRequestedTime`, `CancelationLastAttemptCompleteTime`, `CancelationLastAttemptFailure`, and `CancelationBlockedReason`.
- `temporal workflow describe` also lists **Pending Callbacks** (the async completion callbacks sent from handler Namespace to caller Namespace) with `URL`, `Trigger`, `State`, `Attempt`, and `RegistrationTime`.
- **Bi-directional links** automatically connect caller Nexus Operation events to the corresponding handler Workflow events (and back), wired by SDK builder functions like New-Workflow-Run-Operation.
- Tracing integrates with OpenTelemetry / OpenTracing via an interceptor on the Client or Worker; per-SDK samples exist.
- Metrics are available at three layers: SDK metrics from the Nexus Worker (including `nexus_poll_no_task`, `nexus_task_schedule_to_start_latency`, `nexus_task_execution_failed`, `nexus_task_execution_latency`, `nexus_task_endtoend_latency`), Temporal Cloud metrics (`RespondWorkflowTaskCompleted`, `PollNexusTaskQueue`, `RespondNexusTaskCompleted`, `RespondNexusTaskFailed`), and OSS Cluster metrics (History Service, Concurrency Limiter, Frontend Service).

## Limits (Temporal Cloud)

- Nexus requests count toward the Namespace RPS limit on both caller and target Namespaces.
- 100 Endpoints per Account by default (can be raised via support ticket).
- 30 in-flight Nexus Operations per Workflow Execution.
- 2000 total Callbacks per Workflow Execution (governs how many Nexus callers can attach to a handler Workflow).
- **Less than 10 seconds** maximum for a handler to process a single Nexus start or cancel request.  Available handler time is often shorter because the deadline is measured from the calling History Service and the request must transit matching.  On timeout, the handler receives a context-deadline-exceeded error and the caller retries with exponential backoff until Schedule-to-Close.
- **60-day** maximum Schedule-to-Close for any Nexus Operation; the caller may configure shorter but the server caps at 60 days.

## CLI surfaces

Use the following groups; the orchestrator's `skill-temporal-cli` covers each subcommand in depth.

- `temporal operator nexus endpoint ...` for self-hosted deployments.
- `tcld nexus endpoint ...` for Temporal Cloud.
- `temporal workflow describe` surfaces Pending Nexus Operations and Pending Callbacks.

## Versioning

Task Routing is the simplest way to version Nexus Service code; for backward-incompatible changes, use a different Service name and Task Queue (for example, `prod.payments.v2`) and let callers migrate on their own schedule.

## Per-language references

For SDK-specific APIs, types, and code samples, see:

- `references/python/nexus.md`
- `references/typescript/nexus.md`
- `references/go/nexus.md`
- `references/java/nexus.md`
- `references/dotnet/nexus.md`
