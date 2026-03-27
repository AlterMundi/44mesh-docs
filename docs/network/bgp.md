<!--
SPDX-FileCopyrightText: 2024 AlterMundi <docs@44mesh.net>

SPDX-License-Identifier: CC-BY-SA-4.0
-->

# BGP Routing

44Mesh uses BGP (Border Gateway Protocol) to announce your IP block to the global Internet. The BIRD2 daemon runs on the border router and manages the BGP session with your upstream ISP or IXP.

---

## Overview

BGP is the routing protocol of the Internet. When your border router establishes a BGP session with an ISP and announces your IP prefix, Internet routers globally learn that traffic for your addresses should be sent to your border router.

In 44Mesh:
- The border router runs **BIRD2** as the BGP daemon
- A single eBGP session is established with the upstream peer
- Your mesh range (`MESH_ADDRESS_RANGE`) is the only prefix exported
- All routes received from the peer are accepted (including a default route)

---

## BIRD Configuration

The border router generates its BIRD configuration from a template at startup using `envsubst`. The resulting config looks like:

```
router id <BORDER_ROUTER_IP>;

protocol device {}

protocol direct {
  ipv4;
}

protocol kernel {
  ipv4 {
    export all;
  };
}

protocol static {
  ipv4;
  route <MESH_ADDRESS_RANGE> blackhole;
}

protocol bgp isp {
  local <BORDER_ROUTER_IP> as <BORDER_ROUTER_AS>;
  neighbor <ISP_IP> as <ISP_AS>;
  hold time <BGP_HOLD_TIME>;
  keepalive time <BGP_KEEPALIVE_TIME>;

  ipv4 {
    import all;
    export where source = RTS_STATIC;
  };
}
```

### Key Details

**Static blackhole route**: BIRD requires the announced prefix to exist in its routing table. A blackhole route for `MESH_ADDRESS_RANGE` satisfies this without affecting actual packet forwarding (the kernel routing table handles forwarding independently).

**Export filter**: Only routes from the static protocol (i.e., your mesh range) are exported. This prevents accidentally announcing other routes.

**Import all**: All routes from the peer are accepted. This typically includes a default route (`0.0.0.0/0`) from the ISP, which enables outbound Internet connectivity from the border router.

---

## Environment Variables

| Variable | Description | Example |
|----------|-------------|---------|
| `BORDER_ROUTER_AS` | Your AS number | `65000` |
| `BORDER_ROUTER_IP` | Your BGP router ID and local address | `203.0.113.2` |
| `ISP_IP` | Your ISP's BGP peer address | `203.0.113.1` |
| `ISP_AS` | Your ISP's AS number | `65001` |
| `MESH_ADDRESS_RANGE` | The IP block to announce | `44.30.127.0/24` |
| `BGP_HOLD_TIME` | BGP hold timer (seconds) | `90` |
| `BGP_KEEPALIVE_TIME` | BGP keepalive timer (seconds) | `30` |

---

## Verification

Check BGP session status:

```bash
docker exec bird-border birdc show protocols
```

Expected output when the session is established:

```
Name       Proto      Table      State  Since         Info
device1    Device     ---        up     2024-01-01
direct1    Direct     ---        up     2024-01-01
kernel1    Kernel     master4    up     2024-01-01
static1    Static     master4    up     2024-01-01
isp        BGP        ---        up     2024-01-01    Established
```

View announced routes:

```bash
docker exec bird-border birdc show route
```

View what is being exported to the ISP:

```bash
docker exec bird-border birdc "show route export isp"
```

View BIRD daemon status:

```bash
docker exec bird-border birdc show status
```

---

## Source-Based Policy Routing

In addition to BGP, the border router configures Linux policy routing to ensure symmetric path handling.

At startup, the entrypoint script runs:

```bash
ip rule add from <MESH_ADDRESS_RANGE> lookup 123
ip route replace default via <ISP_IP> table 123
```

This means: any packet whose **source address** is within the mesh range will use routing table 123, which has a default route pointing to the ISP.

This is required because the border router receives traffic from mesh nodes over ZeroTier. Without this rule, return traffic would follow the kernel's default routing table, which may not have a route to the Internet via the ISP.

See [Routing & Policy](routing.md) for more detail.

---

## BGP Session Requirements

For the BGP session to establish:

1. **Port 179/TCP** must be open between the border router IP and the ISP peer IP
2. The **AS numbers** must match what the ISP has configured on their end
3. The **router ID** (`BORDER_ROUTER_IP`) must be reachable from the ISP peer
4. The secondary IP (`BORDER_ROUTER_IP`) must be assigned to a host interface before the container starts

---

## Common Issues

| Symptom | Likely Cause |
|---------|-------------|
| Session stays in `Active` state | Port 179 blocked, wrong peer IP, or ISP hasn't configured their end |
| Session established but no routes | Export filter mismatch; check that your static route is in BIRD's table |
| BGP flapping | Hold timer too short; increase `BGP_HOLD_TIME` |
| No default route from ISP | ISP may require you to request a default route; or add `next hop self` |

See [Troubleshooting](../operations/troubleshooting.md) for more.

---

## Mock ISP for Testing

The `deploy/rpi-isp/` component provides a simulated BGP peer for development and testing. It announces RFC 5737 test prefixes and accepts your mesh range, enabling full BGP testing without a real ISP connection.

See [Mock ISP Deployment](../deployment/mock-isp.md).
