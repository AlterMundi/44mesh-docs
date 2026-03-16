---
title: Addressing
description: IPv4 allocation, address roles, CIDR layout, ZeroTier pool configuration, and route advertisement.
topics: [addressing, ipv4, cidr, amprnet, zerotier-pools, bgp-peering, ingress-node]
related:
  - docs/getting-started/prerequisites.md
  - docs/network/bgp.md
  - docs/network/zerotier-mesh.md
---

# Addressing

44Mesh uses real, globally routable public IPv4 addresses for all mesh nodes. There is no private address space or NAT within the mesh — every node is directly reachable from the Internet.

---

## Address Space Requirements

To operate a border router, you need a **publicly routable IPv4 block**. The minimum practical size is a `/24` (256 addresses), which is also the smallest block typically accepted by Internet routing tables.

### Sources for Address Space

| Source | How to Obtain | Notes |
|--------|--------------|-------|
| **AMPRNet (44/8)** | Apply at [ampr.org](https://www.ampr.org/) | Free; requires amateur radio license; `/24` allocations |
| **RIR allocation** | Apply to ARIN, LACNIC, RIPE, APNIC, or AFRINIC | Requires justification; ongoing fees |
| **Provider space** | Assigned by your ISP or hosting provider | Not portable; tied to the provider |
| **Leased space** | Rent from an IP broker | Portable; available without RIR membership |

---

## Address Roles

Within your allocated block, addresses are assigned to specific roles:

| Role | Quantity | Description |
|------|----------|-------------|
| **Border Router BGP IP** | 1 | The BGP router ID and peering address; assigned as a secondary IP on the host |
| **Network address** | 1 | First address in the block (unusable) |
| **Broadcast** | 1 | Last address in the block (unusable) |
| **Mesh node pool** | Remainder | Assigned dynamically by the ZeroTier controller to joining nodes |

Example for a `/24` block (`44.30.127.0/24`):

```
44.30.127.0     — network address
44.30.127.1     — border router BGP IP (secondary IP on host)
44.30.127.2–254 — mesh node pool (ZeroTier assignment pool)
44.30.127.255   — broadcast
```

---

## ZeroTier Address Assignment

The ZeroTier controller assigns addresses from a configured pool when nodes are authorized. The pool is set during network creation:

```json
{
  "ipAssignmentPools": [
    {
      "ipRangeStart": "44.30.127.2",
      "ipRangeEnd": "44.30.127.254"
    }
  ]
}
```

Each authorized node receives one IP from this pool. Assignments are persistent — the same node will receive the same IP on reconnect, as long as the controller's data volume is preserved.

---

## BGP Peering Subnet

The BGP session between the border router and the ISP uses a separate, small subnet — typically a `/30` or `/31` — assigned by the ISP for the peering link.

```
Example:
ISP assigns: 203.0.113.0/30
  203.0.113.1 — ISP's BGP peer IP  (set as ISP_IP in .env)
  203.0.113.2 — Your border router BGP IP  (set as BORDER_ROUTER_IP in .env)
```

This peering subnet is **not** from your mesh block. It is a link-local address for the BGP session only.

---

## Route Advertisement

The border router announces your entire mesh block (`MESH_ADDRESS_RANGE`) to the ISP via BGP. The ISP then propagates this advertisement to the global routing table.

The BIRD configuration restricts exports to only your mesh range:

```
protocol bgp isp {
  export where net = <MESH_ADDRESS_RANGE>;
  import all;
}
```

This prevents accidental announcement of other routes.

---

## Ingress Node Address

The `ingressNodeV4` parameter set in the ZeroTier network configuration is the **ZeroTier-assigned IP of the border router**. This is the address within the mesh that all nodes use as their default gateway for outbound traffic.

When a node is authorized, the AlterMundi ZeroTier fork reads `ingressNodeV4` and installs:

```bash
ip rule add from <node-public-ip>/32 lookup <table>
ip route add default via <ingressNodeV4> table <table>
```

The `ingressNodeV4` address is typically the first IP in the mesh pool, assigned to the border router when it joins its own network.

---

## Summary of Environment Variables

```bash
# Your announced IP block
MESH_ADDRESS_RANGE=44.30.127.0/24

# BGP peering IPs (from ISP-assigned peering subnet)
BORDER_ROUTER_IP=203.0.113.2      # Your end of the BGP link
ISP_IP=203.0.113.1                # ISP's BGP peer

# AS numbers
BORDER_ROUTER_AS=65000
ISP_AS=65001
```
