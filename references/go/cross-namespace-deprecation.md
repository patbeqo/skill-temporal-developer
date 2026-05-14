# Cross-Namespace Child Workflow APIs: Deprecation (Go)

## Summary

OSS Temporal supports cross-Namespace commands (parent-child, SignalExternal, CancelExternal) through the `system.enableCrossNamespaceCommands` configuration, and this configuration is disabled on Temporal Cloud.
Code using cross-Namespace calls must be updated or removed prior to migration to Temporal Cloud.
A Child Workflow Execution is spawned from within another Workflow in the same Namespace.
For cross-Namespace orchestration use Temporal Nexus; Go SDK support for Nexus is Generally Available.

## What this affects

- `workflow.ExecuteChildWorkflow` — child Workflow Executions live in the same Namespace as the parent; do not attempt to spawn one in a different Namespace.
- `workflow.ChildWorkflowOptions` (applied via `workflow.WithChildOptions`) — do not pass a target-Namespace selector to spawn a child in another Namespace.
- `workflow.SignalExternalWorkflow(ctx, "some-workflow-id", "", "your-signal-name", signal)` — the documented call shape signals a Workflow in the caller's own Namespace; do not introduce a Namespace-targeted variant from a workflow.
- Request-cancel-external on a Workflow in a different Namespace is likewise blocked under the same configuration gate.

## What to do instead

Use Temporal Nexus to connect Temporal Applications within and across Namespaces using a Nexus Endpoint, a Nexus Service contract, and Nexus Operations.
Temporal Go SDK support for Nexus is Generally Available.
Create a Nexus Endpoint to route requests from caller to handler Namespace.
On Temporal Cloud, each Endpoint has an access control policy: an allowlist of caller Namespaces, and Temporal Cloud verifies the caller's Namespace is in the Endpoint's allowlist before routing the request to the handler.
Temporal Cloud provides built-in Endpoint access controls and secure connectivity across Namespaces.

See the Go Nexus feature guide for the caller/handler Workflow and Endpoint creation flow: `docs/develop/go/nexus/feature-guide.mdx`.

## Quick checks (don't / do)

- Don't call `workflow.ExecuteChildWorkflow` expecting the child to land in a different Namespace; do keep child Workflows in the same Namespace as the parent.
- Don't fabricate a target-Namespace option on `workflow.ChildWorkflowOptions`; do model cross-Namespace work as a Nexus Operation.
- Don't invent a 6-argument `workflow.SignalExternalWorkflow` variant that takes a Namespace; do use the documented 5-argument shape with an empty run-id string and keep the target in the caller's Namespace.
- Don't rely on `system.enableCrossNamespaceCommands` behavior in code intended for Temporal Cloud; do remove or replace cross-Namespace calls before migration.
- Don't conflate Nexus with child Workflows; do treat Nexus as its own pattern with an Endpoint, Service contract, and Operations.

## References

- `docs/cloud/migrate/automated.mdx`
- `docs/encyclopedia/child-workflows/child-workflows.mdx`
- `docs/develop/go/workflows/child-workflows.mdx`
- `docs/develop/go/workflows/message-passing.mdx`
- `docs/develop/go/nexus/feature-guide.mdx`
- `docs/encyclopedia/nexus/nexus-security.mdx`
