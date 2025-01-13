# KEP-3953: Node Resource Hot Plug

<!--
A table of contents is helpful for quickly jumping to sections of a KEP and for
highlighting any additional information provided beyond the standard KEP
template.

Ensure the TOC is wrapped with
  <code>&lt;!-- toc --&rt;&lt;!-- /toc --&rt;</code>
tags, and then generate with `hack/update-toc.sh`.
-->

<!-- toc -->
- [Release Signoff Checklist](#release-signoff-checklist)
- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Proposal](#proposal)
  - [User Stories (Optional)](#user-stories-optional)
    - [Story 1](#story-1)
  - [Notes/Constraints/Caveats (Optional)](#notesconstraintscaveats-optional)
  - [Risks and Mitigations](#risks-and-mitigations)
- [Design Details](#design-details)
  - [Test Plan](#test-plan)
      - [Unit tests](#unit-tests)
      - [e2e tests](#e2e-tests)
  - [Graduation Criteria](#graduation-criteria)
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

The proposal aims at enabling hot plugging of node compute resources. This will help in updating cluster resource capacity by just resizing compute resources of nodes rather than adding new node to a cluster.
The updated node configurations are to be reflected at the node and cluster levels automatically without the need to reset the kubelet.

This proposal also aims to improve the initialization and reinitialization of resource managers, such as the CPU manager and memory manager, in response to changes in a node's CPU and memory configurations.

## Motivation
In a typical Kubernetes environment, the cluster resources may need to be altered due to following reasons:
- Incorrect resource assignment during cluster creation.
- Increased workload over time, leading to the need for additional resources in the cluster.

To handle these scenarios, we can:
- Horizontally scale up the cluster by adding compute nodes.
- Vertically scale up the cluster by increasing node capacity. However, currently, the workaround for capturing node resizing in the cluster involves restarting the Kubelet.

Node resource hot plugging will provide advantages in scenarios such as:
- Handling resource demand with a limited set of nodes by increasing the capacity of existing nodes instead of creating new nodes.
- Creating new nodes takes more time compared to increasing the capacity of existing nodes.

### Goals

* Dynamically scale up the node by hot plugging  resources and without restarting the kubelet.
* Ability to reinitialize resource managers (CPU manager, memory manager) to adopt changes in node's resource.

### Non-Goals

* Dynamically adjust system reserved and kube reserved values.
* Hot unplug of node resources.
* Update the autoscaler to utilize resource hot plugging.


## Proposal

This KEP aims to support the node resource hot plugging by adding a polling mechanism in kubelet to fetch the machine-information from cAdvisor's cache which is already updated periodically, This information will be fetched periodically by kubelet, after which the node status updater is responsible for updating this information at node level in the cluster.

Additionally, this KEP aims to improve the initialization and reinitialization of resource managers, such as the memory manager and CPU manager, so that they can adapt to change in node's configurations.

### User Stories

#### Story 1

As a cluster admin, I must be able to increase the cluster resource capacity without adding a new node to the cluster.

#### Story 2

As a cluster admin, I must be able to increase the cluster resource capacity without need to restarting the kubelet.

### Notes/Constraints/Caveats (Optional)

### Risks and Mitigations

1. Node resource hot plugging is an opt-in feature, merging the
   feature related changes won't impact existing workloads. Moreover, the feature
   will be rolled out gradually, beginning with an alpha release for testing and
   gathering feedback. This will be followed by beta and GA releases as the
   feature matures and potential problems and improvements are addressed.
2. Though the node resource is updated dynamically, the dynamic data is fetched from cAdvisor and its well integrated with kubelet.
3. Resource manager are updated to adapt to the dynamic node reconfigurations, Enough tests should be added to make sure its not affecting the existing functionalities.

## Design Details


Below diagram is shows the interaction between kubelet and cAdvisor.


```mermaid
sequenceDiagram
    participant node
    participant kubelet
    participant cAdvisor-cache
    participant machine-info
    kubelet->>cAdvisor-cache: Poll
    cAdvisor-cache->>machine-info: fetch
    machine-info->>cAdvisor-cache: update
    cAdvisor-cache->>kubelet: update
    kubelet->>node: node status update
    kubelet->>node: re-initialize resource managers
```

The interaction sequence is as follows
1. Kubelet will be polling in interval to fetch the machine resource information from cAdvisor's cache, Which is currently updated every 5 minutes.
2. Kubelet's cache will be updated with the latest machine resource information.
3. Node status updater will update the node's status with the latest resource information.
4. Kubelet will reinitialize the resource managers to keep them up to date with dynamic resource changes.

Note: In case of increase in cluster resources, the scheduler will automatically schedule any pending pods.

**Proposed Code changes**

**Dynamic Node Scale Up logic**

```go
	if utilfeature.DefaultFeatureGate.Enabled(features.NodeResourceHotPlug) {
		// Handle the node dynamic scale up
		machineInfo, err := kl.cadvisor.MachineInfo()
		if err != nil {
			klog.ErrorS(err, "Error fetching machine info")
		} else {
			cachedMachineInfo, _ := kl.GetCachedMachineInfo()

			if !reflect.DeepEqual(cachedMachineInfo, machineInfo) {
				kl.setCachedMachineInfo(machineInfo)

				// Resync the resource managers
				if err := kl.ResyncComponents(machineInfo); err != nil {
					klog.ErrorS(err, "Error resyncing the kubelet components with machine info")
				}
			}
		}
	}
```

**Changes to resource managers to adapt to dynamic scale up of resources**

1. Adding ResyncComponents() method to ContainerManager interface
```go
    // Manages the containers running on a machine.
    type ContainerManager interface {
        .
        .
        // ResyncComponents will resyc the resource managers like cpu, memory and topology managers
	// with updated machineInfo
	ResyncComponents(machineInfo *cadvisorapi.MachineInfo) error
	.
	.
    )
```

2. Adding a method Sync to all the resource managers and will be invoked once there is dynamic resource change.

```go
        // Sync will sync the CPU Manager with the latest machine info
	Sync(machineInfo *cadvisorapi.MachineInfo) error
```

### Test Plan

[x] I/we understand the owners of the involved components may require updates to
existing tests to make this code solid enough prior to committing the changes necessary
to implement this enhancement.

##### Unit tests

1. Add necessary tests in kubelet_node_status_test.go to check for the node status behaviour with dynamic node scale up.
2. Add necessary tests in kubelet_pods_test.go to check for the pod cleanup and pod addition workflow.
3. Add necessary tests in eventhandlers_test.go to check for scheduler behaviour with dynamic node capacity change.
4. Add necessary tests in resource managers to check for managers behaviour to adopt dynamic node capacity change.


##### e2e tests

Following scenarios need to be covered:

* Node resource information before and after resource hot plug.
* State of Pending pods due to lack of resources after resource hot plug.
* Resource manager states after the resynch of components.

### Graduation Criteria


#### Phase 1: Alpha (target 1.33)


* Feature is disabled by default. It is an opt-in feature which can be enabled by enabling the `NodeResourceHotPlug`
  feature gate.
* Support the basic functionality for scheduler to consider pod-level resource requests to find a suitable node.
* Support the basic functionality for kubelet to translate pod-level requests/limits to pod-level cgroup settings.
* Unit test coverage.
* E2E tests.
* Documentation mentioning high level design.


### Upgrade / Downgrade Strategy

<!--
If applicable, how will the component be upgraded and downgraded? Make sure
this is in the test plan.

Consider the following in developing an upgrade/downgrade strategy for this
enhancement:
- What changes (in invocations, configurations, API use, etc.) is an existing
  cluster required to make on upgrade, in order to maintain previous behavior?
- What changes (in invocations, configurations, API use, etc.) is an existing
  cluster required to make on upgrade, in order to make use of the enhancement?
-->

### Version Skew Strategy

<!--
If applicable, how will the component handle version skew with other
components? What are the guarantees? Make sure this is in the test plan.

Consider the following in developing a version skew strategy for this
enhancement:
- Does this enhancement involve coordinating behavior in the control plane and
  in the kubelet? How does an n-2 kubelet without this feature available behave
  when this feature is used?
- Will any other components on the node change? For example, changes to CSI,
  CRI or CNI may require updating that component before the kubelet.
-->

## Production Readiness Review Questionnaire

<!--

Production readiness reviews are intended to ensure that features merging into
Kubernetes are observable, scalable and supportable; can be safely operated in
production environments, and can be disabled or rolled back in the event they
cause increased failures in production. See more in the PRR KEP at
https://git.k8s.io/enhancements/keps/sig-architecture/1194-prod-readiness.

The production readiness review questionnaire must be completed and approved
for the KEP to move to `implementable` status and be included in the release.

In some cases, the questions below should also have answers in `kep.yaml`. This
is to enable automation to verify the presence of the review, and to reduce review
burden and latency.

The KEP must have a approver from the
[`prod-readiness-approvers`](http://git.k8s.io/enhancements/OWNERS_ALIASES)
team. Please reach out on the
[#prod-readiness](https://kubernetes.slack.com/archives/CPNHUMN74) channel if
you need any help or guidance.
-->

### Feature Enablement and Rollback

<!--
This section must be completed when targeting alpha to a release.
-->

###### How can this feature be enabled / disabled in a live cluster?

<!--
Pick one of these and delete the rest.

Documentation is available on [feature gate lifecycle] and expectations, as
well as the [existing list] of feature gates.

[feature gate lifecycle]: https://git.k8s.io/community/contributors/devel/sig-architecture/feature-gates.md
[existing list]: https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/
-->

- [x] Feature gate (also fill in values in `kep.yaml`)
    - Feature gate name:NodeResourceHotPlug
    - Components depending on the feature gate: kubelet
- [ ] Other
    - Describe the mechanism:
    - Will enabling / disabling the feature require downtime of the control
      plane?
    - Will enabling / disabling the feature require downtime or reprovisioning
      of a node?

###### Does enabling the feature change any default behavior?

No. This feature is guarded by a feature gate. Existing default behavior does not change if the
feature is not used. 
Even if the feature is enabled via feature gate, If there is no change in 
node configuration the system will continue to work in the same way.

###### Can the feature be disabled once it has been enabled (i.e. can we roll back the enablement)?

Yes. Once disabled any hot plug of resources won't reflect at the cluster level without kubelet restart. 

###### What happens if we reenable the feature if it was previously rolled back?

If the feature is reenabled again, the node resources can be hot plugged in again. Cluster will be automatically udpated
with the new resource information.

###### Are there any tests for feature enablement/disablement?

Yes, the tests will be added along with alpha implementation.
* Validate the hot plug of resource to machine is updated at the node resource level.
* Validate the hot plug of resource made the pending pods to transition into running state.
* Validate the resource managers are update with the latest machine information after hot plug of resources.

### Rollout, Upgrade and Rollback Planning

<!--
This section must be completed when targeting beta to a release.
-->

###### How can a rollout or rollback fail? Can it impact already running workloads?

<!--
Try to be as paranoid as possible - e.g., what if some components will restart
mid-rollout?

Be sure to consider highly-available clusters, where, for example,
feature flags will be enabled on some API servers and not others during the
rollout. Similarly, consider large clusters and how enablement/disablement
will rollout across nodes.
-->

Rollout may fail if the resource managers are not re-synced properly due to programatic errors.
In case of rollout failures, running workloads are not affected, If the pods are on pending state they remain
in the pending state only.
Rollback failure should not affect running workloads.

###### What specific metrics should inform a rollback?

<!--
What signals should users be paying attention to when the feature is young
that might indicate a serious problem?
-->

In case of pending pods and hot plug of resource but still there is no change `scheduler_pending_pods` metric
means the feature is not working as expected.

###### Were upgrade and rollback tested? Was the upgrade->downgrade->upgrade path tested?

<!--
Describe manual testing that was done and the outcomes.
Longer term, we may want to require automated upgrade/rollback tests, but we
are missing a bunch of machinery and tooling and can't do that now.
-->

It will be tested manually as a part of implementation and there will also be automated tests to cover the scenarios.

###### Is the rollout accompanied by any deprecations and/or removals of features, APIs, fields of API types, flags, etc.?

<!--
Even if applying deprecation policies, they may still surprise some users.
-->
No
### Monitoring Requirements

<!--
This section must be completed when targeting beta to a release.

For GA, this section is required: approvers should be able to confirm the
previous answers based on experience in the field.
-->

###### How can an operator determine if the feature is in use by workloads?

<!--
Ideally, this should be a metric. Operations against the Kubernetes API (e.g.,
checking if there are objects with field X set) may be a last resort. Avoid
logs or events for this purpose.
-->

This feature will be built into kubelet and behind a feature gate. Examining the kubelet feature gate would help 
in determining whether the feature is used. The enablement of the kubelet feature gate can be determined from the 
`kubernetes_feature_enabled` metric.

###### How can someone using this feature know that it is working for their instance?

<!--
For instance, if this is a pod-related feature, it should be possible to determine if the feature is functioning properly
for each individual pod.
Pick one more of these and delete the rest.
Please describe all items visible to end users below with sufficient detail so that they can verify correct enablement
and operation of this feature.
Recall that end users cannot usually observe component logs or access metrics.
-->

End user can do a hot plug of resource and verify the same change as reflected at the node resource level.
In case there were any pending pods prior to resource hot plug, those pods should transition into Running with addition
of new resources.

###### What are the reasonable SLOs (Service Level Objectives) for the enhancement?

<!--
This is your opportunity to define what "normal" quality of service looks like
for a feature.

It's impossible to provide comprehensive guidance, but at the very
high level (needs more precise definitions) those may be things like:
  - per-day percentage of API calls finishing with 5XX errors <= 1%
  - 99% percentile over day of absolute value from (job creation time minus expected
    job creation time) for cron job <= 10%
  - 99.9% of /health requests per day finish with 200 code

These goals will help you determine what you need to measure (SLIs) in the next
question.
-->
No increase in the `scheduler_pending_pods` rate.
###### What are the SLIs (Service Level Indicators) an operator can use to determine the health of the service?

<!--
Pick one more of these and delete the rest.
-->

- [X] Metrics
    - Metric name: `scheduler_pending_pods`
    - Components exposing the metric: scheduler

###### Are there any missing metrics that would be useful to have to improve observability of this feature?

<!--
Describe the metrics themselves and the reasons why they weren't added (e.g., cost,
implementation difficulties, etc.).
-->
No
### Dependencies

<!--
This section must be completed when targeting beta to a release.
-->

###### Does this feature depend on any specific services running in the cluster?

<!--
Think about both cluster-level services (e.g. metrics-server) as well
as node-level agents (e.g. specific version of CRI). Focus on external or
optional services that are needed. For example, if this feature depends on
a cloud provider API, or upon an external software-defined storage or network
control plane.

For each of these, fill in the following—thinking about running existing user workloads
and creating new ones, as well as about cluster-level services (e.g. DNS):
  - [Dependency name]
    - Usage description:
      - Impact of its outage on the feature:
      - Impact of its degraded performance or high-error rates on the feature:
-->
No, It does not depend on any service running on the cluster, But depends on cAdvisor package to fetch
the machine resource information.

### Scalability

<!--
For alpha, this section is encouraged: reviewers should consider these questions
and attempt to answer them.

For beta, this section is required: reviewers must answer these questions.

For GA, this section is required: approvers should be able to confirm the
previous answers based on experience in the field.
-->

###### Will enabling / using this feature result in any new API calls?

<!--
Describe them, providing:
  - API call type (e.g. PATCH pods)
  - estimated throughput
  - originating component(s) (e.g. Kubelet, Feature-X-controller)
Focusing mostly on:
  - components listing and/or watching resources they didn't before
  - API calls that may be triggered by changes of some Kubernetes resources
    (e.g. update of object X triggers new updates of object Y)
  - periodic API calls to reconcile state (e.g. periodic fetching state,
    heartbeats, leader election, etc.)
-->

No, It won't add/modify any user facing APIs.
The resource managers might need to be updated with new methods to resync their components with updated
machine information.

###### Will enabling / using this feature result in introducing new API types?

<!--
Describe them, providing:
  - API type
  - Supported number of objects per cluster
  - Supported number of objects per namespace (for namespace-scoped objects)
-->
No 
###### Will enabling / using this feature result in any new calls to the cloud provider?

<!--
Describe them, providing:
  - Which API(s):
  - Estimated increase:
-->
No
###### Will enabling / using this feature result in increasing size or count of the existing API objects?

<!--
Describe them, providing:
  - API type(s):
  - Estimated increase in size: (e.g., new annotation of size 32B)
  - Estimated amount of new objects: (e.g., new Object X for every existing Pod)
-->
No
###### Will enabling / using this feature result in increasing time taken by any operations covered by existing SLIs/SLOs?

<!--
Look at the [existing SLIs/SLOs].

Think about adding additional work or introducing new steps in between
(e.g. need to do X to start a container), etc. Please describe the details.

[existing SLIs/SLOs]: https://git.k8s.io/community/sig-scalability/slos/slos.md#kubernetes-slisslos
-->
Negligible, In the case of resource hot plug the resource manager may take some time to resync.
###### Will enabling / using this feature result in non-negligible increase of resource usage (CPU, RAM, disk, IO, ...) in any components?

<!--
Things to keep in mind include: additional in-memory state, additional
non-trivial computations, excessive access to disks (including increased log
volume), significant amount of data sent and/or received over network, etc.
This through this both in small and large cases, again with respect to the
[supported limits].

[supported limits]: https://git.k8s.io/community//sig-scalability/configs-and-limits/thresholds.md
-->
Negligible computational overhead might be introduced into kubelet as it periodically needs to fetch machine information 
from cAdvisor cache and resync all the resource managers with the updated machine information.
###### Can enabling / using this feature result in resource exhaustion of some node resources (PIDs, sockets, inodes, etc.)?

<!--
Focus not just on happy cases, but primarily on more pathological cases
(e.g. probes taking a minute instead of milliseconds, failed pods consuming resources, etc.).
If any of the resources can be exhausted, how this is mitigated with the existing limits
(e.g. pods per node) or new limits added by this KEP?

Are there any tests that were run/should be run to understand performance characteristics better
and validate the declared limits?
-->
Yes, It could.
Since the nodes computational capacity is increased dynamically there might be more pods scheduled on the node.
This is however be mitigated by maxPods kubelet configuration that limits the number of pods on a node.

### Troubleshooting

<!--
This section must be completed when targeting beta to a release.

For GA, this section is required: approvers should be able to confirm the
previous answers based on experience in the field.

The Troubleshooting section currently serves the `Playbook` role. We may consider
splitting it into a dedicated `Playbook` document (potentially with some monitoring
details). For now, we leave it here.
-->

###### How does this feature react if the API server and/or etcd is unavailable?

This feature is node local and mainly handled in kubelet, It has no dependency on etcd.
In case there are pending pods and there is hot plug of resources, The scheduler relies on the API server to fetch node information. 
Without access to the API server, it cannot make scheduling decisions as the node resources are not updated. The pending pods would remain in same condition.

###### What are other known failure modes?

<!--
For each of them, fill in the following information by copying the below template:
  - [Failure mode brief description]
    - Detection: How can it be detected via metrics? Stated another way:
      how can an operator troubleshoot without logging into a master or worker node?
    - Mitigations: What can be done to stop the bleeding, especially for already
      running user workloads?
    - Diagnostics: What are the useful log messages and their required logging
      levels that could help debug the issue?
      Not required until feature graduated to beta.
    - Testing: Are there any tests for failure mode? If not, describe why.
-->

This feature mainly does two things fetch machine information from cAdvisor and reinitialize resource managers.
Failure scenarios can occur in cAdvisor level that is if it wrongly updated with incorrect machine information.


###### What steps should be taken if SLOs are not being met to determine the problem?
If enabling this feature causes performance degradation, its suggested not to hot plug resources and restart the kubelet
to manually to continue operation as before.


## Implementation History

<!--
Major milestones in the lifecycle of a KEP should be tracked in this section.
Major milestones might include:
- the `Summary` and `Motivation` sections being merged, signaling SIG acceptance
- the `Proposal` section being merged, signaling agreement on a proposed design
- the date implementation started
- the first Kubernetes release where an initial version of the KEP was available
- the version of Kubernetes where the KEP graduated to general availability
- when the KEP was retired or superseded
-->

## Drawbacks

<!--
Why should this KEP _not_ be implemented?
-->

## Alternatives

Existing and the alternative to this effort would be restarting the kubelet manually each time after the node resize.

<!--
What other approaches did you consider, and why did you rule them out? These do
not need to be as detailed as the proposal, but should include enough
information to express the idea and why it was not acceptable.
-->