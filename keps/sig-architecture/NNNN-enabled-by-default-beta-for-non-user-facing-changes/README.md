# KEP-NNNN: Enabled-by-Default Beta for Non-User-Facing Changes

<!-- toc -->
- [Release Signoff Checklist](#release-signoff-checklist)
- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Proposal](#proposal)
  - [Definition: non-user-facing change](#definition-non-user-facing-change)
  - [The rule](#the-rule)
  - [Lifecycle](#lifecycle)
  - [Relationship to existing policy](#relationship-to-existing-policy)
  - [Historical precedent](#historical-precedent)
  - [Risks and Mitigations](#risks-and-mitigations)
- [Drawbacks](#drawbacks)
- [Alternatives](#alternatives)
- [Implementation History](#implementation-history)
<!-- /toc -->

## Release Signoff Checklist

Items marked with (R) are required *prior to targeting to a milestone / release*.

- [ ] (R) Enhancement issue in release milestone, which links to KEP dir in [kubernetes/enhancements] (not the initial KEP PR)
- [ ] (R) KEP approvers have approved the KEP status as `implementable`
- [ ] (R) Design details are appropriately documented
- [ ] (R) Graduation criteria is in place
- [ ] "Implementation History" section is up-to-date for milestone
- [ ] Supporting documentation e.g., additional design documents, links to mailing list discussions/SIG meetings, relevant PRs/issues, release notes

[kubernetes.io]: https://kubernetes.io/
[kubernetes/enhancements]: https://git.k8s.io/enhancements
[kubernetes/kubernetes]: https://git.k8s.io/kubernetes
[kubernetes/website]: https://git.k8s.io/website

## Summary

<<[UNRESOLVED is-a-kep-warranted ]>>
TODO: Decide whether this needs to be a standalone KEP at all. The substantive
permission may already exist: [KEP-5241] sets no requirement for a
disabled-by-default beta "soak" and explicitly allows a gate to "be set to
enabled by default in the release it is introduced," and non-user-facing changes
already ship this way in practice (e.g. [KEP-5116] launched directly to beta;
[KEP-2340] was enabled by default at beta). If so, this is a *clarification*, not
a new rule, and its natural home may be a short amendment to [KEP-5241] or PRR /
developer-guide guidance rather than a new KEP.

The deciding question is empirical: does the disabled-by-default-beta soak get
demanded in review/PRR for non-user-facing changes often enough to be worth a
written position? If yes, codify it (here or in KEP-5241). If no, this is already
de facto resolved and a standalone KEP is overhead. Resolve before promoting past
`provisional`.
<<[/UNRESOLVED]>>

A *non-user-facing change* optimizes how Kubernetes does its work — for scale,
performance, efficiency, or stability — without changing the system's observable
behavior or API surface. RangeStream for etcd ([KEP-5966]) and resilient
watchcache initialization ([KEP-4568]) are recent examples.

This KEP states that a non-user-facing change MAY be promoted to, or introduced
directly as, an **enabled-by-default beta** once it meets the existing beta
promotion criteria ([KEP-5241]). It does not need to spend a release as a
disabled-by-default beta first.

For these changes the feature gate is not an opt-in switch that entices users to
adopt a new capability. It is a **back-to-safety kill switch** that a cluster
administrator can flip in the rare event the change misbehaves in their
environment. Requiring a disabled-by-default beta "soak" for such changes adds
delay without adding safety or signal, and creates an incentive to ship them as
ungated implementation details that skip the KEP and production-readiness
process entirely.

## Motivation

Kubernetes feature gates default to disabled at alpha and enabled at beta
([KEP-5241]). For a *user-facing* feature, the disabled-by-default period is
valuable: motivated users opt in, exercise the new capability, and report
problems. The opt-in gate is an enticement that generates real-world signal
before the feature is on everywhere.

A non-user-facing change has no such enticement. Nobody opts in to "etcd uses a
server-streaming Range RPC internally" or "the watchcache initializes
differently." A disabled-by-default gate therefore gathers essentially no field
signal — it only delays the point at which the change is exercised in
production. The only way to obtain real-world validation at scale for such a
change is to enable it by default.

This produces a perverse incentive. A non-user-facing change is **not strictly
required** to be feature-gated at all: it changes no API and no documented
behavior, so it can legitimately ship as an internal implementation detail
(e.g. [KEP-1152] shipped ungated). Authors gate these changes anyway, out of an
abundance of caution, to gain a kill switch and the review that comes with the
KEP and Production Readiness Review ([KEP-1194]) processes. Imposing an extra
release of graduation delay on the cautious, gated path — relative to the
ungated path that has no delay — penalizes the behavior we want to encourage. It
pushes authors toward shipping internal changes without a gate, without a KEP,
and without a kill switch.

Accepting enabled-by-default beta as a responsible outcome for non-user-facing
changes keeps incentives aligned: the change is exercised in production where the
signal actually is, while remaining observable, reviewed, and disable-able.

### Goals

- State that a non-user-facing change may reach an enabled-by-default beta once
  it meets the beta promotion criteria of [KEP-5241], without an intervening
  disabled-by-default beta release.
- Keep incentives aligned so that authors take the gated, KEP-and-PRR path for
  internal changes rather than shipping them as ungated implementation details.
- Give SIG and PRR reviewers a clear definition and set of conditions to apply.

### Non-Goals

- Change the rules for user-facing features.
- Change the "beta APIs are off by default" rule for REST API versions
  ([KEP-3136]); new or changed APIs are user-facing and out of scope.
- Remove or weaken the feature gate. The disable-able kill switch is required,
  not optional.
- Lower the production-readiness bar. This KEP relies on the [KEP-5241] beta
  criteria; it does not relax them.
- Permit promotion to GA without field experience. GA still follows [KEP-5241]:
  all gaps identified during beta are resolved and promotion carries no
  significant change.

## Proposal

### Definition: non-user-facing change

A non-user-facing change alters *how* Kubernetes performs work without changing
*what* a client observes. It introduces no new user-visible API, field, or
documented behavior; existing clients cannot tell whether it is enabled except
through improved resource usage, latency, or availability. Typical examples are
internal storage and etcd access patterns, the watchcache, serialization, and
internal request handling.

If a change has any observable effect on API responses or documented behavior,
it is user-facing and the normal feature-gate rules apply.

### The rule

A non-user-facing change MAY be promoted to, or introduced directly as, an
**enabled-by-default beta** when all of the following hold:

1. **It is non-user-facing**, as defined above. The owning SIG and PRR approver
   confirm it adds or alters no user-visible API or behavior.
2. **It meets the [KEP-5241] beta criteria**: all functional, security,
   monitoring, and testing requirements are satisfied, and all known issues and
   gaps are resolved. Beta is not a place to finish incomplete work.
3. **The gate is a back-to-safety kill switch**: `disable-supported: true`, and
   disabling the gate returns the component to its prior behavior, per the
   rollback guarantees of [KEP-1194].
4. **Regressions are observable**: metrics or other signals exist to detect a
   performance, stability, or correctness regression caused by the change in a
   production cluster.
5. **Skew-safe**: components interoperate correctly with the gate enabled or
   disabled across a supported version skew, per [KEP-5241].

This is an application of [KEP-5241], not an exception to it. [KEP-5241] already
establishes that a beta gate is enabled by default and must be production-ready
before it is. This KEP clarifies that, for a non-user-facing change, meeting that
bar is *sufficient* to be enabled by default at beta — there is no additional
benefit to a disabled-by-default beta release, because no one opts in to exercise
it.

### Lifecycle

- **Alpha (gate off by default).** Optional and recommended for initial
  validation by the author, in CI, and by early adopters who specifically want
  to validate the change (for example, operators of large clusters). It is not a
  stage to linger in, because it accumulates little field signal.
- **Beta (gate on by default, disable-able).** The change is exercised in
  production at scale; the gate is a kill switch. Requires the conditions above.
  A change may start here, skipping alpha, when confidence already warrants it
  (as [KEP-5116] did).
- **GA.** Per [KEP-5241]: all gaps identified during beta are resolved and the
  promotion carries no significant change. The gate locks on (or remains
  disable-able only in the uncommon new-serialization case described by
  [KEP-5241]).

### Relationship to existing policy

- **[KEP-5241] Beta Feature Gate Promotion Requirements** — this KEP applies
  KEP-5241's bar to the non-user-facing case. It does not change that bar.
- **[KEP-1194] Production Readiness Review** — unchanged. The disable/rollback
  requirement is exactly the kill-switch property this KEP relies on.
- **[KEP-3136] Beta APIs Are Off by Default** — unchanged and out of scope.
  KEP-3136 governs REST API versions, which are user-facing by definition.

### Historical precedent

SIG API Machinery and SIG etcd already ship non-user-facing changes this way.
The following all graduated (or are graduating) to an enabled-by-default beta as
internal optimizations behind a disable-able gate, except where noted:

| KEP | Internal change | Feature gate(s) | Notes |
|---|---|---|---|
| [KEP-5966] etcd RangeStream | Server-streaming `RangeStream` RPC to avoid server memory spikes and redundant count work on paginated lists | (proposed) | sig-etcd; no observable API change; KEP is currently `provisional` |
| [KEP-4568] Resilient watchcache initialization | Avoids overloading kube-apiserver/etcd during watchcache (re)initialization | `ResilientWatchCacheInitialization`, `WatchCacheInitializationPostStartHook` | beta v1.31 → stable v1.34; `disable-supported: true` |
| [KEP-2340] Consistent Reads from Cache | Serves consistent GET/LIST from the watchcache after an etcd revision check | `ConsistentListFromCache` | alpha v1.28 → beta v1.31 (enabled by default) → stable v1.34 |
| [KEP-5116] Streaming Encoding for LIST | Streams LIST response encoding to bound apiserver memory on large lists | `StreamingCollectionEncodingToJSON`, `StreamingCollectionEncodingToProtobuf` | launched **directly to beta** v1.33 (no alpha) → stable v1.34 |
| [KEP-4988] Snapshottable API server cache | Serves paginated lists from cache snapshots | `ListFromCacheSnapshot`, `DetectCacheInconsistency` | alpha v1.33 → beta v1.34; `disable-supported: true` |
| [KEP-1152] Less Object Serializations | Caches serializations to avoid redundant encoding work | (none) | shipped **ungated** — the path this KEP discourages relative to a gated beta |

### Risks and Mitigations

**An author mislabels a user-facing change as non-user-facing.** Mitigation: the
classification is made by the owning SIG and confirmed by the PRR approver, and
any observable behavior change disqualifies the change from this path.

**A change is enabled by default before it is actually production-ready.**
Mitigation: this KEP adds no relaxation of [KEP-5241]; the beta criteria
(complete, tested, monitored, no known gaps) must be met, and the gate must be
disable-able as a kill switch.

**A regression appears only at scale in production.** Mitigation: the kill switch
returns the cluster to prior behavior, monitoring is required to detect the
regression, and version skew is handled so disabling the gate is always safe.

## Drawbacks

This formalizes a position that is already common practice rather than
introducing a new mechanism. The main risk is misapplication — treating a
user-facing change as non-user-facing to skip a disabled-by-default beta. The
definition and the SIG/PRR confirmation step are intended to contain that risk.

## Alternatives

- **Status quo: case-by-case.** Authors already do this, but without a written
  position reviewers apply inconsistent expectations, and the disabled-by-default
  "soak" is sometimes requested where it provides no signal.
- **Always require a disabled-by-default beta release.** Adds delay without
  safety or signal for non-user-facing changes and incentivizes shipping them
  ungated to avoid the delay.
- **Stop gating non-user-facing changes entirely.** Removes the kill switch and
  the review and observability that come with the KEP and PRR processes —
  precisely the safety net this KEP wants to keep authors inside.

## Implementation History

- 2026-06-02: KEP drafted (`provisional`).

<!-- Reference links -->
[KEP-5241]: https://github.com/kubernetes/enhancements/tree/master/keps/sig-architecture/5241-beta-featuregate-promotion-requirements
[KEP-3136]: https://github.com/kubernetes/enhancements/tree/master/keps/sig-architecture/3136-beta-apis-off-by-default
[KEP-1194]: https://github.com/kubernetes/enhancements/tree/master/keps/sig-architecture/1194-prod-readiness
[KEP-5966]: https://github.com/kubernetes/enhancements/tree/master/keps/sig-etcd/5966-etcd-range-stream
[KEP-4568]: https://github.com/kubernetes/enhancements/tree/master/keps/sig-api-machinery/4568-resilient-watchcache-initialization
[KEP-2340]: https://github.com/kubernetes/enhancements/tree/master/keps/sig-api-machinery/2340-Consistent-reads-from-cache
[KEP-5116]: https://github.com/kubernetes/enhancements/tree/master/keps/sig-api-machinery/5116-streaming-response-encoding
[KEP-4988]: https://github.com/kubernetes/enhancements/tree/master/keps/sig-api-machinery/4988-snapshottable-api-server-cache
[KEP-1152]: https://github.com/kubernetes/enhancements/tree/master/keps/sig-api-machinery/1152-less-object-serializations
