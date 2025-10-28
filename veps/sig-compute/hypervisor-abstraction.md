# VEP #95: Hypervisor Abstraction Layer

## Release Signoff Checklist

Items marked with (R) are required *prior to targeting to a milestone / release*.

- [x] (R) Enhancement issue created, which links to VEP dir in [kubevirt/enhancements]
- [ ] (R) Target version is explicitly mentioned and approved
- [ ] (R) Graduation criteria filled

## Overview

Introduce a Hypervisor Abstraction Layer that lets KubeVirt plug in multiple hypervisor backends through a consistent contract, while keeping today's KVM-first behavior unchanged. To start, we scope the contract to device selection, domain tweaks, validation and mutation so as to provide a flexible base that can evolve without introducing a major refactor. 

## Motivation

- KubeVirt currently hard-codes KVM/QEMU assumptions throughout virt-launcher, virt-handler, virt-controller, and node preparation scripts.
- Platform teams that rely on alternative accelerators or hypervisor stacks face invasive forks to replace `/dev/kvm`, libvirt domain types, or resource scheduling hints.
- A scoped abstraction keeps the project approachable for contributors while unlocking new hardware backends.

## Goals

- Document the cluster-wide hypervisor configuration and per-component extension points (defaults, converter, webhooks, node labeller) so downstream implementations can extend behavior without invasive changes to existing components.
- Resolve the active hypervisor early and feed it through the virt-launcher converter so hypervisor-specific behavior stays localized.
- Support both admission-time validation and mutation so administrators can enforce guardrails while still customizing VMIs for a given hypervisor.
- Let components request allocatable device resources declared by the hypervisor configuration, avoiding new scheduling primitives.
- Make it simple for downstreams to implement new hypervisors by following a documented contract.

## Non Goals

- Deliver a full implementation of any specific new hypervisor backend.
- Redesign the VirtualMachineInstance API schema beyond additive fields.
- Replace existing Hyper-V enlightenment features or other architecture-specific helpers.
- Mandate new observability requirements; telemetry hooks remain optional.

## Definition of Users

- Cluster administrators who need to bootstrap KubeVirt on hardware that exposes alternative virtualization devices.
- Platform vendors integrating proprietary or emerging hypervisor stacks with KubeVirt.
- Upstream contributors maintaining virt-launcher, virt-controller, and virt-handler.

## User Stories

1. As a cluster administrator, I can declare a non-KVM hypervisor as the cluster default, and VMI pods schedule only on nodes that expose its required devices.
2. As a platform engineer, I can supply hypervisor-specific VMI spec mutations and libvirt domain adjustments without forking the virt-launcher converter.
3. As an upstream maintainer, I know exactly where to add validation, testing, and documentation when a new hypervisor is introduced.

## Repos

