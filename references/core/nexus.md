# Temporal Nexus

This document is the cross-language conceptual reference for Temporal Nexus. After reading it, see `references/{your_language}/nexus.md` for SDK-specific APIs.

## Overview

Nexus connects Temporal Applications across (and within) isolated Namespaces through typed Service contracts and a managed reverse-proxy Endpoint. <!-- docs/encyclopedia/nexus/nexus.mdx:37-38 --> Each team owns its own Namespace for security and fault isolation and exposes only a stable contract via a Nexus Endpoint. <!-- docs/encyclopedia/nexus/nexus.mdx:38 --> Nexus is peer-to-peer, not hierarchical: caller and handler Workflows are siblings communicating across Namespace boundaries. <!-- docs/encyclopedia/nexus/nexus.mdx:42-43 --> The Nexus platform is Generally Available for Temporal Cloud and self-hosted deployments. <!-- docs/encyclopedia/nexus/nexus.mdx:21 -->

## When to use Nexus

- Cross-team or cross-Namespace orchestration where caller and handler are owned and deployed independently. <!-- docs/encyclopedia/nexus/nexus.mdx:31-33 -->
- Exposing reusable functionality behind a stable Service contract so callers do not depend on internal Workflow IDs, Signals, Queries, or Task Queues. <!-- docs/encyclopedia/nexus/nexus.mdx:56-57 -->
- Composing functionality across multiple Services and teams via multi-level calls (Workflow A -> Nexus Op -> Workflow B -> Nexus Op -> Workflow C). <!-- docs/encyclopedia/nexus/nexus.mdx:114-118 -->
- Connecting Namespaces across regions or clouds without requiring direct connectivity or shared configuration. <!-- docs/encyclopedia/nexus/nexus.mdx:118 -->

## Core vocabulary

- **Nexus Service**: A named collection of Nexus Operations exposed as a contract for sharing across team boundaries. <!-- docs/encyclopedia/nexus/nexus-services.mdx:28 --> Multiple Services can run in the same Worker. <!-- docs/encyclopedia/nexus/nexus-services.mdx:32 -->
- **Nexus Operation**: A unit of work within a Service; can be synchronous or asynchronous, with an Operation token used to re-attach to long-running asynchronous Operations. <!-- docs/encyclopedia/nexus/nexus-operations.mdx:40 -->
- **Nexus Endpoint**: A fully managed reverse proxy that routes requests from a caller Workflow to a single target Namespace and Task Queue. <!-- docs/encyclopedia/nexus/nexus-endpoints.mdx:25-27 --> Callers only know the Endpoint name; the target Namespace, Task Queue, and implementation are encapsulated. <!-- docs/encyclopedia/nexus/nexus-endpoints.mdx:27 -->
- **Nexus Registry**: The catalog that manages Endpoints; in Temporal Cloud it is global across an Account, in self-hosted deployments it is scoped to a Cluster. <!-- docs/encyclopedia/nexus/nexus-registry.mdx:30-35 -->
- **Nexus Machinery**: The built-in delivery machinery that handles at-least-once execution, automatic retries, rate limiting, concurrency limiting, circuit breaking, and load balancing. <!-- docs/encyclopedia/nexus/nexus.mdx:100-109 -->
- **Nexus Task**: The task type handler Workers poll from the Endpoint's target Task Queue to process Nexus Operation requests. <!-- docs/encyclopedia/nexus/nexus.mdx:86-87 -->

## Operation lifecycle modes

Operations are defined using SDK builder functions: **New-Workflow-Run-Operation** for asynchronous Operations (starts a Workflow) and **New-Sync-Operation** for synchronous Operations (invokes a Query/Signal/Update or runs other reliable code via the SDK Client). <!-- docs/encyclopedia/nexus/nexus-operations.mdx:57-60 -->

### Synchronous

Synchronous Operations must complete within the 10-second handler deadline, measured from the caller's Nexus Machinery. <!-- docs/encyclopedia/nexus/nexus-operations.mdx:75 --> They complete as part of the start request, so they do **not** have a `NexusOperationStarted` event in the caller's history. <!-- docs/encyclopedia/nexus/nexus-operations.mdx:197 --> Canonical caller-side event sequence:

