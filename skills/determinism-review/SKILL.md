---
name: determinism-review
description: >
  Step-by-step determinism review of Temporal workflow code changes. Use when
  the user asks to: "check my workflow changes for non-determinism", "is this
  change replay-safe?", "review my diff for determinism issues", "will this
  break running workflows?", "do I need versioning?", "do I need patching?",
  "determinism check", "replay safety review", "can I deploy this workflow
  change?", "non-determinism review", or any request to verify that workflow
  code changes won't cause NondeterminismError. Also activate automatically
  when the user asks to commit, push, or deploy changes and the diff contains
  Temporal workflow code (files importing temporalio, go.temporal.io/sdk,
  @temporalio/workflow, io.temporal.workflow, Temporalio.Workflows, or
  Temporalio::Workflow). Covers Go, Python, TypeScript, Java, .NET, and Ruby.
---

# Determinism Review

Walk a developer through every check a human expert performs before deploying workflow code changes. Two modes:

- **Quick check** (auto on commit/push): Steps 1-3. Compact output. Pauses on blockers.
- **Full wizard** (explicit invocation): All 7 steps. Detailed report with replay test scaffolding and versioning guidance.

## Core Principles

- **Determinism is about replay, not "bad code."** Reordering two valid activity calls is just as dangerous as calling `time.Now()` in a workflow.
- **Pattern detection gives definitive verdicts. Command sequence analysis is best-effort.** The skill is honest about where LLM analysis has limits. Only replay tests prove replay safety.
- **The strongest verdict without passing replay tests is "NO ISSUES DETECTED."** Never HIGH confidence without replay tests.
- **Guide, don't just flag.** When replay tests are missing, provide the scaffolding to create them. When versioning is needed, provide the code.

## Two Modes

### Quick Check (auto-triggered)

Detect Temporal workflow imports in `git diff --cached`. If found:

1. Run Steps 1-3 (identify workflow code, forbidden APIs, command sequence)
2. Output a compact summary: "Your diff touches workflow code in [files]. Quick determinism check: [findings or clean]."
3. If blockers found, surface them before the commit proceeds
4. If no issues: "No determinism issues detected. Consider running replay tests for full confidence."

### Full Wizard (explicit invocation)

Run all 7 steps. Produce the full severity-bucketed report.

## Hook Setup (optional)

For automatic pre-commit checking, add to `.claude/settings.json`:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "if echo \"$TOOL_INPUT\" | grep -qE '(git commit|git push)'; then echo 'TEMPORAL_WORKFLOW_CHECK: Run /determinism-review quick check on staged changes before committing'; fi"
          }
        ]
      }
    ]
  }
}
```

---

## Process

### Step 0: Acquire the Diff

Determine the diff scope. Default to staged changes.

| Mode | Command | When |
|------|---------|------|
| Staged | `git diff --cached` | Default (pre-commit) |
| Unstaged | `git diff` | Working tree review |
| Branch | `git diff <base>..HEAD` | PR review |
| Files | Read specific files | User provides paths |

If the diff is empty: "No changes to review." Stop.

### Step 1: Identify Workflow Code

Check every changed file for workflow markers. If none found, the review is done. For background on why workflow code must be deterministic, see `references/core/determinism.md`.

**Language detection markers:**

| Language | Direct workflow markers | Import patterns |
|----------|----------------------|-----------------|
| Python | `@workflow.defn`, `@workflow.run`, `@workflow.signal`, `@workflow.query`, `@workflow.update` | `from temporalio import workflow`, `from temporalio.workflow import` |
| Go | Function param `workflow.Context`, struct method on registered workflow | `"go.temporal.io/sdk/workflow"` |
| TypeScript | Exported functions from workflow module | `from '@temporalio/workflow'`, `import { ... } from '@temporalio/workflow'` |
| Java | `@WorkflowMethod`, `@SignalMethod`, `@QueryMethod`, `@UpdateMethod`, `@WorkflowInterface` | `import io.temporal.workflow.*` |
| .NET | `[Workflow]`, `[WorkflowRun]`, `[WorkflowSignal]`, `[WorkflowQuery]`, `[WorkflowUpdate]` | `using Temporalio.Workflows` |
| Ruby | Subclass of `Temporalio::Workflow` | `require 'temporalio'` |

**Transitive analysis:** For changed files that are NOT directly workflow code, grep the codebase for imports of those files from workflow code. If found, mark as "transitively workflow-affecting" and include in review.

```bash
# Example: check if a changed helper is imported by any workflow file
grep -rn "import.*changed_helper" --include="*.py" | grep -l "workflow"
```

**If no workflow code affected:**
> No workflow code in this diff. Changes to activities, tests, and utilities don't affect workflow determinism.

Stop here. In quick check mode, allow the commit to proceed.

### Step 2: Forbidden API + Sandbox Bypass Check

Scan ADDED or CHANGED lines (`+` lines in the diff) in files identified as workflow code by Step 1. Step 1 checks the **full file** for workflow markers; this step checks only the **diff lines** for forbidden calls. This catches any forbidden API introduced by this change, even if the only modification to an existing workflow is the forbidden call itself.

Load the per-language determinism reference for the detected language:
- Python: `references/python/determinism.md`
- Go: `references/go/determinism.md`
- TypeScript: `references/typescript/determinism.md`
- Java: `references/java/determinism.md`
- .NET: `references/dotnet/determinism.md`
- Ruby: `references/ruby/determinism.md`

**What to check:**

1. **Forbidden API calls** - time, random, UUID, I/O, threading, concurrency, global state, iteration (per-language tables in reference files)
2. **Sandbox escape hatches added in the diff:**

| Language | Escape hatches to flag |
|----------|----------------------|
| Python | `sandbox_unrestricted()`, `imports_passed_through()` on I/O libs, custom `SandboxRestrictions`, `disable_lazy_sys_module_passthrough` |
| Ruby | `illegal_call_tracing_disabled`, `io_enabled`, `durable_scheduler_disabled` |
| TypeScript | New entries in `bundlerOptions.ignoreModules` |
| Go | `//workflowcheck:ignore` annotations |

