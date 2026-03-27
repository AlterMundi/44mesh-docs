# What is 44Mesh

44Mesh is a networking framework that allows research groups, independent operators or anyone who owns or manages an **Autonomous System (AS)** to run publicly reachable infrastructure on their own terms — without depending on cloud providers, commercial hosting, or NAT workarounds.

44Mesh connects a BGP border router (which announces your IP block to the Internet) with a ZeroTier overlay mesh (which connects your nodes securely, regardless of where they are physically located). The result is that every node in the mesh gets a **real, globally routable public IP address** — whether it's running on a server in a data center, a Raspberry Pi at a remote site, or a laptop behind a home router.

The framework is designed to be sovereign: you own the IP space, you run the infrastructure, and you control who joins the network. 44Mesh provides the tooling to make that operationally practical.

---

## The Problem

Most community networks, research infrastructure, and distributed systems face the same limitations:

- Nodes are behind NAT and unreachable from the Internet
- Public IPs are expensive or unavailable without an ISP relationship
- Traffic flows through centralized cloud services, creating single points of failure and loss of sovereignty
- Scientific and community sensor networks have no standard way to expose data to external consumers, including AI systems

These constraints limit experimentation, collaboration, and the ability to build truly distributed systems.

---

## The Solution

44Mesh solves this by combining:

- **BGP routing** — your network announces a real IP prefix to the global Internet
- **Public IPv4 addresses** — every mesh node receives a globally routable IP
- **ZeroTier overlay** — nodes connect through an encrypted mesh, regardless of underlying network topology

The result is a network where any node — on a home connection, a Raspberry Pi in a remote location, or a server in a data center — becomes a **publicly addressable host** on the Internet.

---

## How it Works (High Level)

A border router sits at the edge of your network. It runs a BGP session with an upstream ISP or IXP and announces your IP block to the Internet. It also runs a ZeroTier controller, which manages the overlay mesh.

Mesh nodes connect to the overlay. Each node receives a public IP from your allocated block. The border router acts as the **ingress node**: all inbound Internet traffic arrives at the border router and is forwarded through ZeroTier to the destination node.

For outbound traffic, a custom ZeroTier fork automatically installs source-based policy routing on each node, ensuring that packets sourced from public IPs exit through the border router — preserving symmetric routing.

---

## Intended Uses

44Mesh is designed for:

| Use Case | Description |
|----------|-------------|
| **Community networks** | Give neighborhoods, cooperatives, or local organizations real Internet presence |
| **Radio networks** | SDR receivers, amateur radio repeaters, spectrum sensors |
| **IoT / sensor networks** | Environmental monitoring, weather stations, distributed instruments |
| **Distributed observatories** | Coordinate scientific sensors across geographies |
| **Research infrastructure** | Low-cost, sovereign compute and networking for research groups |
| **AI agent infrastructure** | A globally routable sensor fabric that AI agents can discover and query |

---

## Relationship to 44Net

44Mesh can operate using address allocations from the **44/8 network** (administered by the [AMPRNet](https://www.ampr.org/) project for amateur radio operators), but it is not limited to that space.

It works equally well with:

- Provider-assigned prefixes
- RIR allocations (ARIN, LACNIC, RIPE, etc.)
- Leased address blocks
- Any other legitimately routable IPv4 space

The name reflects the project's roots in the amateur radio and community networking communities.

---

## What 44Mesh is Not

- **Not a VPN provider** — you bring your own IP space and AS number
- **Not a cloud platform** — nodes run on your own hardware
- **Not a replacement for ZeroTier** — it builds on top of ZeroTier with a custom fork that adds ingress routing
- **Not only for advanced operators** — the Docker Compose deployment is straightforward, but you do need IP space and an AS to participate as a border router