1. `ScheduleNexusOperation` command issued by the caller Worker. <!-- docs/encyclopedia/nexus/nexus-operations.mdx:85 -->
2. `NexusOperationScheduled` event recorded. <!-- docs/encyclopedia/nexus/nexus-operations.mdx:86 -->
3. Handler processes the request via New-Sync-Operation and responds with the result. <!-- docs/encyclopedia/nexus/nexus-operations.mdx:90-91 -->
4. `NexusOperationCompleted` or `NexusOperationFailed` event recorded. <!-- docs/encyclopedia/nexus/nexus-operations.mdx:92 -->

For longer work, use New-Workflow-Run-Operation. <!-- docs/encyclopedia/nexus/nexus-operations.mdx:144 -->

### Asynchronous

Asynchronous Operations start a Workflow and can run up to 60 days (the maximum Schedule-to-Close in Temporal Cloud). <!-- docs/encyclopedia/nexus/nexus-operations.mdx:110 --> Canonical caller-side event sequence:

1. `ScheduleNexusOperation` command issued by the caller Worker. <!-- docs/encyclopedia/nexus/nexus-operations.mdx:119 -->
2. `NexusOperationScheduled` event recorded. <!-- docs/encyclopedia/nexus/nexus-operations.mdx:120 -->
3. Handler processes the request via New-Workflow-Run-Operation and responds with the start Operation response. <!-- docs/encyclopedia/nexus/nexus-operations.mdx:124-125 -->
4. `NexusOperationStarted` event recorded. <!-- docs/encyclopedia/nexus/nexus-operations.mdx:126 -->
5. Handler Workflow completes and a Nexus completion Callback is delivered to the caller's Nexus Machinery. <!-- docs/encyclopedia/nexus/nexus-operations.mdx:127 -->
6. `NexusOperationCompleted` or `NexusOperationFailed` event recorded. <!-- docs/encyclopedia/nexus/nexus-operations.mdx:128 -->

Terminal events on the caller side are one of: `NexusOperationStarted`, `NexusOperationCompleted`, `NexusOperationFailed`, `NexusOperationCanceled`, or `NexusOperationTimedOut`. <!-- docs/encyclopedia/nexus/nexus-operations.mdx:169 -->

## The three timeouts

Set timeouts on the caller when scheduling the Operation. <!-- docs/encyclopedia/nexus/nexus-operations.mdx:188-189 -->

- **Schedule-to-Close**: Total end-to-end cap from schedule to completion. The Nexus Machinery automatically retries internally until this timeout expires, at which point the Operation fails with a `NexusOperationTimedOut` event. <!-- docs/encyclopedia/nexus/nexus-operations.mdx:193-195 --> Maximum in Temporal Cloud is 60 days. <!-- docs/encyclopedia/nexus/nexus-operations.mdx:199 --><!-- docs/evaluate/temporal-cloud/limits.mdx:298 -->
- **Schedule-to-Start**: How long the caller will wait for the Operation to be started (or completed, for sync). Fails with `TIMEOUT_TYPE_SCHEDULE_TO_START`. No enforcement if zero/unset. Requires Temporal Server 1.31.0 or later. <!-- docs/encyclopedia/nexus/nexus-operations.mdx:203-211 -->
- **Start-to-Close**: How long the caller will wait after an asynchronous Operation has started. Fails with `TIMEOUT_TYPE_START_TO_CLOSE`. **Applies only to asynchronous Operations; synchronous Operations ignore this timeout.** No enforcement if zero/unset. Requires Temporal Server 1.31.0 or later. <!-- docs/encyclopedia/nexus/nexus-operations.mdx:216-227 -->

## Automatic retries and circuit breaking

The Nexus Machinery retries on retryable Nexus errors and upstream timeouts up to the default Retry Policy's max attempts and expiration interval, until Schedule-to-Start or Schedule-to-Close is exceeded. <!-- docs/encyclopedia/nexus/nexus-operations.mdx:172-176 --> To stop retries, the handler returns a non-retryable Nexus error. <!-- docs/encyclopedia/nexus/nexus-operations.mdx:183 -->

