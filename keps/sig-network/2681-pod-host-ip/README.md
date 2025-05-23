# KEP-2681: Field status.hostIPs added for Pod

<!-- toc -->
- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Proposal](#proposal)
  - [User Stories](#user-stories)
    - [Story 1](#story-1)
  - [Risks and Mitigations](#risks-and-mitigations)
- [Design Details](#design-details)
  - [Versioned API Change: PodStatus v1 core](#versioned-api-change-podstatus-v1-core)
  - [PodStatus Internal Representation](#podstatus-internal-representation)
  - [Maintaining Compatible Interworking between Old and New Clients](#maintaining-compatible-interworking-between-old-and-new-clients)
  - [Container Environment Variables](#container-environment-variables)
  - [Test Plan](#test-plan)
      - [Prerequisite testing updates](#prerequisite-testing-updates)
      - [Unit tests](#unit-tests)
      - [Integration tests](#integration-tests)
      - [e2e tests](#e2e-tests)
  - [Graduation Criteria](#graduation-criteria)
    - [Alpha](#alpha)
    - [Beta](#beta)
    - [GA](#ga)
    - [Deprecation](#deprecation)
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

## Summary

The proposal aims to improve the Pod's ability to obtain the address of the node

## Motivation

### Goals

- Field `status.hostIPs` added for Pod
- Downward API support for `status.hostIPs`

### Non-Goals

## Proposal

### User Stories

#### Story 1

Applications that originally used IPv4 migrated to IPv6 during the dual-stack transition phase, 
For smooth migration, IP-related attributes should have both IPv4 and IPv6.

### Risks and Mitigations

## Design Details

### Versioned API Change: PodStatus v1 core

In order to maintain backwards compatibility for the core V1 API, this proposal
retains the existing (singular) "HostIP" field in the core V1 version of the
PodStatus V1 core API and adds a new array of structures that store host IPs
along with associated metadata for that IP.

``` go
    // HostIP represents the IP address of a host.
    // IP address information. Each entry includes:
    //    IP: An IP address allocated to the host.
    type HostIP struct {
      // ip is an IP address (IPv4 or IPv6) assigned to the host
      IP string
    }

    // HostIPs holds the IP addresses allocated to the host. If this field is specified, the 0th entry must
    // match the hostIP field. This list is empty if no IPs have been allocated yet.
    // +optional
    HostIPs []HostIP
```

### PodStatus Internal Representation

PodStatus internally indicates that the original use of HostIP will remain unchanged,
and HostIPs is added for subsequent use

``` go
    // HostIP address information for entries in the (plural) HostIPs field.
    // Each entry includes:
    //    IP: An IP address allocated to the host.
    type HostIP struct {
      // ip is an IP address (IPv4 or IPv6) assigned to the host
      IP string `json:"ip,omitempty" protobuf:"bytes,1,opt,name=ip"`
    }

    // HostIPs holds the IP addresses allocated to the host. If this field is specified, the 0th entry must
    // match the hostIP field. This list is empty if no IPs have been allocated yet.
    // +optional
    // +patchStrategy=merge
    // +patchMergeKey=ip
    HostIPs []HostIP `json:"hostIPs,omitempty" protobuf:"bytes,14,rep,name=hostIPs" patchStrategy:"merge" patchMergeKey:"ip"`
```

### Maintaining Compatible Interworking between Old and New Clients

[See the making a singular field plural](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/api_changes.md#making-a-singular-field-plural)

### Container Environment Variables

The Downward API [status.hostIP](https://kubernetes.io/docs/tasks/inject-data-application/downward-api-volume-expose-pod-information/#capabilities-of-the-downward-api)
will preserve the existing single IP address, and will be set to the default IP for each pod.
A new pod API field named `status.hostIPs` will contain a list of IP addresses.
The new pod API will have a slice of structures for the additional IP addresses. 
Kubelet will translate the pod structures and return `status.hostIPs` as a comma-delimited string.

Here is an example of how to define a pluralized `MY_HOST_IPS` environmental
variable in a pod definition yaml file:

``` yaml
- name: MY_HOST_IPS
  valueFrom:
    fieldRef:
      fieldPath: status.hostIPs
```

This definition will cause an environmental variable setting in the pod similar
to the following:

```
# PodHostIPs FeatureGate is enabled
MY_HOST_IPS=fd00:10:20:0:3::3,10.20.3.3

# PodHostIPs FeatureGate is disabled
MY_HOST_IPS=
```

### Test Plan

[x] I/we understand the owners of the involved components may require updates to
existing tests to make this code solid enough prior to committing the changes necessary
to implement this enhancement.

##### Prerequisite testing updates

N/A

##### Unit tests

Every added code should have complete unit test coverage

##### Integration tests

N/A

##### e2e tests

Test whether FeatureGate behaves as expected when it is turned on or off
Test whether Downward API supports `status.hostIPs`

- If PodHostIPs FeatureGate is enabled:
  - `status.hostIPs` there will be all host IPs (IPv4 and IPv6)
- Else:
  - `status.hostIPs` will be empty and the ApiServer will reject Downward API `status.hostIPs`, 
    if Pods already existing Downward API `status.hostIPs`, ensure to ignore it and not affect others.

### Graduation Criteria

#### Alpha

- Feature implemented behind a feature flag
  - Add `status.hostIPs` to the PodStatus API
  - Downward API support for `status.hostIPs`
- Basic units and e2e tests completed and enabled

#### Beta

- Gather feedback from developers and users
- Expand the e2e tests with more scenarios
- Tests are in Testgrid and linked in the KEP

#### GA

- 2 examples of end users using this field
- Allowing time for feedback

#### Deprecation

- Announce deprecation and support policy of the existing flag
- Two versions passed since introducing the functionality that deprecates the flag (to address version skew)
- Address feedback on usage/changed behavior, provided on GitHub issues
- Deprecate the flag

### Upgrade / Downgrade Strategy

N/A

### Version Skew Strategy

N/A

## Production Readiness Review Questionnaire

### Feature Enablement and Rollback

###### How can this feature be enabled / disabled in a live cluster?

- [x] Feature gate (also fill in values in `kep.yaml`)
  - Feature gate name: PodHostIPs
  - Components depending on the feature gate: `kube-apiserver`, `kubelet`
- [ ] Other
  - Describe the mechanism:
  - Will enabling / disabling the feature require downtime of the control
    plane?
  - Will enabling / disabling the feature require downtime or reprovisioning
    of a node?

###### Does enabling the feature change any default behavior?

It changes default behavior of k8s itself by automatically propagating HostIPs field.

###### Can the feature be disabled once it has been enabled (i.e. can we roll back the enablement)?

Yes. Using the featuregate is the only way to enable/disable this feature.

###### What happens if we reenable the feature if it was previously rolled back?

The feature should continue to work just fine.

###### Are there any tests for feature enablement/disablement?

There are tests for feature enablement/disablement in [util_test.go](https://github.com/kubernetes/kubernetes/blob/83f2d89dc987e152f27b31bf630c58ce855d954d/pkg/api/pod/util_test.go#L1168-L1264) and [validation_test.go](https://github.com/kubernetes/kubernetes/blob/83f2d89dc987e152f27b31bf630c58ce855d954d/pkg/apis/core/validation/validation_test.go#L23068-L23113).

### Rollout, Upgrade and Rollback Planning

###### How can a rollout or rollback fail? Can it impact already running workloads?

If the dependent [KEP 563-dual-stack](https://github.com/kubernetes/enhancements/issues/563) is wrong, or could not get IP of host, or setting the field is crashing, this feature may fail to rollout/rollback.
The field is only informative, it doesn't affect running workloads.

###### What specific metrics should inform a rollback?

It will immediately update all running Pods on the node where this feature is enabled.

If any of these phenomena imply that the feature is abnormal and needs to be rolled back, the `status.hostIPs` field in the pod is empty, or it is updated frequently, or it causes other Pods to crash.

So if the `apiserver_requests_total` for pods increases significantly, this may indicate a problem.

###### Were upgrade and rollback tested? Was the upgrade->downgrade->upgrade path tested?

It will test upgrade and rollback in e2e tests if it can be done in e2e.

Upgrade->downgrade->upgrade testing was done manually using the following steps:

Build and run the latest version using Kind

``` console
$ kind build node-image
$ kind create cluster --image kindest/node:latest
...
$ kubectl get node
NAME                 STATUS   ROLES           AGE     VERSION
kind-control-plane   Ready    control-plane   6m40s   v1.28.0-alpha.2.1529+c649dadff44981
```

Deploy a webserver
``` yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: agnhost-server
  labels:
    app: agnhost-server
spec:
  replicas: 1
  selector:
    matchLabels:
      app: agnhost-server
  template:
    metadata:
      labels:
        app: agnhost-server
    spec:
      containers:
      - name: agnhost
        image: registry.k8s.io/e2e-test-images/agnhost:2.40
        args:
        - serve-hostname
        - --port=80
        ports:
        - containerPort: 80
```

Waiting pod be ready
``` console
$ kubectl get pod
NAME                                         READY   STATUS    RESTARTS   AGE
agnhost-server-76fb5c696c-2rqnh              1/1     Running   0          6s
```

Check pod hostIPs
``` console
$ kubectl get pod agnhost-server-76fb5c696c-2rqnh -o jsonpath='{.status.hostIPs}'
[{"ip":"172.18.0.2"}]
```

To disable the feature
``` console
$ docker exec -it kind-control-plane bash

$ cat <<EOF >>/var/lib/kubelet/config.yaml
featureGates:
  PodHostIPs: false
EOF

$ systemctl restart kubelet
```

Add more pod
``` console
$ kubectl scale --replicas=2 kubectl deploy/agnhost-server
```

Check that both Pods do not have HostIPs

``` console
$ kubectl get pod -o jsonpath='{.items[*].status.hostIPs}'
```

To enable the feature
``` console
$ docker exec -it kind-control-plane bash

$ sed -i 's/PodHostIPs: false/PodHostIPs: true/g' /var/lib/kubelet/config.yaml

$ systemctl restart kubelet
```

Check that both Pods have HostIPs
``` console
$ kubectl get pod -o jsonpath='{.items[*].status.hostIPs}'
[{"ip":"172.18.0.2"}] [{"ip":"172.18.0.2"}]
```

###### Is the rollout accompanied by any deprecations and/or removals of features, APIs, fields of API types, flags, etc.?

No.

### Monitoring Requirements

###### How can an operator determine if the feature is in use by workloads?

Pod has a `status.hostIPs` field and use it in downwardAPI to expose it. 

###### How can someone using this feature know that it is working for their instance?

- [ ] Events
  - Event Reason: 
- [x] API .status
  - Other field: `pod.status.hostIPs` is not empty.
- [ ] Other (treat as last resort)
  - Details:

###### What are the reasonable SLOs (Service Level Objectives) for the enhancement?

- The feature is only informative, no increased failure rates during use the feature.

###### What are the SLIs (Service Level Indicators) an operator can use to determine the health of the service?

No, having a metric for this feature is overkill.

###### Are there any missing metrics that would be useful to have to improve observability of this feature?

No additional metrics needed for this new API field.

### Dependencies

###### Does this feature depend on any specific services running in the cluster?

N/A

### Scalability

###### Will enabling / using this feature result in any new API calls?

No

###### Will enabling / using this feature result in introducing new API types?

No

###### Will enabling / using this feature result in any new calls to the cloud provider?

No

###### Will enabling / using this feature result in increasing size or count of the existing API objects?

Yes.

- API type(s): Pod
- Estimated increase in size:
  - New field in Status about 8 bytes, additional bytes based on whether IPv4(about 4 bytes) or IPv6(about 16 bytes) exists

###### Will enabling / using this feature result in increasing time taken by any operations covered by existing SLIs/SLOs?

No

###### Will enabling / using this feature result in non-negligible increase of resource usage (CPU, RAM, disk, IO, ...) in any components?

No

###### Can enabling / using this feature result in resource exhaustion of some node resources (PIDs, sockets, inodes, etc.)?

No

### Troubleshooting

###### How does this feature react if the API server and/or etcd is unavailable?

N/A -- since the feature is part of kube-apiserver.

###### What are other known failure modes?

N/A

###### What steps should be taken if SLOs are not being met to determine the problem?

N/A

## Implementation History

- 2021-05-06: Initial KEP
- 2023-08-15: Alpha release with Kubernetes 1.28
- 2023-10-30: Beta release with Kubernetes 1.29
- 2024-03-01: GA release with Kubernetes 1.30

## Drawbacks

N/A

## Alternatives

N/A

## Infrastructure Needed (Optional)

N/A
