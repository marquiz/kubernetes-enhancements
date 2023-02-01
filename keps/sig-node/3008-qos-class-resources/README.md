<!--
**Note:** When your KEP is complete, all of these comment blocks should be removed.

To get started with this template:

- [ ] **Pick a hosting SIG.**
  Make sure that the problem space is something the SIG is interested in taking
  up. KEPs should not be checked in without a sponsoring SIG.
- [ ] **Create an issue in kubernetes/enhancements**
  When filing an enhancement tracking issue, please make sure to complete all
  fields in that template. One of the fields asks for a link to the KEP. You
  can leave that blank until this KEP is filed, and then go back to the
  enhancement and add the link.
- [ ] **Make a copy of this template directory.**
  Copy this template into the owning SIG's directory and name it
  `NNNN-short-descriptive-title`, where `NNNN` is the issue number (with no
  leading-zero padding) assigned to your enhancement above.
- [ ] **Fill out as much of the kep.yaml file as you can.**
  At minimum, you should fill in the "Title", "Authors", "Owning-sig",
  "Status", and date-related fields.
- [ ] **Fill out this file as best you can.**
  At minimum, you should fill in the "Summary" and "Motivation" sections.
  These should be easy if you've preflighted the idea of the KEP with the
  appropriate SIG(s).
- [ ] **Create a PR for this KEP.**
  Assign it to people in the SIG who are sponsoring this process.
- [ ] **Merge early and iterate.**
  Avoid getting hung up on specific details and instead aim to get the goals of
  the KEP clarified and merged quickly. The best way to do this is to just
  start with the high-level sections and fill out details incrementally in
  subsequent PRs.

Just because a KEP is merged does not mean it is complete or approved. Any KEP
marked as `provisional` is a working document and subject to change. You can
denote sections that are under active debate as follows:

```
<<[UNRESOLVED optional short context or usernames ]>>
Stuff that is being argued.
<<[/UNRESOLVED]>>
```

When editing KEPS, aim for tightly-scoped, single-topic PRs to keep discussions
focused. If you disagree with what is already in a document, open a new PR
with suggested changes.

One KEP corresponds to one "feature" or "enhancement" for its whole lifecycle.
You do not need a new KEP to move from beta to GA, for example. If
new details emerge that belong in the KEP, edit the KEP. Once a feature has become
"implemented", major changes should get new KEPs.

The canonical place for the latest set of instructions (and the likely source
of this file) is [here](/keps/NNNN-kep-template/README.md).

**Note:** Any PRs to move a KEP to `implementable`, or significant changes once
it is marked `implementable`, must be approved by each of the KEP approvers.
If none of those approvers are still appropriate, then changes to that list
should be approved by the remaining approvers and/or the owning SIG (or
SIG Architecture for cross-cutting KEPs).
-->
# [KEP-3008](#3008): QoS-class resources

