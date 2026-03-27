<!--
SPDX-FileCopyrightText: 2024 AlterMundi <docs@44mesh.net>

SPDX-License-Identifier: CC-BY-SA-4.0
-->

# Quickstart

This guide walks through deploying a minimal 44Mesh network: one border router and one mesh node. It assumes you have already met the [prerequisites](prerequisites.md).

For local testing without a real BGP peer, skip to [Step 5 — Mock ISP (Testing)](#step-5--mock-isp-testing).

---

## Step 1 — Configure the Border Router

```bash
cd deploy/bird-border
cp .env.example .env
```

Edit `.env` with your values:

```bash
BORDER_ROUTER_AS=65000           # Your AS number
BORDER_ROUTER_IP=192.0.2.1       # Your BGP peering IP (must exist on the host)
ISP_IP=192.0.2.254               # Your ISP's BGP peer IP
ISP_AS=65001                     # Your ISP's AS number
MESH_ADDRESS_RANGE=44.x.y.0/24  # Your allocated IP block
BGP_HOLD_TIME=90
BGP_KEEPALIVE_TIME=30
```

!!! warning "Secondary IP required"
    `BORDER_ROUTER_IP` must be assigned to a host interface before starting the container. See [Prerequisites → Add Secondary IP](prerequisites.md#add-secondary-ip-for-bgp-peering).

Start the border router:

```bash
docker compose up -d
```

This starts two containers:
- `zerotier` — ZeroTier controller (AlterMundi fork)
- `bird-border` — BIRD2 BGP daemon

Verify both are healthy:

```bash
docker compose ps
```

---

## Step 2 — Create the ZeroTier Network

Use the controller API to create the network. The controller runs on the same host as the border router.

First, get the controller auth token:

```bash
docker exec zerotier cat /var/lib/zerotier-one/authtoken.secret
```

Then create the network:

```bash
curl -s -X POST http://localhost:9993/controller/network/$(docker exec zerotier zerotier-cli info | awk '{print $3}')______ \
  -H "X-ZT1-Auth: <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "44mesh",
    "private": true,
    "v4AssignMode": {"zt": true},
    "ipAssignmentPools": [{"ipRangeStart": "44.x.y.1", "ipRangeEnd": "44.x.y.254"}],
    "routes": [{"target": "0.0.0.0/0", "via": null}],
    "ingressNodeV4": "<border-router-mesh-ip>"
  }'
```

Note the **network ID** returned — you will need it for nodes.

!!! tip "Use the Web UI"
    The [ZeroTier UI](../deployment/zerotier-ui.md) provides a graphical interface for this step. The AlterMundi fork adds a dedicated button to set `ingressNodeV4`.

---

## Step 3 — Deploy the Web UI (Optional)

```bash
cd deploy/zerotier-ui
cp .env.example .env
```

Edit `.env`:

```bash
ZTNCUI_PASSWD=your-admin-password
ZTNCUI_HTTP_PORT=3180
```

```bash
docker compose up -d
```

Access the UI at `http://<host-ip>:3180`.

---

## Step 4 — Deploy a Mesh Node

On the node host:

```bash
cd deploy/zerotier
cp .env.example .env
```

Edit `.env`:

```bash
ZT_NETWORK_ID=<network-id-from-step-2>
```

```bash
docker compose up -d
```

The node will start, join the network, and wait for authorization.

Authorize the node via the UI or API:

```bash
# Get the node ID
docker exec zerotier zerotier-cli info

# Authorize via API (run on the border router host)
curl -s -X POST http://localhost:9993/controller/network/<network-id>/member/<node-id> \
  -H "X-ZT1-Auth: <token>" \
  -H "Content-Type: application/json" \
  -d '{"authorized": true}'
```

Once authorized, the node receives a public IP and the fork installs source routing automatically.

Verify the node has a public IP:

```bash
docker exec zerotier zerotier-cli listnetworks
```

---

## Step 5 — Mock ISP (Testing)

If you don't have a real BGP peer, use the mock ISP for end-to-end testing.

On the same host as the border router (or a second host on the same network):

```bash
cd deploy/rpi-isp
cp .env.example .env
```

Edit `.env`:

```bash
ISP_AS=65001
ISP_IP=192.0.2.254               # Must match ISP_IP in bird-border .env
BORDER_ROUTER_IP=192.0.2.1
BORDER_ROUTER_AS=65000
MESH_ADDRESS_RANGE=44.x.y.0/24
TEST_PREFIX_1=192.0.2.0/24
TEST_PREFIX_2=198.51.100.0/24
TEST_PREFIX_3=203.0.113.0/24
```

```bash
docker compose up -d
```

---

## Step 6 — Verify

### BGP Session

On the border router host:

```bash
docker exec bird-border birdc show protocols
docker exec bird-border birdc show route
```

You should see the BGP session established and your mesh prefix in the routing table.

### ZeroTier Status

On the mesh node:

```bash
docker exec zerotier zerotier-cli listnetworks
```

Expected output shows `OK` status and an IP assignment.

### Public Reachability

From the mesh node, test outbound routing:

```bash
# Replace with your node's assigned public IP
docker exec zerotier curl --interface 44.x.y.z https://ifconfig.me
```

This should return the border router's public IP (traffic exits via the border router).

From outside the network, test inbound connectivity to a service running on a mesh node:

```bash
# Run a simple server on the node
docker exec zerotier python3 -m http.server 8080

# From any Internet-connected host
curl http://44.x.y.z:8080
```

---

## Next Steps

- [Border Router — Full Configuration](../deployment/border-router.md)
- [Mesh Nodes — Options and Persistence](../deployment/mesh-nodes.md)
- [Addressing — IP allocation and assignment](../network/addressing.md)
- [BGP — Configuration and verification](../network/bgp.md)
