---
title: Joining 44Mesh
description: Two paths: join as a mesh node (easy) or run a border router (requires AS + IP block).
topics: [joining, participation, mesh-node, border-router, autonomous-system, amprnet, bgp-peering]
related:
  - docs/getting-started/prerequisites.md
  - docs/getting-started/quickstart.md
  - docs/participation/contributing.md
---

# Joining 44Mesh

There are two ways to participate in a 44Mesh network: as a **mesh node** (joining an existing network) or as a **border router operator** (running your own autonomous system).

---

## Option A — Join as a Mesh Node

This is the simplest path. You connect to an existing 44Mesh network operated by someone else. You get a public IP and full Internet reachability without needing your own AS or IP block.

**What you need**:
- A Linux host (any architecture, including Raspberry Pi)
- Docker and Docker Compose
- The 16-character network ID from the network operator
- Authorization from the network operator

**Steps**:

1. Install Docker and Docker Compose
2. Clone the repository:
   ```bash
   git clone https://github.com/altermundi/44mesh.git
   cd 44mesh/deploy/zerotier
   ```
3. Configure:
   ```bash
   cp .env.example .env
   # Set ZT_NETWORK_ID=<network-id>
   ```
4. Deploy:
   ```bash
   docker compose up -d
   ```
5. Get your node ID:
   ```bash
   docker exec zerotier zerotier-cli info
   ```
6. Send your node ID to the network operator for authorization

After authorization, your node receives a public IP and source routing is installed automatically.

---

## Option B — Run Your Own Border Router

This path gives you full sovereignty: your own AS, your own IP block, and full control over who joins your network.

**What you need**:
- An AS number (from an RIR or borrowed from a sponsor)
- A publicly routable IP block (`/24` minimum)
- A BGP peer (ISP or IXP willing to peer with your AS)
- A Linux server accessible from the Internet
- Docker and Docker Compose

See:
- [Prerequisites](../getting-started/prerequisites.md) for full requirements
- [Quickstart](../getting-started/quickstart.md) for deployment steps
- [Border Router Deployment](../deployment/border-router.md) for detailed configuration

---

## Getting IP Space

### AMPRNet (44/8)

If you are a licensed amateur radio operator, you can apply for a free `/24` allocation from the AMPRNet project:

- Website: [ampr.org](https://www.ampr.org/)
- Requires a valid amateur radio license callsign
- Allocations are from the `44.0.0.0/8` range

### RIR Allocations

Apply directly to your regional Internet registry:

| Region | RIR |
|--------|-----|
| North America | [ARIN](https://www.arin.net/) |
| Latin America | [LACNIC](https://www.lacnic.net/) |
| Europe/Middle East | [RIPE NCC](https://www.ripe.net/) |
| Asia-Pacific | [APNIC](https://www.apnic.com/) |
| Africa | [AFRINIC](https://www.afrinic.net/) |

### Leased Space

IP brokerage services offer IPv4 addresses for lease without RIR membership requirements. Search for "IPv4 lease" or "IPv4 rental."

---

## Finding a BGP Peer

Options for establishing a BGP session:

- **Your current ISP**: Ask if they offer BGP transit or peering
- **IXP (Internet Exchange Point)**: Connect to a local IXP
- **BGP community networks**: Some community networks offer free or low-cost BGP transit for research/community use
- **VPS providers with BGP**: Some cloud providers (Vultr, Hetzner, etc.) offer BGP sessions for bring-your-own-IP customers

---

## Community and Support

- **GitHub Issues**: Report bugs or ask questions at the [44mesh repository](https://github.com/altermundi/44mesh/issues)
- **Contributing**: See [Contributing](contributing.md) if you want to help improve the project
