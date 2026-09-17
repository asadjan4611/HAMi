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
- Publish each MIG CDI file atomically.
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

## CDI Background

### What CDI is and how it works

The Container Device Interface (CDI) is a vendor-neutral specification for
describing container devices. It separates two responsibilities:

- a device owner describes the files, mounts, environment variables, and hooks
  needed to use a device;
- a CDI-enabled container runtime resolves a qualified device name and applies
  those edits to the container's Open Container Initiative (OCI) specification.

A qualified CDI name has this form:

```text
vendor/class=device
```

For example:

```text
k8s.device-plugin.nvidia.com/dynamic-mig=mig-<stable-UUID-hash>
```

CDI specifications normally live in runtime-watched directories such as
`/etc/cdi` and `/var/run/cdi`. The runtime loads and caches those files. When
kubelet asks it to inject a qualified name, the runtime finds the matching
vendor/class and device entry, then applies the entry's container edits.

```text
Device owner writes CDI spec
             |
             v
Runtime watches CDI directory
             |
             v
Kubelet passes vendor/class=device
             |
             v
Runtime resolves device entry
             |
             v
Runtime applies OCI edits and starts container
```

CDI does not create hardware and does not schedule it. In this design, NVML
creates the MIG GI/CI, HAMi publishes its description, and the runtime performs
container injection.

### What a CDI specification contains

A CDI specification is a JSON or YAML document with:

- `cdiVersion`, identifying the schema version;
- `kind`, containing the vendor and class;
- optional top-level `containerEdits`, applied whenever any device from the
  specification is requested;
- `devices`, containing a name and device-specific edits for each device.

A simplified dynamic MIG specification looks like this:

```yaml
cdiVersion: "0.8.0"
kind: "k8s.device-plugin.nvidia.com/dynamic-mig"
containerEdits:
  env:
    - "NVIDIA_VISIBLE_DEVICES=void"
  hooks:
    - hookName: createContainer
      path: /usr/bin/nvidia-cdi-hook
      args:
        - nvidia-cdi-hook
        - create-symlinks
        - --link
        - ../libnvidia-ml.so.1::/usr/lib/libnvidia-ml.so
devices:
  - name: "mig-<stable-UUID-hash>"
    containerEdits:
      deviceNodes:
        - path: /dev/nvidia0
        - path: /dev/nvidia-caps/nvidia-cap42
        - path: /dev/nvidia-caps/nvidia-cap43
```

This example is illustrative, not a hard-coded output template. The pinned
NVIDIA Container Toolkit and CDI libraries determine the actual schema version,
common edits, paths, permissions, mounts, and hooks. HAMi supplies the concrete
parent GPU and GI/CI capability nodes for the live instance and validates the
result before publication.

The device name is local to its `kind`. Combining the example kind and device
name produces:

```text
k8s.device-plugin.nvidia.com/dynamic-mig=mig-<stable-UUID-hash>
```

### How a Kubernetes device plugin uses CDI

The device plugin remains responsible for advertising capacity and deciding
which concrete device satisfies an Allocate request. CDI changes the delivery
part of Allocate:

1. The plugin prepares or selects the concrete device.
2. The plugin ensures a valid CDI entry exists for it.
3. The plugin returns the qualified name to kubelet.
4. Kubelet forwards that name to a CDI-enabled runtime.
5. The runtime resolves the CDI spec and injects the device.

Kubernetes supports two response forms used by HAMi's existing configuration:

- CDI CRI: place qualified names in
  `ContainerAllocateResponse.CDIDevices`;
- CDI annotations: encode qualified names in the configured CDI annotation for
  runtimes that use the annotation path.

The CDI file must be published before Allocate returns. Returning a name first
creates a race in which kubelet or the runtime cannot resolve it.

### NVIDIA legacy injection and CDI mode

Both modes can use NVIDIA Container Toolkit components, but they select and
describe devices differently.

