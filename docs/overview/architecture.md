# Architecture

44Mesh connects distributed nodes into a single, publicly routable network. The architecture has four layers: the Internet edge (BGP), the overlay mesh (ZeroTier), the node layer, and the application layer (services, sensors, AI agents).

---

## System Diagram

```
┌──────────────────────────────────────────────────────────┐
│                        Internet                          │
└────────────────────────────┬─────────────────────────────┘
                             │  BGP (port 179)
                             │  announces your IP block
                             ▼
┌──────────────────────────────────────────────────────────┐
│                     Border Router                        │
│                                                          │
│  ┌─────────────────────┐  ┌────────────────────────────┐ │
│  │   BIRD2 BGP Daemon  │  │  ZeroTier Controller       │ │
│  │                     │  │  (AlterMundi fork)         │ │
│  │  - Announces prefix │  │  - Authorizes nodes        │ │
│  │  - Peers with ISP   │  │  - Assigns public IPs      │ │
│  │  - Source routing   │  │  - Distributes ingressNodeV4│ │
│  └─────────────────────┘  └────────────────────────────┘ │
└──────────────────────────────┬───────────────────────────┘
                               │  ZeroTier Overlay
                               │  (encrypted, NAT-traversing UDP)
              ┌────────────────┼────────────────┐
              ▼                ▼                ▼
     ┌─────────────┐  ┌─────────────┐  ┌─────────────┐
     │  Mesh Node  │  │  Mesh Node  │  │  Mesh Node  │
     │             │  │             │  │             │
     │  Public IP  │  │  Public IP  │  │  Public IP  │
     │  from block │  │  from block │  │  from block │
     │             │  │             │  │             │
     │  Services   │  │  Sensors    │  │  AI Agents  │
     └─────────────┘  └─────────────┘  └─────────────┘
```

---

## Components

### Border Router

The border router is the single entry/exit point between the mesh and the Internet.

It runs two processes in the same container environment:

**BIRD2 BGP Daemon**
- Establishes a BGP session with the upstream ISP or IXP
- Announces your IP block to the Internet
- Accepts routes from the ISP (default route or specific prefixes)
- Installs source-based policy routing: traffic from your IP block exits via the ISP

**ZeroTier Controller** (AlterMundi fork)
- Manages the ZeroTier network
- Authorizes nodes and assigns addresses from your IP pool
- Distributes the `ingressNodeV4` parameter to all joining nodes
- Configured with `allowDefault=0` to prevent routing loops

Both processes share host networking, which is required for direct BGP peering.

---

### ZeroTier Overlay

The overlay network is the transport layer connecting all nodes. It handles:

- **Node discovery** — nodes find each other through the ZeroTier infrastructure
- **NAT traversal** — nodes behind NAT can still communicate
- **Encrypted transport** — all traffic is end-to-end encrypted
- **Address distribution** — the controller assigns IPs from your public block

The critical addition in the 44Mesh fork is the `ingressNodeV4` field. When a node joins the network, the controller distributes this address, and the fork automatically installs per-node source routing so that traffic from the node's public IP is routed back through the border router.

---

### Mesh Nodes

Mesh nodes are the distributed participants. Each node:

- Runs the ZeroTier client (AlterMundi fork, `feature/ingress-node` branch)
- Joins the network and receives a public IP from your allocated block
- Has source routing installed automatically upon authorization
- Can run any services, sensors, or compute workloads

Nodes can be on any infrastructure: cloud VMs, Raspberry Pis, laptops on home connections, servers in data centers. The overlay handles the rest.

---

### ZeroTier UI (Optional)

A web interface for managing the network. It reads the controller auth token from the shared `zerotier_data` volume and provides:

- Member authorization and management
- Network configuration (IP pools, routes)
- `ingressNodeV4` configuration (AlterMundi fork adds a dedicated button)

---

### Mock ISP (Development)

For testing without a real BGP peer, the mock ISP component simulates an upstream provider. It announces RFC 5737 test prefixes and establishes a BGP session with your border router, enabling full end-to-end testing locally.

---

## Traffic Flow

### Inbound (Internet → Mesh Node)

```
Internet
  │
  ▼  BGP routes your prefix to the border router
Border Router
  │
  ▼  ZeroTier overlay to the destination node
Mesh Node (destination)
```

The border router is the BGP next-hop for all addresses in your block. All inbound traffic arrives at the border router and is forwarded over ZeroTier.

### Outbound (Mesh Node → Internet)

```
Mesh Node
  │
  ▼  Source routing: src IP in your block → use table 123
Border Router (via ZeroTier)
  │
  ▼  Source routing: table 123 → default via ISP
ISP
  │
  ▼
Internet
```

Each mesh node has source-based policy routing installed by the ZeroTier fork. When a packet is sourced from its assigned public IP, the kernel uses routing table 123, which sends the packet to the border router. The border router then forwards it to the ISP.

This ensures **symmetric routing**: both inbound and outbound paths flow through the border router, which is required for BGP to work correctly.

---

## Container Architecture

All components run as Docker containers with host networking:

```
docker network: host (required for BGP and ZeroTier)

bird-border:
  - image: buzondefede/44mesh-bird-border
  - capabilities: NET_ADMIN, SYS_ADMIN
  - device: /dev/net/tun
  - volume: zerotier_data (shared with zerotier-ui)

zerotier (mesh node):
  - image: buzondefede/44mesh-zerotier
  - capabilities: NET_ADMIN, SYS_ADMIN
  - device: /dev/net/tun

zerotier-ui (optional):
  - image: buzondefede/44mesh-zerotier-ui
  - volume: zerotier_data (read-only access to controller token)

rpi-isp (mock ISP, dev only):
  - image: buzondefede/44mesh-rpi-isp
  - capabilities: NET_ADMIN
```

---

## CI/CD

GitHub Actions pipelines handle:

- **Multi-arch builds** — amd64, arm64, arm/v7 images pushed to Docker Hub
- **PR validation** — `docker compose config` validation for all components
- **Self-hosted deployment** — automatic deployment on merge to main via a self-hosted runner