Circuit breaking is per caller-Namespace/Endpoint destination pair; each pair trips and resets independently. <!-- docs/encyclopedia/nexus/nexus-operations.mdx:232-233 --> The breaker trips by default after **5 consecutive retryable errors**, opens for **60 seconds**, then transitions to half-open and allows a single probe request; success returns it to closed, failure reopens for another 60 seconds. <!-- docs/encyclopedia/nexus/nexus-operations.mdx:234-239 --> Consecutive request timeouts (e.g., no Workers polling the handler Task Queue) count as retryable errors and trip the breaker. <!-- docs/encyclopedia/nexus/nexus-operations.mdx:242-245 --> Different Operations within the same destination pair share the trip count, so a single Operation may have fewer than 5 attempts when the breaker opens. <!-- docs/encyclopedia/nexus/nexus-operations.mdx:264-265 -->

Circuit breaker state surfaces in Pending Nexus Operations and Pending Callbacks; when open, pending Operations show `State: Blocked` with a `BlockedReason: The circuit breaker is open.` <!-- docs/encyclopedia/nexus/nexus-operations.mdx:254-283 -->

## Execution semantics and idempotency

The Nexus Machinery provides **at-least-once** execution: handlers may be invoked multiple times for the same Operation until Schedule-to-Close expires. <!-- docs/encyclopedia/nexus/nexus-operations.mdx:316-317 --> Handlers should be idempotent (highly recommended, similar to Activities). <!-- docs/encyclopedia/nexus/nexus-operations.mdx:319-320 --> To upgrade to exactly-once, back the Operation with a Workflow that uses a `WorkflowIDReusePolicy` of `RejectDuplicates`, which permits only one Execution per Workflow ID within a Namespace for the Retention Period. <!-- docs/encyclopedia/nexus/nexus-operations.mdx:322-325 -->

## Cancellation vs termination

- **Cancellation**: Cancelling a caller Workflow automatically propagates to all pending Nexus Operations and their underlying handler Workflows; a canceled handler Workflow reports a Canceled Failure to the caller. <!-- docs/encyclopedia/nexus/nexus-operations.mdx:329-330 -->
- **Termination**: Terminating a caller Workflow **abandons** all pending Nexus Operations; no cancel request is sent to the handler Namespace, so handler Workflows keep running until they time out or are manually stopped. <!-- docs/encyclopedia/nexus/nexus-operations.mdx:334-336 --> Termination also prevents compensation logic from running. <!-- docs/encyclopedia/nexus/nexus-operations.mdx:337-338 --> **Prefer cancellation over termination.** <!-- docs/encyclopedia/nexus/nexus-operations.mdx:339 -->

## Attaching multiple callers to a handler Workflow

Operations started with New-Workflow-Run-Operation automatically attach a completion Callback to the handler Workflow. <!-- docs/encyclopedia/nexus/nexus-operations.mdx:350 --> Additional callers can attach to the same handler Workflow using a Workflow-ID-Conflict-Policy of `Use-Existing`. <!-- docs/encyclopedia/nexus/nexus-operations.mdx:351 --> Each handler Workflow has a per-Workflow Callback limit (2000 total Callbacks per Workflow Execution in Temporal Cloud); callers that exceed the limit receive an error. <!-- docs/encyclopedia/nexus/nexus-operations.mdx:354-355 --><!-- docs/evaluate/temporal-cloud/limits.mdx:273-275 --> A single Workflow Execution can have a maximum of 30 in-flight Nexus Operations. <!-- docs/evaluate/temporal-cloud/limits.mdx:281 --> When a handler Workflow uses Continue-As-New, existing completion Callbacks are copied to the new Execution; the previous Execution's Callbacks remain in `Standby` state indefinitely. <!-- docs/encyclopedia/nexus/nexus-operations.mdx:357-358 -->

## Errors

By default, handler errors are retryable unless they are one of the following: <!-- docs/encyclopedia/nexus/nexus-error-handling.mdx:30 -->

- Application Failures explicitly marked as non-retryable. <!-- docs/encyclopedia/nexus/nexus-error-handling.mdx:32 -->
- Nexus Operation errors that resolve the Operation as failed or canceled. <!-- docs/encyclopedia/nexus/nexus-error-handling.mdx:33 -->
- Non-retryable Nexus errors. <!-- docs/encyclopedia/nexus/nexus-error-handling.mdx:34 -->

When the caller's Nexus Machinery receives an error: <!-- docs/encyclopedia/nexus/nexus-error-handling.mdx:36 -->

- **Non-retryable** -> `NexusOperationFailed` event is added to the caller's history. <!-- docs/encyclopedia/nexus/nexus-error-handling.mdx:38 -->
- **Retryable** -> automatically retried; surfaces in Pending Operations. <!-- docs/encyclopedia/nexus/nexus-error-handling.mdx:39 -->