| Area | Legacy NVIDIA injection | CDI injection |
| --- | --- | --- |
| Device request | Environment variables such as `NVIDIA_VISIBLE_DEVICES`, plus mounts/device specs when configured | Qualified CDI name such as `vendor/class=device` |
| Device description | NVIDIA runtime/toolkit discovers and applies edits from the legacy request at container creation | A generated CDI document declares the required OCI edits |
| Runtime dependency | NVIDIA-aware runtime configuration or hooks process the legacy request | Container runtime must support CDI and watch the configured CDI directories |
| Allocate response | Environment variables, mounts, and/or device nodes | `CDIDevices` or CDI annotations |
| Dynamic update requirement | Raw MIG UUID can be passed through the legacy NVIDIA path | A resolvable CDI entry must exist before the qualified name is returned |

CDI mode does not mean that the NVIDIA Container Toolkit is removed. HAMi uses
the toolkit to generate correct NVIDIA-specific edits, and a generated spec can
invoke `nvidia-cdi-hook`. CDI standardizes how the selected device and its edits
are handed to the container runtime.

HAMi must preserve the legacy path because existing clusters may not have CDI
enabled in their runtime. Selecting a CDI device-list strategy explicitly
enables the new synchronization path; selecting only a legacy strategy leaves
the existing behavior unchanged.

### How the NVIDIA DRA Driver implements Dynamic MIG

The NVIDIA DRA Driver uses Kubernetes Dynamic Resource Allocation rather than
the traditional device-plugin Allocate API. Its high-level lifecycle is:

```text
ResourceClaim allocation
        |
        v
DRA kubelet plugin prepares claimed device
        |
        v
Create or resolve dynamic MIG instance
        |
        v
Build a claim-scoped transient CDI spec
        |
        v
Checkpoint prepared device and CDI device IDs
        |
        v
Kubelet/runtime inject qualified CDI device
        |
        v
Unprepare claim -> remove CDI spec -> destroy MIG when no longer shared
```

Important implementation ideas used as references by this design are:

- preparation resolves concrete hardware before publishing CDI state;
- reusable NVIDIA CDI fragments are cached to reduce repeated discovery work;
- dynamic MIG entries combine toolkit-generated parent/common edits with
  explicit GI and CI capability device nodes;
- a transient CDI specification is tied to the owning ResourceClaim;
- prepared-device state and returned CDI device IDs are checkpointed for
  recovery;
- Unprepare removes the claim CDI specification and destroys the dynamic MIG
  device only when it is no longer used;
- startup cleanup reconciles hardware and checkpoint state instead of trusting
  leftover files.

HAMi cannot copy that lifecycle directly. HAMi uses Pod annotations, the
device-plugin API, and its MIG instance manager instead of ResourceClaims and
DRA Prepare/Unprepare calls. This design therefore adapts the same principles:
publish after hardware realization, keep durable ownership information,
reconcile at restart, remove CDI state with hardware lifecycle, and never
return an unresolved CDI name.

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

The implementation targets the topology-aware dynamic MIG path in:

- `pkg/device-plugin/nvidiadevice/nvinternal/plugin/migmgr.go`;
- `pkg/device-plugin/nvidiadevice/nvinternal/plugin/util.go`;
- `pkg/device-plugin/nvidiadevice/nvinternal/plugin/server.go`;
- `pkg/device-plugin/nvidiadevice/nvinternal/cdi/`;
- `cmd/device-plugin/nvidia/plugin-manager.go`.

The legacy geometry/template implementation is not a second integration
target. The CDI lifecycle is attached to the manager that owns concrete
profile-and-placement allocations.

## Design Principles

### Hardware state is authoritative

A CDI entry must never be published before the GI/CI exists and its MIG UUID
and required device nodes have been resolved successfully.

### Per-device files are derived from live hardware

The MIG manager's live allocation records are the runtime source of truth.
Each HAMi-owned CDI file describes exactly one MIG UUID. The file is not an
independent database: startup recovery compares files with verified live
instances, while runtime reconciliation deletes only files for instances it
actually destroyed. Runtime reconciliation must never replace a full file set
from a possibly stale snapshot.

### Ownership is explicit