**Severity calibration:** Forbidden API findings in sandboxed languages (Python, TypeScript) are warnings if the sandbox is active. In unsandboxed languages (Go, Java, .NET, Ruby), they are blockers.

**Report format:**

| Location | Forbidden call | Use instead | Severity |
|----------|---------------|-------------|----------|
| `file.py:42` | `datetime.now()` | `workflow.now()` | Blocker |
| `file.py:67` | `random.random()` | `workflow.random()` | Warning (sandbox active) |

### Step 3: Command Sequence Analysis

Check if the diff adds, removes, or reorders any command-producing operations. Load `references/core/command-sequence-patterns.md` for examples.

**Command-producing operations:**

| Operation | Examples across SDKs |
|-----------|---------------------|
| Execute activity | `workflow.execute_activity`, `workflow.ExecuteActivity`, `proxyActivities`, `Workflow.newActivityStub` |
| Execute local activity | `workflow.execute_local_activity`, `workflow.ExecuteLocalActivity` |
| Start child workflow | `workflow.execute_child_workflow`, `workflow.ExecuteChildWorkflow` |
| Signal external workflow | `workflow.signal_external_workflow`, `SignalExternalWorkflow` |
| Cancel external workflow | `workflow.request_cancel_external_workflow`, `RequestCancelExternalWorkflow` |
| Sleep / timer | `workflow.sleep`, `workflow.Sleep`, `sleep()` (TS) |
| SideEffect | `workflow.SideEffect`, `workflow.side_effect` |
| MutableSideEffect | `workflow.MutableSideEffect` |
| UpsertSearchAttributes | `workflow.upsert_search_attributes`, `workflow.UpsertSearchAttributes` |
| UpsertMemo | `workflow.upsert_memo` |
| ContinueAsNew | `workflow.continue_as_new`, `workflow.NewContinueAsNewError` |
| Goroutine creation (Go) | `workflow.Go(ctx, ...)` - ordering matters |

**Patterns to flag:**

1. New command-producing call added
2. Existing command-producing call removed
3. Reorder of command-producing calls
4. Existing commands wrapped in new conditional on mutable state
5. Loop iteration count changed where loop body produces commands
6. `continue-as-new` condition or carried-over args changed
7. Sequential to parallel conversion (`await a; await b` to `Promise.all`)
8. Error handling path added/changed that calls different commands
9. Timer removed between command-producing calls

**Important caveat to include in output:**
> Static analysis can detect obvious command sequence changes but cannot catch all cases. These findings are candidates for non-determinism - verify with replay tests for definitive confirmation.

**Quick check mode stops here.** Report findings and either proceed (clean) or pause (blockers/warnings found).

---

### Step 4: Handler + Serialization Issues

**Handler checks:**

| Handler type | Rule | What to look for in diff |
|-------------|------|-------------------------|
| Query | Must NOT mutate state | Assignments to `self.*` / `this.*` / struct fields in query handler body |
| Update validator | Must NOT mutate state or block | Assignments or `await`/`sleep` in validator body |
| Signal | Same determinism rules as workflow code | All checks from Steps 2-3 apply to signal handler body |

**Serialization checks:**

