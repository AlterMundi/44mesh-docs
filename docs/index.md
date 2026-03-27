# 44Mesh Documentation

44Mesh is an open networking framework that makes your distributed devices, sensors, and services globally reachable — under your own autonomous system, without cloud dependencies, accessible to both humans and agents.

No cloud provider required. You bring your own IP space and AS number; 44Mesh handles the rest.

---

## What it does

Every node in 44Mesh gets a **public IP address** that is reachable from anywhere on the Internet. Traffic is routed through a border router using BGP, while the underlying connectivity between nodes uses an encrypted ZeroTier mesh overlay. This means nodes can be behind NAT, on mobile connections, or on any infrastructure — and still participate as publicly reachable hosts.

```
Internet
   │
   │ BGP (BIRD2)
   ▼
Border Router  ◄──── ZeroTier Controller
   │
   │ ZeroTier Overlay (encrypted)
   ▼
Mesh Nodes (public IPs, any location)
   │
   ▼
Services / Sensors / AI Agents
```

---

## Core Components

| Component | Description |
|-----------|-------------|
| **Border Router** | Connects the mesh to the Internet via BGP; acts as the ingress node for all mesh traffic |
| **ZeroTier Controller** | Manages mesh membership, address assignment, and ingress routing configuration |
| **Mesh Nodes** | Distributed hosts running services, sensors, or AI workloads |
| **ZeroTier UI** | Web interface for managing network members and configuration |
| **Mock ISP** | Simulated upstream provider for local development and testing |

---

## Key Features

- **Deploy in your AS** — operate with your own Autonomous System number and IP allocations
- **BGP-announced public IPs** — nodes are reachable from the global Internet
- **ZeroTier overlay** — handles NAT traversal and encrypted transport automatically
- **Ingress routing** — custom fork of ZeroTierOne installs per-node source routing automatically
- **AI-native** — sensor data model and observation APIs designed for machine consumers
- **Multi-arch containers** — all components ship as Docker images for amd64, arm64, and arm/v7
- **CI/CD included** — GitHub Actions pipelines for build, validation, and self-hosted deployment

---

## Use Cases

- **Scientific sensors** — connect distributed instruments (weather, radio, environmental) under a single public routable network
- **Radio systems** — SDR receivers, amateur radio nodes, spectrum monitoring
- **AI agent infrastructure** — provide a globally addressable sensor fabric that AI agents can query and coordinate across

---

## Requirements

* **IPv4 Address Range** – You must have an assigned block of IP addresses.
* **Autonomous System (AS)** – You must possess a private or registered **ASN** to establish peering. This allows the network to receive and process your **BGP** (Border Gateway Protocol) requests.
* **Direct Connectivity** – Your gateway must be **publicly reachable** from the internet to establish stable peering sessions with other BGP nodes.


---

## Navigate the Docs

| Section | Content |
|---------|---------|
| [Overview](overview/what-is-44mesh.md) | Concepts, architecture, design goals |
| [Getting Started](getting-started/prerequisites.md) | What you need and how to deploy |
| [Network](network/addressing.md) | Addressing, BGP, ZeroTier, routing details |
| [Deployment](deployment/border-router.md) | Step-by-step deployment for each component |
| [Operations](operations/monitoring.md) | Monitoring, security, troubleshooting |
| [Participation](participation/joining-44mesh.md) | How to join, operate a node, or contribute |
| [AI Agents](ai-agents/overview.md) | Sensor data model, APIs, example agents |
| [Reference](references/glossary.md) | Glossary and external links |
