# Multi-Cluster Architecture with Virtual Destinations

This diagram illustrates the high-level architecture of the multi-cluster setup with Virtual Destinations and FailoverPolicies.

```mermaid
flowchart TB
    subgraph mgmt["Management Cluster"]
        vds["Virtual Destinations"]
        fps["Failover Policies"]
    end
    
    subgraph central["central-pp-rs Cluster"]
        central_services["Services (infra ns)"]
        central_clients["Clients (multiple namespaces)"]
    end
    
    subgraph test["test-pp-rs Cluster"]
        test_services["Services (infra ns)"]
        test_clients["Clients (multiple namespaces)"]
    end
    
    vds --> central_services
    vds --> test_services
    fps --> central_services
    fps --> test_services
    
    central_clients -->|"Request to VD\ngo-vuderani-algebra"| central_services
    test_clients -->|"Request to VD\ngo-vuderani-algebra"| test_services
    
    classDef cluster fill:#f9f9f9,stroke:#333,stroke-width:2px
    classDef component fill:#e1f5fe,stroke:#01579b,stroke-width:1px
    
    class mgmt,central,test cluster
    class vds,fps,central_services,central_clients,test_services,test_clients component
```

The diagram shows:
- A management cluster that hosts the Virtual Destinations and FailoverPolicies
- Two workload clusters: central-pp-rs and test-pp-rs
- Services and clients in each cluster
- How Virtual Destinations and FailoverPolicies are applied to services across clusters 