Dynamic MIG mode requires exclusive MIG lifecycle ownership for each managed
physical GPU. HAMi must be the only controller creating, reconfiguring, or
destroying GI/CI instances on that GPU. The ownership boundary is **per GPU**:
one physical GPU cannot be shared between HAMi dynamic MIG and another MIG
controller. The current plugin selects its operating mode per node, so an
installation using the existing node-wide dynamic MIG mode must give HAMi
exclusive control of the MIG lifecycle on all GPUs managed by that plugin.
Per-GPU mixed static/dynamic configuration would require an explicit future
configuration and discovery contract; this design does not enable it.
Operators must not run NVIDIA MIG Manager or another dynamic MIG controller
against HAMi-owned GPUs.

HAMi writes a dedicated file per managed MIG UUID and removes only those files.
It does not edit or delete CDI files owned
by the NVIDIA Container Toolkit, GPU Operator, DRA driver, or an administrator.

### Allocation does not succeed before publication

When CDI is required, Allocate must not return a qualified device name until
the corresponding entry has been published successfully.

### Existing paths remain independent

When CDI is disabled, dynamic MIG continues through the existing
environment-variable or mount-based injection path without touching dynamic
CDI state.

## Correctness Invariants

The implementation is complete only while all of these remain true:

1. Every returned dynamic CDI name exists in the last successfully published
   specification and identifies the MIG UUID returned for that allocation.
2. A CDI name never changes to refer to another MIG UUID.
3. A failed create publishes no entry, and a failed publication returns no CDI
   name.
4. An Allocate request returns no CDI names unless every requested MIG entry
   was ensured; rollback removes only entries and hardware it created.
5. Hardware destruction is followed by removal of the corresponding CDI file;
   failed removal remains retryable and observable.
6. Startup state is reconstructed from verified live hardware and active Pod
   records, not trusted from a leftover CDI file.
7. CDI-disabled allocation never depends on CDI initialization, files, or
   synchronization.
8. HAMi mutates only its own dynamic CDI files.
9. At runtime, no operation replaces all dynamic MIG CDI files from a full
   snapshot; full replacement is restricted to startup before Allocate serves.

## Proposed Architecture

Add per-MIG file operations to the CDI handler. A small lifecycle coordinator
in the NVIDIA plugin orders manager operations, CDI synchronization,
allocation response construction, and rollback without making the MIG manager
depend on CDI file formats. There is no node-wide runtime registry transaction.

```text
                    +----------------------+
MIG manager ------> | CDI lifecycle handler |
 create/adopt       | ensure/remove by UUID |
 destroy            +----------+-----------+
                               |
                               | atomic per-device write
                               v
                    /var/run/cdi/hami-dynamic-mig-<stable-UUID-hash>.json
                               |
                               v
                    containerd / CRI-O
```

The dynamic specification uses the existing HAMi/NVIDIA CDI vendor and a
dedicated class, for example:

```text
k8s.device-plugin.nvidia.com/dynamic-mig=mig-<stable-UUID-hash>
```

A separate class gives HAMi clear ownership and prevents a dynamic entry from
colliding with an entry in the startup-generated `gpu` specification. The
exact class is a compatibility-sensitive constant and must be covered by
tests.

In dynamic MIG mode, the base `gpu` specification must exclude MIG devices.
Otherwise a MIG device present during a plugin restart could be published once
by startup discovery and again by its dynamic per-UUID file, and the startup entry
could become stale after destruction. Full GPUs and statically managed devices
retain the existing base-spec behavior outside dynamic MIG mode.

## Device Identity and Mapping

The hardware identity is the exact MIG UUID returned by NVML. UUIDs are
treated as opaque identities and are not case-folded or rewritten. Some valid
MIG UUID formats contain `/`, which CDI device names forbid. Therefore the
CDI device name and filename use a deterministic `mig-` plus a SHA-256 digest
of the exact UUID. The file remains associated with the original UUID, and
repeated requests for that UUID produce the same name. The CDI input record is:

```go
type DynamicMIGDevice struct {
    MIGUUID          string
    ParentGPUUUID    string
    ParentMinor      int
    GPUInstanceID    uint32
    ComputeInstanceID uint32
}
```

The MIG UUID is the runtime identity passed to NVIDIA device discovery. For
the lifetime of a live GI/CI, repeated allocations produce the same qualified
name. Destruction ends that identity. If NVIDIA assigns a new UUID after
recreation, HAMi publishes a new CDI name and removes the old one.