- **Struct/proto field changes** in types used for: workflow input/output, activity input/output, signal payloads, update payloads, ContinueAsNew args, search attributes. Adding/removing/renaming fields can cause deserialization failures on replay that look like NondeterminismError to the developer.
- **DataConverter / PayloadCodec changes** - any modification to serialization/deserialization is dangerous. An activity result encoded with the old converter won't decode with the new one on replay.

### Step 5: Existing Protection Assessment

Check what automated determinism protection exists in this codebase. Load the per-language determinism-protection reference:
- `references/{lang}/determinism-protection.md`

| Language | Protection | How to check | Status indicators |
|----------|-----------|--------------|-------------------|
| Python | Sandbox | Default enabled. Check worker config for `SandboxedWorkflowRunner` overrides, `sandbox_unrestricted=True` | Active / Overridden / Bypassed |
| Go | `workflowcheck` | Check CI config (`.github/workflows/`, `Makefile`). Suggest: `workflowcheck ./...` | In CI / Available / Not found |
| Java | `temporal-workflowcheck` | Check `pom.xml` / `build.gradle` for the dependency | Configured / Not found |
| TypeScript | V8 sandbox | Automatic. Check for `ignoreModules` additions in worker config | Active / Holes via ignoreModules |
| .NET | Runtime EventListener | Check for `Temporalio.Extensions.DiagnosticSource` | Configured / Not found |
| Ruby | TracePoint | Check worker `illegal_workflow_calls` config | Active / Customized / Disabled |

### Step 6: Replay Test Assessment

Search the codebase for existing replay tests. Load the per-language testing reference:
- `references/{lang}/testing.md`

**Search patterns:**

| Language | Replay test indicators |
|----------|----------------------|
| Python | `Replayer(`, `replay_workflow(` |
| Go | `worker.NewWorkflowReplayer()`, `ReplayWorkflowHistory` |
| TypeScript | `Worker.runReplayHistory(`, `runReplayHistory` |
| Java | `WorkflowReplayer.replay` |
| .NET | `WorkflowReplayer`, `ReplayWorkflowAsync` |
| Ruby | `WorkflowReplayer.new` |

Also search for history JSON files: `find . -name "*.json" -path "*/test*" | head -20` and check for workflow history structure.

**Three outcomes:**

**1. Replay tests exist → run them**
- Execute the tests: `pytest` (Python), `go test ./...` (Go), `npm test` / `npx vitest` (TS), `mvn test` / `gradle test` (Java), `dotnet test` (.NET), `bundle exec rspec` (Ruby).
- If tests pass: HIGH confidence. Check history file freshness with `git log -1 -- <history_file>`. Flag if only 1 history file for a workflow with multiple code branches (low coverage).
- If tests fail: the diff broke replay compatibility. Three options:
  1. Fix the code to preserve the command sequence
  2. Use the Patching API to support both old and new paths (see Step 7)
  3. Re-record histories if this is an intentional, versioned change

**2. No replay tests, but history files exist**
- Write the replay test using scaffolding from `references/{lang}/testing.md` (Replay Testing section).
- Run the test. Report pass/fail.

**3. No replay tests and no history files**
- This is the single most impactful improvement for determinism safety.
- Load `references/{lang}/testing.md` (Replay Testing section) and provide:
  1. History capture command: `temporal workflow show --workflow-id <id> --output json > history.json`
  2. Test file template from the testing reference
  3. CI integration guidance
- If a local dev server is running (`temporal server start-dev`), offer to capture a history by running the workflow, then write and execute the replay test.

### Step 7: Versioning Recommendation

Only relevant if Step 3 found command sequence changes. Load `references/core/versioning.md` and `references/{lang}/versioning.md` for language-specific patterns. Ask the developer:

> Are there running workflows in production using the old code? How long-lived are they?

The skill cannot query the Temporal server, so this information must come from the developer.

**Decision matrix:**

| Scenario | Recommended approach |
|----------|---------------------|
| Small change, few running workflows | Patching API |
| Major rewrite | New workflow type (e.g., `OrderWorkflowV2`) |
| Frequent deploys, short-lived workflows | Worker Versioning with PINNED behavior |
| Long-running workflows needing bug fixes | Worker Versioning with AUTO_UPGRADE + Patching |
| Can wait for all workflows to complete | Wait for drain, then deploy |

**Patching API lifecycle:**
1. **Patch In** - add both old and new code paths with a descriptive patch ID (e.g., `"add-fraud-check"`, not `"patch-1"`)
2. **Deprecate** - after all old workflows complete, remove old path, keep deprecation marker
3. **Remove** - after all deprecated workflows complete, remove patch entirely

