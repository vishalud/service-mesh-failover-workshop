# Failover Scenario: Primary Cluster Failure

This diagram illustrates what happens during a failover scenario when the primary cluster is scaled to zero.

```mermaid
flowchart TD
    subgraph normal["Normal Operation"]
        client_central1["Client in central-pp-rs"] -->|Request| service_central1["Service in central-pp-rs"]
        client_test1["Client in test-pp-rs"] -->|Request| service_test1["Service in test-pp-rs"]
    end
    
    subgraph failover["After Scaling central-pp-rs to 0"]
        client_central2["Client in central-pp-rs"] 
        service_central2["Service in central-pp-rs
        (scaled to 0)"]
        service_test2["Service in test-pp-rs
        (with label serves-central-pp-rs)"]
        
        client_central2 -->|"Failover Request"| service_test2
        client_test2["Client in test-pp-rs"] -->|"Local Request"| service_test2
    end
    
    normal --> failover
    
    classDef normalbox fill:#e0f7fa,stroke:#006064,stroke-width:2px
    classDef failoverbox fill:#fff8e1,stroke:#ff6f00,stroke-width:2px
    classDef clientbox fill:#e8f5e9,stroke:#2e7d32,stroke-width:1px
    classDef servicebox fill:#f3e5f5,stroke:#7b1fa2,stroke-width:1px
    classDef downservice fill:#ffebee,stroke:#b71c1c,stroke-width:1px,stroke-dasharray: 5 5
    
    class normal normalbox
    class failover failoverbox
    class client_central1,client_test1,client_central2,client_test2 clientbox
    class service_central1,service_test1,service_test2 servicebox
    class service_central2 downservice
```

The diagram shows:
- Normal operation where clients in each cluster communicate with services in their own cluster
- What happens when the service in the central-pp-rs cluster is scaled to zero
- How the FailoverPolicy redirects traffic from the central-pp-rs cluster to the test-pp-rs cluster
- How the label `serves-central-pp-rs` enables the test-pp-rs service to handle requests for both clusters 