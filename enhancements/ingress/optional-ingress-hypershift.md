---
title: optional-ingress-hypershift
authors:
  - "@alebedev87"
reviewers:
  - "@Miciah"
  - "@bparees"
  - "@cewong"
  - "@wking"
approvers:
  - "@Miciah"
  - "@bparees"
  - "@wking"
api-approvers:
  - N/A
creation-date: 2023-06-02
last-updated: 2023-06-02
tracking-link:
  - https://issues.redhat.com/browse/NE-1129
see-also:
  - "/enhancements/installer/component-selection.md"
  - "/enhancements/ingress/aws-load-balancer-operator.md"
replaces:
superseded-by:
---

# Make Ingress Operator optional on HyperShift

## Summary

This enhancement describes the implementation of a new capability to enable/disable the default ingress of the OpenShift cluster.
The new capability (from here on called the ingress capability) includes [the cluster ingress operator](https://github.com/openshift/cluster-ingress-operator) as well as the custom resources managed by the operator.
This enhancement focuses on the requirements for [RFE-3395](https://issues.redhat.com/browse/RFE-3395) which targets the ROSA clusters spawned and managed by HyperShift. The implementation of the ingress capability on the standalone OpenShift is out of the scope of this enhancement.

## Motivation

Users of the ROSA managed OpenShift service would like to use [AWS load balancers](/enhancements/ingress/aws-load-balancer-operator.md) as a way to accept the ingress traffic into the cluster.
The cluster ingress operator becomes unnecessary and can be disabled to:
- reduce the resource consumption on
  - the HyperShift client clusters (ingress controllers)
  - the HyperShift hosted control plane (operator)
- make the HyperShift hosted control plane lighter and simpler, therefore more manageable
- enable the HyperShift cluster users to make their choice for the cluster ingress

### Goals
- Implement the new ingress capability.
- Add the ingress capability to the ingress operator's payload.
- Make the dependent cluster operators tolerate the absence of the ingress operator.
- Do not lock the subsequent implementation for the standalone OpenShift.

### Non-Goals
- Use a different way for enabling/disabling the ingress other than via [the cluster capability](/enhancements/installer/component-selection.md).
- Fully implement the ingress capability for the standalone OpenShift:
  - Make the OpenShift installer tolerate the absence of the default ingress controller.
  - Make the cluster authentication operator tolerate the absence of the default ingress controller.
  - Make the cluster monitoring operator tolerate the absence of the default ingress controller.
- Disable the Route API or try to tolerate the absence of the default ingress controller.
- Design the capability API for HyperShift.
- Implement the ingress operator disabling in the HyperShift's control plane operator.

## Proposal

### User Stories

- As a HyperShift engineer, I want to be able to disable the ingress operator on the clusters where the ingress is implemented differently so that the hosted control plane can become lighter.
- As a HyperShift engineer, I want the cluster operators to tolerate the absence of the ingress operator to keep the clusters from being degraded.
- As a HyperShift hosted cluster's user, I want to see the ingress capability as a known capability to bring the visibility to all the cluster actors (admins, users, other operators).
- As a HyperShift hosted cluster's user, I want the default ingress controller to not occupy the cluster resources while the ingress traffic is routed by the cloud provider's load balancers.

### Workflow Description

The HyperShift hosted cluster's user wants to use an AWS ALB to route the traffic to its example application deployed on a HyperShift's hosted cluster.

#### Terminology
[HyperShift concepts and personas](https://hypershift-docs.netlify.app/reference/concepts-and-personas/).

#### Disable the default ingress

- Cluster service consumer requests a hosted cluster without the ingress capability using the HostedCluster API (**note**: the HostedCluster's capability API does not exist and it's out of scope of this enhancement).
- HyperShift creates the hosted cluster.
- HyperShift skips the deployment of the ingress operator on the hosted control plane.
- HyperShift creates `ClusterVersion` without the ingress capability on the hosted cluster.
- Hosted cluster's cluster version operator does not deploy the cluster ingress operator's payload (CRDs, RBACs, etc.).

#### Setup AWS ALB as the new ingress

- Cluster instance admin installs [AWS Load Balancer Operator](/enhancements/ingress/aws-load-balancer-operator.md).
- Cluster instance admin sets up the default `AWSLoadBalancerController`.
- Cluster instance user creates the `Ingress` resource for the default aws load balancer controller's ingress class.
- AWS load balancer controller provisions an ALB on AWS.
- Cluster instance admin can install [External DNS Operator](/enhancements/dns/externaldns-operator.md) to help cluster instance users set up convenient DNS names.

### API Extensions

N/A - no api extensions are being introduced in this proposal

### Implementation Details/Notes/Constraints

#### Ingress capability

The name of the new capability is `Ingress`. It includes the cluster ingress operator and [all its APIs](#ingress-operator-apis) but does NOT include [the related APIs](#related-apis).
The implementation of the new capability should follow the instructions of [how to implement a new capability](/enhancements/installer/component-selection.md#how-to-implement-a-new-capability), to name some:
- `Ingress` capability should be added to `openshift/api` repository.
- The ingress capability should be part of the `vCurrent` and `v4.14` baseline sets.
- The openshift api vendored dependency should be bumped into `cluster-version-operator` repository.
  - Additionally some more changes will need to be done in the cluster version operator to address [the standalone use case](#standalone-openshift).
- The openshift api vendored dependency should be bumped in `installer` repository.
  - This will allow the users of the standalone OpenShift to see the ingress capability. While the enabling of the ingress capability is not supposed to cause any issue, the disabling may have some consequences therefore will be not possible, see [Standalone OpenShift chapter](#standalone-openshift) for more details.

Similar to any other capability, the ingress capability:
- can be enabled not only as part of the capability sets (`vCurrent`, `v4,14`) but explicitly by using [the additional enabled capability field](https://github.com/openshift/api/blob/master/config/v1/types_cluster_version.go#L433-L438).
- if enabled, should appear in [the list of the enabled capabilities](https://github.com/openshift/api/blob/master/config/v1/types_cluster_version.go#L445-L448) of `ClusterVersion` object.
- if enabled, cannot be disabled post install.

#### HyperShift

HyperShift doesn't use the OpenShift installer to spawn the user clusters. HyperShift deploys the cluster version operator to manage the cluster operators though.
However not all the cluster operators are of use for HyperShift.
The functionality of some cluster operators is fully replaced by the logic in the hosted control plane (e.g. authentication operator).
The ingress operator is an example of a cluster operator whose life cycle is shared between the cluster version operator and HyperShift.
HyperShift doesn't pass all the ingress operator's payload to the cluster version operator.
Certain manifests are filtered out, most importantly: [the ingress operator's deployment](https://github.com/openshift/hypershift/blob/main/control-plane-operator/controllers/hostedcontrolplane/cvo/reconcile.go#L75).
The reason for this is that the ingress operator is considered as a part of the hosted control plane hence deployed on the control plane cluster.
However [the ingress operator's APIs](#ingress-operator-apis) and some RBACs to manage those APIs are still kept in the cluster version operator's payload.
The ingress capability comes handy here as HyperShift can leverage it to control the remaining (not filtered) payload.

The HyperShift's API which will handle the hosted cluster capabilities is out of the scope of this enhancement. However this enhancement is built with the assumption that the `ClusterVersion`'s capability API (spec and status) is sufficient for any capability implementation in HyperShift.

#### Standalone OpenShift

Making the ingress operator optional for the standalone OpenShift is a bigger scope than for HyperShift.
The changes impact the components which are not used (or used differently) in HyperShift like the installer, the authentication and the monitoring operators.
To prevent the scope creep for [RFE-3395](https://issues.redhat.com/browse/RFE-3395), the following approach is proposed:
- the cluster version operator should be able to add required capabilities.
- the required capabilities is a configuration option of the cluster version operator (e.g. `--required-capabilities` flag).
- the cluster version operator should [implicitly enable](https://github.com/openshift/cluster-version-operator/blob/master/pkg/cvo/status.go#L161-L165) all the provided required capabilities so that they appear in the status of `ClusterVersion` object.
- the required capabilities should be added to [the cluster version operator manifests](https://github.com/openshift/cluster-version-operator/blob/master/install/0000_00_cluster-version-operator_03_deployment.yaml) only.
- the ingress capability should be required.

This approach should make any required capability read only on the standalone OpenShift. Also, it doesn't let the required capabilities to be set on HyperShift because HyperShift doesn't use the cluster version operator's deployment manifest but [manages the operator's deployment directly](https://github.com/openshift/hypershift/blob/main/control-plane-operator/controllers/hostedcontrolplane/cvo/reconcile.go#L318).

#### Ingress operator APIs

The following APIs are not available if the ingress capability is disabled:
- CRDs which are part of the cluster operator's manifest payload:
  - [IngressController](https://github.com/openshift/cluster-ingress-operator/blob/master/manifests/00-custom-resource-definition.yaml)
  - [DNSRecord](https://github.com/openshift/cluster-ingress-operator/blob/master/manifests/00-custom-resource-definition-internal.yaml)
- CRDs which are created at runtime if `gateway-api` feature gate is set:
  - [GatewayClass](https://github.com/openshift/cluster-ingress-operator/blob/master/assets/gateway-api/gateway.networking.k8s.io_gatewayclasses.yaml)
  - [Gateway](https://github.com/openshift/cluster-ingress-operator/blob/master/assets/gateway-api/gateway.networking.k8s.io_gateways.yaml)
  - [HTTPRoute](https://github.com/openshift/cluster-ingress-operator/blob/master/assets/gateway-api/gateway.networking.k8s.io_httproutes.yaml)

#### Related APIs

The following APIs are left intact (see [Add Route API to the ingress capability](#add-route-api-to-the-ingress-capability) to know why) if the ingress capability is disabled:
- [Route API](https://docs.openshift.com/container-platform/4.13/rest_api/network_apis/route-route-openshift-io-v1.html). Implemented in two places: OpenShift API as [CRD](https://github.com/openshift/api/blob/master/route/v1/route.crd.yaml) and in [openshift-api-server](https://github.com/openshift/openshift-apiserver/tree/master/pkg/route).
- [Ingress Config API](https://docs.openshift.com/container-platform/4.13/rest_api/config_apis/ingress-config-openshift-io-v1.html). Implemented in OpenShift API as [CRD](https://github.com/openshift/api/blob/master/config/v1/0000_10_config-operator_01_ingress.crd.yaml).
- [Ingress API](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.27/#ingress-v1-networking-k8s-io). Implemented in Kubernetes.

However the implementation of these APIs (or parts of them) depend on the ingress operator:
- The ingress operator creates the default ingress controller to manage the backends for the routes. See [the drawbacks](#drawbacks) for further implications.
  - The ingress objects are implicitly managed by the default ingress controller. See [the ingress to route controller risk](#ingress-to-route-controller) for further implications.
- For [the component routes](https://docs.openshift.com/container-platform/4.13/rest_api/config_apis/ingress-config-openshift-io-v1.html#spec-componentroutes) the ingress operator creates the required role and role binding to get the custom TLS certificate secret from the `openshift-config` namespace. See [the component routes risk](#component-routes) for further implications.

#### Console operator

In case the ingress capability is not enabled in `ClusterVersion`, the following issues arise with the console components:
- The console operator becomes degraded due to the failed admission checks of the console and download routes, see [the usage](https://github.com/search?q=repo%3Aopenshift%2Fconsole-operator%20IngressURI&type=code) of [IngressURI function](https://pkg.go.dev/github.com/openshift/library-go/pkg/route/routeapihelpers#IngressURI).
- The console route is not served by the default ingress controller.

To resolve the former problem, it is necessary to skip the route admission checks when the ingress capability is not enabled.   
The later issue is discussed as a general case of the missing default ingress controller in [the drawbacks](#drawbacks).

#### Authentication operator

No change to the authentication operator is needed until the implementation of the ingress capability for the standalone OpenShift.
HyperShift doesn't use the authentication operator, the authentication server is [managed by HyperShift's control plane directly](https://github.com/openshift/hypershift/blob/7cb87788779ae928d90d0e7760c1e9359f04e58a/control-plane-operator/controllers/hostedcontrolplane/oauth/deployment.go#L123-L127).

#### Monitoring operator

No change to the monitoring operator is needed until the implementation of the ingress capability for the standalone OpenShift.
The monitoring operator skips the alert manager's route check if `ingress` cluster operator is not there.

### Risks and Mitigations

#### Partially implemented capability

The ingress capability will become a part of the capability API. Meanwhile its implementation will be incomplete due to the missing support of the standalone OpenShift. However the usage of the capability is an important stepping stone, it unifies the implementation of the optional ingress for all the OpenShift installation types.

#### Component routes

[The component routes](https://docs.openshift.com/container-platform/4.13/rest_api/config_apis/ingress-config-openshift-io-v1.html#spec-componentroutes) is an optional list of routes that are managed by OpenShift components that a cluster-admin is able to configure the hostname and serving certificate for.
The two main OpenShift components which use the component routes are:
[the web console](https://docs.openshift.com/container-platform/4.13/web_console/customizing-the-web-console.html#customizing-the-console-route_customizing-web-console) and [the authentication](https://docs.openshift.com/container-platform/4.13/authentication/configuring-internal-oauth.html#customizing-the-oauth-server-url_configuring-internal-oauth).
The console and the authentication operators implement the component routes by watching [the Ingress Config API](https://docs.openshift.com/container-platform/4.13/rest_api/config_apis/ingress-config-openshift-io-v1.html) and updating the corresponding routes.    
The ingress operator creates the required role and role binding for the OpenShift component's service account so that it can get the custom service certificate secret from the `openshift-config` namespace.  
In the absence of the ingress operator, the component routes:
- cannot be customized by the owner components due to the lack of the RBAC
- become not implemented due to [the absence of the default ingress controller](#drawbacks)

The cluster instance administrator will still be able use [the Ingress Config API](https://docs.openshift.com/container-platform/4.13/rest_api/config_apis/ingress-config-openshift-io-v1.html)
but the component owner (operator) won't be able to access the secret or rely on the route implementation.     
This may be misleading for the cluster instance administrator. However the component routes were never fully implemented on HyperShift.
For instance, the authentication operator which is supposed to implement the authentication component route is not used at all.

#### Ingress to route controller

[The Kubernetes Ingress API](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.27/#ingress-v1-networking-k8s-io) on OpenShift is implemented through translation into routes.
[The ingress-to-route controller](https://github.com/openshift/route-controller-manager#ingress-to-route-controller) is the component responsible for this functionality. When the default ingress controller is not present, the routes are left without serving, resulting in the ingresses also being unserved. On HyperShift this can be mitigated by [any ingress controller implementation](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/).
In the context of this enhancement, [AWS Load Balancer Operator](/enhancements/ingress/aws-load-balancer-operator.md) is the expected implementation.

### Drawbacks

In the absence of the default ingress controller, the Route API is left unserved. OpenShift components or workloads can create routes, but there is no available controller to support them. An important consequence of this is the impact on the ingress API which relies on [the ingress to route translation](#ingress-to-route-controller). All this may be misleading for the users.

## Design Details

### Open Questions [optional]

N/A

### Test Plan
TBD

### Graduation Criteria

#### Dev Preview -> Tech Preview
N/A

#### Tech Preview -> GA
N/A

#### Removing a deprecated feature
N/A

### Upgrade / Downgrade Strategy

Same as [the component selection strategy](/enhancements/installer/component-selection.md#upgrade--downgrade-strategy).     
Note that:
- the capabilities [cannot disabled post install](/enhancements/installer/component-selection.md#capabilities-cannot-be-uninstalled)
- the capabilities can be [implicitly enabled](/enhancements/installer/component-selection.md#updates) (e.g. cluster version with `None` baseline upgrades to the version which moves some of the core payloads to a capability)

### Version Skew Strategy
N/A

### Operational Aspects of API Extensions
N/A - no api extensions are being introduced in this proposal

#### Failure Modes
N/A - no api extensions are being introduced in this proposal

#### Support Procedures
N/A - no api extensions are being introduced in this proposal

## Implementation History
N/A

## Alternatives

### Implement the ingress capability for the standalone OpenShift

The goal of this enhancement is to satisfy the HyperShift's requirements. The implementation of the ingress capability for the standalone OpenShift brings a lot of additional complexity, unnecessary for HyperShift. Most probably the following components would need to be updated to tolerate the absence of the ingress operator:
- The openshift installer. It [checks for the admission of the console route](https://github.com/wking/openshift-installer/blob/master/cmd/openshift-install/create.go#L543) by the ingress controller.
- The authentication operator. It checks the availability of [the authentication server's](https://github.com/openshift/cluster-authentication-operator/blob/da39951a53ad95be28e32a48f278cc23f41e99a7/pkg/controllers/oauthendpoints/oauth_endpoints_controller.go#L30-L31) 
and [console](https://github.com/openshift/cluster-authentication-operator/blob/da39951a53ad95be28e32a48f278cc23f41e99a7/pkg/controllers/configobservation/console/observe_consoleurl.go#L15) routes.
- [The component routes API](https://docs.openshift.com/container-platform/4.13/rest_api/config_apis/ingress-config-openshift-io-v1.html#status-componentroutes). The ingress operator is [responsible for the RBAC](https://github.com/openshift/cluster-ingress-operator/blob/master/pkg/operator/controller/configurable-route/controller.go#L42-L52)
which the component operators need to be able to access the custom TLS certificates from `openshift-config` namespace.

**Note**: it's difficult to anticipate the impact of the ingress operator removal on the installation process without the actual capability implementation.

### Allow standalone OpenShift to be broken if the ingress is disabled

This approach implies the implementation of the ingress capability the way it's described in [how to implement a new capability](/enhancements/installer/component-selection.md#how-to-implement-a-new-capability).
But unlike this enhancement, it doesn't have a goal to tolerate the absence of the ingress operator.
That is, anything which can break during or after the cluster installation is allowed to break.
This approach has a clear downside of the postponed failure which questions the correctness of the capability implementation from the user's point of view.
The upside is the simpler implementation and the absence of the hidden magic going on under the covers. The magic which a user might accidentally or deliberately circumvent by, for example, rendering the cluster resources and then modifying the `ClusterVersion` object prior to creating the cluster.

### Forbid the standalone installation without the ingress capability

This approach is an alternative to the proposed implementation of the required capabilities in the cluster version operator.
The idea is to make the installer forbid the installation without the ingress capability.
This could be achieved either via a validation error [after](https://github.com/openshift/installer/blob/master/pkg/types/validation/installconfig.go#L59) the unmarshaling of the install config or via the implicit enabling of the capability [before](https://github.com/openshift/installer/blob/master/pkg/types/defaults/installconfig.go#L32) it's passed to the cluster version operator.      

While having similarly poor level of the user experience the implementation in the cluster version operator has some advantages over this approach:
- it's a better fit for different OpenShift form factors: standalone, SNO, etc.
- the required capability logic pairs well with [the existing implicit enabling](https://github.com/openshift/cluster-version-operator/blob/master/pkg/cvo/status.go#L161-L165), so both are better to belong to the same repository.

### Alternative to the cluster capability to disable the ingress operator

Due to the lack of understanding of how HyperShift uses the cluster version operator to manage the ingress operator, we thought that the capability would serve only the informative purpose for the other OpenShift components like the console operator.
That brought the idea of giving up with the capability in favor of some other cluster resource (existing or not) which could fulfill the need of informing other OpenShift components.
Potentially that could save us from the difficult design and implementation decisions which the cluster capability implies. [The hypershift chapter](#hypershift) explains in details why the capability should be preferred over any other API.

### Add Route API to the ingress capability

The Route API is a fundamental part of OpenShift, a lot of OpenShift components and user workloads rely on the routes. The impact of the Router API removal is difficult to anticipate and plan.
Unlike the absence of the ingress operator which can be simulated without a dedicated capability, the removal of the route api needs the code changes in the [openshift-api-server](https://github.com/openshift/openshift-apiserver/tree/master/pkg/route).
This puts the implementation of the RFE at risk. In the situation when the ingress is disabled the choice lies in between the following options:
1. Keep the Route API and let the OpenShift components create routes which are not served by any ingress controller.
2. Keep the Route API and let the OpenShift components use an alternative ingress which supports the Route API (e.g. [AWS Load Balancer Controller](/enhancements/ingress/aws-load-balancer-operator.md)).
3. Remove the Route API and leave the OpenShift components adapt to the situation which implies the code changes to stop creating the routes and to migrate to a different ingress API (e.g. Kubernetes Ingress).

The first option seems to be the easiest (for the implementation), the second is likely the most suitable for the long term needs (of the RFE this enhancement tries to address) and the third seems to be the most explicit and "honest" (with respect to the api users) but the most intrusive at the same time.
The choice is not obvious as all the options have drawbacks.    
However the second one seems to be the better match for the ROSA users which already want to use the AWS load balancers.
Although the implementation of the second option requires the upstream work (AWS Load Balancer Controller doesn't support non Kubernetes APIs) and deserves a dedicated enhancement (and RFE).
Since the first option doesn't exclude the second one it seems to be pragmatic to pick it leaving the room for the follow up.

### Add multiple cluster capabilities

The ingress operator supports multiple ingress options:
- Route API which is implemented by the HAProxy based ingress controller (default)
- Gateway API which is implemented using the OpenShift Service Mesh operator (feature gated)

Both of these options could have been separate capabilities, additional to the ingress capability.
This alternative can be implemented later when the standalone OpenShift support will be needed.
