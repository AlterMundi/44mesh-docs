<!--
SPDX-FileCopyrightText: 2024 AlterMundi <docs@44mesh.net>

SPDX-License-Identifier: CC-BY-SA-4.0
-->

# Design Principles

44Mesh is built around a small set of principles that guide its architecture and implementation decisions.

---

## 1. Sovereignty First

Operators own their infrastructure. The network uses your IP space, your AS number, and your hardware. There is no dependency on a central 44Mesh service — the project provides the tooling, not a hosted platform.

This means:
- You control address allocation and routing policy
- You can peer with any ISP or IXP
- Your nodes are independent of any 44Mesh-operated infrastructure

---

## 2. Public Reachability by Default

Every mesh node gets a **globally routable IP address**. This is the core difference from typical overlay networks, which keep nodes on private address space behind NAT.

Public IPs enable:
- Inbound connections without port forwarding
- Hosting services directly on mesh nodes
- Participation in the global Internet without cloud intermediaries

---

## 3. Minimal Assumptions About the Underlying Network

Nodes can be on any network: home connections, mobile, cloud VMs, or dedicated hardware. The ZeroTier overlay handles NAT traversal, peer discovery, and encrypted transport regardless of the underlying topology.

The only requirement for a mesh node is outbound UDP connectivity.

---

## 4. Symmetric Routing

BGP requires that inbound and outbound traffic follow the same path at the AS boundary. 44Mesh enforces this through source-based policy routing installed automatically by the custom ZeroTier fork.

When a node is authorized, routing rules are installed so that any traffic sourced from the node's public IP exits through the border router — even if the underlying path to the border router is through the ZeroTier overlay.

This is done without requiring manual configuration on each node.

---

## 5. Automation Over Manual Configuration

The deployment is designed to minimize manual steps:
- Environment variable templates cover all configuration
- The border router entrypoint installs routing rules automatically
- The ZeroTier fork installs per-node source routing on authorization
- Docker Compose orchestrates everything

The goal is that adding a new node requires only joining the network and authorizing it — no additional configuration on the node.

---

## 6. Composability

Each component is independently deployable:
- The border router can run without the web UI
- The mock ISP is purely a development tool
- Mesh nodes have no dependency on the UI

This allows operators to adopt components selectively and integrate with existing infrastructure.

---

## 7. Open Standards

Where possible, 44Mesh uses standard protocols:
- **BGP** (RFC 4271) for Internet routing
- **ZeroTier** for overlay networking (open source, auditable)
- **BIRD2** for BGP implementation (widely deployed, well-documented)
- **Docker** for containerization

Proprietary lock-in is avoided. An operator who wants to migrate to a different overlay or BGP implementation can do so by replacing individual components.
