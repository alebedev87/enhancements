---
title: aws-dual-stack-support-for-ingresscontrollers
authors:
  - TBD
reviewers: # Include a comment about what domain expertise a reviewer is expected to bring and what area of the enhancement you expect them to focus on. For example: - "@networkguru, for networking aspects, please look at IP bootstrapping aspect"
  - TBD
approvers: # This should be a single approver. The role of the approver is to raise important questions, ensure the enhancement receives reviews from all applicable areas/SMEs, and determine when consensus is achieved such that the EP can move forward to implementation.  Having multiple approvers makes it difficult to determine who is responsible for the actual approval. Team leads and staff engineers often make good approvers.
  - TBD
api-approvers: # In case of new or modified APIs or API extensions (CRDs, aggregated apiservers, webhooks, finalizers). If there is no API change, use "None". Once your EP is published, ask in #forum-api-review to be assigned an API approver.
  - TBD
creation-date: 2026-02-02
last-updated: 2026-02-02
tracking-link:
  - https://issues.redhat.com/browse/NE-2021
status: provisional
see-also:
  - TBD
replaces:
  - TBD
superseded-by:
  - TBD
---

# AWS Dual-stack Support for IngressControllers

## Summary

This enhancement enables cluster administrators to configure dual-stack
(IPv4 and IPv6) IP address types for IngressController publishing
services on AWS clusters using Network Load Balancers (NLB). Currently,
IngressControllers only support single-stack IPv4 load balancers. This
enhancement extends the IngressController API to allow specification of
dual-stack IP address types, enabling services to be accessible over
both IPv4 and IPv6 networks simultaneously. The existing DNSRecord
functionality works without modification for AWS dual-stack support, as
AWS NLB provides a hostname that automatically resolves to both IPv4 and
IPv6 addresses.

## Motivation

AWS Network Load Balancers support dual-stack IP address types, allowing
services to be accessible via both IPv4 and IPv6 addresses. However,
OpenShift IngressControllers currently lack the ability to leverage this
capability. As IPv6 adoption increases and organizations require
dual-stack networking for compliance, accessibility, or future-proofing
their infrastructure, OpenShift needs to provide a way to configure
IngressControllers with dual-stack support.

AWS Classic Load Balancers do not support dual-stack, so this feature
is specifically targeted at clusters using Network Load Balancers on AWS.

### User Stories

* As a cluster administrator deploying on AWS with NLB, I want to
  configure my IngressController to use dual-stack IP addresses so that
  my applications are accessible over both IPv4 and IPv6 networks.

* As a network engineer managing OpenShift clusters, I want to specify
  dual-stack IP address types for ingress traffic so that I can meet
  organizational requirements for IPv6 support and future-proof the
  network infrastructure.

* As a cluster administrator, I want to migrate existing IPv4-only
  IngressControllers to dual-stack configuration so that I can support
  clients connecting over IPv6 without disrupting existing IPv4 traffic.

* As a platform operations engineer, I want to monitor and troubleshoot
  dual-stack IngressControllers at scale so that I can ensure reliable
  ingress traffic handling across both IPv4 and IPv6 networks.

### Goals

* Enable cluster administrators to specify dual-stack IP address types
  for IngressController publishing services on AWS with NLB.

* Ensure dual-stack IngressControllers work correctly across different
  cluster topologies including standalone clusters, Hypershift/hosted
  control planes, single-node deployments, and MicroShift.

* Provide clear error messages and validation when attempting to use
  dual-stack on unsupported platforms or load balancer types.

* Support smooth upgrades and downgrades with feature gate control,
  preserving existing single-stack configurations.

* Maintain backward compatibility with existing IngressController
  configurations.

### Non-Goals

* Supporting dual-stack IngressControllers on cloud platforms other than
  AWS (e.g., Azure, GCP) in the initial implementation.

* Supporting dual-stack with AWS Classic Load Balancers, which do not
  have dual-stack capability.

* Implementing IPv6-only (single-stack IPv6) IngressControllers as part
  of this enhancement.

## Proposal

This enhancement proposes extending the IngressController API to include
a new field for specifying the IP address type (IPv4, dual-stack) for
the publishing service. The implementation will:

1. Add a new API field to the IngressController CRD to specify IP
   address type for the load balancer service.

2. Implement validation to ensure dual-stack is only allowed on
   supported platforms (AWS with NLB).

