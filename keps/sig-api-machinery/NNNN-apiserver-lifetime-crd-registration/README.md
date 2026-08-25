# KEP-NNNN: Static CustomResourceDefinitions

<!-- toc -->
- [Release Signoff Checklist](#release-signoff-checklist)
- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Proposal](#proposal)
  - [User Stories](#user-stories)
    - [Story 1: A Dedicated etcd Cluster for One High-Volume Custom Resource](#story-1-a-dedicated-etcd-cluster-for-one-high-volume-custom-resource)
    - [Story 2: A Platform Type That Cannot Be Deleted Through the API](#story-2-a-platform-type-that-cannot-be-deleted-through-the-api)
    - [Story 3: Disabling the Watch Cache for a Large, Cold Custom Resource](#story-3-disabling-the-watch-cache-for-a-large-cold-custom-resource)
    - [Story 4: Changing a Manifest CRD Across a Staggered Fleet, and Rolling It Back](#story-4-changing-a-manifest-crd-across-a-staggered-fleet-and-rolling-it-back)
    - [Story 5: An API-Only User Installing, Syncing, and Uninstalling Against a Manifest CRD](#story-5-an-api-only-user-installing-syncing-and-uninstalling-against-a-manifest-crd)
  - [Notes/Constraints/Caveats](#notesconstraintscaveats)
  - [Risks and Mitigations](#risks-and-mitigations)
- [Design Details](#design-details)
  - [Why Real API Objects Rather Than an Invisible Parallel Universe](#why-real-api-objects-rather-than-an-invisible-parallel-universe)
  - [Configuration Surface](#configuration-surface)
    - [New Flag and Config Kind](#new-flag-and-config-kind)
    - [Manifest File Format](#manifest-file-format)
    - [Existing Storage Flags](#existing-storage-flags)
    - [Flag Grammar Asymmetry](#flag-grammar-asymmetry)
  - [Feature Gates and Disablement Semantics](#feature-gates-and-disablement-semantics)
  - [The Materialized CRD Object](#the-materialized-crd-object)
  - [Design Rule: No Admin-Environment Detail on API Objects](#design-rule-no-admin-environment-detail-on-api-objects)
  - [Loading, Decoding, and Validation](#loading-decoding-and-validation)
  - [Manifest Revisions and Convergence](#manifest-revisions-and-convergence)
    - [The Declaration and the Record](#the-declaration-and-the-record)
    - [The Ordering Rule](#the-ordering-rule)
    - [The Behind Guard](#the-behind-guard)
    - [Rollback](#rollback)
    - [What Is Revertible](#what-is-revertible)
    - [Convergence Observability](#convergence-observability)
  - [The Reconciler](#the-reconciler)
  - [Declarative Removal: Tombstones](#declarative-removal-tombstones)
  - [Write Protection](#write-protection)
    - [Denial Messages](#denial-messages)
  - [Storage Configuration](#storage-configuration)
    - [Etcd Server Overrides for Custom Resources](#etcd-server-overrides-for-custom-resources)
    - [Watch Cache Control for Custom Resources](#watch-cache-control-for-custom-resources)
    - [The Storage Binding Witness](#the-storage-binding-witness)
    - [Pre-Activation Emptiness Check](#pre-activation-emptiness-check)
    - [The Per-Request Binding Guard](#the-per-request-binding-guard)
    - [Health and Readiness for Override Backends](#health-and-readiness-for-override-backends)
    - [Storage Warm-Up](#storage-warm-up)
  - [Lifecycle and Precedence](#lifecycle-and-precedence)
    - [Precedence Rules](#precedence-rules)
    - [Startup Sequence](#startup-sequence)
    - [Manifest Reload](#manifest-reload)
    - [Readiness Contract](#readiness-contract)
    - [Demotion and Decommissioning](#demotion-and-decommissioning)
    - [Transitions Between Dynamic and Static Registration](#transitions-between-dynamic-and-static-registration)
    - [storedVersions and Storage Version Migration](#storedversions-and-storage-version-migration)
  - [Workflow Matrix](#workflow-matrix)
  - [Bug Fixes Carried by This KEP](#bug-fixes-carried-by-this-kep)
  - [Observability](#observability)
    - [Metrics](#metrics)
    - [Audit Annotations](#audit-annotations)
  - [API Changes](#api-changes)
  - [Test Plan](#test-plan)
      - [Prerequisite testing updates](#prerequisite-testing-updates)
      - [Unit tests](#unit-tests)
      - [Integration tests](#integration-tests)
      - [e2e tests](#e2e-tests)
  - [Graduation Criteria](#graduation-criteria)
    - [Alpha](#alpha)
    - [Beta](#beta)
    - [GA](#ga)
  - [Upgrade / Downgrade Strategy](#upgrade--downgrade-strategy)
  - [Version Skew Strategy](#version-skew-strategy)
- [Production Readiness Review Questionnaire](#production-readiness-review-questionnaire)
  - [Feature Enablement and Rollback](#feature-enablement-and-rollback)
  - [Rollout, Upgrade and Rollback Planning](#rollout-upgrade-and-rollback-planning)
  - [Monitoring Requirements](#monitoring-requirements)
  - [Dependencies](#dependencies)
  - [Scalability](#scalability)
  - [Troubleshooting](#troubleshooting)
- [Implementation History](#implementation-history)
- [Drawbacks](#drawbacks)
- [Alternatives](#alternatives)
  - [Fully Invisible Static CRDs (the KEP-5793 Model)](#fully-invisible-static-crds-the-kep-5793-model)
  - [Read-Only Synthetic CRD Objects Merged into the Registry](#read-only-synthetic-crd-objects-merged-into-the-registry)
  - [Inline Per-CRD Storage Configuration in the Manifest](#inline-per-crd-storage-configuration-in-the-manifest)
  - [Extending the Override Flag Grammar](#extending-the-override-flag-grammar)
  - [A Storage Binding Without Registration](#a-storage-binding-without-registration)
  - [Renaming the etcd Override Health Checks](#renaming-the-etcd-override-health-checks)
  - [Hot-Reloadable Storage Configuration](#hot-reloadable-storage-configuration)
  - [Derived Orderings Instead of an Authored Revision](#derived-orderings-instead-of-an-authored-revision)
  - [Coordination-Based Arbitration](#coordination-based-arbitration)
  - [Server-Side Apply as the Reconciler's Write or Arbitration Mechanism](#server-side-apply-as-the-reconcilers-write-or-arbitration-mechanism)
  - [Accepting Divergent API Writes (Revert, Ignore, or Warn)](#accepting-divergent-api-writes-revert-ignore-or-warn)
  - [Delete via Manifest, and a Separate &quot;Retire&quot; Tombstone Level](#delete-via-manifest-and-a-separate-retire-tombstone-level)
  - [The Revision as a <code>CustomResourceConfiguration</code> Field](#the-revision-as-a-customresourceconfiguration-field)
  - [Per-CRD Status Lists for Fleet Convergence](#per-crd-status-lists-for-fleet-convergence)
  - [Member-Local Spec-Update Allowance](#member-local-spec-update-allowance)
- [Infrastructure Needed (Optional)](#infrastructure-needed-optional)
<!-- /toc -->

## Release Signoff Checklist

Items marked with (R) are required *prior to targeting to a milestone / release*.

- [ ] (R) Enhancement issue in release milestone, which links to KEP dir in [kubernetes/enhancements] (not the initial KEP PR)
- [ ] (R) KEP approvers have approved the KEP status as `implementable`
- [ ] (R) Design details are appropriately documented
- [ ] (R) Test plan is in place, giving consideration to SIG Architecture and SIG Testing input (including test refactors)
  - [ ] e2e Tests for all Beta API Operations (endpoints)
  - [ ] (R) Ensure GA e2e tests meet requirements for [Conformance Tests](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/conformance-tests.md)
  - [ ] (R) Minimum Two Week Window for GA e2e tests to prove flake free
- [ ] (R) Graduation criteria is in place
  - [ ] (R) [all GA Endpoints](https://github.com/kubernetes/community/pull/1806) must be hit by [Conformance Tests](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/conformance-tests.md) within one minor version of promotion to GA
- [ ] (R) Production readiness review completed
- [ ] (R) Production readiness review approved
- [ ] "Implementation History" section is up-to-date for milestone
- [ ] User-facing documentation has been created in [kubernetes/website], for publication to [kubernetes.io]
- [ ] Supporting documentation—e.g., additional design documents, links to mailing list discussions/SIG meetings, relevant PRs/issues, release notes

[kubernetes.io]: https://kubernetes.io/
[kubernetes/enhancements]: https://git.k8s.io/enhancements
[kubernetes/kubernetes]: https://git.k8s.io/kubernetes
[kubernetes/website]: https://git.k8s.io/website

## Summary

This KEP adds *static CustomResourceDefinitions*: CRDs that the kube-apiserver loads from manifest
files on local disk at startup and registers for the lifetime of the process. A static CRD is to a
CRD what a static pod is to a pod: declared in a file on the control-plane host, mirrored into the
API as an apiserver-owned object, and not mutable or deletable through the API. The one deliberate
difference from the static-pod analogy is that removing the file does not delete the object — or
anything else: absence asserts nothing. Removal is expressed by an explicit *tombstone* document,
which *demotes* the object back to an ordinary CRD (deleting a CRD cascades into deleting every
custom resource of that type, so the manifest layer never deletes data). The manifest set carries
an operator-authored *revision* that totally orders changes across an HA fleet: a staggered
rollout converges with exactly one write per change instead of members fighting over the shared
object, and rollback is an ordinary forward change carrying old content.

Because a static CRD's identity is fixed at process start by the same operator who owns the
kube-apiserver's flags, two long-requested storage controls become safe to honor for its custom
resources:

1. `--etcd-servers-overrides` may name a static CRD's group/resource, placing that custom
   resource's data in a dedicated etcd cluster.
2. `--watch-cache-sizes` may name a static CRD's group/resource with size `0`, disabling the
   watch cache for that custom resource (the same mechanism used for `events` today).

Dynamically created (API-object) CRDs continue to receive neither control. That invariant is
deliberate and load-bearing: the justification for storage-level overrides is precisely that the
type-to-process binding is known at process start, established by the flag owner.

A durable *storage binding witness* recorded on the CRD object, checked on every request, makes it
impossible for a configuration change to silently relocate a resource's data to a different etcd
cluster: mismatches fail loudly (HTTP 503 for that one resource) instead of serving an empty
backend.

This KEP is modeled on KEP-5793 (Manifest Based Admission Control Config) — same loader, same file
watching, same metric naming conventions — but deliberately departs from its invisibility model,
because a CRD defines first-class API resources whose data outlives the process. See
[Why Real API Objects Rather Than an Invisible Parallel Universe](#why-real-api-objects-rather-than-an-invisible-parallel-universe).

## Motivation

Two concrete storage controls exist for built-in resources and are impossible for custom
resources today. Both have documented, long-standing production demand. Both are unsafe to offer
for dynamically created CRDs and become safe once the CRD is registered for the lifetime of the
process. That lifetime binding is the enabling precondition, and providing it is the purpose of
this KEP.

**1. Per-resource etcd server overrides (`--etcd-servers-overrides`).**

Operators shard etcd by resource today: the `/events#...` override is standard practice, and
Gardener runs a dedicated `etcd-events` cluster per control plane specifically to keep
high-churn data away from `etcd-main` ("It is possible to shard the etcd cluster based on
resource types using the `--etcd-servers-overrides` CLI flag" —
[Gardener etcd docs](https://gardener.cloud/docs/gardener/concepts/etcd/)). Custom resources
cannot participate:

- kube-apiserver accepts an override naming a CRD's group/resource, registers healthz and readyz
  checks for its backend
  (`staging/src/k8s.io/apiserver/pkg/server/options/etcd.go`, `addEtcdHealthEndpoint`,
  ~lines 438-478), applies it to the `DefaultStorageFactory`
  (`pkg/kubeapiserver/default_storage_factory_builder.go`, ~lines 149-159, with no validation
  that the resource exists) — and then silently ignores it, because custom resource storage is
  built from a `SimpleStorageFactory` that never consults overrides
  (`staging/src/k8s.io/apiextensions-apiserver/pkg/cmd/server/options/options.go`,
  `NewCRDRESTOptionsGetter`).
- The demand is documented in [kubernetes/kubernetes#81508](https://github.com/kubernetes/kubernetes/issues/81508)
  (2019), [kubernetes/kubernetes#118858](https://github.com/kubernetes/kubernetes/issues/118858)
  (2023, triage-accepted), and [kubernetes/kubernetes#139817](https://github.com/kubernetes/kubernetes/issues/139817)
  (2026, a production incident where `workflows.argoproj.io` grew large enough to pressure the
  main etcd). Two implementation PRs (#84467, #82580) were abandoned.
- The reason it was never done is on the record. @liggitt, on #81508: "Static cli options
  (--etcd-servers-overrides) that pair with dynamic api-specified options (dynamic custom
  resource API types) are pretty strange, so it's not clear this is the right way to accomplish
  parceling out custom resources to different etcd instances."

This KEP dissolves that objection rather than arguing with it: static registration makes the
type static, so the pairing of a static flag with a static type is no longer strange. The
operator who set the flag and the operator who registered the type are the same principal, and
the binding is known at process start. That is also the only contract the surrounding machinery
can honor: etcd health checks are physically un-addable after the healthz endpoint is installed,
the storage db-size monitor snapshots its backend list once, and the etcd client bakes its server
list in at construction time — a startup-frozen binding is what the existing health, metrics,
and client machinery actually supports.

**2. Per-resource watch cache control (`--watch-cache-sizes`).**

`DefaultWatchCacheSizes()` ships `events#0` and `events.events.k8s.io#0` — per-resource watch
cache disablement is established, in-production behavior for built-ins
(`pkg/kubeapiserver/default_storage_factory_builder.go`). Custom resources are the only
kube-apiserver-served storage with no per-resource cache control. The flag's help text says the
setting "is only meaningful for resources built into the apiserver, not ones defined by CRDs"
(`staging/src/k8s.io/apiserver/pkg/server/options/etcd.go`).

The truth is messier, and this KEP must own it: the line that was supposed to enforce that
restriction, `etcdOptionsCopy.WatchCacheSizes = nil // this control is not provided for custom
resources` (`apiextensions-apiserver/pkg/cmd/server/options/options.go`, line 157), has been
dead code since commit `e9e4acb1dde` (~v1.28) — the getter two lines later is built from the
*original* `etcdOptions`, not the copy, so `--watch-cache-sizes=<plural>.<group>#0` already
disables the watch cache for *any* CRD, dynamic ones included, contradicting the comment, the
flag help, and with zero tests pinning either behavior. This has been verified empirically at
HEAD. So the KEP's job for this motivation is not to add a capability but to make an accidental,
undocumented, unscoped capability intentional, tested, and scoped to static CRDs — and to decide
what happens to the accidental behavior for dynamic CRDs
(see [Bug Fixes Carried by This KEP](#bug-fixes-carried-by-this-kep)).

**3. Co-benefit: an undeletable platform type.**

Today, anyone with `delete` permission on `customresourcedefinitions` triggers an irreversible
cascade: the CRD finalizer lists every custom resource of the type and issues `DeleteCollection`
per namespace (`apiextensions-apiserver/pkg/controller/finalizer/crd_finalizer.go`,
`deleteInstances`), with no `--cascade` control, no confirmation, and no per-CR opt-out. For a
platform-critical type — and doubly so for one whose data lives in a dedicated etcd cluster the
operator provisioned — this is a data-loss footgun with no guardrail. Static registration closes
it categorically: the CRD object is admission-protected against API deletion and mutation, in the
same spirit as KEP-5793's self-protection story, but with irreversible data loss rather than
policy bypass as the stake.

**4. Minor benefit: deterministic availability.**

A static CRD is known at flag-parse time, so its type can be Established and served
deterministically as part of startup rather than eventually after the CRD informer syncs and
status controllers converge. This is the CRD analogue of KEP-5793's "active before the first
request" property, though weaker (see Non-Goals: the custom resources themselves still live in
etcd, so etcd-independence is not on offer and is not claimed).

### Goals

1. Register CRDs from manifest files on local disk for the lifetime of the kube-apiserver
   process, using the same loader, file-watching, and validation conventions as KEP-5793.
2. Materialize each static CRD as a real, etcd-backed `CustomResourceDefinition` object owned by
   the apiserver, so that discovery, OpenAPI, RBAC aggregation, garbage collection, quota,
   namespace deletion, `kubectl`, storage version migration, and `status.storedVersions`
   bookkeeping all work unchanged.
3. Protect static CRD objects from API-driven spec mutation and deletion via a compiled-in
   admission plugin, while allowing status writes (so existing status controllers and the
   storage version migrator keep working) and allowing writes whose result is identical to the
   manifest (so `kubectl apply` of the same content is not an error).
4. Honor `--etcd-servers-overrides` for a custom resource if and only if its CRD is
   statically registered, with a durable witness and a pre-activation data-safety check that
   make silent data relocation structurally impossible.
5. Honor `--watch-cache-sizes` (disable-only, size `0`) for a custom resource if and only if its
   CRD is statically registered, and pin the semantics with tests.
6. Preserve the invariant that dynamically created CRDs never receive storage-level overrides,
   structurally (the override binding set is computed at startup from the manifest set; a
   dynamic CRD has no entry) rather than by a bypassable runtime check.
7. Never delete custom resource data. No operation in this feature — registration, reload,
   demotion, gate disablement, rollback — deletes or relocates stored custom resources.
8. Keep custom resource etcd key layout byte-identical to dynamic CRDs
   (`/registry/<group>/<plural>/...`), so a CRD can move between static and dynamic registration
   without touching data.
9. Make every supported lifecycle transition — introduce, change, roll back, promote an existing
   dynamic CRD to static, demote back to dynamic — expressible as manifest changes alone,
   ordered by an operator-authored revision, with no step requiring a hand-written API write.
   The valid transitions are enumerated in
   [Transitions Between Dynamic and Static Registration](#transitions-between-dynamic-and-static-registration).

### Non-Goals

1. Serving custom resources while etcd is unavailable. The custom resources live in etcd (or in
   an override etcd), and the RBAC objects that authorize access to them live in etcd. Unlike
   KEP-5793's admission objects, etcd-independence is incoherent for this feature and is not
   claimed. The materialized CRD object's etcd dependency therefore costs nothing.
2. Hot-reloading storage configuration. Etcd server bindings are startup-frozen (see
   [Alternatives](#hot-reloadable-storage-configuration) for the mechanical reasons: healthz
   registration closes at install time, the db-size monitor snapshots once, rebinding a live
   GroupResource is data-unsafe, and the etcd client captures its server list at construction).
3. Sizing the watch cache. Only disablement (`#0`) is offered. Positive sizes have been
   logged-and-dropped since watch cache sizing became dynamic
   (`etcd.go`, `GetRESTOptions`: "Dropping watch-cache-size for %v - watchCache size is now
   dynamic"); reviving them would re-litigate dynamic sizing and would not bound the dominant
   memory cost (the object store, which has never been size-bounded).
4. Per-CRD storage media type, storage encoding, per-override etcd TLS, or etcd prefix
   overrides. Custom resource storage encoding remains fixed at the existing chokepoint
   (unstructured JSON, or CBOR under `CBORServingAndStorage`). Per-override TLS is a real gap
   for dedicated-etcd deployments and is named as a beta consideration, not an alpha feature.
5. Migrating data between etcd backends. Changing a resource's backend is an operator-managed,
   etcd-level data migration; this KEP ships the guardrails that prevent doing it accidentally
   and the runbook for doing it deliberately, not tooling that performs it.
6. Cross-apiserver synchronization of manifest files. As in KEP-5793, each API server is
   configured individually by external means; unlike KEP-5793, identical configuration across
   members is a *hard requirement* here rather than a soft expectation, because divergence in
   storage bindings is data-plane split-brain rather than policy drift. The design makes
   divergence loud (per-resource 503) and detectable (config-hash metrics) rather than silent.
7. Conversion webhooks for static CRDs (alpha). `spec.conversion.strategy` must be `None` or
   absent. A conversion webhook is a runtime dependency resolved through the Services informer,
   its failure mode is a 500 on read rather than fail-open, and cert-manager's CA injector
   cannot patch a write-protected object. URL-only webhook conversion with static CA bundles is
   a beta consideration.
8. Registering any resource type other than `apiextensions.k8s.io/v1` CustomResourceDefinitions.
   (The manifest *directory* additionally accepts two small `apiserver.config.k8s.io`
   configuration documents — the manifest-set revision and tombstones — which are configuration,
   not registrations, and never materialize as API objects.)
9. Automatic storage version migration. A manifest version-drop blocked by the `storedVersions`
   guard is surfaced, never auto-resolved: creating a `storagemigration.k8s.io`
   StorageVersionMigration object — a cluster-wide rewrite of every custom resource of the type
   — remains an explicit act with an acting principal in the audit log. An opt-in automatic mode
   is a beta consideration (see
   [storedVersions and Storage Version Migration](#storedversions-and-storage-version-migration)).

## Proposal

kube-apiserver gains a flag naming a directory of plain CRD manifests. At startup, the manifests
are strictly decoded and validated with the same validation the REST API applies; structural
errors are fatal startup errors. A reconciler then creates or adopts a matching
`CustomResourceDefinition` object in etcd for each manifest, marked with an ownership label, a
content-hash annotation, and a revision record, and keeps it converged (recreating it if
deleted out-of-band, restoring its spec if changed out-of-band). A compiled-in admission plugin
denies API-driven spec changes and deletion of these objects while restoring the server-owned
markers that PUT-shaped clients omit. Everything downstream of the CRD object — naming and establishing controllers,
discovery, OpenAPI, aggregated discovery, APIService autoregistration, garbage collection, quota,
namespace deletion, `kubectl wait --for=condition=Established`, storage version migration —
operates on a real object and needs no changes.

The manifest directory also carries two small documents in the configuration API group: a
required singleton `CustomResourceManifestSet` declaring an operator-authored, monotonically
increasing `revision` for the whole set, and zero or more `CustomResourceDefinitionTombstone`
documents, each expressing the demotion of one CRD. The reconciler records the revision on every
object it writes and arbitrates purely on it: a member holding a strictly newer revision writes
once; a member holding an older one defers — serving and protecting normally, writing nothing;
equal revisions with different content freeze loudly as a misconfiguration. This is what makes a
staggered rollout (config-management push, static-pod manifest edit, machine replacement — all
inherently node-by-node) converge with exactly one spec write per changed CRD, makes rollback an
ordinary forward change carrying old content, and makes removal and demotion declarative rather
than restart-plus-hand-written-API-write. See
[Manifest Revisions and Convergence](#manifest-revisions-and-convergence) and
[Declarative Removal: Tombstones](#declarative-removal-tombstones).

Separately gated, `--etcd-servers-overrides` and `--watch-cache-sizes` entries whose
GroupResource matches a statically registered CRD become effective for that custom resource's
storage, through a small override-aware storage factory at the existing single chokepoint
(`NewCRDRESTOptionsGetter`). An etcd override activates only after a pre-activation check proves
the source backend holds no data for that resource, and the active binding is recorded as a
witness annotation on the CRD object. Every request to the resource verifies the witness against
the locally effective binding; a mismatch — a skewed member, a changed flag, a downgraded
binary's leftover state — returns a descriptive 503 for that one resource instead of silently
reading a different (empty) backend.

### User Stories

#### Story 1: A Dedicated etcd Cluster for One High-Volume Custom Resource

As the operator of a cluster where `workflows.argoproj.io` holds hundreds of thousands of
objects and dominates etcd storage (the scenario in kubernetes/kubernetes#139817), I provision a
second etcd cluster, add the CRD manifest to `/etc/kubernetes/crd-manifests/` on each control
plane node, point the custom resource config at that directory, and set:

```
--custom-resource-config=/etc/kubernetes/custom-resource-config.yaml
--etcd-servers-overrides=argoproj.io/workflows#https://etcd-wf-0:2379;https://etcd-wf-1:2379
```

```yaml
# /etc/kubernetes/custom-resource-config.yaml
apiVersion: apiserver.config.k8s.io/v1alpha1
kind: CustomResourceConfiguration
staticManifestsDir: "/etc/kubernetes/crd-manifests/"
```

On a fresh cluster, workflow data lands in the dedicated cluster from the first write, the
dedicated backend is health-checked (this already works today via the flag), compacted, and
visible in storage metrics. On an existing cluster with workflow data in the main etcd, the
apiserver *refuses to activate the override* (the pre-activation check finds data in the main
backend), keeps serving safely from the main backend, and tells me — via a startup log, a
condition, and a metric — that I must run the documented migration runbook first. My data does
not silently vanish behind an empty backend in either case.

#### Story 2: A Platform Type That Cannot Be Deleted Through the API

As a platform administrator, my platform's core configuration type is a CRD whose custom
resources drive node provisioning. Today a single `kubectl delete crd` — by a compromised
credential, a GitOps prune misfire, or a mistaken administrator — cascades into deleting every
instance. By moving the CRD into the static manifest directory, the definition can no longer be
deleted or altered through the API on any correctly configured member; the failure message
explains that the definition is server-managed, names the data loss it prevented, and points at
the supported decommissioning procedure (never at a file — see Denial Messages). `kubectl apply`
of the identical CRD YAML (for example from an operator's install bundle) still succeeds as a
no-op, so existing install tooling does not break.

#### Story 3: Disabling the Watch Cache for a Large, Cold Custom Resource

As an operator of a cluster with a CRD holding several hundred thousand large objects that are
written in bursts, listed rarely, and watched by only one controller, I do not want the
apiserver to hold a decoded copy of every object (as `*unstructured.Unstructured*`, materially
heavier per byte than typed structs) plus an event history ring in memory on every apiserver.
With the CRD statically registered, I set `--watch-cache-sizes=bigthings.example.com#0` — the
same mechanism the apiserver applies to `events` by default — accepting the documented costs
(each watcher becomes a separate etcd watch, each write gains an etcd read, no periodic
bookmarks). Note this is deliberately the *opposite* workload profile from Story 1: the
watch-cache control is for large-and-cold resources; write-hot, widely-watched resources should
keep the cache and use the etcd override instead.

#### Story 4: Changing a Manifest CRD Across a Staggered Fleet, and Rolling It Back

As a platform administrator, my CRD manifests live in a git repository, authored and reviewed
long before any cluster actuates them. Rollout to my three control-plane nodes is inherently
staggered — a config-management push, a static-pod manifest edit, or a Cluster API machine
rollout, all node-by-node over minutes to hours (or days, when a canary node soaks). When I
merge a schema change I also raise `revision` in the set's `CustomResourceManifestSet` document
(my CI lints that any change under the manifest directory changes the revision). The first node
to receive the commit writes the new spec once; the moment that write lands, every member serves
the new schema, because members serve the shared object, not their local files. The other nodes
then observe an equal-or-newer record on the object and write nothing: nothing flaps, no watch
storms, and the object's `static-revision` annotation tells me which configuration is in effect.
When a change turns out bad, I `git revert` the content **and raise the revision again** —
rollback is a newer revision carrying older content, the same model as `kubectl rollout undo`.
If I forget the bump, the fleet freezes that one CRD loudly (a condition, a metric, and a log
line naming both content hashes and the one-line fix) rather than flapping or silently adopting
either side.

#### Story 5: An API-Only User Installing, Syncing, and Uninstalling Against a Manifest CRD

As a platform consumer with no access to the control plane, my tooling (Helm, Argo CD, Flux,
OLM, `kubectl apply`, a CI suite using controller-runtime's CRD installer against a shared
cluster) manages a bundle that includes `workflows.argoproj.io`, which the platform team has
promoted to a static CRD. Applies of identical content keep succeeding forever — client-side
apply, server-side apply, and PUT-shaped writes (`kubectl replace`, envtest's `CreateCRDs`,
Argo CD `Replace=true`) all behave the same, because the server restores its own markers and
judges only whether my write would change the spec. My GitOps loop stays green at zero cost:
identical writes never reach etcd. If my bundle diverges from the platform definition, I get a
denial that explains itself — no filenames, a stable `StaticCustomResourceDefinition:` sentinel,
the field my write would change, and who to ask (see
[Denial Messages](#denial-messages)). If my uninstall tries `kubectl delete crd`, the denial
tells me the definition outlives my install and what deleting it would have destroyed. When the
platform team eventually hands the CRD back to my team (demotes it), my tooling observes
nothing: same object, same UID, same data, and my next sync succeeds exactly as the previous
one did.

### Notes/Constraints/Caveats

1. CRD manifests are plain `apiextensions.k8s.io/v1` CustomResourceDefinition YAML — the same
   content you would `kubectl apply`. This is deliberate: promoting a dynamic CRD is a file
   copy plus a revision bump, and demoting one back is a four-line tombstone — no bespoke
   format in either direction.
2. The two storage flags keep their existing, mutually inconsistent grammars
   (`group/resource#servers` vs `resource[.group]#size`). Aliasing them is unsafe (see
   [Flag Grammar Asymmetry](#flag-grammar-asymmetry)); the design compensates with diagnostics.
3. Per-API-server configuration. Each API server loads its own manifest directory and flags. In
   HA setups, identical manifests *and* identical storage flags across members are a hard
   requirement; the design detects and contains violations (per-resource 503, drift-detection
   hash metrics) but cannot repair them.
4. Static CRDs pass through the ordinary admission chain for their custom resources, and
   dynamic admission (webhooks, VAP/MAP) can target their CRs. This is an intentional asymmetry
   with KEP-5793: manifest-based *admission* escapes the API universe; static CRDs' *data* does
   not.
5. Watch cache disablement is an opt-out from the entire modern cache stack for that resource:
   consistent reads from cache (KEP-2340), resilient watchcache initialization (KEP-4568),
   snapshottable cache lists (KEP-4988), and sharded list/watch (KEP-5866) all live on the
   cacher. It is a deliberate, per-resource reversion to direct etcd serving.
6. Encryption at rest composes unchanged: the transformer-wrapping storage factory replaces only
   the transformer and leaves the server list alone, so a resource in a dedicated etcd still
   gets whatever transformer the encryption configuration selects (including `*.*`). "Separate
   etcd" does not mean "separate encryption".
7. **A manifest change is fleet-atomic at first landing.** Every member serves a custom
   resource from the shared CRD object in etcd (the crdHandler builds serving state from the
   CRD lister, never from local files), so the instant the first rolled member's reconciler
   writes a change, every member serves it. There is no per-node canary for a static CRD
   change; validation happens in staging clusters and via `kubectl --dry-run=server`. The
   compensating property, named as a design virtue: structurally invalid manifests are fatal
   before the listener opens, so every health-gated rollout mechanism (kubelet static-pod
   restart backoff, Cluster API machine rollouts, kOps validate-between-nodes, OpenShift
   revision rollouts) halts at the first member, with a blast radius of one node.
8. Delivery mechanics: kubelet projects ConfigMap and projected volumes atomically via the
   `..data` symlink swap (`pkg/volume/util/atomic_writer.go`), so a projected directory is
   never observed half-written — and the loader must skip directory entries whose names begin
   with `..`. A `subPath` ConfigMap mount never receives updates (a silently frozen
   configuration, visible as a stale revision gauge once the fleet moves on). rsync/scp
   delivery can present partial directories; the required `CustomResourceManifestSet` document
   and the reload debounce contain this (see Manifest Reload).
9. Manifest changes and storage-flag changes are never combined in one commit, and no new
   revision should ship while a restart-only rollout is incomplete. Promote: manifest change
   first, verify fleet convergence, then flags. Retire a bound CRD: drain data back, drop the
   flag entry, then tombstone. Violations are loud and transient rather than silent (see the
   Workflow Matrix), but the ordering rule avoids the noise.

### Risks and Mitigations

| Risk | Description | Mitigation |
|------|-------------|------------|
| Silent data relocation | An etcd override activating (or deactivating) against a resource that already has data would make every object invisible: LIST returns empty, controllers react to an empty world. | Pre-activation emptiness check refuses to activate a binding when the source backend holds data; the per-request binding guard 503s on any witness mismatch. The failure mode is *loud unavailability of one resource*, never silent emptiness. |
| Downgrade to a pre-feature binary | An older binary ignores the witness and silently serves the (empty) default backend for a resource bound to an override. This is the worst outcome in the design and cannot be guarded by code that does not exist in the old binary. | Documented hard requirement: migrate bound data back to the default backend and clear witnesses before downgrading across the feature boundary; verify `apiserver_custom_resource_storage_binding{state="override"} == 0` first. Named in Version Skew, Upgrade/Downgrade, and Troubleshooting. |
| HA manifest or flag skew | Members holding different manifest content for the same CRD would, naively, force-update the shared object in turn for the entire duration of a staggered rollout; members disagreeing about a storage binding would be data split-brain. | The manifest-set revision arbitrates: a strictly newer configuration writes exactly once, an older one defers (serving and protecting normally), and equal revisions with different content freeze loudly — see Manifest Revisions and Convergence. Real objects eliminate the APIService create/delete war that an invisible design has; the witness guard turns binding skew into a per-resource 503 on the disagreeing members; `..._last_config_info` and `..._last_storage_config_info` hashes expose drift per member for alerting. |
| Forgotten or misused revision | An admin changes manifest content without raising `revision` (or two branches raise it to the same value with different content); or rollout tooling (`clusterctl alpha rollout undo`, `git reset`) restores an older configuration wholesale, revision included. | Equal-revision-different-content freezes that one CRD loudly (no write, previous spec keeps serving, condition + metric + log naming both hashes and the one-line remedy); an older configuration goes inert rather than writing backwards, with a gauge and a log naming the exact minimum revision that would win. Nothing flaps and nothing is silently adopted in either case; a recommended CI lint (content changed => revision changed) makes forgetting rare. |
| Fleet-atomic change, no canary | Because members serve the shared object, a bad-but-valid manifest change (a CEL rule rejecting live objects, a `served: false` on a version in use) reaches 100% of the fleet the moment the first rolled member writes it. | Documented constraint (Notes 7): there is no per-node canary for a static CRD change; validate in a staging cluster and with `kubectl --dry-run=server` against a canary of the target version. Structurally invalid content is fatal-at-load and halts health-gated rollouts at one node. |
| Schema narrowing hides data on read | Removing a property from a manifest schema does not merely reject future writes: the storage codec prunes to the *current* structural schema on read (`customresource_handler.go`, `schemaCoercingDecoder`), so existing stored fields become invisible to every client fleet-wide within one reload interval, with no write and no error, and are destroyed on each object's next write. | Named here because it is the same failure shape (silent emptiness) the storage witness exists to prevent, arriving via the schema door. Mitigation is procedural at alpha: schema-narrowing changes get the same staging-cluster scrutiny as storage changes; an acknowledgement gate for narrowing reloads is a beta consideration. |
| Protection begins at adoption, not at manifest claim | Write/delete protection keys on the object's `static=true` label (fleet-uniform), which the reconciler applies at adoption. A first-time promotion of an existing dynamic CRD is unprotected until the reconciler's first write lands; a terminally divergent adoption (e.g. scope differs) never gets the label at all. | The agreeing-promotion path stamps markers in the first write, metadata-only and immediate after startup/reload; the promotion pre-flight (see Transitions) verifies agreement first. A terminal conflict deliberately leaves the object editable so the operator can resolve it (protecting an unadoptable object would freeze the only repair path). Documented window; `StaticSynced=False` names it. |
| Wedged GitOps teardown | Unconditional DELETE denial wedges tooling that prunes by finalizer: an Argo CD Application carrying the CRD under `resources-finalizer.argocd.argoproj.io` sits in `Deleting` forever; an operator that force-installs its own CRDs at boot (controller-runtime `CreateCRDs`, OLM) CrashLoopBackOffs on a divergent bundle. | Documented per-tool guidance (see Transitions pre-flight and the denial-message docs link): annotate the CRD `Prune=false` in Argo sources before promotion; identify self-installing controllers and confirm their bundled spec matches before promoting. The denial message carries a stable sentinel so the failure is greppable from inside a wedged pipeline. |
| Per-resource 503 blast radius | A guard-blocked resource stays in discovery (removing it would make namespace deletion silently skip its CRs), so namespace deletion and GC monitors for it error-and-retry: namespaces containing (or possibly containing) that resource stay Terminating, and the GC graph for it stalls, for the duration of the mismatch window. Behind a load balancer, a partial-fleet mismatch presents as intermittent failures proportional to the unrolled fraction. | Documented as the deliberate trade (unavailable strictly beats split-brain); the 503 carries a stable reason (`StorageBindingMismatch`), both binding values, and the remediation; `apiserver_custom_resource_storage_binding{state="mismatch"}` is the alert; rolling-enable procedure is required to complete promptly, not linger. |
| Watch cache disablement misuse | Disabling the cache on a write-hot, widely-watched resource increases etcd load (per-write read, per-watcher etcd watch, no serialization memoization) — the opposite of the intended relief. | Documentation states the intended envelope (large-and-cold); the APF list-cost correction (scoped to these resources) prevents under-charged full-etcd LISTs; Scalability section carries the honest cost table. |
| Accidental behavior regression | Fixing the dead `WatchCacheSizes = nil` code removes a capability that has silently worked for dynamic CRDs since ~v1.28. | Shipped as a prerequisite change with its own opt-out window, startup logging of affected entries, and a release note, per the KEP-5793 `ExcludeAdmissionWebhookVirtualResources` precedent. Decided (fix-and-gate, not grandfather); see Bug Fixes. |
| Witness planting | The guard honors the witness annotation, so writing a bogus witness to a CRD would 503 that resource. | The admission plugin denies writes to reserved `storage.apiextensions.k8s.io/*` keys on **all** CRDs (not only static ones) whenever the storage gate is enabled, on both the main resource and the status subresource; the only permitted user write is setting it to the local member's computed value (the unlock path). Residual risk while gates are off is documented in Write Protection. |
| Storage capture escalation | A user creating a *dynamic* CRD at a flag-named GroupResource must not inherit the override (CRD creation must never redirect where data is written). | Structural: the override binding set is computed at startup from the static manifest set; a dynamic CRD has no entry. Additionally, creating a dynamic CRD whose name a static CRD claims is denied outright. |
| Squatting a claimed name before enablement | A pre-existing dynamic CRD with the static CRD's name but a divergent spec prevents the manifest spec from applying. | Adoption force-updates when update validation permits; a terminal conflict never mutates the object and never blocks startup or readiness — it is surfaced via `StaticSynced=False`, `apiserver_static_crd_unsynced`, and logs, and the operator resolves it. |
| Protection is a fleet property | Admission protection applies only on members running the gate; during the enablement roll (and any rollback), un-upgraded members accept writes/deletes other members deny. | Documented enablement procedure: enable the gate fleet-wide (empty manifest dir is fine) before distributing manifests; verify hash agreement via `..._last_config_info`. |
| Test-environment divergence | envtest and similar harnesses run without static manifests; a test asserting that `kubectl delete crd` or a CRD update succeeds passes locally and fails against a static-CRD cluster. | Documented; the denial message is stable and greppable so failures are self-explaining. |
| GitOps friction | Continuous-apply tooling reconciling the CRD sees provenance markers as drift and, if the manifest and the repo diverge, a hard denial that stays red on every sync. | Identical-content writes succeed as no-ops for every write shape — apply, server-side apply, and PUT (`kubectl replace`, envtest, Argo `Replace=true`) — because reserved markers are server-owned and restored rather than demanded of clients, and equality is judged against the object (see Write Protection). The steady state is free: identical writes never reach etcd. A red sync on genuine divergence is the intended coordination signal — two declarers disagree and a human must choose; the denial message explains itself without filenames. Marker placement (metadata vs a status stanza) is an open issue tracked in The Storage Binding Witness. |

## Design Details

### Why Real API Objects Rather Than an Invisible Parallel Universe

KEP-5793's model — manifest objects invisible to the API, in a parallel universe — is the wrong
model for CRDs, and this KEP departs from it deliberately. The argument, in three sentences:

A `ValidatingWebhookConfiguration` is inert data; hiding it creates a parallel universe with no
API-visible consequence. A `CustomResourceDefinition` defines API resources: its custom
resources appear in discovery, OpenAPI, RBAC, garbage collection, quota, and namespace deletion,
and occupy etcd keys that outlive the process. Hiding the definition therefore does not create a
parallel universe; it creates a *served resource with no inspectable definition* — and it does
not even achieve invisibility, because the aggregator builds legacy `/apis` discovery solely
from persisted APIService objects
(`staging/src/k8s.io/kube-aggregator/pkg/apiserver/handler_apis.go`), so one etcd object per
group/version leaks regardless, and under HA skew the members without the manifest actively
delete the APIService the members with it create
(`kube-aggregator/pkg/controllers/autoregister/autoregister_controller.go`, the
`desired == nil` deletion case).

Three specific facts close the decision:

1. **`status.storedVersions` has no other home.** It is the only durable record of which
   serialized apiVersions exist in etcd. `ValidateCustomResourceDefinitionStoredVersions`
   (`apiextensions-apiserver/pkg/apis/apiextensions/validation/validation.go`, line 263) is the
   only thing preventing a version drop that renders stored objects permanently undecodable (the
   CR converter hard-fails reads of an apiVersion absent from `spec.versions`, breaking GET and
   LIST — including the finalizer's own LIST). A file has no place to keep this record; a
   per-member state file does not survive HA.
2. **The storage binding witness has no other home.** The data-safety mechanism at the heart of
   the storage half requires a durable, cluster-consistent, per-resource record readable by
   every member. A CRD object in the default backend is exactly that.
3. **The ecosystem's CRD idioms are load-bearing.** `kubectl wait --for=condition=Established
   crd/...` is the universal CRD install gate (Helm CRD ordering, kubebuilder `make install`,
   cert-manager and Istio installers); against a nonexistent object it hangs to timeout.
   `kubectl get crd`, server-side apply, and the in-tree storage version migrator (which looks
   the CRD object up by name and silently degrades when it is absent) all assume the object
   exists.

The in-tree precedents for "apiserver-owned, reconciled, protected API objects" are exactly the
pattern used here: automanaged APIServices (created, restored, and labeled by the apiserver on
CRDs' behalf), bootstrap RBAC policy (post-start reconciler with an opt-out annotation), and API
Priority and Fairness mandatory objects (in-memory authority, etcd mirror kept converged with
`alwaysAutoUpdateSpec`). See [Alternatives](#fully-invisible-static-crds-the-kep-5793-model) for
the full comparison, including the panic-class hazards of the synthetic approach.

### Configuration Surface

#### New Flag and Config Kind

This KEP follows the two-layer shape KEP-5793 established for manifest-based admission control:
a flag names a **versioned configuration file**, and a field inside that configuration names a
**directory of plain manifests**. The naming is deliberately mechanism-neutral — "custom
resource" is the general term, as "admission control" is for webhooks and policies — so the kind
can carry future custom-resource configuration that is not about CRD registration.

```
--custom-resource-config=/etc/kubernetes/custom-resource-config.yaml
```

Following the convention of the newer structured-configuration flags (`--authentication-config`,
`--authorization-config`) rather than the older `-file`-suffixed ones
(`--admission-control-config-file`, `--tracing-config-file`).

```yaml
apiVersion: apiserver.config.k8s.io/v1alpha1
kind: CustomResourceConfiguration
staticManifestsDir: "/etc/kubernetes/crd-manifests/"
```

`CustomResourceConfiguration` is a new kind in the existing `apiserver.config.k8s.io` group,
alongside `AdmissionConfiguration`, `EgressSelectorConfiguration`, `TracingConfiguration`,
`AuthenticationConfiguration`, and `AuthorizationConfiguration`. The name deliberately avoids
`ResourceConfiguration`: `ResourceConfig`
(`staging/src/k8s.io/apiserver/pkg/server/storage/resource_config.go`) and its flag
`--runtime-config` already mean *which API groups and resources are enabled*, and a same-package
neighbour meaning *which CRDs are statically registered* would be a durable source of confusion.

`staticManifestsDir` — the field name is identical to KEP-5793's — takes an absolute path to a
directory. All direct-children `.yaml`, `.yml`, and `.json` files are loaded in `os.ReadDir`
(alphabetical) order; multi-document YAML is supported; empty files and documents are skipped.
Glob patterns, relative paths, and recursion are not supported. If the flag is set and the
config file or the directory does not exist or is not readable, the API server fails to start. A
readable-but-empty directory loads zero static CRDs (see the Workflow Matrix for the interaction
with previously materialized objects).

Both layers are watched: a change to either the config file or the manifest directory triggers
the reload path described in Manifest Reload, and the config file's own content contributes to
the `..._last_config_info` hash so that per-member drift in *either* layer is detectable.

`CustomResourceConfiguration` deliberately does **not** carry the manifest-set revision or any
other per-set lifecycle declaration. The revision orders the *directory's content*, and the
config file and the directory are delivered through independent channels — two mounts, two
ConfigMaps, frequently two repositories (cluster-bootstrap vs platform) — that can and do skew
against each other on a single node for minutes, or indefinitely under a `subPath` mount. A
revision that arrives before or after the content it labels would mis-record or freeze a
perfectly authored change. The revision therefore lives inside the directory itself, in the
`CustomResourceManifestSet` document (next section), riding the same atomic `..data` swap, the
same content hash, and the same review diff as the manifests it orders.

**Decided: versioned config kind, matching KEP-5793.** A bare
`--static-crd-manifest-dir` flag pointing straight at the directory was the alpha proposal and
is rejected. KEP-5793 requires files at two layers — an existing flag naming an
`apiserver.config.k8s.io` object, whose field names a manifest directory — and consistency with
the immediately adjacent, already-`implementable` KEP outweighs the smaller diff. Note that
KEP-5793 added *zero* flags because `AdmissionConfiguration` already existed and webhooks are
admission plugins; CRD registration has no such host object, so exactly one flag is added either
way. The choice is only what that flag points at.

The config kind is also the sole plausible home for what beta is expected to need and flag
grammar cannot express: a stable operator-supplied backend identity (see the witness UNRESOLVED),
per-GroupResource rebinding acknowledgements, and per-backend etcd TLS. Introducing it now avoids
adding it after the fact around an established flag.

This remains compatible with the committed decision that storage *settings* stay on the two
existing flags: `CustomResourceConfiguration` carries registration configuration, never storage
endpoints — which the Design Rule section independently forbids placing where API readers can
reach.

Accepted costs: a new config API kind, codegen, validation, a round-trip fuzz test the
`apiserver.config.k8s.io` group currently lacks, and the v1alpha1/v1beta1 group-name split
(`ConfigSchemeGroupVersion` vs `SchemeGroupVersion`) that a new kind must navigate.


#### Manifest File Format

A manifest document is one of exactly three kinds; any other kind is a fatal load error
(KEP-5793 precedent: a manifest layer may accept several kinds):

1. A plain `apiextensions.k8s.io/v1` **CustomResourceDefinition** — exactly the YAML you would
   `kubectl apply`. There is no *bespoke* envelope kind and no storage stanza: the manifest
   declares what the cluster's API looks like; the flags declare what this process's storage
   looks like, in the same place every other resource's storage is configured.
2. An `apiserver.config.k8s.io/v1alpha1` **CustomResourceManifestSet** — the required singleton
   declaring the set's `revision` (example below; semantics in
   [Manifest Revisions and Convergence](#manifest-revisions-and-convergence)).
3. An `apiserver.config.k8s.io/v1alpha1` **CustomResourceDefinitionTombstone** — the
   declarative expression of removal (example below; semantics in
   [Declarative Removal: Tombstones](#declarative-removal-tombstones)).

Matching KEP-5793's manifest layer, a file may contain a single document, several separated by
YAML document markers (`---`), or a `v1` `List` whose items are all of accepted kinds. The
forms are equivalent after loading; `List` is accepted because it is what
`kubectl get crd -o yaml` emits and what existing CRD bundles already ship, so operators can
point `staticManifestsDir` at files they already have.

```yaml
# /etc/kubernetes/crd-manifests/argo-workflows.yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: workflows.argoproj.io          # must equal <plural>.<group> (normal CRD validation)
spec:
  group: argoproj.io
  scope: Namespaced
  names:
    plural: workflows
    singular: workflow
    kind: Workflow
    listKind: WorkflowList
    shortNames: [wf]
  versions:
  - name: v1alpha1                      # multiple versions ARE allowed (see storedVersions)
    served: true
    storage: true
    schema:
      openAPIV3Schema:
        type: object
        x-kubernetes-preserve-unknown-fields: true
  conversion:
    strategy: None                      # alpha: Webhook is rejected at load
# status: MUST be absent. A manifest carrying status fails startup.
```

```yaml
# /etc/kubernetes/crd-manifests/manifest-set.yaml
# REQUIRED whenever the directory contains any CustomResourceDefinition or tombstone document;
# exactly one per directory (zero, when other documents exist, or two are fatal load errors).
# An entirely empty directory remains valid and loads zero static CRDs.
apiVersion: apiserver.config.k8s.io/v1alpha1
kind: CustomResourceManifestSet
# Operator-authored, monotonically increasing, >= 0, one value for the whole set. Raised in the
# SAME commit as any content change — including a rollback: a rollback is a new, higher
# revision carrying old content. See Manifest Revisions and Convergence.
revision: 48
# Optional recovery hatch, normally absent: recorded revisions this configuration deliberately
# replaces despite carrying a lower number (raw git-reset recovery; hijacked-revision
# recovery). Entries that do not exceed `revision` are vacuous: warned about and ignored,
# never fatal — a leftover entry must not brick a later boot.
#supersedes: [49]
```

```yaml
# /etc/kubernetes/crd-manifests/argo-workflows.yaml
# A tombstone conventionally REPLACES the CRD's own file in place, so the retiring diff reads
# as "-2000 lines of schema, +4 lines of explicit intent".
apiVersion: apiserver.config.k8s.io/v1alpha1
kind: CustomResourceDefinitionTombstone
crdName: workflows.argoproj.io
action: Demote            # sole alpha value; strict decoding makes future values fail closed
```

Why the two config-group kinds live in the *directory* rather than in the
`CustomResourceConfiguration` file: they order and retire the directory's content, so they must
be delivered atomically with it — one `..data` swap, one content hash, one repository, one
reviewable diff. Two independently delivered files can skew on a single node, and a revision
that arrives before or after the content it labels would mis-order a perfectly authored change.
They are `apiserver.config.k8s.io` kinds, not `apiextensions.k8s.io` ones, because they are
configuration, not API objects: they never materialize, and they must fail loudly if someone
`kubectl apply`s them against a cluster.

A useful safety property falls out of requiring the `CustomResourceManifestSet` document:
because removal intent (a tombstone) and ordering intent (the revision) are both *positive*
documents, no infrastructure failure can synthesize either. A broken, empty, half-written,
unreadable, or `subPath`-frozen mount produces *absence*, and absence asserts nothing — it can
never demote, delete, or reorder anything. A directory that contains CRDs but no manifest-set
document is structurally invalid (fatal at startup; whole-reload-rejected at reload), which is
what makes a transiently empty or half-projected mount read as "broken", never as "intentional
wholesale removal".

#### Existing Storage Flags

```
# EXISTING flag. Newly effective for a custom resource IFF that GroupResource is statically
# registered AND CustomResourceStorageOverrides is enabled (activation rules below).
--etcd-servers-overrides=argoproj.io/workflows#https://etcd-wf-0:2379;https://etcd-wf-1:2379

# EXISTING flag. Same conjunctive condition. Only size 0 is meaningful.
--watch-cache-sizes=workflows.argoproj.io#0
```

Both flags' help text is corrected: the current text claims the settings do not apply to CRDs,
which is false today for `--watch-cache-sizes` (dead-code leak) and becomes conditionally false
for both under this KEP.

`<group>/*` wildcard entries in `--etcd-servers-overrides` are ignored for custom resources,
with a startup warning. Honoring them would mean adding a CRD to the manifest silently activates
an override for it, violating the explicit-binding rule below.

There is deliberately no `--accept-crd-storage-rebinding`-style break-glass flag. A persistent
process flag that disables a data-safety check outlives the migration it was set for and
silently disarms the check for the next, accidental rebinding. The acknowledgement path is
per-object and per-value instead: the binding guard's 503 message prints the exact
`kubectl annotate --overwrite` command that records the operator's intent (see The Storage
Binding Witness), which can only authorize the one transition the operator actually typed.

#### Flag Grammar Asymmetry

The same CRD is named `argoproj.io/workflows` by one flag and `workflows.argoproj.io` by the
other. This asymmetry is accepted, not aliased:

- Both grammars are load-bearing for built-ins today, with operator muscle memory and automation
  attached to each.
- An alias is genuinely unsafe: `ParseWatchCacheSizes` uses `schema.ParseGroupResource`, which
  splits on the *first* dot, so `--watch-cache-sizes=argoproj.io/workflows#0` parses to
  `{Group: "io/workflows", Resource: "argoproj"}` — silently wrong. Supporting the slash form
  there would require format-sniffing inside a long-stable parser, a worse bug surface than the
  asymmetry.

Instead, three diagnostics (the encryption-config validator's targeted
`events.k8s.io`-spelling error is the precedent,
`staging/src/k8s.io/apiserver/pkg/apis/apiserver/validation/validation_encryption.go`):

1. When the storage gate is on, one startup log line per entry of each flag echoes the
   *resolved* `{group, resource}` and whether it matched a built-in, a static CRD, or nothing —
   so a typo is visible in the boot log rather than in a capacity-investigation postmortem.
2. A targeted error for the cross-grammar mistake: a `--watch-cache-sizes` token containing
   `/`, or an `--etcd-servers-overrides` token without one, produces an error naming the
   correct grammar.
3. At alpha, an override or watch-cache entry matching neither a built-in nor a static CRD is a
   startup warning plus a metric (existing clusters carry stale, inert entries; failing on day
   one would break them the moment they enable the gate). At beta it becomes a fatal
   config error.

### Feature Gates and Disablement Semantics

Two gates, both `kube-apiserver`, both alpha-off:

- **`StaticCustomResourceDefinitions`** — the registration half: manifest loading, the
  reconciler, write protection, and the metrics.
- **`CustomResourceStorageOverrides`** — the storage half: honoring the two flags for static
  CRDs. Requires `StaticCustomResourceDefinitions`. Graduates independently.

What disablement does, stated precisely because the honest answer is layered:

- Disabling `StaticCustomResourceDefinitions`: the loader, reconciler, and protection plugin
  vanish. Previously materialized CRDs remain as ordinary API objects carrying inert provenance
  markers — including the revision record; discovery, RBAC, data, and tooling are untouched;
  users can now edit and delete them. Registration rollback is clean by construction.
  Re-enabling resumes arbitration exactly where it left off: the record is durable, and a spec
  edited out-of-band during the gate-off window is detected (the recorded content hash no
  longer matches the object's spec) and repaired back to the manifest — see The Reconciler's
  tamper rule.
- Disabling `CustomResourceStorageOverrides`: no *new* storage bindings can be created
  (activation, witness stamping, and the pre-activation check are off). **Already-recorded
  bindings continue to be honored**: a resource whose CRD carries a witness naming an override
  backend, where the flag still supplies that backend's server list, keeps serving from that
  backend. This is the deliberate deviation from clean gate semantics, and it exists because the
  alternative — reverting routing to the default backend on gate-off — would serve an *empty*
  resource: controllers reading "empty" recreate and delete real things, which is data loss, not
  rollback. Full rollback of a binding is therefore an explicit drain: migrate the data back to
  the default backend, clear the witness, drop the flag entry.
- If the flag entry is removed (or the gate stack can no longer resolve a server list) while a
  non-default witness stands, the resource returns 503 via the binding guard — loud
  unavailability, never silent emptiness.

```
<<[UNRESOLVED un-gated honoring surface: PRR ruling required @liggitt @deads2k ]>>
Three narrow pieces of this design remain active regardless of gate state, and PRR must rule on
each rather than the author judging:
1. The per-request binding guard (a map/annotation comparison that returns "allow" for every
   CRD without a witness — i.e. every CRD in every cluster that never used the feature).
2. Routing of already-recorded bindings (witness present + flag supplies the matching servers),
   per the rollback reasoning above.
3. Server ownership of reserved keys — `storage.apiextensions.k8s.io/*` under the storage
   gate, and `apiextensions.kubernetes.io/static*` (label, content hash, revision record,
   superseded list) under the registration gate — on **all** CRDs, both subresources: the
   mutating restore of omitted keys, the denial of divergent values, and the denial of CREATEs
   carrying them. Proposed: enforced whenever the respective gate is enabled; while gates are
   off, an existing witness is still honored (data safety) but plants by
   `patch customresourcedefinitions` holders are possible — a residual documented in Write
   Protection (for the witness, a per-resource DoS; for the revision record, a
   reconciler-freeze on re-enable, cleanable via the reserved-key-removal allowance). This is
   an admission *behavior* change for clusters using a gate without the feature on a given
   object, and the mutating phase (versus a pure-validating plugin) needs its own explicit
   ruling.
Gating all three cleanly would mean a gate-off with a live binding silently serves an empty
resource. Not gating them deviates from clean feature-gate semantics. This needs an explicit
PRR decision; the KEP proposes the layered semantics above.
<<[/UNRESOLVED]>>
```

### The Materialized CRD Object

For each manifest, the reconciler converges an object of this shape in etcd:

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: workflows.argoproj.io
  labels:
    apiextensions.kubernetes.io/static: "true"          # protection key
  annotations:
    # Content hash only. The source file name and path are deliberately NOT recorded
    # on the object — see Design Rule: No Admin-Environment Detail on API Objects.
    apiextensions.kubernetes.io/static-content-hash: "sha256:9f2c..."
    # The revision record: the manifest-set revision whose configuration this object last
    # converged to. Written atomically with every reconciler write.
    apiextensions.kubernetes.io/static-revision: "48"
    # Usually absent: recorded revisions this record disowns (mirrored, filtered `supersedes`
    # values — see Manifest Revisions and Convergence).
    apiextensions.kubernetes.io/static-superseded: "49"
    # The storage binding witness. Stamped for every static CRD, including ones on the
    # default backend. Value: "default", or sha256 over the canonicalized override server list.
    storage.apiextensions.k8s.io/etcd-servers: "sha256:71ab..."
spec: { ...exactly the manifest spec... }
status:
  acceptedNames: { ... }              # naming controller, unchanged
  storedVersions: ["v1alpha1"]        # registry strategy PrepareForCreate/Update, unchanged
  conditions:
  - { type: NamesAccepted, status: "True" }
  - { type: Established,   status: "True" }
  - { type: StaticSynced,  status: "True", reason: ReconcileSucceeded }   # new condition type
```

The label and the revision record together encode lifecycle state with no new API field:
**label present + record = managed as of that revision; label absent + record = demoted as of
that revision; no record = never asserted (an ordinary CRD).** All of these markers are
metadata-only, which is load-bearing for cost: the crdHandler tears down per-CRD storage only
when `spec` or `status.acceptedNames` change (`customresource_handler.go`
`updateCustomResourceDefinition`), the CRD registry strategy bumps `metadata.generation` only
on spec changes, and the OpenAPI aggregation cache keys on `{UID, Generation}` — so recording,
confirming, demoting, and re-promoting-without-spec-change are all free at every layer. The
only expensive operation in the entire lifecycle model is an actual spec change.

Because the object is real, the naming, establishing, nonstructuralschema, and apiapproval
controllers, the discovery and OpenAPI v2/v3 controllers, crd-autoregistration, and the storage
version migrator all operate on it unchanged. None of them are modified by this KEP. The CRD
finalizer is likewise unmodified: a protected object can never acquire a deletionTimestamp, so
the finalizer never processes it (and adoption of an already-Terminating object is refused — see
the Workflow Matrix). One narrow admission carve-out exists so the finalizer can *finish* work
begun before protection covered an object: see the status-subresource rules in Write
Protection.

### Design Rule: No Admin-Environment Detail on API Objects

Static CRDs are configured by a cluster operator with host filesystem access, but the objects
they produce are ordinary `CustomResourceDefinition` objects readable by anyone holding
`get customresourcedefinitions` — commonly granted, and granted to every controller that
discovers types. The manifest directory layout, the file names within it, the etcd endpoints
behind an override, and any TLS material paths are **operator-environment detail**, not API
content, and this KEP treats leaking them onto readable objects as a defect.

The rule, binding on every marker this KEP adds and on any future addition:

- **Values on API objects are hashes, fixed tokens, or operator-authored ordinals.**
  `static-content-hash` is a sha256 over the manifest content.
  `storage.apiextensions.k8s.io/etcd-servers` is either the literal `default` or a sha256 over
  the canonicalized server list — never the URLs themselves. A reader can tell *that* two
  members disagree, or *that* a CRD is bound to a non-default backend, and can compare values
  for equality; they learn no endpoint, host name, port, or path. The lifecycle model adds the
  third class: `static-revision` and `static-superseded` carry integers the operator chose,
  which order configurations and carry nothing else. The obligation that comes with the class:
  documentation recommends small monotonic integers starting at 1, and the loader logs a soft
  warning for date- or epoch-shaped revisions (`20260825`, Unix timestamps), which would
  publish the operator's change *timing* to every CRD reader. No hard cap — arbitrary limits
  were rejected — but the rule's enforcement test covers the ordinal class too.
- **No source file name or path is recorded on the object.** The manifest-file-to-CRD mapping is
  genuinely useful when debugging, so it is emitted where the operator already has host access
  and ordinary readers do not: the API server log, at startup and on every reload, one line per
  loaded manifest naming the file and the resulting CRD name. The `static-content-hash`
  annotation is what correlates an object back to a log line without exposing the path.
- **Client-visible errors carry the same restriction.** The binding guard's 503 names the flag
  that produced a binding, never the flag's value.
- **Metric labels carry the same restriction.** Config-drift metrics expose hashes
  (`..._last_config_info`, `..._last_storage_config_info`); no metric label carries a file path
  or an etcd endpoint. Where an existing STABLE metric already exposes backend identity
  positionally, this KEP does not widen it (see the `storage_cluster_id` UNRESOLVED block).

Enforcement is by review and by test: the integration suite asserts that no object materialized
by the reconciler carries a value matching the manifest directory path or any file name within
it, and that a `StorageBindingMismatch` 503 body contains no substring of the configured server
lists.

### Loading, Decoding, and Validation

The loader reuses KEP-5793's already-merged manifest machinery
(`staging/src/k8s.io/apiserver/pkg/admission/plugin/manifest/`: direct-children enumeration,
multi-document splitting, NUL-separated sha256 content hash, `filesystem.WatchUntil` with
fsnotify plus a 1-minute polling fallback).

```
<<[UNRESOLVED loader package location @aramase @BenTheElder ]>>
The shared loader currently lives under the admission plugin tree
(`k8s.io/apiserver/pkg/admission/plugin/manifest`). Proposed: promote it to a neutral package
(e.g. `k8s.io/apiserver/pkg/util/manifest`) so apiextensions-apiserver can import it without
depending on admission plugin internals. Small refactor, cross-KEP negotiation with 5793's
authors; fall back to copying the ~200 lines if promotion stalls. Blocks implementation
convenience, not the KEP.
<<[/UNRESOLVED]>>
```

Per CustomResourceDefinition document, in order:

1. Strict decode with the apiextensions install scheme (`serializer.EnableStrict`): unknown and
   duplicate fields are errors.
2. Static-CRD restrictions: `status` must be absent (operators cannot hand-forge
   `storedVersions`); `spec.conversion.strategy` must be `None` or unset; the group/version must
   not be served by a compiled-in API; the GroupResource must not be in
   `OverlappingBuiltInResources()` (`crd_finalizer.go`); duplicate names across the manifest set
   are errors; fields that the current feature-gate settings would silently strip on write
   (`spec.versions[*].selectableFields` without `CustomResourceFieldSelectors`,
   `x-kubernetes-validations[*].optionalOldSelf` without `CRDValidationRatcheting`) are errors —
   fail closed on operator input rather than replicating `dropDisabledFields`'s
   old-object-conditional behavior inside the equality comparison.
3. Status synthesis, reproducing what the registry strategy's `PrepareForCreate` does
   (`apiextensions-apiserver/pkg/registry/customresourcedefinition/strategy.go`): clear status,
   set `storedVersions` to the declared storage version. (Required before validation:
   `ValidateCustomResourceDefinition` validates status and rejects an empty `storedVersions`.)
4. The **unmodified** `validation.ValidateCustomResourceDefinition`. This brings along, for
   free and identically to the API path: structural schema requirements, CEL compilation and
   cost budgets, `metadata.name == plural + "." + group`, and — deliberately —
   `validateAPIApproval` for protected `*.k8s.io` / `*.kubernetes.io` groups. A static CRD in a
   protected group needs the `api-approved.kubernetes.io` annotation (or the documented
   `unapproved...` prefix, called out here so nobody discovers it later and calls it a hole).
   No exemption: exempting static CRDs would create an invisible group-squatting laundering
   path, which is worse than the one-annotation cost.
5. **Canonicalization, once.** The loader computes each manifest's *canonical spec* — deep-copy,
   `apiextensionsv1.SetDefaults_CustomResourceDefinitionSpec`, then scheme-convert to the
   internal `apiextensions` type — and stores it in the loaded set. The reconciler and the
   protection plugin share this same value by pointer; there is exactly one comparison function,
   `apiequality.Semantic.DeepEqual` on canonical specs, so their notions of "agrees" cannot
   diverge. The internal version is the comparison version because `apiextensionsv1.JSON`
   retains raw bytes (`v1/marshal.go` `UnmarshalJSON`), making v1-level equality sensitive to
   YAML key ordering inside `default:`/`example:` values — a trap that would produce a
   single-node perpetual write loop on any manifest whose defaults are not alphabetized.
   Defaulting before comparison is equally load-bearing: almost no hand-written manifest carries
   `names.singular`, `names.listKind`, or `conversion`, all of which `SetDefaults` fills
   (`v1/defaults.go`), and the live object always carries them (precedent for
   default-then-compare: the APF ensurer's `flowSchemaSpecEqual`).

```
<<[UNRESOLVED canonical comparison version @jpbetz apiextensions reviewers ]>>
Internal-version equality after v1 defaulting (proposed: immune to the `JSON.Raw` ordering
trap, and free for the admission plugin, whose `a.GetObject()` is already the internal type)
versus defaulted-v1 equality (byte parity with the serving stack's own teardown test at
`customresource_handler.go` `updateCustomResourceDefinition`, which compares v1 specs). No
constructed case distinguishes them, but they are not the same function; if v1 is chosen, the
manifest must be round-tripped v1->internal->v1 to normalize `JSON.Raw`. Wants an explicit
ruling rather than an implicit choice; the v1beta1 storage round-trip regression test ships
either way (CRDs are stored as v1beta1, whose defaulting differs).
<<[/UNRESOLVED]>>
```

Per `CustomResourceManifestSet` document: exactly one must exist whenever the directory
contains any CRD or tombstone document (zero-with-other-documents, or more than one, is fatal);
`revision` is required, an integer `>= 0`; `supersedes` entries must be integers, non-negative,
without duplicates, at most 8 — entries that do not exceed `revision` are **vacuous: warned
about and ignored, never fatal**, because a leftover recovery entry must not brick an unrelated
later boot. An entirely empty directory is valid and loads zero static CRDs.

Per `CustomResourceDefinitionTombstone` document: `crdName` must be shaped `<plural>.<group>`,
must not match a compiled-in group or `OverlappingBuiltInResources()`, and must not also appear
as a live CustomResourceDefinition in the same set (fatal — one set cannot both declare and
retire a name); duplicate tombstones for one name are fatal; `action` must be `Demote` (strict
decoding makes any future value a load error on binaries that predate it — fail closed in
exactly the right direction). A tombstone naming a CRD that is neither manifest-known nor
static-labeled in the cluster is *inert with a startup warning* (the typo diagnostic): a
tombstone legitimately outlives the object it retired.

Set-level diagnostics: the loader detects the **silent-twin rename** — a new manifest CRD whose
`spec.group` matches an existing static CRD's group with overlapping
`kind`/`singular`/`shortNames` — and warns loudly. `spec.group` and `spec.names.plural` are
unconditionally immutable (`validation.go`, `validateCustomResourceDefinitionSpecUpdate`:
"these affect the resource name, which is always immutable") and they are the etcd key prefix,
so a "rename" in a manifest does not rename anything: it silently registers a *second* CRD and
strands 100% of the data under the old key. The legal path for a group/plural change is a new
CRD plus an external data migration.

**Two-tier load-error handling.** The KEP-5793 rule — any load error is fatal before the
listener opens — is preserved for *structural* errors but must be refined for *content*
validation, because `ValidateCustomResourceDefinition` (the CREATE validator) is strictly
stronger than `ValidateCustomResourceDefinitionUpdate` on several independent, old-object-
derived axes (`requireStructuralSchema`, `requirePrunedDefaults`, `requireAtomicSetType`,
`allowInvalidCABundle`, `allowTooManySelectableFields`, the CEL `StoredExpressions` vs
`NewExpressions` environments, per-expression cost suppression — all derived from `oldObj` in
`ValidateCustomResourceDefinitionUpdate`, all hardcoded strict on create). Consequently the
same manifest can be a fatal error on a fresh cluster and perfectly updatable on a cluster
whose object already exists — and CEL expressions compile against the binary's
`DefaultCompatibilityVersion()` (binary−1), so a rule using a library shipped in the current
release is uncompilable for exactly one release. A commit validated against cluster A must not
brick cluster B's boot while B would happily serve that content:

- **Tier 1 — fatal before the listener opens:** parse and strict-decode failures, wrong kinds,
  `status` present, `conversion: Webhook`, duplicate names, compiled-in-group claims,
  missing/invalid `CustomResourceManifestSet`, malformed `revision`/`supersedes`, and
  gate-stripped fields (step 2 above).
- **Tier 2 — content-validation errors** (non-structural schema, unpruned defaults, more than
  8 `selectableFields`, invalid conversion CA bundle, CEL constructs outside the binary−1
  environment, missing `api-approved.kubernetes.io`): fatal **only when no object of that name
  exists** — checked through the same direct-storage read path the pre-activation emptiness
  check already uses, so a fresh-cluster bootstrap still fails loudly and halts health-gated
  rollouts. When an object exists, the error defers to reconcile time, where
  `ValidateCustomResourceDefinitionUpdate` adjudicates and a refusal surfaces as
  `StaticSynced=False` — the object keeps serving, exactly as a terminal conflict does.

The corollary the admin documentation carries: an existing unapproved `*.k8s.io` CRD, a CRD
with nine `selectableFields`, or one with a stale CEL construct is *adoptable but not
creatable* — promotion succeeds where fresh-cluster bootstrap from the same directory fails,
and `kubectl apply --dry-run=server` against a canary of the target version is the sanctioned
pre-flight. CRD warnings (for example unrecognized `format:` values) are logged at load, since
a manifest load has no API client to receive them.

```
<<[UNRESOLVED two-tier load validation: PRR ruling required @liggitt @deads2k ]>>
Tier 2 softens the KEP-5793 fail-closed rule and makes boot outcome depend on etcd content (an
existence check per Tier-2-failing manifest, via direct storage read before the listener
opens). The alternative — uniformly fatal — means a valid, servable configuration can brick a
control-plane node because the create-time validator is stricter than the update-time one the
content will actually face. PRR must bless the boundary and the existence-check mechanism
explicitly; the KEP proposes the tiering above.
<<[/UNRESOLVED]>>
```

Any Tier-1 error in the initial load is a **fatal process-startup error**, before the listener
opens (KEP-5793 rule: fail closed on operator-owned input). Errors involving API-writable state
(a conflicting etcd object) are never fatal — see The Reconciler.

### Manifest Revisions and Convergence

Two invariants everything in the lifecycle model hangs on:

**Invariant A — serving state is the shared object.** Every member builds a custom resource's
serving stack from the CRD object read through the shared informer
(`customresource_handler.go` `getOrCreateServingInfoFor` reads `r.crdLister`), never from its
local manifest. A manifest change is therefore **fleet-atomic at first landing**: the instant
the first rolled member writes it, every member serves it, within informer latency. The
consequence for design: the HA ordering problem is *not* consensus — the fleet does not need to
agree before a change applies. It is an **exactly-once** problem: for each change, exactly one
member should write it, once, and nobody should write anything else. The revision is an
idempotence token, not a quorum.

**Invariant B — recency is authored, never inferred.** Mid-roll, "two members holding content
A, one holding content B" is byte-identical between a forward roll and a rollback; the
difference is intent, which exists only in git. No consensus-free scheme can infer direction
from local files plus the shared object, so the manifests must *carry* an ordering statement.
This kills every derived ordering structurally, not merely practically: file mtime is a
distribution artifact (kubelet stamps a fresh timestamped directory on every projection, and it
inherits full clock-skew dependence); a content hash or opaque generation string is equality
with no order, so every change is a tie; auto-increment (write recorded+1 on divergence) lets
two divergent members escalate forever, mechanizing the fight and destroying the
misconfiguration signal; an authored timestamp is a manually bumped integer with false
automaticity; a hash-chain parent pointer gives true causal order but cannot be hand-authored
without mandatory tooling and deadlocks on a raw `git reset` identically. The redundancy of an
explicit revision *next to* the content hash is the safety mechanism: "hash changed, revision
didn't" is detectable precisely because both are declared. Any alternative proposal must answer
where its ordering information comes from.

#### The Declaration and the Record

The `CustomResourceManifestSet` document (see Manifest File Format) declares one
`revision` (int64 `>= 0`, required, monotonically increasing, one bump point per deployed
change — set-level, not per-CRD: N bump points multiply forgetting, and set-level acts like
tombstones and file moves have no per-CRD home; per-CRD granularity is already carried by the
per-document content hash) and optionally `supersedes` (exact recorded revisions this
configuration deliberately replaces despite a lower number).

Every reconciler write stamps the **record** on the object, atomically with whatever else the
write changes: `apiextensions.kubernetes.io/static-revision: "<n>"` and (when applicable)
`apiextensions.kubernetes.io/static-superseded: "<v1>,<v2>"`. An absent or unparseable record
reads as revision −1 (logged once, repaired by the next legitimate write) — which is also why
adoption of pre-feature and demoted objects is always possible.

#### The Ordering Rule

One total comparator, shared by the reconciler, the behind guard, and (for its narrow uses) the
protection plugin:

```
rank(cfg, record):
  if record.superseded contains cfg.revision: LOSES   # the record explicitly disowns me
  if cfg.supersedes contains record.revision: WINS    # I explicitly disown the record
  cfg.revision >  record.revision: WINS
  cfg.revision <  record.revision: LOSES
  otherwise:                       TIE
```

A mutual disown resolves LOSES — both sides are misconfigurations of one fleet, and the
already-applied configuration wins so the loser freezes loudly instead of writing.

**Sticky supersession.** When a write at revision `R` stamps the record, the recorded
superseded set becomes `{v in (old record.superseded ∪ cfg.supersedes) | v > R}`, bounded (at
most 8, keeping the largest). Two consequences: a hijacked huge revision stays disowned across
subsequent ordinary commits *without* keeping the `supersedes` entry in git forever; and
values at or below the new revision drop out automatically, because ordinary comparison already
outranks them. The one rule the operator must follow, printed in the recovery log line: **the
next ordinary revision after a `supersedes` recovery must exceed every ordinary revision the
recovery superseded** (after `revision: 46, supersedes: [47]`, the next bump is 48, not 47 —
reusing 47 would tie against any member still running the retired 47 tree, converting its
silent deferral into a loud freeze; contained, but noisy).

#### The Behind Guard

```
behind(cfg) :=
  the CRD informer has not completed its initial sync            # fail closed at cold start
  OR there exists an object O such that
       (O carries the static=true label OR O's name is in cfg's assertion set)
       AND rank(cfg, record(O)) == LOSES
```

A behind member performs **no writes of any kind** — no creates, no adoptions, no spec updates,
no record confirms, no demotions, no condition writes. Serving and admission protection are
completely unaffected (both read the object). It logs once per observed revision transition,
naming the blocking object and the literal minimum revision that would win, and exposes
`apiserver_static_crd_controller_config_behind{group, resource}` for the blocking object plus
its own `..._manifest_revision` gauge. Being behind never affects readiness: a readiness
predicate keyed on convergence would mark n−1 of n members unready the instant the first member
rolls — the inverse of a safe rolling update.

One global guard rather than per-object deferral, deliberately: the one transition that is
inherently unguardable per-object is CREATE of an *absent* object (there is no record to
consult), and the guard closes it with zero new machinery — a member returning from months
offline cannot recreate a retired CRD as long as any live labeled object records a newer
revision. The scope of the scan is equally deliberate, on both edges:

- **Only labeled or locally-asserted objects count.** An unlabeled, unclaimed object carrying a
  planted record must not be able to wedge the fleet. Plants are closed off under the gate
  (CREATE carrying reserved keys is denied on **all** CRDs; UPDATE to reserved keys likewise —
  see Write Protection), and residue from gate-off windows is cleanable: reserved-key *removal*
  is admission-permitted on objects that carry no static label and are claimed by no local
  manifest (the same allowance as break-glass demotion).
- **Unasserted demoted objects do not count.** A demoted object (label absent) that this
  configuration says nothing about retains its record purely as the per-object fence against
  stale re-promotion; it does not contribute to the global floor, so a retired type's residue
  cannot freeze a future configuration whose numbering was legitimately restarted. A demoted
  object this configuration *does* assert (a manifest or tombstone for the name) counts: an
  assertion outranked by the record proves the whole configuration predates the fleet's — the
  ancient-member case the guard exists for. (A *still-managed* labeled object's record wedges
  a down-numbered configuration regardless — correctly: per Invariant B, a config claiming
  revision 1 against an object recording 100 is indistinguishable from an ancient stale
  member, and the remedy — bump past it, or supersede — is printed.)

The honest cost: while a roll is in progress, un-rolled members reconcile nothing at all.
Acceptable because with the gate on, reconciliation has almost nothing to do in steady state
(deletes are denied, reserved keys are server-restored), and serving, protection, and readiness
never depend on it. One transient consequence worth naming: a reload-safe revision bump that
lands while a *restart-only* rollout (a flag-named CRD add) is still incomplete makes the
already-restarted member behind, pausing its pending flag-bound create until that member
receives the newer configuration — self-healing when the roll completes, visible via
`config_behind` plus an asserted-but-absent warning, and avoided entirely by the ordering rule
in Notes/Constraints item 9.

#### Rollback

Three flows, all one git commit, none an API write:

1. **Primary: revert the content and raise the revision.** Going backwards is going forwards
   with old content. This is Kubernetes' own model: `kubectl rollout undo` re-annotates the old
   ReplicaSet with `MaxRevision+1` (`deployment.kubernetes.io/revision` "records its rollout
   sequence" — `pkg/controller/deployment/util/deployment_util.go`), and OpenShift's static-pod
   `currentRevision` carries a "must only increase" validation rule. One spec write restores
   the old schema fleet-wide.
2. **The trap, named:** a bare `git revert` (or any tooling that restores a previous
   configuration wholesale — `clusterctl alpha rollout undo`, a `git reset`-and-rerender
   pipeline, a health-gate auto-rollback) restores the *old revision number too*. The whole
   fleet then ranks LOSES against its own records and goes inert, serving the content the
   admin meant to retire. **Nothing flaps and nothing is silently adopted**: every member
   reports `config_behind`, and the log names both exits — raise `revision` past the record, or
   ship `supersedes`. This is a wedge of *convergence*, never of serving, and it names its own
   key. Deployments whose rollback tooling restores files wholesale should derive `revision`
   from a forward-only source (a CI sequence number), mirroring the OpenShift rule above.
3. **`supersedes` recovery**, for the raw-reset tree and for hijack: `supersedes: [47]` at
   revision 46 declares "this configuration deliberately replaces recorded revision 47". The
   write records `static-revision: 46, static-superseded: 47`, so members still running the
   un-reverted 47 tree read a record that disowns them and go inert — recovery sticks
   fleet-wide from the first write, and (per sticky supersession) survives the entry's removal
   from git in a later commit. Exact values only: a `supersedesThrough` threshold was rejected
   because a lazy MaxInt64 would quietly re-enable permanent last-writer-wins between divergent
   configurations.

**The forgotten bump** — content changed, revision untouched — is the scheme's main UX cost,
and it fails safe: the first rolled member ranks TIE (equal revision, different content) and
**freezes that one CRD**: no write, the previous spec keeps serving, and the member emits
`frozen{reason="RevisionTie"}`, a guarded `StaticSynced=False` condition, and a log naming both
content hashes and the one-line remedy ("content for X differs from the content recorded at
revision 47; if intended, raise revision above 47"). Un-rolled members stay quiet — their
content matches the record. The fix hot-reloads and converges with exactly one write. The rule
is uniform, including on single-member clusters where no fight was possible: a self-overwrite
exemption ("write if the record matches what I last wrote") was rejected as
process-history-dependent (reload would converge but restart would freeze), violating the
pure-function-of-current-state philosophy, and because it reopens the HA fight through
whichever member happened to write last. There is deliberately **no hash tie-break** at TIE: a
tie-break would resolve a genuine misconfiguration by coin flip, flip winners on cosmetic
edits, and silence the one signal that exists only because revision and content are declared
independently.

**The fork, honestly:** if two branches both raise the revision to 48 with *different* content,
the freeze fires only once one of them is recorded — and which one is recorded is decided by
whichever differently-configured member reconciles first, after which (per Invariant A) the
whole fleet serves the race winner while the losers freeze loudly with both content hashes in
their diagnostics (which members hold which tree is readable off the per-member
`last_config_info` hash gauges). The design cannot do better without consensus —
two authors picking the same number *is* the misconfiguration — so the KEP contains rather than
prevents it: a newly observed strictly-newer revision is actuated only after the directory
content hash is observed unchanged on a subsequent scan (a short debounce, which also absorbs
partially-rsynced directories), the frozen side's diagnostics make the fork identifiable, and
the recommended CI lint is strengthened to "revision values are unique across branches" (derive
from a CI counter). An integration test asserts this actual outcome, not a prettier one.

#### What Is Revertible

Classified rather than promised away:

- **Revertible by revert-plus-bump:** schema, CEL rules, printer columns, selectable fields,
  subresources, `served` flags, shortNames, deprecation markers, tombstones (reverting one
  re-promotes), promotions (a tombstone demotes).
- **One-way until migration:** a storage-version move. The registry strategy appends the new
  storage version to `status.storedVersions` on the same update
  (`customresourcedefinition/strategy.go` `PrepareForUpdate`), and the main-resource path can
  never shrink it (`newCRD.Status = oldCRD.Status`), so a revert restores serving but leaves
  `storedVersions: [v1, v2]` — neither version can then be dropped without a storage version
  migration. The reconciler logs and counts (`reconcile_total{action="storage-version-move"}`)
  at the moment it actuates one, so the one-way step is visible when taken, not discovered at
  revert time.
- **Permanently non-declarative:** `spec.scope` and `spec.names.kind` (immutable once
  `Established`, and `Established` is sticky — the naming controller copies the existing
  condition forward and nothing ever sets it back to False), and `spec.group` /
  `spec.names.plural` (unconditionally immutable; they are the etcd key prefix). The only path
  is a new CRD plus an external data migration, and under this KEP the retiring side is:
  tombstone-demote, migrate, then ordinary API delete by the data's owner.

#### Convergence Observability

Two distinct questions, answered separately:

**"Is revision N in effect?"** One member suffices (Invariant A). Answer from the API:
`kubectl get crd -o custom-columns=NAME:.metadata.name,REV:".metadata.annotations.apiextensions\.kubernetes\.io/static-revision"`
shows `>= N` on every claimed CRD. The converged-confirm write (see The Reconciler) is what
keeps this truthful for *unchanged* CRDs across a bump — annotation-only, hence free — and it
is also what publishes the fleet floor the behind guard reads.

**"Has every member loaded revision N?"** Required before starting a storage-flag roll, before
the handoff gate in Transitions, and (at beta) before garbage-collecting a tombstone. At alpha
the answer is metrics-only, with an honest caveat: `min(apiserver_static_crd_controller_manifest_revision) >= N`
and `max(...config_behind) == 0` and `..._last_config_info{hash}` agreement across members —
which requires scraping every member *and* knowing how many members there should be; a member
that is down and stale reports nothing and silently drops out of the `min`. At beta, each
member stamps its manifest-set revision as an annotation on its own apiserver identity Lease
via the existing `ProcessLeaseFunc` hook (the peer-proxy address already rides this mechanism —
`pkg/controlplane/apiserver/server.go` `labelAPIServerHeartbeatFunc`), giving an API-visible
roster that self-GCs by lease expiry (no reaper to build; machine-replacement rollouts leave no
ghost members). Rules: a lease not renewed within a staleness cutoff is ignored
(`IdentityLeaseDurationSeconds` and the GC period are both 3600s, so lingering leases are
real); a lease without the annotation counts as **unknown, never converged**; `APIServerIdentity`
disabled degrades to the metrics answer with a logged warning. A per-CRD status list keyed by
apiServerID (the StorageVersion shape) was rejected: entries-times-members write amplification
plus a mandatory lease-based reaper, for an answer the annotation and gauges already provide.

```
<<[UNRESOLVED identity-lease revision stamp (beta) @liggitt @deads2k sig-api-machinery ]>>
Publishing each member's manifest-set revision on its apiserver identity Lease is
Design-Rule-clean by value class (an ordinal), but it makes per-member configuration state and
fleet size readable by any principal with `get leases -n kube-system`, and it couples the beta
CREATE floor and tombstone-GC signal to the `APIServerIdentity` gate. Proposed: accept, with
the staleness cutoff and unknown-never-converged rules above, degrading to metrics-only when
the gate is off. Needs PRR and API review at beta; alpha ships metrics-only.
<<[/UNRESOLVED]>>
```

The recommended CI lint for the git CUJ, stated non-normatively: if files under
`staticManifestsDir` differ from the parent commit, `revision` must differ, and revision values
must be unique across branches. This makes the only new admin obligation near-automatic.

### The Reconciler

A post-start hook (`apiextensions/static-crd-reconciler`) runs a workqueue keyed by CRD name,
fed by the loaded assertion set (manifests and tombstones) and by the existing shared CRD
informer (filtered to asserted names and objects labeled
`apiextensions.kubernetes.io/static=true`). No item is processed — and in particular the
CREATE branch is never taken — before the informer's initial sync completes; the behind guard
fails closed until then. Per item, with `rank` and `behind` as defined above:

- **Behind:** return before any per-object work. No writes of any kind.
- **Unasserted name, labeled object, this configuration outranks its record:** the one
  diagnostic case — a CRD file was deleted without a tombstone. The member writes a guarded
  `StaticSynced=False, reason: ManifestRemovedWithoutTombstone` condition (idempotent across
  identically-configured members) naming the tombstone as the supported removal expression. No
  marker is ever touched: protection is label-keyed and retained; absence demotes nothing.
- **Unasserted otherwise:** untouched.
- **Object absent, manifest asserted:** CREATE via the loopback client — spec, markers,
  witness, and record in the same create. `AlreadyExists` is a lost race: requeue and
  re-arbitrate against the winner (never blind-retry). If this process previously observed the
  name in Terminating state, refuse and alarm rather than recreate through a cascade — stated
  plainly: this refusal is process-local memory and cannot survive a restart; the *durable*
  resurrection fences are the behind guard and the retained tombstone (see Declarative
  Removal).
- **Object present with a non-zero `deletionTimestamp`:** adoption is refused (terminal
  conflict; `frozen{reason="terminating"}`). Adopting a Terminating object would let the
  in-flight finalizer cascade resume — and complete — against a type the operator just declared
  protected, deleting all its data and leaving the reconciler to recreate an empty shell.
- **Converged** (manifest asserted: object spec equals the canonical manifest spec and markers
  are correct; tombstone asserted: label absent): if `rank == WINS`, advance the record with an
  annotation-only **confirm** write (zero storage churn — `customresource_handler.go` tears
  down only on spec/acceptedNames changes); set `StaticSynced=True` in the manifest case. The
  confirm is what keeps `kubectl get crd` truthful about the newest confirming configuration
  and publishes the floor un-rolled members defer to; at most one lands fleet-wide because
  members re-check before writing.
- **`rank == WINS`, divergent:** actuate — the only spec-writing path.
  - *Manifest asserted (adopt / update / re-promote):* one full-object **Update** (PUT)
    replacing `spec` wholesale, setting the label, content hash, record, and witness, and — on
    first adoption — **removing any pre-existing `metadata.ownerReferences`** (an
    OLM-CSV-owned CRD would otherwise wedge the garbage collector on a delete that can never
    complete; promotion severs third-party ownership, and the pre-flight in Transitions says
    so). `ValidateCustomResourceDefinitionUpdate` — the `storedVersions` drop guard,
    scope/kind/group/plural immutability — adjudicates. A rejected update is a *terminal
    conflict*: the object keeps serving its current valid spec, `StaticSynced=False` with a
    classified reason (`StoredVersionsBlocked`, `ImmutableField`, ...), the record does **not**
    advance, and — because the label rides the same rejected write — a never-adopted object
    stays an ordinary, *editable* CRD, deliberately: protecting an unadoptable object would
    freeze the only path by which the operator can resolve the conflict. Never mutate
    data-bearing identity to force convergence; kcp-dev/kcp#731 is the worked example of
    incremental reconciliation producing an unstartable server.
  - *Tombstone asserted (demote):* one metadata-only Update removing the label and content
    hash, retaining the witness, stamping the record — followed by a best-effort status write
    removing the `StaticSynced` condition (the object is no longer managed; a future owner
    inherits clean conditions).
- **`rank == TIE`:** split on the tamper witness. The recorded content hash names the spec the
  last reconciler write left; compare it to the hash of the object's *current* canonical spec:
  - Record hash ≠ object's spec hash: the object was **modified out of band** (a gate-off
    window edit, a pre-protection write). This is not a tie; it is tamper. If this member's
    manifest hash equals the recorded hash — it holds exactly the content the record names —
    it repairs: write the manifest spec (`reconcile_total{action="repair"}`, rate-limit
    backstop applies). Otherwise freeze and alarm. Without this rule, an out-of-band edit
    would be *enforced* by the equal-revision freeze and by the object-anchored admission
    rule — a durable violation of the invariant that no API write changes a manifest CRD's
    spec.
  - Record hash = object's spec hash, local content differs: the genuine
    forgotten-bump/forked-commit **freeze** (`frozen{reason="RevisionTie"}`): no spec write,
    the guarded condition write described under Rollback, a log naming both hashes and the
    remedy. Never a hash tie-break.

**Write discipline** (each item load-bearing, with in-tree precedent): plain **Update (PUT)
with a resourceVersion CAS, never server-side Apply**. SMD's `Updater.Update` forces and
replaces wholesale, which is exactly right for a single source of truth; `Apply`'s
prune-and-`addBackOwnedItems` semantics let a co-owning GitOps manager pin fields the manifest
removed — an unbounded, error-free write loop — and `spec.versions` is `listType=atomic` so
apply ownership is all-or-nothing anyway; every in-tree bootstrap reconciler (APF ensurer,
bootstrap RBAC, automanaged APIServices) uses Update. Writes carry
`FieldManager: apiserver-static-crd` — that string is user-visible in SSA conflict errors and
managedFields, chosen to be self-describing and Design-Rule-clean. On 409/404: yield and
requeue — the informer redelivers the winner and arbitration re-runs as a pure function of
current state (at most 2 in-pass attempts, then workqueue backoff; RBAC/autoregister
precedent). Comparison is always between canonical specs (see Loading), never raw bytes: every
real spec delta costs a full per-member storage teardown bounded at twice `--request-timeout`
plus a whole-cluster OpenAPI re-merge, so benign YAML formatting drift must never trigger one.
A strictly-newer revision observed via *reload* is actuated only after the debounce described
under Rollback; startup actuates immediately.

**Condition discipline:** only a member that is not behind writes `StaticSynced` or the record,
so the condition has a single coherent writer per revision epoch — the configuration that
actuated it. Behind members report exclusively via metrics and logs. The two diagnostic
exceptions (`RevisionTie`, `ManifestRemovedWithoutTombstone`) are guarded writes — skipped when
the condition already carries the same reason and message — so identically-configured members
converge on one write; under a genuine fork the alternating hashes in that message *are* the
fork signal, bounded by the per-object rate limit.

The reconciler **never deletes objects, and removes markers only in execution of an explicit,
operator-authored tombstone** — never inferred from absence. With arbitration, a correctly
operated fleet performs exactly one spec write per revision per changed CRD, plus at most one
annotation-only confirm per unchanged CRD; the per-object rate limit (floor measured in
minutes) is therefore demoted from primary containment to a **backstop**, retained solely for a
misconfigured member still running a retired or hijacked configuration. Because this
arbitration ships in the first alpha, no force-update-on-divergence binary ever ships, and
there is no mixed-arbitration skew window to manage.

If etcd is unreachable at startup, the reconciler retries with backoff; the server stays up,
`/readyz` reports the `static-crds` check unready, and no crash-loop occurs (bootstrap-RBAC
precedent, `pkg/registry/rbac/rest/storage_rbac.go`).

### Declarative Removal: Tombstones

Every system that deletes declaratively and safely shares three ingredients: an explicit
membership marker on live state (kubectl applyset's id label, Flux's `status.inventory`, Argo
CD's tracking annotation, APF's `autoupdate-spec`), comparison against a complete versioned
snapshot of desired state, and removal as a *positive opt-in assertion* — while legacy
`kubectl apply --prune`, which infers removal from absence, has spent a decade in alpha behind
layered guardrails. This design has the first two (the static label; the revisioned manifest
set); the tombstone supplies the third. Its syntax and placement are in Manifest File Format;
its semantics:

**Intent set: `Demote`, and only `Demote`, at alpha.** Executing a tombstone strips the static
label and content-hash annotation, **retains the witness** (it is the data-location record, and
the binding guard keeps honoring it, so a demoted CRD bound to an override keeps serving from
the right backend), and stamps the record. That is the entirety of what a tombstone does, and
it fully replaces the hand-written marker-stripping API write the previous draft required.
`Delete` via manifest is **rejected outright, not deferred**: Goal 7 says no operation in this
feature deletes custom resource data; the finalizer cascade (`crd_finalizer.go`
`deleteInstances`: per-namespace DeleteCollection, no cascade control) is the footgun Story 2
exists to kill; and a KEP that denies DELETE even to cluster-admin cannot coherently let a file
drop trigger the same cascade on a staggered node-by-node schedule with no confirmation and no
acting principal in the audit log. The supported destructive composition is
**tombstone-demote, then ordinary `kubectl delete crd`** — RBAC-governed, audited, performed by
the data's owner on what is by then an ordinary object. (A `DeleteIfEmpty` action — reconciler
deletes only after proving zero stored CRs, with creates denied while checking — is data-safe
in principle and noted as a beta consideration only.) A separate demote-but-keep-delete-denial
level was rejected: the transition contract requires demotion to leave the API CRD fully in
charge, *including deletable*, and because protection is label-keyed, demotion flips
fleet-atomically at the single demote write — there is no partial-fleet window for a softer
level to protect.

**Execution and cost.** The first non-behind member whose configuration outranks the record
performs one metadata-only CAS write (plus the best-effort condition cleanup). Verified free at
every layer: no generation bump, no storage teardown, no OpenAPI ETag churn, no watch
interruption, no `Established` transition, same UID. Because protection keys on the label, API
writability flips for the whole fleet at that write — **a tombstone is effective cluster-wide
the moment the first member applies it, and must not be treated as canary-able**; the
pre-demotion gates in Transitions run *before* the tombstone ships, not during its rollout.

**The fence is the record, not the git file.** A member that never receives the tombstone
commit — stuck at an older revision with the live CRD manifest — reads label-absent plus a
record that outranks it, and defers indefinitely: deletion-as-state rather than
deletion-as-event, the same move `deletionTimestamp` makes for two-phase delete. The git
tombstone's persistence is thereby decoupled from the fencing function for every member that
can read the object. Symmetrically, a member still holding the tombstone after the fleet has
re-promoted at a higher revision is behind and defers — the tombstone cannot be replayed out
of order. Reversal is not a special case: reverting the tombstone commit restores the CRD
manifest at a newer revision, and the ordinary adoption path re-promotes.

**Tombstones are permanent manifest content at alpha.** The natural GC rule — "removable once
every member has loaded a revision at or past the tombstone's" — has a precondition that alpha
cannot evaluate (a metrics `min()` without an authoritative member census: a powered-off stale
member reports nothing and silently drops out), and it is genuinely unsafe in this KEP's own
flagship shape: a cluster whose *only* static CRD is demoted and later API-deleted retains
zero records anywhere, so a member that was offline through the whole sequence finds nothing to
defer to and recreates the type — empty, protected, and (if the GroupResource is still
flag-named) rebound. The residue of permanence is roughly four lines of YAML per retired type.
At beta, the identity-Lease revision floor supplies the census, and GC becomes: removable one
rollout after every live member's lease reports a revision at or past the tombstone's, and only
while some live labeled record remains as a floor backstop. The residual even then — demote,
API-delete, *every* record gone, ancient member returns — is documented honestly: recreation is
loud (config-hash disagreement, revision gauges, `registered` counts), destroys no data (the
cascade already ran under an audited principal), and is repaired by rolling the member; the
in-process Terminating refusal narrows but cannot close it, because it is process-local.

**Flag-named GroupResources stay restart-only in both directions.** A reload whose delta
includes a tombstone for a GR currently named in `--etcd-servers-overrides` or
`--watch-cache-sizes` is rejected atomically (same poison-pill treatment as flag-named adds),
because the startup-frozen override map would otherwise carry a live entry for a GR that is no
longer static — a dormant entry a later dynamic CRD in the same process lifetime could inherit.
The documented decommissioning order for a bound CRD: drain the data back, clear the witness,
drop the flag entry, **then** tombstone and restart. Beta's stale-flag-entry fatality enforces
this order mechanically.

### Write Protection

Protection is a compiled-in **admission plugin** (`StaticCustomResourceDefinition` protection
plugin — validating, plus the narrow mutating phase described below) on
`customresourcedefinitions` and `customresourcedefinitions/status`, active when
`StaticCustomResourceDefinitions` is enabled. Admission — not a `(*REST).Delete`
wrapper and not strategy-level immutability — because it is the only complete enforcement
point: `Store.DeleteCollection` calls the *embedded* Store's `Delete`, bypassing any outer REST
wrapper, while both DELETE and DELETECOLLECTION run admission per object
(`staging/src/k8s.io/apiserver/pkg/endpoints/handlers/delete.go`). So
`kubectl delete crd --all` is covered. The apiextensions server already receives the shared
kube-apiserver admission chain.

The plugin has a narrow **mutating phase** (reserved-key restoration, below) ahead of its
validating phase; both run on every CRD write, and both run under server dry-run
(`DryRunnableStorage` executes the full update closure and skips only the etcd write), so
`kubectl diff` and `--dry-run=server` report exactly the live decisions. One ordering fact is
load-bearing for the whole design and is stated here so review does not have to rediscover it:
for server-side apply, the structured merge (and its conflict detection) runs while computing
the updated object, strictly *before* validating admission — the plugin receives the merged,
defaulted, internal-typed object — so `--force-conflicts` can strip SSA field ownership but can
never bypass the denial. Admission and SSA conflicts are two independent gates, and admission
is the later one.

**What is protected, keyed on what — deliberately split:**

- **UPDATE and DELETE protection keys on the object alone**: an object is protected iff it
  carries the `apiextensions.kubernetes.io/static=true` label. The label is on the shared
  object every member reads, so protection is *fleet-uniform at every instant* — it turns on
  fleet-atomically at adoption's write and off fleet-atomically at demotion's write,
  regardless of which members have loaded which configuration. (A member-local clause here
  would make DELETE — the irreversible verb — answer differently behind a load balancer during
  every promotion and demotion window.)
- **CREATE name reservation keys on this member's loaded manifest set**: creating a CRD whose
  name a local manifest claims, when no object exists, is denied. Member-local is acceptable
  here because the failure mode is a denied create, not a lost protection, and the window
  closes at the reconciler's own create.
- Consequence, documented in Risks: **protection begins at adoption, not at manifest claim.** A
  claimed-but-not-yet-adopted dynamic CRD remains ordinary and editable — deliberately, since
  a terminally divergent object's only repair path is being edited toward adoptability — and a
  first-time promotion is unprotected until the reconciler's first write lands.

Rules for protected (labeled) objects:

| Request | Rule |
|---|---|
| UPDATE changing `spec` | **Allowed iff the resulting canonical spec equals the object's *current* canonical spec** — i.e. the write is a spec no-op; denied otherwise. Anchoring on the object (not on "this member's manifest") makes the decision fleet-uniform mid-roll and closes both member-local failure shapes: a no-op apply spuriously denied by a not-yet-rolled member, and *old* content written back through a stale member. It is also the direct enforcement of the invariant: no API write changes a manifest CRD's spec, and identical content always succeeds because it changes nothing. (Identical writes still *reach* admission — the store runs `updateValidation` inside `GuaranteedUpdate` — but never reach etcd: the etcd3 store short-circuits byte-identical serializations, so a GitOps loop re-applying every 60s costs a decode and a compare, with no resourceVersion bump and no watch traffic.) |
| UPDATE changing `metadata.finalizers` or `metadata.ownerReferences` | Denied (an ownerReference would let GC wedge on a delete it can never complete) |
| UPDATE changing `api-approved.kubernetes.io` | Denied; the manifest's value is authoritative and the reconciler restores divergence. Without this, any `patch customresourcedefinitions` holder could downgrade a manifest CRD's approval state to `unapproved...` durably — the one non-reserved annotation with semantic weight, so it is managed explicitly. |
| Other label/annotation changes | Allowed — operators legitimately label CRDs |
| Status subresource, status fields | Allowed in general (the naming/establishing/apiapproval/nonstructuralschema controllers and the storage version migrator's `ApplyStatus` must keep working), with two **shape rules** closing the fields that steer serving and change authority: a write changing `status.acceptedNames` is allowed only if every changed field's new value equals the corresponding `spec.names` value or the prior accepted value (the naming controller only ever writes requested-or-retained names, so it passes; a squatter steering serving identity away from spec does not); a write changing `status.storedVersions` is allowed only if the result is unchanged or a subset of the prior value with every remaining entry present in `spec.versions` (the SVM trim shape passes; arbitrary clobbering — which would permanently veto every future spec update via the drop guard, with no reconciler repair path since the main-resource strategy overwrites status — does not). |
| Status subresource, metadata deltas | Denied, with exactly one carve-out: a delta whose *only* change is the **removal of `apiextensions.k8s.io/CustomResourceCleanupFinalizer`**. The CRD status strategy currently passes metadata mutations through (`statusStrategy.PrepareForUpdate` calls `metav1.ResetObjectMetaForStatus(&newObj.ObjectMeta, &newObj.ObjectMeta)` — a deliberate no-op pending [kubernetes/kubernetes#137681](https://github.com/kubernetes/kubernetes/issues/137681)), and `ValidateUpdateCustomResourceDefinitionStatus` does not constrain labels/annotations/finalizers — so without the denial, `update customresourcedefinitions/status` permission would strip the protection label or plant a witness. The carve-out exists because the CRD finalizer removes its own finalizer *via UpdateStatus* (`crd_finalizer.go`: `CRDRemoveFinalizer` then `UpdateStatus`) — a blanket denial would deadlock any labeled CRD that entered Terminating before protection covered it (deleted during a gate-off window, then the gate enabled): the cascade would have already destroyed data, and the object would sit Terminating forever, unadoptable and undeletable. |
| DELETE / DELETECOLLECTION | Denied unconditionally, including for `system:apiserver`. The message names the prevented data loss and the supported path: tombstone-demote (an ordinary manifest commit), then delete. |

**Reserved keys are server-owned, on all CRDs.** The reserved keys are
`apiextensions.kubernetes.io/static*` (label and annotations: the protection label, content
hash, revision record, superseded list) under the registration gate, and
`storage.apiextensions.k8s.io/*` (the witness) under the storage gate. Three rules, replacing
the previous deny-list:

1. **Restore, don't punish, omission.** The mutating phase copies each reserved key's value
   from the old object into any incoming write that *omits* it. This is what makes
   "identical content succeeds" true for every write shape: `kubectl replace`,
   controller-runtime envtest's `CreateCRDs` (a full PUT of the file's object),
   Argo CD `Replace=true`, Flux helm-controller `crds: CreateReplace`, and OLM's CRD update
   path all send objects without the markers — and those PUT-shaped tools are exactly what
   users of oversized CRDs (this KEP's own flagship example) are forced onto by the
   256KiB `last-applied-configuration` annotation limit. Restoration also covers *removal*
   (which a PUT does implicitly) and SSA pruning via either subresource.
2. **Deny divergent values.** A write carrying a reserved key with a *different* value than the
   old object is denied — deliberate tampering, as opposed to omission. Exception, with
   precedence over restoration: a write setting the witness to (or removing it at) *this
   member's locally computed binding value* is honored, not restored away — that is the
   documented acknowledgement/unlock path, and the mutating phase must recognize it first or
   the only remediation for a stale witness would be silently discarded while the resource
   503s. A forged unlock cannot poison other members, because each member compares against its
   own computed value. The restore also never *invents* keys absent from both old and new
   objects.
3. **CREATE carrying any reserved key is denied, on all CRDs**, whenever the respective gate is
   enabled — there is nothing to restore from, and this closes record/witness planting at
   creation (a planted `static-revision` would otherwise be a one-write freeze of the
   fleet's reconcilers; a planted witness would 503 the resource).

Cleanup allowance, mirroring break-glass demotion: reserved-key **removal** is permitted on an
object that carries no static label and is claimed by no local manifest — so residue planted
during gate-off windows, or left on a long-demoted object, is cleanable through the API without
disabling the gate fleet-wide. Residual risk while gates are off is unchanged and documented in
the un-gated-surface UNRESOLVED.

**Loopback bypass, narrowed:** the reconciler's own writes are recognized not by user
(`system:apiserver` alone would trust any client-CA-signed certificate with that CN) but by
user *plus shape*, and every shape is additionally tied to this member's loaded configuration
so that a forged principal can only write what the local configuration already says:

1. Actuation: spec equals the local canonical manifest spec, markers correct, record equal to
   the local revision (and superseded set).
2. Confirm: annotation-only, advancing the record to exactly the local revision on an
   otherwise-converged object.
3. Demotion: removes the label and content hash, stamps the record, changes nothing else —
   honored **only when this member's loaded set contains a `Demote` tombstone for that name
   and the local configuration outranks the object's record**. (Without the tie to a local
   tombstone, the demote shape would be a user-forgeable unlock of DELETE.)
4. Break-glass manual demotion: the marker-strip write is admission-permitted when no local
   manifest claims the name — retained as an undocumented disaster-recovery path (the
   requirement is that no API write be *required*, not that one be forbidden), and it emits a
   distinct audit annotation so fleets can detect out-of-band demotions. Revisit its fate at
   GA once tombstones are proven.

DELETE is never exempted, for any principal or shape.

#### Denial Messages

Client-visible denials carry a stable, greppable sentinel (`StaticCustomResourceDefinition:`),
name no file or path (the file-to-CRD mapping stays in the API server log, per the Design
Rule — this corrects an earlier draft's self-contradiction), state what happened, what *is*
allowed, and who to ask, because the receiving population has no control-plane access and has
often never heard of this feature. The canonical texts, pinned by integration tests:

- Divergent spec update:
  `customresourcedefinitions.apiextensions.k8s.io "workflows.argoproj.io" is forbidden:
  StaticCustomResourceDefinition: this CustomResourceDefinition is defined by this cluster's
  API server configuration and its spec cannot be changed through the API. Writes that leave
  the spec unchanged are accepted; this request would change spec.versions[1].schema. To
  change or remove this definition, ask a cluster administrator to update the API server's
  static CustomResourceDefinition configuration. See https://k8s.io/docs/.../static-crds`
- DELETE:
  `... is forbidden: StaticCustomResourceDefinition: this CustomResourceDefinition is defined
  by this cluster's API server configuration and cannot be deleted through the API. Deleting
  it would delete every Workflow object in the cluster. Ask a cluster administrator to remove
  it from the API server's static CustomResourceDefinition configuration first. See <docs>`
- CREATE of a claimed name:
  `... is forbidden: StaticCustomResourceDefinition: this name is reserved by this cluster's
  API server configuration. The definition already exists and is being served; your installer
  does not need to create it. See <docs>`
- Reserved-key divergence:
  `... is forbidden: StaticCustomResourceDefinition: the annotation
  "storage.apiextensions.k8s.io/etcd-servers" is managed by the API server and cannot be
  changed through the API.` (Omission is silently repaired by the mutating restore; only
  divergent values reach this denial.)

One documented two-step for SSA users: a divergent `kubectl apply --server-side` first
receives a **409 Conflict naming the field manager `apiserver-static-crd`** — raised during
merge computation, before admission runs — and reaches the 403 above only with
`--force-conflicts` (at which point nothing is persisted, so no ownership is actually lost).
Argo CD's `ServerSideApply=true` forces by default and lands directly on the 403. The
documentation also carries a per-tool opt-out table for API-side owners who need to stop
managing a promoted CRD: Helm — ship the CRD in `crds/` (never upgraded, skipped when present,
never deleted on uninstall) or `--skip-crds`; Argo CD —
`argocd.argoproj.io/sync-options: Prune=false` (also the *pre-promotion* requirement that keeps
Application deletion from wedging on the resources finalizer) or `RespectIgnoreDifferences`
with `ignoreDifferences` on spec; Flux — `kustomize.toolkit.fluxcd.io/reconcile: disabled`,
`ssa: IfNotPresent`, or helm-controller `crds: Skip`.

```
<<[UNRESOLVED unconditional DELETE denial @liggitt @deads2k ]>>
Denying DELETE on static CRDs even to cluster-admin is the most opinionated rule in this KEP: a
resource that has always been deletable stops being deletable, and `kubectl delete crd` fails
for the highest-privileged principal. Alternatives considered: warning plus a required
confirmation annotation; or denial only while a non-default storage witness is present. The KEP
proposes unconditional denial because it matches the trust argument (the principal who
provisioned the etcd backend is the one who may destroy its contents, and that principal edits
files on the control plane, not API objects) and because the deletion cascade
(`crd_finalizer.go` `deleteInstances`: per-namespace DeleteCollection, no cascade control) is
the co-benefit Story 2 exists to kill categorically. The tombstone flow lowers the stakes of
this ruling: releasing a CRD for deletion is now a one-commit declarative act (demote, then the
ordinary delete verb works), so unconditional denial no longer implies a restart-and-hand-edit
ordeal — but the ruling itself is unchanged and still needed.
<<[/UNRESOLVED]>>
```

### Storage Configuration

#### Etcd Server Overrides for Custom Resources

The single chokepoint through which all custom resource storage is built is
`NewCRDRESTOptionsGetter` (`apiextensions-apiserver/pkg/cmd/server/options/options.go`), which
today wraps one shared `StorageConfig` in a `SimpleStorageFactory`. A new constructor:

```go
func NewCRDRESTOptionsGetterWithStorageOverrides(
    etcdOptions genericoptions.EtcdOptions,
    resourceTransformers storagevalue.ResourceTransformers,
    tracker flowcontrolrequest.StorageObjectCountTracker,
    serverOverrides map[schema.GroupResource][]string,   // activated bindings only
    watchCacheDisabled sets.Set[schema.GroupResource],   // static ∩ watch-cache-sizes#0
) genericregistry.RESTOptionsGetter
```

is called from `pkg/controlplane/apiserver/apiextensions.go` with maps computed during config
construction (before the listener opens):

```
overrideCandidates  := staticManifestGRs ∩ ParseEtcdServersOverrides(--etcd-servers-overrides)
watchCacheDisabled  := staticManifestGRs ∩ {gr | ParseWatchCacheSizes(--watch-cache-sizes)[gr] <= 0}
serverOverrides     := {gr ∈ overrideCandidates | pre-activation check passed OR recorded witness matches}
```

The maps are immutable for the process lifetime. The crdHandler never asks "is this static?" —
a dynamic CRD at a flag-named GroupResource simply has no entry. The anti-escalation property
is structural, not a bypassable check. Two consequences interlock: because the maps are
startup-frozen, a manifest reload must refuse to add or remove a CRD whose GroupResource either
flag names (otherwise the add would silently get the default backend, and a removal would leave
a dormant entry a later CRD could inherit); see Manifest Reload.

The override-aware factory (`staticCRDStorageFactory`) replacing `SimpleStorageFactory`:

- `NewConfig` copies the base `StorageConfig` and swaps **only** `Transport.ServerList` for
  GroupResources in the map.
- `ResourcePrefix` stays `gr.Group + "/" + gr.Resource`, byte-identical to
  `SimpleStorageFactory`. Non-negotiable: `DefaultStorageFactory.ResourcePrefix` falls back to
  the lowercased resource with **no group segment**
  (`staging/src/k8s.io/apiserver/pkg/server/storage/storage_factory.go`), so reusing it would
  silently relocate every CR from `/registry/argoproj.io/workflows/...` to
  `/registry/workflows/...` — data loss on upgrade. An integration test pins the prefix.
- `Prefix`, `Codec`, and `EncodeVersioner` untouched. The transformer-wrapping factory
  (encryption) composes over it unchanged.
- `Configs()` returns the base config plus one config per override backend, so the db-size
  monitor can observe override backends (see Bug Fixes for the getter-clobbering fix this
  depends on).

Compaction needs zero changes: the compactor registry is process-global and refcounted by
transport config; the first storage built against the new backend starts its compactor, the
last destroy stops it.

#### Watch Cache Control for Custom Resources

Implemented by *synthesizing* the existing mechanism rather than adding logic: the constructor
sets the copied options' `WatchCacheSizes` to exactly the filtered static entries and builds the
getter from the copy. That one change simultaneously implements the feature and fixes the
dead-code leak (the current code nils the copy and then uses the original). Semantics are the
existing ones, now pinned by tests:

- Global `--watch-cache=false` wins absolutely (`EnableWatchCache` gates the whole block in
  `StorageFactoryRestOptionsFactory.GetRESTOptions`).
- Size `0` swaps the decorator to `generic.UndecoratedStorage` for that GroupResource (all
  served versions).
- Positive sizes log "Dropping watch-cache-size ... watchCache size is now dynamic" and do
  nothing; negatives are rejected at parse. The honest description of this control is a
  boolean: disable, or default.

Costs of disabling, stated here because they are the review surface: every UPDATE and DELETE
gains an etcd read (the cached-object suggestion to `GuaranteedUpdate`/`Delete` is gone); N
watchers become N etcd watch streams with N decodes and no serialization memoization; periodic
watch bookmarks disappear (only the `initialEventsEnd` bookmark survives); exact-RV and
continue-token lists lose snapshot serving, so long paginated walks that outrun etcd compaction
get `410 Expired` with no cache fallback; and APF's list cost estimation is corrected for these
resources (below). WatchList/sendInitialEvents keeps working (the etcd3 watcher implements it
natively), at full-LIST cost per establishment.

APF correction, scoped: the list work estimator currently assumes rv=0 and consistent lists are
cache-served (`staging/src/k8s.io/apiserver/pkg/util/flowcontrol/request/list_work_estimator.go`
carries `// TODO: Check whether watchcache is enabled.`), so an uncached hot resource gets
~10-seat charges for full etcd range reads. This KEP plumbs a per-GroupResource
cacher-disabled signal into the estimator and enables storage-side size estimation **for the
GroupResources in the static watch-cache-disabled set only**, leaving every existing cluster
(including `events#0` and `--watch-cache=false` clusters) bit-identical. The general fix — the
same correction for all uncached resources — is a genuine improvement but is an un-gated APF
behavior change for every cluster and does not belong inside an alpha feature; it is filed
separately.

```
<<[UNRESOLVED sequencing of the general APF estimator fix @wojtek-t @MikeSpreitzer ]>>
Should the general estimator correction (all uncached resources, resolving the in-code TODO)
land as its own reviewed PR before or alongside this KEP's scoped version, so the scoped
version is a subset rather than a special case? Doing it properly improves existing
--watch-cache=false and events#0 clusters, which argues for a standalone PR with its own PRR
note; this KEP takes only the scoped correction either way.
<<[/UNRESOLVED]>>
```

```
<<[UNRESOLVED interaction with KEP-5866 sharded list/watch @jefftree ]>>
Shard filtering lives on the Cacher's dispatch path, so an uncached custom resource cannot
serve sharded list/watch. The two features target the same workload (huge resources) in
opposite directions. Decide: reject sharded requests against an uncached resource, or fall
back to unsharded? Cheap to decide now, expensive to discover at beta.
<<[/UNRESOLVED]>>
```

#### The Storage Binding Witness

The witness is the durable record of which backend a resource's data lives in:
`storage.apiextensions.k8s.io/etcd-servers: "default" | "sha256:<hash>"`, stamped on the CRD
object (which lives in the default backend and is readable by every member) for every static
CRD, atomically with creation or at activation time. The hash is computed over the
canonicalized override server list (sorted, deduplicated, byte-exact URLs); the
canonicalization algorithm is a versioned compatibility contract — changing it would 503 every
bound resource on upgrade.

Known wart, accepted for alpha: rotating an override backend's *endpoint list* (same logical
etcd cluster, new URLs) changes the hash and trips the guard, requiring the operator to
acknowledge via the unlock command even though no data moved.

```
<<[UNRESOLVED witness value and location @serathius @wojtek-t @liggitt ]>>
Two coupled questions:
1. Value: a server-list hash (alpha proposal) versus a stable operator-supplied backend
   identity. An identity fixes the endpoint-rotation wart, but there is nowhere to declare one
   without either extending the override flag grammar (declined on k/k#82580: "I don't love
   extending the ad-hoc string format further") or the versioned registration config kind from
   the configuration-surface UNRESOLVED. This is the strongest argument for that config kind at
   beta, alongside per-backend etcd TLS.
2. Location: metadata annotations (alpha proposal; precedent:
   `kube-aggregator.kubernetes.io/automanaged`, `rbac.authorization.kubernetes.io/autoupdate`)
   versus a new `status.registration` stanza on the GA CRD type. Annotations cost permanent
   GitOps diff noise, require the reserved-key admission rules above, and leak storage topology
   to any `get customresourcedefinitions` reader; a status stanza is excluded from apply
   ownership and diffs but is a larger GA API change than the new condition type. Alpha ships
   annotations; beta revisits.
<<[/UNRESOLVED]>>
```

#### Pre-Activation Emptiness Check

Runs **synchronously during config construction, before the listener opens** — it needs only
the flags, the loaded manifests, the existing CRD objects (read via a direct storage client, not
the loopback API), and etcd clients, none of which require serving. For each override
candidate GroupResource, compare the computed binding to the recorded witness (absent, for a
brand-new CRD):

- Witness matches computed: activated (steady state).
- No witness / witness differs (activation or rebinding): issue one ranged etcd `Get` with
  `WithPrefix()`, `WithCountOnly()`, `WithLimit(1)` against
  `<etcd-prefix>/<group>/<plural>/` on the **source** backend (the currently
  witnessed backend, or the default backend for first activation).
  - Count zero: activate; the reconciler stamps the witness (in the CREATE for new objects;
    as a marker update for adopted ones).
  - Count non-zero: **refuse to activate.** The GroupResource is left out of the override map;
    the resource keeps serving from where its data actually is; a startup log, the
    `StaticSynced` condition reason, and
    `apiserver_custom_resource_storage_precheck_total{result="refused-nonempty"}` name both
    backends and the migration runbook. Because the check re-runs at every startup, the
    migration flow is natural: quiesce writes, copy `/registry/<group>/<plural>/` keys to the
    target backend (`etcdctl make-mirror` or snapshot/restore with a key filter), **delete
    them from the source**, restart — the next startup finds the source empty and activates.
  - Source unreachable: fail closed for that resource (cannot prove emptiness, do not bind);
    `result="refused-unreachable"`.

This is one `Count` RPC per affected CRD per startup. It is what makes the answer to "an
operator who set this override years ago believing it worked" structural rather than
documentary: enabling the feature cannot relocate their data, because the check finds it.
StorageVersionMigration is explicitly not a migration tool here — it reads and writes whichever
backend is currently configured and would report success against the empty target.

#### The Per-Request Binding Guard

The guard is evaluated **per request**, in the crdHandler's serving path alongside the existing
NamesAccepted/Established gate — not once at storage construction. This matters because storage
teardown is triggered only by spec or acceptedNames changes
(`customresource_handler.go` `updateCustomResourceDefinition` explicitly ignores updates where
"neither spec, nor accepted names changed"), so an annotation-only witness change would never be
noticed by construction-time-only logic, and a member that built storage before a witness
appeared would keep serving from the wrong backend indefinitely. Rules, comparing the object's
current witness (from the lister — a map lookup, no I/O) to this member's effective binding:

- Witness present, equals effective binding: allow.
- Witness present, differs: **deny with HTTP 503**, `reason: StorageBindingMismatch`, a message
  naming the CRD, the recorded value, this member's computed value, the *name* of the flag that
  produced it (never the flag's value — server lists and endpoints stay out of client-visible
  errors, per Design Rule: No Admin-Environment Detail on API Objects), and the exact
  `kubectl annotate --overwrite` unlock command, whose argument is the computed hash. No `Retry-After` (retry does
  not help until an operator or a rollout acts). An audit annotation records the rejection.
- Witness absent, effective binding is default: allow (every pre-feature CRD in every cluster).
- Witness absent, effective binding is an override (activation pending its stamp): **deny 503**
  until the reconciler stamps — briefly unavailable rather than briefly wrong. (For brand-new
  static CRDs this window does not exist: no object means requests 404 via the delegate before
  any storage is built, and the reconciler creates the object with the witness already present.)

Companion mechanics:

- A witness-annotation change observed by the informer triggers an explicit storage
  invalidation (`crdHandler` gains an `InvalidateStorageFor(uid)` entry point wrapping the
  existing `removeStorage_locked`/teardown path), so a member serving from a now-mismatched
  backend also tears down its live cacher and etcd watches rather than merely 503ing new
  requests.
- The guard's denial must not be swallowed into the crdHandler's generic
  `error resolving resource` internal error (the current path discards the real error into logs
  and returns a bare 500 — verified at `customresource_handler.go`, the
  `getOrCreateServingInfoFor` error branch). A typed error and a dedicated branch return the
  503 shape above; the integration tests assert on it.
- The blocked resource **stays in discovery**: removing it would make namespace deletion
  silently skip its CRs and GC drop its edges. The documented consequence (namespaces
  containing the resource stay Terminating, GC monitors retry, for the duration of the
  mismatch) is in Risks and Troubleshooting; the trade is deliberate — a wedge that names
  itself beats silent data divergence.

#### Health and Readiness for Override Backends

`EtcdOptions.ApplyTo` already registers `etcd-override-<n>` healthz and
`etcd-override-readiness-<n>` readyz checks per distinct override server set, straight from the
flag with no storage-factory involvement (`etcd.go`, `addEtcdHealthEndpoint`) — so basic health
probing of a CR-targeted backend works today, before this KEP. One change: when the storage
gate is enabled, override entries that resolve **only** to static-CRD GroupResources are
registered as healthz checks but excluded from the global readyz gate. Rationale: an operator
who moves one custom resource onto a dedicated etcd must not thereby make that etcd's
availability a precondition for the entire apiserver being in the load-balancer rotation —
consistent with the per-resource-degradation philosophy of the guard. Backend unavailability
surfaces as per-resource request errors, the storage binding gauge, and the named healthz
check. Entries covering any built-in resource keep today's global readyz behavior. (For
previously inert CR-only entries this loosens readiness — the safe direction — and is release
noted.)

#### Storage Warm-Up

Custom resource storage is built lazily on first request, holding the single
`customStorageLock` across etcd client construction — up to two ~20s blocking dials against an
unreachable backend, serializing *every other* CRD's first-touch storage construction, with
errors not cached. Acceptable when the only backend is the main etcd (if it is down the server
is dead anyway); a cross-CRD availability hazard when a per-resource backend can be
independently down.

A post-start hook therefore pre-touches storage for every static CRD with a non-default
binding: bounded concurrency (small fixed number), a per-CRD deadline after which warm-up gives
up and leaves that storage lazy, completion metrics. Warm-up is asynchronous and does **not**
register with the storage readiness hook — gating `/readyz` on a cold LIST of a 100k-object
resource is a startup-latency regression this KEP declines. Moving etcd dialing out from under
`customStorageLock` (so a slow backend cannot serialize unrelated CRDs even on the lazy path)
is a targeted improvement scheduled with alpha implementation; if it slips, the bound and
deadline contain the exposure and "first-request latency for unrelated CRDs during warm-up" is
a measured test-plan item either way.

### Lifecycle and Precedence

#### Precedence Rules

1. `--watch-cache=false` (global) wins absolutely; per-resource settings are irrelevant.
2. A storage flag entry affects a custom resource iff **all** hold: `CustomResourceStorageOverrides`
   enabled; the GroupResource statically registered *at process start*; and (for etcd
   overrides) the binding activated by the pre-activation rules and matching the witness at
   request time.
3. Dynamically created CRDs never receive either control — structurally (no map entry), plus
   name-claim denial at CREATE.
4. Built-in > aggregated-with-service > CRD, unchanged: the delegation chain places
   apiextensions innermost, so a static CRD can never shadow a compiled-in resource, and the
   loader additionally rejects claims on compiled-in group/versions. If an admin deletes an
   aggregated APIService whose group/version a static CRD also claims, the CRD silently takes
   over serving (pre-existing CRD semantics); the loader warns at startup when a static CRD's
   group/version is claimed by a service-backed APIService.
5. Within CRDs, name conflicts are adjudicated by the existing machinery on real objects: the
   naming controller's group-wide accepted-names union applies to static CRDs like any others.
   A static CRD whose kind/shortNames collide with an established dynamic CRD in the same
   group gets `NamesAccepted=False` like any late claimant. One honesty note: serving is built
   from `status.acceptedNames`, not `spec.names`, and `Established` is sticky — so a name
   change can converge in *spec* (`StaticSynced=True`) while the cluster keeps serving the old
   accepted names indefinitely, with no failure anywhere. The reconciler therefore also
   compares `spec.names` against `status.acceptedNames` and exposes divergence via
   `frozen{reason="names_divergent"}` and a log; the effect gap is visible even though the
   spec has converged.

#### Startup Sequence

1. Flags parsed; manifests loaded, strictly decoded, statically validated — **any Tier-1
   error, or a Tier-2 error with no existing object of that name, is fatal, before the
   listener opens** (see the two-tier rules in Loading).
2. Flag/manifest cross-validation: grammar diagnostics, wildcard warnings, stale-entry
   warnings.
3. Pre-activation emptiness checks run (direct etcd reads); the frozen override and watch-cache
   maps are computed; the CR RESTOptionsGetter is built.
4. Serving starts (`SecureServingInfo.Serve` runs before post-start hooks —
   `staging/src/k8s.io/apiserver/pkg/server/genericapiserver.go` — which is why steps 1-3 must
   precede it and why nothing in them may require the loopback client).
5. Post-start: the reconciler waits for the CRD informer's initial sync (the behind guard
   fails closed until then — a cold cache must never be read as "no records exist"), then
   arbitrates and creates/adopts objects and stamps witnesses (etcd unreachable: retry with
   backoff, `static-crds` readyz check unready, server stays up). Naming and establishing
   controllers run normally; note the establishing controller's existing 5-second delay when
   `masterCount > 1` applies to static CRDs too and is not bypassed at alpha.
6. Storage warm-up runs (bounded, async).
7. `static-crds` readyz passes per the Readiness Contract.

#### Manifest Reload

`filesystem.WatchUntil` (fsnotify + 1-minute polling fallback), sha256 short-circuit, strict
re-validation, atomic swap, keep-last-known-good, and **poison-pill** on structurally invalid
content (the authentication-config policy, not encryption's retry-forever: a bad schema never
becomes valid by retrying, so the same bad content is not retried every minute).

Both configuration layers are watched and reload identically. A change to the
`CustomResourceConfiguration` file that repoints `staticManifestsDir` is treated as a wholesale
manifest-set change, subject to the same rules below — in particular, a repointed directory can
add, update, and (via tombstones) demote, but a name merely *absent* from the new directory is
unclaimed, never demoted: the reconciler never infers anything from absence. Both layers feed
the `..._last_config_info` hash, so per-member drift in either is detectable by the same alert.

Tombstones dissolve the previous draft's most awkward rule — wholesale rejection of removals —
because intentional removal is now a positive document and absence is uniformly meaningless.
Startup and reload process tombstones identically. The taxonomy, each entry with its reason:

**Reload-safe** (detected within the ~1-minute polling fallback worst case):

| Operation | Why |
|---|---|
| Adding a CRD whose GroupResource is in neither storage flag | Object create; storage built on demand |
| Spec updates passing `ValidateCustomResourceDefinitionUpdate` (schema/CEL/printer-column/served-flag changes) | Same class of change dynamic CRDs make today, with the same per-member storage teardown/drain/rebuild cost — actuated once fleet-wide by the first member whose revision outranks the record |
| **Demoting via tombstone** a CRD whose GR is in neither storage flag | The demote is a metadata-only write: zero storage churn, zero watch interruption, no `Established` transition |
| Re-promotion of an unbound CRD without spec change (reverting a demotion) | Metadata-only adoption |
| A revision bump with unchanged content | Annotation-only confirm writes; free at every layer |
| Removing a tombstone file (beta, after the lease-floor convergence check; tombstones are permanent at alpha) | Inert — the on-object record carries the fence |
| A CRD file disappearing **without** a tombstone | The name becomes *unclaimed by this member*: no object changes, label protection unaffected; a member whose configuration outranks the record writes the `ManifestRemovedWithoutTombstone` diagnostic condition. Never a demotion. |

**Rejected atomically** (whole reload, keep-last-known-good, failure metric, poison-pill — the
authentication-config policy, since a bad schema never becomes valid by retrying):

| Operation | Why |
|---|---|
| Any document failing Tier-1 validation; a conversion webhook; a supplied `status`; a missing, duplicated, or malformed `CustomResourceManifestSet` | Fail closed on operator-owned input. Requiring the manifest-set document is also the broken-mount guard: a transiently empty or half-projected directory is structurally invalid and keeps last-known-good, rather than being read as intent. |
| Adding **or re-promoting** a CRD whose GroupResource is named in `--etcd-servers-overrides` or `--watch-cache-sizes` | The storage maps are startup-frozen; activating a binding requires the startup-time emptiness check |
| A tombstone for a CRD whose GroupResource is named in either storage flag | The frozen override map would carry a dormant entry for a no-longer-static GR that a later dynamic CRD in the same process lifetime could inherit; the log names the CRD, the flag, and the drain-first restart procedure |

**Fails at reconcile, not at reload:** a spec change that server-side update validation rejects
(e.g. dropping a version still in `storedVersions`), and Tier-2 content errors against existing
objects. The old spec keeps serving, `StaticSynced=False`, metric, log; self-heals when the
operator completes the migration (see storedVersions) or reverts (with a bump). Readiness is
unaffected (see Readiness Contract).

An *entirely empty* readable directory remains valid at startup and at reload: it asserts
nothing, claims nothing, changes nothing — labeled objects stay protected (their protection
keys on the object) — and a warning is logged when labeled objects exist (likely a broken
mount, though note the manifest-set requirement means most mount failures reject instead). One
honest asymmetry, owned: `--watch-cache-sizes` has none of the etcd-client/healthz-registration
entanglements that force the override map to freeze, and *could* be made reloadable; it stays
restart-only as an anti-escalation policy choice for symmetry with the etcd override, not as a
mechanical necessity.

#### Readiness Contract

Readiness is a pure function of current state, never of restart history (a readiness predicate
depending on process history takes members out of rotation one-by-one on routine restarts for
a condition that was green fleet-wide before them). One rule:

> The `static-crds` readyz check is ready when, for every CRD in this member's manifest set,
> a CustomResourceDefinition object exists and has `Established=True` — regardless of whether
> its spec currently matches the manifest — or the reconciler has not yet completed its first
> pass.

Every reconcile failure — terminal conflict, rejected version drop, Terminating adoption
refusal — sets `StaticSynced=False` with a reason, increments
`reconcile_total{action="conflict"}`, and logs; none of them fail readiness once the type is
serving from *some* valid spec. The revision states are equally excluded, with a stronger
argument: because a manifest change lands fleet-wide at the first member's write, n−1 members
are simultaneously "behind" the moment any roll begins — a readiness predicate keyed on
convergence would take most of the fleet out of rotation at the *start* of every rollout, the
inverse of a safe rolling update. Behind and frozen members serve identically (they read the
same object), stay ready, and report via gauges and logs only. Consequences worth naming: a
bad manifest edit made after initial registration never takes members out of rotation (it
surfaces via the unsynced metric and condition instead), and a pre-created conflicting object
cannot be used to take the fleet out of rotation on its next restart — the divergence is loud
but not availability-affecting. The signals to alert on are `apiserver_static_crd_unsynced > 0`
persisting, `frozen > 0` (pages: misconfiguration with a printed remedy), and `config_behind`
sustained beyond the operator's expected roll duration (no fixed default threshold: the last
node to roll is legitimately behind for the entire roll, and roll durations range from minutes
to days).

#### Demotion and Decommissioning

The feature never deletes custom resource data, and — with tombstones — no step of
decommissioning requires an API write. The flow:

1. **Demote declaratively:** replace the CRD's manifest file with a `Demote` tombstone, raise
   the manifest-set revision, commit, roll out. For a CRD whose GroupResource is in neither
   storage flag this is reload-safe end-to-end: no restarts. The first member whose
   configuration outranks the record executes the demotion — one metadata-only write removing
   the label and content hash, retaining the witness, stamping the record — and removes the
   `StaticSynced` condition. Members still holding the live manifest at an older revision read
   the record and defer; members that receive the tombstone later find themselves converged
   and at most confirm. The witness annotation is **retained** through demotion: it is the
   data-location record, and the guard keeps honoring it, so a demoted CRD bound to an
   override keeps serving from the right backend (while the flag remains) instead of silently
   reading the empty default backend.
2. The object is now an ordinary dynamic CRD — editable and deletable, fleet-uniformly, from
   the moment the demote write lands (protection keys on the label). Deleting it, with the
   normal finalizer cascade, is now an intentional, RBAC-governed, audited act by whoever owns
   the data — deliberately an API act, never a manifest act: "stop managing" and "delete" are
   separable, and the manifest layer only ever expresses the former.
3. If an override was in play and the operator wants the data back on the default backend:
   migrate the keys back (reverse runbook), then clear the witness via the unlock rule and
   drop the flag entry. For a *bound* CRD this ordering is mandatory and restart-scoped:
   drain, clear witness, drop flag, then tombstone + restart (the tombstone-for-a-bound-GR
   reload rejection enforces the order; beta's stale-flag-entry fatality hardens it).

The previous draft's manual demotion — a hand-written API write stripping the markers after a
fleet-wide restart — is gone as a documented flow, retained only as the audited break-glass
allowance described in Write Protection (for when the manifest distribution channel itself is
broken).

#### Transitions Between Dynamic and Static Registration

Per CRD name there are four states — **Absent** (no object), **Dynamic** (ordinary CRD),
**Static** (label + record, manifest-managed), **Demoted** (record without label: a Dynamic
with history) — and every valid transition is declarative except the two that destroy data,
which are deliberately API-scoped:

| # | Transition | Trigger | Scope |
|---|-----------|---------|-------|
| 1 | Absent → Static | New manifest + bump | Reload (restart if flag-named) |
| 2 | Dynamic → Static (promotion) | Manifest added at a revision outranking any record | Reload (restart if flag-named) |
| 3 | Static → Static′ (spec change) | Manifest edit + bump; server-side update validation adjudicates | Reload |
| 4 | Static → Demoted | Tombstone + bump | Reload (restart if flag-named, drain first) |
| 5 | Demoted → Static (re-promotion) | Manifest re-added at a newer revision — including a plain `git revert` of the demotion commit | Reload |
| 6 | Demoted/Dynamic → Absent | Ordinary `kubectl delete crd` by the data's owner — audited principal, normal cascade | API, deliberately |
| 7 | Static → Absent | **Not offered.** The composition 4 + 6 is the supported path; data deletion stays out of the manifest layer categorically | — |
| 8 | Any rollback | `git revert` of the content plus a bump — a newer revision asserting prior content; no special casing | Same as the forward transition |

Non-transitions, equally explicit: absence without a tombstone transitions nothing, from any
member, at any time; denied API writes transition nothing; gate disablement transitions Static
to inert-marked Dynamic as already described.

**The safety theorem the whole transition story rests on, stated once: the manifest CRD and
the API CRD are the same object.** Coexistence is one object with two agreeing declarers, and
handoff is the removal of a label. There is no gap because there is nothing to hand over, and
no data motion because etcd keys are group/plural-based and UID-independent.

**Direction 1 — promotion (Dynamic → Static):**

1. *Capture.* `kubectl get crd <name> -o yaml`; strip `status`, `uid`, `resourceVersion`,
   `generation`, `creationTimestamp`, `managedFields`,
   `kubectl.kubernetes.io/last-applied-configuration`, and tool-ownership annotations. The
   export is already defaulted, so it agrees with the canonical spec by construction;
   hand-authored manifests differ only pre-defaulting, which the loader normalizes.
2. *Coordinate — the pre-flight, all of it checkable before anything changes:* enumerate the
   CRD's current writers (Helm release, Argo Application, Flux Kustomization, OLM CSV, and —
   easily missed — any controller that force-installs its own CRDs at boot, which will
   CrashLoopBackOff rather than merely go red if its bundle diverges). Each must either keep
   applying identical content (which keeps working forever) or stop managing the CRD. Confirm
   `kubectl diff` / `--dry-run=server` of the manifest against the live object is empty. For
   Argo-managed CRDs, annotate `argocd.argoproj.io/sync-options: Prune=false` in the source
   *before* promoting, or Application deletion will later wedge on the resources finalizer
   against the DELETE denial. Know that adoption **severs `ownerReferences`** (an OLM-owned
   CRD stops being garbage-collectable with its CSV — that is the point).
3. *Commit* the manifest plus a bump; roll. Agreeing spec ⇒ adoption is one metadata-only
   write (markers, record, witness, ownerReference severance): zero storage teardown, zero
   watch interruption, `Established` never flips, and protection turns on fleet-atomically at
   that write. Divergent spec ⇒ **the manifest wins**: a force-update through
   `ValidateCustomResourceDefinitionUpdate` overwrites the API-authored spec — stated bluntly
   as the promotion contract (*promote only content you mean to impose*) — and a validation
   refusal is a terminal conflict: object unchanged, unlabeled, still editable,
   `StaticSynced=False`, readyz green.

**Direction 2 — demotion with handoff (Static → Demoted, the agree-then-drop CUJ):**

1. *Coexist.* The API-side owner carries the identical CRD and syncs green forever at zero
   marginal cost (identical writes are admission-allowed no-ops that never reach etcd).
2. *Verify agreement without reading their repo.* The gate:
   `apiserver_static_crd_denied_writes_total{reason="spec_divergent"}` for the CRD, flat at
   **zero on every member** across the observation window. A climbing counter means someone
   will rewrite the spec the instant protection drops — this metric is a documented
   pre-demotion readiness gate, not telemetry.
3. *Verify fleet convergence* at or past the current revision (see Convergence Observability):
   agreement must hold when protection releases, and the release is performed by the first
   member to receive the tombstone.
4. *Drop:* ship the tombstone per Demotion above. Protection releases fleet-atomically at the
   demote write. Same UID, same spec, same `storedVersions`, `Established` never flips, no
   storage teardown, no discovery blip, no data motion.
5. *After:* the owner's next sync succeeds exactly as before — it was already succeeding;
   divergent syncs now also succeed. `kubectl wait --for=condition=Established` never
   regresses at any point in either direction. Residue: the revision record and the witness.
   Optionally, the owner later deletes the CRD — transition 6.

**What "agrees" means, precisely:** (i) semantic deep-equality of the *canonical specs* —
decode, v1 defaulting, internal-version comparison, exactly the one shared function from
Loading; (ii) spec only — `metadata.labels`, `metadata.annotations` (including
`last-applied-configuration`), `ownerReferences`, `finalizers`, and all of `status` are
excluded and governed by their own admission rows; (iii) for the handoff, the API-side desired
state must additionally be safe to become authoritative: it must not drop a version still in
`status.storedVersions` (existing validation enforces this at the moment it tries), should not
un-serve a version clients still use, and cannot change `scope`/`names.kind`/`group`/`plural`
anyway. Round-trips compose with no special cases; one caveat is inherited from the one-way
class: an API-side edit made *while demoted* that added a version now in `storedVersions`
blocks re-promotion until a storage version migration trims it, surfacing as
`StoredVersionsBlocked`.

#### storedVersions and Storage Version Migration

This entire problem class is solved by the object being real, and the KEP makes no stronger
claim than that:

- CR writes append the current storage version to `status.storedVersions` via the unchanged
  registry strategy.
- A manifest edit dropping a version still present in `storedVersions` is rejected by the
  unchanged `ValidateCustomResourceDefinitionStoredVersions` at reconcile time ("must remain
  in spec.versions until a storage migration ensures no data remains persisted"), surfacing as
  `StaticSynced=False` while the previous spec keeps serving. There is no path — manifest edit,
  restart, or reload — that silently strands stored data on an undecodable version.
- The in-tree storage version migrator finds the CRD object by name like any other, sets its
  conditions, and trims `storedVersions` after migration, unchanged. The supported flow for a
  storage version bump on a static CRD is exactly the dynamic-CRD flow: add the new version in
  the manifest (reload applies it), let SVM rewrite, then drop the old version in the manifest.
  No apiserver restart is required for any step of it.
- Multi-version static CRDs are therefore permitted at alpha. (Conversion between versions is
  restricted to `strategy: None` — see Non-Goals.)

The version-drop case is this KEP's one honest exception to "every transition is
manifest-expressible", and it is surfaced rather than papered over: a manifest edit dropping a
version still in `storedVersions` parks as `StaticSynced=False, reason: StoredVersionsBlocked`,
with a message naming the **exact** `storagemigration.k8s.io` StorageVersionMigration object to
create; the record does not advance; the previous spec keeps serving; readiness is unaffected.
The reconciler does **not** auto-create StorageVersionMigration objects at alpha — a
config-file edit must not be able to trigger a cluster-wide rewrite of every custom resource of
a type without an acting principal in the audit log (Non-Goal 9). At beta, an opt-in
`storageVersionMigration: Automatic` field on `CustomResourceConfiguration` is the candidate
design (the in-tree migrator is GA and on by default as of v1.37, so the dependency is safe),
with a stalled or failed migration surfacing as `StaticSynced=False` rather than a retry storm.
The supported alpha flow is exactly the dynamic-CRD flow, two-phase and explicit: create the
named SVM object, wait for the trim, and the reconciler's next pass applies the drop.

For rollback interactions — a storage-version *move* is one-way; a revert leaves both versions
in `storedVersions` — see What Is Revertible under Manifest Revisions and Convergence.

One honest caveat, so the KEP does not overclaim: `storedVersions` on the *status subresource*
path is validated only structurally (`ValidateUpdateCustomResourceDefinitionStatus` checks
ObjectMeta and status shape, not stored-version drops) for **dynamic** CRDs. A principal with
`update customresourcedefinitions/status` can clobber it on any dynamic CRD today; that is a
pre-existing gap, pursued as an upstream hardening separately. For **static** CRDs this KEP
closes it with the status shape rule in Write Protection (unchanged-or-SVM-trim-shape only) —
necessary because the reconciler has no repair path for status (the main-resource strategy
overwrites `newCRD.Status` with the old value on every update), so a clobbered `storedVersions`
would otherwise permanently veto every future manifest spec change. Sequencing note: this
static-CRD status rule is safe to ship at alpha precisely because the declarative
version-*drop* path above exists; the broader dynamic-CRD hardening must not ship before an
equivalent exists there, or the only escape hatch for a blocked version drop would be removed.
The guarantee this KEP claims for dynamic CRDs is exactly today's, preserved.

### Workflow Matrix

Bindings below: "etcd-B" is an override backend; "witness" is the
`storage.apiextensions.k8s.io/etcd-servers` annotation; "guard" is the per-request binding
guard; "record" is the `static-revision` (+ `static-superseded`) annotation pair; "behind" and
"frozen" are the arbitration states from Manifest Revisions and Convergence.

| # | Workflow | Behavior | Rationale / mechanism |
|---|----------|----------|------------------------|
| 1 | Fresh cluster, static CRD, no override | Manifests validated (Tier-1 error = process exit before listener opens); reconciler CREATEs object with markers + record + witness `default`; naming/establishing run normally; `static-crds` readyz passes | Real object; no controller edits |
| 2 | Fresh cluster, N members start together | No records exist, nobody is behind, all attempt CREATE; one wins, the others hit `AlreadyExists`, requeue, find the record, and no-op | Lost races re-read and re-arbitrate; never blind-retry |
| 3 | Fresh cluster, static CRD with override | Pre-activation check: default backend empty for that prefix; binding activates; object created with witness `sha256:...`; warm-up dials etcd-B, compactor starts; `etcd-override-N` healthz covers etcd-B | Data lands in etcd-B from the first write; the flag owner and the type owner are the same principal |
| 4 | Steady-state restart | Content hash unchanged; record matches; reconciler no-ops; witness matches; storage builds; ready after informer sync | Idempotent; pure function of current state |
| 5 | Startup, etcd unreachable | Server starts; reconciler retries with backoff; `static-crds` readyz unready; no crash loop | Bootstrap-RBAC precedent |
| 6 | Staggered manifest change (rev 47→48), node 1 rolled | Node 1 writes spec+markers+record once; serving switches fleet-wide at that instant (members serve the shared object); nodes 2–3 read record 48, go behind, write nothing | Exactly one spec write per roll; `config_behind` on two members is the *normal* mid-roll signature |
| 7 | Roll stalls for days (paused CAPI rollout, canary soak) | Steady and silent: un-rolled members serve the new content from the shared object, protect, stay ready, write nothing | No mechanism depends on the skew window being short |
| 8 | Revision bump, content unchanged | First rolled member advances each claimed CRD's record with an annotation-only confirm; no teardown, no generation bump, no OpenAPI churn | `customresource_handler.go` teardown fires only on spec/acceptedNames change |
| 9 | Rollback: revert content and bump (rev 49) | Ordinary forward change carrying old content; one spec write | Deployment rollout-undo / OpenShift `currentRevision` must-only-increase precedent |
| 10 | Bare `git revert` (content **and** revision restored), incl. automated rollout-undo tooling | Whole fleet ranks LOSES against its own records: inert, serving the content the admin meant to retire; nothing flaps; `config_behind` fleet-wide; log names both exits (bump past the record, or `supersedes`) | A wedge of convergence, never of serving, that names its own key; forward-only CI-derived revisions avoid it |
| 11 | `supersedes: [47]` recovery at revision 46 | First member writes the reverted content, records `revision: 46, superseded: 47`; members still running the 47 tree read a record that disowns them and go inert; sticky supersession survives the entry's later removal from git | Recovery sticks fleet-wide from the first write; next ordinary revision must be ≥ 48 (printed in the log) |
| 12 | Forgotten bump: content changed, revision untouched | First rolled member ranks TIE with matching record-vs-object hash: freezes that one CRD, previous spec serves, `frozen{reason="RevisionTie"}` + guarded condition + log with both hashes and the remedy; un-rolled members quiet | Fail-safe; never a hash tie-break; fix hot-reloads and converges with one write |
| 13 | Fork: two branches both bump to 48 with different content | Whichever differently-configured member reconciles first wins the race and its content serves fleet-wide; the other side then freezes loudly with both content hashes (member-to-tree mapping via the `last_config_info` gauges) | Contained, not prevented (two authors picking one number *is* the misconfiguration); debounce narrows the race; CI lint: unique revisions across branches |
| 14 | Hijack: one member ships `revision: 10^12` | It writes once; the correct fleet goes loudly behind; the hijacker gained no authority it lacked (admission, update validation, never-delete all still apply) | Recovery: `supersedes: [10^12]` — the hijacking member reads the mirrored disown and goes inert before its file is even fixed |
| 15 | Emergency SSH hand-edit on one member | Requires raising the revision above the record (the behind/frozen log prints the literal minimum winning value); an oversized emergency value is recovered later by one `supersedes` commit | Documented two-step runbook (Troubleshooting); no generator tooling required |
| 16 | Adoption: existing dynamic CRD, identical spec, no override | Reconciler adds markers + record via one metadata-only update, severing any pre-existing `ownerReferences`; `storedVersions` preserved; zero storage teardown; CRs keep serving on identical keys; protection on fleet-atomically at that write | Dynamic-to-static migration is a file copy + one reload; keys are group/plural-based and UID-independent |
| 17 | Adoption with override, CRD already has data in default backend | Pre-activation check finds keys: **binding refused**; resource keeps serving from default backend; no witness beyond `default`; startup log + `StaticSynced` reason + `precheck_total{result="refused-nonempty"}`; readyz green | Refusal is loud and safe; the migration runbook (quiesce, copy keys, delete source keys, restart) naturally activates on the next startup |
| 18 | Adoption: existing object, terminally divergent spec (e.g. scope differs) | Force-update rejected by update validation; existing object serves unchanged, **unlabeled and still editable**; `StaticSynced=False`; conflict metric; **readyz stays green**; process starts | Never mutate data-bearing identity (kcp#731); protecting an unadoptable object would freeze the only repair path — the operator edits it toward adoptability, then the reconciler adopts |
| 19 | Adoption: existing object already Terminating | Adoption refused (terminal conflict); reconciler neither updates nor stamps; condition + metric; the finalizer completes via the status carve-out | Adopting would let the finalizer cascade resume against a "protected" type, then recreate it empty |
| 20 | CRD deleted during a gate-off window; gate re-enabled mid-cascade | Object is labeled + Terminating; status writes and the sole-finalizer-removal carve-out let `crd_finalizer.go` finish (it removes its finalizer via `UpdateStatus`); reconciler refuses to adopt or recreate through the cascade | Without the carve-out the object would wedge Terminating forever, unadoptable and undeletable |
| 21 | Spec edited out-of-band (gate-off window), gate re-enabled | Record's content hash ≠ object's spec hash: **tamper**, not a tie; the member holding the recorded content repairs (`reconcile_total{action="repair"}`) | The recorded-hash-vs-actual-spec comparison is the tamper witness; restores the never-overridden invariant durably |
| 22 | `kubectl apply` of content identical to the object, any member, mid-roll or not | Allowed (spec no-op against the object); byte-identical serialization short-circuits in the etcd3 store: no RV bump, no watch event | Object-anchored rule is fleet-uniform; steady-state GitOps cost is a decode and a compare |
| 23 | Mid-roll apply of *old* content through a not-yet-rolled member | Denied (would change the object's spec) | The object, not the member's manifest, is the authority — old content cannot be written back through a stale member |
| 24 | Identical content via PUT shapes: `kubectl replace`, envtest `CreateCRDs`, Argo `Replace=true`, Flux `CreateReplace`, OLM | Mutating phase restores the omitted reserved keys; spec judged a no-op; allowed | Marker preservation is a server obligation, not a client one; these are the tools oversized CRDs are forced onto (256KiB last-applied limit) |
| 25 | `kubectl apply`/`edit` with a divergent spec | Denied with the sentinel message naming the changed field and who to ask — never a file name | Single source of truth; instructive failure; see Denial Messages |
| 26 | Divergent `kubectl apply --server-side` | 409 Conflict naming field manager `apiserver-static-crd` first (merge-time, pre-admission); with `--force-conflicts`, the 403 denial; nothing persisted either way | SSA conflicts and admission are independent gates; admission is later and sees the merged, defaulted object |
| 27 | `kubectl diff` / `--dry-run=server` against a divergent manifest CRD | Returns the plugin's denial verbatim; no diff output | Dry-run executes the full mutating+validating chain and skips only the etcd write |
| 28 | `kubectl label`/`annotate`, non-reserved key | Allowed | Operators legitimately label CRDs |
| 29 | Adding an ownerReference or finalizer to a static CRD | Denied | GC must not wedge on an impossible delete |
| 30 | `kubectl delete crd <name>` / `--all` | Denied per object, including for `system:apiserver`; collection delete errors on the protected object; message names the prevented data loss and the tombstone path | Kills the cascade footgun (`crd_finalizer.go` per-namespace DeleteCollection) categorically |
| 31 | Argo CD Application carrying the CRD is deleted (resources finalizer) | Prune hits the DELETE denial; the Application wedges in `Deleting` until a human intervenes — unless the CRD was annotated `Prune=false` in the source, which the promotion pre-flight requires | Documented per-tool consequence of unconditional DELETE denial |
| 32 | Vendor operator force-installs its own (divergent) CRD at boot | The install write is denied; the operator exits non-zero and CrashLoopBackOffs, stopping its CR reconciliation | Promotion pre-flight requires identifying self-installing controllers and confirming bundle agreement first |
| 33 | Status writes by controllers / SVM | Allowed: naming (requested-or-retained names), condition writers, SVM's trim all match the status shape rules | Establishing/naming/apiapproval/SVM keep working unmodified |
| 34 | Status write steering `acceptedNames` away from spec (kind-squat), or clobbering `storedVersions` | Denied by the status shape rules | Serving identity is built from `status.acceptedNames` and change authority from `storedVersions`; neither is reconciler-repairable (the main-resource strategy overwrites status), so admission is the only guard |
| 35 | User attempts to strip the protection label via the status subresource (typed or SSA prune) | Denied (metadata diff on status path; sole exception is the cleanup-finalizer removal) | Closes the k/k#137681 passthrough explicitly, for SSA too (status apply can own metadata) |
| 36 | User plants a witness or record key on any CRD (CREATE or UPDATE) | Denied by the server-owned reserved-key rules while the respective gate is on | Prevents a `patch crd` holder from 503ing arbitrary types or freezing the fleet's reconcilers with one write |
| 37 | Gate-off-window residue: unlabeled, unclaimed object carrying a planted record | Excluded from the behind guard's floor (not labeled, not asserted); reserved-key *removal* permitted on such objects for cleanup | One planted annotation must not wedge the feature fleet-wide, and must be cleanable without a gate flip |
| 38 | Storage version bump v1 -> v2 with migration, then drop v1 | Manifest adds v2 + bump (reload) -> SVM migrates and trims `storedVersions` -> manifest drops v1 + bump (reload). No restart at any step | Works verbatim because the object is real; the drop is blocked until SVM has trimmed |
| 39 | Manifest edit drops a version still in `storedVersions` | Reconcile update rejected server-side; previous spec serves; `StaticSynced=False, reason: StoredVersionsBlocked` naming the exact SVM object to create; record not advanced; readyz green; self-heals after SVM | The data-loss guardrail survives verbatim; the two-phase flow is the honest exception to manifest-only operation (no auto-SVM at alpha) |
| 40 | Storage-version move, then `git revert` of it | The move actuates and appends to `storedVersions` (logged, `action="storage-version-move"`); the revert restores serving but `storedVersions` keeps both — neither droppable without SVM | One-way class, named in What Is Revertible; the residue is a validation-enforced fact, not a bug |
| 41 | Hot reload: add a CRD (no flag-named GR) | Detected within ~1 min; validated; debounced if a new revision; reconciled; Established; served | Additive reload is the safe subset (5793 semantics) |
| 42 | Hot reload: add or re-promote a CRD whose GR is named in either storage flag | **Whole reload rejected**; last-known-good retained; `automatic_reloads_total{status="failure"}`; poison-pill; log names the CRD and the flag; restart required | Binding activation requires the startup-time emptiness check; the frozen maps are the anti-escalation mechanism |
| 43 | Hot reload: CRD file deleted **without** a tombstone | Reload accepted; the name is unclaimed by this member; no object changes; label protection retained; an outranking member writes the `ManifestRemovedWithoutTombstone` diagnostic condition | Absence asserts nothing; the mistake names itself on the object, not only in metrics |
| 44 | Hot reload: tombstone for an unbound CRD (the demotion) | Reload-safe; first outranking member demotes with one metadata-only write (label + content hash off, witness kept, record stamped) and removes `StaticSynced`; protection releases fleet-atomically | Zero storage churn; `Established` never flips; users observe nothing |
| 45 | Partial tombstone roll (1 of 3 members) | The demoting member acts once; behind members cannot re-promote (record outranks them) and their claims are irrelevant to protection (label-keyed) | The on-object record, not the git file, is the fence; no zombie protection behind the LB |
| 46 | Behind member later receives the tombstone | Finds itself converged (label absent); at most confirms the record | Records only advance; idempotent |
| 47 | Demote, owner API-deletes, ancient member returns with pre-tombstone manifests | CREATE blocked while any live labeled object records a newer revision (behind guard); if *every* record is gone (single-static-CRD retirement), recreation happens: loud (config-hash disagreement, revision gauges), empty, no data loss (the cascade already ran under an audited principal); repair = roll the member | The documented residual; tombstone permanence at alpha narrows it, the beta lease floor closes it |
| 48 | Tombstone naming a CRD that is neither manifest-known nor labeled | Inert, startup warning | Typo diagnostic; a tombstone legitimately outlives the object it retired |
| 49 | Hot reload: tombstone for a flag-named GR | **Whole reload rejected**; restart required, drain-first order in the log | Dormant-override-map-entry rule (see Declarative Removal) |
| 50 | Hot reload: schema/CEL/printer-column change | First outranking member updates the object once; existing spec-change teardown drains (bounded by 2x request timeout) and rebuilds per-version storage on every member; watchers relist | Identical blast radius to a dynamic CRD update today — but once per change, not once per member |
| 51 | Transiently empty or half-projected manifest dir at reload | Missing/duplicated `CustomResourceManifestSet` (or any Tier-1 failure): whole reload rejected, last-known-good, failure metric. An *entirely valid but empty* directory: accepted, asserts nothing, changes nothing, warning if labeled objects exist | The manifest-set requirement makes most broken mounts structurally invalid; pure absence is harmless by construction |
| 52 | Manifest change and storage-flag change land in one commit | The manifest half applies fleet-wide at first landing; the flag half applies per member at restart; the witness guard 503s the affected resource on skewed members until the restart roll completes | Loud, contained, self-clearing — and avoidable: the ordering rule (Notes item 9) forbids combining them |
| 53 | Reload-safe revision bump lands while a restart-only roll is incomplete | The already-restarted member goes behind and pauses its pending flag-bound create; self-heals when it receives the newer configuration (reload) or the roll completes | Transient by construction; visible via `config_behind` + an asserted-but-absent warning; avoided by Notes item 9 |
| 54 | Fleet decommission of an unbound CRD | Commit the tombstone + bump; roll ConfigMaps; optional later API delete by the data's owner. No restarts, no API pokes | The full declarative flow; contrast the previous draft's remove-everywhere + restart-everywhere + hand-written demotion write |
| 55 | Operator changes the override to a different etcd cluster while data exists | Pre-activation check against the witnessed backend finds data: no restamp; guard 503s the resource on members computing the new binding, naming both values and the unlock; after etcd-level migration + witness acknowledgement, serving resumes | The binding is immutable while data exists, enforced, not documented |
| 56 | Endpoint-list rotation for the same etcd cluster | Trips the hash; same acknowledgement flow as 55; the unlock write is *honored*, not restored away (mutating-phase precedence) | Accepted alpha wart; stable backend identity is the beta fix (see witness UNRESOLVED) |
| 57 | HA rolling first-enable of an override (2 of 3 members rolled) | Rolled members activate and stamp; the unrolled member's guard sees witness `sha256:...` vs its computed `default`: **503 for that one resource** on that member (its live storage also torn down via witness-change invalidation); converges when the roll completes | Per-request guard + invalidation make skew loud on day one; unavailable strictly beats dual-backend writes; behind an LB this presents as partial-fleet errors — named in Risks/Troubleshooting |
| 58 | Disable `StaticCustomResourceDefinitions` | Loader/reconciler/protection gone; objects remain ordinary CRDs with inert markers including the record; recorded storage bindings still honored (see 59); re-enable resumes arbitration where it left off | Registration rollback clean by construction; the record is durable |
| 59 | Disable `CustomResourceStorageOverrides` (or both) with a live binding | Recorded binding continues to be honored while the flag supplies the servers (un-gated honoring); no new bindings; full rollback = drain runbook | Gate-off must not silently serve an empty backend; see the gate-semantics UNRESOLVED |
| 60 | Flag entry removed while witness names etcd-B | Guard 503s the resource (computed `default` vs witness `sha256:...`) | Loud, reversible (restore the flag, or migrate + acknowledge) |
| 61 | Downgrade to a pre-feature binary with a live binding | The old binary ignores the witness and silently serves the empty default backend; controllers see an empty world | The one hazard code cannot guard; hard pre-downgrade requirement documented in Upgrade/Downgrade + Troubleshooting: migrate back and verify `binding{state="override"} == 0` first |
| 62 | User creates a dynamic CRD at a flag-named GR (escalation attempt) | If the name is manifest-claimed: CREATE denied. Otherwise the CRD is created and gets the **default** backend: the override map has no entry for it | Structural anti-escalation; CRD creation can never redirect data placement |
| 63 | Manifest declares `conversion.strategy: Webhook` or includes `status` | Fatal at load (Tier 1) | Alpha restrictions (see Non-Goals) |
| 64 | Static CRD in a `*.k8s.io` group | `validateAPIApproval` enforced at load (Tier 2: fatal only when no object exists) and at reconcile; the annotation is manifest-managed on protected objects | No exemption; no laundering path; adoption succeeds where fresh creation fails (create-vs-update asymmetry, documented) |
| 65 | Name change accepted in spec but rejected by the naming controller | Spec converges (`StaticSynced=True`) but serving keeps the old `status.acceptedNames` (Established is sticky, so nothing un-establishes); `frozen{reason="names_divergent"}` exposes the effect gap | Serving is built from acceptedNames, not spec.names; documented limit of `StaticSynced` |
| 66 | GC, quota, namespace deletion, `kubectl wait/get/explain/api-resources`, aggregated discovery, peer proxy (UVIP) | Unchanged | Real object on every member; the invisible design's asymmetries do not arise |

### Bug Fixes Carried by This KEP

Each is a standalone, individually reviewable change; the first is a prerequisite.

1. **The `--watch-cache-sizes` dead-code leak.** Restore the intended nil-ing in
   `NewCRDRESTOptionsGetter` (pass the copy, not the original), so the flag once again does not
   apply to dynamically created CRDs — then this KEP re-introduces it deliberately, gated, for
   static CRDs only. Because the accidental behavior has existed since ~v1.28, the removal is
   held to the KEP-5793 `ExcludeAdmissionWebhookVirtualResources` standard, not silently
   release-noted: its own opt-out mechanism for at least one release, a startup log naming
   every `--watch-cache-sizes` entry that resolves to a dynamically registered CRD, and a
   release note.

   **Decided: fix-and-gate.** This is settled for the purposes of this KEP; it is not carried as
   an open question. The dead line is restored so the flag once again does not apply to
   dynamically created CRDs, and this KEP re-introduces the control deliberately, gated, for
   static CRDs only. The bug itself is tracked as a standalone kubernetes/kubernetes issue filed
   independently of this KEP, so the dead line, the contradicting flag help text, and the absent
   test coverage can be fixed on their own timeline regardless of this KEP's fate.

   The removal is still announced rather than silently release-noted: a post to
   kubernetes-sig-api-machinery precedes it, and the opt-out window above runs for at least one
   release. If the announcement surfaces operators genuinely relying on the accidental behavior,
   the response is to extend the opt-out window and lengthen the migration runway — not to
   reverse the decision. Grandfathering was considered and rejected: it would bless an ungated,
   unpinned storage control on arbitrary dynamic CRDs, which is precisely the property this KEP
   exists to prevent, and the leak's own two-year invisibility is the argument against leaving
   such controls ungated.

2. **Storage db-size monitor clobbering.** `ApplyWithStorageFactoryTo` unconditionally calls
   the process-global `metrics.SetStorageMonitorGetter`; in kube-apiserver the aggregator's
   single-backend `SimpleStorageFactory` applies last and wins, so the STABLE
   `apiserver_storage_size_bytes` reports only the default backend even for existing built-in
   overrides. Add `SkipStorageSizeMetrics` to `EtcdOptions` (sibling of `SkipHealthEndpoints`),
   set by apiextensions and the aggregator. Shipping an etcd-isolation feature on top of a
   db-size metric that cannot see the isolated backend would be a poor bargain.
3. **Nondeterministic `storage_cluster_id`.** `DefaultStorageFactory.configs()` ranges a Go
   map, so the positional `etcd-<n>` labels of `apiserver_storage_size_bytes` reshuffle per
   process. Make the ordering deterministic (a bug fix that does not change the label schema).

```
<<[UNRESOLVED storage_cluster_id semantics @dgrisonnet sig-instrumentation ]>>
`apiserver_storage_size_bytes` is STABLE with a positional `storage_cluster_id` label. This
KEP proposes deterministic ordering (bug fix) plus a separate ALPHA join metric
(`apiserver_custom_resource_storage_binding{..., storage_cluster_id}`) rather than adding a
backend-identifying label to the STABLE metric. Confirm with sig-instrumentation that this is
preferred over petitioning to change the STABLE metric's labels.
<<[/UNRESOLVED]>>
```

4. **crdHandler nil-condition panic.** The create path dereferences
   `FindCRDCondition(crd, Established).LastTransitionTime` without a nil check; a CRD passing
   the serving gate via NamesAccepted alone panics on first CREATE. Defensive fix regardless of
   this feature.
5. **Swallowed serving errors.** The crdHandler's storage-resolution error branch returns a
   bare `error resolving resource` 500 and logs the real error; guard denials (and other
   storage errors) get typed, descriptive responses.

### Observability

#### Metrics

Following the KEP-5793 naming convention (`apiserver_<subsystem>_controller_*`,
`{status, apiserver_id_hash}` labels, sha256-hashed apiserver ID):

Registration health:
- `apiserver_static_crd_controller_automatic_reloads_total{status, apiserver_id_hash}`
- `apiserver_static_crd_controller_automatic_reload_last_timestamp_seconds{status, apiserver_id_hash}`
- `apiserver_static_crd_controller_last_config_info{apiserver_id_hash, hash}` — manifest
  content hash (5793-parity drift detection).
- `apiserver_static_crd_controller_last_storage_config_info{apiserver_id_hash, hash}` — hash
  over the *effective storage configuration*: sorted static names, resolved override map
  (GroupResource to sorted server list), watch-cache-disabled set. This, not the manifest
  hash, is the HA drift SLI: two members with identical manifest directories and different
  `--etcd-servers-overrides` values differ here and nowhere else.
- `apiserver_static_crd_controller_reconcile_total{group, resource, action}` — action in
  `create|adopt|update|confirm|repair|demote|storage-version-move|conflict|restore`.
- `apiserver_static_crd_controller_last_reconcile_success_timestamp_seconds{group, resource}`
  — a counter alone cannot express "wedged for six hours"; this can.
- `apiserver_static_crd_registered{group, resource}` (gauge, bounded cardinality:
  operator-authored set) and `apiserver_static_crd_unsynced` (gauge; the alert).

Revision convergence (see Manifest Revisions and Convergence for alert guidance):
- `apiserver_static_crd_controller_manifest_revision{apiserver_id_hash}` (gauge) — this
  member's loaded revision. Fleet spread = roll progress; a wild outlier = the fat-finger
  detector.
- `apiserver_static_crd_controller_config_behind{group, resource}` (gauge, 0/1) — set for the
  object(s) whose record outranks this member's configuration, so the *blocking object* is
  identifiable from the metric, not only the log (bounded cardinality: labeled-or-asserted
  objects only). Nonzero during any normal roll; alert only on duration beyond the
  operator-configured expected roll window.
- `apiserver_static_crd_controller_frozen{group, resource, reason}` (gauge) — reason in
  `revision_tie|terminal_conflict|terminating|storedversions_blocked|names_divergent`.
  `revision_tie` pages: it is a misconfiguration whose log line carries its own remedy.
- `apiserver_static_crd_denied_writes_total{group, resource, reason}` — reason in
  `spec_divergent|delete|create_reserved_name|reserved_key|status_shape`. Doubles as the
  pre-demotion handoff gate: `reason="spec_divergent"` flat at zero fleet-wide is the evidence
  that releasing the CRD is safe (see Transitions).

Storage binding:
- `apiserver_custom_resource_storage_binding{group, resource, state, storage_cluster_id}`
  (gauge) — `state` in `default|override|mismatch|refused`. Set to 1 for the current state;
  stale state series are deleted, not left at 0.
- `apiserver_custom_resource_storage_precheck_total{group, resource, result}` — result in
  `ok|refused-nonempty|refused-unreachable|acknowledged`. The most important safety event in
  the feature must be countable, not log-only.
- `apiserver_custom_resource_storage_binding_rejected_total{group, resource}` — guard 503s
  (the skew signal, distinct from generic request errors).
- `apiserver_static_crd_storage_warmup_duration_seconds` plus a completion gauge.

#### Audit Annotations

- Requests served for a custom resource of a static CRD carry
  `staticcrd.apiextensions.k8s.io/registration: "static"` (constant string; no per-request
  allocation).
- Protection denials carry `staticcrd.apiextensions.k8s.io/protected-write-rejected: "<name>"`.
- Guard denials carry `staticcrd.apiextensions.k8s.io/storage-binding-mismatch: "<name>"`.
- A break-glass manual demotion (the marker-strip write permitted when no local manifest
  claims the name) carries `staticcrd.apiextensions.k8s.io/manual-demotion: "<name>"`, so
  fleets can detect out-of-band demotions; the documented flow is tombstone-only.
- Definition-change provenance moves from CRD-object audit events to file provenance on the
  control-plane host — the same trust and audit model as static pods and KEP-5793 manifests —
  *plus* the reconciler's loopback writes, which do produce ordinary audit records of every
  materialization.

### API Changes

Declared explicitly, because parts of this land in GA types regardless of gate state:

1. **New `CustomResourceDefinitionConditionType` value `StaticSynced`** in
   `apiextensions.k8s.io/v1`. Appears in the GA constant set and generated clients
   unconditionally (precedent: KEP-4192 added `StorageMigrating`); only ever *set* on static
   CRDs; nothing in-tree gates behavior on a client understanding it. Reasons include
   `ReconcileSucceeded`, `TerminalConflict`, `StoredVersionsBlocked`, `RevisionTie`,
   `ManifestRemovedWithoutTombstone`, `Terminating`. The condition is removed at demotion (a
   future owner inherits clean conditions).
2. **Reserved metadata keys**: label `apiextensions.kubernetes.io/static`, annotations
   `apiextensions.kubernetes.io/static-content-hash`,
   `apiextensions.kubernetes.io/static-revision`,
   `apiextensions.kubernetes.io/static-superseded`,
   `storage.apiextensions.k8s.io/etcd-servers`. Every reserved value is a hash, a fixed token,
   or an operator-authored ordinal; none carries admin-environment detail (see Design Rule: No
   Admin-Environment Detail on API Objects). Reserved-prefix enforcement is via the admission
   plugin's server-ownership rules (gate-conditioned as described), not schema validation.
3. **New HTTP failure shape**: the guard's 503 with `reason: StorageBindingMismatch` and a
   structured message (CRD name, recorded value, computed value, remediation command). The
   reason string is API surface; integration tests pin it. The protection denials' stable
   sentinel `StaticCustomResourceDefinition:` is likewise pinned (see Denial Messages).
4. **Two new kinds in `apiserver.config.k8s.io/v1alpha1`**, loaded only from
   `staticManifestsDir`, never served: `CustomResourceManifestSet` (`revision` int64 required,
   `supersedes` []int64 optional) and `CustomResourceDefinitionTombstone` (`crdName`, `action`
   with sole value `Demote`). Codegen, validation, and round-trip fuzz costs join the
   `CustomResourceConfiguration` line item.
5. No new REST resources, no new served API groups, no changes to the CRD spec schema.

```
<<[UNRESOLVED lifecycle naming bikeshed @jpbetz, then API review ]>>
Names carried as proposed, flagged for a deliberate pass before implementable: (a)
`CustomResourceDefinitionTombstone` — "tombstone" is jpbetz's term and matches the
Cassandra-style fencing semantics, but the kind is read in platform-repo PRs by non-experts,
and "tombstone" connotes deletion, which is the one thing the mechanism never does; softer
candidates: `...Release`, `...Retirement`. (b) `crdName` vs `name`. (c) Keeping `action` with a
single allowed value at alpha (proposed: yes — self-documenting diffs, fail-closed forward
compatibility). (d) `supersedes` vs a direction-explicit `supersedeRecordedRevisions` (a bare
integer list reads ambiguously as "revisions that supersede me"). (e) Whether `supersedes`
ships at alpha at all — proposed: yes, because the hijacked-huge-revision case has no other
declarative exit (one cannot out-bump MaxInt64), and the on-object mirror is what makes
recovery stick; deferring it would leave that recovery an API poke. (f) Whether a demoted
object should carry an explicit self-describing state token in addition to the
label-absent+record-present encoding.
<<[/UNRESOLVED]>>
```

### Test Plan

[x] I/we understand the owners of the involved components may require updates to
existing tests to make this code solid enough prior to committing the changes necessary
to implement this enhancement.

##### Prerequisite testing updates

- The apiextensions options package currently has **zero tests** pinning the
  `--watch-cache-sizes` interaction with custom resources (which is how the dead-code leak
  survived ~10 releases). The prerequisite fix lands with tests pinning both the restored
  restriction for dynamic CRDs and (under the gate) the static-CRD behavior.
- An integration test pinning the CR etcd key prefix (`/registry/<group>/<plural>/...`) for
  static CRDs, guarding against any storage-factory substitution relocating data.
- `test/integration/apiserver/watchcache_test.go`'s `multiEtcdSetup` (two fresh etcds plus
  `--etcd-servers-overrides` before first start) is the existing harness the storage tests
  build on.

##### Unit tests

New packages, targeting the standard coverage bar for new apiserver code:

- `apiextensions-apiserver/pkg/staticcrd/loader`: decode/restrict/synthesize/validate paths,
  including status-present, webhook-conversion, protected-group, duplicate-name,
  overlapping-builtin, gate-stripped-field, missing/duplicate `CustomResourceManifestSet`,
  live-CRD-plus-tombstone, malformed-tombstone, and vacuous-`supersedes` (warn, not fatal)
  cases; Tier-1 vs Tier-2 classification.
- Arbitration: `rank()` over the full truth table (WINS/TIE/LOSES, supersedes in both
  directions, mutual disown, absent record); sticky-supersession filtering; the behind guard's
  scoping (labeled, asserted, demoted-unasserted, planted-unlabeled objects; unsynced lister
  fails closed); the TIE tamper split (record hash vs object hash vs local hash, all eight
  combinations).
- Canonicalization: a regression test with a `default:` map whose YAML keys are deliberately
  out of alphabetical order, asserting zero reconciler writes (the `apiextensionsv1.JSON` raw
  bytes trap); a manifest omitting `names.singular`/`listKind`/`conversion`, asserting zero
  writes; a v1beta1 storage round-trip test (CRDs are stored as v1beta1); a differential fuzz
  test asserting `pluginWouldAllow(spec) == reconcilerConsidersConverged(spec)` over generated
  specs.
- `apiextensions-apiserver/pkg/staticcrd/reconciler`: create/adopt/confirm/repair/demote/
  no-op/terminal-conflict/Terminating-refusal/rate-limit-backstop paths against a fake client;
  ownerReference severance at adoption; lost-race (`AlreadyExists`, 409) requeue-and-
  re-arbitrate behavior.
- The protection plugin: every row of the Write Protection table, both subresources, SSA and
  non-SSA identical-apply, PUT-shaped identical writes (reserved-key restoration), the unlock
  precedence over restoration, the status shape rules (naming-controller-shaped accepted-names
  writes pass, squats fail; SVM-trim-shaped storedVersions writes pass, clobbers fail), the
  cleanup-finalizer carve-out, the loopback shape checks (including that the demote shape is
  refused without a local tombstone), and the server-owned reserved-key rules.
- `staticCRDStorageFactory`: ServerList-only override, prefix invariance, `Configs()` contents,
  transformer composition.
- Guard: all four witness/binding states; canonicalization golden tests for the witness hash.
- Flag cross-validation and grammar diagnostics.

##### Integration tests

Named as alpha graduation criteria, because every interesting case here is a *transition* with
data present (the class of test KEP-5793 did not need and this KEP cannot skip):

1. Adopt a CRD that already has objects, with an override configured: assert refusal, no
   witness, no storage against the override backend, data intact and served from the default
   backend, readyz green, precheck metric.
2. Bind a fresh CRD to etcd-B, write objects, then disable each gate in turn and restart:
   assert the resource still serves from etcd-B (un-gated honoring), and that no new binding
   can be created.
3. Simulated skew: two apiservers sharing the default etcd, one with the override flag and one
   without; assert the flagless member 503s exactly that resource with
   `reason: StorageBindingMismatch` while serving everything else, and that its live storage is
   invalidated when the witness appears.
4. Reload with the manifest directory half-projected (CRDs present, `CustomResourceManifestSet`
   missing): assert rejection, last-known-good retention, protection retained, failure metric.
   Reload with the directory *entirely* emptied: assert acceptance, zero writes, protection
   retained, warning; content restored: reconciliation resumes.
5. Upgrade -> downgrade -> upgrade with data in etcd-B, exercising the documented pre-downgrade
   drain and asserting the guard re-arms on re-upgrade.
6. Full protection matrix over the wire, including `kubectl delete crd --all` (DELETECOLLECTION
   path), status-subresource metadata denial, identical-spec apply success, and divergent-apply
   denial message content.
7. storedVersions lifecycle: v1 -> v2 bump with SVM trim, manifest drop rejected before trim
   and accepted after, no restarts.
8. Watch-cache disablement end-to-end: decorator is undecorated for the named GR, cacher for a
   control GR, APF seat correction observed for the uncached GR only.
9. Startup with an unreachable override backend: fail-closed precheck, bounded warm-up, no
   cross-CRD first-request stall beyond the budget.
10. Revision arbitration under simulated skew (two apiservers, shared etcd, different loaded
    configurations): a staggered spec change produces exactly one spec write fleet-wide; a
    revert-without-bump leaves both members inert with `config_behind` set and a
    distinguishable log; equal-revision-different-content freezes (no write, `frozen`,
    guarded condition) rather than writing or tie-breaking; a `supersedes` recovery converges
    with one write and the mirrored record silences the member still running the superseded
    tree; the fork case lands whichever content wins the race and freezes the loser — the
    test asserts the actual contained outcome, not a prettier one.
11. Staggered demotion: tombstone loaded on one of two members — one metadata-only demote
    write, no re-promotion by the behind member, DELETE allowed uniformly through both members
    afterward, zero storage teardown observed; then revert of the tombstone at a higher
    revision re-promotes.
12. Resurrection fencing: demote + API-delete on one member while a second member (stale,
    pre-tombstone configuration) is running with another labeled record present — assert no
    recreate; then with *every* record removed, assert the loud recreate path (documented
    residual) including its metrics.
13. Tamper repair: edit a static CRD's spec with the gate off, re-enable, assert the
    reconciler repairs to the manifest (`action="repair"`) rather than freezing on the tie;
    and assert the witness-unlock write on a 503ing resource is honored, not restored away.
14. Mid-roll admission uniformity: with two members at different loaded revisions, `kubectl
    apply` of the object's current content succeeds through both; apply of the older content
    fails through both; the denial messages match the pinned texts (no file names).
15. Gate-off cascade completion: delete a labeled CRD with the gate off, re-enable mid-
    Terminating, assert the finalizer completes via the status carve-out (no wedge) and the
    reconciler refuses to recreate through the cascade in-process.
16. Behind-guard cold start: boot a member with an ancient manifest set against a cluster
    whose only evidence is one newer labeled record; assert zero writes, including during the
    post-start window before informer sync.
17. Renumbered configuration versus demotion residue: with only a *demoted* object recording
    revision 100, a fresh configuration at revision 1 still creates a newly added CRD (the
    global floor excludes demoted-unasserted objects) while re-promotion of the demoted name
    itself defers until outranked.

##### e2e tests

Not applicable for the same reason as KEP-5793: the feature requires apiserver startup flags
and on-disk control-plane files that e2e suites (running against externally configured
clusters) cannot manipulate. Integration tests above carry the load, including the multi-etcd
and multi-apiserver cases.

### Graduation Criteria

#### Alpha

- Both gates implemented, default off; storage gate requires registration gate.
- Prerequisite watch-cache leak fix merged (with its opt-out window) in the release before or
  the same release as alpha.
- Loader (all three kinds, two-tier validation), revision arbitration (rank, behind guard,
  tamper split, debounce), tombstone execution, reconciler, protection plugin (including
  server-owned reserved keys and the status shape rules), witness, pre-activation check,
  per-request guard, bounded warm-up, and all listed metrics implemented. Arbitration ships in
  the first alpha, so no force-update-on-divergence binary ever exists.
- Integration tests 1-17 above passing.
- The APF list-cost correction, scoped to static watch-cache-disabled GroupResources.
- Documentation: manifest format, the revision model (including the rollback runbook: revert
  plus bump; `supersedes` recovery; the emergency hand-edit two-step), the tombstone flow and
  its alpha permanence, the transition protocol with both pre-flights, the two-flag model, the
  migration runbook (both directions), the pre-downgrade drain procedure, and the per-tool
  opt-out table for API-side owners.

#### Beta

- All alpha UNRESOLVED blocks resolved; in particular the gate-semantics PRR ruling, the
  DELETE-denial ruling, and the witness value/location decision (stable backend identity vs
  hash; annotations vs status stanza), with migration for any change to the witness contract.
- Measured numbers published (see Scalability): bytes per cached unstructured CR object;
  cacher init time and etcd load for a 100k-object CRD; the cost of the periodic cache
  consistency check on such a CRD; rolling-restart relist cost against an uncached
  100k-object CRD with a realistic watcher count. No memory-benefit claims before these exist.
- Stale flag entries (matching neither built-ins nor static CRDs) become fatal at startup
  (this also mechanically enforces the drain-first decommissioning order for bound CRDs).
- The identity-Lease revision stamp (each member publishes its manifest-set revision on its
  apiserver identity Lease via the existing `ProcessLeaseFunc` hook), resolving its UNRESOLVED
  block — which unlocks: an API-visible fleet-convergence answer
  (`kubectl get lease -n kube-system -l apiserver.kubernetes.io/identity=kube-apiserver`),
  tombstone garbage collection (removable one rollout after every live member's lease reports
  a revision at or past the tombstone's, with a live labeled record remaining as a floor
  backstop), and a CREATE floor that closes the last resurrection residual. Unified with the
  storage-config-hash Lease candidate below.
- `storageVersionMigration: Automatic` opt-in on `CustomResourceConfiguration` (the SVM
  controller is GA/default-on since v1.37): the reconciler creates the StorageVersionMigration
  object for a blocked version drop; a stalled or failed migration surfaces as
  `StaticSynced=False`, never a retry storm; decide whether the reconciler GCs its completed
  SVM objects.
- Candidate hardening, decided by then: URL-only conversion webhooks with static CA bundles;
  per-override etcd TLS; an acknowledgement gate for schema-*narrowing* reloads (read-path
  pruning makes narrowing a silent-data-invisibility hazard — see Risks); the dynamic-CRD
  storedVersions status hardening (only after a declarative version-drop equivalent exists
  there); cross-member binding agreement via apiserver-identity Leases (refuse to activate a
  binding until all live members publish the same storage-config hash — turns the
  rolling-enable 503 window into a positively enforced invariant, at the cost of a startup
  dependency on the coordination API); `DeleteIfEmpty` tombstone action if users demand
  single-artifact retirement.
- At least one non-author production user exercising an override binding.

#### GA

- Two production users (the Gardener and Argo communities are the named candidates from the
  motivating issues) providing feedback across two releases.
- Conformance-adjacent guarantees pinned: key-prefix invariance, guard response shape,
  protection matrix.
- Shared-etcd-client-per-transport (kubernetes/kubernetes#111476) evaluated for static CRDs
  (their fixed version set makes it tractable); adopted or explicitly declined with data.
- Gate lock and removal per policy.

### Upgrade / Downgrade Strategy

Upgrade:
- Enabling is opt-in twice over (gate plus flag/manifests). Clusters not opting in see no
  change other than the prerequisite watch-cache fix (which carries its own opt-out and
  release note).
- Recommended enablement order in HA: enable `StaticCustomResourceDefinitions` fleet-wide with
  an empty manifest directory first (an empty directory is valid and asserts nothing;
  protection is a fleet property: during the enablement roll, un-upgraded members accept
  writes that upgraded members deny — including reserved-key plants, which is why the gate
  goes on before any record is trusted); verify `..._last_config_info` agreement; then
  distribute manifests (with their `CustomResourceManifestSet`, starting at revision 1 or any
  small integer); then, if desired, enable the storage gate and add flag entries, completing
  the roll promptly (the guard 503s the affected resource on unrolled members for the
  duration). Never combine a manifest change and a storage-flag change in one commit, and do
  not ship a new revision while a restart-only rollout is incomplete (Notes item 9).

Downgrade / rollback:
- Disabling `StaticCustomResourceDefinitions` is clean: objects persist as ordinary CRDs.
- Disabling `CustomResourceStorageOverrides` stops new bindings; recorded bindings keep
  serving (see Feature Gates). Full unbinding is the documented drain.
- **Before downgrading to a binary that predates this feature**, with any live binding:
  migrate all bound custom resource data back to the default backend, clear the witnesses,
  drop the flag entries, and verify
  `apiserver_custom_resource_storage_binding{state="override"} == 0` on every member. A
  pre-feature binary ignores the witness and will silently serve the empty default backend
  for a still-bound resource — LIST returns 200 with zero items, and controllers will act on
  that. This is the single worst failure mode in the design and it is a procedural
  requirement, not a code guarantee.
- Removing the new flag before downgrade is required (unknown flag = startup failure), same
  as any flag.

### Version Skew Strategy

Control-plane-internal; kubelet, scheduler, and controller-manager interact only with the
ordinary CRD API objects and discovery, so node/component skew is unaffected.

Within an HA control plane, skew is the central design concern and is engineered rather than
waved at:

- The premise, stated first because every skew conclusion follows from it: **a manifest change
  is not staggered in effect.** Members serve the shared object (Invariant A), so the first
  rolled member's write changes serving fleet-wide; what *is* per-member during a roll is only
  the set of loaded assertions and revisions — which the arbitration turns into "one member
  writes once, the rest are inert" rather than into per-member serving differences.
- Registration skew (a member without the manifest, mid-roll): the object exists in etcd, so
  the member serves the type normally (it is just an unclaimed, label-protected CRD to it).
  No APIService fights, no discovery flapping — the specific failure modes of the invisible
  alternative, which this design was chosen to avoid.
- Revision skew (the normal state of every staggered roll): older-configured members are
  *behind* — serving the new content, ready, protecting, writing nothing — for the duration of
  the roll, which may legitimately be days (canary soaks, paused rollouts). No mechanism in
  this KEP depends on that window being short. Divergent content at *equal* revisions is a
  misconfiguration and freezes loudly instead of fighting; the per-object rate limit survives
  only as a backstop for a member left running a retired or hijacked configuration. Because
  arbitration ships in the first alpha binary, there is no mixed-arbitration binary skew to
  manage.
- Storage-binding skew (divergent flags, or a partial storage-gate roll): the disagreeing
  member 503s exactly the affected resource via the guard, with its live storage invalidated;
  everything else serves. The `..._last_storage_config_info` hash is the drift alert. Identical
  storage flags across members are a hard requirement; the design contains violations, it does
  not tolerate them silently.
- Peer proxy (KEP-4020 UVIP): unchanged — the GV exclusion manager consumes the CRD informer,
  which sees the real object on every member, so the asymmetric proxying that an invisible
  design would induce does not arise.
- A member running a binary older than this feature ignores witnesses entirely: see the
  pre-downgrade requirement above. Skewed *upgrades* (new binary, feature off) are safe: the
  un-gated honoring surface only routes when a witness and a matching flag entry exist, which
  cannot occur before the feature was ever enabled.

## Production Readiness Review Questionnaire

### Feature Enablement and Rollback

###### How can this feature be enabled / disabled in a live cluster?

- [x] Feature gate (also fill in values in `kep.yaml`)
  - Feature gate name: `StaticCustomResourceDefinitions`
  - Components depending on the feature gate: `kube-apiserver`
  - Feature gate name: `CustomResourceStorageOverrides`
  - Components depending on the feature gate: `kube-apiserver`
- [x] Other
  - Mechanism: `--custom-resource-config` naming a `CustomResourceConfiguration` whose
    `staticManifestsDir` names a directory of CRD manifests; existing
    `--etcd-servers-overrides` / `--watch-cache-sizes` entries naming static CRDs.
  - Enabling/disabling requires API server restart. No node impact.

###### Does enabling the feature change any default behavior?

`StaticCustomResourceDefinitions`: No, until a manifest directory is configured. With
manifests: the named CRD objects become apiserver-owned and write-protected (including against
cluster-admin deletion — an explicit, documented posture change for those objects), and
attempts to create dynamic CRDs at claimed names are denied.

`CustomResourceStorageOverrides`: No, until a flag entry matches a static CRD. Independently
of either gate, the prerequisite watch-cache fix changes the (accidental, undocumented)
behavior of `--watch-cache-sizes` for dynamic CRDs, with its own opt-out and release note.

###### Can the feature be disabled once it has been enabled (i.e. can we roll back the enablement)?

`StaticCustomResourceDefinitions`: Yes, cleanly. Objects persist as ordinary, fully mutable
CRDs; no data or discovery changes.

`CustomResourceStorageOverrides`: Yes for the feature's *growth* (no new bindings). An
already-recorded binding is deliberately still honored after gate-off, because reverting
routing would serve an empty backend and cause data-destroying controller reactions; fully
unbinding a resource is the documented drain procedure (migrate data back, clear witness,
drop flag). This layered answer, and the small un-gated guard/honoring surface behind it, is
flagged for explicit PRR review in the Feature Gates section's UNRESOLVED block.

###### What happens if we reenable the feature if it was previously rolled back?

Registration: the reconciler re-adopts the (still present) objects; idempotent. Storage: the
pre-activation check re-runs; a binding whose witness and flag still agree resumes; one whose
state changed while disabled is refused or guarded exactly as at first enablement. Re-enable
is tested (integration test 2 and 5).

###### Are there any tests for feature enablement/disablement?

Yes; integration tests 2 and 5 in the Test Plan cover gate-off-with-data, re-enable, and the
upgrade -> downgrade -> upgrade path with an override backend populated.

### Rollout, Upgrade and Rollback Planning

###### How can a rollout or rollback fail? Can it impact already running workloads?

Rollout failures:
- Invalid manifests fail API server startup (before the listener opens; Tier-2 content errors
  are fatal only when no existing object bears the name — see Loading); a member crash-looping
  on a bad manifest is the loud, intended failure, and it halts every health-gated rollout
  mechanism at one node.
- A content change shipped without a revision bump freezes that CRD's reconciliation loudly
  on the first rolled member (`frozen{reason="revision_tie"}`, condition, log with the
  remedy); a configuration restored wholesale by rollback tooling goes inert
  (`config_behind`) rather than writing backwards. Neither affects serving or readiness.
- A partial storage-gate roll 503s the bound resources on disagreeing members (by design);
  behind a load balancer this presents as intermittent failures for those resources
  proportional to the unrolled fraction, and namespaces containing those resources will not
  finish deleting during the window. Complete the roll promptly.
- An unreachable override backend at startup: precheck fails closed for that resource; the
  rest of the apiserver is unaffected (bounded warm-up, no global readyz coupling).

Rollback failures:
- Downgrading past the feature with a live binding silently serves an empty backend for that
  resource (the worst case; procedural pre-downgrade drain is mandatory — see
  Upgrade / Downgrade Strategy).
- Removing gates/flags without draining leaves bindings honored (safe) or guarded-503
  (loud); neither destroys data.

Already-running workloads are unaffected except through API availability of the specific
bound resources as described.

###### What specific metrics should inform a rollback?

- `apiserver_static_crd_controller_automatic_reloads_total{status="failure"}` increasing.
- `apiserver_static_crd_unsynced > 0` persisting (reconcile conflicts).
- `apiserver_static_crd_controller_frozen > 0` — a revision tie or terminal state whose log
  line names its own remedy; `reason="revision_tie"` pages.
- `apiserver_static_crd_controller_config_behind == 1` sustained beyond the expected roll
  duration on any member (a stuck or reverted-without-bump configuration), or on *every*
  member (the bare-revert signature — the printed remedy is a bump or `supersedes`).
- Spread in `apiserver_static_crd_controller_manifest_revision` across members after a roll
  should have converged.
- `apiserver_custom_resource_storage_binding{state="mismatch"}` nonzero after a roll should
  have converged (skew not resolving).
- `apiserver_custom_resource_storage_binding_rejected_total` rate (guard 503s).
- `apiserver_custom_resource_storage_precheck_total{result=~"refused.*"}` at startup.
- Divergence across members of `..._last_storage_config_info{hash}`.

###### Were upgrade and rollback tested? Was the upgrade->downgrade->upgrade path tested?

Integration test 5 covers upgrade -> downgrade -> upgrade with data present in an override
backend, including the drain procedure; gate-flip tests cover enablement transitions. These
are alpha graduation criteria, not aspirations.

###### Is the rollout accompanied by any deprecations and/or removals of features, APIs, fields of API types, flags, etc.?

Yes, one: the accidental applicability of `--watch-cache-sizes` to dynamically created CRDs
(live since ~v1.28 via dead code) is removed by the prerequisite fix, with an opt-out window,
startup logging of affected entries, and a release note, per the KEP-5793
`ExcludeAdmissionWebhookVirtualResources` precedent. Both storage flags' help text is
corrected. No other deprecations.

### Monitoring Requirements

###### How can an operator determine if the feature is in use by workloads?

- `apiserver_static_crd_registered > 0`; `apiserver_custom_resource_storage_binding` with
  `state="override"` for the storage half.
- The CRD objects themselves: `kubectl get crd -l apiextensions.kubernetes.io/static=true`.
  (Deliberately API-visible, unlike KEP-5793.)

###### How can someone using this feature know that it is working for their instance?

- [x] Metrics
  - `apiserver_static_crd_controller_automatic_reloads_total{status="success"}` advancing on
    reloads; `..._last_reconcile_success_timestamp_seconds` recent per CRD;
    `apiserver_static_crd_unsynced == 0`.
  - `apiserver_custom_resource_storage_binding{state="override"}` for the bound resource, and
    `apiserver_storage_size_bytes` for the override `storage_cluster_id` growing as data is
    written (after the monitor-clobbering fix).
- [x] API .status
  - `StaticSynced=True` condition on each static CRD; `Established=True` as usual; the
    `static-revision` annotation showing the intended revision in effect ("is revision N
    live?" is answerable from any single member, because serving state is the shared object).
- [x] Other (treat as last resort)
  - Startup log lines echoing each storage-flag entry's resolution and the effective
    manifest-set revision.

###### What are the reasonable SLOs (Service Level Objectives) for the enhancement?

- Startup: added time linear in manifest count; budget <= 50ms per typical CRD for
  decode+validate (CEL-heavy schemas dominate; measured, not assumed), plus one loopback
  write per divergent object, plus one etcd Count RPC per override candidate. Target: < 1s
  added for 20 static CRDs with no bindings; < 2s with bindings against healthy backends.
- Warm-up: does not gate readiness; per-CRD deadline (default ~30s) bounds background work.
- Steady state: no change to CR request latency for unbound resources; bound resources meet
  the same SLOs as the default backend given an equivalently provisioned override backend.
- Reload detection: within the 1-minute polling fallback worst case.

###### What are the SLIs (Service Level Indicators) an operator can use to determine the health of the service?

- [x] Metrics
  - `apiserver_static_crd_unsynced` (should be 0), reload failure rate (should be 0),
    `binding{state="mismatch"}` (should be 0 outside rollout windows), guard rejection rate,
    per-member storage-config-hash agreement.

###### Are there any missing metrics that would be useful to have to improve observability of this feature?

Per-file load status (deferred, cardinality); a served-from-override request counter
(deferred: derivable from existing per-resource request metrics joined with the binding
gauge).

### Dependencies

###### Does this feature depend on any specific services running in the cluster?

No cluster services. The registration half depends on the default etcd (like every CRD). The
storage half adds a dependency, per bound resource, on the operator-provisioned override etcd
cluster: that backend's availability governs *that resource's* availability only — it is
deliberately excluded from the apiserver's global readiness (see Health and Readiness for
Override Backends), so an override backend outage does not evict the apiserver from the
load-balancer rotation. Operators own provisioning, TLS (inherited from the global `--etcd-*`
flags at alpha), backup, and compaction sizing of override backends; the apiserver runs its
standard compactor against them.

### Scalability

###### Will enabling / using this feature result in any new API calls?

Loopback only: per revision, exactly one spec write per *changed* CRD fleet-wide plus at most
one annotation-only confirm per unchanged CRD (the arbitration guarantees the once-ness; the
per-object rate limit is a backstop, unused in correctly operated fleets); zero in steady
state. One etcd Count RPC per override candidate per startup. No new steady-state API traffic.

###### Will enabling / using this feature result in introducing new API types?

No new REST resources. One new condition-type constant on the existing GA type; reserved
label/annotation keys (see API Changes).

###### Will enabling / using this feature result in any new calls to the cloud provider?

No.

###### Will enabling / using this feature result in increasing size or count of the existing API objects?

One CustomResourceDefinition object per static CRD (which would typically have existed as a
dynamic CRD anyway), plus ~5 annotations and 1 label on each (~300 bytes; the revision record
adds two small integer-valued annotations).

###### Will enabling / using this feature result in increasing time taken by any operations covered by existing SLIs/SLOs?

Startup, per the SLO section (bounded, measured). Request path: one map/annotation comparison
per CR request (the guard — no I/O; the witness comes from the already-consulted lister
object). For watch-cache-disabled resources, the *documented and intended* latency profile
change: LISTs and GETs at etcd cost, writes plus one etcd read, watches per-connection etcd
streams. These are per-resource, operator-elected, and confined to the named resource.

###### Will enabling / using this feature result in non-negligible increase of resource usage (CPU, RAM, disk, IO, ...) in any components?

Stated honestly, since this section is where the storage half earns or loses its keep:

- An override binding costs, **per served-or-storage version** of the CRD (not per CRD): one
  dedicated etcd client (clients are not shared across storages —
  kubernetes/kubernetes#111476), one cacher independently list+watching the same key range,
  and one count poller; plus, per distinct backend, one compactor client and the flag's
  health-prober clients. A 3-version CRD on a dedicated backend is 3 clients + 3 cachers + 3
  pollers. The KEP does not promise arbitrary numbers of override backends; the intended
  envelope is a handful of statically known, high-value resources.
- While `DetectCacheInconsistency` remains enabled (beta default-on), each *cached* resource —
  including one on an override backend — incurs a full paginated list-and-digest of that
  resource roughly every 5 minutes against its backend. For a huge CRD this is a standing
  load on the dedicated etcd and is part of why an operator might instead disable the cache.
- Disabling the watch cache *saves* the decoded object store and event ring on every
  apiserver, and *costs* per-write etcd reads and per-watcher etcd watch streams and decode
  work. Which side wins depends entirely on the workload shape (see Story 3); the beta
  criteria require published measurements (bytes per cached unstructured object, 100k-object
  init and steady-state costs, relist storms on rolling restarts) before any memory-benefit
  claim is made.

###### Can enabling / using this feature result in resource exhaustion of some node resources (PIDs, sockets, inodes, etc.)?

Two fsnotify watches — one on the `CustomResourceConfiguration` file, one on the
`staticManifestsDir` it names (inotify limits noted, as in KEP-5793). Override
backends add a bounded number of gRPC connections (clients enumerated above). Watch-cache
disablement moves watch fan-out from in-memory to per-watcher etcd streams: a
watch-heavy uncached resource multiplies sockets and goroutines toward the override backend —
part of the documented envelope guidance, bounded by APF like all watches.

### Troubleshooting

###### How does this feature react if the API server and/or etcd is unavailable?

- Default etcd unavailable at startup: server starts, reconciler retries, `static-crds`
  readyz unready; CR serving is down anyway (data is in etcd) — this feature adds no new
  etcd-availability claims and explicitly disclaims KEP-5793's etcd-independence motivation.
- An override backend unavailable: only its resources fail (request errors; precheck
  fail-closed at startup; named healthz check red); global readiness unaffected.

###### What are other known failure modes?

| Failure Mode | Detection | Mitigation | Diagnostics |
|--------------|-----------|------------|-------------|
| Invalid manifest at startup | API server fails to start | Fix manifest; restart | Startup log names file and error |
| Invalid manifest on reload | Reload failure metric; logs | Fix manifest (poison-pill: identical bad content is not retried); last-known-good serves | Log names file and error |
| Manifest edit rejected server-side (e.g. storedVersions drop) | `StaticSynced=False`; `apiserver_static_crd_unsynced > 0` | Run SVM / revert edit (with a bump) | Condition reason names the exact SVM object; reconcile conflict metric |
| Pre-existing conflicting CRD object | Same as above; adoption refused | Operator reconciles the two specs (the object stays editable until adopted) or retires the manifest via tombstone | Condition reason names the divergence |
| Content changed, revision not raised (or forked equal revisions) | `frozen{reason="revision_tie"}`; guarded `StaticSynced=False, reason: RevisionTie` | Raise `revision` above the recorded value (log prints it); for a fork, first decide which branch was right | Log names both content hashes and (for a fork) the winning member's id hash |
| Configuration behind the record (stalled roll, bare revert, restored old config) | `config_behind == 1`; on every member simultaneously = the bare-revert signature | Complete the roll; or raise `revision` past the record; or ship `supersedes: [<recorded>]` | Log prints the blocking object and the literal minimum winning revision |
| Emergency hand-edit needed on one member | n/a (operator-initiated) | Edit the manifest AND raise `revision` above the recorded value printed in the behind/frozen log; afterwards reconcile git (revert-plus-bump, or `supersedes` if an oversized emergency value was used) | The two-step is documented; every relevant log line prints the minimum winning value so no API read is required |
| CRD file deleted without a tombstone | `StaticSynced=False, reason: ManifestRemovedWithoutTombstone` on the object | Ship a tombstone (demotion intended) or restore the file (deletion accidental) | The condition — visible in `kubectl get crd` — names the tombstone as the supported removal expression |
| GitOps Application wedged deleting (finalizer prune vs DELETE denial) | Application stuck `Deleting`; `denied_writes_total{reason="delete"}` | Annotate the CRD `Prune=false` in the source (prevention: promotion pre-flight); remove the finalizer by hand (cure) | Documented per-tool consequence of unconditional DELETE denial |
| Guard 503 (`StorageBindingMismatch`) on one resource | `binding{state="mismatch"}`; `binding_rejected_total`; clients see 503 with reason | Complete the roll; or follow the migration runbook; or run the printed unlock command | The 503 message contains both binding values and the remediation |
| Intermittent per-resource failures behind LB during a storage roll | Same metrics, member-scoped | Complete the roll | Compare `..._last_storage_config_info{hash}` across members |
| Namespaces stuck Terminating during a mismatch window | Namespace controller retry logs; mismatch metrics | Resolve the mismatch (above) | The stuck namespace contains (or could contain) the guarded resource |
| Override backend down | Named `etcd-override-N` healthz red; request errors for that resource only | Restore backend | Global readyz unaffected by design |
| Empty/partial manifest directory (broken mount) | Partial content (CRDs without the manifest-set document, or any Tier-1 failure): reload rejected / startup fatal. Entirely empty: startup or reload warning when labeled objects exist | Fix the mount; last-known-good or label protection carries through; absence never demotes | Log distinguishes empty-dir-with-labeled-objects from structurally invalid content |
| Custom resource LIST returns empty after a control-plane **downgrade** | Workload symptoms; no metric (old binary) | Re-upgrade, or restore data per runbook; **prevent via the mandatory pre-downgrade drain** | Check the CRD object's witness annotation against the running binary's feature support |
| Warm-up exhausting its deadline (slow backend) | Warm-up metrics | Storage falls back to lazy construction; investigate backend | First-touch latency on that resource |

###### What steps should be taken if SLOs are not being met to determine the problem?

1. `apiserver_static_crd_unsynced` and reload failure metrics; then the named conditions on
   the affected CRDs.
2. `apiserver_custom_resource_storage_binding` states and cross-member
   `..._last_storage_config_info` hashes for skew.
3. Startup log lines echoing storage-flag resolution (typos surface here).
4. For bound-resource latency: the override backend's own health/size metrics
   (`apiserver_storage_size_bytes` by `storage_cluster_id`) and the per-version client/cacher
   multiplication in Scalability.
5. Last resort: demote (registration) or drain (storage) per the documented procedures —
   both are non-destructive by construction.

## Implementation History

- 2026-08-24: KEP created in `provisional` status. KEP number is TBD pending the enhancement
  tracking issue (this directory is named with the `NNNN` placeholder until then). Proposed
  as the consolidation of kubernetes/kubernetes#118858 (and duplicates #81508, #139817), which
  should be adopted as the tracking issue's origin, with the reporters (@geetasg, @unmarshall,
  @xigang) CC'd.

## Drawbacks

1. Two more feature gates and a new flag on an already flag-heavy binary, plus a protection
   posture (undeletable-by-cluster-admin API objects) novel for CRDs.
2. The witness/guard machinery adds a per-request comparison and a new failure mode (503 on
   mismatch) whose *purpose* is to convert silent data hazards into loud unavailability — a
   trade some operators will experience as new brittleness during rollouts.
3. The endpoint-rotation wart: rotating an override backend's URLs requires an explicit
   acknowledgement despite no data moving (until a stable backend identity exists).
4. Provenance markers in metadata produce GitOps diff noise on the CRD object (tracked for a
   possible status-stanza move at beta).
5. The two storage flags' grammars remain inconsistent; the KEP compensates with diagnostics
   rather than fixing the inconsistency.
6. Protection is only as strong as the fleet's weakest member during enablement rolls.
7. The un-gated honoring surface (guard + recorded-binding routing) deviates from clean
   feature-gate semantics, deliberately, for data safety.
8. One integer the admin must raise per deployed change — the revision — enforced by a freeze
   that names its own fix; and rollback tooling that restores configurations wholesale must be
   paired with a forward-only revision source. This is the honest cost of eliminating the
   multi-writer fight; the alternative was indefinite bounded flapping with no fix to name.
9. Tombstones are permanent manifest content at alpha (~4 lines of YAML per retired type);
   garbage collection arrives at beta with the lease-based convergence signal.
10. A manifest change cannot be canaried per node: serving state is the shared object, so the
    first member's write is fleet-wide. Validation happens before rollout, not during it.

## Alternatives

### Fully Invisible Static CRDs (the KEP-5793 Model)

Port 5793's design: synthetic CRDs merged into the informer/lister layer, invisible to the
API. Rejected on four independent grounds, any one sufficient:

1. `status.storedVersions` has no durable home, silently re-enabling the version-drop data
   loss that validation prevents today; the in-tree storage version migrator additionally
   degrades to a silent no-op (its CRD lookup treats NotFound as "built-in" and skips all
   status handling).
2. Invisibility is unachievable anyway: crd-autoregistration persists an APIService per served
   group/version, legacy `/apis` is built solely from those objects, and skewed members fight
   over them (create/delete war, fleet-wide discovery flapping).
3. Ecosystem breakage: `kubectl wait --for=condition=Established` (the universal CRD install
   gate) hangs; `kubectl get crd` 404s; `kubectl apply` creates a colliding duplicate; SSA
   managedFields have no home.
4. Implementation hazards are panic-class: the naming controller's mutation cache
   `panic()`s on a non-integer ResourceVersion; the crdHandler nil-derefs on a missing
   Established condition; six status controllers need forkable skip-guards that will rot; the
   merged lister must coherently satisfy both `Lister()` and `Informer().GetIndexer()` or
   deleting any unrelated CRD tears down every static CRD's storage.

Its one real advantage — servability before the CRD informer syncs — buys availability for a
resource whose data and authorizing RBAC are both in etcd anyway.

### Read-Only Synthetic CRD Objects Merged into the Registry

Serve synthetic objects through GET/LIST on `customresourcedefinitions` while keeping them out
of etcd. Rejected: it violates list/watch semantics unfixably — synthetic entries never appear
in watch streams (informers see them only at relist and hold stale entries after removal),
continue tokens have no position for them, fabricated resourceVersions poison client RV
comparisons, and there is no in-tree precedent for a GET/LIST/WATCH-able non-persisted object
(`selfsubjectreview` is create-only, the opposite shape).

### Inline Per-CRD Storage Configuration in the Manifest

A `storage:` stanza (or envelope kind) in the manifest, carrying etcd endpoints and watch-cache
settings. Honestly attractive: atomic and self-describing (one artifact declares the type and
its data location), structurally immune to the flag-grammar typo class, the natural home for a
stable backend identity and per-backend TLS, and the shape reviewers of the abandoned PRs
asked for ("we have to pass through a custom override data structure" — sttts, k/k#84467).
Ruled out for alpha by the committed design decision, and on the merits: it would split
storage policy across two mechanisms (every other resource's placement lives on the flags),
would forfeit the free `etcd-override-N` health plumbing keyed off the flag, and would put
etcd endpoint (and eventually credential) configuration into a manifest directory with a
different review and distribution lifecycle than the flags. Revisit at beta strictly for what
the flags cannot express (backend identity, TLS) — see the witness UNRESOLVED.

### Extending the Override Flag Grammar

Per-backend TLS, etcd prefixes, or grammar aliases inside `--etcd-servers-overrides`.
Rejected: declined upstream twice (k/k#82580: "the existing override option is pretty
unfriendly to use, and I don't love extending the ad-hoc string format further"), and the
alias specifically is unsafe because `schema.ParseGroupResource` splits on the first dot.

### A Storage Binding Without Registration

A strictly smaller feature: no manifest, no reconciler, no protection — just a startup-time
binding table that *claims* an existing (dynamic) CRD's GroupResource for the two storage
flags, with the same witness/guard/precheck safety half. It delivers motivations 1 and 3 of
this KEP at a fraction of the cost, and the entire storage half of this design transfers to it
unchanged. Not chosen because it re-opens the objection this KEP exists to dissolve (a static
flag paired with a type that remains API-mutable and API-deletable: the CRD's spec, versions,
scope — and existence — could change under a fixed binding), and because it abandons the
deletion-protection co-benefit entirely. Recorded here deliberately as risk management: if SIG
review rejects the registration premise, this shape is the documented fallback rather than a
restart.

### Renaming the etcd Override Health Checks

`etcd-override-<n>` positional names are poor (adding an entry renumbers others), and a
second consumer class makes them worse. Rejected as a gratuitous monitoring break for
existing users; the `apiserver_custom_resource_storage_binding{storage_cluster_id}` join
metric provides the resource-to-backend mapping instead.

### Hot-Reloadable Storage Configuration

Rejected on mechanics, not taste, and each mechanism is citable: healthz/readyz checks are
physically un-addable once the endpoints are installed (`healthz.go` guards on
`checksInstalled`) and un-removable (a decommissioned backend's probers would fail forever);
the db-size monitor cache snapshots its backend list once; rebinding a live GroupResource is
data-unsafe (foreign resourceVersion space, orphaned keys, GC/finalizer flows straddling
backends during the drain window); and the etcd client bakes its server list in at
construction, with none of the `atomic.Value` indirection that lets encryption config reload.
Startup-frozen bindings are the only contract the surrounding machinery can honor — which is
also the KEP's core thesis.

### Derived Orderings Instead of an Authored Revision

Every scheme that infers "newer" rather than having the operator declare it, each killed
structurally by Invariant B (mid-roll, older-content members are observationally identical
between a forward roll and a rollback): **file mtime** — a distribution artifact (kubelet
re-stamps a fresh timestamped directory on every projection; a rebuilt node looks newest) with
full clock-skew dependence; **authored timestamps** — a manually bumped integer with false
automaticity (forgotten exactly because it looks automatic; merge conflicts auto-resolve it
wrongly; rollback still goes backwards); **auto-increment** (write recorded+1 on divergence) —
two divergent members escalate forever, mechanizing the fight and destroying the
misconfiguration signal, which depends on revision and content being independently declared;
**content-hash-only or opaque generation strings** — equality with no order, so every change
is a tie and every roll freezes or flaps; **hash-chain parent pointers** — true causal order
and fork detection, but a hand-authored file cannot carry a correct predecessor hash without
mandatory CI tooling (the stated CUJ is hand-authoring, including emergency SSH edits), and it
deadlocks on a raw `git reset` identically. Also rejected: a **hash tie-break at equal
revision** (resolves a genuine misconfiguration by coin flip, flips winners on cosmetic edits,
and silences the strongest signal the scheme produces) and **per-CRD revisions** (N bump
points multiply forgetting; set-level acts — tombstones, moving content between files — have
no per-CRD home; per-CRD granularity is already supplied by the per-document content hash).

### Coordination-Based Arbitration

**Leader election** gives at-most-one-writer-at-a-time, not exactly-one-write-per-change:
leadership is uncorrelated with manifest content, so a rolling restart still produces
alternating writes as leadership moves between differently-configured members; it orders
nothing (a leader cannot distinguish roll from rollback — every hard requirement here survives
election intact); and it adds a lease dependency that contradicts the pure-function readiness
contract. **Per-member claim tables** (the StorageVersion pattern: each member publishes a row
keyed by apiServerID, the max-revision holder writes) need lease-based garbage collection of
stale rows — upstream GCs StorageVersion rows via identity Leases and a dedicated controller —
and cross-apiserver synchronization is a stated Non-Goal. **Content-addressed intent chains
with fleet-agreement two-phase commit** (immutable revision snapshots plus an activation
gated on all live members acknowledging) make hijack structurally impossible but require a
mandatory manifest-generator tool, couple every spec change's liveness to the identity-Lease
roster (one wedged-but-renewing member parks the fleet), and would be the first in-tree
DAG-ordered commit protocol — while still conceding that the change lands fleet-atomically at
commit. The authored integer achieves the defer/freeze behavior with hand-editable ergonomics.

### Server-Side Apply as the Reconciler's Write or Arbitration Mechanism

Same-manager applies from all members are *one actor* to the merge engine: conflicts are only
raised between managers, so the fight would proceed silently at full speed — laundered past
SSA's only detection mechanism. Distinct per-member managers invert the failure: conflicts
veto, they do not order, so a genuinely newer member is rejected unless it forces, and forcing
steals ownership back and forth. `Apply`'s prune-and-add-back semantics additionally let a
co-owning GitOps manager pin fields the manifest removed (an unbounded, error-free loop), and
`managedFields` records wall-clock time per manager and upstream pointedly refuses to
arbitrate with it. The reconciler uses plain CAS PUTs; SSA remains what user tooling speaks.

### Accepting Divergent API Writes (Revert, Ignore, or Warn)

**Accept-and-revert** is the bootstrap-RBAC model, and against a *remote* GitOps loop it is a
permanent write/revert flap in which every round is a real spec change — a fleet-wide storage
teardown and watch relist every 60–180 seconds, driven by a controller the cluster operator
cannot see or throttle. **Accept-and-ignore** makes the API lie about what it did; tooling
reads success, then diffs drift forever. **Warn-and-accept** has the same problem, because no
GitOps controller surfaces warning headers. Denial with an instructive message is the only
honest option, and its cost — a red sync — is the correct signal that two declarers disagree.

### Delete via Manifest, and a Separate "Retire" Tombstone Level

A `Delete` tombstone action would violate Goal 7, re-create the finalizer-cascade footgun via
a file drop with no acting principal in the audit log, and contradict a design that denies
DELETE to cluster-admin. Demote-then-ordinary-API-delete is the supported composition;
`DeleteIfEmpty` (emptiness-proven, creates denied while checking) is noted for beta only. A
softer `Retire` level (demote but keep DELETE denied) was rejected: the transition contract
requires demotion to leave the API CRD fully in charge — *including deletable* — and because
protection keys on the label, demotion is fleet-atomic at one write, so there is no
partial-fleet window for a softer level to protect. Also rejected: tombstones as a list in the
`CustomResourceConfiguration` file (two-artifact skew on one node; a merge-conflict magnet), a
`retire: true` annotation on a still-present manifest (keeps a dead 2000-line schema in review
scope, plus a two-step lifecycle that reintroduces silent no-ops), and a `retired/`
subdirectory (breaks direct-children enumeration).

### The Revision as a `CustomResourceConfiguration` Field

The merged design originally carried `revision`/`supersedes` on the flag-named config file.
Rejected: the config file and the manifest directory are independently delivered artifacts
(two mounts, frequently two repositories), so the ordering token could arrive before or after
the content it orders — a config-first landing would CONFIRM the new revision against old
content and then permanently tie-freeze the real change; a directory-first landing transiently
freezes a perfectly authored commit. It also splits one logical change across two review
diffs and makes the recommended CI lint unwritable when the artifacts live in different repos.
The in-directory `CustomResourceManifestSet` document makes revision, content, and tombstones
one atomic swap. (A middle path — keep the field on the config file plus a declared
`manifestSetHash` the directory must match — was considered and set aside as strictly more
operator burden for the same atomicity.)

### Per-CRD Status Lists for Fleet Convergence

A `status` list of per-member entries keyed by apiServerID (the StorageVersion shape:
per-member rows plus an `AllEncodingVersionsEqual`-style derived condition) gives the best
`kubectl wait` UX for "has every member loaded revision N", but costs entries×members status
writes per revision and *requires* a storageversiongc-style reaper (machine-replacement
rollouts guarantee apiServerID churn; without reaping, ghost members hold convergence false
forever). The revision annotation plus per-member gauges (alpha) and the identity-Lease stamp
(beta — self-GC by lease expiry, no reaper to build) answer both convergence questions without
either cost.

### Member-Local Spec-Update Allowance

The earlier draft allowed a spec-touching UPDATE when the result matched *this member's
manifest*. Member-local admission is member-dependent admission behind a load balancer: mid-
roll it either spuriously denies no-op applies on not-yet-rolled members (breaking the Story 2
promise exactly when install tooling runs) or — in the variant that adds "or matches the local
manifest" as an alternative arm — lets *old* content be written back through a stale member,
wedging the fleet into a tie-freeze. The object-anchored no-op rule is fleet-uniform by
construction and is the direct enforcement of the never-overridden invariant.

## Infrastructure Needed (Optional)

None beyond standard CI. The integration tests use the existing multi-etcd harness pattern
(`multiEtcdSetup`).
