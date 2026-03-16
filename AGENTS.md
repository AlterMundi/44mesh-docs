# 44Mesh Documentation — Agent Entry Point

44Mesh gives any device a publicly reachable IPv4 address using BGP routing and a ZeroTier overlay mesh. No NAT, no cloud dependencies. This repo is the MkDocs documentation site.

## Repository Structure

```
docs/
├── index.md
├── overview/           # what 44Mesh is, architecture, design principles
├── getting-started/    # prerequisites, quickstart
├── network/            # addressing, BGP, ZeroTier, routing policy
├── deployment/         # border router, mesh nodes, ZeroTier UI, mock ISP
├── operations/         # monitoring, security, troubleshooting
├── participation/      # joining, operating a node, contributing
├── ai-agents/          # agent overview, discovery, data model, API, example
└── references/         # glossary, external resources
mkdocs.yml
```

Each doc file has YAML frontmatter with `description`, `topics`, and `related` paths for programmatic triage.

## Task Routing

### Understand 44Mesh
`docs/overview/what-is-44mesh.md` → `docs/overview/architecture.md` → `docs/overview/design-principles.md`

### Deploy a full network
`docs/getting-started/prerequisites.md` → `docs/getting-started/quickstart.md` → `docs/deployment/border-router.md` → `docs/deployment/mesh-nodes.md`

### Deploy for local testing
`docs/getting-started/prerequisites.md` → `docs/deployment/mock-isp.md` → `docs/getting-started/quickstart.md` (step 5)

### Understand the networking
`docs/network/addressing.md` → `docs/network/bgp.md` → `docs/network/zerotier-mesh.md` → `docs/network/routing.md`

### Build an AI agent
`docs/ai-agents/overview.md` → `docs/ai-agents/data-model.md` → `docs/ai-agents/observation-api.md` → `docs/ai-agents/agent-discovery.md` → `docs/ai-agents/example-agent.md`

### Operate and troubleshoot
`docs/operations/monitoring.md` → `docs/operations/security.md` → `docs/operations/troubleshooting.md`

### Join or contribute
`docs/participation/joining-44mesh.md` → `docs/participation/operating-a-node.md` → `docs/participation/contributing.md`

## Key Concepts

- **Border Router** — runs BGP + ZeroTier controller, announces your IP prefix to the Internet
- **Mesh Node** — any device with a ZeroTier client that gets a public IP from the border router
- **ZeroTier Overlay** — encrypted P2P mesh (AlterMundi fork) connecting nodes across NAT/firewalls
- **Source-Based Policy Routing** — Linux routing tables ensuring return traffic exits through the correct border router
- **ingressNodeV4** — ZeroTier per-member field pointing each mesh node to its border router gateway

Full glossary: `docs/references/glossary.md`

## Observation API

Mesh nodes expose REST endpoints on their public IP:

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/capabilities` | Sensor types and metadata |
| `GET` | `/observations/latest` | Most recent reading per capability |
| `GET` | `/observations?type=temperature&since=...` | Historical observations with filters |
| `POST` | `/actions` | Trigger a node action |

Full spec: `docs/ai-agents/observation-api.md`
