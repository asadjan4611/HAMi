# Dynamic MIG CDI Integration Design

## Status

Proposed.

## Summary

HAMi can create NVIDIA Multi-Instance GPU (MIG) GPU Instance (GI) and
Compute Instance (CI) pairs while handling a container allocation. A MIG
device created at that point did not exist when the NVIDIA device plugin
generated its startup Container Device Interface (CDI) specification.

This design keeps CDI state synchronized with the lifecycle of HAMi-managed
dynamic MIG devices. After a GI/CI pair is created or adopted, HAMi publishes
an atomic CDI specification containing the resulting MIG UUID before it
returns the allocation response. Repeated allocations reuse the same valid
entry. When HAMi permanently destroys the MIG device, it removes the entry.

CDI integration remains opt-in through the existing device-list strategy.
The current environment-variable and device-mount injection paths continue to
work without CDI.

## Motivation

A CDI-qualified device name is useful only when the container runtime can
resolve it in a valid CDI specification. The current startup-generated
specification represents devices visible at device-plugin startup. Dynamic MIG
instances are created later, during allocation, so their MIG UUIDs can be
missing from that file.

The incomplete lifecycle is:

```text
Device plugin starts
        |
        v
Startup CDI spec is generated
        |
        v
Allocate creates a new MIG GI/CI and obtains MIG-UUID-X
        |
        v
Allocate returns a CDI name containing MIG-UUID-X
        |
        v
Runtime cannot resolve the name if no CDI entry contains MIG-UUID-X
```

Dynamic MIG needs one ordered transaction:

```text
Reserve placement -> create/reuse GI+CI -> resolve MIG UUID
                                            |
                                            v
                             publish/verify CDI entry
                                            |
                                            v
                               return allocation response
```

## Goals

- Publish a CDI device entry after HAMi creates or adopts a dynamic MIG
  instance.
- Keep a deterministic one-to-one mapping between a CDI device name and a live
  MIG UUID.
- Return the qualified CDI name through the configured CDI allocation
  strategy.
- Reuse a valid entry when an idempotent allocation reuses the same live MIG
  instance.
- Remove the entry when HAMi permanently destroys the corresponding instance.
- Reconstruct CDI state for live, adopted instances after device-plugin
  restart.
- Publish complete CDI files atomically.
- Preserve non-CDI behavior and mixed CDI/non-CDI configurations.
- Make lifecycle behavior deterministic, observable, race-free, and testable
  without requiring GPU hardware for unit tests.

## Non-goals

- Replacing the Kubernetes device-plugin API with Dynamic Resource Allocation
  (DRA).
- Changing scheduler MIG profile or placement selection.
- Creating a general idle-instance pool.
- Changing the lifetime policy of dynamic MIG instances.
- Managing MIG instances created by another controller.
- Changing static MIG CDI behavior.
- Enabling CDI in containerd, CRI-O, or another runtime on behalf of the
  operator.

## Terminology

| Term | Meaning |
| --- | --- |
| Dynamic MIG instance | A GI/CI pair created and owned by HAMi at runtime. |
| MIG UUID | The NVIDIA runtime identity of the created MIG device. |
| Allocation key | Parent GPU, profile, and placement used to identify one HAMi allocation. |
| CDI device name | The device-local name stored in a CDI specification. |
| Qualified name | A complete CDI reference in `vendor/class=device` form. |
| Base spec | The existing CDI specification generated for devices visible at plugin startup. |
| Dynamic spec | A dedicated specification containing only HAMi-managed dynamic MIG entries. |

In this design, "reuse" means that an idempotent allocation finds the same
still-live MIG instance for the same allocation key. Maintaining unused MIG
instances as an idle pool is outside this proposal.

## Existing Architecture

The relevant ownership boundaries are:

- The scheduler reserves the parent GPU, profile, and placement.
- The NVIDIA device plugin owns GI/CI creation and destruction on its node.
- The MIG instance manager tracks allocation keys and live MIG UUIDs.
- The CDI handler generates qualified names and CDI specifications.
- The Allocate response selects CDI annotations, CRI CDI devices, or the
  existing non-CDI injection mechanisms according to configuration.

The CDI handler currently exposes startup specification generation and name
construction. Dynamic MIG lifecycle operations do not update that startup
snapshot.

## Design Principles

### Hardware state is authoritative

A CDI entry must never be published before the GI/CI exists and its MIG UUID
and required device nodes have been resolved successfully.

### Published CDI state is a derived snapshot

The in-memory registry of live HAMi-managed instances is the source for the
dynamic CDI specification. The file is a replaceable snapshot, not an
independent database.

### Ownership is explicit

