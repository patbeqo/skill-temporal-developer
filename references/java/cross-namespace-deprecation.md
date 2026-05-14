# Cross-Namespace Child Workflow APIs: Deprecation (Java)

## Summary

Cross-Namespace commands (parent-child, SignalExternal, RequestCancelExternal) are gated server-side by the `system.enableCrossNamespaceCommands` configuration, and that configuration is disabled on Temporal Cloud.
Code using cross-Namespace calls must be updated or removed prior to migration to Temporal Cloud.
A Child Workflow Execution is spawned from within another Workflow in the same Namespace.
The supported alternative for cross-Namespace orchestration is Temporal Nexus, which has GA Java SDK support.

## What this affects

- `Workflow.newChildWorkflowStub(...)` — spawns a Child Workflow in the same Namespace as the parent; do not attempt to target another Namespace.
- `Workflow.newUntypedChildWorkflowStub("WorkflowType", ChildWorkflowOptions...)` — same-Namespace only.
- `Workflow.newExternalWorkflowStub(OtherWorkflow.class, otherWorkflowID)` — used for sending Signals (and request-cancel) to another running Workflow Execution; cross-Namespace use is gated by `system.enableCrossNamespaceCommands`.
- `Workflow.newUntypedExternalWorkflowStub("OtherWFId")` — same constraint applies.
- Any target-Namespace selector that a Java SDK external/child workflow stub exposes is unusable on Temporal Cloud while `system.enableCrossNamespaceCommands` is disabled.

The documented Java shapes for external workflow stubs are only:

```java
// Typed external workflow stub (same-Namespace Signal/Cancel target)
OtherWorkflow other = Workflow.newExternalWorkflowStub(OtherWorkflow.class, otherWorkflowID);
other.mySignalMethod();
```

```java
// Untyped external workflow stub by Workflow Id
ExternalWorkflowStub callOtherWorkflow = Workflow.newUntypedExternalWorkflowStub("OtherWFId");
```

Do not introduce a third "target Namespace" argument to these calls — no such overload is documented in the Java SDK docs.

## What to do instead

Use Temporal Nexus from Java to connect Temporal Applications within and across Namespaces using a Nexus Endpoint, a Nexus Service contract, and Nexus Operations.
Java SDK support for Nexus is Generally Available.

On Temporal Cloud, built-in Endpoint access controls and secure connectivity exist across Namespaces.
Each Endpoint has an access control policy (allowlist of caller Namespaces); Temporal Cloud verifies the caller's Namespace is in the Endpoint's allowlist before routing the request to the handler.

A Nexus Endpoint routes requests from a caller Namespace to a handler Namespace, for example:

```
temporal operator nexus endpoint create \
  --name my-nexus-endpoint-name \
  --target-namespace my-target-namespace \
  --target-task-queue my-handler-task-queue
```

See the Java Nexus feature guide for the full workflow: defining the Service contract, implementing Operation handlers, and developing a caller Workflow.

## Quick checks

- Don't spawn a Child Workflow targeting another Namespace; do spawn it in the same Namespace as the parent.
- Don't rely on `Workflow.newExternalWorkflowStub` / `Workflow.newUntypedExternalWorkflowStub` to reach a Workflow in a different Namespace on Temporal Cloud; do model the cross-Namespace call as a Nexus Operation.
- Don't invent a target-Namespace overload on external-workflow or child-workflow stub factory methods; the only documented shapes are the ones listed above.
- Don't issue SignalExternal or RequestCancelExternal across Namespaces; do use Nexus Operations whose handler Workflow in the target Namespace performs the Signal/Cancel locally.
- Don't assume cross-Namespace calls will work on Temporal Cloud — `system.enableCrossNamespaceCommands` is disabled there.

## References

- `docs/cloud/migrate/automated.mdx` (lines 419-422) — `system.enableCrossNamespaceCommands` is disabled on Temporal Cloud; cross-Namespace code must be updated or removed prior to migration.
- `docs/encyclopedia/child-workflows/child-workflows.mdx` (line 31) — Child Workflows are spawned in the same Namespace.
- `docs/develop/java/workflows/child-workflows.mdx` — Java Child Workflow API surface (`Workflow.newChildWorkflowStub`, `Workflow.newUntypedChildWorkflowStub`, `ChildWorkflowOptions.newBuilder()`).
- `docs/develop/java/workflows/message-passing.mdx` (lines 287-293, 463) — `Workflow.newExternalWorkflowStub`, `Workflow.newUntypedExternalWorkflowStub`.
- `docs/develop/java/client/temporal-client.mdx` (line 739) — additional `Workflow.newUntypedExternalWorkflowStub("OtherWFId")` shape.
- `docs/develop/java/nexus/feature-guide.mdx` — Java Nexus feature guide (GA).
- `docs/encyclopedia/nexus/nexus-security.mdx` (lines 24, 28, 33, 41) — cross-Namespace security model in Temporal Cloud.
