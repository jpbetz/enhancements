# KEP-5975: Feature-Gate Field Dropping — Design Note

Design note for the "Feature-Gate Field Dropping" section of KEP-5975.

**Status: implemented** as a 5-commit stack on branch `kep5975-featuregate-tag`
(`21d9812f096 .. 758dcc9a8fc`, base `808a553dc88`). The chosen architecture is
**Model C2 — full versioned processing**: a `+k8s:featureGate(drop: true)` tag on
**versioned** types drives a generated drop that is registered into a
`runtime.Scheme` and run in the apiserver write path by converting the internal
object to the request version, dropping, and converting back. Coverage is the
`output_tests/drops` generator fixture plus scheme/write-path unit tests (§5); no
shipped resource has been migrated yet. All file:line references are against that
branch. (See §8 for the design history — the model went through two reversals
before landing here.)

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
   field's location in the type graph. The generator emits both from one tag:
   `dropDisabled_<Gate>_<Type>` (clear) and `inUse_<Gate>_<Type>` (read).

2. **validation-gen emits one function per type and delegates** — it does *not*
   produce flat root-rooted paths. `Validate_DeviceClaim` calls
   `EachSliceVal(..., Validate_DeviceRequest)`
   (`pkg/apis/resource/v1/zz_generated.validations.go`). The type graph is a
   memoized per-type DAG. The drop emitter **reuses this exact shared
   `typeDiscoverer` graph** (`discovered.typeNodes`; see `drops.go`
   `gatesForNode`/`gatesViaFields` and `targets.go`, where discovery "builds the
   type-graph that BOTH genValidations and genDrops consume"). So drop emission
   mirrors the per-type delegating structure and is **cycle-safe by
   construction**; it does not re-walk types or re-parse comments.

3. **Dropping runs on the *versioned* object, via an explicit in-write-path
   conversion.** Declarative validation already converts internal→versioned and
   validates a throwaway copy (`validateDeclaratively`, `rest/validate.go`).
   Dropping does the same but **mutates and writes back**: `DropDeclaratively`
   (`rest/validate.go:98`) converts internal→request-version
   (`Scheme.ConvertToVersion`), runs `Scheme.Drop` (mutates in place), then
   converts the dropped object **back** to internal
   (`Scheme.Convert(versionedObj, obj, nil)`, `rest/validate.go:137`). The cost
   of the round-trip is addressed by a cheap pre-check — `Scheme.HasDropFunc(gvk)`
   (`scheme.go:442`) short-circuits before any conversion for resources with no
   registered drop, so declarative-validation resources without drops pay zero
   added cost. The *remaining* tradeoff is **fidelity**, not cost: the
   convert-back can be lossy (see §4.3). (The earlier internal-typed model that
   avoided conversion was abandoned — §8.) `pkg/api/resourceclaimspec/` is
   **untouched** upstream code; there is no generated drop there.

4. **A `+k8s:featureGate` tag must NOT auto-imply a drop.** HPA adds the
   `HPAScaleToZero` DV option that *allows* `MinReplicas==0`
   (`pkg/registry/autoscaling/horizontalpodautoscaler/strategy.go`) but **never
   drops `MinReplicas`** — it drops only `Tolerance` (a different gate).
   Auto-dropping every gated field would silently wipe a user's `minReplicas: 0`.
   Drop is an explicit opt-in: a bare `+k8s:featureGate=<Gate>` records no
   `DropSpec`; only `+k8s:featureGate(drop: true)=<Gate>` opts in.

5. **Drop happens after `Prepare*` and before validation.** It is invoked in
   `rest.BeforeCreate` (`create.go:133`) / `rest.BeforeUpdate` (`update.go:139`),
   *after* `PrepareForCreate`/`PrepareForUpdate` and *before* `strategy.Validate`
   /`ValidateUpdate`/`ValidateDeclaratively`, so validation sees the post-drop
   object. It runs via a `DeclarativeMutationStrategy` type-assertion
   (`validate.go:86`) and is **unconditional** — independent of the
   `DeclarativeValidation`/`DeclarativeValidationBeta` feature gates (dropping is
   correctness/security, not an opt-in validation behavior). The generated code
   itself reads `DefaultFeatureGate.Enabled(features.<Gate>)` (from
   `k8s.io/apiserver/pkg/util/feature`). On update the drop runs before the
   UID/timestamp metadata fix-ups.

6. **The framework exists** (this branch), and it is *not* the one the early KEP
   draft proposed. There is **no** `registry.NewREST`, `registry.RESTConfig`,
   `strategy.Config[T]`, `SpecAccessor`/`StatusAccessor`, or a
   `DropDisabledFields` strategy hook. Instead:
   - `runtime.Scheme` gains drop dispatch (`AddDropFunc`/`AddDropOptionsFunc`/
     `Drop`/`HasDropFunc`/`DropOptions`, `scheme.go:407-455`).
   - Generated `RegisterDrops` registers per-root drop + options closures into a
     `Scheme` via `localSchemeBuilder`.
   - The apiserver adds the `DeclarativeMutationStrategy` interface +
     `DropDeclaratively` (`validate.go:86-140`), satisfied by embedding
     `rest.DeclarativeValidation`.
   The `+k8s:featureGate` tag is landed with **multi-valued** `Conditions`
   (`OptionsEnabled`/`OptionsDisabled []string` + `Inverted bool`), not the
   single-valued POC form.

7. **`+featureGate=<Gate>` docs comments already exist** (no `k8s:` prefix; 86
   on internal types, 334 on versioned) but are **incomplete** vs actual drop
   logic and are not machine-parsed. They are a starting inventory, not the
   source of truth, and are unrelated to the new tag.

## 3. Realistic coverage (background; nothing shipped is migrated yet)

The realistic mechanically-generatable surface is well under an "80%" first
guess. This analysis motivated the escape-hatch design; it is **decoupled from
what shipped** — the implementation proves the mechanism on synthetic types and
migrates no real resource.

Mechanically generatable:

- **Presence-gated single leaf** on pointer/slice/map (in-use = `!=nil` /
  `len>0`): PVC/PV `VolumeAttributesClassName`, deviceclass
  `ExtendedResourceName`, most DRA spec leaves.
- **Nested presence leaves** reached by descending slices/maps/structs, incl.
  the same gate on a field appearing in multiple sub-types (DRA `Tolerations` in
  both `ExactDeviceRequest` and `DeviceSubRequest`) via a **multi-leaf
  OR-reduced** in-use.

What the generator concretely supports today (`drops.go`):

- **Drop leaves**: pointer / slice / map (cleared to `nil`) and builtin scalars
  (cleared to the zero literal `false`/`""`/`0`).
- **Recursion**: struct, pointer-to-struct, slice-of-(struct | pointer-struct).
- **Loud failures** (`klog.Fatalf`, never silently skipped): value-typed struct
  leaves; map-of-struct recursion; unresolvable slice element type.

Escape-hatch by construction (verified hand-written examples that are *not*
generatable today):

- **Value-conditioned in-use**: ProcMount in-use is `*ProcMount != DefaultProcMount`,
  and it **resets to `&DefaultProcMount`**, not `nil`; HPA `*MinReplicas==0`;
  containerRestartRules keeps InitContainer `RestartPolicy==Always`.
- **Compound new-object precondition**: PVC `dataSourceRef` drops only if
  `DataSourceRef!=nil && Namespace!=nil && len(*Namespace)!=0 && !inUse(old)`.
- **Clear-leaf-then-collapse-empty-parent**: `dropContainerStopSignals` nils the
  whole `Lifecycle` once empty.
- **List-element removal with reference fixups**: `dropImageVolumes` rebuilds
  `Volumes` and patches every `VolumeMount` by name.
- **Set-membership compaction that must run last**: `dropDeallocatedStatusDevices`
  ("NOP if fields got dropped, so do this last").
- **Pod container-nested fields generally**: `EphemeralContainer` embeds
  `EphemeralContainerCommon`, a *distinct* type bridged by unsafe cast; the
  type-graph walker has no notion of the init/main/ephemeral tri-slice.

## 4. Design (Model C2)

### 4.1 Tag

A single marker on **versioned** types (where validation-gen reads tags):

- `+k8s:featureGate=<Gate>` — declarative-validation gating. The gate name is the
  required string payload. Scope is **field only**; the validator rejects
  non-pointer structs (`featuregate.go`). Stability: alpha. Repeatable → gates
  are **ANDed**: the field is forbidden when *any* required gate is disabled
  (emitted as one `!(op.HasOption("A") && op.HasOption("B"))` via the
  multi-valued `Conditions{OptionsEnabled: gates, Inverted: true}`).
- `(drop: true)` named-bool arg — **mandatory opt-in** to dropping. A bare
  `+k8s:featureGate` is DV-only and is *never* dropped (the HPA-safe default,
  §2 fact 4). The only drop metadata recorded is `DropSpec{Gates []string}`.

```go
// DV + auto drop (presence in-use):
//   +k8s:featureGate(drop: true)=VolumeAttributesClass
//   +k8s:optional
VolumeAttributesClassName *string `json:"volumeAttributesClassName,omitempty"`

// DV gated, intentionally NOT dropped (reject-at-validation policy):
//   +k8s:featureGate=HPAScaleToZero
MinReplicas *int32 `json:"minReplicas,omitempty"`
```

Multi-gate drop: the field is available only when **all** gates are enabled →
drop when **any** gate is disabled AND the field is not in-use in `old`.

> There is no custom-in-use tag arg. An earlier draft proposed
> `(drop, inUse: "custom")`; it was not implemented (§4.6).

### 4.2 Generator (`genDrops`)

A second generator inside validation-gen, run in the **same pass** as
`genValidations` (no separate command/binary). It consumes the shared
`typeDiscoverer` graph and reads each field's `childNode.fieldValidations.DropSpec`
(`DropSpec{Gates []string}`, surfaced through the normal `Validations` plumbing
alongside `DefaultConditions`).

Emitted symbols, per root type with a drop-gated descendant:

- `dropDisabled_<Gate>_<Type>(obj *<Type>)` — clears the gate's leaves in `obj`
  and recurses into sub-types via inline range loops (one function per
  `(gate, type)`, cycle-safe by construction).
- `inUse_<Gate>_<Type>(obj *<Type>) bool` — OR-reduces "is any of the gate's
  leaves set" over the same delegating walk.
- `DropDisabledFields_<Root>(newObj, oldObj *<Root>)` — exported spec-scope
  entry; per gate emits
  `if !DefaultFeatureGate.Enabled(features.<Gate>) && !inUse_<Gate>_<Root>(oldObj) { dropDisabled_<Gate>_<Root>(newObj) }`.
- `DropDisabledStatusFields_<Root>(newObj, oldObj *<Root>)` — status-scope entry
  (emitted only when there are status-scope drops; see §4.5).
- `DeclarativeValidationOptions_<Root>(oldObj) []string` — the DV options to
  enable (gate-enabled OR in-use), reusing the same `inUse_*` helpers so drop and
  validation ratcheting cannot desync (§4.5).
- `RegisterDrops(scheme)` + `func init()` that registers it with
  `localSchemeBuilder` (§4.3).

Gate symbol resolution: `features.<Gate>` is rendered via the gengo universe
(`types.Name{Package: <features-package>, Name: gate}`) where the package comes
from `--features-package` (default `k8s.io/kubernetes/pkg/features`); the runtime
check uses `DefaultFeatureGate` from `k8s.io/apiserver/pkg/util/feature`.

**Per-package gate override.** Both the accessor (`DefaultFeatureGate`) and the
gate-constant package are overridable per-package via
`+k8s:validation-gen-feature-gates=<pkg>`; when set, the generated drops
reference `<pkg>.DefaultFeatureGate` and `<pkg>.<Gate>` instead of the apiserver
/ `--features-package` defaults. This exists so the generator output test
(`output_tests/drops`, §4.4) — which lives in the `k8s.io/code-generator` module
and therefore cannot import the real apiserver/`k8s.io/kubernetes` feature
packages — can point the generated drops at a tiny in-package fake gate facility
(`gates.go`: a `Feature` type, a `DefaultFeatureGate` with `Enabled(Feature)`,
and the gate constants). Pointing the override at the fixture's own package makes
those references emit unqualified.

**Spec/status partition** is by field name: a root field named `Status` is status
scope, everything else is spec scope (`partitionRootFields`). The two scopes
share the per-`(gate, type)` worker functions.

**Empty-file suppression**: `genDrops` is attached to a package only when at
least one root has a drop-gated field (`hasDrops()`, `drops.go`). Drop-free
packages emit no `zz_generated.drops.go`. Output is one file per package via
`--drops-output-file`; `hack/update-codegen.sh` passes it (and
`--features-package`) on the single validation-gen pass and globs
`**/zz_generated.drops.go` for cleanup.

### 4.3 Runtime integration — scheme dispatch + write-path hook

This replaces the abandoned internal/versioned dual-body bridge (§8). There is
**no internal-typed drop and no generator-side conversion**; drops are emitted
once on the versioned types (into `pkg/apis/<grp>/<ver>/zz_generated.drops.go`,
in the main module, which may import `pkg/features`) and bridged to internal
**at runtime**.

**Scheme drop API** (`staging/src/k8s.io/apimachinery/pkg/runtime/scheme.go`),
mirroring the existing validation funcs:

- `dropFuncs` / `dropOptionsFuncs` maps keyed by `reflect.Type`.
- `AddDropFunc(srcType, func(ctx, op, object, oldObject))` (`:407`) and
  `AddDropOptionsFunc(srcType, func(oldObject) []string)` (`:414`).
- `Drop(ctx, op, object, oldObject) bool` (`:425`) — dispatches by
  `reflect.TypeOf`; **returns whether a func ran** (so the caller can skip the
  write-back when nothing dropped).
- `HasDropFunc(gvk) bool` (`:442`) — keyed on `reflect.PointerTo(gvkToType[gvk])`;
  the cheap pre-check.
- `DropOptions(object, oldObject) []string` (`:455`).

**Generated registration**: `RegisterDrops` registers a per-root closure that
switches on `op.Request.SubresourcePath()` (`"/"` → `DropDisabledFields_<Root>`,
`"/status"` → `DropDisabledStatusFields_<Root>`) and an options closure
returning `DeclarativeValidationOptions_<Root>`.

**Write-path hook** (`staging/src/k8s.io/apiserver/pkg/registry/rest/validate.go`):
the `DeclarativeMutationStrategy` interface (`:86`) with
`DropDeclaratively(ctx, obj, oldObj, opType, config)` (`:98`), satisfied by
embedding `rest.DeclarativeValidation`. Its body:

1. resolve the request `GroupVersion` from `requestInfo` (+ subresource mapper);
2. **pre-check**: `ObjectKinds(obj)` then `HasDropFunc(gv.WithKind(kind))`
   (`:112`) — return immediately if nothing is registered (zero added cost for
   declarative-validation resources without drops);
3. `ConvertToVersion(obj, gv)` (and `ConvertToVersion(oldObj, gv)` on update);
4. `Scheme.Drop(...)` (`:132`) — if it returns `false`, return without writing
   back (avoid a pointless, possibly-lossy round-trip);
5. otherwise `Scheme.Convert(versionedObj, obj, nil)` (`:137`) — write the
   dropped result back into the internal object that will be persisted.

The hook is invoked in `BeforeCreate` (`create.go:133`, old=nil) and
`BeforeUpdate` (`update.go:139`) via the type-assertion, unconditionally (§2
fact 5). This is precedented by mutating admission, which likewise converts
internal→versioned, mutates, and writes back via `GetObjectConvertor().Convert`.

**Fidelity risk (the central new tradeoff).** Validation discards the versioned
copy, so it tolerates lossy conversion; dropping writes it back. A wholesale
`Convert(versionedObj, obj)` can lose internal-only or newer-version-only data
(e.g. an update whose preserved `old.Status` carries newer-version-only fields,
or a non-round-tripping manual `Convert_`). The implemented mitigations cover
**cost** (the `HasDropFunc` pre-check; skip write-back when `Drop` returns
false), not lossiness. A per-resource **round-trip fidelity test** (fuzz
internal→version→internal, assert equal outside dropped fields) should gate each
real-resource migration; this is future work (§5).

### 4.4 Generated code (the generator fixture)

The shipped fixture is `output_tests/drops` (in the code-generator module). Its
`Root` carries gated leaves at several shapes; the generated
`zz_generated.drops.go` shows the **per-type delegation** (the root entry drives
a per-type worker that descends into the `RootSpec` sub-type — exactly the
genValidations structure, §2 fact 2) and references the package's **local fake
gate facility** (`DefaultFeatureGate`, `MultiLeafGate`, …) *unqualified*, via the
`+k8s:validation-gen-feature-gates` override (§4.2) — with no apiserver /
k8s.io/kubernetes imports:

```go
func DropDisabledFields_Root(newObj, oldObj *Root) {
    if newObj == nil { return }
    if !DefaultFeatureGate.Enabled(MultiLeafGate) && !inUse_MultiLeafGate_Root(oldObj) {
        dropDisabled_MultiLeafGate_Root(newObj)
    }
    // ... one if-block per gate (PointerLeafGate, ValueLeafGate)
}

func dropDisabled_MultiLeafGate_Root(obj *Root) {
    if obj == nil { return }
    dropDisabled_MultiLeafGate_RootSpec(&obj.Spec) // descend into the sub-type
}

func inUse_MultiLeafGate_Root(obj *Root) bool {
    if obj == nil { return false }
    if inUse_MultiLeafGate_RootSpec(&obj.Spec) { return true }
    return false
}

func DeclarativeValidationOptions_Root(oldObj *Root) []string {
    var opts []string
    if DefaultFeatureGate.Enabled(MultiLeafGate) || inUse_MultiLeafGate_Root(oldObj) {
        opts = append(opts, "MultiLeafGate")
    }
    // ... one block per gate
    return opts
}
```

`MultiLeafGate` reaches the same gate through two sub-types —
`RootSpec.Requests[*].Tolerations` (slice-of-struct) and
`RootSpec.Requests[*].Exactly.Tolerations` (pointer-to-struct) — so
`inUse_MultiLeafGate_RootSpec` **OR-reduces** across both leaves. The fixture
also exercises a pointer leaf, a value (string) leaf cleared to `""`, and a
**status-scope** leaf (`DropDisabledStatusFields_Root`). For a real API the gate
references would instead be `feature.DefaultFeatureGate` + `features.<Gate>` (the
§4.2 defaults); the per-type workers are also what `RegisterDrops` wires into the
scheme.

### 4.5 Semantics

- **Create** (old=nil): drop every `(drop: true)` field whose gates aren't all
  enabled (`inUse(nil)` = false). **Update**: drop iff (some gate disabled) AND
  (not in-use in **old**). These match the hand-written ratcheting contract.
- **Status scope**: status drops are a separate generated entry
  `DropDisabledStatusFields_<Root>`, selected at **runtime** by the subresource
  switch inside the registered drop closure on `op.Request.SubresourcePath()`
  (`"/"` spec, `"/status"` status). There is no strategy-substrategy plumbing;
  the partition is purely by the root field named `Status`.
- **Relationship to DV ratcheting**: the drop-vs-validation desync is closed by
  the generated `DeclarativeValidationOptions_<Root>(oldObj)` (gate-enabled OR
  in-use, including status in-use), surfaced via `Scheme.DropOptions`. The
  intended strategy wiring is `DeclarativeValidationConfig` returning
  `{Options: s.Scheme.DropOptions(obj, oldObj)}` — the standard wiring for a
  strategy with generated drops — which makes the generated forbidden-when-disabled
  validator **allow** an in-use disabled field that survived dropping (so the
  `+k8s:featureGate` forbidden check is defense-in-depth). No shipped strategy
  wires this yet; it lands with the first real-resource migration (§6 future
  work). Both halves consume the same `inUse_*` helpers, so they stay consistent
  by construction. Value-keyed options (HPA `*MinReplicas==0`) cannot be fed by a
  presence helper and stay hand-built.

### 4.6 Escape hatch (what actually exists)

1. **DV-only (default)**: omit `(drop: true)`.
2. **Fully hand-written drop**: don't tag the field for dropping; keep the
   bespoke drop (value-conditioned, default-reset, compound precondition,
   parent-collapse, list-removal, set-compaction, Pod container-nested). There is
   no generated custom-in-use hook.
3. **Codegen-forced**: unsupported leaf/recursion shapes (value-typed struct
   leaves, map-of-struct recursion) fail codegen loudly (`klog.Fatalf`), forcing
   the field to a hand-written drop.

> Not implemented (proposed in earlier drafts, listed here as future work): a
> `(drop, inUse: "custom")` custom-in-use level; an approver-owned exemption
> file; force-failing both-spec-and-status or manual-conversion types
> (spec/status is partitioned, not rejected; there is no internal conversion
> bridge to fail on).

### 4.7 Ordering

The real, delivered ordering guarantee is **write-path ordering**:
`DropDeclaratively` runs after `Prepare*` and before all validation
(`create.go:133` / `update.go:139`). Within a root entry, each gate's drop is
emitted independently (a sequence of per-gate `if !enabled && !inUse { ... }`).

There is **no** implemented data-dependency ordering analysis and **no**
ancestor/descendant co-generation refusal. Drops whose order matters relative to
each other or to bespoke drops (e.g. the `dropDeallocatedStatusDevices`
"must-run-last" case) are a **known limitation / future work**, not a delivered
guard — such a resource stays on a hand-written drop for now.

## 5. Testing & enforcement

Coverage is split across three layers (there is no single end-to-end
real-apiserver fixture; that was prototyped and then removed in favor of the
unit-level coverage below):

**Shipped:**

- **Generator** — `output_tests/drops/drops_test.go` calls the generated
  functions directly, covering direct pointer drop, value-leaf clearing,
  multi-leaf in-use preservation (OR-reduced across two sub-types), the options
  builder (ratcheting), and status-scope isolation, across the create/update ×
  in-use × gate matrix. Gates are toggled on the in-package fake facility (§4.4).
  (`testscheme` *stores* but does not *invoke* drop funcs, so these tests call
  `DropDisabledFields_*` directly.)
- **Scheme dispatch** — `apimachinery/.../scheme_test.go` `TestSchemeDrop`:
  `AddDropFunc`/`AddDropOptionsFunc` registration, `HasDropFunc` before/after,
  `Drop` runs + reports it ran (and reports *not* run for an unregistered type),
  `DropOptions`.
- **Write-path plumbing** — `apiserver/.../rest/validate_test.go`
  `TestDropDeclaratively`: the convert → drop → write-back round-trip with a stub
  drop func (asserts the dropped result lands on the internal object), plus the
  `HasDropFunc` no-op pre-check path.
- `make update`/`make verify` reproduce all generated files (the `hasDrops` guard
  keeps the diff to packages that actually have drops).

**Future work (not implemented):**

1. Per-`(drop)` **generated** unit tests (`zz_generated.drops_test.go`).
2. **Whole-`Prepare*` differential** fuzz harness (the migration gate): run the
   composed old-vs-new strategy pipeline over every gate combo and assert
   identical output; any divergence reclassifies the field to the escape hatch.
   This is the gate that would let a hand-written drop be deleted, and it is the
   prerequisite for the convert-back fidelity assertion (§4.3).
3. Blackbox cross-cutting `TestFeatureGateFieldDropping` (modeled on
   `TestFieldsWipingConsistency`) with allowlists and a DV-vs-drop consistency
   assertion.
4. Closed-world **lint invariant**: "any field carrying a gate-conditioned
   validator OR a drop MUST also carry `+k8s:featureGate`."
5. Approver-owned **exemption list** of intentionally non-conforming
   `(resource.version)` entries.

## 6. Implementation status — the landed stack

Branch `kep5975-featuregate-tag` (base `808a553dc88`), five bisectable,
codegen-reproducible commits:

- **C1 `21d9812f096` — `+k8s:featureGate` tag.** The validation tag +
  multi-valued `Conditions` (`OptionsEnabled`/`OptionsDisabled []string` +
  `Inverted`), `Validations.DefaultConditions` (AND-merged in `Add`, resolved in
  `registry.ExtractValidations`), `Validations.DropSpec{Gates []string}`, the
  single `featureGateTagValidator` (ScopeField; non-pointer-struct rejected;
  forbidden-when-disabled emitted via a deferred aggregation), and the validation
  output-test fixture (`output_tests/tags/featuregate`, validation-gating only —
  no `drop: true`).
- **C2 `c6d3badc1a2` — `runtime.Scheme` drop registration + dispatch**
  (`AddDropFunc`/`AddDropOptionsFunc`/`Drop`/`HasDropFunc`/`DropOptions`) +
  `TestSchemeDrop`.
- **C3 `d7bdca250ac` — `genDrops` generator.** Reuses the discovery graph; emits
  the per-`(gate, type)` workers + root entries + options builder + `RegisterDrops`;
  `--drops-output-file` / `--features-package` flags; the per-package
  `+k8s:validation-gen-feature-gates` override; `hasDrops` empty-file guard;
  supported-shape matrix with loud failures; `update-codegen.sh` wiring.
- **C4 `a6f74786790` — apiserver write-path integration.**
  `DeclarativeMutationStrategy` + `DropDeclaratively` (convert → drop →
  write-back, behind the `HasDropFunc` pre-check); hooks in
  `BeforeCreate`/`BeforeUpdate`; `TestDropDeclaratively`.
- **C5 `758dcc9a8fc` — `output_tests/drops` fixture.** The genDrops generator
  fixture (testscheme + in-package fake gate facility via the
  `+k8s:validation-gen-feature-gates` override): multi-leaf same-gate across
  sub-types, pointer leaf, value leaf, spec/status partition.

Future work: real-resource migration (with the §5.2 differential + §4.3 fidelity
test) — which also lands the first strategy that wires `Scheme.DropOptions` into
`DeclarativeValidationConfig` (§4.5); the remaining §5 enforcement items; the
§4.7 ordering analysis; map-of-struct recursion.

## 7. Decisions and open questions

### Resolved

1. **Emission target — versioned, registered on `runtime.Scheme`, run via a
   write-path hook.** (This reverses the interim "internal-typed, in strategy"
   decision — see §8.) `genDrops` emits versioned drops into
   `pkg/apis/<grp>/<ver>/zz_generated.drops.go`; execution is
   `DeclarativeMutationStrategy.DropDeclaratively` in `rest.BeforeCreate`/
   `BeforeUpdate`, which converts internal→versioned, runs `Scheme.Drop`, and
   converts back. There **is** a runtime versioned/internal bridge and there
   **is** a new write-path hook; there is **no** generator-side dual-body
   emission and **no** input-redirect/helper package.
2. **Generate the DV Options builder** from the shared in-use helper —
   *implemented*: `DeclarativeValidationOptions_<Root>` → `Scheme.DropOptions`.
   The strategy-side wiring (`DeclarativeValidationConfig` returning those
   options) lands with the first real-resource migration (§4.5). Value-keyed
   gates stay hand-built.
3. **No shipped resource migrated.** (This reverses the interim
   "resourceclaimspec / DRADeviceTaints first" decision.) The proof is the
   `output_tests/drops` generator fixture plus the scheme/write-path unit tests
   (§5); the prototyped self-contained end-to-end example was removed once those
   unit tests covered the dispatch and round-trip. `resourceclaimspec` remains
   the untouched hand-written drop.
4. **Scope restrictions** — ScopeField-only and non-pointer-struct rejection are
   the shipped behavior. **Multi-gate** "drop if any required gate disabled &
   not-in-use" is implemented (AND-merged `DefaultConditions` + forbidden
   `Inverted` over all gates; `DropSpec` accumulates gates) — though still
   unproven against a real hand-written precedent.

