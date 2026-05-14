# Resource tuning (Go)

Reference for the Go SDK Worker tuner and slot supplier APIs that live in the `worker` package, plus the `sysinfo` contrib provider used for resource-based tuning and host resource reporting.

## Overview

Worker performance in the Go SDK is constrained by three resources: compute (CPU), memory, and IO (network/polling).

A Slot Supplier hands out execution slots of one type — Workflow, Activity, Local Activity, or Nexus — and a Worker Tuner assigns Slot Suppliers to each slot type.

Three slot supplier strategies exist: fixed-size, resource-based (auto-tuning on CPU and memory), and custom.

In containerized environments, all SDKs use cgroups for both CPU and memory; CPU is accounted for at the container level.

## Imports

Use these two packages — and only these two — for resource-based tuning in Go:

```go
import (
    "go.temporal.io/sdk/contrib/sysinfo"
    "go.temporal.io/sdk/worker"
)
```

- **Don't import `go.temporal.io/sdk/contrib/resourcetuner`.** All tuner and slot-supplier constructors live in `go.temporal.io/sdk/worker`; the only contrib import for this feature is `go.temporal.io/sdk/contrib/sysinfo`.

## Go SDK defaults

The Go row of the defaults tables:

| Setting | Go default |
|---|---|
| `MaxConcurrentWorkflowTaskExecutionSize` | 1,000  |
| `MaxConcurrentActivityTaskExecutionSize` | 1,000  |
| `MaxConcurrentLocalActivityTaskExecutionSize` | 1,000  |
| `MaxCachedWorkflows` / `StickyWorkflowCacheSize` | 10,000  |
| `MaxConcurrentWorkflowTaskPollers` | 2  |
| `MaxConcurrentActivityTaskPollers` | 2  |
| Namespace APS | 400  |
| `TaskQueueActivitiesPerSecond` | Unlimited  |

