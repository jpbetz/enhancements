# Consistent Reads from Cache

## Table of Contents

<!-- toc -->
- [Release Signoff Checklist](#release-signoff-checklist)
- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Proposal](#proposal)
  - [Risks and Mitigations](#risks-and-mitigations)
- [Design Details](#design-details)
  - [Progress notify interval selection](#progress-notify-interval-selection)
  - [Pagination](#pagination)
  - [What if the watch cache is stale?](#what-if-the-watch-cache-is-stale)
  - [Ability to Opt-out](#ability-to-opt-out)
  - [Test Plan](#test-plan)
  - [Rollout Plan](#rollout-plan)
    - [Serving consistent reads from cache](#serving-consistent-reads-from-cache)
    - [Reflectors](#reflectors)
  - [Graduation Criteria](#graduation-criteria)
- [Implementation History](#implementation-history)
- [Alternatives](#alternatives)
- [Potential Future Improvements](#potential-future-improvements)
<!-- /toc -->

## Release Signoff Checklist

<!--
**ACTION REQUIRED:** In order to merge code into a release, there must be an
issue in [kubernetes/enhancements] referencing this KEP and targeting a release
milestone **before the [Enhancement Freeze](https://git.k8s.io/sig-release/releases)
of the targeted release**.

For enhancements that make changes to code or processes/procedures in core
Kubernetes—i.e., [kubernetes/kubernetes], we require the following Release
Signoff checklist to be completed.

Check these off as they are completed for the Release Team to track. These
checklist items _must_ be updated for the enhancement to be released.
-->

Items marked with (R) are required *prior to targeting to a milestone / release*.

- [ ] (R) Enhancement issue in release milestone, which links to KEP dir in [kubernetes/enhancements] (not the initial KEP PR)
- [ ] (R) KEP approvers have approved the KEP status as `implementable`
- [ ] (R) Design details are appropriately documented
- [ ] (R) Test plan is in place, giving consideration to SIG Architecture and SIG Testing input (including test refactors)
  - [ ] e2e Tests for all Beta API Operations (endpoints)
  - [ ] (R) Ensure GA e2e tests for meet requirements for [Conformance Tests](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/conformance-tests.md)
  - [ ] (R) Minimum Two Week Window for GA e2e tests to prove flake free
- [ ] (R) Graduation criteria is in place
  - [ ] (R) [all GA Endpoints](https://github.com/kubernetes/community/pull/1806) must be hit by [Conformance Tests](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/conformance-tests.md)
- [ ] (R) Production readiness review completed
- [ ] (R) Production readiness review approved
- [ ] "Implementation History" section is up-to-date for milestone
- [ ] User-facing documentation has been created in [kubernetes/website], for publication to [kubernetes.io]
- [ ] Supporting documentation—e.g., additional design documents, links to mailing list discussions/SIG meetings, relevant PRs/issues, release notes

<!--
**Note:** This checklist is iterative and should be reviewed and updated every time this enhancement is being considered for a milestone.
-->

[kubernetes.io]: https://kubernetes.io/
[kubernetes/enhancements]: https://git.k8s.io/enhancements
[kubernetes/kubernetes]: https://git.k8s.io/kubernetes
[kubernetes/website]: https://git.k8s.io/website

## Summary

Kubernetes List requests are guaranteed to be "consistent reads" if the
`resourceVersion` parameter is not provided. Consistent reads are served from
etcd using a "quorum read".

But often the watch cache contains sufficiently up-to-date data to serve the
read request, and could serve it far more efficiently.

This KEP proposes a mechanism to serve consistent reads from the watch cache
while still providing the same consistency guarantees as serving the
read from etcd.

## Motivation

Serving reads from the watch cache is more performant and scalable than reading
them from etcd, deserializing them, applying selectors, converting them to the
desired version and then garbage collecting all the objects that were allocated
during the whole process.

An [attempt to fix](https://github.com/kubernetes/kubernetes/pull/83520) the ["stale read" problem](https://github.com/kubernetes/kubernetes/issues/59848)
was [reverted](https://github.com/kubernetes/kubernetes/pull/86823) due the performance cost of serving consistent reads
from etcd for a code path that previously served the reads from the watch cache.

We will need to measure the impact to performance and scalability, but we have
enough data and experience from prior work with the watch cache to be confident
there is significant scale/perf opportunity here, and we would like to introduce
an alpha implementation.

We expect the biggest gain to be from node-originating requests (e.g. kubelet
listing pods scheduled on its node). For those requests, the size of the
response is small (it fits a single page, assuming you won't make it extremely
small), whereas the number of objects to process is proportional to cluster-size
(so fairly big). For example, when kubelets requests pods schedule against it in
a 5k node cluster with 30pods/node, the kube-apiserver must list the 150k pods
from etcd and then filter that list down to the list of 30 pods that the kubelet
actually need. This must occur for each list request from each of the 5k
kubelets. If served from watch cache, this same request can be served by simply
filtering out the 30 pods each kubelet needs from the data in the cache.

### Goals

- Resolve the "stale read" problem (https://github.com/kubernetes/kubernetes/issues/59848)
- Improve the scalability and performance of Kubernetes for Get and List requests, when the watch cache is enabled

### Non-Goals

- Avoid allowing true quorum reads. See https://github.com/kubernetes/enhancements/pull/1404#discussion_r381528406 for discussion.

## Proposal

When a consistent LIST request is received (and the watch cache is enabled):

- Get the watch cache list size. This will be used as the expected list size in subsequent steps (it doesn't need to be exact, and when the API server is partitioned, it's okay if it is wildly wrong).
- Get the current revision from etcd for the resource type being served using [`WithLatestRev`](https://pkg.go.dev/github.com/zxpan/etcd/clientv3#WithLastRev). The returned revision is strongly consistent (guaranteed to be the latest revision via a quorum read).
  - If this request fails because the API server or the etcd it made the request to is partitioned, fail the consistent LIST request.
- Calculate how long of a wait to tolerate for the watch cache to become up-to-date, based on the expected list size.
  - For sufficiently small list sizes, there is a negligible benefit to serving the request from cache, so the wait toleration should be very short (e.g. 10ms).
  - For sufficiently large list sizes, even a large wait for the watch cache to become up-to-date is negligible, so the wait toleration can be long (e.g. 500ms).
  - For all other list sizes we should strive to minimize latency, but should prioritize API server scalability/throughput over latency.
    `WatchProgressRequest` requests can be sent to etcd to influence the expected wait time for the watch cache to become up-to-date
    (See "progress event scheduling" section for details).
- Check if the watch cache is up-to-date with the revision retrieved from etcd:
  - If so, serve the request from the watch cache
  - If not use the existing `waitUntilFreshAndBlock` function in the watch cache to wait briefly for the watch to catch up to the current revision.
  - If the block times out
    - If the list size is sufficiently large that serving it from etcd would negatively impact API server scalability, fail the request (see "What if the watch cache is stale?" section for details).
    - Otherwise, server the request directly from etcd

Progress event scheduling:

In order for the watch cache to know that it is sufficiently up-to-date, we rely on the etcd progress notify capability.
Progress notify is already being used by [Efficient Watch Resumption](https://github.com/kubernetes/enhancements/tree/master/keps/sig-api-machinery/1904-efficient-watch-resumption)
with `--experimental-watch-progress-notify-interval` flag typically set a relatively short duration. [Prior scalability testing](https://github.com/kubernetes/kubernetes/pull/86769) showed that
the performance impact was negligible with a progress notify interval of 250ms.

For this feature we want to ensure that a progress events are received regardless of how the etcd `--experimental-watch-progress-notify-interval` flag is configured.
So we will schedule requests to the etcd [`WatchProgressRequest`](https://etcd.io/docs/v3.5/dev-guide/api_reference_v3/#message-watchprogressrequest-apietcdserverpbrpcproto) operation.
For example, if `--experimental-watch-progress-notify-interval` flag is not configured, then the default interval is 10 minutes.
The default is insufficient for both this feature and for Efficient Watch Resumption, so we will update the API server to schedule events at least every 250ms (but
no more than, say, every 125ms, even when consistent read load is high). Since progress notifications triggered by the `--experimental-watch-progress-notify-interval`
can be observed, we can account the events already being received when scheduling. For example, if `--experimental-watch-progress-notify-interval` is set to
250ms, and we need to receive events every 300ms, we can schedule to send a `WatchProgressRequest` request every 300ms, but cancel it after ~250ms when we observe
a progress notify event from etcd and reschedule it again for 300ms in the future.

The API server can also schedule `WatchProgressRequest` more frequently when it benefits the overall system. For example, if the API server receives a consistent LIST request and the next scheduled
`WatchProgressRequest` is 249ms in the future, it could be rescheduled to be sooner. The scheduling algorithm can balance the cost of handling additional progress
notify events against the benefits of reducing latency to clients.

A major benefit to scheduling `WatchProgressRequest` events is that it has roughly the effect as setting `--experimental-watch-progress-notify-interval` to the same value but is operationally simpler.
Both this feature and [Efficient Watch Resumption](https://github.com/kubernetes/enhancements/tree/master/keps/sig-api-machinery/1904-efficient-watch-resumption) need progress notification
events, and ensuring they occur regardless of how etcd was configured simplifies Kubernetes cluster administration.

Consistent GET requests will continue to be served directly from etcd. We will only serve consistent LIST requests from cache. This would be revisited
in the future.

### Risks and Mitigations

Risk: etcd was not configured with a short enough `--experimental-watch-progress-notify-interval`

- Mitigation: `WatchProgressRequest` operations will be scheduled by the API server

Risk: burst of consistent read requests trigger a large volume of progress notify events

- Mitigation: Schedule `WatchProgressRequest` operations so that the maximum number sent per unit time is bounded

Risk: the etcd version used doesn't support `WatchProgressRequest` then it won't send progress notify events when asked. 

- Mitigation: Serve consistent reads from etcd. The API Server could check the etcd version on startup and only attempt to
  wait for progress notify events for etcd 3.3.14 (the etcd version were `WatchProgressRequest` was introduced).

## Design Details

### Progress notify interval selection

Not all LIST requests require consistency, and we're taking the view that if you
really want to have a consistent LIST (we should explicitly exclude GETs from
it), then you may need to pay additional tax latency for it. For this reason, we
intend to start with a 250ms progress notify interval, which will on average 125ms
latency to each consistent LIST request. The scheduling of `WatchProgressRequest` operations
will be aligned with this, typically sending requests for progress notify events only when >250ms have passes
since the last progress notify event was received, and sending them no more often then every 125ms even when
there is heavy consistent read load.

The requests this is expected to impact are:

- Reflector list/relist requests, which occur at startup and after a reflector
  falls to far behind processing events (e.g. it was partitioned or resource starved)
- Controllers that directly perform consistent LIST requests

In all cases, increasing latency in exchange for higher overall system
throughput seems a good trade off. Use cases that need low latency have multiple
options: Watching resources with shared informers or reflectors, LIST requests
with a minimum resource version specified.

During our testing of this feature we will gather more data about the impact of
selecting 250ms for the progress notify interval. We will also use alpha to
gather feedback from the community on the latency impact.

### Pagination

This feature will interact with pagination:

All consistent read requests prior to the introduction of this feature will receive paginated responses if a pagination
limit is requested because the requests are served by etcd, not the watch cache. But unless watch cache pagination is
introduced, all consistent reads after the introduction of this feature will receive unpaginated responses.

Pagination is not easily added to the watch cache because the cache only supports serving lists requests at the
resource version of the most recent resource version observed in the watch events.

Reconciling how this feature interacts with pagination is listed as a precondition to graduating to beta.

### What if the watch cache is stale?

The most likely reasons for the watch cache to be stale:

- The API Server is partitioned from etcd, or the etcd the API server is receiving watch events from it partitioned from the etcd cluster.
- The etcd version in use doesn't support `WatchProgressRequest` (the feature is [available](https://github.com/etcd-io/etcd/blob/main/CHANGELOG-3.3.md#v3314-2019-08-16) on etcd 3.3.14 or newer)

If the API server is partitioned, it should refuse to serve a consistent read request. TODO: What exact response should it send?

If the etcd version doesn't support `WatchProgressRequest` then it won't send progress notify events when asked. This is discussed
in the mitigations section.

### Ability to Opt-out

How to opt out of this behavior and still get a "normal" quorum read? We'll need this ability for our own debugging if nothing else.
See https://github.com/kubernetes/enhancements/pull/1404#issuecomment-588433911

The most uniform way to disable this feature at runtime would be to provide a way to skip the watch cache for any particular request.
But this complicates the API and bleeds through implementation details. We could instead provide a flag that is only available when the feature
is in alpha and beta that skips the watch cache just for this feature.

### Test Plan

Correctness:

- Verify that we don't violate the linerizability guranentees of consistent reads:
  - Unit test with a mock storage backend (instead of an actual etcd) that
    various orderings of progress notify events and "current revision" response
    result in the watch cache serving consistent read requests correctly
  - Soak test to ensure that consistent reads always return data at resource
    versions no older that previous writes occurred at. In either e2e tests,
    scalability tests or a dedicated tester that we run for an extended
    duration, we can add a checker that periodically performs writes and
    consistent reads and ensure the read resource versions are not older than
    the resource versions of the writes.
  - Introduce e2e test that run both with etcd progress notify events enabled
    and disable to ensure both configurations work correctly (both with this
    feature enabled and disabled)

Performance:

- Benchmark consistent reads from cache against consistent reads from etcd for:
  - list result sizes of 1, 10, ..., 100000
  - object sizes of 5kb, 25kb, 100kb
  - measure latency and throughput
  - document results in this KEP

Scalability:

- 5k scalability tests verifying that introducing etcd progress notify events
  don't degrade performance/scailability (early results available here:
  https://github.com/kubernetes/kubernetes/pull/86769)
- 5k scalability tests verifying that there are substantial scalability benefits
  to enabling consistent reads from cache for the pod list from kubelet use case
  - Latency output contains what we need to ensure the impact to latency of
    delaying consistent reads for the progress notify interval is what we expect
    (~250ms more latency for these requests on average)
  - Scalability output contains what we need to ensure we are within SLOs and
    our scalability goals
  - Since pod list requests were previously served from the watch cache (but
    without a consistency guarantee), we expect scalability to be roughly the
    same as baseline (but with the benefit of improved correctness)

### Rollout Plan

#### Serving consistent reads from cache

Guard feature with the `WatchCacheConsistentReads` feature gate.

#### Reflectors

- Provide a way for reflectors to be configured to use resourceVersion=”” for initial list, but for backward compatibility, resourceVersion=”0” must remain the default for reflectors.
- Upgrade the reflectors of in-tree components to use resourceVersion=”" based on a flag or configuration option. Administrators and administrative tools would need to enable this only when using etcd 3.4 or higher.
- If at some point in the (far) future, the lowest etcd supported version for kubernetes is 3.4 or higher, reflectors could be changed to default to resourceVersion="".

### Graduation Criteria

Beta:

- Interaction to pagination (calls that previously were paginated by etcd will be unpaginated when served from the watch cache) is addressed.

## Implementation History

TODO

## Alternatives

Do nothing:

- Leaves the "stale read" problem unsolved, although we have a PR fixing reflector relist which helps mitigate the larger issue.
- Does not impact scale or performance.

Instruct clients to track the resource version they provide to reflectors:

- We've already seen that clients elect to use resourceVersion=”0” even if it violates their consistency needs
- It can be difficult for clients to keep track of the last resourceVersion they observed. For example many controllers are written to be stateless.

## Potential Future Improvements

Modify etcd to allow echo back a user provided ID in progress events.

- Client generates a UUID and provides to the ProgressNotify request
- Once client sees a progress event with the same UUID, it knows the watch is up-to-date
- This reduces the worst case number of round trips required to do a consistent read from two to one since client doesn't need to get the lastest revision from etcd first

Potential optimiation: We could delay requests, accumulate multiple in-flight
read requests over some short time period, and at the end of the period, get the
current revision from etcd, wait for the watch cache to catch up, and then serve
all the the in-flight reads from cache. This would reduce the number of "get
current revision" requests that need to be made to etcd in exchange for higher
request latency (but only for consistent reads). A simple implementation would
be to do this on a fix interval, where, obviously, if there were not reqests
during the period, we don't bother to fetch a current revision from etcd. It
is unclear if this will result in actual gain, and it would complicate the
code, so should be explored with care.