The allocation key remains useful for idempotent hardware realization, but it
must not be used as a CDI name that silently points to a different MIG UUID
after recreation.

The hash-derived filename contains only safe characters; untrusted UUID text
cannot escape the configured CDI root.

## CDI Specification Ownership

HAMi writes one CDI specification per live, HAMi-managed MIG UUID. A file is
named from a validated UUID, for example:

```text
/var/run/cdi/hami-dynamic-mig-<stable-UUID-hash>.json
```

The actual path uses the configured CDI root instead of a hard-coded
directory. The number of MIG instances on a node is hardware-bounded. Each
create or destroy touches only its own small file, so concurrent allocations
cannot overwrite one another's CDI entries.

The file contains:

- one vendor/class pair owned by HAMi;
- common NVIDIA container edits generated by the existing NVIDIA CDI library;
- one device entry for that MIG UUID;
- parent GPU and MIG GI/CI device nodes required by the runtime.

Specification construction reuses the NVIDIA Container Toolkit CDI library
already used by HAMi. For a dynamic MIG entry, it obtains common edits and the
parent GPU edit from the toolkit, then adds the GI and CI capability device
nodes derived from the resolved parent minor, GI ID, and CI ID. It does not
depend on `GetDeviceSpecsByID(MIG_UUID)` returning a complete MIG spec. This
follows the dynamic-MIG approach used by the NVIDIA DRA driver and makes the
required device-node set explicit.

## Atomic Publication

For each MIG UUID independently:

1. Generate and validate its complete CDI specification.
2. Write a temporary file in the configured CDI directory.
3. Close the file successfully and atomically rename it over the final file.
4. Remove the temporary file after any failure; preserve an existing valid
   final file.

An idempotent ensure verifies that an existing file still describes the
expected UUID and device edits. If it is absent, invalid, or stale, ensure
rebuilds that file. No process-global registry is needed to decide whether an
entry exists. For one Allocate involving multiple MIG devices, ensure files
one by one and return no Allocate response until every ensure succeeds.
Rollback removes only the newly created files and GI/CI instances; preexisting
live entries remain untouched.

The implementation should prefer the CDI library's atomic `WriteSpec` or
`Save` operation when the pinned library version provides the required
temporary-file-and-rename guarantee. A second custom writer should not be
introduced unless the library cannot meet the contract.

Removal deletes only the validated, HAMi-owned filename for the destroyed MIG
UUID. A missing file is treated as success. HAMi must never remove another
producer's file.

## Interface Changes

Extend the internal CDI interface with lifecycle-focused methods. The precise
names may follow the package convention, but the semantic contract should be:

```go
type Interface interface {
    CreateSpecFile() error
    QualifiedName(class, id string) string
    AdditionalDevices() []string

    EnsureDynamicMIGDevice(DynamicMIGDevice) (string, error)
    RemoveDynamicMIGDevice(string) error
    ReplaceDynamicMIGDevices([]DynamicMIGDevice) error
}
```

- `EnsureDynamicMIGDevice` validates or publishes only the specified UUID's
  file and returns its qualified name.
- `RemoveDynamicMIGDevice` removes only the specified UUID's file.
- `ReplaceDynamicMIGDevices` is **startup-only**, before the plugin serves
  Allocate. It ensures files for verified adopted live instances and removes
  stale HAMi-owned files. It must not be called during runtime reconciliation.
- The null CDI handler implements these methods as no-ops only when CDI is not
  configured. Callers must not request a qualified name from the null handler.

Keeping the filesystem and spec-generation details behind the CDI interface
allows unit tests to use an in-memory fake and prevents the MIG manager from
depending on CDI file formats.

The one-file-per-UUID interface keeps unrelated MIG allocations independent.
An Allocate response remains all-or-nothing even though filesystem publication
occurs per file: partial success must be rolled back before returning an error.

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

The CDI handler must receive dynamic-MIG mode as an explicit construction
option. In that mode, startup base-spec generation excludes MIG entries and
the plugin follows this order before registering with kubelet:

