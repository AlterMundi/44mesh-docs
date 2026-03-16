# Mesh Nodes

Mesh nodes are the distributed participants in the 44Mesh network. Each node connects to the ZeroTier overlay, receives a public IP from the border router controller, and becomes directly reachable from the Internet.

---

## What Gets Deployed

`docker compose up` in `deploy/zerotier/` starts a single container:

| Container | Image | Purpose |
|-----------|-------|---------|
| `zerotier` | `buzondefede/44mesh-zerotier` | ZeroTier client (AlterMundi fork) |

The container runs on host networking and requires `NET_ADMIN` and `SYS_ADMIN` capabilities for source routing and tun/tap device management.

---

## Host Requirements

- Linux (amd64, arm64, or arm/v7)
- Docker 20.10+
- Docker Compose 2.x
- Port 9993/UDP accessible outbound (for ZeroTier peer connectivity)
- No inbound port requirements — ZeroTier handles NAT traversal

!!! note "Raspberry Pi support"
    The mesh node image supports arm64 and arm/v7, making it suitable for Raspberry Pi deployments.

---

## Configuration

```bash
cd deploy/zerotier
cp .env.example .env
```

Minimum required configuration:

```bash
ZT_NETWORK_ID=<16-character-hex-network-id>
```

Optional configuration for persistent node identity:

```bash
# Pre-seed a specific ZeroTier identity (preserves node ID across restarts)
ZEROTIER_IDENTITY_PUBLIC=<public-key>
ZEROTIER_IDENTITY_SECRET=<secret-key>

# Pre-seed the API auth token
ZEROTIER_API_SECRET=<token>
```

---

## Deploy

```bash
docker compose up -d
```

The entrypoint will:
1. Pre-seed identity files if `ZEROTIER_IDENTITY_PUBLIC/SECRET` are set
2. Copy per-network configuration (`local.conf`, `network.local.conf`)
3. Start the ZeroTier daemon
4. Join the configured network

Watch the logs:

```bash
docker compose logs -f
```

---

## Authorize the Node

After the node starts, it will appear in the controller as an unauthorized member. Until authorized, it has no IP assignment and no routing.

### Via API

On the border router host:

```bash
TOKEN=$(docker exec zerotier cat /var/lib/zerotier-one/authtoken.secret)
NODE_ID=<10-char-node-id>
NETWORK_ID=<16-char-network-id>

curl -X POST \
  -H "X-ZT1-Auth: ${TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{"authorized": true}' \
  "http://localhost:9993/controller/network/${NETWORK_ID}/member/${NODE_ID}"
```

Get the node ID from the node itself:

```bash
docker exec zerotier zerotier-cli info
```

### Via ZeroTier UI

If the [ZeroTier UI](zerotier-ui.md) is deployed, authorize the node through the web interface.

---

## Verify

After authorization, the node receives an IP and source routing is installed:

```bash
# Check network status and IP assignment
docker exec zerotier zerotier-cli listnetworks
```

Expected output:

```
200 listnetworks <network-id> 44mesh OK 44.30.127.42/24 ...
```

Check that source routing rules were installed:

```bash
ip rule show
ip route show table <table-number>
```

Test outbound routing from the public IP:

```bash
curl --interface 44.30.127.42 https://ifconfig.me
```

This should return the border router's public IP (traffic exits via the border router).

---

## Node Identity Persistence

By default, ZeroTier generates a new identity each time a fresh container starts. This means the node gets a different ID on each restart, requiring re-authorization.

To preserve the node ID across restarts, pre-seed the identity:

### 1. Get the current identity

```bash
docker exec zerotier cat /var/lib/zerotier-one/identity.public
docker exec zerotier cat /var/lib/zerotier-one/identity.secret
```

### 2. Store in `.env`

```bash
ZEROTIER_IDENTITY_PUBLIC=<contents of identity.public>
ZEROTIER_IDENTITY_SECRET=<contents of identity.secret>
```

The entrypoint script writes these files before starting the daemon, preserving the node ID.

!!! warning "Protect the secret key"
    The identity secret is a private key. Do not commit it to version control. Use proper secrets management (environment variable injection, Docker secrets, etc.).

---

## Running Services on Mesh Nodes

Once a node has a public IP, it can host any service directly. The service binds to the node's public IP (or `0.0.0.0`) and is reachable from the Internet.

Example — run a web server:

```bash
# On the mesh node host
python3 -m http.server 8080 --bind 44.30.127.42
```

Access from anywhere:

```bash
curl http://44.30.127.42:8080
```

No port forwarding, no NAT configuration, no reverse proxy required for basic use.

---

## Multiple Nodes

Each node runs the same Docker Compose stack, connecting to the same network ID. They are independent — nodes can be added or removed without affecting the rest of the network.

The controller assigns each node a unique IP from the pool. Nodes can communicate directly with each other over ZeroTier, not just through the border router.

---

## Container Details

The mesh node image is a multi-stage build:
- **Stage 1**: Ubuntu 22.04 + Rust toolchain → builds the AlterMundi ZeroTierOne fork
- **Stage 2**: Ubuntu 22.04 runtime image with the built binary

Build source: `github.com/AlterMundi/ZeroTierOne`, branch `feature/ingress-node`

Capabilities:
- `NET_ADMIN` — routing rules and interface configuration
- `SYS_ADMIN` — tun/tap device and network namespace management
- `/dev/net/tun` — ZeroTier overlay interface