### Still open

- **Convert-back fidelity**: confirm the whole-pipeline differential + a
  per-resource round-trip fidelity test are sufficient to safely write back a
  versioned-converted object on update, or whether a field-scoped write-back is
  needed for resources with lossy/manual conversions (§4.3).
- **Real-resource migration** + equivalence differential vs the existing
  hand-written drop (the §6 "future work").
- **Ordering**: do any first-wave targets need the data-dependency / must-run-last
  analysis, or can they stay hand-written until one appears (§4.7)?
- **Enforcement surface**: exemption-file ownership/location; is "new field needs
  a gate" OWNERS-review-only, or worth a per-field introduced-version mechanism
  (separate KEP)?
- **Generalize to a mutation pass?** (Under consideration.) The runtime seam is
  already mutation-shaped — the interface is `DeclarativeMutationStrategy`, not
  drop-specific, and `DropDeclaratively` already converts **both** new and old
  and writes back, so reading `old` (needed for ratcheting) generalizes for free.
  Plausible additional tenants of one strategy-time pass: **field wiping** (same
  shape as dropping) and **reset-to-old** (immutable-per-subresource fields —
  viable on the existing convert-new+old→write-back rails; the open part is the
  *tag modeling*, not the execution). **Defaulting stays separate**: it runs at
  decode on the versioned object across many code paths (etcd reads, list, watch,
  conversion) and is already generated by defaulter-gen; the mutation pass runs
  *after* defaults on the internal object. If pursued, this is a file/registration
  rename (`zz_generated.drops.go` → `zz_generated.mutations.go`, `RegisterDrops`
  → an umbrella `RegisterMutations`) plus possibly generalizing `Scheme.Drop` to
  `Scheme.Mutate` — keeping the drop-specific `DropOptions` (validation
  ratcheting) separate, since it is not a mutation.

