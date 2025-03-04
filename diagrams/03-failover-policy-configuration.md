# FailoverPolicy Priority Labels and Selection Logic

This diagram illustrates how FailoverPolicy prioritizes endpoints and applies to Virtual Destinations.

```mermaid
flowchart TD
    subgraph fp["FailoverPolicy (fp-vuderani-for-test-pp-rs)"]
        apply["applyToDestinations:
        - kind: VIRTUAL_DESTINATION
          selector:
            name: vd-vuderani-for-test-pp-rs-infra
          port: 80"]
        config["config:
        priorityLabels:
          labels:
          - topology.istio.io/subzone
          - mesh.opentable.com/serves-test-pp-rs"]
    end
    
    fp --> logic["Endpoint Selection Logic"]
    
    subgraph selection["Selection Priority"]
        priority1["Priority 1: Same subzone
        (topology.istio.io/subzone)"]
        priority2["Priority 2: Labeled with
        mesh.opentable.com/serves-test-pp-rs"]
        result["Result: Local endpoints preferred
        over remote endpoints"]
        
        priority1 --> priority2 --> result
    end
    
    logic --> selection
    
    classDef fpbox fill:#e8eaf6,stroke:#3949ab,stroke-width:2px
    classDef logicbox fill:#f3e5f5,stroke:#7b1fa2,stroke-width:1px
    classDef selectionbox fill:#fce4ec,stroke:#c2185b,stroke-width:1px
    
    class fp fpbox
    class logic logicbox
    class selection,priority1,priority2,result selectionbox
```

The diagram shows:
- The configuration of a FailoverPolicy with its destination selectors and priority labels
- The priority order of labels used for endpoint selection
- How this priority order affects the selection of endpoints (local vs. remote) 