1. Reset only GPUs proven idle by the existing fail-closed startup logic.
2. Adopt live dynamic MIG allocations from Pod annotations and NVML.
3. Reconcile HAMi-owned per-MIG CDI files with the verified adopted live
   records; remove stale files and ensure missing files.
4. Generate or verify the non-dynamic base CDI specifications.
5. Start and register the device-plugin server.

CDI-only startup fails before registration if step 3 cannot publish a valid
snapshot. In a mixed CDI and legacy strategy, the default is also fail-closed:
the operator explicitly selected CDI, so silently returning only the legacy
path would produce configuration-dependent behavior. A future fallback option
would require a separate proposal.

Runtime capability cannot be proven only by checking a directory. Operators
remain responsible for enabling CDI in containerd or CRI-O. HAMi should log
the selected strategy, CDI root, dynamic-spec filename prefix, and whether dynamic MIG
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
Create all required GI/CI pairs and resolve their MIG UUIDs
      |
      v
Atomically publish one CDI file per MIG UUID
      |
      v
Record runtime MIG information
      |
      v
Return CDI-qualified name
```

If any hardware creation or CDI publication fails, the allocation fails.
Rollback removes CDI files and GI/CI pairs created by this request, including
earlier successful entries of a multi-device request. Reused instances and
their CDI files are never destroyed by that rollback. No qualified names are
returned until every requested file is valid.

### Reuse

When `EnsureAllocation` finds a tracked live instance:

1. Obtain its MIG UUID and runtime information.
2. Ask the CDI handler to ensure the entry exists and matches that UUID.
3. Reuse the existing file without rewriting it if it is valid for that live
   UUID and its required device edits. Lazy recycling retains this file for as
   long as the underlying GI/CI remains alive.
4. Regenerate only that UUID's file when it is missing or stale.

This fast path avoids unnecessary disk writes on repeated Allocate calls.

### Destroy

For permanent destruction:

1. Serialize against allocation on the same physical GPU.
2. Destroy the CI and GI through NVML.
3. Remove the manager's live allocation record.
4. Remove only the corresponding HAMi-owned CDI file.

If CDI removal fails after hardware destruction, retain the UUID for an
incremental cleanup retry; do not replace all files from a manager snapshot.
A stale entry cannot provide access to destroyed hardware, but it must not
remain indefinitely or be reused for a different identity.

### Allocation rollback

Rollback follows the same destruction path. It must remove a CDI entry only
for a MIG instance created by the failed allocation. It must not remove a
shared or previously existing live entry.

### Restart recovery

After the plugin adopts live MIG instances from Pod allocation records and
NVML, and before it registers with kubelet:

1. Build a complete list of successfully adopted runtime identities.
2. Ensure a valid per-MIG CDI file for each adopted UUID.
3. Remove stale HAMi-owned per-MIG files that have no live adopted instance.
4. Do not touch CDI files owned by other producers.
5. Begin serving Allocate only after synchronization succeeds when CDI-only
   operation is configured.

This is the only phase that uses `ReplaceDynamicMIGDevices`. It runs before
Allocate starts serving, so no concurrent new allocation can be removed by a
stale startup snapshot. A failed or incomplete hardware/Pod inventory must
fail closed and must not authorize deletion of existing files.

## Transaction and Failure Semantics

| Failure | Required result |
| --- | --- |
| GI creation fails | Return allocation error; publish nothing. |
| CI creation fails | Destroy partial GI; publish nothing. |
| MIG UUID discovery fails | Destroy newly created GI/CI; publish nothing. |
| Device-spec generation fails | Fail allocation and roll back newly created GI/CI. |
| Atomic CDI write fails | Keep the previous complete file for that UUID; fail CDI allocation. |
| CDI refresh fails for reused live MIG | Keep hardware; fail this allocation and retry later. |
| Hardware destruction fails | Keep CDI entry because the device may still be live. |
| CDI removal fails after successful destruction | Retain UUID for incremental cleanup retry. |
| Pod annotation update fails after creation | Existing allocation rollback removes newly created hardware and CDI state. |

The publication API must distinguish a newly created instance from a reused
instance so error rollback never destroys an instance owned by an earlier
successful allocation.

## Concurrency and Lock Ordering

CDI lifecycle updates can race with concurrent Allocate calls, periodic
reconciliation, and rollback. The existing device-plugin `applyMutex`
serializes Allocate and periodic reconciliation for one plugin instance.
The MIG manager keeps its per-GPU hardware locks. The CDI handler serializes
filesystem operations only for the **same UUID**; independent files require
no global publication lock.

The coordinator executes operations in this order:

```text
device-plugin allocation lock
        |
        +--> MIG manager call (takes and releases its per-GPU lock)
        |
        +--> CDI ensure/remove for that UUID