## 8. Design history (two reversals)

The emission/execution model reversed twice before landing on Model C2; recorded
here so the reversals aren't re-litigated.

1. **Versioned (pre-conversion) — first attempt.** Run the generated drop on the
   decoded+defaulted versioned object before it is converted to internal.
   Abandoned after finding the write handlers expose no pre-conversion versioned
   object (decode fuses decode+default+convert-to-hub).
2. **Internal-typed, in strategy — interim.** Emit internal-typed drops (a
   "drops-only" generation mode + a helper-package input-redirect) run in
   `PrepareForCreate`/`PrepareForUpdate`, and migrate `resourceclaimspec`.
   Abandoned because it duplicated generation machinery and split the tag away
   from the versioned types where declarative validation reads it. **None of this
   shipped**: there is no `--drops-only` flag, no `+k8s:validation-gen-drops`
   activation, no input-redirect, and `resourceclaimspec` is untouched.
3. **Model C2 — versioned via explicit in-write-path conversion (shipped).**
   Keep the tag and emission on the versioned types, but instead of relying on a
   pre-conversion object, **explicitly** convert internal→versioned inside the
   write path (`DropDeclaratively`), drop, and convert back. This answers the
   original open sub-problems: (a) hook location = `BeforeCreate`/`BeforeUpdate`
   after `Prepare*`, before validation; (b) old in versioned form =
   `ConvertToVersion(oldObj)` on update only; (c) per-resource opt-in =
   `RegisterDrops` + embedding `rest.DeclarativeValidation` + the `HasDropFunc`
   pre-check; (d) defaulting interaction = drop runs post-default on a freshly
   converted copy, then converts back.

Two architectural assumptions validated by the original spike carried through
all three models and remain load-bearing: the **per-type delegating** drop model
is behaviorally equivalent to hand-written drops (incl. multi-leaf OR-reduced
in-use), and **validation-gen can resolve gate constants by package path**
through the gengo universe without a Go import (no layering violation).
