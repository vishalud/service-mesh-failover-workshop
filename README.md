# FailoverPolicy and Virtual Destinations Demo

This repository contains tests and examples demonstrating how FailoverPolicy and Virtual Destinations work together to enable resilient service communication across multiple Kubernetes clusters.

## Test Steps

Step 1 - Create servers and clients
Step 2 - Install VDs and Failover Policies
Step 3 - Initial request tests - collect istioctl output as well as request distribution to VD from same and different NSs
Step 4 - Label remote server to serve local cluster
Step 5 - more request tests, expected to be same as step 3
Step 6 - scale local server to zero, more tests, expected to be 100% remote
Step 6.1 - scale remote server to 3, more tests
Step 6.2 - scale remote to 9, more tests
Step 7 - scale local back to 9, more tests
Step 7.1 - scale local to 3, more tests
Step 7.2 - scale local to 9, more tests
Step 8 - Update policy priority, invert order and add AVG, more tests
Step 8.1 - Update priority, remove AVG, more tests
Step 8.2 - update priority, invert again, more tests
Step 9 - Clean up

## Documentation

For a comprehensive guide to understanding FailoverPolicy and Virtual Destinations, see [Understanding_FailoverPolicy_and_VirtualDestinations.md](Understanding_FailoverPolicy_and_VirtualDestinations.md).

## Diagrams

The `diagrams/` directory contains Mermaid diagrams that illustrate key concepts:

1. [Multi-Cluster Architecture](diagrams/01-multi-cluster-architecture.md)
2. [Virtual Destination Configuration](diagrams/02-virtual-destination-configuration.md)
3. [FailoverPolicy Configuration](diagrams/03-failover-policy-configuration.md)
4. [Failover Scenario](diagrams/04-failover-scenario.md)
5. [Priority Label Inversion](diagrams/05-priority-label-inversion.md)

## Other Notes

Other changes to initial zip file:
- Export labels removed from policies
- Removed app label from priority labels