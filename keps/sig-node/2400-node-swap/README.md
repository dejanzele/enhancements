# KEP-2400: Node system swap support

<!-- toc -->
- [Release Signoff Checklist](#release-signoff-checklist)
- [Summary](#summary)
- [Motivation](#motivation)
  - [Scenarios](#scenarios)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Proposal](#proposal)
  - [Enable Swap Support only for Burstable QoS Pods](#enable-swap-support-only-for-burstable-qos-pods)
    - [Set Aside Swap for System Critical Daemon](#set-aside-swap-for-system-critical-daemon)
  - [Steps to Calculate Swap Limit](#steps-to-calculate-swap-limit)
    - [Example](#example)
  - [User Stories](#user-stories)
    - [Improved Node Stability](#improved-node-stability)
    - [Long-running applications that swap out startup memory](#long-running-applications-that-swap-out-startup-memory)
    - [Memory Flexibility](#memory-flexibility)
    - [Local development and systems with fast storage](#local-development-and-systems-with-fast-storage)
    - [Low footprint systems](#low-footprint-systems)
    - [Virtualization management overhead](#virtualization-management-overhead)
  - [Notes/Constraints/Caveats (Optional)](#notesconstraintscaveats-optional)
  - [Risks and Mitigations](#risks-and-mitigations)
    - [Security risk](#security-risk)
- [Design Details](#design-details)
  - [Enabling swap as an end user](#enabling-swap-as-an-end-user)
  - [API Changes](#api-changes)
    - [KubeConfig addition](#kubeconfig-addition)
    - [CRI Changes](#cri-changes)
  - [Test Plan](#test-plan)
      - [Prerequisite testing updates](#prerequisite-testing-updates)
      - [Unit tests](#unit-tests)
      - [Integration tests](#integration-tests)
      - [e2e tests](#e2e-tests)
  - [Graduation Criteria](#graduation-criteria)
    - [Alpha](#alpha)
    - [Alpha2](#alpha2)
    - [Beta 1](#beta-1)
    - [Beta 2](#beta-2)
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
  - [Just set <code>--fail-swap-on=false</code>](#just-set-)
  - [Restrict swap usage at the cgroup level](#restrict-swap-usage-at-the-cgroup-level)
- [Infrastructure Needed (Optional)](#infrastructure-needed-optional)
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
- [ ] (R) Graduation criteria is in place
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

Kubernetes currently does not support the use of [swap
memory](https://en.wikipedia.org/wiki/Paging#Linux) on Linux, as it is
difficult to provide guarantees and account for pod memory utilization when
swap is involved. As part of Kubernetes’ earlier design, [swap support was
considered out of scope](https://github.com/kubernetes/kubernetes/issues/7294).

However, there are a [number of use cases](#user-stories) that would benefit
from Kubernetes nodes supporting swap. Hence, this proposal aims to add swap
support to nodes in a controlled, predictable manner so that Kubernetes users
can perform testing and provide data to continue building cluster capabilities
on top of swap.

## Motivation

There are two distinct types of user for swap, who may overlap:
- node administrators, who may want swap available for node-level performance
  tuning and stability/reducing noisy neighbour issues
- application developers, who have written applications that would benefit from
  using swap memory

There are hence a number of possible ways that one could envision swap use on a
node.

### Scenarios

1. Swap is enabled on a node's host system, but the kubelet does not permit
   Kubernetes workloads to use swap. (This scenario is a prerequisite for the
   following use cases.)
1. Swap is enabled at the node level. The kubelet can permit Kubernetes
   workloads scheduled on the node to use some quantity of swap, depending on
   the configuration.
1. Swap is set on a per-workload basis. The kubelet sets swap limits for each
   individual workload.

This KEP will be limited in scope to the first two scenarios. The third can be
addressed in a follow-up KEP. The enablement work that is in scope for this KEP
will be necessary to implement the third scenario.


### Goals

- On Linux systems, when swap is provisioned and available, Kubelet can start
  up with swap on.
- Configuration is available for kubelet to set swap utilization available to
  Kubernetes workloads, defaulting to 0 swap.
- Cluster administrators can enable and configure kubelet swap utilization on a
  per-node basis.
- Use of swap memory with both cgroupsv1 and cgroupsv2 is supported.

### Non-Goals

- Addressing non-Linux operating systems. Swap support will only be available
  for Linux.
- Provisioning swap. Swap must already be available on the system.
- Setting [swappiness]. This can already be set on a system-wide level outside
  of Kubernetes.
- Allocating swap on a per-workload basis with accounting (e.g. pod-level
  specification of swap). If desired, this should be designed and implemented
  as part of a follow-up KEP. This KEP is a prerequisite for that work. Hence,
  swap will be an overcommitted resource in the context of this KEP.
- Supporting zram, zswap, or other memory types like SGX EPC. These could be
  addressed in a follow-up KEP, and are out of scope.

[swappiness]: https://en.wikipedia.org/wiki/Memory_paging#Swappiness

## Proposal

We propose that, when swap is provisioned and available on a node, cluster
administrators can configure the kubelet such that:

- It can start with swap on.
- It will direct the CRI to allocate Kubernetes workloads 0 swap by default.
- It will have configuration options to configure swap utilization for the
  entire node.

This proposal enables scenarios 1 and 2 above, but not 3.

### Enable Swap Support only for Burstable QoS Pods
Before enabling swap support through the pod API, it is crucial to build confidence in this feature by carefully assessing its impact on workloads and Kubernetes. As an initial step, we propose enabling swap support for Burstable QoS Pods by automatically calculating the appropriate swap values, rather than allowing users to input these values manually. 

Swap access is granted only for pods of Burstable QoS. Guaranteed QoS pods are usually higher-priority pods, therefore we want to avoid swap's performance penalty for them. Best-Effort pods, on the contrary, are low-priority pods that are the first to be killed during node pressures. In addition, they're unpredictable, therefore it's hard to assess how much swap memory is a reasonable amount to allocate for them. 

By doing so, we can ensure a thorough understanding of the feature's performance and stability before considering the manual input of swap values in a subsequent beta release. This cautious approach will ensure the efficient allocation of resources and the smooth integration of swap support into Kubernetes.

Allocate the swap limit equal to the requested memory for each container and adjust the proportion of swap based on the total swap memory available.

#### Set Aside Swap for System Critical Daemon

System critical daemons (such as Kubelet) are essential for node health. Usually, an appropriate portion of system resources (e.g., memory, CPU) is reserved as system reserved. However, swap doesn't inherently support reserving a portion out of the total available. For instance, in the case of memory, we set `memory.min` on the node-level cgroup to ensure an adequate amount of memory is set aside, away from the pods, and for system critical daemons. But there is no equivalent for swap; i.e., no `memory.swap.min` is supported in the kernel. 

Since this proposal advocates enabling swap only for the Burstable QoS pods, this can be done by setting `memory.swap.max` on the cgroups used by the Burstable QoS pods. The value of this `memory.swap.max` can be calculated by:

memory.swap.max = total swap memory available on the system - system reserve (memory)

This is the total amount of swap available for all the Burstable QoS pods; let's call it `TotalPodsSwapAvailable`. This will ensure that the system critical daemons will have access to the swap at least equal to the system reserved memory. This will indirectly act as having support for swap in system reserved.

### Steps to Calculate Swap Limit

1. **Calculate the container's memory proportionate to the node's memory:**
  - Divide the container's memory request by the total node's physical memory. Let's call this value `ContainerMemoryProportion`.
  - If a container is defined with memory requests == memory limits, its `ContainerMemoryProportion` is defined as 0. Therefore, as can be seen below, its overall swap limit is also 0.

2. **Multiply the container memory proportion by the available swap memory for Pods:**
  - Meaning: `ContainerMemoryProportion * TotalPodsSwapAvailable`.

#### Example
Suppose we have a Burstable QoS pod with two containers:

- Container A: Memory request 20 GB
- Container B: Memory request 10 GB

Let's assume the total physical memory is 40 GB and the total swap memory available is also 40 GB. Also assume that the system reserved memory is configured at 2GB, 

Step 1: Determine the containers memory proportion:
- Container A: `20G/40G` = `0.5`.
- Container B: `10G/40G` = `0.25`.

Step 2: Determine swap limitation for the containers:
- Container A: `ContainerMemoryProportion * TotalPodsSwapAvailable` = `0.5 * 38G` = `19G`.
- Container B: `ContainerMemoryProportion * TotalPodsSwapAvailable` = `0.25 * 38G` = `9.5G`.

In this example, Container A would have a swap limit of 19 GB, and Container B would have a swap limit of 9.5 GB.

This approach allocates swap limits based on each container's memory request and adjusts the proportion based on the total swap memory available in the system. It ensures that each container gets a fair share of the swap space and helps maintain resource allocation efficiency.

### User Stories

#### Improved Node Stability

cgroupsv2 improved memory management algorithms, such as oomd, strongly
recommend the use of swap. Hence, having a small amount of swap available on
nodes could improve better resource pressure handling and recovery.

- https://man7.org/linux/man-pages/man8/systemd-oomd.service.8.html
- https://www.kernel.org/doc/html/latest/admin-guide/cgroup-v2.html#id1
- https://chrisdown.name/2018/01/02/in-defence-of-swap.html
- https://media.ccc.de/v/ASG2018-175-oomd
- https://github.com/facebookincubator/oomd/blob/master/docs/production_setup.md#swap

This user story is addressed by scenario 1 and 2, and could benefit from 3.

#### Long-running applications that swap out startup memory

- Applications such as the Java and Node runtimes rely on swap for optimal
  performance
  https://github.com/kubernetes/kubernetes/issues/53533#issue-263475425
- Initialization logic of applications can be safely swapped out without
  affecting long-running application resource usage
  https://github.com/kubernetes/kubernetes/issues/53533#issuecomment-615967154

This user story is addressed by scenario 2, and could benefit from 3.

#### Memory Flexibility

This user story addresses cases in which cost of additional memory is
prohibitive, or elastic scaling is impossible (e.g. on-premise/bare metal
deployments).

- Occasional cron job with high memory usage and lack of swap support means
  cloud nodes must always be allocated for maximum possible memory utilization,
  leading to overprovisioning/high costs
  https://github.com/kubernetes/kubernetes/issues/53533#issuecomment-354832960
- Lack of swap support would require provisioning 3x the amount of memory as
  required with swap
  https://github.com/kubernetes/kubernetes/issues/53533#issuecomment-617654228
- On-premise deployment can’t horizontally scale available memory based on load
  https://github.com/kubernetes/kubernetes/issues/53533#issuecomment-637715138
- Scaling resources is technically feasible but cost-prohibitive, swap provides
  flexibility at lower cost
  https://github.com/kubernetes/kubernetes/issues/53533#issuecomment-553713502

This user story is addressed by scenario 2, and could benefit from 3.

#### Local development and systems with fast storage

Local development or single-node clusters and systems with fast storage may
benefit from using available swap (e.g. NVMe swap partitions, one-node
clusters).

- Single node, local Kubernetes deployment on laptop
  https://github.com/kubernetes/kubernetes/issues/53533#issuecomment-361748518
- Linux has optimizations for swap on SSD, allowing for performance boosts
  https://github.com/kubernetes/kubernetes/issues/53533#issuecomment-589275277

This user story is addressed by scenarios 1 and 2, and could benefit from 3.

#### Low footprint systems

For example, edge devices with limited memory.

- Edge compute systems/devices with small memory footprints (\<2Gi)
  https://github.com/kubernetes/kubernetes/issues/53533#issuecomment-751398086
- Clusters with nodes \<4Gi memory
  https://github.com/kubernetes/kubernetes/issues/53533#issuecomment-751404417

This user story is addressed by scenario 2, and could benefit from 3.

#### Virtualization management overhead

This would apply to virtualized Kubernetes workloads such as VMs launched by
kubevirt.

Every VM comes with a management related overhead which can sporadically be
pretty significant (memory streaming, SRIOV attachment, gpu attachment,
virtio-fs, …). Swap helps to not request much more memory to deal with short
term worst-case scenarios.

With virtualization, clusters are typically provisioned based on the workloads’
memory consumption, and any infrastructure container overhead is overcommitted.
This overhead could be safely swapped out.

- Required for live migration of VMs
  https://github.com/kubernetes/kubernetes/issues/53533#issuecomment-754878431

This user story is addressed by scenario 2, and could benefit from 3.

### Notes/Constraints/Caveats (Optional)

In updating the CRI, we must ensure that container runtime downstreams are able
to support the new configurations.

We considered adding parameters for both per-workload `memory-swap` and
`swappiness`. These are documented as part of the Open Containers [runtime
specification] for Linux memory configuration. Since `memory-swap` is a
per-workload parameter, and `swappiness` is optional and can be set globally,
we are choosing to only expose `memory-swap` which will adjust swap available
to workloads.

Since we are not currently setting `memory-swap` in the CRI, the current
default behaviour when `--fail-swap-on=false` is set is to allocate the same
amount of swap for a workload as memory requested. We will update the default
to not permit the use of swap by setting `memory-swap` equal to `limit`.

[runtime specification]: https://github.com/opencontainers/runtime-spec/blob/1c3f411f041711bbeecf35ff7e93461ea6789220/config-linux.md#memory

### Risks and Mitigations

Having swap available on a system reduces predictability. Swap's performance is
worse than regular memory, sometimes by many orders of magnitude, which can
cause unexpected performance regressions. Furthermore, swap changes a system's
behaviour under memory pressure, and applications cannot directly control what
portions of their memory usage are swapped out. Since enabling swap permits
greater memory usage for workloads in Kubernetes that cannot be predictably
accounted for, it also increases the risk of noisy neighbours and unexpected
packing configurations, as the scheduler cannot account for swap memory usage.

This risk is mitigated by preventing any workloads from using swap by default,
even if swap is enabled and available on a system. This will allow a cluster
administrator to test swap utilization just at the system level without
introducing unpredictability to workload resource utilization.

Additionally, we will mitigate this risk by determining a set of metrics to
quantify system stability and then gathering test and production data to
determine if system stability changes when swap is available to the system
and/or workloads in a number of different scenarios.

Since swap provisioning is out of scope of this proposal, this enhancement
poses low risk to Kubernetes clusters that will not enable swap.

#### Security risk

Enabling swap on a system without encryption poses a security risk, as critical information, such as Kubernetes secrets, may be swapped out to the disk. If an unauthorized individual gains access to the disk, they could potentially obtain these secrets. To mitigate this risk, it is recommended to use encrypted swap. However, handling encrypted swap is not within the scope of kubelet; rather, it is a general OS configuration concern and should be addressed at that level. Nevertheless, it is essential to provide documentation that warns users of this potential issue, ensuring they are aware of the potential security implications and can take appropriate steps to safeguard their system.

To guarantee that system daemons are not swapped, the kubelet must configure the `memory.swap.max` setting to `0` within the system reserved cgroup. Moreover, to make sure that burstable pods are able to utilize swap space, kubelet should verify that the cgroup associated with burstable pods should not be nested under the cgroup designated for system reserved.

Additionally, end user may decide to disable swap completely for a Pod or a container in beta 1 by making Pod guaranteed or set request == limit for a container. This way, there will be no swap enabled for the corresponding containers and there will be no information exposure risks.

## Design Details

We summarize the implementation plan as following:

1. Add a feature gate `NodeSwap` to enable swap support.
1. Leave the default value of kubelet flag `--fail-on-swap` to `true`, to avoid
   changing default behaviour.
1. Introduce a new kubelet config parameter, `MemorySwap`, which configures how
   much swap Kubernetes workloads can use on the node.
1. Introduce a new CRI parameter, `memory_swap_limit_in_bytes`.
1. Ensure container runtimes are updated so they can make use of the new CRI
   configuration.
1. Based on the behaviour set in the kubelet config, the kubelet will instruct
   the CRI on the amount of swap to allocate to each container. The container
   runtime will then write the swap settings to the container level cgroup.

### Enabling swap as an end user

Swap can be enabled as follows:

1. Provision swap on the target worker nodes,
1. Enable the `NodeMemorySwap` feature flag on the kubelet,
1. Set `--fail-on-swap` flag to `false`, and
1. (Optional) Allow Kubernetes workloads to use swap by setting
   `MemorySwap.SwapBehavior=UnlimitedSwap` in the kubelet config.

### API Changes

#### KubeConfig addition

We will add an optional `MemorySwap` value to the `KubeletConfig` struct
in [pkg/kubelet/apis/config/types.go] as follows:

[pkg/kubelet/apis/config/types.go]: https://github.com/kubernetes/kubernetes/blob/6baad0a1d45435ff5844061aebab624c89d698f8/pkg/kubelet/apis/config/types.go#L81

```go
// KubeletConfiguration contains the configuration for the Kubelet
type KubeletConfiguration struct {
	metav1.TypeMeta
...
	// Configure swap memory available to container workloads.
	// +featureGate=NodeSwap
	// +optional
	MemorySwap MemorySwapConfiguration
}

type MemorySwapConfiguration struct {
	// Configure swap memory available to container workloads. May be one of
	// "", "LimitedSwap": workload combined memory and swap usage cannot exceed pod memory limit
	// "UnlimitedSwap": workloads can use unlimited swap, up to the allocatable limit.
	SwapBehavior string
}
```

We want to expose common swap configurations based on the [Docker] and open
container specification for the `--memory-swap` flag. Thus, the
`MemorySwapConfiguration.SwapBehavior` setting will have the following effects:

* If `SwapBehavior` is not set or set to `"LimitedSwap"`, containers do not have
  access to swap beyond their memory limit. This value prevents a container
  from using swap in excess of their memory limit, even if it is enabled on a
  system.
  * With cgroups v1, it is possible for a container to use _some_ swap if its
    combined memory and swap usage do not exceed the
    [`memory.memsw.limit_in_bytes`] limit.
  * With cgroups v2, swap is configured independently from memory. Thus, the
    container runtimes can set [`memory.swap.max`] to 0 in this case, and _no_ swap
    usage will be permitted.
* If `SwapBehavior` is set to `"UnlimitedSwap"`, the container is allowed to
  use unlimited swap, up to the maximum amount available on the host system.

[docker]: https://docs.docker.com/config/containers/resource_constraints/#--memory-swap-details
[`memory.memsw.limit_in_bytes`]: https://www.kernel.org/doc/html/latest/admin-guide/cgroup-v1/memory.html
[`memory.swap.max`]: https://www.kernel.org/doc/html/latest/admin-guide/cgroup-v2.html#memory

#### CRI Changes

The CRI requires a corresponding change in order to allow the kubelet to set
swap usage in container runtimes.  We will introduce a parameter
`memory_swap_limit_in_bytes` to the CRI API (found in
[k8s.io/cri-api/pkg/apis/runtime/v1/api.proto]):

[k8s.io/cri-api/pkg/apis/runtime/v1/api.proto]: https://github.com/kubernetes/kubernetes/blob/6baad0a1d45435ff5844061aebab624c89d698f8/staging/src/k8s.io/cri-api/pkg/apis/runtime/v1/api.proto#L563-L580

```go
// LinuxContainerResources specifies Linux specific configuration for
// resources.
message LinuxContainerResources {
...
    // Memory + swap limit in bytes. Default: 0 (not specified).
    int64 memory_swap_limit_in_bytes = 9;
...
}
```

### Test Plan

<!--
**Note:** *Not required until targeted at a release.*
The goal is to ensure that we don't accept enhancements with inadequate testing.
All code is expected to have adequate tests (eventually with coverage
expectations). Please adhere to the [Kubernetes testing guidelines][testing-guidelines]
when drafting this test plan.
[testing-guidelines]: https://git.k8s.io/community/contributors/devel/sig-testing/testing.md
-->

[X] I/we understand the owners of the involved components may require updates to
existing tests to make this code solid enough prior to committing the changes necessary
to implement this enhancement.

##### Prerequisite testing updates

<!--
Based on reviewers feedback describe what additional tests need to be added prior
implementing this enhancement to ensure the enhancements have also solid foundations.
-->

All existing tests needs to pass with and without swap enabled.

##### Unit tests

<!--
In principle every added code should have complete unit test coverage, so providing
the exact set of tests will not bring additional value.
However, if complete unit test coverage is not possible, explain the reason of it
together with explanation why this is acceptable.
-->

<!--
Additionally, for Alpha try to enumerate the core package you will be touching
to implement this enhancement and provide the current unit coverage for those
in the form of:
- <package>: <date> - <current test coverage>
The data can be easily read from:
https://testgrid.k8s.io/sig-testing-canaries#ci-kubernetes-coverage-unit
This can inform certain test coverage improvements that we want to do before
extending the production code to implement this enhancement.
-->

This KEP introduces minor additions of memory swap controlling configuration parameters.

- Kubelet configuration parameters are tested in the package `k8s.io/kubernetes/pkg/kubelet/apis/config/validation`
- Passing parameters to runtime is tested in `k8s.io/kubernetes/pkg/kubelet/kuberuntime`

Both packages has near 100% coverage and new functionality was covered.

In alpha2, tests will be extended in these packages to support kube-reserved swap settings.


##### Integration tests

<!--
This question should be filled when targeting a release.
For Alpha, describe what tests will be added to ensure proper quality of the enhancement.
For Beta and GA, add links to added tests together with links to k8s-triage for those tests:
https://storage.googleapis.com/k8s-triage/index.html
-->

No new tests added for Alpha and Alpha2 releases.

##### e2e tests

<!--
This question should be filled when targeting a release.
For Alpha, describe what tests will be added to ensure proper quality of the enhancement.
For Beta and GA, add links to added tests together with links to k8s-triage for those tests:
https://storage.googleapis.com/k8s-triage/index.html
We expect no non-infra related flakes in the last month as a GA graduation criteria.
-->

For alpha:

- Swap scenarios are enabled in test-infra for at least two Linux
  distributions. e2e suites will be run against them.
  - Container runtimes must be bumped in CI to use the new CRI.
- Data should be gathered from a number of use cases to guide beta graduation
  and further development efforts.
  - Focus should be on supported user stories as listed above.

Test grid tabs enabled:
- [kubelet-gce-e2e-swap-ubuntu](https://testgrid.k8s.io/sig-node-kubelet#kubelet-gce-e2e-swap-ubuntu): Green
- [kubelet-gce-e2e-swap-ubuntu-serial](https://testgrid.k8s.io/sig-node-kubelet#kubelet-gce-e2e-swap-ubuntu-serial): Green
- [kubelet-gce-e2e-swap-fedora](https://testgrid.k8s.io/sig-node-kubelet#kubelet-gce-e2e-swap-fedora): Degraded
- [kubelet-gce-e2e-swap-fedora-serial](https://testgrid.k8s.io/sig-node-kubelet#kubelet-gce-e2e-swap-fedora-serial): Degraded

No new e2e tests introduced.

For alpha2:

- Add e2e tests that exercise all available swap configurations via the CRI.
- Verify MemoryPressure behavior with swap enabled and document any changes
  for configuring eviction.
- Verify new system-reserved settings for swap memory.

For beta 1:

- Add e2e tests that verify pod-level control of swap utilization.
- Add e2e tests that verify swap performance with pods using a tmpfs.

### Graduation Criteria

#### Alpha

- Kubelet can be started with swap enabled and will support two configurations
  for Kubernetes workloads: `LimitedSwap` and `UnlimitedSwap`.
- Kubelet can configure CRI to allocate swap to Kubernetes workloads. By
  default, workloads will not be allocated any swap.
- e2e test jobs are configured for Linux systems with swap enabled.

#### Alpha2

In alpha2 the focus will be on making sure that the feature can be used on
subset of production scenarios to collect more feedback before entering beta.
Specifically, security and test coverage will be increased. As well as the new
setting that will split swap between kubelet and workload will be introduced.

Once functionality part is resolved while in alpha, beta will be more about
performance and feedback on wider range of scenarios.

This will allow to collect feedback from the following scenarios reasonably safe:

- on cgroupv2: allow host system processes to use swap to increase
  system reliability under memory pressure.
- enable swap for the workload in "single large pod per node" scenarios.

Here are specific improvements to be made:

- Address swap impact on memory-backed volumes: https://github.com/kubernetes/kubernetes/issues/105978.
- Investigate swap security when enabling on system processes on the node.
- Improve coverage for appropriate scenarios in testgrid.
- Add the ability to set a system-reserved quantity of swap from what kubelet
  detects on the host.
- Consider introducing new configuration modes for swap, such as a node-wide
  swap limit for workloads.
- Investigate eviction behavior with swap enabled.

#### Beta 1
- Enable Swap Support using Burstable QoS Pods only. 
- Enable Swap Support for Cgroup v2 Only.
- Add swap memory to the Kubelet stats api.
- Determine a set of metrics for node QoS in order to evaluate the performance
  of nodes with and without swap enabled.
- Make sure node e2e jobs that use swap are healthy
- Improve coverage for appropriate scenarios in testgrid.

#### Beta 2
- Publish a Kubernetes doc page encoring user to use encrypted swap if they wish to enable this feature.
- Add [swap specific tests](https://github.com/kubernetes/kubernetes/issues/120798) such as, handling the usage of
  swap during container restart boundaries for writes to tmpfs (which may require pod cgroup change beyond what
  container runtime will do at (container cgroup boundary).
- Fix flaking/failing swap node e2e jobs.
- Address eviction related [issue](https://github.com/kubernetes/kubernetes/issues/120800) in swap implementation.

[via cgroups]: #restrict-swap-usage-at-the-cgroup-level

#### GA

_(Tentative.)_

- Test a wide variety of scenarios that may be affected by swap support.
- Remove feature flag.
- Remove the Swap Support using Burstable QoS Pods only deprecated in Beta 2. 


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

No changes are required on upgrade to maintain previous behaviour.

It is possible to downgrade a kubelet on a node that was using swap, but this
would require disabling the use of swap and setting `swapoff` on the node.

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

Feature flag will apply to kubelet only, so version skew strategy is N/A.

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
-->

- [x] Feature gate (also fill in values in `kep.yaml`)
  - Feature gate name: NodeSwap
  - Components depending on the feature gate: API Server, Kubelet
- [x] Other
  - Describe the mechanism: `--fail-swap-on=false` flag for kubelet must also
    be set at kubelet start
  - Will enabling / disabling the feature require downtime of the control
    plane? Yes. Flag must be set on kubelet start. To disable, kubelet must be
    restarted. Hence, there would be brief control component downtime on a
    given node.
  - Will enabling / disabling the feature require downtime or reprovisioning
    of a node?
    Yes. See above; disabling would require brief node downtime.

###### Does enabling the feature change any default behavior?

<!--
Any change of default behavior may be surprising to users or break existing
automations, so be extremely careful here.
-->

No. If the feature flag is enabled, the user must still set
`--fail-swap-on=false` to adjust the default behaviour.

A node must have swap provisioned and available for this feature to work. If
there is no swap available, but the feature flag is set to true, there will
still be no change in existing behaviour.

###### Can the feature be disabled once it has been enabled (i.e. can we roll back the enablement)?

<!--
Describe the consequences on existing workloads (e.g., if this is a runtime
feature, can it break the existing applications?).

NOTE: Also set `disable-supported` to `true` or `false` in `kep.yaml`.
-->

No. The feature flag can be disabled while the `--fail-swap-on=false` flag is
set, but this would result in undefined behaviour.

To turn this off, the kubelet would need to be restarted. If a cluster admin
wants to disable swap on the node without repartitioning the node, they could
stop the kubelet, set `swapoff` on the node, and restart the kubelet with
`--fail-swap-on=true`. The setting of the feature flag will be ignored in this
case.

###### What happens if we reenable the feature if it was previously rolled back?

N/A

###### Are there any tests for feature enablement/disablement?

<!--
The e2e framework does not currently support enabling or disabling feature
gates. However, unit tests in each component dealing with managing data, created
with and without the feature, are necessary. At the very least, think about
conversion tests if API types are being modified.
-->

N/A. This should be tested separately for scenarios with the flag enabled and
disabled.

### Rollout, Upgrade and Rollback Planning

<!--
This section must be completed when targeting beta to a release.
-->

###### How can a rollout fail? Can it impact already running workloads?

<!--
Try to be as paranoid as possible - e.g., what if some components will restart
mid-rollout?
-->

If a new node with swap memory fails to come online, it will not impact any
running components.

It is possible that if a cluster administrator adds swap memory to an already
running node, and then performs an in-place upgrade, the new kubelet could fail
to start unless the configuration was modified to tolerate swap. However, we
would expect that if a cluster admin is adding swap to the node, they will also
update the kubelet's configuration to not fail with swap present.

Generally, it is considered best practice to add a swap memory partition at
node image/boot time and not provision it dynamically after a kubelet is
already running and reporting Ready on a node.

###### What specific metrics should inform a rollback?

<!--
What signals should users be paying attention to when the feature is young
that might indicate a serious problem?
-->

Workload churn or performance degradations on nodes. The metrics will be
application/use-case specific, but we can provide some suggestions, based on
the stability metrics identified earlier.

###### Were upgrade and rollback tested? Was the upgrade->downgrade->upgrade path tested?

<!--
Describe manual testing that was done and the outcomes.
Longer term, we may want to require automated upgrade/rollback tests, but we
are missing a bunch of machinery and tooling and can't do that now.
-->

N/A because swap support lacks a runtime upgrade/downgrade path; kubelet must
be restarted with or without swap support.

###### Is the rollout accompanied by any deprecations and/or removals of features, APIs, fields of API types, flags, etc.?

<!--
Even if applying deprecation policies, they may still surprise some users.
-->

No.

### Monitoring Requirements

<!--
This section must be completed when targeting beta to a release.
-->

###### How can someone using this feature know that it is working for their instance?

1. Kubelet stats API will be extended to show swap usage details.

###### How can an operator determine if the feature is in use by workloads?

<!--
Ideally, this should be a metric. Operations against the Kubernetes API (e.g.,
checking if there are objects with field X set) may be a last resort. Avoid
logs or events for this purpose.
-->

KubeletConfiguration has set `failOnSwap: false`.

The prometheus `node_exporter` will also export stats on swap memory
utilization.

###### What are the SLIs (Service Level Indicators) an operator can use to determine the health of the service?

<!--
Pick one more of these and delete the rest.
-->

TBD. We will determine a set of metrics as a requirement for beta graduation.
We will need more production data; there is not a single metric or set of
metrics that can be used to generally quantify node performance.

This section to be updated before the feature can be marked as graduated, and
to be worked on during 1.23 development.

We will also add swap memory utilization to the Kubelet stats API, to provide a means of monitoring this beyond cadvisor Prometheus stats.

- [ ] Metrics
  - Metric name:
  - [Optional] Aggregation method:
  - Components exposing the metric:
- [ ] Other (treat as last resort)
  - Details:

###### What are the reasonable SLOs (Service Level Objectives) for the above SLIs?

<!--
At a high level, this usually will be in the form of "high percentile of SLI
per day <= X". It's impossible to provide comprehensive guidance, but at the very
high level (needs more precise definitions) those may be things like:
  - per-day percentage of API calls finishing with 5XX errors <= 1%
  - 99% percentile over day of absolute value from (job creation time minus expected
    job creation time) for cron job <= 10%
  - 99,9% of /health requests per day finish with 200 code
-->

N/A

###### Are there any missing metrics that would be useful to have to improve observability of this feature?

<!--
Describe the metrics themselves and the reasons why they weren't added (e.g., cost,
implementation difficulties, etc.).
-->

N/A

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

No.

### Scalability

<!--
For alpha, this section is encouraged: reviewers should consider these questions
and attempt to answer them.

For beta, this section is required: reviewers must answer these questions.

For GA, this section is required: approvers should be able to confirm the
previous answers based on experience in the field.
-->

###### Can enabling / using this feature result in resource exhaustion of some node resources (PIDs, sockets, inodes, etc.)?

No.

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

No.

###### Will enabling / using this feature result in introducing new API types?

<!--
Describe them, providing:
  - API type
  - Supported number of objects per cluster
  - Supported number of objects per namespace (for namespace-scoped objects)
-->

No.

###### Will enabling / using this feature result in any new calls to the cloud provider?

<!--
Describe them, providing:
  - Which API(s):
  - Estimated increase:
-->

No.

###### Will enabling / using this feature result in increasing size or count of the existing API objects?

<!--
Describe them, providing:
  - API type(s):
  - Estimated increase in size: (e.g., new annotation of size 32B)
  - Estimated amount of new objects: (e.g., new Object X for every existing Pod)
-->

The KubeletConfig API object may slightly increase in size due to new config
fields.

###### Will enabling / using this feature result in increasing time taken by any operations covered by existing SLIs/SLOs?

<!--
Look at the [existing SLIs/SLOs].

Think about adding additional work or introducing new steps in between
(e.g. need to do X to start a container), etc. Please describe the details.

[existing SLIs/SLOs]: https://git.k8s.io/community/sig-scalability/slos/slos.md#kubernetes-slisslos
-->

It is possible for this feature to affect performance of some worker node-level
SLIs/SLOs. We will need to monitor for differences, particularly during beta
testing, when evaluating this feature for beta and graduation.

###### Will enabling / using this feature result in non-negligible increase of resource usage (CPU, RAM, disk, IO, ...) in any components?

<!--
Things to keep in mind include: additional in-memory state, additional
non-trivial computations, excessive access to disks (including increased log
volume), significant amount of data sent and/or received over network, etc.
This through this both in small and large cases, again with respect to the
[supported limits].

[supported limits]: https://git.k8s.io/community//sig-scalability/configs-and-limits/thresholds.md
-->

Yes. It will permit the utilization of swap memory (i.e. disk) on nodes. This
is expected, as this enhancement is enabling cluster administrators to access
this resource.

### Troubleshooting

<!--
This section must be completed when targeting beta to a release.

The Troubleshooting section currently serves the `Playbook` role. We may consider
splitting it into a dedicated `Playbook` document (potentially with some monitoring
details). For now, we leave it here.
-->

###### How does this feature react if the API server and/or etcd is unavailable?

No change. Feature is specific to individual nodes.

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


Individual nodes with swap memory enabled may experience performance
degradations under load. This could potentially cause a cascading failure on
nodes without swap: if nodes with swap fail Ready checks, workloads may be
rescheduled en masse.

Thus, cluster administrators should be careful while enabling swap. To minimize
disruption, you may want to taint nodes with swap available to protect against
this problem. Taints will ensure that workloads which tolerate swap will not
spill onto nodes without swap under load.

###### What steps should be taken if SLOs are not being met to determine the problem?

It is suggested that if nodes with swap memory enabled cause performance or
stability degradations, those nodes are cordoned, drained, and replaced with
nodes that do not use swap memory.

## Implementation History

- **2015-04-24:** Discussed in [#7294](https://github.com/kubernetes/kubernetes/issues/7294).
- **2017-10-06:** Discussed in [#53533](https://github.com/kubernetes/kubernetes/issues/53533).
- **2021-01-05:** Initial design discussion document for swap support and use cases.
- **2021-04-05:** Alpha KEP drafted for initial node-level swap support and implementation (KEP-2400).
- **2021-08-09:** New in Kubernetes v1.22: alpha support for using swap memory: https://kubernetes.io/blog/2021/08/09/run-nodes-with-swap-alpha/

## Drawbacks

<!--
Why should this KEP _not_ be implemented?
-->

When swap is enabled, particularly for workloads, the kubelet’s resource
accounting may become much less accurate. This may make cluster administration
more difficult and less predictable.

Currently, there exists an unsupported workaround, which is setting the kubelet
flag `--fail-swap-on` to false.

## Alternatives

### Just set `--fail-swap-on=false`

This is insufficient for most use cases because there is inconsistent control
over how swap will be used by various container runtimes. Dockershim currently
sets swap available for workloads to 0. The CRI does not restrict it at all.
This inconsistency makes it difficult or impossible to use swap in production,
particularly if a user wants to restrict workloads from using swap when using
the CRI rather than dockershim.

### Restrict swap usage at the cgroup level

Setting a swap limit at the cgroup level would allow us to restrict the usage
of swap on a pod-level, rather than container-level basis.

For alpha, we are opting for the container-level basis to simplify the
implementation (as the container runtimes already support configuration of swap
with the `memory-swap-limit` parameter). This will also provide the necessary
plumbing for container-level accounting of swap, if that is proposed in the
future.

In beta, we may want to revisit this.

See the [Pod Resource Management design proposal] for more background on the
cgroup limits the kubelet currently sets based on each QoS class.

[Pod Resource Management design proposal]: https://github.com/kubernetes/design-proposals-archive/blob/master/node/pod-resource-management.md#pod-level-cgroups

## Infrastructure Needed (Optional)

<!--
Use this section if you need things from the project/SIG. Examples include a
new subproject, repos requested, or GitHub details. Listing these here allows a
SIG to get the process for these resources started right away.
-->

We may need Linux VM images built with swap partitions for e2e testing in CI.