3. Update the ingress-operator to configure the underlying load balancer
   service with the appropriate IP address type annotation.

4. Ensure the router pods and infrastructure can handle traffic from
   both IPv4 and IPv6 addresses.

5. Gate this functionality behind the `IngressControllerDualStack`
   feature gate in the TechPreviewNoUpgrade feature set initially.

Note: DNSRecord resource management continues to work without
modification, as it already supports hostname-based targets. AWS NLB
provides a dual-stack hostname that automatically resolves to both IPv4
and IPv6 addresses.

### Workflow Description

**cluster administrator** is a human user responsible for configuring
and managing IngressControllers in an OpenShift cluster.

**ingress-operator** is the OpenShift operator responsible for managing
IngressController resources and their associated router deployments.

1. The cluster administrator creates or updates an IngressController
   resource, specifying dual-stack as the desired IP address type in the
   AWS Network Load Balancer parameters under endpointPublishingStrategy
   (`.spec.endpointPublishingStrategy.loadBalancer.providerParameters.aws.networkLoadBalancer.ipAddressType`).

2. The ingress-operator receives the IngressController resource update
   and validates that:
   - The cluster is running on AWS
   - The load balancer type is Network Load Balancer (NLB)
   - The `IngressControllerDualStack` feature gate is enabled
   - The cluster network is configured for dual-stack

3. If validation passes, the ingress-operator configures the
   LoadBalancer service with the appropriate AWS-specific annotations
   to request a dual-stack Network Load Balancer.

4. AWS provisions a dual-stack NLB with a hostname (e.g.,
   `abc123.elb.us-east-1.amazonaws.com`) that resolves to both IPv4 and
   IPv6 addresses.

5. The Kubernetes LoadBalancer service status is updated with the AWS
   NLB hostname in `status.loadBalancer.ingress[].hostname`.

6. The ingress-operator creates or updates the DNSRecord resource with
   the AWS NLB hostname as the target.

7. The DNSRecord controller creates an alias record (Route53 alias)
   pointing the wildcard domain (e.g., `*.apps.example.com`) to the AWS
   NLB hostname.

8. When clients query the wildcard domain, DNS resolution follows the
   alias to the AWS NLB hostname, which AWS automatically resolves to
   both IPv4 (A records) and IPv6 (AAAA records) addresses based on the
   query type.

9. The ingress-operator updates the IngressController status to reflect
   the dual-stack configuration.

10. The cluster administrator verifies the IngressController status and
    confirms that DNS queries return both A and AAAA records for the
    wildcard domain.

11. Applications exposed through routes become accessible via both IPv4
    and IPv6 addresses.

#### Variation: Changing IP Address Type

1. The cluster administrator updates an existing IngressController to
   change the IP address type from IPv4 to Dualstack (or vice versa).

2. The ingress-operator updates the LoadBalancer service annotation
   `service.beta.kubernetes.io/aws-load-balancer-ip-address-type`.

3. AWS reconfigures the Network Load Balancer to the new IP address
   type. The NLB hostname remains the same.

4. For IPv4 → Dualstack: AWS starts resolving the hostname to both IPv4
   (A records) and IPv6 (AAAA records).

5. For Dualstack → IPv4: AWS stops publishing IPv6 addresses; the
   hostname only resolves to IPv4 (A records).

6. The DNSRecord resource remains unchanged since the NLB hostname stays
   the same.

7. Applications continue to be accessible with minimal disruption. IPv4
   clients continue to work throughout the transition.

#### Variation: Unsupported Platform or Load Balancer Type

1. The cluster administrator attempts to configure dual-stack on a
   cluster using Classic Load Balancer or on a non-AWS platform.

2. The ingress-operator validates the configuration and rejects it with
   a clear error message indicating that dual-stack is only supported on
   AWS with Network Load Balancers.

3. The IngressController status condition is updated to reflect the
   degraded state with the validation error.

### API Extensions

This enhancement modifies the IngressController CRD in the
`operator.openshift.io/v1` API group. No changes are required to the
DNSRecord API for AWS dual-stack support.

#### IngressController API Changes

A new optional field will be added to the AWS Network Load Balancer
parameters within the `EndpointPublishingStrategy` to specify the IP
address type:

The `AWSNetworkLoadBalancerParameters` struct already exists and contains
other NLB-specific configuration options. This enhancement adds a new
field to this existing struct:

```go
// AWSNetworkLoadBalancerParameters holds configuration parameters for an
// AWS Network Load Balancer.
type AWSNetworkLoadBalancerParameters struct {
    // other existing fields...

    // ipAddressType specifies the IP address type for the Network Load
    // Balancer.
    // Valid values are "IPv4" and "Dualstack".
    // "IPv4" configures a load balancer with IPv4 addresses only (default).
    // "Dualstack" configures a load balancer with both IPv4 and IPv6
    // addresses.
    //
    // This field is only applicable to AWS Network Load Balancers. Classic
    // Load Balancers do not support dual-stack configuration.
    //
    // When set to "Dualstack", AWS provisions a Network Load Balancer with
    // a hostname that resolves to both IPv4 (A records) and IPv6 (AAAA
    // records). Clients can connect using either IPv4 or IPv6.
    //
    // This field is gated by the IngressControllerDualStack feature gate.
    //
    // +optional
    // +kubebuilder:default:="IPv4"
    // +default="IPv4"
    // +openshift:enable:FeatureGate=IngressControllerDualStack
    IPAddressType IPAddressType `json:"ipAddressType,omitempty"`
}

// IPAddressType defines the IP address type for a load balancer.
// +kubebuilder:validation:Enum=IPv4;Dualstack
// +openshift:validation:FeatureGateAwareEnum:featureGate=IngressControllerDualStack,enum=Dualstack
type IPAddressType string

const (
    // IPv4IPAddressType configures the load balancer with IPv4 addresses
    // only.
    IPv4IPAddressType IPAddressType = "IPv4"

    // DualstackIPAddressType configures the load balancer with both IPv4
    // and IPv6 addresses.
    DualstackIPAddressType IPAddressType = "Dualstack"
)
```

This API extension adds a new optional field to the AWS Network Load
Balancer parameters. When set to "Dualstack", the ingress-operator will
configure the load balancer service with AWS-specific annotations to
request a dual-stack Network Load Balancer.

**Example IngressController Configuration:**

The following example shows how to configure a dual-stack IngressController
on AWS using Network Load Balancer:

```yaml
apiVersion: operator.openshift.io/v1
kind: IngressController
metadata:
  name: apps-dualstack
  namespace: openshift-ingress-operator
spec:
  domain: apps-dualstack.example.com
  endpointPublishingStrategy:
    type: LoadBalancerService
    loadBalancer:
      scope: External
      providerParameters:
        type: AWS
        aws:
          type: NLB
          networkLoadBalancer:
            ipAddressType: Dualstack
```

Key points:
- `scope: External` configures an external-facing load balancer (Internal
  scope also supports NLB but uses private subnets)
- `providerParameters.type: AWS` specifies AWS-specific parameters
- `providerParameters.aws.type: NLB` specifies Network Load Balancer
  (required for dual-stack; Classic Load Balancers do not support
  dual-stack)
- `providerParameters.aws.networkLoadBalancer.ipAddressType: Dualstack`
  enables dual-stack support (this field is gated by the
  IngressControllerDualStack feature gate)
- For IPv4-only load balancers (default behavior), omit the
  `ipAddressType` field or set it to `IPv4`

**Example LoadBalancer Service Created by Ingress Operator:**

When the ingress-operator processes the above IngressController
configuration, it creates a LoadBalancer service with the appropriate
annotations:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: router-apps-dualstack
  namespace: openshift-ingress
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
    service.beta.kubernetes.io/aws-load-balancer-ip-address-type: "dualstack"
spec:
  type: LoadBalancer
  selector:
    ingresscontroller.operator.openshift.io/deployment-ingresscontroller: apps-dualstack
  ports:
  - name: http
    port: 80
    protocol: TCP
    targetPort: http
  - name: https
    port: 443
    protocol: TCP
    targetPort: https
status:
  loadBalancer:
    ingress:
    - hostname: abc123-1234567890.elb.us-east-1.amazonaws.com