- [kubevirt/kubevirt](https://github.com/kubevirt/kubevirt)
- [kubevirt/enhancements](https://github.com/kubevirt/enhancements) (this VEP)

## Design

### Hypervisor Extension Points

Cluster configuration (`spec.configuration.hypervisorConfiguration.name`) declares the active hypervisor for the entire installation, and each control-plane package exposes focused extension contracts so downstream implementations only touch the areas they actually need:

 - **Defaults mutator dispatcher (`pkg/defaults/dispatch/`)** – Defaulting is expressed as small, pure mutator functions grouped by specificity: Base, Hypervisor, Architecture, Hypervisor+Architecture, then Finalizers. At admission/reconcile time we build a deterministic slice and execute each mutator in order. Later mutators may refine (override) values set earlier. Registration helpers populate global maps during `init()`; selection just assembles the pipeline for the resolved hypervisor+arch. This keeps mutators loosely coupled and testable as independent units.
- **Runtime interface (`pkg/hypervisor/runtime/`)** – Provides a shared `HypervisorRuntime` contract for runtime-specific behavior such as `AdjustResources`, `HandleHousekeeping`, and `GetMemoryOverhead`. Each implementation registers under the same hypervisor key so `virt-controller`, `virt-handler`, and virt-launcher can resolve the correct runtime hooks.
- **Converter library (`pkg/virt-launcher/virtwrap/converter/hypervisor/`)** – Implements the new `HypervisorConverter` interface described below. Each hypervisor file focuses on XML/domain differences while `base.go` holds the shared helpers. The converter selects the correct implementation via a local registry keyed by hypervisor name.
- **Admission webhooks (`pkg/virt-api/webhooks/validating-webhook/admitters/hypervisor/`)** – Surface validation and mutation helpers that wrap existing webhook entry points. The admitters use the same cluster-configured hypervisor value as `virt-controller` and invoke the matching implementation.
- **Node labeller (`pkg/virt-handler/node-labeller/hypervisor/`)** – Adds a lightweight hook so each hypervisor can declare the devices to probe, the preferred libvirt `virt-type`, and optional feature discovery (such as Hyper-V enlightenments for KVM on amd64).

This split preserves the “implement once, reuse everywhere” story without routing everything through a monolithic interface. New hypervisors can land incrementally—start with defaults and webhooks, add converter support, then extend node labelling—while keeping the contract for each area explicit and testable.

### Selection

- `virt-config` loads cluster-wide defaults from an additive `hypervisorConfiguration` field on the `KubeVirt` CR. The `name` selects the cluster-wide default hypervisor implementation. A dedicated feature gate, `ConfigurableHypervisor`, guards the new functionality:

```yaml
spec:
  configuration:
    hypervisorConfiguration:
      name: kvm
    developerConfiguration:
      featureGates:
        - ConfigurableHypervisor
```

- `virt-controller` reads the configured hypervisor from `ClusterConfig` when generating launcher manifests and threads that ID through the `ConverterContext` so downstream components can act consistently.
- Each package's registry uses the configured name to locate its implementation, avoiding a monolithic factory while keeping selection logic consistent.

### Integration with Defaults, Runtime, and Converter

1. `virt-controller` and `virt-handler` read the configured hypervisor from `ClusterConfig`, add that ID to the serialized `ConverterContext` they already ship alongside the launcher pod, and virt-launcher folds it into its `DomainContext` right before domain generation.
2. `pkg/defaults` pulls the `DefaultsExtension` associated with the configured hypervisor to mutate the VMI and surface `DeviceRequests`. The controller uses those requests when constructing launcher pods so the scheduler can account for hypervisor-specific devices.
3. Control-plane components resolve the `HypervisorRuntime` implementation to run `AdjustResources` and `GetMemoryOverhead`, keeping pod-level resource calculations in sync with the mutated spec. The same runtime contract is reused by virt-handler for memlock sizing and ancillary bookkeeping.
4. When virt-launcher converts the VMI, it instantiates both the `HypervisorConverter` and the `HypervisorRuntime`. The converter stamps baseline domain defaults and interleaves its edits with the existing architecture helpers, while the runtime's `HandleHousekeeping` hook attaches timers, watchdogs, and other hypervisor-specific tweaks immediately before the domain is finalized.
5. The launcher still reuses the existing `setLaunchSecurity`, disk, and network helpers; hypervisor-specific cases funnel through the converter and runtime abstractions so defaults remain declarative.

### Converter Restructuring Inside Virt-launcher

- The existing `pkg/virt-launcher/virtwrap/converter` package becomes a reusable library with a new `hypervisor` subpackage. Shared translation helpers for disks, NICs, CPU topology, and security settings live in `converter/hypervisor/base.go` as `BaseHypervisorConverter` utilities.
- A narrow `HypervisorConverter` interface (for example, `SetDomainType`) captures every point where the current converter branches on hypervisor-specific logic.
- Per-hypervisor implementations (e.g., `converter/hypervisor/kvm.go`, `converter/hypervisor/hyperv-layered.go`) embed the base helper and override only the methods they need. `NewHypervisorConverter(name string)` returns the correct implementation based on the resolved hypervisor name, mirroring the runtime factory used across the control plane.
- Existing call sites inside the converter depend on the interface, keeping shared logic untouched while cleanly isolating hypervisor-specialized code paths.
- Unit tests port alongside the refactor so every converter branch remains covered; new tests exercise the factory and ensure fallback to the KVM implementation when an unknown hypervisor is requested.

### Hypervisor-Specific Defaults

The functional dispatcher model expresses defaulting as ordered mutator functions instead of embedded provider structs. This reduces structural boilerplate and makes each concern testable in isolation.

Precedence (least → most specific):
1. Base mutators (generic cluster configuration driven)
2. Hypervisor mutators (e.g. kvm-wide adjustments)
3. Architecture mutators (generic amd64, arm64, s390x adjustments)
4. Hypervisor+Architecture mutators (fine-grained divergence)
5. Finalizers (derived/status + feature dependency resolution)

Core types:
```go
type VMIContext struct {
  Original      *v1.VirtualMachineInstance // snapshot before any mutation
  ClusterConfig *virtconfig.ClusterConfig
}

// VMIMutator mutates the VMI spec in place.
type VMIMutator func(spec *v1.VirtualMachineInstanceSpec, ctx *VMIContext) error

// VMIFinalizer runs after all mutators; may derive status or dependent fields.
type VMIFinalizer func(vmi *v1.VirtualMachineInstance, ctx *VMIContext) error
```

Registration stores mutators by specificity:
```go
var baseMutators []VMIMutator
var hypervisorMutators = map[string][]VMIMutator{}
var archMutators = map[string][]VMIMutator{}
var hypervisorArchMutators = map[string]map[string][]VMIMutator{} // hv -> arch -> []VMIMutator
var finalizers = map[string][]VMIFinalizer{} // keyed by hypervisor (arch-specific finalizers rarely needed)

func RegisterBase(mutators ...VMIMutator) { baseMutators = append(baseMutators, mutators...) }
func RegisterHypervisor(hv string, mutators ...VMIMutator) { hypervisorMutators[hv] = append(hypervisorMutators[hv], mutators...) }
func RegisterArch(arch string, mutators ...VMIMutator) { archMutators[arch] = append(archMutators[arch], mutators...) }
func RegisterHypervisorArch(hv, arch string, mutators ...VMIMutator) {
  m := hypervisorArchMutators[hv]
  if m == nil { m = map[string][]VMIMutator{}; hypervisorArchMutators[hv] = m }
  m[arch] = append(m[arch], mutators...)
}
func RegisterFinalizers(hv string, fns ...VMIFinalizer) { finalizers[hv] = append(finalizers[hv], fns...) }
```

Applying defaults (webhook/controller):
```go
func ApplyDefaults(vmi *v1.VirtualMachineInstance, hypervisor, arch string, cc *virtconfig.ClusterConfig) error {
  ctx := &VMIContext{Original: vmi.DeepCopy(), ClusterConfig: cc}
  spec := &vmi.Spec

  // Execute mutators in deterministic precedence order
  for _, fn := range baseMutators { if err := fn(spec, ctx); err != nil { return err } }
  for _, fn := range hypervisorMutators[hypervisor] { if err := fn(spec, ctx); err != nil { return err } }
  for _, fn := range archMutators[arch] { if err := fn(spec, ctx); err != nil { return err } }
  if hvMap := hypervisorArchMutators[hypervisor]; hvMap != nil {
    for _, fn := range hvMap[arch] { if err := fn(spec, ctx); err != nil { return err } }
  }
  for _, fin := range finalizers[hypervisor] { if err := fin(vmi, ctx); err != nil { return err } }
  return nil
}
```

Migration steps from legacy architecture functions:
1. Wrap existing architecture-specific logic in mutators and register under the appropriate bucket (e.g. `RegisterArch("amd64", Amd64CPUModelMutator)`).
2. Introduce hypervisor mutators returning the same defaults currently hard-coded for KVM; enable feature gate for hypervisor selection.
3. Add combined hypervisor+arch mutators only where divergence appears (e.g. machine type differences).
4. Implement finalizers for derived data currently handled in follow-up controller passes.
5. Deprecate old `SetAmd64Defaults` style functions after confirming parity via integration tests.


### Hypervisor-Specific Validations

`pkg/virt-api/webhooks/validating-webhook/admitters/hypervisor/` provides per-hypervisor validation and mutation helpers. Each implementation enforces compatibility (required devices, unsupported feature combinations) after defaults have populated the spec. The admitters resolve the active hypervisor from `ClusterConfig` and delegate to the matching validator.

Key points:
* Validation is distinct from defaulting: validators never set user-facing defaults (that is handled by `DefaultsProvider`).
* Hypervisor-specific rejection messages surface early (webhook) instead of deferring to runtime/libvirt errors.
* Tests cover both acceptance of valid specs and explicit rejection of incompatible feature / device combos.

### Scheduling and Device Management

- `virt-controller` reads the `DeviceRequests` declared by the defaults extension to determine the device plugin resources (for example, `devices.kubevirt.io/mshv` plus an auxiliary firmware device) to request. Kubernetes schedules VMI pods only on nodes that advertise the required quantities; entries flagged `Optional: true` may be skipped when the resource is absent.
- `virt-handler`'s device manager uses the same list when spawning its permanent `GenericDevicePlugin` instances, so the existing lifecycle for `/dev/kvm` seamlessly extends to `/dev/mshv` or composite requirements.
- The node-labeller sidecar in `virt-handler` is seeded with the resolved hypervisor. It probes only the devices declared by the active implementation and sets libvirt's preferred `virt-type` before querying capabilities, so downstream hypervisors can surface their own CPU/memory traits without patching the container image.
- Node labelling remains optional telemetry. Operators can surface informative labels, but functionality relies solely on allocatable resources. When the hypervisor/architecture combination implies additional feature discovery (for example, Hyper-V enlightenments), the labeller defers to helper hooks exposed by the implementation. In the MVP we continue to evaluate Hyper-V enlightenments only when the hypervisor is `kvm` and the architecture is `amd64`, matching the current behaviour while providing a seam for future backends.

### Validation & Mutation Webhooks

- The mutating webhook shares the same resolution flow and invokes `MutateVMI` early in the admission chain, giving providers a chance to normalize the spec (for example, seeding Hyper-V Layered feature blocks or toggling defaults) before Kubernetes persists the object.
- Hypervisors can use `Validate` to enforce requirements. The validating webhooks inside `virt-api` read the configured hypervisor from `ClusterConfig`, matching the value embedded by `virt-controller`, so admission stays consistent with reconciliation.
- In tandem, these hooks enable opinionated defaults for each hypervisor while still rejecting incompatible specs, delivering flexibility without sacrificing guardrails.

### Observability Hooks

- Monitoring can leverage existing metrics that expose allocatable device resources (e.g., `devices_kubevirt_io_*`). No new mandatory metrics are introduced.

## API Examples

### Cluster Configuration

```yaml
apiVersion: kubevirt.io/v1
kind: KubeVirt
metadata:
  name: kubevirt
  namespace: kubevirt
spec:
  configuration:
    hypervisorConfiguration:
      name: hyperv-layered
    developerConfiguration:
      featureGates:
        - ConfigurableHypervisor
    imagePullPolicy: Always
  imagePullPolicy: Always
```

With this configuration in place, every VMI reconciled by the control plane inherits the `hyperv-layered` behavior automatically—no per-object annotations are required.

### Adding a Hypervisor Implementation

1. **Defaults Mutators** – `pkg/defaults/dispatch/sample.go` (and optionally `sample_amd64.go` for arch-specific refinements):

   ```go
   // sample.go
   package dispatch

   func init() {
     // Base mutator example: supply generic defaults if unset
     RegisterBase(func(spec *v1.VirtualMachineInstanceSpec, ctx *VMIContext) error {
       if spec.Domain.Machine == nil { spec.Domain.Machine = &v1.Machine{Type: "pc"} }
       return nil
     })

     // Hypervisor-wide mutator: refine machine type for sample hypervisor
     RegisterHypervisor("sample", func(spec *v1.VirtualMachineInstanceSpec, ctx *VMIContext) error {
       // only override if user did not specify
       if ctx.Original.Spec.Domain.Machine == nil || ctx.Original.Spec.Domain.Machine.Type == "" {
         spec.Domain.Machine.Type = "sample-q35"
       }
       return nil
     })

     // Optional combined hypervisor+arch mutator
     RegisterHypervisorArch("sample", "amd64", func(spec *v1.VirtualMachineInstanceSpec, ctx *VMIContext) error {
       if ctx.Original.Spec.Domain.CPU == nil { spec.Domain.CPU = &v1.CPU{Model: "sample-zen"} }
       return nil
     })

     // Finalizer: derive CPU topology snapshot
     RegisterFinalizers("sample", func(vmi *v1.VirtualMachineInstance, ctx *VMIContext) error {
       // populate status fields or annotations based on finalized spec
       return nil
     })
   }
   ```

2. **Runtime** – Add `pkg/hypervisor/runtime/sample.go` implementing the `HypervisorRuntime` interface so controllers, handlers, and virt-launcher share runtime hooks.

   ```go
   type sampleRuntime struct{}

   func (sampleRuntime) AdjustResources(_ *v1.VirtualMachineInstance, _ *string) error {
     // reuse existing helpers; return nil when no adjustments are needed
     return nil
   }

   func (sampleRuntime) GetMemoryOverhead(vmi *v1.VirtualMachineInstance, arch string, ratio *string) resource.Quantity {
     return baseMemoryOverhead(vmi, arch, ratio)
   }

   func (sampleRuntime) HandleHousekeeping(_ *v1.VirtualMachineInstance, dom *api.Domain) error {
     // attach timers/watchdogs if the hypervisor requires it
     return nil
   }

   func init() {
     RegisterHypervisorRuntime("sample", sampleRuntime{})
   }
   ```

3. **Converter** – Add `pkg/virt-launcher/virtwrap/converter/hypervisor/sample.go` implementing the `HypervisorConverter` interface (overriding only the methods that differ from the base helper).

   ```go
   type sampleConverter struct {
     *BaseHypervisorConverter
   }

   func newSampleConverter() HypervisorConverter {
     return &sampleConverter{
       BaseHypervisorConverter: NewBaseHypervisorConverter(),
     }
   }

   func (c *sampleConverter) SetDomainType(dom *api.Domain) {
     dom.Spec.Type = "sample"
   }

   func init() {
     RegisterHypervisorConverter("sample", newSampleConverter)
   }
   ```

4. **Admission** – Create `pkg/virt-api/webhooks/validating-webhook/admitters/hypervisor/sample.go` that exports `MutateVMI` and `Validate` functions and register them in the webhook registry.

4. **Node labeller (optional for MVP)** – Provide `pkg/virt-handler/node-labeller/hypervisor/sample.go` declaring the devices and libvirt `virt-type` to probe. If the hypervisor relies on architecture-specific features, add the corresponding helper hooks.

   ```go
   type sampleLabeller struct{}

   func (sampleLabeller) Devices() []DeviceProbe {
     return []DeviceProbe{
       {Path: "/dev/sample", ResourceName: "devices.kubevirt.io/sample"},
     }
   }

   func (sampleLabeller) PreferredVirtType() string {
     return "sample"
   }

   func init() {
     RegisterHypervisorLabeller("sample", sampleLabeller{})
   }
   ```

## Alternatives

1. **Status quo** – Continue duplicating KVM assumptions everywhere. This blocks new hypervisors and increases maintenance burden.
2. **Deep plugin model** – Move domain generation to separate binaries per hypervisor. Rejected for complexity and duplication of KubeVirt control-plane logic.
3. **Libvirt-only configuration** – Attempt to encode all variability via libvirt XML fragments in CRDs. Lacks validation, testing, and integration with device management.

### Future Enhancements

Full abstraction with—multi-device descriptors, richer domain defaults, hypervisor-driven validation, and alternate libvirt transports.

- Extend `virt-handler` node-labeller to surface per-hypervisor capability labels so schedulers and operators can audit readiness without relying on device resources alone.

## Scalability

- Hypervisor selection resolves once per reconcile, and each component performs a constant-time lookup in its registry, keeping control-loop complexity independent of how many implementations ship.
- Converter mutators execute in a deterministic order to avoid combinatorial growth in conditionals.

## Update/Rollback Compatibility

- The feature is additive. Clusters without hypervisor configuration continue to use KVM exclusively.
- Rolling back to a version without the abstraction reverts to the existing KVM default once the new configuration field is removed or ignored.

## Functional Testing Approach

- Unit tests for each hypervisor implementation verifying domain defaults and mutators.
- Integration tests covering virt-controller manifest rendering and device manager plugin registration with other hypervisors enabled.
- End-to-end lanes that launch VMIs under at least two hypervisors (e.g., KVM plus a stub hypervisor) to confirm scheduling and domain generation.

## Implementation History

- 2025-Oct-7: Initial VEP draft.

## Graduation Requirements

### Alpha

- Feature gate covers configurable hypervisor
- Cluster-wide hypervisor configuration implemented and consumed by defaults, converter, and webhooks.
- Basic functional tests for alternative hypervisor scheduling and domain generation.

### Beta

- Monitoring and observability hooks consumed by community dashboards.
- Upgrade/rollback testing executed in CI.

### GA

- Documentation reflects hypervisor lifecycle and contributor workflow.
