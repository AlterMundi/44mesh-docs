<!--
SPDX-FileCopyrightText: 2024 AlterMundi <docs@44mesh.net>

SPDX-License-Identifier: CC-BY-SA-4.0
-->

# AI Agents Overview

44Mesh is designed from the ground up to support **AI agents as first-class network participants**. The network is not just infrastructure for humans — it is a globally addressable sensor fabric that AI systems can discover, query, and act upon.

---

## The Vision

Traditional infrastructure is built for humans:
- Web interfaces for configuration
- Dashboards for monitoring
- Manual intervention for responses

44Mesh is built for a world where AI agents need to:
- **Discover** what sensors and capabilities exist across a distributed network
- **Query** real-time and historical observation data
- **Coordinate** with other agents and nodes across geographies
- **Act** on edge conditions — triggering alerts, adjusting parameters, requesting human review

Every mesh node is a potential **sensor endpoint**, **compute host**, or **agent runtime**. The network gives each of these a stable, globally routable public IP address and a machine-readable API.

---

## Why This Matters

Consider a distributed environmental monitoring network:
- 50 nodes across a continent, each hosting temperature, humidity, and radiation sensors
- An AI agent needs to detect anomalies, correlate readings, and predict weather events
- Without 44Mesh: each node is behind NAT, uses a different API format, and requires manual discovery
- With 44Mesh: each node has a public IP, advertises its capabilities in a standard format, and exposes a consistent observation API

Or a radio telescope network:
- 20 nodes each running an SDR receiver pointed at different parts of the sky
- An AI agent needs to coordinate observations, synchronize timing, and aggregate data
- 44Mesh provides the transport, addressing, and discovery layer

---

## Agent Interaction Model

AI agents interact with mesh nodes through:

```
Agent
  │
  ▼ HTTP/REST (public IP, no NAT)
Mesh Node
  │
  ├── GET /capabilities  → what this node can do
  ├── GET /observations  → sensor readings (current or historical)
  └── POST /actions      → trigger node behavior (optional)
```

Nodes expose **machine-readable metadata** describing their capabilities. Agents can discover this metadata without prior knowledge of the node's specific configuration.

---

## Network Properties That Enable AI Agents

| Property | Benefit for AI Agents |
|----------|----------------------|
| **Public IPs on every node** | Agents can reach any node directly, no NAT traversal required |
| **Stable addressing** | Node IPs persist (controlled by the ZeroTier controller); agents can maintain long-term references |
| **Standard REST APIs** | No need for custom protocol adapters per node |
| **Structured data model** | Observations have consistent schema; agents can parse without custom code per sensor |
| **Discovery endpoints** | Agents can enumerate capabilities without prior configuration |

---

### **Understand the data model** — See [Data Model](data-model.md)

---

## Future Directions

The AI agent infrastructure in 44Mesh is evolving. Planned additions include:

- **Agent registry** — A network-wide directory of running agents and their capabilities
- **Event streams** — WebSocket or SSE endpoints for real-time observation feeds
- **Agent-to-agent coordination** — Standard protocols for agents to discover and communicate with each other
- **Capability negotiation** — Agents advertise their needs; nodes advertise their capabilities; the network matches them

Contributions in this area are particularly welcome — see [Contributing](../participation/contributing.md).