```

The AWS cloud controller manager provisions a dual-stack NLB and
populates the hostname in the service status. This hostname resolves to
both IPv4 (A records) and IPv6 (AAAA records).

**Note**: When changing the `ipAddressType` between IPv4 and Dualstack
on an existing IngressController, AWS keeps the same NLB hostname. Only
the IP address resolution behavior changes (adding or removing AAAA
records). This means DNS records pointing to the NLB hostname do not
need to be updated.

**Example Combining ipAddressType with Other NLB Parameters:**

The `ipAddressType` field can be used alongside other existing AWS NLB
parameters such as `eipAllocations` (Elastic IP addresses). The
following example shows a dual-stack IngressController using specific
Elastic IP addresses:

```yaml
apiVersion: operator.openshift.io/v1
kind: IngressController
metadata:
  name: apps-dualstack-eip
  namespace: openshift-ingress-operator
spec:
  domain: apps-eip.example.com
  endpointPublishingStrategy:
    type: LoadBalancerService
    loadBalancer:
      scope: External
      providerParameters:
        type: AWS
        aws:
          type: NLB
          networkLoadBalancer:
            eipAllocations:
            - eipalloc-0956fea34de4cb7ab
            - eipalloc-0e9a3077a70de050a
            - eipalloc-0b69fc4691f54cdd0
            ipAddressType: Dualstack
```

In this configuration:
- The NLB uses the specified Elastic IP addresses
- The NLB is configured for dual-stack (both IPv4 and IPv6)
- Each EIP will have both IPv4 and IPv6 addresses assigned
- The number of EIP allocations must match the number of availability
  zones/subnets

#### DNSRecord Resource Changes

On AWS, when a dual-stack Network Load Balancer is provisioned, AWS
automatically assigns a hostname (e.g., `abc123.elb.us-east-1.amazonaws.com`)
that resolves to both IPv4 (A records) and IPv6 (AAAA records). This
hostname is populated in the LoadBalancer service's
`status.loadBalancer.ingress[].hostname` field.

**No DNSRecord API changes are required** for AWS dual-stack support.
The existing DNSRecord implementation already supports hostname-based
targets and will work as-is:

- The ingress-operator will populate the DNSRecord's `spec.targets`
  field with the AWS-provided hostname from the load balancer service
  status.
- The DNSRecord controller will create an alias record (Route53 alias
  or CNAME) pointing the wildcard domain (e.g., `*.apps.example.com`)
  to the AWS NLB hostname.
- When clients query the wildcard domain, DNS resolution follows the
  alias to the AWS NLB hostname, which AWS automatically resolves to
  both IPv4 and IPv6 addresses based on the query type (A or AAAA).
- This approach leverages AWS's native dual-stack DNS management without
  requiring DNSRecord to handle IPv6 addresses directly.

Example DNSRecord for a dual-stack IngressController on AWS:

```yaml
apiVersion: ingress.operator.openshift.io/v1
kind: DNSRecord
metadata:
  name: default-wildcard
  namespace: openshift-ingress-operator
spec:
  dnsName: '*.apps.example.com'
  targets:
  - abc123.elb.us-east-1.amazonaws.com  # AWS NLB hostname (has both A and AAAA)
  recordType: CNAME
  recordTTL: 30