HAMi writes a dedicated dynamic specification and removes only that file or
entries represented in that file. It does not edit or delete CDI files owned
by the NVIDIA Container Toolkit, GPU Operator, DRA driver, or an administrator.

### Allocation does not succeed before publication

When CDI is required, Allocate must not return a qualified device name until
the corresponding entry has been published successfully.

### Existing paths remain independent

When CDI is disabled, dynamic MIG continues through the existing
environment-variable or mount-based injection path without touching dynamic
CDI state.

## Proposed Architecture

Add a dynamic MIG registry to the CDI handler. The registry stores immutable
records indexed by MIG UUID and serializes specification publication.

```text
                    +----------------------+
MIG manager ------> | Dynamic CDI registry | <------ restart adoption
 create/adopt       | UUID -> device entry |
 destroy            +----------+-----------+
                               |
                               | complete snapshot
                               v
                    +----------------------+
                    | atomic CDI publisher |
                    +----------+-----------+
                               |
                               v
                    /var/run/cdi/<HAMi-owned spec>
                               |
                               v
                    containerd / CRI-O
```

The dynamic specification uses the existing HAMi/NVIDIA CDI vendor and a
dedicated class, for example:

```text
k8s.device-plugin.nvidia.com/dynamic-mig=MIG-xxxxxxxx
```

A separate class gives HAMi clear ownership and prevents a dynamic entry from
colliding with an entry in the startup-generated `gpu` specification. The
exact class is a compatibility-sensitive constant and must be covered by
tests.

## Device Identity and Mapping

The canonical device name is the normalized MIG UUID. The registry record is:

```go
type DynamicMIGDevice struct {
    MIGUUID          string
    ParentGPUUUID    string
    GPUInstanceID    uint32
    ComputeInstanceID uint32
    Profile          string
    PlacementStart   uint32
    PlacementSize    uint32
}
```

The MIG UUID is suitable because it is the runtime identity passed to NVIDIA
device discovery. For the lifetime of a live GI/CI, repeated allocations
produce the same qualified name. Destruction ends that identity. If NVIDIA
assigns a new UUID after recreation, HAMi publishes a new CDI name and removes
the old one.

The allocation key remains useful for idempotent hardware realization, but it
must not be used as a CDI name that silently points to a different MIG UUID
after recreation.

## CDI Specification Ownership

HAMi writes one node-local dynamic specification containing all currently live
HAMi-managed dynamic MIG devices. A suggested filename is:

```text
/var/run/cdi/hami-dynamic-mig.yaml
```

The actual path must use the configured CDI root instead of a hard-coded
directory. A single snapshot avoids one file per allocation, makes restart
reconciliation simple, and bounds filesystem operations.

The file contains:

- one vendor/class pair owned by HAMi;
- common NVIDIA container edits generated by the existing NVIDIA CDI library;
- one device entry per live MIG UUID;
- parent GPU and MIG GI/CI device nodes required by the runtime.

Specification construction should reuse the NVIDIA Container Toolkit CDI
library already used by HAMi. If toolkit discovery for a newly created MIG
device does not return all GI/CI capability nodes, the provider must derive
those nodes from the resolved GI/CI runtime information, following the
dynamic-MIG approach used by the NVIDIA DRA driver. Hand-written device edits
should be limited to data the toolkit cannot provide.

## Atomic Publication

Every add, refresh, or remove operation rebuilds the specification from one
locked registry snapshot and publishes it atomically:

1. Copy and validate the in-memory registry while holding its mutex.
2. Generate the complete CDI specification from the snapshot.
3. Validate the specification with the CDI library.
4. Write a temporary file in the configured CDI directory.
5. Close the file successfully.
6. Atomically rename it over the final file.
7. Remove the temporary file after any failure.

The implementation should prefer the CDI library's atomic `WriteSpec` or
`Save` operation when the pinned library version provides the required
temporary-file-and-rename guarantee. A second custom writer should not be
introduced unless the library cannot meet the contract.

When the registry becomes empty, remove the HAMi-owned dynamic spec. A missing
file is treated as success. HAMi must never remove another producer's file.

## Interface Changes

Extend the internal CDI interface with lifecycle-focused methods. The precise
names may follow the package convention, but the semantic contract should be:

```go
type Interface interface {
    CreateSpecFile() error
    QualifiedName(class, id string) string
    AdditionalDevices() []string

    EnsureDynamicMIGDevice(DynamicMIGDevice) (qualifiedName string, err error)
    RemoveDynamicMIGDevice(migUUID string) error
    ReplaceDynamicMIGDevices([]DynamicMIGDevice) error
}
```

