# KEP-5975: Feature-Gate Field Dropping — Design Note

Working design note for the "Feature-Gate Field Dropping" section of KEP-5975.
Status: proposal for discussion. All file:line references are against
kubernetes/kubernetes at the time of writing.

## 1. Problem

Every API strategy hand-writes field-dropping code: in `PrepareForCreate` /
`PrepareForUpdate` it clears feature-gated fields when the gate is off, while
*preserving* a disabled field that the stored object already had ("in-use" /
ratcheting). The canonical shape (`pkg/api/pod/util.go:711`):

```go
if !utilfeature.DefaultFeatureGate.Enabled(features.UserNamespacesSupport) &&
    !hostUsersInUse(oldPodSpec) {
    podSpec.SecurityContext.HostUsers = nil
}
```

There are ~17 top-level `DropDisabled*` functions and dozens of `dropDisabled*`
helpers across `pkg/api/*/util.go` and `pkg/registry/*/strategy.go`. They are
repetitive, easy to get wrong, and a missing drop is a
[hard-to-fix data bug](https://github.com/kubernetes/kubernetes/pull/137715).

The goal: generate the mechanical cases from a field tag, keep an explicit
escape hatch for the rest, and make non-conformance detectable.

## 2. Key facts that constrain the design (all verified in-tree)

1. **The "in-use" predicate is the same path-walk as the drop**, read instead
   of cleared ("is this field set anywhere in `old`"). That is what makes
   generation feasible — both the clear and the in-use helper derive from the
   field's location in the type graph.

2. **validation-gen emits one function per type and delegates** — it does *not*
   produce flat root-rooted paths. `Validate_DeviceClaim` calls
   `EachSliceVal(..., Validate_DeviceRequest)`
   (`pkg/apis/resource/v1/zz_generated.validations.go`). The type graph is a
   memoized per-type DAG (`validation.go:321`, path rebooted per named type at
   `:338`). So a drop emitter must mirror the **per-type delegating** structure
   and is **cycle-safe by construction**; a flat emitter would infinite-loop on
   recursive types.

3. **Declarative validation runs on the *versioned* object** — `validateDeclaratively`
   converts internal→versioned and validates a throwaway copy
   (`staging/src/k8s.io/apiserver/pkg/registry/rest/validate.go:140`). Validation
   only reads, so the copy is fine. **Dropping mutates the persisted object** and
   runs on **internal** types in the strategy
   (`pkg/api/resourceclaimspec/util.go:33` takes `*resource.ResourceClaimSpec`).
   A convert→drop→convert-back round-trip on every write would be costly and
   conversion-lossy. This internal/versioned split is the central seam.

4. **A `+k8s:featureGate` tag must NOT auto-imply a drop.** HPA adds the
   `HPAScaleToZero` DV option that *allows* `MinReplicas==0`
   (`pkg/registry/autoscaling/horizontalpodautoscaler/strategy.go:89-104`) but
   **never drops `MinReplicas`** — it drops only `Tolerance` (a different gate).
   Auto-dropping every gated field would silently wipe a user's `minReplicas: 0`.
   Drop must be an explicit opt-in.

5. **Drop happens before validation**: `PrepareForCreate`→`Validate`→
   `ValidateDeclaratively` (`rest/create.go`), so validation sees the post-drop
   object. Generated drops call `utilfeature.DefaultFeatureGate.Enabled(...)`
   directly — `Prepare*` carries no `Options` (those are built later, in
   `DeclarativeValidationConfig`).

6. **The proposed framework does not exist yet**: `registry.NewREST`,
   `registry.RESTConfig`, `strategy.Config[T]`, the `DropDisabledFields` hook,
   and `SpecAccessor`/`StatusAccessor` are KEP-only (grep finds them only in the
   doc). The `+k8s:featureGate` *validation* tag exists only on the
   `origin/default-option-poc` branch (validation-only, `ScopeField`-only,
   rejects non-pointer structs); mainline `Conditions` is single-valued.

7. **`+featureGate=<Gate>` docs comments already exist** (no `k8s:` prefix; 86
   on internal types, 334 on versioned) but are **incomplete** vs actual drop
   logic (Pod drops touch ~35 gates; ~10 of the fields are marked) and are not
   machine-parsed. They are a starting inventory, not the source of truth.

## 3. Realistic coverage

Well under the "80%" first guess. Mechanically generatable:

- **Presence-gated single leaf** on pointer/slice/map (in-use = `!=nil` /
  `len>0`): PVC/PV `VolumeAttributesClassName`, deviceclass
  `ExtendedResourceName`, workload `SchedulingConstraints`/`PriorityClassName`,
  most DRA spec leaves.
- **Nested presence leaves** reached by descending slices/maps/structs, incl.
  the same gate on a field appearing in multiple sub-types (DRA `Tolerations` in
  both `ExactDeviceRequest` and `DeviceSubRequest`) via **multi-leaf OR-reduced**
  in-use.

Escape-hatch by construction (verified examples):

- **Value-conditioned in-use**: ProcMount in-use is `*ProcMount != DefaultProcMount`,
  not `!=nil`, and it **resets to `&DefaultProcMount`**, not `nil`
  (`util.go:1198,1430`); HPA `*MinReplicas==0`; containerRestartRules keeps
  InitContainer `RestartPolicy==Always`.
- **Compound new-object precondition**: PVC `dataSourceRef` drops only if
  `DataSourceRef!=nil && Namespace!=nil && len(*Namespace)!=0 && !inUse(old)`
  — reads a sibling subfield on **new** (`persistentvolumeclaim/util.go:48`).
- **Clear-leaf-then-collapse-empty-parent**: `dropContainerStopSignals` nils the
  whole `Lifecycle` once empty (`util.go:828`); same for sleep action.
- **List-element removal with reference fixups**: `dropImageVolumes` rebuilds
  `Volumes` and patches every `VolumeMount` by name (`util.go:1608`).
- **Set-membership compaction that must run last**: `dropDeallocatedStatusDevices`
  (`resourceclaim/strategy.go:200`, "NOP if fields got dropped, so do this last").
- **Pod container-nested fields generally**: `EphemeralContainer` embeds
  `EphemeralContainerCommon`, a *distinct* type from `Container` bridged by
  unsafe cast; the type-graph walker has no notion of the init/main/ephemeral
  tri-slice that `VisitContainers` traverses uniformly. Escape-hatch in Phase 1.

## 4. Design

### 4.1 Tag

Two-part marker on **versioned** types (where validation-gen reads tags):

- `+k8s:featureGate=<Gate>` — DV gating; `ScopeField`; repeatable (ANDed).
- `(drop)` attribute — **mandatory opt-in** to dropping. Bare
  `+k8s:featureGate` is DV-only and is *never* dropped (the HPA-safe default).

```go
// DV + auto drop (presence in-use):
//   +k8s:featureGate=VolumeAttributesClass(drop)
//   +k8s:optional
VolumeAttributesClassName *string `json:"volumeAttributesClassName,omitempty"`

// DV gated, intentionally NOT dropped (reject-at-validation policy):
//   +k8s:featureGate=HPAScaleToZero
MinReplicas *int32 `json:"minReplicas,omitempty"`

// Drop body generated, in-use hand-written:
//   +k8s:featureGate=DRAAdminAccess(drop, inUse: "custom")
AdminAccess *bool `json:"adminAccess,omitempty"`
```

Multi-gate semantics: field available only when **all** gates enabled → drop
when **any** disabled AND not in-use.

### 4.2 Generator

A **new emitter inside validation-gen** that reuses the type *graph*
(`typeDiscoverer`/`typeNode`/`childNode`) but emits **mutation** code:

- Annotate `childNode` (`validation.go:213`) with `featureGates []string` +
  `dropSpec` during struct discovery; the emitter consumes the graph without
  re-parsing.
- **Per-type delegating functions**: `dropDisabled_<Type>(obj *<Type>, drop bool)`
  descends one level and delegates to element/sub-type drop fns via new
  `DropEachSliceVal`/`DropEachMapVal` mutation helpers (siblings of
  `EachSliceVal`). At a gated leaf, emit the clear.
- **Whole-object in-use hoisted to the root**: a per-type fn can't see the whole
  object, so a root `dropDisabledFields_<RootSpecType>(new, old)` computes, per
  gate, `drop := !allEnabled(G) && !inUse_G_<Root>(old)` and threads the bool
  down.
- **Multi-leaf in-use aggregation**: for gate `G`, collect *all* `(drop)` leaves
  transitively reachable carrying `G` and emit one `inUse_G_<Root>(old)` that
  OR-reduces over them (a missing leaf = data-loss bug, so this is mandatory).
- **Gate symbol resolution**: `features.<Gate>` → `types.Name{Package:
  "k8s.io/kubernetes/pkg/features", Name: Gate}`; load `pkg/features` into the
  gengo universe and assert the constant exists (fail loudly).
- **Output**: `zz_generated.drops.go` per package; `--drops-output-file` flag;
  `hack/update-codegen.sh` globs it for cleanup.

**Compile-safety guards** (force escape-hatch at codegen, with a clear error):
gated leaf in a map value or slice-of-value-struct (non-addressable, won't
compile); type reachable through both Spec and Status (ambiguous partition);
type with a manual/non-1:1 conversion override (no field correspondence for the
internal bridge — these exist, e.g. RC↔RS in `core/v1/conversion.go`).

### 4.3 Internal/versioned bridge

Emit into `pkg/apis/<group>/<version>/zz_generated.drops.go` — the **main
module** (`k8s.io/kubernetes`), which may import `pkg/features` (verified
`core/v1/defaults.go`). Staging `k8s.io/api/*` must **not** import
`pkg/features`/`apiserver` (layering), so drops cannot live there.

Strategies operate on **internal** types, so the generator emits **two bodies
from the one tag** — versioned and internal — guarded by an
**auto-1:1-conversion check**; manual-conversion types fail closed to the escape
hatch. (Open question 5.1: sig-api-machinery may instead prefer standing up
validation-gen over internal `pkg/apis` packages.)

### 4.4 Generated code (DRA `Tolerations`, gate `DRADeviceTaints`, two leaves)

```go
func dropDisabledFields_ResourceClaimSpec(new, old *resourcev1.ResourceClaimSpec) {
    if new == nil { return }
    dropDeviceTaints := !utilfeature.DefaultFeatureGate.Enabled(features.DRADeviceTaints) &&
        !inUse_DRADeviceTaints_ResourceClaimSpec(old)
    dropDisabled_DeviceClaim(&new.Devices, dropDeviceTaints)
}

func dropDisabled_DeviceRequest(obj *resourcev1.DeviceRequest, dropDeviceTaints bool) {
    if obj == nil { return }
    if obj.Exactly != nil { dropDisabled_ExactDeviceRequest(obj.Exactly, dropDeviceTaints) }
    for i := range obj.FirstAvailable { dropDisabled_DeviceSubRequest(&obj.FirstAvailable[i], dropDeviceTaints) }
}

func dropDisabled_ExactDeviceRequest(obj *resourcev1.ExactDeviceRequest, dropDeviceTaints bool) {
    if obj == nil { return }
    if dropDeviceTaints { obj.Tolerations = nil }
}

// multi-leaf OR-reduced in-use over the same delegating walk
func inUse_DRADeviceTaints_ResourceClaimSpec(old *resourcev1.ResourceClaimSpec) bool {
    if old == nil { return false }
    for i := range old.Devices.Requests {
        r := &old.Devices.Requests[i]
        if r.Exactly != nil && len(r.Exactly.Tolerations) > 0 { return true }
        for j := range r.FirstAvailable {
            if len(r.FirstAvailable[j].Tolerations) > 0 { return true }
        }
    }
    return false
}
```

Create: `dropDisabledFields_ResourceClaimSpec(&new.Spec, nil)` (in-use(nil)=false).
Update: `(&new.Spec, &old.Spec)`.

### 4.5 Semantics

- **Create** (old=nil): drop every `(drop)` field whose gates aren't all
  enabled. **Update**: drop iff (some gate disabled) AND (not in-use in **old**).
- **Status** drops emit a separate `dropDisabledStatusFields_<Type>` that may
  assign only under `new.Status` (codegen-enforced); status in-use may read
  `old.Spec` and `old.Status` (use custom in-use). Split-strategy resources call
  it in the status substrategy after `new.Spec=old.Spec`+`ResetObjectMetaForStatus`;
  Pod-style unified entrypoints call spec+status together (incl. create).
- **Relationship to DV in-use/ratcheting** (scoped honestly): the
  "drop and DV share one in-use" unification holds only for **declarative +
  `op.HasOption`** resources, where the DV option is
  `allowed = gate || inUse(old)` and the drop is its exact complement — both must
  consume the same exported `inUse_<Gate>_<Type>`. For **imperative** resources
  (Pod builds `opts.Allow*` and threads them into `ValidatePodSpec`, not
  `op.HasOption` — `util.go:478`), the contract is instead that the generated
  `inUse_*` is **exported and reused** by hand-written `GetValidationOptions`.
  Value-specific options (HPA `*MinReplicas==0`) can't be fed by a presence
  helper and are escape-hatch.

### 4.6 Escape hatch (opt-out levels)

1. **DV-only (default)**: omit `(drop)`.
2. **Custom in-use**: `(drop, inUse: "custom")` — generator emits the drop body,
   calls a hand-written `inUse_<Gate>_<Type>(old)`; codegen verifies the symbol.
3. **Custom/omitted drop**: keep the hand-written drop entirely (value-conditioned,
   default-reset, compound precondition, parent-collapse, list-removal, set
   compaction, Pod container-nested).
4. **Auto-forced**: non-addressable targets, both-Spec-and-Status types, and
   manual-conversion types are forced to the hatch with a codegen error.

Every `+k8s:featureGate` field is then in exactly one discoverable state:
generated, partial-bespoke (intentional), DV-only (intentional keep), or
listed in the approver-owned exemption file.

### 4.7 Ordering

Drop precedes validation. **Order is by data dependency, declared per resource**
— not a global "bespoke-first" or lexical sort (counterexample:
`dropDeallocatedStatusDevices` must run **last**). The generator emits
ancestor-before-descendant and **refuses to co-generate** two drops where one's
clear target is an ancestor of the other's iteration root (forces escape-hatch).
Coexistence is verified by the whole-`Prepare*` differential (below).

## 5. Testing & enforcement

1. **Generated unit tests** per `(drop)` field (extend validation-gen's fixture
   mechanism → `zz_generated.drops_test.go`): create-drops-when-off,
   create-keeps-when-on, update-drops-when-off-not-in-use,
   update-keeps-when-off-in-use, update-keeps-when-on.
2. **Behavioral equivalence (the migration gate)**: a differential fuzz test
   (apitesting fuzzer) runs the **whole strategy `Prepare*` pipeline** old vs
   new (generated + bespoke composed) over every gate combo, asserting identical
   output. *Not* byte-for-byte of the isolated fn — generated structure differs
   even when behavior matches, and wrapping the whole pipeline is what covers
   ordering interactions. Any divergence reclassifies the field to escape-hatch.
3. **Blackbox cross-cutting** `TestFeatureGateFieldDropping`, modeled on the
   existing `TestFieldsWipingConsistency`
   (`test/integration/apiserver/apidefinitions/`, PR #137689) with
   `sets.New(...)` allowlists and "DO NOT ADD NEW ENTRIES". Includes a
   DV-vs-drop consistency assertion for declarative+`op.HasOption` resources.
4. **Enforcement, scoped honestly**: implement the closed-world, verifiable-today
   lint invariant *"any field carrying a gate-conditioned validator OR a drop
   MUST also carry `+k8s:featureGate`."* The aspirational *"every new field needs
   a gate"* is an **OWNERS/api-approver review policy**, not a mechanical gate —
   there is no per-field introduced-version metadata (only type-level
   `+k8s:prerelease-lifecycle-gen:introduced`), and a git-diff detector is
   advisory at best.
5. **Exemption list**: one approver-owned file enumerating intentionally
   non-conforming `(resource.version)` entries (legacy no-gate, fully-bespoke,
   manual-conversion); new entries need api-approver sign-off.

## 6. Phased implementation

- **PR -1 (spike, gating)**: prove validation-gen can load `pkg/features` and
  resolve `features.X`; hand-write the per-type delegating drop + multi-leaf
  in-use for `DRADeviceTaints` and **behaviorally diff** it against
  `pkg/api/resourceclaimspec/util.go` across the full create/update/in-use/gate
  matrix. Re-estimate coverage from the result. If feature-loading or the
  per-type model fails, revise before any other PR.
- **PR 0 (prereq)**: land `+k8s:featureGate` DV tag to mainline; extend
  `Conditions` to multi-gate ANDing; add `(drop)`/`(inUse:"custom")` parsing;
  freeze the `childNode` contract; decide ScopeField-only / non-pointer-struct
  scoping (value-structs out of Phase 1).
- **PR 1**: carry gate+drop annotations on `childNode`; expose discoverer
  `dropSpecsByField`. No emission.
- **PR 2**: `genDrops` + per-type delegating emitter + root in-use hoist +
  multi-leaf aggregation + `DropEachSliceVal`/`DropEachMapVal` + namers +
  `--drops-output-file`; **mechanical presence-leaf single-gate only**; gate
  symbol load-and-assert; non-addressable rejection; versioned+internal bodies;
  generated fixture tests.
- **PR 3**: multi-gate ANDing; nested-leaf traversal across shared sub-types
  (the DRA two-leaf case); wire `update-codegen.sh`; auto-1:1-conversion guard.
- **PR 4**: status-scope partitioning + assigns-only-under-Status invariant;
  refuse both-Spec-and-Status; split vs unified call-site docs.
- **PR 5**: escape-hatch attributes + hand-written-helper existence check;
  data-dependency ordering + ancestor/descendant co-generation refusal.
- **PR 6**: whole-`Prepare*` differential harness; migrate PVC
  `VolumeAttributesClassName` (dataSourceRef stays bespoke).
- **PR 7+**: per-resource takeover loop (PV, deviceclass, nested DRA leaves):
  tag → prove equivalence → switch call site → delete hand-written after a
  release.
- **PR N (parallel)**: blackbox test + exemption file.
- **PR N+1 (parallel)**: closed-world lint invariant.
- **PR N+2 (deferred)**: Phase 2 auto-wiring via `registry.NewREST` /
  `strategy.Config[T]` / `SpecAccessor` `DropDisabledFields` hook (blocked on
  that framework, which doesn't exist yet).

## 6a. PR -1 spike results (done)

The gating spike was implemented and run in `kubernetes/kubernetes`
(`pkg/api/resourceclaimspec/`, throwaway `*_spike_test.go`):

- **Per-type delegating model is behaviorally equivalent.** Hand-wrote the
  would-be-generated per-type `dropDisabled_*` + root-hoisted, multi-leaf
  OR-reduced `inUse_*` for `DRADeviceTaints` (internal-typed), and diffed it
  against the real `dropDisabledDRADeviceTaintsFields` over a crafted + 500-spec
  fuzz corpus × {create, old-empty, old-in-use, old-equals-new} × {gate on,
  off}. **Output identical in every cell**; 903 comparisons exercised a
  non-trivial drop (non-vacuity guard). The two-leaf aggregation
  (`Exactly.Tolerations` + every `FirstAvailable[*].Tolerations`) is reproduced
  by the per-type walk. The PrioritizedList→DeviceTaints ordering dependency is
  pinned by a dedicated test.
- **validation-gen can resolve gate constants without a Go import.** Drove
  gengo's parser on the path string `k8s.io/kubernetes/pkg/features`: 171
  constants parsed, `DRADeviceTaints`/`DRAPrioritizedList`/`DRAAdminAccess`
  resolved; a typo does not (via the raw `Constants` map — note `Package.Constant()`
  is get-or-create and unsafe for existence checks). This confirms the generator
  in the published `k8s.io/code-generator` module can validate gate names and
  emit `features.<Gate>` with no layering violation (gengo parses by path, not
  Go import).

Both assumptions the architecture rests on hold. Mechanical-coverage
re-estimation (the spike's third goal) stands: the realistic first wave is
presence-pointer + nested-presence DRA leaves; value-conditioned/compound/Pod
cases remain escape-hatch.

## 6b. PR 0 status (done, on branch `kep5975-featuregate-tag`)

The `+k8s:featureGate` **validation** tag is landed on current mainline. Note a
material finding: the POC (`origin/default-option-poc`, base Mar 2026) is built
on a validation-gen architecture that mainline has since **refactored** — the
second-pass `RegisterFieldValidator`/`FieldValidator` mechanism the POC relied
on was removed in favor of a `Deferred`/shared-map idiom (see `update.go`). So
this was an *adaptation*, not a cherry-pick:

- `Conditions` made multi-valued: `OptionsEnabled []string` / `OptionsDisabled
  []string` / `Inverted bool` (was single-valued); `emitCondition` helper emits
  `op.HasOption("A") && op.HasOption("B")`, optionally negated. `Conditions` had
  **no producer** on mainline, so this was low-risk dormant infra.
- `Validations.DefaultConditions *Conditions` added; merged (AND) in `Add`;
  resolved onto sibling functions in `registry.ExtractValidations`. This is how
  the gate gates the field's *other* validations.
- `validators/featuregate.go`: a single `TagValidator` (ScopeField) that records
  gates per field path, returns `DefaultConditions` for sibling gating, and emits
  the aggregated forbidden+optional pair via a `Deferred` callback (so a
  multi-gated field yields one `!(A && B)` check, matching the POC's intent).
- Output test `output_tests/tags/featuregate/` (ported from POC, behavior-based):
  all pass — single/multi gate, pointer/slice/map/value/struct-ptr, create +
  update ratcheting. Full validation-gen suite green; gofmt/build clean; tag
  shows in `--docs`.

**Deliberate deviation from the §6 plan:** PR 0 here is the **validation tag
only**. The `(drop)` / `(inUse:"custom")` attribute parsing and the
`childNode` drop-spec contract are *not* included — they are inert without the
drop emitter and would freeze tag syntax prematurely. They move to the
drop-emitter PRs (was §6 PR 1/PR 2). This keeps PR 0 a clean, independently
useful, reviewable DV change owned by the declarative-validation effort.

## 6c. PR 1 status (done, on branch `kep5975-featuregate-tag`)

Drop opt-in parsing + per-field drop-spec plumbing landed (no emission):

- `+k8s:featureGate(drop:true)=<Gate>` — a `drop` named-bool arg opts a field
  into automatic dropping. A bare `+k8s:featureGate=<Gate>` is DV-only and is
  never dropped (the HPA-safe default). Codetags arg syntax is no-space:
  `(drop:true)`, gate as the value.
- `featureGateTagValidator` now records a per-field `featureGateFieldSpec{gates
  []string, drop bool}` (the "drop spec" the emitter will consume), replacing the
  previous gate-name-only map.
- The `drop` arg is declared in `Docs()` (type bool) so the registry accepts it.
- Output test `DropStruct` (single + multi gate, one carrying `drop:true`) plus
  `TestDropStruct` confirm the drop arg is accepted and **declarative-validation
  behavior is identical** to a gated field without it. Full validation-gen suite
  green (147 pkgs), module builds, gofmt clean.

## 6d. PR 2 status (done — emitter built & verified end-to-end)

The `genDrops` generator is implemented in validation-gen and proven end-to-end
on a self-contained internal-typed fixture:

- New `--drops-output-file` flag (+ `--features-package`, default
  `k8s.io/kubernetes/pkg/features`) adds `genDrops` alongside `genValidations`;
  inactive without the flag (full validation-gen suite stays green, 147 pkgs).
- `genDrops` emits the **per-type, per-gate** model the spike validated:
  `dropDisabled_<Gate>_<Type>` (clears + recurses), `inUse_<Gate>_<Type>`
  (OR-reduces), and the entry `dropDisabledFields_<Root>(newObj, oldObj)` that
  drops each gate when `!DefaultFeatureGate.Enabled(features.<Gate>) &&
  !inUse_<Gate>_<Root>(oldObj)`. Per-(gate,type) functions are cycle-safe by
  construction; gate constants are referenced via `features.<Gate>` resolved
  through the gengo universe (spike Part B).
- Supported mechanical cases: pointer/slice/map drop leaves; recursion through
  structs, pointers-to-structs, and slices-of-(structs|pointers). Unsupported
  cases (value-typed leaves, map-of-struct recursion) **fail codegen loudly**,
  never silently skipped.
- Fixture `pkg/api/fielddroptest/` (Root→Spec→Requests[]→{Tolerations,
  Exactly→Tolerations}) exercises a direct pointer leaf (one gate) and a
  **multi-leaf** gate spanning two sub-types. Generated code compiles in the
  main module; behavior tests pass the full create/update × in-use × gate matrix,
  including multi-leaf in-use preservation.

Follow-ups since landed:

- **Value-typed drop leaves** — gated non-pointer scalars (string/bool/numeric)
  clear to their zero value (`""`/`false`/`0`), in-use via `!= zero`. Fixture
  `Mode string` leaf + test.
- **Spec/status partitioning** — root types emit two entries,
  `dropDisabledFields_<Root>` (spec) and `dropDisabledStatusFields_<Root>`
  (fields reached through the `Status` member), mirroring the hand-written
  `DropDisabledFields` / `DropDisabledFieldsFromStatus` split. Per-gate root
  helpers are member-scoped; shared sub-type helpers are reused across scopes.
  Fixture `Status` field + scope-isolation tests.

Remaining for the emitter: map-of-struct recursion (currently fails loudly);
package-qualified function names for cross-package sub-types; and the real-API
wiring (the versioned/internal bridge, design §7 Q1) — the fixture sidesteps the
bridge by being internal-typed and self-contained.

## 7. Decisions and open questions

### Resolved (2026-06-15)

1. **Emission target — versioned-only, drop pre-conversion.** Drops are emitted
   versioned-typed and applied to the *versioned* object early in the write
   pipeline (decode → default → **drop** → convert to internal → store), NOT in
   the internal strategy. This removes the versioned/internal bridge and
   dual-body generation entirely (genDrops emits a single body for the type it
   runs on). Cost: a new pipeline integration point, and the proposed
   `strategy.Config.DropDisabledFields` hook no longer fits — dropping moves out
   of the strategy. **Implication:** since validation runs after the drop, the
   forbidden-on-disabled check becomes redundant for dropped fields; ratcheting
   (allow an in-use disabled field) is preserved by the generated Options builder
   (below). The pipeline hook (where new-versioned and a convertible old meet
   before internal conversion) is the next design/impl task — see §8.
2. **Generate the DV Options builder** from the shared in-use helper, mechanically
   closing the drop-vs-validation desync loop for declarative + `op.HasOption`
   resources. Value-keyed gates (e.g. HPA `MinReplicas==0`) stay hand-built.
3. **First migration target — resourceclaimspec / DRADeviceTaints**, matching the
   spike and fixture; proven via a whole-pipeline differential vs the existing
   `dropDisabledDRADeviceTaintsFields`.

### Still open

- Keep the POC's ScopeField-only + non-pointer-struct restrictions for Phase 1?
- **Multi-gate drop semantics**: no hand-written precedent to diff against —
  confirm "drop if any required gate disabled & not-in-use" or defer until a real
  case appears.
- Exemption-file ownership/location; is the "new field needs a gate" policy
  OWNERS-review-only, or worth a per-field introduced-version mechanism
  (separate KEP)?
- Soak: is whole-pipeline fuzz equivalence enough to delete a hand-written drop,
  or require a release-cycle CI soak first?

## 8. Pre-conversion drop pipeline integration (next task)

The versioned-only decision requires a hook that runs the generated
`dropDisabledFields_<VersionedType>(newObj, oldObj)` on the decoded+defaulted
versioned object before it is converted to the internal type. The open
sub-problems: (a) the exact hook location in the create/update handlers where the
new versioned object is available; (b) supplying the old object in versioned form
for in-use/ratcheting on update (old is fetched from storage as the storage
version); (c) per-resource opt-in/registration of the generated drop function;
(d) interaction with defaulting (drop must run after defaults) and with the
storage-version codec. This is investigated in the implementation notes before
wiring resourceclaimspec.
