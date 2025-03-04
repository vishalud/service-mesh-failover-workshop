# Effect of Inverting Priority Labels

This diagram illustrates how inverting the priority labels in a FailoverPolicy affects traffic routing.

```mermaid
flowchart TD
    subgraph original["Original Priority Labels"]
        original_config["1. topology.istio.io/subzone
        2. mesh.opentable.com/serves-test-pp-rs"]
        original_result["Result: Prefer endpoints in same subzone"]
    end
    
    subgraph inverted["Inverted Priority Labels"]
        inverted_config["1. mesh.opentable.com/serves-test-pp-rs
        2. topology.istio.io/subzone"]
        inverted_result["Result: Prefer endpoints with specific label,
        regardless of subzone"]
    end
    
    original --> inverted
    
    subgraph traffic["Traffic Flow Comparison"]
        subgraph original_flow["Original"]
            client1["Client"] -->|"Request"| local1["Local Endpoints"]
            local1 -.->|"Failover only"| remote1["Remote Endpoints"]
        end
        
        subgraph inverted_flow["Inverted"]
            client2["Client"] 
            local2["Local Endpoints (if labeled)"]
            remote2["Remote Endpoints (if labeled)"]
            
            client2 -->|"Request"| local2
            client2 -->|"Request"| remote2
        end
    end
    
    inverted --> traffic
    
    classDef configbox fill:#e8eaf6,stroke:#3949ab,stroke-width:2px
    classDef resultbox fill:#f3e5f5,stroke:#7b1fa2,stroke-width:1px
    classDef flowbox fill:#e0f7fa,stroke:#006064,stroke-width:2px
    classDef clientbox fill:#e8f5e9,stroke:#2e7d32,stroke-width:1px
    classDef endpointbox fill:#fff8e1,stroke:#ff6f00,stroke-width:1px
    
    class original,inverted configbox
    class original_result,inverted_result resultbox
    class original_flow,inverted_flow flowbox
    class client1,client2 clientbox
    class local1,remote1,local2,remote2 endpointbox
```

The diagram shows:
- The original priority label order that prefers endpoints in the same subzone
- The inverted priority label order that prefers endpoints with a specific label
- How traffic flows differently under each configuration:
  - Original: Local endpoints preferred, remote used only for failover
  - Inverted: Endpoints with the specific label preferred, regardless of location 