**Per-language patching example:**

| Language | API |
|----------|-----|
| Python | `if workflow.patched("add-notification"):` |
| Go | `v := workflow.GetVersion(ctx, "add-notification", workflow.DefaultVersion, 1)` |
| TypeScript | `if (patched("add-notification")) {` |
| Java | `int v = Workflow.getVersion("add-notification", DEFAULT_VERSION, 1);` |
| .NET | `if (Workflow.Patched("add-notification")) {` |
| Ruby | `if workflow.patched("add-notification")` |

---

## Output Template

### Quick Check Mode

```
## Quick Determinism Check

Your diff touches workflow code in: [file list]
Language: [X] | Files: [N]

[CLEAN: No determinism issues detected. Consider replay tests for full confidence.]

-- or --

[ISSUES FOUND:]
| Location | Issue | Severity |
|----------|-------|----------|
| file:line | description | Blocker/Warning |

[Action needed before committing: ...]
```

### Full Wizard Mode

```
## Determinism Review

Language: [X] | Diff: [staged/branch/files] | Files: [N]

### Blockers (will cause NondeterminismError)

| Location | Issue | Fix |
|----------|-------|-----|
| ... | ... | ... |

(or: None found.)

### Warnings (likely needs versioning)

| Location | Issue | Recommendation |
|----------|-------|----------------|
| ... | ... | ... |

(or: None found.)

### Info (verify manually)

| Location | Observation | Why it matters |
|----------|------------|----------------|
| ... | ... | ... |

### Protections in Place

| Protection | Status | Coverage |
|------------|--------|----------|
| ... | ... | ... |

### Replay Tests

[Status + guidance or scaffolding]

### Versioning Recommendation

[Decision matrix result + code example, or "Not needed"]

---

### Verdict

**[NO ISSUES DETECTED / NEEDS_VERSIONING / NEEDS_REPLAY_TESTS / BLOCKED]**

Confidence: [HIGH / MEDIUM / LOW]

- HIGH: replay tests exist and pass, no forbidden APIs, no command sequence changes
- MEDIUM: no forbidden APIs or obvious command sequence changes, but no replay tests to confirm
- LOW: complex changes, limited static analysis coverage, no replay tests

Next steps:
1. [specific actionable item]
2. [...]
```

---

## Common Pitfalls

| Pitfall | What goes wrong | Fix |
|---------|----------------|-----|
| Flagging activity code as non-deterministic | Activities run outside the workflow sandbox. I/O, randomness, and side effects are expected in activities. | Only apply forbidden operations checks to workflow code (Step 1 filters this). |
| Ignoring transitive dependencies | A changed utility function is imported by a workflow. The utility's changes affect replay. | Always trace imports from changed non-workflow files back to workflow code. |
| Treating all changes as command sequence changes | Variable renames, logging changes, and local computation are safe. | Only flag changes that affect the order/type/existence of Temporal Commands. |
| Missing conditional wrapping as a sequence change | An `if` statement wrapping an existing `execute_activity` changes which commands are generated for some paths. | Flag any new conditional logic around command-producing operations. |
| Over-recommending Patching | Many changes don't affect command sequence (adding query handlers, changing retry policies, modifying activity implementations). | Use the full versioning decision matrix. Patching is one of five options. |
| Assuming sandboxes catch everything | Python and TypeScript sandboxes prevent forbidden API calls but do NOT prevent command sequence changes. | Steps 3 and 6 are always necessary regardless of sandbox status. |
| Giving SAFE/HIGH without replay tests | Static analysis has limits. Only replay tests prove compatibility. | Never assign HIGH confidence without passing replay tests. Best without tests: "NO ISSUES DETECTED - verify with replay tests." |
| Skipping replay test scaffolding | Telling developers "you need replay tests" without showing how. | Always provide the concrete scaffolding: history capture, test template, CI suggestion. |
| Confusing workflow versioning with Worker Versioning | Patching API (code-level branching) vs Worker Versioning / Build IDs (deployment-level routing). Different problems. | Present the correct approach from the decision matrix based on the scenario. |
| Flagging timer duration changes as blockers | Changing a sleep from 5s to 10s is a parameter change, not a sequence change. Won't cause NDE. | Flag as Info, not Blocker. Note: in-flight timers use the original duration. |
| False positive on imports for type annotations only | Python `import datetime` for type hints doesn't run at workflow execution time. | Check if the import is used in executable code, not just type annotations. |
| Missing Go goroutine ordering | Reordering `workflow.Go()` calls changes the internal command sequence even if each goroutine does the same work. | Flag any reordering of `workflow.Go()` calls as a potential command sequence change. |