Caller-side error shape: a Nexus Operation Failure containing the operation name, token, and failure reason; the `cause` field indicates the type (for example, Application Error or Canceled Error). <!-- docs/encyclopedia/nexus/nexus-error-handling.mdx:50-51 -->

Observed handler error category strings in the encyclopedia include `INTERNAL` and `UPSTREAM_TIMEOUT`, surfaced as `handler error (CATEGORY): message` with `applicationFailureInfo.type: "NexusHandlerError"`. <!-- docs/encyclopedia/nexus/nexus-operations.mdx:281 --><!-- docs/encyclopedia/nexus/nexus-operations.mdx:308 --><!-- docs/encyclopedia/nexus/nexus-execution-debugging.mdx:69 -->

<!-- VERIFY: Encyclopedia files do not enumerate the full set of handler error categories (BAD_REQUEST, UNAUTHENTICATED, UNAUTHORIZED, NOT_FOUND, NOT_IMPLEMENTED, RESOURCE_EXHAUSTED, INTERNAL, UNAVAILABLE, UPSTREAM_TIMEOUT). Per-language files should confirm the complete list against the failures reference. -->

## Deployment patterns

Two deployment patterns: <!-- docs/encyclopedia/nexus/nexus-patterns.mdx:26-29 -->

- **Collocated (default)**: Operation handlers run in the same Worker and on the same Task Queue as the underlying Workflows; the Endpoint targets that Task Queue. <!-- docs/encyclopedia/nexus/nexus-patterns.mdx:33-37 --> Supports Eager Workflow Start when the handler starts a Workflow in the same Worker, executing the first Workflow Task locally while still recording durable state. <!-- docs/encyclopedia/nexus/nexus-patterns.mdx:42 --> Use by default. <!-- docs/encyclopedia/nexus/nexus-patterns.mdx:27 -->
- **Router-queue**: A dedicated Nexus Worker polls a "router" Task Queue and starts Workflows on different Task Queues in the same Namespace. <!-- docs/encyclopedia/nexus/nexus-patterns.mdx:56 --> Use when you need independent scaling of Nexus routing from Workflow execution, different IAM permissions per Worker fleet, or to add Nexus without modifying existing Workers. <!-- docs/encyclopedia/nexus/nexus-patterns.mdx:60-63 -->

## Endpoints and Registry

- One Endpoint targets one Namespace plus one Task Queue; the supported `EndpointSpec` target type is `Worker`. <!-- docs/encyclopedia/nexus/nexus-endpoints.mdx:35-40 --> Endpoints are **not** general-purpose proxies and do not route to multiple backends. <!-- docs/encyclopedia/nexus/nexus-endpoints.mdx:34-37 -->
- Multiple Endpoints can target different Task Queues in the same Namespace. <!-- docs/encyclopedia/nexus/nexus-endpoints.mdx:30 -->
- Endpoint names must be unique within the Registry. <!-- docs/encyclopedia/nexus/nexus-registry.mdx:33 --> Adding an Endpoint deploys it immediately for runtime use. <!-- docs/encyclopedia/nexus/nexus-endpoints.mdx:44 --><!-- docs/encyclopedia/nexus/nexus-registry.mdx:32 -->
- Access is **deny by default**: the Access Policy is an explicit allowlist of caller Namespaces, and no callers are allowed by default even if in the same Namespace as the target. <!-- docs/encyclopedia/nexus/nexus-registry.mdx:66-67 --><!-- docs/encyclopedia/nexus/nexus-security.mdx:33 -->
- Everything except the Endpoint name can be edited; new Operations route to the updated target immediately. <!-- docs/encyclopedia/nexus/nexus-registry.mdx:76-77 --> Changing the target Namespace is permitted but: in-flight async completion callbacks still point to the original handler Namespace, Cancel requests route to the new target, and Workflow ID uniqueness is per-Namespace (Signal-With-Start can create duplicates in the new target). **Drain existing Operations before changing the target Namespace.** <!-- docs/encyclopedia/nexus/nexus-registry.mdx:79-85 -->
- The Registry is global across the Account in Temporal Cloud, Cluster-scoped self-hosted. <!-- docs/encyclopedia/nexus/nexus-registry.mdx:34-35 -->
- Manage via the Temporal UI, CLI, Terraform provider, or Cloud Ops API; the Operator API is available for self-hosted. <!-- docs/encyclopedia/nexus/nexus-registry.mdx:39 --><!-- docs/encyclopedia/nexus/nexus-registry.mdx:127-130 -->