- `EnsureDynamicMIGDevice` is idempotent. It validates an existing record or
  publishes an updated snapshot.
- `RemoveDynamicMIGDevice` is idempotent and removes only the matching UUID.
- `ReplaceDynamicMIGDevices` is used during startup recovery so the published
  state exactly matches adopted live instances.
- The null CDI handler implements these methods as no-ops only when CDI is not
  configured. Callers must not request a qualified name from the null handler.

Keeping the filesystem and spec-generation details behind the CDI interface
allows unit tests to use an in-memory fake and prevents the MIG manager from
depending on CDI file formats.

## Configuration and Capability Detection

No new default behavior is required. Dynamic MIG CDI synchronization is active
only when all of the following are true:

1. The NVIDIA plugin operates in dynamic MIG mode.
2. At least one configured device-list strategy uses CDI.
3. The CDI handler and configured CDI directory are available.

The existing strategy remains the operator-facing switch:

- `cdi-annotations`: return CDI annotations.
- `cdi-cri`: return CRI `CDIDevice` entries.
- `envvar` or `volume-mounts`: preserve the non-CDI path.
- mixed strategy: produce both configured response forms.

Startup validation should fail with a clear error when CDI-only operation is
requested but CDI cannot be initialized. When CDI is not selected, lack of CDI
runtime support must not prevent the device plugin from starting.

Runtime capability cannot be proven only by checking a directory. Operators
remain responsible for enabling CDI in containerd or CRI-O. HAMi should log
the selected strategy, CDI root, dynamic-spec path, and whether dynamic MIG
CDI synchronization is active.

## Lifecycle Workflows

### Create

```text
Allocate request
      |
      v
Ensure scheduler reservation is valid
      |
      v
Create GI/CI and resolve MIG UUID
      |
      v
Build and atomically publish CDI entry
      |
      v
Record runtime MIG information
      |
      v
Return CDI-qualified name
```

If publication fails for a newly created instance, the allocation fails and
the existing partial-allocation rollback destroys that GI/CI. No qualified
name is returned.

### Reuse

When `EnsureAllocation` finds a tracked live instance:

1. Obtain its MIG UUID and runtime information.
2. Ask the CDI handler to ensure the entry exists and matches that UUID.
3. Reuse the existing entry without rewriting the file if the in-memory record
   and last published generation are unchanged.
4. Republish when the entry is missing, stale, or not yet recovered.

This fast path avoids unnecessary disk writes on repeated Allocate calls.

### Destroy

For permanent destruction:

1. Serialize against allocation on the same physical GPU.
2. Destroy the CI and GI through NVML.
3. Remove the manager's live allocation record.
4. Remove the MIG UUID from the CDI registry and atomically publish the new
   snapshot.

If CDI removal fails after hardware destruction, retain a reconciliation
marker and retry. A stale entry cannot provide access to destroyed hardware,
but it must not remain indefinitely or be reused for a different identity.

### Allocation rollback

Rollback follows the same destruction path. It must remove a CDI entry only
for a MIG instance created by the failed allocation. It must not remove a
shared or previously existing live entry.

### Restart recovery

After the plugin adopts live MIG instances from Pod allocation records and
NVML:

1. Build a complete list of successfully adopted runtime identities.
2. Replace the dynamic CDI registry with that list.
3. Publish one atomic snapshot.
4. Remove entries for instances that no longer exist.
5. Begin serving Allocate only after synchronization succeeds when CDI-only
   operation is configured.

Recovery is replace-based instead of a sequence of individual additions, so a
restart cannot preserve entries from an earlier process accidentally.

## Transaction and Failure Semantics

| Failure | Required result |
| --- | --- |
| GI creation fails | Return allocation error; publish nothing. |
| CI creation fails | Destroy partial GI; publish nothing. |
| MIG UUID discovery fails | Destroy newly created GI/CI; publish nothing. |
| Device-spec generation fails | Fail allocation and roll back newly created GI/CI. |
| Atomic CDI write fails | Keep the previous complete file; fail CDI allocation. |
| CDI refresh fails for reused live MIG | Keep hardware; fail this allocation and retry later. |
| Hardware destruction fails | Keep CDI entry because the device may still be live. |
| CDI removal fails after successful destruction | Record/log failure and retry reconciliation. |
| Pod annotation update fails after creation | Existing allocation rollback removes newly created hardware and CDI state. |

The publication API must distinguish a newly created instance from a reused
instance so error rollback never destroys an instance owned by an earlier
successful allocation.

## Concurrency and Lock Ordering

CDI lifecycle updates can race with concurrent Allocate calls, periodic
reconciliation, startup adoption, and rollback. Use two lock domains:

- the existing per-GPU MIG mutation lock protects GI/CI state;
- one CDI registry mutex protects registry generations and file publication.

The required lock order is:

```text
per-GPU MIG lock -> CDI registry lock
```

No CDI method may call back into the MIG manager while holding the CDI lock.
This fixed order prevents lock inversion. The registry mutex remains held
through publication so an older snapshot cannot overwrite a newer snapshot.

Each successful mutation increments an in-memory generation. An idempotent
ensure can skip publication when its record is identical and the current
generation was published successfully.

## Allocation Response

The dynamic path returns the qualified name produced by
`EnsureDynamicMIGDevice`, rather than independently reconstructing it. This
couples the returned identity to the entry that was actually published.

For CDI annotations, add that name through the existing annotation helper. For
CRI CDI, append it to `ContainerAllocateResponse.CDIDevices`. Additional CDI
devices such as IMEX remain unchanged.

The non-CDI path continues to receive the raw MIG UUID and existing response
edits. Static full-GPU and static MIG allocations retain their existing
`gpu`-class behavior.

## Security and File Safety

- Accept only validated NVIDIA GPU and MIG UUID formats as names.
- Do not allow UUIDs or configuration values to become filesystem paths.
- Write only below the configured CDI root.
- Use predictable final filenames but unpredictable temporary filenames.
- Use normal CDI file permissions; do not make specifications writable by
  workload containers.
- Validate the complete spec before replacement.
- Never follow an externally controlled path outside the CDI root.
- Do not include Pod secrets or namespace data in the node-local spec.
- Log UUIDs and lifecycle actions, but not full Pod specifications.

## Observability

Add structured logs for:

- dynamic CDI synchronization enabled or disabled;
- entry added, reused, refreshed, and removed;
- snapshot generation and entry count;
- publication duration and failure;
- restart reconstruction count;
- deferred stale-entry cleanup.

Recommended metrics are:

```text
hami_dynamic_mig_cdi_entries
hami_dynamic_mig_cdi_updates_total{operation,result}
hami_dynamic_mig_cdi_update_duration_seconds
hami_dynamic_mig_cdi_reconcile_errors_total
```

Metric labels must remain bounded. MIG UUID, Pod, and container names must not
be metric labels.

## Compatibility

This proposal is backward-compatible:

- CDI remains disabled unless selected by configuration.
- Existing `envvar` and `volume-mounts` behavior is unchanged.
- Mixed strategies remain supported.
- Existing static GPU and static MIG CDI entries remain in their current
  specification and class.
- Pod resource requests and scheduler annotations do not change.
- No Kubernetes API or CRD is added.

Changing the dynamic CDI class or device-name format after release would break
stored runtime references. Both must therefore be constants with compatibility
tests.

## Testing Strategy

### Unit tests without GPU hardware

- Adding the first MIG record creates a valid specification.
- Adding multiple records produces one complete specification.
- Re-adding an identical record returns the same qualified name and avoids an
  unnecessary write.
- Replacing a conflicting UUID record does not preserve the old name.
- Removing a record republishes the remaining entries.
- Removing the final record removes the HAMi-owned spec.
- Missing-file removal succeeds.
- Invalid UUIDs and invalid generated specs are rejected.
- A failed write leaves the previous complete file readable.
- Concurrent ensure/remove operations pass `go test -race` and the final file
  matches the final registry generation.
- Startup replacement removes stale entries and retains adopted live entries.
- The null handler remains safe when CDI is disabled.
- CDI annotation and CRI response strategies receive the published qualified
  name.
- Non-CDI and mixed strategies preserve their current response data.
- Creation failure and publication failure exercise allocation rollback.
- Destruction failure retains the CDI record.

Use a temporary CDI directory, an injected specification provider, a fake
atomic writer where failure injection is required, and the existing mock CDI
interface. No NVML or GPU is required for these tests.

### Component tests without GPU hardware

- Run the device-plugin allocation path with fake MIG manager and fake CDI
  provider.
- Validate generated YAML/JSON with the pinned CDI library.
- Start a CDI cache against the temporary directory and resolve every returned
  qualified name.
- Simulate device-plugin restart and adoption.
- Run lifecycle tests repeatedly and with the Go race detector.

### Required hardware validation before upstream submission

Because this changes device allocation and in-container injection, final
upstream validation requires a MIG-capable NVIDIA GPU and CDI-enabled runtime.
Record GPU model, driver, NVIDIA Container Toolkit, runtime, Kubernetes, and
HAMi versions.

Validate:

- new GI/CI creation followed by a CUDA workload;
- repeated Allocate reuse and stable qualified name;
- multiple profiles on one physical GPU;
- cleanup after Pod completion;
- device-plugin restart with a running workload;
- recreation after destruction and removal of the old UUID;
- CDI annotations and CRI CDI strategies supported by the test environment;
- the legacy non-CDI path;
- concurrent Pod allocation and deletion;
- absence of partial-spec parse errors in kubelet and runtime logs.

## Rollout Plan

1. Add the lifecycle interface, in-memory registry, atomic writer, and unit
   tests without changing allocation behavior.
2. Connect dynamic MIG create/reuse to CDI ensure and return the published
   qualified name.
3. Connect rollback, destruction, and reconciliation to removal.
4. Add restart replacement from adopted allocations.
5. Add logs and bounded metrics.
6. Validate with a CDI-enabled runtime and real MIG hardware.
7. Document operator prerequisites and troubleshooting after hardware results
   are known.

## Alternatives Considered

### Regenerate the startup GPU specification after every change

This is simple but mixes static discovery with HAMi-owned dynamic lifecycle,
increases collision risk, and makes cleanup ownership unclear. A dedicated
dynamic specification is safer.

### One CDI file per MIG UUID

This gives simple deletion but creates more filesystem churn and stale-file
cleanup work. A single atomic snapshot is easier to reconcile and bound.

### Use allocation key as the CDI name

An allocation key can survive hardware recreation while the MIG UUID changes.
That can make one CDI name silently refer to a new hardware identity. Naming by
MIG UUID makes identity changes explicit.

### Return the qualified name before writing the specification

This reduces allocation latency but creates a race in which the runtime reads
a name that does not exist. Publication must complete first.

### Require CDI and remove the legacy path

This would break clusters whose runtime is not CDI-enabled. CDI remains an
explicit strategy and the legacy path is preserved.

### Copy the NVIDIA DRA implementation directly

The DRA driver owns ResourceClaims and can use per-claim transient specs. HAMi
uses the device-plugin API and a different reservation lifecycle. HAMi should
reuse the CDI lifecycle principles and toolkit behavior, not the DRA ownership
model.

## Risks and Mitigations

| Risk | Mitigation |
| --- | --- |
| Runtime reads a partial file | CDI library atomic write/rename. |
| Stale entry survives restart | Replace registry from adopted live instances before serving. |
| Concurrent snapshots overwrite each other | One registry mutex and monotonic generation. |
| CDI failure leaks new hardware | Integrate with existing newly-created allocation rollback. |
| Cleanup removes another producer's spec | Dedicated HAMi-owned filename and class. |
| Duplicate device names across specs | Dedicated class and UUID-based names. |
| CDI toolkit misses MIG capability nodes | Injected provider with explicit GI/CI node fallback and hardware tests. |
| Metrics create high cardinality | Never label metrics with UUIDs or Pod identity. |
| Legacy deployments regress | Gate all new behavior on the existing CDI strategy. |

## Open Questions

1. Confirm the final vendor, class, and filename constants with maintainers.
2. Confirm which pinned NVIDIA Container Toolkit versions return complete
   device edits for newly created MIG UUIDs.
3. Decide whether a CDI cleanup failure after successful hardware destruction
   should make reconciliation unhealthy or remain a retryable warning.
4. Confirm support expectations for `cdi-annotations`, `cdi-cri`, or both in
   dynamic MIG mode.
5. Decide whether an operator-visible readiness condition is needed when CDI
   reconciliation cannot converge.
6. Confirm whether a future idle MIG pool is planned; it is not introduced by
   this design.

## Acceptance Criteria

- A newly created dynamic MIG device has a valid CDI entry before Allocate
  returns its qualified name.
- The name resolves to the correct live MIG UUID and required device nodes.
- Reusing the same live instance returns the same name without unnecessary
  file writes.
- Destroying an instance removes its entry without touching other CDI files.
- Restart recovery reconstructs exactly the set of adopted live instances.
- All specification replacements are atomic.
- Concurrent lifecycle operations pass race-detector tests.
- CDI-only startup and allocation failures produce clear errors.
- Non-CDI and mixed strategies retain their documented behavior.
- Unit and component tests pass without GPU hardware.
- Real MIG hardware validation is recorded before an upstream implementation
  is submitted.

## References

- [HAMi dynamic MIG design](./dynamic-mig.md)
- [Container Device Interface specification](https://github.com/cncf-tags/container-device-interface)
- [NVIDIA DRA Driver for GPUs](https://github.com/kubernetes-sigs/dra-driver-nvidia-gpu)
- [NVIDIA DRA Driver releases](https://github.com/kubernetes-sigs/dra-driver-nvidia-gpu/releases)
