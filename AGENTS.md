# 44Mesh Documentation — Agent Entry Point

44Mesh is an open network architecture that gives any device a publicly reachable IPv4 address using BGP routing and a ZeroTier overlay mesh. It is designed for community networks, IoT sensors, and AI agents that need direct addressability without NAT or centralized cloud dependencies. This repository contains the MkDocs documentation site for the project.

## Repository Structure

```
docs/
├── index.md                          # Landing page and feature overview
├── overview/
│   ├── what-is-44mesh.md             # Problem statement, use cases, relationship to AMPRNet
│   ├── architecture.md               # System diagram, traffic flows, container architecture
│   └── design-principles.md          # Eight core design principles
├── getting-started/
│   ├── prerequisites.md              # Requirements for border routers and mesh nodes
│   └── quickstart.md                 # Step-by-step first deployment
├── network/
│   ├── addressing.md                 # IPv4 allocation, address roles, CIDR layout
│   ├── bgp.md                        # BIRD2 config, eBGP sessions, route filters
│   ├── zerotier-mesh.md              # ZeroTier controller/client, AlterMundi fork, ingress routing
│   └── routing.md                    # Source-based policy routing, routing tables, traffic paths
├── deployment/
│   ├── border-router.md              # Full border router deployment with Docker Compose
│   ├── mesh-nodes.md                 # Mesh node deployment and authorization
│   ├── zerotier-ui.md                # Web UI for ZeroTier network management
│   └── mock-isp.md                   # Simulated BGP peer for local testing
├── operations/
│   ├── monitoring.md                 # Health checks, BGP status, connectivity tests
│   ├── security.md                   # Container capabilities, credentials, TLS, firewall
│   └── troubleshooting.md            # Diagnostic steps for BGP, ZeroTier, connectivity
├── participation/
│   ├── joining-44mesh.md             # How to join as node operator or border router
│   ├── operating-a-node.md           # Node maintenance, updates, service deployment
│   └── contributing.md               # Code, docs, and bug report contribution guide
├── ai-agents/
│   ├── overview.md                   # Vision for AI agents as network participants
│   ├── agent-discovery.md            # Finding nodes via IP scanning or controller API
│   ├── data-model.md                 # Observation, capability, alert data structures
│   ├── observation-api.md            # REST API spec (endpoints, parameters, responses)
│   └── example-agent.md              # Working Python anomaly detection agent
└── references/
    ├── glossary.md                   # 23 key terms defined
    └── external-resources.md         # Links to specs, tools, and related projects
mkdocs.yml                            # MkDocs Material configuration and nav tree
```

## Task Routing

### "Understand what 44Mesh is"
1. `docs/overview/what-is-44mesh.md` — problem, solution, use cases
2. `docs/overview/architecture.md` — system diagram and traffic flows
3. `docs/overview/design-principles.md` — core design decisions

### "Deploy a full network"
1. `docs/getting-started/prerequisites.md` — what you need (AS, IP block, BGP peer, Docker)
2. `docs/getting-started/quickstart.md` — end-to-end walkthrough
3. `docs/deployment/border-router.md` — border router setup
4. `docs/deployment/mesh-nodes.md` — mesh node setup

### "Deploy for local testing"
1. `docs/getting-started/prerequisites.md` — host requirements
2. `docs/deployment/mock-isp.md` — simulated BGP peer with RFC 5737 test prefixes
3. `docs/getting-started/quickstart.md` — step 5 covers mock ISP integration

### "Understand the networking"
1. `docs/network/addressing.md` — IPv4 allocation and address roles
2. `docs/network/bgp.md` — BIRD2, eBGP sessions, route advertisement
3. `docs/network/zerotier-mesh.md` — overlay mesh, NAT traversal, ingress routing
4. `docs/network/routing.md` — source-based policy routing for symmetric paths

### "Build an AI agent for 44Mesh"
1. `docs/ai-agents/overview.md` — why agents, interaction model, agent types
2. `docs/ai-agents/data-model.md` — observation/capability/alert schemas
3. `docs/ai-agents/observation-api.md` — REST endpoint specifications
4. `docs/ai-agents/agent-discovery.md` — finding nodes and capabilities
5. `docs/ai-agents/example-agent.md` — complete Python reference implementation

### "Operate and troubleshoot"
1. `docs/operations/monitoring.md` — health checks, metrics, connectivity tests
2. `docs/operations/security.md` — credentials, TLS, firewall, container capabilities
3. `docs/operations/troubleshooting.md` — diagnostic procedures for common failures

### "Join or contribute"
1. `docs/participation/joining-44mesh.md` — two paths: mesh node or border router
2. `docs/participation/operating-a-node.md` — maintenance and service deployment
3. `docs/participation/contributing.md` — code, docs, bug reports

## Key Concepts

- **Border Router**: The ingress node that holds the BGP session with an upstream peer, runs the ZeroTier controller, and announces the IP prefix to the Internet.
- **Mesh Node**: Any device running the ZeroTier client that receives a public IP from the border router's pool and becomes directly reachable.
- **ZeroTier Overlay**: Encrypted peer-to-peer mesh (using AlterMundi's fork) that connects all nodes regardless of NAT or firewall topology.
- **Source-Based Policy Routing**: Linux routing rules (table 123 on border router, per-node tables on mesh nodes) that ensure return traffic exits through the correct ingress node for symmetric paths.
- **ingressNodeV4**: A per-member ZeroTier metadata field that tells each mesh node which border router IP to use as its default gateway for source-routed traffic.
- **Observation**: The core data unit in the AI agent system — a timestamped sensor reading with location, value, unit, and confidence score.

Full glossary: `docs/references/glossary.md`

## Observation API Quick Reference

All mesh nodes running the observation API expose these REST endpoints on their public IP:

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/capabilities` | List sensor types and metadata for this node |
| `GET` | `/observations/latest` | Most recent reading per capability |
| `GET` | `/observations?type=temperature&since=2024-01-01T00:00:00Z` | Query historical observations with filters |
| `POST` | `/actions` | Trigger an action on the node |

Full spec: `docs/ai-agents/observation-api.md`
