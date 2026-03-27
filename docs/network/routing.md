<!--
SPDX-FileCopyrightText: 2024 AlterMundi <docs@44mesh.net>

SPDX-License-Identifier: CC-BY-SA-4.0
-->

# Routing & Policy

This page explains how traffic flows through the 44Mesh network in detail, focusing on the source-based policy routing that makes symmetric routing work.

---

## The Routing Problem

Standard BGP routing creates an asymmetry challenge in 44Mesh:

- **Inbound**: Internet → ISP → Border Router → ZeroTier → Mesh Node ✓
- **Outbound**: Mesh Node → ??? → ISP → Internet

A mesh node's default route points to the border router over ZeroTier. But the border router receives traffic from mesh nodes via ZeroTier, and needs to forward it to the ISP. Without special configuration, the kernel might use a different path for the return traffic.

More critically: for BGP to work correctly, traffic from your announced IP block must exit through the border router that announced it. If a mesh node somehow reached the Internet through a different path, the return traffic would arrive at the border router but have no way to reach the originating node.

---

## Solution: Source-Based Policy Routing

Linux policy routing allows different routing tables to be used based on packet attributes. 44Mesh uses routing tables keyed on **source IP address**.

### On the Border Router

The entrypoint script runs at container start:

```bash
ip rule add from <MESH_ADDRESS_RANGE> lookup 123
ip route replace default via <ISP_IP> table 123
```

**Effect**: Any packet whose source address is within the mesh range uses routing table 123. Table 123 has a single default route pointing to the ISP.

This handles traffic that arrives at the border router from mesh nodes (sourced from their public IPs) and needs to exit to the Internet.

### On Each Mesh Node

The AlterMundi ZeroTier fork runs at authorization time:

```bash
ip rule add from <node-public-ip>/32 lookup <table>
ip route add default via <ingressNodeV4> table <table>
```

**Effect**: Packets sourced from the node's public IP use a dedicated routing table. That table has a default route via `ingressNodeV4` (the border router's ZeroTier IP). All other traffic follows the system default route.

---

## Full Traffic Path

### Outbound (Node → Internet)

```
Mesh Node
  Packet src=44.x.y.z, dst=8.8.8.8
  │
  ▼ ip rule: src=44.x.y.z → lookup table N
  ▼ table N: default via ingressNodeV4
  │
  ▼ ZeroTier overlay (encrypted)
  │
Border Router
  Packet src=44.x.y.z arrives from ZeroTier
  │
  ▼ ip rule: src in 44.x.y.0/24 → lookup table 123
  ▼ table 123: default via ISP_IP
  │
  ▼ ISP link
  │
Internet
  Sees src=44.x.y.z (your announced IP)
```

### Inbound (Internet → Node)

```
Internet
  Packet dst=44.x.y.z
  │
  ▼ BGP: 44.x.y.0/24 is announced by your AS
  │
ISP
  │
  ▼ Forwarded to border router (BGP next-hop)
  │
Border Router
  Packet dst=44.x.y.z
  │
  ▼ Kernel routing table: 44.x.y.z is a ZeroTier-assigned address
  ▼ ZeroTier overlay
  │
Mesh Node 44.x.y.z
  Receives packet
```

---

## Routing Tables

| Table | Location | Content | Purpose |
|-------|----------|---------|---------|
| `main` (253) | All hosts | Normal routes | General traffic |
| `123` | Border Router | `default via <ISP_IP>` | Outbound path for mesh-sourced traffic |
| `N` (auto) | Mesh Nodes | `default via <ingressNodeV4>` | Outbound path for node's public IP traffic |

The table number `N` on mesh nodes is managed by the ZeroTier fork and is typically derived from the network ID.

---

## Rule Priority

Linux evaluates routing rules in priority order (lower number = higher priority).

On the border router:

```bash
ip rule show
# Example output:
0:      from all lookup local
32765:  from 44.x.y.0/24 lookup 123    ← added by entrypoint
32766:  from all lookup main
32767:  from all lookup default
```

On a mesh node:

```bash
ip rule show
# Example output:
0:      from all lookup local
100:    from 44.x.y.42/32 lookup 500   ← added by ZeroTier fork
32766:  from all lookup main
32767:  from all lookup default
```

---

## Verifying Routing

On the border router:

```bash
# Check source routing rules
ip rule show | grep "from 44"

# Check routing table 123
ip route show table 123

# Trace a packet from a mesh IP
ip route get 8.8.8.8 from 44.x.y.1
```

On a mesh node:

```bash
# Check source routing rules
ip rule show

# Check the per-node routing table
ip route show table <table-number>

# Test outbound routing from the public IP
curl --interface 44.x.y.z https://ifconfig.me
```

---

## Persistence

The source routing rules installed by the entrypoint script and the ZeroTier fork are **not persistent across reboots** of the host or container restarts.

- **Border router rules**: Reinstalled by the entrypoint script each time the container starts
- **Mesh node rules**: Reinstalled by the ZeroTier fork each time ZeroTier connects and receives its network config

No manual persistence configuration is needed.