### RBAC

In Temporal Cloud the Registry enforces RBAC: viewing/searching Endpoints requires the Read-only role (or higher) at the Account level; managing Endpoints requires the Developer role (or higher) **plus** Namespace Admin on the target Namespace. <!-- docs/encyclopedia/nexus/nexus-registry.mdx:114-117 --> Self-hosted deployments can implement a custom Authorizer plugin. <!-- docs/encyclopedia/nexus/nexus-registry.mdx:108 -->

## Security and payload encryption

- Temporal Cloud has built-in mTLS for all cross-Namespace Nexus traffic (start, cancel, and completion callbacks) across cells and regions; self-hosted relies on Cluster security. <!-- docs/encyclopedia/nexus/nexus-security.mdx:48-59 -->
- Workers authenticate to their Namespace using mTLS or API key. <!-- docs/encyclopedia/nexus/nexus-security.mdx:40 --><!-- docs/encyclopedia/nexus/nexus-security.mdx:58 -->
- On each Operation, Temporal Cloud verifies the caller's Namespace is in the Endpoint's allowlist before routing the request. <!-- docs/encyclopedia/nexus/nexus-security.mdx:40-42 -->
- Endpoints are only accessible from within a Temporal Cloud Account through the Temporal SDK and are not externally accessible. <!-- docs/encyclopedia/nexus/nexus-security.mdx:60 -->
- Nexus uses the **same Data Converter** as Workflows and Activities. A Codec used for encryption also encrypts Nexus payloads. Caller and handler Workers must have compatible Data Converters. The sender encrypts: the caller encrypts the input, the handler encrypts the result. <!-- docs/encyclopedia/nexus/nexus-security.mdx:64-68 -->

Three approaches for cross-Namespace payload encryption: <!-- docs/encyclopedia/nexus/nexus-security.mdx:70 -->

| Approach | When to pick |
|---|---|
| Same encryption key on both Namespaces. <!-- docs/encyclopedia/nexus/nexus-security.mdx:72-75 --> | Simplest; no additional configuration. <!-- docs/encyclopedia/nexus/nexus-security.mdx:74 --> |
| Per-Namespace key with the KMS key ID in payload metadata. <!-- docs/encyclopedia/nexus/nexus-security.mdx:77-83 --> | Each Namespace keeps its own key; the Codec Server needs KMS decrypt permissions for all relevant keys. <!-- docs/encyclopedia/nexus/nexus-security.mdx:83 --> |
| Wrapper types (for example, `EndpointValue`) for endpoint-specific encryption keys. <!-- docs/encyclopedia/nexus/nexus-security.mdx:87-90 --> | Teams that do not want to share Namespace encryption keys across teams. <!-- docs/encyclopedia/nexus/nexus-security.mdx:96-97 --> |

Options 1 and 2 work with the standard Data Converter; option 3 is advanced. <!-- docs/encyclopedia/nexus/nexus-security.mdx:96-97 -->

## Observability

