# Understanding FailoverPolicy and Virtual Destinations

This document provides a comprehensive guide to understanding how FailoverPolicy and Virtual Destinations work together to enable resilient service communication across multiple clusters. The guide is based on a series of tests that demonstrate various scenarios and configurations.

## Table of Contents

1. [Introduction](#introduction)
2. [Key Concepts](#key-concepts)
3. [Test Environment Setup](#test-environment-setup)
4. [Step-by-Step Walkthrough](#step-by-step-walkthrough)
5. [Advanced Configurations](#advanced-configurations)
6. [Troubleshooting and Best Practices](#troubleshooting-and-best-practices)
7. [Conclusion](#conclusion)
8. [Diagrams](#diagrams)

## Introduction

In a multi-cluster Kubernetes environment, ensuring reliable service communication is critical. Virtual Destinations (VDs) and FailoverPolicies are powerful features that enable cross-cluster service discovery and traffic management. This guide will walk you through how these components work together to provide resilient service communication.

For a visual overview of the architecture, see the [Multi-Cluster Architecture diagram](#multi-cluster-architecture-with-virtual-destinations) in the Diagrams section.

## Key Concepts

### Virtual Destinations (VDs)

A Virtual Destination is a custom resource that:
- Creates a DNS name that can be used to access services across clusters
- Maps to actual services in one or more clusters
- Provides a consistent endpoint regardless of where the actual service instances are running
- Enables cross-namespace and cross-cluster service discovery

For a visual representation of how Virtual Destinations work, see the [Virtual Destination Configuration diagram](#virtual-destination-configuration-and-dns-resolution) in the Diagrams section.

Example of a Virtual Destination:
```yaml
apiVersion: networking.gloo.solo.io/v2
kind: VirtualDestination
metadata:
  labels:
    app: vuderani-vd-server
    identity: vuderani-vd-server
  name: vd-vuderani-for-test-pp-rs-infra
  namespace: infra
spec:
  hosts:
  - go-vuderani-algebra.infra.svc.test-pp-rs
  ports:
  - number: 80
    protocol: HTTP
    targetPort:
      name: http
  services:
  - labels:
      app: vuderani-vd-server
      mesh.opentable.com/serves-test-pp-rs: "true"
    namespace: infra
```

### FailoverPolicy

A FailoverPolicy defines:
- How traffic should be routed when services are unavailable
- Priority order for selecting service instances
- Labels used to determine routing decisions
- Which Virtual Destinations the policy applies to

For a visual representation of how FailoverPolicies work, see the [FailoverPolicy Configuration diagram](#failoverpolicy-priority-labels-and-selection-logic) in the Diagrams section.

Example of a FailoverPolicy:
```yaml
apiVersion: resilience.policy.gloo.solo.io/v2
kind: FailoverPolicy
metadata:
  name: fp-vuderani-for-test-pp-rs
  namespace: infra
  labels:
    app: vuderani-vd-server
spec:
  applyToDestinations:
  - kind: VIRTUAL_DESTINATION
    selector:
      name: vd-vuderani-for-test-pp-rs-infra
    port:
      number: 80
  config:
    priorityLabels:
      labels:
      - topology.istio.io/subzone
      - mesh.opentable.com/serves-test-pp-rs
```

## Test Environment Setup

Our test environment consists of:

1. Multiple Kubernetes clusters:
   - `central-pp-rs`: Primary cluster
   - `test-pp-rs`: Secondary cluster

2. Components in each cluster:
   - Server deployments (`vuderani-vd-server`)
   - Client deployments (`vuderani-vd-client`)
   - Virtual Destinations
   - FailoverPolicies

3. Management cluster:
   - Used to deploy and manage VDs and FailoverPolicies

## Step-by-Step Walkthrough

### Step 1: Create Servers and Clients

In this step, we deploy server and client applications to each cluster:

```bash
# Create servers and clients for each context
kubectl --context=${PREFIX}-${cluster} apply -f ../clients
kubectl --context=${PREFIX}-${cluster} apply -f ../servers-${CLUSTER_PREFIX}/server-${cluster}.yaml
```

The servers are deployed with specific labels that will be used by the FailoverPolicy to determine routing.

### Step 2: Install Virtual Destinations and FailoverPolicies

Next, we deploy the Virtual Destinations and FailoverPolicies to the management cluster:

```bash
# Install VDs and FailoverPolicies
kubectl --context=${MGMT_CONTEXT} apply -f ../vds-${CLUSTER_PREFIX}/vd-for-${cluster}.yaml
kubectl --context=${MGMT_CONTEXT} apply -f ../policies-${CLUSTER_PREFIX}/policy-for-${cluster}.yaml
```

The Virtual Destinations create DNS names that can be used to access the services, while the FailoverPolicies define how traffic should be routed.

### Step 3: Initial Request Tests

In this step, we test the initial configuration by sending requests from clients to the Virtual Destinations:

1. We collect `istioctl` output to understand the routing configuration
2. We send requests from clients in the same namespace
3. We send requests from clients in different namespaces

The expected behavior is that requests are routed to services in the same cluster as the client.

### Step 4: Label Remote Server

In this step, we add labels to the server in the `test-pp-rs` cluster to indicate that it can serve requests for the `central-pp-rs` cluster:

```bash
# Update test-pp-rs to serve both itself and central-pp-rs
kubectl --context=${TEST_CONTEXT} apply -f ../servers-labeled-${CLUSTER_PREFIX}
```

This is a key step that enables cross-cluster failover. The label `mesh.opentable.com/serves-central-pp-rs: "true"` indicates that this server can handle requests intended for the central cluster.

### Step 5: Verify Routing with Labels

After labeling, we repeat the request tests to verify that the routing behavior remains the same. At this point, the primary cluster's services are still available, so requests should still be routed to the local cluster.

### Step 6: Test Failover by Scaling Down Primary Cluster

Now we test the failover capability by scaling down the server deployment in the primary cluster:

```bash
# Scale central to 0
kubectl --context=${PREFIX}-central-${CLUSTER_PREFIX}-rs scale deployment vuderani-vd-server -n infra --replicas=0
```

We then send requests and observe that:
- Requests from the `central-pp-rs` cluster are now routed to the `test-pp-rs` cluster
- Requests from the `test-pp-rs` cluster continue to be served locally

This demonstrates the failover capability of the FailoverPolicy, which routes traffic to the next available service based on the priority labels.

For a visual representation of this failover scenario, see the [Failover Scenario diagram](#failover-scenario-primary-cluster-failure) in the Diagrams section.

### Steps 6.1 and 6.2: Test Different Scale Configurations

We test different scaling configurations to observe how the load is distributed:
- Scale remote server to 3 replicas
- Scale remote server to 9 replicas

The distribution of requests should reflect the number of available endpoints.

### Step 7: Restore Primary Cluster

We restore the primary cluster by scaling the server deployment back up:

```bash
# Scale central back to 9
kubectl --context=${PREFIX}-central-${CLUSTER_PREFIX}-rs scale deployment vuderani-vd-server -n infra --replicas=9
```

After this, requests from the `central-pp-rs` cluster should be routed back to the local services.

### Step 8: Update Policy Priority

In this step, we modify the FailoverPolicy to change the priority order of labels:

```bash
# Install failoverpolicy with inverted priorityLabels and AVG label in list
kubectl --context=${MGMT_CONTEXT} apply -f ../policies-${CLUSTER_PREFIX}/policy-for-${cluster}-inverted-with-avg.yaml
```

The inverted policy prioritizes the `mesh.opentable.com/serves-*-pp-rs` label over the `topology.istio.io/subzone` label, which changes the routing behavior.

We also add an additional label (`topology.istio.io/subzone` with AVG value) to the priority list to observe its effect on routing.

For a visual representation of how inverting priority labels affects routing, see the [Priority Label Inversion diagram](#effect-of-inverting-priority-labels) in the Diagrams section.

## Advanced Configurations

### Modifying Priority Labels

The priority labels in the FailoverPolicy determine the order in which service instances are selected:

```yaml
config:
  priorityLabels:
    labels:
    - topology.istio.io/subzone
    - mesh.opentable.com/serves-test-pp-rs
```

By changing the order of these labels, you can control the failover behavior:
- When `topology.istio.io/subzone` is first, traffic prefers services in the same zone
- When `mesh.opentable.com/serves-test-pp-rs` is first, traffic prefers services with this specific label

### Adding Outlier Detection

The tests also include OutlierDetectionPolicy, which automatically ejects failing endpoints:

```yaml
apiVersion: resilience.policy.gloo.solo.io/v2
kind: OutlierDetectionPolicy
metadata:
  name: ot-vuderani-for-test-pp-rs
spec:
  config:
    baseEjectionTime: 60s
    consecutiveErrors: 5
    interval: 1s
    maxEjectionPercent: 100
```

This policy ejects endpoints that experience 5 consecutive errors, improving the overall resilience of the system.

## Troubleshooting and Best Practices

### Verifying Configuration

Use `istioctl` to verify the routing configuration:

```bash
istioctl pc clusters --context=${PREFIX}-${cluster} ${POD}.${NS}
istioctl pc ep --context=${PREFIX}-${cluster} ${POD}.${NS}
istioctl pc routes --context=${PREFIX}-${cluster} ${POD}.${NS} -o yaml
```

### Testing Failover

To test failover scenarios:
1. Scale down deployments in the primary cluster
2. Send requests to the Virtual Destination
3. Verify that requests are routed to the secondary cluster

### Label Management

Carefully manage labels on your services:
- Use consistent labeling schemes across clusters
- Document the meaning and purpose of each label
- Test label changes in a non-production environment first

## Conclusion

Virtual Destinations and FailoverPolicies provide powerful capabilities for building resilient multi-cluster applications:

1. **Service Discovery**: VDs create stable DNS names that work across clusters
2. **Traffic Management**: FailoverPolicies control how traffic is routed
3. **Resilience**: Automatic failover when services are unavailable
4. **Flexibility**: Configurable priority labels for fine-grained control

By understanding and leveraging these features, you can build highly available applications that can withstand cluster failures and provide consistent service to users.

## Diagrams

The following diagrams illustrate key concepts and scenarios related to Virtual Destinations and FailoverPolicies:

### Multi-Cluster Architecture with Virtual Destinations

See [diagrams/01-multi-cluster-architecture.md](diagrams/01-multi-cluster-architecture.md)

This diagram shows the high-level architecture of the multi-cluster setup with Virtual Destinations and FailoverPolicies.

### Virtual Destination Configuration and DNS Resolution

See [diagrams/02-virtual-destination-configuration.md](diagrams/02-virtual-destination-configuration.md)

This diagram illustrates how a Virtual Destination creates a DNS name and maps to services in different clusters.

### FailoverPolicy Priority Labels and Selection Logic

See [diagrams/03-failover-policy-configuration.md](diagrams/03-failover-policy-configuration.md)

This diagram shows how FailoverPolicy prioritizes endpoints and applies to Virtual Destinations.

### Failover Scenario: Primary Cluster Failure

See [diagrams/04-failover-scenario.md](diagrams/04-failover-scenario.md)

This diagram illustrates what happens during a failover scenario when the primary cluster is scaled to zero.

### Effect of Inverting Priority Labels

See [diagrams/05-priority-label-inversion.md](diagrams/05-priority-label-inversion.md)

This diagram shows how inverting the priority labels in a FailoverPolicy affects traffic routing. 