```

**Implementation Approach:**

The ingress-operator's existing DNS management logic continues to work
without modification for AWS dual-stack:
1. Extract hostname from LoadBalancer service `status.loadBalancer.ingress`
2. Populate DNSRecord targets with the hostname (existing behavior)
3. DNSRecord controller creates alias/CNAME record (existing behavior)
4. AWS handles dual-stack DNS resolution transparently

**Note on future platform support:** If dual-stack support is extended
to other cloud platforms that expose individual IPv4 and IPv6 addresses
instead of a dual-stack hostname, DNSRecord may need enhancements to
handle multiple IP addresses and create both A and AAAA records directly.

### Topology Considerations

#### Hypershift / Hosted Control Planes

This enhancement is applicable to Hypershift deployments. In Hypershift
architectures, the IngressController runs in the hosted cluster and the
load balancer is provisioned in the hosting infrastructure. The dual-stack
configuration will work as long as:

- The hosting AWS infrastructure supports dual-stack networking
- The hosted cluster's network is configured for dual-stack
- Network Load Balancers are used

No special modifications are required for the management cluster
components.

#### Standalone Clusters

This enhancement is fully applicable to standalone clusters running on
AWS with Network Load Balancers. This is the primary use case for this
feature.

#### Single-node Deployments or MicroShift

Single-node deployments (SNO) and MicroShift can benefit from this
enhancement if they are deployed on AWS with NLB and require dual-stack
ingress capabilities. The resource consumption impact is minimal, as the
dual-stack configuration only affects the load balancer provisioning and
does not significantly increase CPU or memory usage of the
IngressController operator or router pods.

For MicroShift, if dual-stack ingress capabilities are desired, the
relevant configuration options should be exposed through the MicroShift
configuration file in addition to the IngressController API.

#### OpenShift Kubernetes Engine

This enhancement is applicable to OKE deployments on AWS that use
Network Load Balancers. OKE includes ingress capabilities, and dual-stack
support for IngressControllers would be available if the underlying
infrastructure supports it.

### Implementation Details/Notes/Constraints

The implementation involves changes to the following components:

1. **openshift/api**: Add the new `IPAddressType` field to the
   AWS Network Load Balancer parameters
   (`AWSNetworkLoadBalancerParameters`) in the IngressController API with
   appropriate feature gate markers. Update the API validation to ensure
   the field is only accepted when the `IngressControllerDualStack`
   feature gate is enabled.

2. **openshift/cluster-ingress-operator**: Update the ingress-operator
   to:
   - Read the new `IPAddressType` field from IngressController resources
   - Validate that dual-stack is only configured on AWS with NLB
   - Set the appropriate service annotations for AWS load balancer
     (e.g., `service.beta.kubernetes.io/aws-load-balancer-ip-address-type:
     dualstack`)
   - Update DNSRecord resources with the AWS NLB hostname from the
     service status (existing behavior, no changes needed)
   - Update IngressController status to reflect dual-stack configuration
   - Handle upgrades/downgrades based on feature gate state

3. **Feature Gate**: Create a new feature gate `IngressControllerDualStack`
   in https://github.com/openshift/api/blob/master/features/features.go
   with the TechPreviewNoUpgrade feature set initially. Per
   dev-guide/feature-zero-to-hero.md, all new features must be gated
   behind a feature gate and start disabled by default.

4. **Validation**: Implement CRD validation and runtime validation to:
   - Reject dual-stack configuration on non-AWS platforms
   - Reject dual-stack configuration with Classic Load Balancers
   - Provide clear, actionable error messages

**Constraints:**
- Dual-stack support is limited to AWS with NLB only in this initial
  implementation
- Requires the cluster network to be configured for dual-stack
- The underlying VPC and subnets must support IPv6
- AWS NLB must be used (Classic Load Balancers do not support dual-stack)

### Risks and Mitigations

**Risk**: Users may attempt to configure dual-stack on unsupported
platforms or load balancer types, leading to confusion.

**Mitigation**: Implement comprehensive validation with clear error
messages. Document the platform and load balancer requirements
prominently in the API documentation and user-facing documentation.

**Risk**: Dual-stack configuration may interact unexpectedly with
existing network policies or firewall rules that only account for IPv4.

**Mitigation**: Provide clear documentation on network requirements and
considerations for dual-stack deployments. Include examples of network
policies that work with dual-stack configurations.

**Risk**: Upgrading or downgrading across versions with different feature
gate states could lead to unexpected behavior.

**Mitigation**: The feature gate will control the behavior. When the
feature gate is disabled, the field will be hidden from the CRD and any
existing dual-stack configurations will be ignored. Clear documentation
on upgrade/downgrade behavior will be provided.

**Risk**: DNS record propagation delays or failures could result in the
wildcard domain not resolving to the AWS NLB hostname, making the
IngressController inaccessible.

**Mitigation**: The ingress-operator will verify that DNSRecord resources
are successfully updated and will report status conditions if DNS
publishing fails. Monitoring and alerting on DNSRecord status conditions
will help detect and remediate DNS issues quickly.

**Note on operational safety**: Changing the `ipAddressType` field
between IPv4 and Dualstack (or vice versa) is a low-risk operation. AWS
maintains the same NLB hostname throughout the change; only the IP
address resolution behavior changes (adding or removing AAAA records).
This means:
- No DNS record updates are required
- IPv4 connectivity remains stable throughout the transition
- The change can be easily reverted if needed

### Drawbacks

* This enhancement is platform-specific (AWS only initially), which may
  create expectations for similar support on other cloud platforms.

* Dual-stack support adds complexity to the IngressController
  configuration and may require additional expertise to troubleshoot
  networking issues.

* Organizations may need to update firewall rules, network policies, and
  monitoring systems to account for IPv6 traffic.

* Troubleshooting connectivity issues requires checking both IPv4 and
  IPv6 paths, as clients may connect via either protocol.

## Alternatives (Not Implemented)

### Alternative 1: Support All Cloud Platforms Simultaneously

Instead of limiting initial support to AWS with NLB, we could implement
dual-stack support across all cloud platforms (Azure, GCP, etc.)
simultaneously.

**Why not chosen**: Different cloud platforms have varying levels of
dual-stack support and different APIs for configuring it. Implementing
for all platforms at once would significantly increase the scope and
delay delivery. Starting with AWS allows us to validate the approach
and gather feedback before expanding to other platforms.

### Alternative 2: IPv6-Only Support

Instead of dual-stack, implement IPv6-only (single-stack IPv6)
IngressControllers.

**Why not chosen**: IPv6-only configurations are less common in practice,
and most organizations requiring IPv6 support need dual-stack to
maintain compatibility with existing IPv4 infrastructure. Dual-stack
provides more value and flexibility.

### Alternative 3: Automatic Detection and Configuration

Automatically detect when a cluster supports dual-stack and
automatically configure IngressControllers with dual-stack without
requiring explicit configuration.

**Why not chosen**: Automatic configuration could lead to unexpected
behavior and networking issues if users are not prepared for IPv6
traffic. Explicit opt-in gives cluster administrators control over when
dual-stack is enabled and ensures they can prepare their network
infrastructure accordingly.

## Open Questions [optional]

## Test Plan

<!-- TODO: This section needs to be filled in with detailed test plan -->

Tests must include the following labels per dev-guide/feature-zero-to-hero.md:
- `[OCPFeatureGate:IngressControllerDualStack]` for the feature gate
- `[Jira:"Routing"]` for the ingress/routing component
- Appropriate test type labels like `[Suite:...]`, `[Serial]`, `[Slow]`,
  or `[Disruptive]` as needed

Reference dev-guide/test-conventions.md for additional test labeling
conventions.

**Test Strategy:**
- Unit tests for API validation and operator logic
- Integration tests for IngressController configuration on AWS with NLB
- E2E tests verifying:
  - Dual-stack IngressController can be created and configured
  - Load balancer service is provisioned with dual-stack annotation
  - AWS NLB hostname is populated in service status
  - DNSRecord resources are created/updated with the AWS NLB hostname as
    target
  - DNS alias record points the wildcard domain to the AWS NLB hostname
  - AWS NLB hostname resolves to both IPv4 and IPv6 addresses
  - Applications are accessible via both IPv4 and IPv6
  - DNS queries for the wildcard domain return both A and AAAA records
    (via the alias to the AWS NLB hostname)
  - IPv4-only IngressControllers continue to work unchanged (backward
    compatibility)
  - Validation errors occur on unsupported platforms/load balancer types
  - Upgrade/downgrade scenarios work correctly with feature gate control

## Graduation Criteria

<!-- TODO: Define graduation criteria based on dev-guide/feature-zero-to-hero.md -->

Per dev-guide/feature-zero-to-hero.md, promotion from TechPreviewNoUpgrade
to Default requires:
- Minimum 5 tests with the `[OCPFeatureGate:IngressControllerDualStack]`
  label
- Tests run at least 7 times per week
- Tests run at least 14 times per supported platform
- 95% pass rate across all tests
- Tests running on all supported platforms (AWS, Azure, GCP, vSphere,
  Baremetal with various network stacks)
- Note: For this feature, tests on non-AWS platforms may be skipped or
  should verify that dual-stack is properly rejected on unsupported
  platforms

### Dev Preview -> Tech Preview

N/A - Starting directly in Tech Preview (TechPreviewNoUpgrade feature set)

### Tech Preview -> GA

- Successful deployment and operation of dual-stack IngressControllers
  in production-like environments
- User facing documentation created in openshift-docs
- Sufficient test coverage as outlined above
- Positive feedback from early adopters
- No major bugs or security issues identified
- Performance testing shows no degradation compared to IPv4-only
  IngressControllers
- Support procedures documented for troubleshooting dual-stack issues

### Removing a deprecated feature

N/A

## Upgrade / Downgrade Strategy

Upgrades and downgrades will be controlled by the feature gate state:

**Upgrade from version without feature to version with feature:**
- If the feature gate is disabled (default state), the new
  `IPAddressType` field is hidden from the CRD and not available for use
- Existing IngressControllers continue to work as IPv4-only
- Users can enable the TechPreviewNoUpgrade feature set to access the
  new dual-stack capability

**Upgrade with feature gate enabled:**
- Existing IngressControllers with dual-stack configuration continue to
  work with dual-stack
- The dual-stack configuration is preserved

**Downgrade to version without feature:**
- If the feature gate is enabled and dual-stack IngressControllers exist,
  the downgrade may result in the field being ignored or the
  IngressController reverting to IPv4-only behavior
- The AWS NLB hostname remains the same; only the IP address type changes
  (from dual-stack back to IPv4-only)
- DNS records remain unchanged as they point to the same NLB hostname
- Users should be warned about this possibility in the documentation
- The CVO may delete the new field as it no longer exists in the target
  version

**Downgrade with feature gate disabled:**
- No impact, as the feature was not in use

Clusters will remain available during upgrades/downgrades. The router
pods and load balancers will continue to serve traffic. When the IP
address type changes, the AWS NLB hostname remains the same and DNS
records are unaffected; only the NLB's IP address resolution behavior
changes (dual-stack vs IPv4-only).

## Version Skew Strategy

Version skew is primarily a concern during cluster upgrades when
different components may be at different versions:

- **ingress-operator and router pods**: The ingress-operator is
  responsible for configuring router pods. During an upgrade, older
  router pods will continue to function with their existing
  configuration (IPv4 or dual-stack). When the ingress-operator is
  updated, it will apply the current IngressController configuration to
  new router pods as they are rolled out.

- **API server and ingress-operator**: If the API server is upgraded
  before the ingress-operator, the operator will simply ignore the new
  field until it is also upgraded. If the ingress-operator is upgraded
  first, it will handle the absence of the field gracefully by
  maintaining current behavior.

- **Feature gate state**: The feature gate ensures consistent behavior
  across components. All components will respect the feature gate state
  when deciding whether to enable dual-stack functionality.

No special version skew handling is required beyond normal OpenShift
upgrade procedures.

## Operational Aspects of API Extensions

### API Extension Impact

This enhancement adds a new optional field to the IngressController CRD.
This is a relatively low-impact change as it:
- Does not introduce new CRDs
- Does not add webhooks or aggregated API servers
- Does not add finalizers
- Only extends an existing CRD with an optional field

**SLIs for Health Monitoring:**
- `ingress-operator` deployment health and pod status
- IngressController resource status conditions (Available, Degraded,
  Progressing)
- Load balancer service creation and configuration success/failure rates
- DNSRecord resource status conditions (Published)

**Impact on Existing SLIs:**
- No significant impact on API throughput expected, as this is a
  configuration field that is rarely changed
- Load balancer provisioning time may be slightly longer for dual-stack
  configurations compared to IPv4-only, but this is a one-time operation
  per IngressController
- No impact on pod scheduling or other cluster-wide operations

**Measurement and Monitoring:**
- Existing ingress-operator metrics and conditions will cover this
  functionality
- QE will validate through standard release testing processes
- Performance team should be consulted if any performance degradation is
  observed during testing

**Failure Modes:**
- **Invalid configuration**: User configures dual-stack on unsupported
  platform/load balancer type
  - Impact: IngressController will degrade with validation error in status
  - Detection: IngressController status condition shows Degraded=True
  - Teams: Ingress team (Routing)

- **Load balancer provisioning failure**: AWS fails to provision
  dual-stack NLB
  - Impact: IngressController shows Degraded=True, routes not accessible
  - Detection: IngressController status, cloud provider events
  - Teams: Ingress team, potentially AWS support

- **DNS record publishing failure**: Route53 fails to publish A or AAAA
  records for the dual-stack IngressController
  - Impact: Routes may be accessible via only IPv4 or IPv6, or not
    accessible at all depending on which records failed
  - Detection: DNSRecord status shows Failed or Degraded condition,
    IngressController may show DNSReady=False
  - Teams: Ingress team, potentially AWS support

- **Feature gate disabled with existing dual-stack config**: Feature gate
  is disabled on a cluster with dual-stack IngressControllers
  - Impact: Dual-stack field is ignored, IngressController may revert to
    IPv4-only
  - Detection: IngressController status shows IPv4 address only
  - Teams: Ingress team

## Support Procedures

**Detecting dual-stack configuration issues:**

Symptoms:
- IngressController status shows Degraded=True with condition message
  indicating validation error
- `ingress-operator` logs show errors related to dual-stack configuration
  or load balancer provisioning
- Load balancer service does not have a hostname in its status or the
  hostname does not resolve to both IPv4 and IPv6 addresses

Diagnosis:
1. Check IngressController status: `oc get ingresscontroller -n
   openshift-ingress-operator <name> -o yaml`
2. Review ingress-operator logs: `oc logs -n openshift-ingress-operator
   deployment/ingress-operator`
3. Check load balancer service status: `oc get svc -n openshift-ingress
   router-<name> -o yaml` - verify it has the dual-stack annotation and a
   hostname in `status.loadBalancer.ingress[].hostname`
4. Verify the AWS NLB hostname resolves to both IPv4 and IPv6: `dig A
   <nlb-hostname>` and `dig AAAA <nlb-hostname>`
5. Verify feature gate is enabled: `oc get featuregate cluster -o yaml`
6. Verify platform is AWS and load balancer type is NLB
7. Check DNSRecord status: `oc get dnsrecord -n openshift-ingress-operator
   <name>-wildcard -o yaml`
8. Verify DNS alias is published: `dig <wildcard-domain>` should show
   alias to AWS NLB hostname, and both `dig A <wildcard-domain>` and `dig
   AAAA <wildcard-domain>` should resolve

**Disabling dual-stack functionality:**

To disable dual-stack for a specific IngressController:
1. Edit the IngressController resource: `oc edit ingresscontroller -n
   openshift-ingress-operator <name>`
2. Remove the `ipAddressType: Dualstack` field from
   `.spec.endpointPublishingStrategy.loadBalancer.providerParameters.aws.networkLoadBalancer`
   or set it to `IPv4`
3. The ingress-operator will reconcile and update the load balancer to
   IPv4-only

Consequences:
- The load balancer will be reconfigured to IPv4-only
- The AWS NLB hostname remains the same, but it will only resolve to IPv4
  addresses (A records); AAAA records will no longer be published by AWS
- IPv6 connectivity will no longer be available
- Existing IPv4 connections continue to work; IPv6 connections may be
  disrupted during the transition
- DNS records (alias to the NLB hostname) remain unchanged; only the NLB
  hostname's resolution behavior changes

To disable the feature entirely:
1. Disable the TechPreviewNoUpgrade feature set (requires cluster
   re-installation as feature sets cannot be disabled once enabled)

**Detecting DNS record issues:**

Symptoms:
- Routes are not accessible or accessible via IPv4 but not IPv6, or vice
  versa
- DNSRecord status shows Failed or Degraded condition
- DNS queries for the wildcard domain don't return expected results

Diagnosis:
1. Check DNSRecord status: `oc get dnsrecord -n openshift-ingress-operator
   <name>-wildcard -o yaml`
2. Verify DNSRecord target contains the AWS NLB hostname
3. Verify the alias record exists: `dig <wildcard-domain>` should show
   CNAME/alias to AWS NLB hostname
4. Query the AWS NLB hostname directly: `dig A <nlb-hostname>` and `dig
   AAAA <nlb-hostname>` to verify AWS is resolving both
5. Query the wildcard domain: `dig A <wildcard-domain>` and `dig AAAA
   <wildcard-domain>` to verify both record types resolve (following the
   alias)
6. Check Route53 console to verify the alias record exists and points to
   the correct NLB
7. Review ingress-operator logs for DNS-related errors

**Graceful failure and recovery:**

The dual-stack configuration fails gracefully:
- If AWS fails to provision a dual-stack NLB, the IngressController will
  show Degraded=True and provide error details in the status
- If DNS record publication fails, the DNSRecord status will show the
  failure and the IngressController will report the issue in its status
  conditions
- If the feature gate is disabled, the field is ignored and existing
  behavior is maintained
- When re-enabled or reconfigured, the ingress-operator will reconcile
  the configuration and attempt to establish the dual-stack load balancer
  and publish the DNS alias record

## Infrastructure Needed [optional]

**CI Infrastructure:**
- AWS-based CI clusters with dual-stack VPC and subnet configuration
- Periodic jobs to run dual-stack IngressController tests on AWS
- Jobs should cover both TechPreviewNoUpgrade and Default feature sets
  (for post-promotion testing)

**Development/Testing:**
- Access to AWS accounts with IPv6-enabled VPCs for development and
  manual testing
- Documentation and examples for setting up dual-stack test environments
