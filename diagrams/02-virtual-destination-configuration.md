# Virtual Destination Configuration and DNS Resolution

This diagram illustrates how a Virtual Destination creates a DNS name and maps to services in different clusters.

```mermaid
flowchart TD
    subgraph vd["Virtual Destination (vd-vuderani-for-test-pp-rs-infra)"]
        hosts["hosts: go-vuderani-algebra.infra.svc.test-pp-rs"]
        ports["ports: 80 (HTTP)"]
        services["services: 
        - labels:
            app: vuderani-vd-server
            mesh.opentable.com/serves-test-pp-rs: true
          namespace: infra"]
    end
    
    vd --> dns["DNS Resolution: go-vuderani-algebra.infra.svc.test-pp-rs"]
    
    subgraph resolution["Service Resolution"]
        test_services["Services in test-pp-rs cluster
        with matching labels"]
        central_services["Services in central-pp-rs cluster
        with matching labels"]
    end
    
    dns --> test_services
    dns --> central_services
    
    classDef vdbox fill:#e8f5e9,stroke:#2e7d32,stroke-width:2px
    classDef dnsbox fill:#fff3e0,stroke:#e65100,stroke-width:1px
    classDef servicebox fill:#e3f2fd,stroke:#1565c0,stroke-width:1px
    
    class vd vdbox
    class dns dnsbox
    class resolution,test_services,central_services servicebox
```

The diagram shows:
- The configuration of a Virtual Destination with its hosts, ports, and service selectors
- How the Virtual Destination creates a DNS name that can be used by clients
- How the DNS name resolves to actual services in different clusters based on label matching 