<!-- toc -->
- [Release Signoff Checklist](#release-signoff-checklist)
- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Implementation phases](#implementation-phases)
  - [Phase 1](#phase-1)
  - [Future work](#future-work)
    - [Update sandbox-level QoS-class resources](#update-sandbox-level-qos-class-resources)
    - [In-place pod vertical scaling](#in-place-pod-vertical-scaling)
    - [Access control](#access-control)
- [Proposal](#proposal)
  - [User Stories (Optional)](#user-stories-optional)
    - [Story 1](#story-1)
    - [Story 2](#story-2)
    - [Story 3](#story-3)
    - [Story 4](#story-4)
  - [Notes/Constraints/Caveats (Optional)](#notesconstraintscaveats-optional)
  - [Risks and Mitigations](#risks-and-mitigations)
- [Design Details](#design-details)
  - [CRI API](#cri-api)
    - [ContainerConfig](#containerconfig)
    - [UpdateContainerResourcesRequest](#updatecontainerresourcesrequest)
    - [PodSandboxConfig](#podsandboxconfig)
    - [ContainerStatus](#containerstatus)
    - [RuntimeStatus](#runtimestatus)
    - [Consts](#consts)
  - [Kubernetes API](#kubernetes-api)
    - [PodSpec](#podspec)
    - [NodeStatus](#nodestatus)
  - [Kubelet](#kubelet)
  - [API server](#api-server)
  - [Scheduler](#scheduler)
  - [Kubectl](#kubectl)
  - [Container runtimes](#container-runtimes)
  - [Open Questions](#open-questions)
    - [Pod QoS class](#pod-qos-class)
    - [Default class](#default-class)
  - [Test Plan](#test-plan)
      - [Prerequisite testing updates](#prerequisite-testing-updates)
      - [Unit tests](#unit-tests)
      - [Integration tests](#integration-tests)
      - [e2e tests](#e2e-tests)
  - [Graduation Criteria](#graduation-criteria)
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
  - [Pod annotations](#pod-annotations)
    - [Kubelet](#kubelet-1)
    - [API server](#api-server-1)
    - [Class capacity with extended resources](#class-capacity-with-extended-resources)
  - [RDT-only](#rdt-only)
  - [Widen the scope](#widen-the-scope)
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

Add support to Kubernetes for declaring _quality-of-service_ resources, and
assigning these to Pods. A quality-of-service (QoS-class) resource is similar
to other Kubernetes resource types (i.e. native resources such as `cpu` and
`memory` or extended resources) because you can assign that resource to a
particular container. However, QoS-class resources are also different from
those other resources because they are used to assign a _class identifier_,
rather than to declare a specific amount of capacity that is allocated.

Main characteristics of the new resource type (and the technologies they are
aimed at enabling) are:

- multiple containers can be assigned to the same class of a certain type of
  QoS-class resource
- QoS-class resources are represented by an enumerable set of class identifiers
- each type of QoS-class resource has an independent set of class identifiers

With QoS-class resources, Pods and their containers can request opaque
QoS-class identifiers (classes) of certain QoS mechanism (QoS-class resource
type). Kubelet relays this information to the container runtime, without
directing how the request is enforced in the underlying system. Being opaque to
Kubernetes means that QoS-class resources have to be supported by the container
runtime as it is responsible for the actual low-level management of them.

## Motivation

This enhancement proposal aims at improving the quality of service of
applications by making available a new type of resource control mechanism in
Kubernetes. Certain types of resources are inherently shared by applications,
e.g. cache, memory bandwidth and disk I/O. While there are technologies for
controlling how these resources are shared between applications, there is
currently no meaningful way to support these technologies in Kubernetes. This
proposal suggests to address the issue above in a generalized way by extending
the Kubernetes resource model with a new type of resources, i.e. QoS-class
resources.

This KEP identifies two technologies that can immediately be enabled with
QoS-class resources. However, these are just two examples  and the proposed
changes are generic (and not tied to these two QoS-class resource types in any
way), making it easier to implement new QoS-class resource types.

[Intel RDT][intel-rdt] implements a class-based mechanism for controlling the
cache and memory bandwidth QoS of applications. All processes in the same
hardware class share a portion of cache lines and memory bandwidth. RDT
provides a way for mitigating noisy neighbors and fulfilling SLAs. In Linux
control happens via resctrl -- a pseudo-filesystem provided by the kernel which
makes it virtually agnostic of the hardware architecture. The OCI runtime-spec
has supported Intel RDT for a while already. Other hardware vendors have
comparable technologies which use the same [resctrl interface][linux-resctrl].

The Linux Block IO controller parameters depend very heavily on the underlying
hardware and system configuration (device naming/numbering, IO scheduler
configuration etc) which makes it very impractical to control from the Pod spec
level. In order to hide this complexity the concept of blockio classes has been
added to the container runtimes (CRI-O and containerd). A system administrator
is able to configure blockio controller parameters on per-class basis and the
classes are then made available for CRI clients. Following this model also
provides a possible framework for the future improvements, for instance
enabling class-based network or memory type prioritization of applications.

Currently, there is no mechanism in Kubernetes to use these types of resources.
CRI-O and containerd runtimes have support for RDT and blockio classes and they
provide an bridge-gap user interface through special pod annotations. The goal
is to get these types of resources first class citizens and properly supported
in Kubernetes, providing visibility, a well-defined user interface, and
permission controls.

It seems necessary to support both container-level and pod-level QoS-class
resources as independent concepts. Intel RDT (above) is per-container by
design because of the hardware implementation (the control/class hiearchy is
flat). Also, the current support for blockio is container-level only (it is not
possible to configure pod sandbox-level cgroup parameters). However, having
pod-level QoS-class resources makes it possible to implement support for
sandbox-level blockio parameters. Other usage for pod sandbox-level QoS-class
resources would be communicating the Kubernetes Pod QoS class from
kubelet to the container runtime.

### Goals

- Make it possible to request QoS-class resources
  - Support RDT class assignment of containers. This is already supported by
    the containerd and CRI-O runtime and part of the OCI runtime-spec
  - Support blockio class assignment of containers.
  - Support Pod-level (sandbox-level) QoS-class resources
- Make the API to support updating QoS-class resource assignment of running containers
- Make the extensions flexible, enabling simple addition of other QoS-class
  resource types in the future.
- Make QoS-class resources opqaue (as possible) to the CRI client
- Discovery of the available QoS-class resources
- API changes to support updating Pod-level (sandbox-level) QoS-class resource
  assignment of running pods
- Resource status/capacity
- Access control ([future work](#future-work))

### Non-Goals

- Interface or mechanism for configuring the QoS-class resources (responsibility of
  the container runtime).
- Enumerating possible (QoS-class) resource types or their detailed behavior

## Implementation phases

This proposal splits the full implementation of QoS-class resources into
multiple phases, building functionality gradually. The goal is to
make the discussions more focused and easier. We may also learn on the way,
insights from earlier phases affecting design choices made in the later phases,
hopefully resulting in a better overall end result. However, we also outline
all the future steps to not lose the overall big picture.

The figure below illustrates the design of implementation phase 1. The first
implementation phase covers the communication between kubelet and the container
runtime (i.e. CRI API) and Kubernetes API for NodeStatus and PodSpec and their
respecive implementations (kubelet and kube-scheduler). Access control (e.g.
resource quota) is left to future work. This KEP (the [Proposal](#proposal) and
[Design Details](#design-details)) in its current form implements this first
phase.

![design](./design.svg)

QoS-class resources are designed to be opaque to the CRI client in the sense
that the container runtime takes care of configuration and control of the
resources and the classes within.

### Phase 1

The goal is to enable a functional base for users to leverage QoS-class
resources and start experimenting with them in Kubernetes:

- extend the CRI protocol to allow QoS-class resource assignment and updates to
  be communicated from kubelet to the runtime
- extend the CRI protocol to allow runtime to communicate available QoS-class
  resources (the types of resources and the classes within) to kubelet
- introduce a feature gate for enabling QoS-class resource support in kubelet
- extend PodSpec to support assignment of QoS-clsss resources
- extend NodeStatus to show availability/capacity of QoS-clsss resources on a
  node

### Future work

This section sheds light on the end goal of this work in order to better
evaluate this KEP in a broader context. What a fully working solution would
consists of and what the (next) steps to accomplish that would be. These topics
are currently listed as "future work" in [Goals](#goals).

In practice, the future work mostly consists of changes to the Kubernetes API
and control plane components.

#### Update sandbox-level QoS-class resources

This future step would be a second extension to the CRI API.

Currently there is no endpoint in the CRI API to update the configuration of
pod sandboxes. In contrast, container-level resources can be updated with the
UpdateContainerResources API endpoint. In order to make container and pod
(sandbox) level QoS-class resources symmetric we want to make it possible to
update of pod-level resource assignments, too.

This will likely require a new API endpoint in CRI:

```diff
@@ -38,6 +38,8 @@ service RuntimeService {
     // RunPodSandbox creates and starts a pod-level sandbox. Runtimes must ensure
     // the sandbox is in the ready state on success.
     rpc RunPodSandbox(RunPodSandboxRequest) returns (RunPodSandboxResponse) {}
+    // UpdatePodSandboxConfig updates the configuration of an existing pod sandbox.
+    rpc UpdatePodSandboxConfig(UpdatePodSandboxConfigŔequest) returns (UpdatePodSandboxConfigŔesponse) {}
```

#### In-place pod vertical scaling

Properly integrating QoS-class resources to
[In-place pod vertical scaling](#1287) will allow update of running containers.
However, some QoS-class resources may be immutable, meaning the in-place
updates are not possible.

#### Access control

This future step adds support for controlling the access to available QoS-class
resources.

Access control could be achieved e.g. by extending ResourceQuotaSpec which
would implement restrictions based on the namespace.

```diff
 // ResourceQuotaSpec defines the desired hard limits to enforce for Quota.
 type ResourceQuotaSpec struct {
     // hard is the set of desired hard limits for each named resource.
     Hard ResourceList
     // A collection of filters that must match each object tracked by a quota.
     // If not specified, the quota matches all objects.
     Scopes []ResourceQuotaScope 
     // scopeSelector is also a collection of filters like scopes that must match each
     // object tracked by a quota but expressed using ScopeSelectorOperator in combination
     // with possible values.
     ScopeSelector *ScopeSelector
+    // QoSResources contains the desired set of allowed class resources.
+    // +featureGate=QoSResources
+    // +optional
+    QoSResources QoSResourceQuota
 }

+// QoSResourceQuota contains the allowed class resources.
+type QoSResourceQuota struct {
+       // Pod contains the allowed class resources for pods.
+       // +optional
+       Pod []AllowedQoSResource
+       // Container contains the allowed class resources for pods.
+       // +optional
+       Container []AllowedQoSResource
+}

+// AllowedQoSResource specifies access to one QoS-class resources type.
+type AllowedQoSResource struct {
+       // Name of the resource.
+       Name QoSResourceName
+       // Allowed classes.
+       Classes []string
+       // Capacity is the hard limit for usage of the class.
+       // +optional
+       Capacity int64
+}


 // ResourceQuotaStatus defines the enforced hard limits and observed use.
 type ResourceQuotaStatus struct {
        ...
        // Used is the current observed total usage of the resource in the namespace
        // +optional
        Used ResourceList
+       // QoSResources contains the enforced set of available class resources.
+       // +featureGate=QoSResources
+       // +optional
+       QoSResources QoSResourceQuota
 }


```
## Proposal

This section currently covers [implementation phase 1](#phase-1) (see
[implementation phases](#implementation-phases) for an outline of the complete
implementation).

We extend the CRI protocol to contain information about the QoS-class
resource assignment of containers and pods. Resource assignment requests will
be simple key-value pairs (*resource-type=class-name*)

We also extend the CRI protocol to support updates of QoS-class resource
assignment of running containers. We recognize that currently container
runtimes lack the capability to update either of the two types of QoS-class
resources we have identified (RDT and blockio). However, there is no technical
limitation in that and we are planning to implement update support for them
in the future.

We also extend the Kubernetes API to support the assignment of QoS-class
resources to applications via PodSpec, and to users and the kube-scheduler to
see availability of QoS-class resources on a per-node basis via NodeStatus.

Container runtime is expected to be aware of all resource types and the classes
within. The CRI protocol is extended to be able to communicate the available
QoS-class resources from the runtime to the client. This information includes:
- Available QoS-class resource types.
- Available classes within each resource type.
- Whether the resource type is immutable or if it supports in-place updates.
  In-place updates of resoures might not be possible because of runtime
  limitations or the underlying technology, for example.

QoS-class resources may be bounded in the way that the number of applications
that can be assigned to a specific class (of one QoS-class resource) on a node
can be limited. This limit is configuratble on a per-class (and per-node)
basis. This can be used to e.g. limit access to a high-tier class.

Pod-level and container-level QoS-class resources are completely independent
resource types. E.g. specifying something in the pod-level request does not
mean specifying a pod-level default for all containers of the pod.

Currently we identify two types of container-level QoS-class resources (RDT and
blockio) but the API changes will be generic so that it will serve other
similar resources in the future. Currently there are no immediately enabled
pod-level QoS-class resources but we see usage scenarios for those in the
future (communicating the pod QoS class to the runtime and enabling pod-level
cgroup controls for blockio).

We introduce a feature gate that enables Kubernetes components (kubelet,
kube-apiserver, kube-scheduler) to support QoS-class resources.

### User Stories (Optional)

#### Story 1

As a user I want to minimize the interference of other applications to my
workload by assigning it to a class with exclusive cache allocation.

#### Story 2

As a user I want to make sure my low-priority, I/O-intensive background task
will not disturb more important workloads running on the same node.

#### Story 3

As a cluster administrator I want to throttle I/O bandwidths of certain
DaemonSets, and I want that exact throttling values depend on the SSD model in
my heterogenous cluster.

#### Story 4

As a user I want to assign a low priority task into an (RDT) class that limits
the available memory bandwidth.

### Notes/Constraints/Caveats (Optional)

The proposal only describes the details of implementation Phase 1 and some
parts like permission control are not covered. The
risk in this piecemeal approach is finding devil in the details, resulting in
inconsistent and/or crippled and/or cumbersome end result. However, the missing
parts are already outlined in [future work](#future-work) and there is aa lot
of experience in extending the API and understanding which sort of solutions
are functional and practical, mitigating the problem.

### Risks and Mitigations

<!--
What are the risks of this proposal, and how do we mitigate? Think broadly.
For example, consider both security and how this will impact the larger
Kubernetes ecosystem.

How will security be reviewed, and by whom?

How will UX be reviewed, and by whom?

Consider including folks who also work outside the SIG or subproject.
-->

- User assigning container to “unauthorized” class, causing interference and
  access to unwanted set/amount of resources. This will be addressed in future
  KEP introducing permission controls.

## Design Details

The detailed design presented here covers [implementation phase 1](#phase-1)
(see [implementation phases](#implementation-phases) for an outline of all
planned changes).

Configuration and management of the QoS-class resources is fully handled by the
underlying container runtime and is invisible to kubelet. An error to the CRI
client is returned if the specified class is not available.

### CRI API

The following additions to the CRI protocol are suggested.

#### ContainerConfig

The `ContainerConfig` message will be supplemented with new `class_resources`
field, providing per-container setting for QoS-class resources. This will be
used in `CreateContainerRequest` to communicate the container-level QoS-class
resource assignments to the runtime.

```diff
 message ContainerConfig {

 ...
     // Configuration specific to Linux containers.
     LinuxContainerConfig linux = 15;
     // Configuration specific to Windows containers.
     WindowsContainerConfig windows = 16;
+
+    // Configuration of QoS resources.
+    ContainerQoSResources qos_resources = 17;
 }

+// ContainerQoSResources specifies the configuration of QoS resources of a
+// container.
+message ContainerQoSResources {
+    // QoS resources the container will be assigned to.
+    // Key-value pairs where key is name of the QoS resource and value is the
+    // name of the class.
+    map<string, string> classes = 1;
+}
```

#### UpdateContainerResourcesRequest

Similar to `CreateContainerRequest`, the `UpdateContainerResourcesRequest`
message will extended to allow updating of QoS-class resource configuration of
a running container.  Depending on runtime-level support of a particular
resource (and possibly the type of resource) UpdateContainerResourcesRequest
might fail. Resource discovery (see [Runtime status](#runtime-status)) has
the capability to distinguish immutable resource types.

Note that neither of the existing QoS-class resource types (RDT or blockio)
support updates because of runtime limitations, yet.

```diff
 message UpdateContainerResourcesRequest {

 ...
     // resources to update or other options to use when updating the container.
     map<string, string> annotations = 4;
+    // Configuration of QoS resources.
+    ContainerQoSResources qos_resources = 5;
}
```

#### PodSandboxConfig

The `PodSandboxConfig` will be supplemented with a new `class_resources` field
that specifies the assignment of pod-level QoS-class resources. The intended
use for this would be to be able to communicate pod-level QoS-class resource
assignments at sandbox creation time (`RunPodSandboxRequest`).

```diff
 message PodSandboxConfig {
@@ -45,5 +45,14 @@ message PodSandboxConfig {
     LinuxPodSandboxConfig linux = 8;
     // Optional configurations specific to Windows hosts.
     WindowsPodSandboxConfig windows = 9;
+    // Configuration of QoS resources.
+    PodQoSResources qos_resources = 10;
 }

+// PodQoSResources specifies the configuration of QoS resources of a pod.
+message PodQoSResources {
+    // QoS resources the pod will be assigned to.
+    // Key-value pairs where key is name of the QoS resource and value is the
+    // name of the class.
+    map<string, string> classes = 1;
+}
```

#### ContainerStatus

The `ContainerResources` message (part of `ContainerStatus`) will be extended
to report back QoS-class resource assignments of a container, similar to other
resources.

```diff
@@ -1251,6 +1269,8 @@ message ContainerResources {
     LinuxContainerResources linux = 1;
     // Resource limits configuration specific to Windows container.
     WindowsContainerResources windows = 2;
+    // Configuration of QoS resources.
+    ContainerQoSResources qos_resources = 3;
```

#### RuntimeStatus

Extend the `RuntimeStatus` message with new `resources` field that is used to
communicate the available QoS-class resources from the runtime to the client.

This information can be used by the client (kubelet) to validate QoS-class
resource assignments before starting a pod. In future steps kubelet will patch
this information into node status.

```diff
 message RuntimeStatus {
     // List of current observed runtime conditions.
     repeated RuntimeCondition conditions = 1;
+    // Information about the discovered resources
+    ResourcesInfo resources = 2;
+}

+// ResourcesInfo contains information about the resources discovered by the
+// runtime.
+message ResourcesInfo {
+    // Pod-level QoS resources available.
+    repeated QoSResourceInfo pod_qos_resources = 1;
+    // Container-level QoS resources available.
+    repeated QoSResourceInfo container_qos_resources = 2;
+}

+// QoSResourceInfo contains information about one type of QoS resource.
+message QoSResourceInfo {
+    // Name of the QoS resources.
+    string Name = 1;
+    // Mutable is set to true if this resource supports in-place updates i.e.
+    // the class of a running container or sandbox can be changed.
+    bool Mutable = 2;
+    // List of classes of this QoS resource.
+    repeated QoSResourceClassInfo classes = 3;
+}

+// QoSResourceClassInfo contains information about one class of certain
+// QoS resource.
+message QoSResourceClassInfo {
+    // Name of the class
+    string name = 1;
+    // Capacity is the number of maximum allowed simultaneous assignments into this class
+    // Zero means "infinite" capacity i.e. the usage is not restricted.
+    uint64 capacity = 2;
 }
```

#### Consts

Also, define "known" QoS-class resource types to more easily align container
runtime implementations:

```diff
+const (
+       // QoSResourceRdt is the name of the RDT QoS-class resource
+       QoSResourceRdt = "rdt"
+       // QoSResourceBlockio is the name of the blockio QoS-class resource
+       QoSResourceBlockio = "blockio"
+)
```

### Kubernetes API

The PodSpec will be extended to support assignment of pod-level and
container-level QoS-class devices. NodeStatus will be extended to include
information about the available QoS-class resources on a node.

#### PodSpec

We introduce a new field, QoSResources into the existing ResourceRequirements
struct. This will enable the assignment of QoS resources for containers.

```diff
 type ResourceRequirements struct {
        // Limits describes the maximum amount of compute resources allowed.
@@ -2195,6 +2198,10 @@ type ResourceRequirements struct {
        // +featureGate=DynamicResourceAllocation
        // +optional
        Claims []ResourceClaim
+       // QoSResources specifies the QoS resources.
+       // +optional
+       QoSResources map[QoSResourceName]string
 }

+// QoSResourceName is the name of a QoS resource.
+type QoSResourceName string
```

Also, we add a Resources field to the PodSpec, to enable assignment of
pod-level QoS resources. We will re-use the existing ResourceRequirements type
but Limits and Requests and Claims must be left empty. QoSResources may be set
and they represent the Pod-level assignment of QoS-class resources,
corresponding the PodQoSResources message in PodSandboxConfig in the CRI
API.

```diff
 type PodSpec struct {
@@ -3062,6 +3069,10 @@ type PodSpec struct {
        ResourceClaims []PodResourceClaim
+       // Pod-level resources. Claims, requests and limits are not allowed
+       // to be specified for pods.
+       // +optional
+       Resources ResourceRequirements
 }
```

There is already an ongoing effort to add [Pod level resource limits][kep-2837]
that aims at adding a pod level `Resources` field in a similar fashion. Thus,
we opt for adding ResourceRequirements insted of QoSResources directly into the
PodSpec.

As an example, a Pod requesting class "fast" of a (exemplary) pod-level QoS
resource named "network", with one container requesting class "gold" of
container-level QoS resource named "rdt":

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: qos-resource-example
spec:
  resources:
    qosResources:
      network: fast
  containers:
  - name: cnt
    image: nginx
    resources:
      qosResources:
        rdt: gold
```

#### NodeStatus

We extend NodeStatus to list available QoS-class resources on a node, This
consists of the list of the available QoS-class resource types and the classes
available within each of these resource types.



```diff
 type NodeStatus struct {
@@ -4444,6 +4482,11 @@ type NodeStatus struct {
        // Status of the config assigned to the node via the dynamic Kubelet config feature.
        Config *NodeConfigStatus
+       // QoSResources contains information about the QoS resources that are
+       // available on the node.
+       // +featureGate=QoSResources
+       // +optional
+       QoSResources QoSResourceStatus
 }

...

+// QoSResourceStatus describes QoS resources available on the node.
+type QoSResourceStatus struct {
+       // PodQoSResources contains the QoS resources that are available for pods
+       // to be assigned to.
+       PodQoSResources []QoSResourceInfo
+       // ContainerQoSResources contains the QoS resources that are available for
+       // containers to be assigned to.
+       ContainerQoSResources []QoSResourceInfo
+}

+// QoSResourceInfo contains information about one QoS resource type.
+type QoSResourceInfo struct {
+       // Name of the resource.
+       Name QoSResourceName
+       // Mutable is set to true if the resource supports in-place updates.
+       Mutable bool
+       // Classes available for assignment.
+       Classes []QoSResourceClassInfo
+}

+// QoSResourceClassInfo contains information about single class of one QoS
+// resource.
+type QoSResourceClassInfo struct {
+       // Name of the class.
+       Name string
+       // Capacity is the number of maximum allowed simultaneous assignments into this class
+       // Zero means "infinite" capacity i.e. the usage is not restricted
+       // +optional
+       Capacity int64
+}
```

### Kubelet

Kubelet gets QoS-class resource assignment from the [PodSpec](#podspec) and
translates these into corresponding `QoSResources` data in the CRI API. This is
ContainerConfig message at container creation time (CreateContainerRequest) and
PodSandboxConfig at sandbox creation time (RunPodSandboxRequest). In practice,
there is no translation, just copying key-value pairse.

Kubelet will receive the information about available QoS-class resources (the
types of reqources and their classes) from the runtime over the CRI API (new
Resources field in [RuntimeStatus](#runtimestatus) message). The kubelet
updates the new QoSResources field in [NodeStatus](#nodestatus) accordingly,
making QoS-class resources on the node visible to users and the kube-scheduler.

An admission handler is added into kubelet to validate the QoS-class resource
request against the resource availability on the node. Pod is rejected if
sufficient resources do not exist.

Invalid class assignments will cause an error in the container runtime which
causes the corresponding CRI RuntimeService request (e.g.  RunPodSandbox or
CreateContainer) to fail with an error. This can happen, despite the admission
check, e.g. when the requested QoS-class resource is not available on the
runtime (e.g. kubelet sends the pod or container start request before getting
an update of changed QoS-class resource availability from the runtime).

A feature gate QoSResources enables kubelet to interpretthe specific pod
annotations. If the feature gate is disabled the annotations are simply ignored
by kubelet.

### API server

Input validation of QoS-class resource names and class names, very similar to
labels is implemented: keys (`QoSResourceName`) and values must be non-empty,
less than 64 characters long, must start and end with an alphanumeric character
and may contain only alphanumeric characters, dashes, underscores or dots (`-`,
`_` or `.`). Also similar to labels, a namespace prefix (FQDN subdomain separated
with a slash) in the key is allowed (e.g. `vendor/qos-resource`).

### Scheduler

The kube-scheduler will be extended to do node pre-filtering based on the
QoS-class resource requests of a Pod. If any of the requested QoS-class
resources are unavailable on a node, that node is not fit. If no nodes can
satisfy all of the requests the Pod is marked as unschedulable and will be
scheduled in the future when/if requests can be satisfied by some node.

In principle, scheduling will behave similarly to native and extended
resources. The kube-scheduler will also take in account the per-class capacity
on the nodes if that is provided.

### Kubectl

The kubectl describe command will be extended to:

- available QoS-class resources of nodes
- display QoS-class resource requests of pods

### Container runtimes

Currently, there is support (container-level QoS-class resources) for Intel RDT
and blockio in CRI-O and containerd runtimes:

- cri-o:
  - [~~Add support for Intel RDT~~](https://github.com/cri-o/cri-o/pull/4830)
  - [~~Support for cgroups blockio~~](https://github.com/cri-o/cri-o/pull/4873)
- containerd:
  - [~~Support Intel RDT~~](https://github.com/containerd/containerd/pull/5439)
  - [~~Support for cgroups blockio~~](https://github.com/containerd/containerd/pull/5490)

The design paradigm here is that the container runtime configures the QoS-class
resources according to a given configuration file. Enforcement on containers is
done via OCI. User interface is provided through pod and container annotations.

Container runtimes will be updated to support the
[CRI API extensions](#cri-api)

### Open Questions

#### Pod QoS class

The Pod QoS class could be communicated to the container runtime as a QoS-class
resource, too. This information is currently internal to kubelet. However,
container runtimes (CRI-O, at least) are already depending on this information
and currently determining it indirectly by evaluating other CRI parameters. It
would be better to explicitly state the Pod QoS class and QoS-class resources would
look like a logical place for that. This also makes it techically possible to
have container-specific QoS classes (as a possible future enhancement of K8s).

Making this change, it would also be possible to separate `oom_score_adj` from
the pod qos class in the future.  The runtime could provide a set of OOM
classes, making it possible for the user to specify a burstable pod with low
oom priority (low chance of being killed).

#### Default class

A mechanism for indicating that the (runtime) default class should be used. The
default class would/should be a node/runtime specific attribute. How should
this be specified in the CRI protocol/`cri-api` and Pod spec?


### Test Plan

<!--
**Note:** *Not required until targeted at a release.*
The goal is to ensure that we don't accept enhancements with inadequate testing.

All code is expected to have adequate tests (eventually with coverage
expectations). Please adhere to the [Kubernetes testing guidelines][testing-guidelines]
when drafting this test plan.

[testing-guidelines]: https://git.k8s.io/community/contributors/devel/sig-testing/testing.md
-->

[x] I/we understand the owners of the involved components may require updates to
existing tests to make this code solid enough prior to committing the changes necessary
to implement this enhancement.

##### Prerequisite testing updates

<!--
Based on reviewers feedback describe what additional tests need to be added prior
implementing this enhancement to ensure the enhancements have also solid foundations.
-->

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

- `k8s.io/kubernetes/pkg/kubelet/kuberuntime`: `2022-06-13` - `66.8%`
- `k8s.io/kubernetes/pkg/apis/core/validation/validation.go`: `2022-06-13` - `82.1%`
- `k8s.io/kubernetes/pkg/scheduler`

##### Integration tests

<!--
This question should be filled when targeting a release.
For Alpha, describe what tests will be added to ensure proper quality of the enhancement.

For Beta and GA, add links to added tests together with links to k8s-triage for those tests:
https://storage.googleapis.com/k8s-triage/index.html
-->

Alpha: Existing integration tests for kubelet and kube-scheduler will be
extended to cover QoS-class resources.

Beta: Existing integration tests for affected components (e.g. scheduler, node
status, quota) are extended to cover QoS-class resources.

- <test>: <link to test coverage>

##### e2e tests

<!--
This question should be filled when targeting a release.
For Alpha, describe what tests will be added to ensure proper quality of the enhancement.

For Beta and GA, add links to added tests together with links to k8s-triage for those tests:
https://storage.googleapis.com/k8s-triage/index.html

We expect no non-infra related flakes in the last month as a GA graduation criteria.
-->

Alpha: no specific e2e-tests are planned.

In order to be able to run e2e tests, a cluster with nodes having runtime
support for QoS-class resources is required.

### Graduation Criteria

<!--
**Note:** *Not required until targeted at a release.*

Define graduation milestones.

These may be defined in terms of API maturity, [feature gate] graduations, or as
something else. The KEP should keep this high-level with a focus on what
signals will be looked at to determine graduation.

Consider the following in developing the graduation criteria for this enhancement:
- [Maturity levels (`alpha`, `beta`, `stable`)][maturity-levels]
- [Feature gate][feature gate] lifecycle
- [Deprecation policy][deprecation-policy]

Clearly define what graduation means by either linking to the [API doc
definition](https://kubernetes.io/docs/concepts/overview/kubernetes-api/#api-versioning)
or by redefining what graduation means.

In general we try to use the same stages (alpha, beta, GA), regardless of how the
functionality is accessed.

[feature gate]: https://git.k8s.io/community/contributors/devel/sig-architecture/feature-gates.md
[maturity-levels]: https://git.k8s.io/community/contributors/devel/sig-architecture/api_changes.md#alpha-beta-and-stable-versions
[deprecation-policy]: https://kubernetes.io/docs/reference/using-api/deprecation-policy/

Below are some examples to consider, in addition to the aforementioned [maturity levels][maturity-levels].

#### Alpha

- Feature implemented behind a feature flag
- Initial e2e tests completed and enabled

#### Beta

- Gather feedback from developers and surveys
- Complete features A, B, C
- Additional tests are in Testgrid and linked in KEP

#### GA

- N examples of real-world usage
- N installs
- More rigorous forms of testing—e.g., downgrade tests and scalability tests
- Allowing time for feedback

**Note:** Generally we also wait at least two releases between beta and
GA/stable, because there's no opportunity for user feedback, or even bug reports,
in back-to-back releases.

**For non-optional features moving to GA, the graduation criteria must include
[conformance tests].**

[conformance tests]: https://git.k8s.io/community/contributors/devel/sig-architecture/conformance-tests.md

#### Deprecation

- Announce deprecation and support policy of the existing flag
- Two versions passed since introducing the functionality that deprecates the flag (to address version skew)
- Address feedback on usage/changed behavior, provided on GitHub issues
- Deprecate the flag
-->

#### Beta

- Gather feedback from developers and surveys
- In addition to the initial changes in CRI API, implement the following
  - Extend CRI API to support updating sandbox-level QoS-class resources
  - Parmission control (resource quota)
- Well-defined behavior with [In-place pod vertical scaling](https://github.com/kubernetes/enhancements/tree/master/keps/sig-node/1287-in-place-update-pod-resources)
- Additional tests are in Testgrid and linked in KEP
- User documentation is available

#### GA

- More rigorous forms of testing—e.g., downgrade tests and scalability tests
- Allowing time for feedback


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

In this section we refer to different
[implementation phases](#implementation-phases). In this KEP we're now
targeting phase 1.

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
  - Feature gate name: QoSResources
  - Components depending on the feature gate:
    - Implementation Phase 1:
        - kubelet
        - kube-apiserver (validation of annotations)
    - Future phases (with updated pod spec and scheduler and quota support):
        - kubelet
        - kube-apiserver
        - kube-scheduler
        - kube-controller-manager
- [ ] Other
  - Describe the mechanism:
  - Will enabling / disabling the feature require downtime of the control
    plane?
  - Will enabling / disabling the feature require downtime or reprovisioning
    of a node? (Do not assume `Dynamic Kubelet Config` feature is enabled).

###### Does enabling the feature change any default behavior?

<!--
Any change of default behavior may be surprising to users or break existing
automations, so be extremely careful here.
-->

No.

###### Can the feature be disabled once it has been enabled (i.e. can we roll back the enablement)?

<!--
Describe the consequences on existing workloads (e.g., if this is a runtime
feature, can it break the existing applications?).

Feature gates are typically disabled by setting the flag to `false` and
restarting the component. No other changes should be necessary to disable the
feature.

NOTE: Also set `disable-supported` to `true` or `false` in `kep.yaml`.
-->

Yes it can.

Implementation Phase 1: In this phase pod annotations are used as the user
interface for assigning QoS-class resources to workloads. Existing
running workloads continue to work without any changes as their QoS-class
resource assigment in the runtime is not changed.
Restarting or re-deploying a workload causes it to lose its QoS-class resource
assignment as the annotation parsing in kubelet is disabled. In other words,
the workload is able to run but the QoS-class resource assignment requests from
the user, i.e. via pod annotations, are ignored by kubelet.

Future implementation phases: running workloads continue to work without any
changes. Restarting or re-deploying a workload causes it to fail as the
requested QoS-class resources are not available in Kubernetes anymore. The
resources are still supported by the underlying runtime but disabling the
feature in Kubernetes makes them unavailable and the related PodSpec fields are
not accepted in validation.

###### What happens if we reenable the feature if it was previously rolled back?

Implementation Phase 1: workloads need to be restarted to re-evaluate the pod
annotations to correctly communicate QoS-class resource assignments to the
container runtime.

Future implementation phases: workloads might have failed because of
unsupported fields in the pod spec and need to be restarted.

###### Are there any tests for feature enablement/disablement?

<!--
The e2e framework does not currently support enabling or disabling feature
gates. However, unit tests in each component dealing with managing data, created
with and without the feature, are necessary. At the very least, think about
conversion tests if API types are being modified.

Additionally, for features that are introducing a new API field, unit tests that
are exercising the `switch` of feature gate itself (what happens if I disable a
feature gate after having objects written with the new field) are also critical.
You can take a look at one potential example of such test in:
https://github.com/kubernetes/kubernetes/pull/97058/files#diff-7826f7adbc1996a05ab52e3f5f02429e94b68ce6bce0dc534d1be636154fded3R246-R282
-->

Implementation phase 1: Unit test will be added to kubelet to test that
inspection of [pod annotations](#pod-annotations) is correctly disabled/enabled
with the feature gate.

Future implementation phases: unit tests for handling the changes in pod spec
are implemented.

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

Implementation Phase 1: we rely on inspection of pod annotations inside kubelet
which should make rollout/rollback failure-safe. Already running workloads are
not affected.

Future implementation phases: TBD.

###### What specific metrics should inform a rollback?

<!--
What signals should users be paying attention to when the feature is young
that might indicate a serious problem?
-->

Implementation Phase 1: watch for non-ready pods with CreateContainerError
status. The error message will indicate the if the failure is related to
QoS-class resources.

Future implementation phases: TBD.

###### Were upgrade and rollback tested? Was the upgrade->downgrade->upgrade path tested?

<!--
Describe manual testing that was done and the outcomes.
Longer term, we may want to require automated upgrade/rollback tests, but we
are missing a bunch of machinery and tooling and can't do that now.
-->

TBD in future implementation phases.

###### Is the rollout accompanied by any deprecations and/or removals of features, APIs, fields of API types, flags, etc.?

<!--
Even if applying deprecation policies, they may still surprise some users.
-->

Implementation Phase 1: No.

Future implementation phases: TBD but should be no. Disabling the feature
should preserve the data of new fields (e.g. in pod spec) even if they are
disabled.

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

Implementation Phase 1: by examining pod annotations.

Future implementation phases: by examining the new fields in pod spec.

###### How can someone using this feature know that it is working for their instance?

<!--
For instance, if this is a pod-related feature, it should be possible to determine if the feature is functioning properly
for each individual pod.
Pick one more of these and delete the rest.
Please describe all items visible to end users below with sufficient detail so that they can verify correct enablement
and operation of this feature.
Recall that end users cannot usually observe component logs or access metrics.
-->

- [x] Events
  - Event Reason: Failed (CreateContainerError)

<!--
- [ ] API .status
  - Condition name: 
  - Other field: 
- [ ] Other (treat as last resort)
  - Details:
-->

To be defined in more detail in future implementation phases and for beta.

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

TBD in future implementation phases but basically the existing SLOs for Pods
should be adequate.

###### What are the SLIs (Service Level Indicators) an operator can use to determine the health of the service?

<!--
Pick one more of these and delete the rest.
-->

<!--
- [ ] Metrics
  - Metric name:
  - [Optional] Aggregation method:
  - Components exposing the metric:
- [ ] Other (treat as last resort)
  - Details:
-->

TBD in future implementation phases and for beta.

###### Are there any missing metrics that would be useful to have to improve observability of this feature?

<!--
Describe the metrics themselves and the reasons why they weren't added (e.g., cost,
implementation difficulties, etc.).
-->

TBD in future implementation phases.

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

Implementation Phase 1: A container runtime with support for the new CRI API
fields is required.

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

No, enabling or using the feature does not induce any new API calls in
Kubernetes.

###### Will enabling / using this feature result in introducing new API types?

<!--
Describe them, providing:
  - API type
  - Supported number of objects per cluster
  - Supported number of objects per namespace (for namespace-scoped objects)
-->

Implementation Phase 1: No.

Future implementation phases: QoS-class resources do extend existing API types
but presumably not introduce new types of objects. However, the design for
resource discovery and permission control is not ready which might change this.

###### Will enabling / using this feature result in any new calls to the cloud provider?

<!--
Describe them, providing:
  - Which API(s):
  - Estimated increase:
-->

No, enabling or using the feature does not result in any new calls to the cloud
provider.

###### Will enabling / using this feature result in increasing size or count of the existing API objects?

<!--
Describe them, providing:
  - API type(s):
  - Estimated increase in size: (e.g., new annotation of size 32B)
  - Estimated amount of new objects: (e.g., new Object X for every existing Pod)
-->

Implementation Phase 1: [pod annotations](#pod-annotations) are used as the
initial user interface so assign QoS-class resources to containers. Exact size
of each annotation varies (depending on the type of resource) but the
annotation key is expected to be few tens of bytes. The value part is the name
of the class expected to be a few bytes long.

Future implementations: New fields in the pod spec will increase the size of
`Pod` objects by a few bytes per class requested. New fields will be added to
NodeStatus which will increase its size. New field will be added to
ResourceQuotaSpec increasing its size.

###### Will enabling / using this feature result in increasing time taken by any operations covered by existing SLIs/SLOs?

<!--
Look at the [existing SLIs/SLOs].

Think about adding additional work or introducing new steps in between
(e.g. need to do X to start a container), etc. Please describe the details.

[existing SLIs/SLOs]: https://git.k8s.io/community/sig-scalability/slos/slos.md#kubernetes-slisslos
-->

No, this is not expected.

###### Will enabling / using this feature result in non-negligible increase of resource usage (CPU, RAM, disk, IO, ...) in any components?

<!--
Things to keep in mind include: additional in-memory state, additional
non-trivial computations, excessive access to disks (including increased log
volume), significant amount of data sent and/or received over network, etc.
This through this both in small and large cases, again with respect to the
[supported limits].

[supported limits]: https://git.k8s.io/community//sig-scalability/configs-and-limits/thresholds.md
-->

No, this is not expected.

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

Pods cannot be scheduled which makes the feature unavailable for new workloads.
Existing workloads continue to work without interruption (with respect to this
feature).

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

TBD.

###### What steps should be taken if SLOs are not being met to determine the problem?

TBD.

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

<!--
What other approaches did you consider, and why did you rule them out? These do
not need to be as detailed as the proposal, but should include enough
information to express the idea and why it was not acceptable.
-->

### Pod annotations

Instead of updating CRI and Kubernetes API in lock-step, the API changes could
be split into two phases. Similar to e.g. how seccomp support was added, Pod
annotations could be used as an initial user interface before the Kubernetes
API changes are merged.

Specifically, annotations for specifying RDT and blockio class would be
supported. These are the two types of QoS-class resources that already have
basic support in the container runtimes.

- `rdt.resources.alpha.kubernetes.io/default` for setting a Pod-level default RDT
  class for all containers
- `rdt.resources.alpha.kubernetes.io/container.<container-name>` for
  container-specific RDT class settings
  blockio class for all containers
- `blockio.resources.alpha.kubernetes.io/default` for setting a Pod-level default
  blockio class for all containers
- `blockio.resources.alpha.kubernetes.io/container.<container-name>` for
  container-specific blockio class settings

#### Kubelet

Kubelet would interpret the specific [pod annotations](#pod-annotations) and
translate them into corresponding `QoSResources` data in the CRI
ContainerConfig message at container creation time (CreateContainerRequest).
Pod-level QoS-class would not supported at this point (via pod annotations).

A feature gate QoSResources would enable kubelet to interpretthe specific pod
annotations. If the feature gate is disabled the annotations would simply be
ignored by kubelet.

#### API server

A validation check (core api validation) would added in the API server to
reject changes to the QoS-class resource specific
[pod annotations](#pod-annotations) after a Pod has been created. This would
ensure that the annotations always reflect the actual assignment of QoS-class
resources of a Pod. It also would serve as part of the UX to indicate the
in-place updates of the resources via annotations is not supported.

#### Class capacity with extended resources

Support for class capacity could be left out of the proposal to simplify the
concept. It would be possible to implement "class capacity" by leveraging
extended resources and mutating admission webhooks:

1. An extended resource with the desited capacity is created for each class
   which needs to be controlled. A possible example:
  ```plaintext
  Allocatable:
  qos/container/resource-X/class-A:        2
  qos/container/resource-X/class-B:        4
  qos/container/resource-Y/class-foo:      1
  ```

  In this example *class-A* and *class-B* of QoS-class resource *resource-X*
  would be limited to 2 and 4 assignments, respectively (on this node).
  Similarly, class *class-foo* of QoS-class resource *resource-Y* would be
  limited to only one assignment.

2. A specific mutating admission webhook is deployed which syncs the specific
   extended resources. That is, if a specific QoS-class resource assignment is
   requested in the pod spec, a request for the corresponding extended resource
   is automatically added. And vice versa: if a specific extended resource is
   requested a QoS-class resource request is automatically put in place.

However, this approach is quite involved, has shortcomings and caveats and thus
it is probably suitable only for targeted usage scenarios, not as a general
solution. Downsides include:

- requires implementation of "side channel" control mechanisms, e.g. admission
  webhook and some solution for capacity management (extended resources)
- deployment of admission webhooks is cumbersome
- management of capacity is limited and cumbersome
  - management of extended resources needs a separate mechanism
  - ugly to handle a scenario where *class-A* on *node-1* would be unlimited
    but on *node-2* it would have some capacity limit (e.g. use a very high
    integer capacity of the extended resource on *node-2*)
  - dynamic changes are harder
- not well-suited for managing pod-level QoS-class resources as pod-level
  resource requests (for the extended resource) are not supported in Kubernetes
- possible confusion for users regarding the double accounting (QoS-class
  resources and extended resources)


### RDT-only

The scope of the KEP could be narrowed down by concentrating on RDT only,
dropping support for blockio. This would center the focus on RDT only which is
well understood and specified in the OCI runtime specification.

### Widen the scope

The currently chosen strategy of this KEP is "minimum viable product" with
incremental future steps of improving and supplementing the functionality. This
strategy was chosen in order to make the review easier by handling smaller
digestible (but still coherent and self-contained) chunks at a time.

An alternaive would be to widen the scope of this KEP to include some or all of
the subjects mentioned in [future work](#future-work) (i.e. resource discovery,
status/capacity and access control).

## Infrastructure Needed (Optional)

<!--
Use this section if you need things from the project/SIG. Examples include a
new subproject, repos requested, or GitHub details. Listing these here allows a
SIG to get the process for these resources started right away.
-->

For proper end-to-end testing of RDT, a cluster with nodes that have RDT
enabled would be required. Similarly, for end-to-end testing of blockio, nodes
with blockio cgroup controller and suitable i/o scheduler enabled would be
required.

<!-- References -->
[intel-rdt]: https://www.intel.com/content/www/us/en/architecture-and-technology/resource-director-technology.html
[linux-resctrl]: https://www.kernel.org/doc/html/latest/x86/resctrl.html
[kep-2837]: https://github.com/kubernetes/enhancements/pull/1592