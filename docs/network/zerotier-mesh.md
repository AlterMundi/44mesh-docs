# ZeroTier Mesh

ZeroTier provides the overlay network that connects all mesh nodes. 44Mesh uses a [custom fork of ZeroTierOne](https://github.com/AlterMundi/ZeroTierOne) maintained by AlterMundi, with additions specifically designed for ingress routing.

---

## What ZeroTier Does in 44Mesh

| Function | Description |
|----------|-------------|
| **Node discovery** | Nodes find each other through ZeroTier's peer discovery infrastructure |
| **NAT traversal** | Nodes behind NAT can communicate without port forwarding |
| **Encrypted transport** | All overlay traffic is end-to-end encrypted |
| **Address assignment** | The controller assigns public IPs from your block to authorized nodes |
| **Ingress routing** | The AlterMundi fork installs per-node source routing automatically |

---

## The AlterMundi Fork

The standard ZeroTier client does not install source-based policy routing. The AlterMundi fork (`github.com/AlterMundi/ZeroTierOne`, branch `feature/ingress-node`) adds:

1. **`ingressNodeV4` field** — A new network configuration parameter that specifies which node is the default gateway for Internet-bound traffic
2. **Automatic source routing** — When a node is authorized and receives an IP, the fork installs:
   ```bash
   ip rule add from <assigned-public-ip>/32 lookup <table>
   ip route add default via <ingressNodeV4> table <table>
   ```
3. **Deadlock fix** — A mutex guard in `syncManagedStuff()` prevents a race condition during network join

This automation is why mesh nodes require no manual networking configuration after authorization.

---

## Controller vs. Client Roles

### Border Router (Controller)

The border router runs ZeroTier in controller mode. The controller:
- Maintains the network definition (name, IP pools, routes, `ingressNodeV4`)
- Authorizes or rejects joining nodes
- Assigns IP addresses from the configured pool
- Distributes `ingressNodeV4` to all joining nodes

The border router sets `allowDefault=0` in its local ZeroTier configuration. This prevents the border router from accepting a default route from the network it controls — which would create a routing loop.

### Mesh Nodes (Clients)

Mesh nodes run ZeroTier in client mode. Each node:
- Joins the network using the 16-character network ID
- Waits for authorization from the controller
- Receives a public IP and `ingressNodeV4` from the controller
- Has source routing installed automatically by the fork

Mesh nodes set `allowDefault=1` to accept the default route from the controller, enabling the ingress routing mechanism.

---

## Per-Network Configuration

The ZeroTier client on each node uses a per-network configuration file. The AlterMundi fork reads this to set node behavior:

```json
{
  "allowManaged": true,
  "allowGlobal": true,
  "allowDefault": true,
  "allowDNS": false
}
```

| Setting | Meaning |
|---------|---------|
| `allowManaged` | Accept IP assignments from the controller |
| `allowGlobal` | Accept routes to global (non-private) IP space |
| `allowDefault` | Accept a default route from the controller (required for ingress routing) |
| `allowDNS` | Accept DNS configuration from the controller (disabled by default) |

---

## Network Creation

When creating the ZeroTier network, the key parameters are:

```json
{
  "name": "44mesh",
  "private": true,
  "v4AssignMode": {"zt": true},
  "ipAssignmentPools": [
    {
      "ipRangeStart": "44.x.y.2",
      "ipRangeEnd": "44.x.y.254"
    }
  ],
  "routes": [
    {"target": "0.0.0.0/0", "via": null}
  ],
  "ingressNodeV4": "<border-router-zt-ip>"
}
```

The `ingressNodeV4` value is the ZeroTier-assigned public IP of the border router node within the mesh. All other nodes will route Internet-bound traffic through this address.

!!! important "Set ingressNodeV4 after the border router joins"
    The border router must join its own network first and receive an IP assignment before `ingressNodeV4` can be set. The AlterMundi ZeroTier UI fork provides a button to set this after the network is created.

---

## Node Lifecycle

```
Node starts
   │
   ▼
zerotier-cli join <network-id>
   │
   ▼
Controller receives join request
   │
   ▼
Admin authorizes node (UI or API)
   │
   ▼
Controller assigns IP from pool
Controller sends ingressNodeV4
   │
   ▼
Fork installs source routing rules
   │
   ▼
Node is publicly reachable
```

---

## Useful Commands

On any ZeroTier node:

```bash
# Check ZeroTier daemon status
docker exec zerotier zerotier-cli info

# List joined networks and assigned IPs
docker exec zerotier zerotier-cli listnetworks

# List known peers
docker exec zerotier zerotier-cli listpeers

# Join a network manually
docker exec zerotier zerotier-cli join <network-id>
```

Controller API (run on the border router host):

```bash
# Get auth token
docker exec zerotier cat /var/lib/zerotier-one/authtoken.secret

# List networks
curl -H "X-ZT1-Auth: <token>" http://localhost:9993/controller/network

# List members of a network
curl -H "X-ZT1-Auth: <token>" http://localhost:9993/controller/network/<network-id>/member

# Authorize a member
curl -X POST -H "X-ZT1-Auth: <token>" \
  -d '{"authorized": true}' \
  http://localhost:9993/controller/network/<network-id>/member/<node-id>
```

---

## ZeroTier Ports

| Port | Protocol | Purpose |
|------|----------|---------|
| `9993` | UDP | ZeroTier peer-to-peer communication |
| `9993` | TCP | ZeroTier local API (localhost only) |

Port 9993/UDP should be reachable from the Internet for optimal peer connectivity. ZeroTier can work through NAT without this, but direct connectivity improves performance.