For the Go SDK cache, use [`SetStickyWorkflowCacheSize`](https://pkg.go.dev/go.temporal.io/sdk/worker#SetStickyWorkflowCacheSize).

## Choosing fixed vs. resource-based vs. custom

- **Workflow Tasks are well-served by fixed-size suppliers** — they make minimal CPU demands and normally do not consume much memory.
- **For maximum throughput and lowest task-completion latency, avoid resource-based auto-tuning suppliers.** A fixed-size tuner with appropriately chosen configuration outperforms the resource-based tuner.
- **Use resource-based suppliers when you want acceptable performance without profiling**, for fluctuating workloads with low per-Task consumption (e.g. blocking I/O), or to protect against OOM with unpredictable per-task resource use.
- **Resource-based suppliers cannot guarantee targets are never exceeded** — resources consumed during a task cannot be known ahead of time.
- **Use custom slot suppliers when you need complete control over slot allocation logic.**

## Resource-based tuner

A `ResourceBasedTuner` uses one resource controller (driven by CPU and memory targets) to hand out slots for every task type:

```go
func resourceBasedTuner() (worker.Options, error) {
    tuner, err := worker.NewResourceBasedTuner(worker.ResourceBasedTunerOptions{
        TargetMem:    0.8,
        TargetCpu:    0.9,
        InfoSupplier: sysinfo.SysInfoProvider(),
    })
    if err != nil {
        return worker.Options{}, err
    }
    return worker.Options{
        Tuner: tuner,
    }, nil
}
```

- `worker.ResourceBasedTunerOptions` fields are `TargetMem`, `TargetCpu`, `InfoSupplier`.
- `InfoSupplier` is filled with `sysinfo.SysInfoProvider()` from `go.temporal.io/sdk/contrib/sysinfo`, which is gopsutil-based and supports cgroup metrics on containerized Linux.
- `worker.NewResourceBasedTuner` returns `(Tuner, error)` — propagate the error.
- Attach the returned tuner to `worker.Options.Tuner`.

## Composite tuner

A composite tuner mixes slot supplier strategies per task type — e.g. fixed-size for Workflow and Nexus Tasks, resource-based for Activity and Local Activity Tasks.

```go
func compositeTuner() (worker.Options, error) {
    options := worker.DefaultResourceControllerOptions()
    options.MemTargetPercent = 0.8
    options.CpuTargetPercent = 0.9
    options.InfoSupplier = sysinfo.SysInfoProvider()
    controller := worker.NewResourceController(options)

    wfSS, err := worker.NewFixedSizeSlotSupplier(10)
    if err != nil {
        return worker.Options{}, err
    }

    actSS, err := worker.NewResourceBasedSlotSupplier(controller, worker.DefaultActivityResourceBasedSlotSupplierOptions())
    if err != nil {
        return worker.Options{}, err
    }
    laSS, err := worker.NewResourceBasedSlotSupplier(controller, worker.DefaultActivityResourceBasedSlotSupplierOptions())
    if err != nil {
        return worker.Options{}, err
    }
    nexusSS, err := worker.NewFixedSizeSlotSupplier(10)
    if err != nil {
        return worker.Options{}, err
    }

    compositeTuner, err := worker.NewCompositeTuner(worker.CompositeTunerOptions{
        WorkflowSlotSupplier:      wfSS,
        ActivitySlotSupplier:      actSS,
        LocalActivitySlotSupplier: laSS,
        NexusSlotSupplier:         nexusSS,
    })
    if err != nil {
        return worker.Options{}, err
    }
    return worker.Options{
        Tuner: compositeTuner,
    }, nil
}
```

Key shapes for code generation:

- `worker.DefaultResourceControllerOptions()` returns a struct with mutable fields `MemTargetPercent`, `CpuTargetPercent`, `InfoSupplier`.
- **Don't write `MemTargetPercent` / `CpuTargetPercent` on `ResourceBasedTunerOptions`, and don't write `TargetMem` / `TargetCpu` on the controller options.** The tuner-options struct uses `TargetMem` / `TargetCpu`; the controller-options struct uses `MemTargetPercent` / `CpuTargetPercent`. Both structs take `InfoSupplier`.
- `worker.NewResourceController(options)` builds the controller from those options.
- `worker.NewFixedSizeSlotSupplier(n)` returns `(SlotSupplier, error)`.
- `worker.NewResourceBasedSlotSupplier(controller, worker.DefaultActivityResourceBasedSlotSupplierOptions())` is the canonical constructor for activity-style and local-activity-style suppliers; the docs use `DefaultActivityResourceBasedSlotSupplierOptions()` for both.
- `worker.NewCompositeTuner` takes a `worker.CompositeTunerOptions` struct with four fields: `WorkflowSlotSupplier`, `ActivitySlotSupplier`, `LocalActivitySlotSupplier`, `NexusSlotSupplier`.
- **Don't call `NewCompositeTuner` positionally and don't omit `NexusSlotSupplier`.** Always pass the named-struct form with all four fields.
- Every constructor in this snippet (`NewFixedSizeSlotSupplier`, `NewResourceBasedSlotSupplier`, `NewCompositeTuner`) returns `(value, error)` — propagate each error.

## Constraints and metric caveats

- **Setting both `Tuner` and any `MaxConcurrentXXXTask` option on the same Worker errors at Worker initialization.** Pick one: use `Tuner` for slot-supplier-based tuning, `MaxConcurrent...` for fixed slot counts without a `Tuner`.
- **`worker_task_slots_available` does not work with resource-based slot suppliers** — it is fixed-supplier-only. Use `worker_task_slots_used` if you need a metric that works with both.
- **`rampThrottle` is the resource-based slot-supplier option** that controls the minimum wait between handing out new slots after passing the minimum slot count; a higher value trades performance for safety.

## `worker.Options.SysInfoProvider` — host resource reporting

`worker.Options.SysInfoProvider` is a separate concern from the resource-based tuner's `InfoSupplier`. It controls what the Worker reports for CPU and memory usage in Worker heartbeats.

By default, the Go SDK reports `0` for CPU and memory usage in Worker heartbeats; set `SysInfoProvider` on `worker.Options` to enable host resource reporting.

```go
import (
    "go.temporal.io/sdk/contrib/sysinfo"
    "go.temporal.io/sdk/worker"
)

w := worker.New(c, "my-task-queue", worker.Options{
    SysInfoProvider: sysinfo.SysInfoProvider(),
})
```

- `sysinfo.SysInfoProvider()` from `go.temporal.io/sdk/contrib/sysinfo` is the same function used for the tuner `InfoSupplier`; both options accept it, but they sit on different structs and serve different purposes (heartbeat reporting vs. resource-based tuner input).
- You can implement the `worker.SysInfoProvider` interface to provide your own resource metrics.

## Custom slot suppliers

The Go custom-supplier interface is [`SlotSupplier`](https://pkg.go.dev/go.temporal.io/sdk/worker#SlotSupplier).

A custom Slot Supplier must implement these methods:

- `reserveSlot` — called before polling for new tasks; may block; must return a Slot Permit once it accepts new work.
- `tryReserveSlot` — called for slot reservations in cases like eager activity processing; must not block.
- `markSlotUsed` — called when a slot is about to be used for a task (not while it is held during polling).
- `releaseSlot` — called when a slot is no longer needed, whether or not it was used.

## Common mistakes

- Importing `go.temporal.io/sdk/contrib/resourcetuner`. The correct paths are `go.temporal.io/sdk/worker` (tuners and slot suppliers) and `go.temporal.io/sdk/contrib/sysinfo` (`SysInfoProvider`).
- Calling `resourcetuner.NewResourceBasedTuner(...)`. The constructor is `worker.NewResourceBasedTuner(...)`.
- Writing `worker.ResourceBasedTunerOptions{MemTargetPercent: ..., CpuTargetPercent: ...}`. Those fields belong to the controller options struct; the tuner options struct uses `TargetMem` and `TargetCpu`.
- Mutating `worker.DefaultResourceControllerOptions()` fields with the tuner-options names (`TargetMem`, `TargetCpu`). The controller options use `MemTargetPercent` and `CpuTargetPercent`.
- Calling `worker.NewCompositeTuner` positionally or with only three slot suppliers. Use the four-field `worker.CompositeTunerOptions` struct including `NexusSlotSupplier`.
- Combining `worker.Options{Tuner: t, MaxConcurrentActivityExecutionSize: 100}`. This errors at Worker initialization — pick one style.
- Alerting on `worker_task_slots_available` for a Worker that uses a resource-based slot supplier. Track `worker_task_slots_used` instead.
- Using non-Go default values (e.g. 200 activity slots from the Java row, 100 from Python). The Go executor defaults are 1,000 / 1,000 / 1,000; the cache default is 10,000; pollers default to 2.
- Treating `worker.Options.SysInfoProvider` and the tuner-options `InfoSupplier` as the same setting. They are two different fields on two different structs; both can be filled with `sysinfo.SysInfoProvider()` independently.