- `temporal workflow describe` surfaces **Pending Nexus Operations** with fields including `Endpoint`, `Service`, `Operation`, `OperationToken`, `State`, `Attempt`, `ScheduleToCloseTimeout`, `NextAttemptScheduleTime`, `LastAttemptCompleteTime`, `LastAttemptFailure`, and `BlockedReason`. <!-- docs/encyclopedia/nexus/nexus-execution-debugging.mdx:55-70 --><!-- docs/encyclopedia/nexus/nexus-operations.mdx:274-283 -->
- Cancellation requests on async Operations surface the same pattern with `CancelationState`, `CancelationAttempt`, `CancelationRequestedTime`, `CancelationLastAttemptCompleteTime`, `CancelationLastAttemptFailure`, and `CancelationBlockedReason`. <!-- docs/encyclopedia/nexus/nexus-operations.mdx:285-309 -->
- `temporal workflow describe` also lists **Pending Callbacks** (the async completion callbacks sent from handler Namespace to caller Namespace) with `URL`, `Trigger`, `State`, `Attempt`, and `RegistrationTime`. <!-- docs/encyclopedia/nexus/nexus-execution-debugging.mdx:75-100 -->
- **Bi-directional links** automatically connect caller Nexus Operation events to the corresponding handler Workflow events (and back), wired by SDK builder functions like New-Workflow-Run-Operation. <!-- docs/encyclopedia/nexus/nexus-execution-debugging.mdx:28-39 -->
- Tracing integrates with OpenTelemetry / OpenTracing via an interceptor on the Client or Worker; per-SDK samples exist. <!-- docs/encyclopedia/nexus/nexus-execution-debugging.mdx:103-111 -->
- Metrics are available at three layers: SDK metrics from the Nexus Worker (including `nexus_poll_no_task`, `nexus_task_schedule_to_start_latency`, `nexus_task_execution_failed`, `nexus_task_execution_latency`, `nexus_task_endtoend_latency`), Temporal Cloud metrics (`RespondWorkflowTaskCompleted`, `PollNexusTaskQueue`, `RespondNexusTaskCompleted`, `RespondNexusTaskFailed`), and OSS Cluster metrics (History Service, Concurrency Limiter, Frontend Service). <!-- docs/encyclopedia/nexus/nexus-metrics.mdx:29-56 -->

## Limits (Temporal Cloud)

- Nexus requests count toward the Namespace RPS limit on both caller and target Namespaces. <!-- docs/evaluate/temporal-cloud/limits.mdx:126-128 --><!-- docs/evaluate/temporal-cloud/limits.mdx:134 -->
- 100 Endpoints per Account by default (can be raised via support ticket). <!-- docs/evaluate/temporal-cloud/limits.mdx:203-204 -->
- 30 in-flight Nexus Operations per Workflow Execution. <!-- docs/evaluate/temporal-cloud/limits.mdx:281 -->
- 2000 total Callbacks per Workflow Execution (governs how many Nexus callers can attach to a handler Workflow). <!-- docs/evaluate/temporal-cloud/limits.mdx:273-275 -->
- **Less than 10 seconds** maximum for a handler to process a single Nexus start or cancel request. <!-- docs/evaluate/temporal-cloud/limits.mdx:287 --> Available handler time is often shorter because the deadline is measured from the calling History Service and the request must transit matching. <!-- docs/evaluate/temporal-cloud/limits.mdx:289-290 --> On timeout, the handler receives a context-deadline-exceeded error and the caller retries with exponential backoff until Schedule-to-Close. <!-- docs/evaluate/temporal-cloud/limits.mdx:293-294 -->
- **60-day** maximum Schedule-to-Close for any Nexus Operation; the caller may configure shorter but the server caps at 60 days. <!-- docs/evaluate/temporal-cloud/limits.mdx:298 --><!-- docs/evaluate/temporal-cloud/limits.mdx:303 -->

## CLI surfaces

Use the following groups; the orchestrator's `skill-temporal-cli` covers each subcommand in depth.

- `temporal operator nexus endpoint ...` for self-hosted deployments. <!-- docs/encyclopedia/nexus/nexus-registry.mdx:45 -->
- `tcld nexus endpoint ...` for Temporal Cloud. <!-- docs/encyclopedia/nexus/nexus-registry.mdx:44 -->
- `temporal workflow describe` surfaces Pending Nexus Operations and Pending Callbacks. <!-- docs/encyclopedia/nexus/nexus-execution-debugging.mdx:43 --><!-- docs/encyclopedia/nexus/nexus-execution-debugging.mdx:78 -->

<!-- VERIFY: The exhaustive `endpoint` subcommand set (create | update | delete | list | get) and the `tcld`-only `--allow-namespace` flag are not enumerated in the encyclopedia. Confirm against `cli/operator#nexus` and `cloud/tcld/nexus` reference pages. -->

## Versioning

Task Routing is the simplest way to version Nexus Service code; for backward-incompatible changes, use a different Service name and Task Queue (for example, `prod.payments.v2`) and let callers migrate on their own schedule. <!-- docs/encyclopedia/nexus/nexus-operations.mdx:343-345 -->

## Per-language references

For SDK-specific APIs, types, and code samples, see:

- `references/python/nexus.md`
- `references/typescript/nexus.md`
- `references/go/nexus.md`
- `references/java/nexus.md`
- `references/dotnet/nexus.md`
