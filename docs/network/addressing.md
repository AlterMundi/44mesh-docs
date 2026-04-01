<!--
SPDX-FileCopyrightText: 2024 AlterMundi <docs@44mesh.net>

SPDX-License-Identifier: CC-BY-SA-4.0
-->

# Addressing

44Mesh uses real, globally routable public IPv4 addresses for all mesh nodes. There is no private address space or NAT within the mesh — every node is directly reachable from the Internet.

---

## Address Space Requirements

To operate a border router, you need a **publicly routable IPv4 block**. The minimum practical size is a `/24` (256 addresses), which is also the smallest block typically accepted by Internet routing tables.

---

## ZeroTier Address Assignment

The ZeroTier controller assigns addresses from a configured pool when nodes are authorized. The pool is set during network creation:

```json
{
  "ipAssignmentPools": [
    {
      "ipRangeStart": "<your-block-first-usable-ip>",
      "ipRangeEnd": "<your-block-last-usable-ip>"
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
ISP assigns: <peering-subnet>/30
  <peering-subnet>.1 — ISP's BGP peer IP  (set as ISP_IP in .env)
  <peering-subnet>.2 — Your border router BGP IP  (set as BORDER_ROUTER_IP in .env)
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
MESH_ADDRESS_RANGE=<your-block>/24

# BGP peering IPs (from ISP-assigned peering subnet)
BORDER_ROUTER_IP=<your-bgp-ip>    # Your end of the BGP link
ISP_IP=<isp-bgp-ip>               # ISP's BGP peer

# AS numbers
BORDER_ROUTER_AS=<your-asn>
ISP_AS=<isp-asn>
```