```

The manager's per-GPU lock is not held during CDI filesystem I/O. The
existing plugin lock stays held across both calls, so create and destroy for
the same logical allocation cannot pass each other. This preserves the
current plugin serialization policy rather than introducing a second
allocation lock order.

No CDI method may call the MIG manager, and no manager method may call the CDI
handler. Periodic reconciliation must return the UUIDs of instances it
**actually destroyed**, then remove only those CDI files through the
coordinator. A stale list of desired allocations must never trigger a runtime
full replacement. Before deleting a file after reconciliation, recheck under
the allocation lock that the UUID has not been reused or adopted by another
allocation. A missing, truncated, externally replaced, or invalid file is
repaired only by an ensure for its live UUID.

## Allocation Response

The dynamic path ensures every requested entry before building a response.
The response derives each qualified name from the same deterministic UUID
mapping used by `EnsureDynamicMIGDevice`; no name is returned if any ensure
fails. A test must check that the response name resolves to the published
entry.

The current response helper accepts raw device IDs and qualifies them as the
`gpu` class. Dynamic MIG must select the dedicated class and CDI-safe name
from the raw MIG UUID. It must not pass an already qualified name through
that path, because doing so would qualify it twice. The existing raw-ID path
remains unchanged for full GPUs and static MIG devices outside dynamic mode.

For CDI annotations, add those names through the existing annotation helper.
For CRI CDI, append them to `ContainerAllocateResponse.CDIDevices`. Additional
CDI devices such as IMEX remain unchanged.

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
- per-device publication and live entry count;
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

- Adding the first MIG record creates a valid per-UUID specification.
- Adding multiple records produces independent valid files.
- A multi-device Allocate returns names only after every per-UUID ensure
  succeeds; partial failure rolls back only new instances and files.
- Re-adding an identical record returns the same qualified name and avoids an
  unnecessary write.
- Re-adding an identical record repairs a missing, corrupted, or externally
  replaced dynamic spec instead of taking the no-op path.
- Distinct UUIDs resolve to distinct, HAMi-owned filenames.
- Replacing a conflicting UUID record does not preserve the old name.
- Removing one UUID deletes only its own file and preserves others.
- Removing the final UUID leaves no HAMi-owned dynamic MIG files.
- Missing-file removal succeeds.
- Invalid UUIDs and invalid generated specs are rejected.
- A failed write leaves the previous complete file readable.
- A failed write leaves other UUIDs' files unchanged; a retry regenerates
  only the failed UUID's file.
- Concurrent ensure/remove operations for distinct UUIDs pass `go test -race`
  and never overwrite another UUID's file.
- Startup replacement removes stale entries and retains adopted live entries.
- The null handler remains safe when CDI is disabled.
- CDI annotation and CRI response strategies receive the published qualified
  name.
- A qualified dynamic name is never qualified a second time by the response
  builder.
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
- Keep a reader loop parsing a per-UUID final path while its writer replaces
  that file; the reader must observe only the old or new valid specification,
  never a partial document.
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

1. Pass dynamic-MIG mode and the CDI root into the CDI handler; filter MIG
   entries from the base spec in this mode.
2. Add per-UUID lifecycle operations, atomic per-file writer, and unit tests
   without changing allocation behavior.
3. Connect dynamic MIG create/reuse to per-UUID ensure and return the
   published qualified names without double qualification.
4. Connect rollback, destruction, and periodic reconciliation to incremental
   removal of UUIDs actually destroyed.
5. Add pre-registration startup replacement from adopted allocations only.
6. Add logs and bounded metrics.
7. Validate with a CDI-enabled runtime and real MIG hardware.
8. Document operator prerequisites and troubleshooting after hardware results
   are known.

## Alternatives Considered

### Regenerate the startup GPU specification after every change

This is simple but mixes static discovery with HAMi-owned dynamic lifecycle,
increases collision risk, and makes cleanup ownership unclear. A dedicated
dynamic specification is safer.

### One node-wide CDI file for all dynamic MIG devices

This reduces the file count but requires every runtime create or destroy to
rewrite a shared snapshot. An older snapshot can overwrite an entry published
by a concurrent allocation. The bounded number of MIG instances does not
justify this extra synchronization and failure scope. Per-UUID files isolate
updates and simplify runtime lifecycle operations.

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
| Stale file survives restart | Startup-only replacement reconciles files against verified adopted instances before serving. |
| Concurrent allocation loses a CDI entry | Separate per-UUID files; runtime never replaces the whole set. |
| Failed publication damages another allocation | Atomic per-UUID write; other files remain unchanged. |
| CDI failure leaks new hardware | Integrate with existing newly-created allocation rollback. |
| Cleanup removes another producer's spec | Dedicated HAMi-owned filename and class. |
| Duplicate device names across specs | Dedicated class and UUID-based names. |
| Startup discovery republishes dynamic MIG in the base spec | Filter MIG entries from the base spec in dynamic mode. |
| Qualified name is qualified twice | Give the response builder an explicit qualified-name input. |
| CDI toolkit misses MIG capability nodes | Injected provider with explicit GI/CI node fallback and hardware tests. |
| Metrics create high cardinality | Never label metrics with UUIDs or Pod identity. |
| Legacy deployments regress | Gate all new behavior on the existing CDI strategy. |

## Design Decisions

- Vendor: reuse `k8s.device-plugin.nvidia.com`.
- Class: use `dynamic-mig` so HAMi owns a separate identity namespace.
- Filename: use `hami-dynamic-mig-<stable-UUID-hash>.json` below the handler's CDI root.
- CDI root: preserve `/var/run/cdi` as the default and inject it through an
  internal option for tests and future packaging needs.
- Device edits: reuse toolkit common and parent-GPU edits, then construct the
  two MIG capability nodes from the resolved parent/GI/CI runtime identity.
- Strategy coverage: support both `cdi-annotations` and `cdi-cri`, including
  configurations that also select a legacy strategy.
- Failure mode: fail startup before kubelet registration when initial required
  CDI reconciliation fails; fail the individual Allocate on later publication
  errors.
- Cleanup failure: emit an error metric and retry removal of that UUID during
  the existing periodic reconciliation loop; do not replace all files or
  delete another producer's file.
- Readiness: do not add a new readiness API. Startup failure, allocation
  errors, logs, and bounded metrics provide the signals required by this
  design.
- Idle pool: do not retain unreferenced GI/CI pairs. Reuse applies only to the
  same still-live allocation.

## Acceptance Criteria

- A newly created dynamic MIG device has a valid CDI entry before Allocate
  returns its qualified name.
- Every device in one multi-device allocation has a valid per-UUID file before
  any qualified names are returned; partial failure rolls back only new work.
- The name resolves to the correct live MIG UUID and required device nodes.
- Reusing the same live instance returns the same name without unnecessary
  file writes.
- Destroying an instance removes its entry without touching other CDI files.
- Restart recovery reconstructs exactly the set of adopted live instances.
- Device-plugin registration occurs only after required startup CDI
  reconciliation succeeds.
- All per-UUID file replacements are atomic.
- A failed replacement leaves the previous valid file for that UUID and all
  unrelated files unchanged.
- Runtime reconciliation removes only UUIDs of instances actually destroyed;
  full file-set replacement occurs only during startup recovery.
- HAMi is the exclusive MIG lifecycle controller for each managed GPU.
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
- [NVIDIA DRA Driver CDI implementation](https://github.com/kubernetes-sigs/dra-driver-nvidia-gpu/blob/main/cmd/gpu-kubelet-plugin/cdi.go)
- [NVIDIA DRA Driver device lifecycle](https://github.com/kubernetes-sigs/dra-driver-nvidia-gpu/blob/main/cmd/gpu-kubelet-plugin/device_state.go)
