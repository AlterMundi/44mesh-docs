# Border Router Deployment

The border router is the core component of 44Mesh. It runs the ZeroTier controller and the BIRD2 BGP daemon on the same host, connecting the mesh to the Internet.

---

## What Gets Deployed

`docker compose up` in `deploy/bird-border/` starts two containers:

| Container | Image | Purpose |
|-----------|-------|---------|
| `zerotier` | `buzondefede/44mesh-bird-border` | ZeroTier controller (AlterMundi fork) |
| `bird-border` | `buzondefede/44mesh-bird-border` | BIRD2 BGP daemon |

Both containers run on host networking and share a `zerotier_data` volume.

---

## Host Preparation

### 1. Enable IP Forwarding

```bash
echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

### 2. Add Secondary IP for BGP Peering

The BGP daemon needs `BORDER_ROUTER_IP` to be assigned to a host interface. This is the IP your ISP will peer with.

=== "NetworkManager"
    ```bash
    sudo nmcli con mod <connection-name> +ipv4.addresses <BORDER_ROUTER_IP>/32
    sudo nmcli con up <connection-name>
    ```

=== "systemd-networkd"
    ```ini
    # /etc/systemd/network/10-border.conf
    [Match]
    Name=eth0

    [Network]
    Address=<BORDER_ROUTER_IP>/32
    ```
    ```bash
    sudo systemctl restart systemd-networkd
    ```

=== "Debian interfaces.d"
    ```
    # /etc/network/interfaces.d/border-router
    auto eth0:0
    iface eth0:0 inet static
        address <BORDER_ROUTER_IP>
        netmask 255.255.255.255
    ```

Verify the IP is assigned:

```bash
ip addr show | grep <BORDER_ROUTER_IP>
```

---

## Configuration

```bash
cd deploy/bird-border
cp .env.example .env
```

Edit `.env`:

```bash
# Your BGP AS number
BORDER_ROUTER_AS=65000

# Your BGP peering IP (must be assigned to a host interface)
BORDER_ROUTER_IP=203.0.113.2

# Your ISP's BGP peer IP
ISP_IP=203.0.113.1

# Your ISP's AS number
ISP_AS=65001

# Your public IP block to announce
MESH_ADDRESS_RANGE=44.30.127.0/24

# BGP timers
BGP_HOLD_TIME=90
BGP_KEEPALIVE_TIME=30
```

---

## Deploy

```bash
docker compose up -d
```

Watch the logs during startup:

```bash
docker compose logs -f
```

The `bird-border` entrypoint will:
1. Wait for the ZeroTier interface (`zt*`) to appear
2. Install source routing rules (`ip rule`, `ip route`)
3. Generate the BIRD config from the template using `envsubst`
4. Start the BIRD2 daemon

---

## Verify

### Container Health

```bash
docker compose ps
```

Both containers should show `healthy` status.

### BGP Session

```bash
docker exec bird-border birdc show protocols
```

Look for `isp` protocol with `Established` state.

```bash
docker exec bird-border birdc show route
```

Your mesh prefix should appear. You should also see the default route (`0.0.0.0/0`) received from the ISP.

### Source Routing

```bash
ip rule show | grep "44\."
ip route show table 123
```

### ZeroTier Controller

```bash
docker exec zerotier zerotier-cli info
```

Note the node ID (10-character hex string) — you'll need it to construct the network ID.

---

## Create the ZeroTier Network

After the controller is running, create the overlay network:

```bash
# Get the controller's auth token
TOKEN=$(docker exec zerotier cat /var/lib/zerotier-one/authtoken.secret)

# Get the controller's node ID
NODE_ID=$(docker exec zerotier zerotier-cli info | awk '{print $3}')

# Create network
curl -s -X POST "http://localhost:9993/controller/network/${NODE_ID}______" \
  -H "X-ZT1-Auth: ${TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "44mesh",
    "private": true,
    "v4AssignMode": {"zt": true},
    "ipAssignmentPools": [
      {"ipRangeStart": "44.30.127.2", "ipRangeEnd": "44.30.127.254"}
    ],
    "routes": [
      {"target": "0.0.0.0/0", "via": null}
    ]
  }'
```

Note the `id` field in the response — this is your **network ID**.

Then have the border router join its own network:

```bash
docker exec zerotier zerotier-cli join <network-id>
```

Authorize it and note its assigned IP, then set `ingressNodeV4`:

```bash
# Get the border router's member ID (same as its node ID)
curl -s -H "X-ZT1-Auth: ${TOKEN}" \
  "http://localhost:9993/controller/network/<network-id>/member"

# Authorize and set IP
curl -X POST -H "X-ZT1-Auth: ${TOKEN}" \
  -d '{"authorized": true}' \
  "http://localhost:9993/controller/network/<network-id>/member/<node-id>"

# Set ingressNodeV4 to the border router's assigned IP
curl -X POST -H "X-ZT1-Auth: ${TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{"ingressNodeV4": "44.30.127.2"}' \
  "http://localhost:9993/controller/network/<network-id>"
```

---

## Data Persistence

The `zerotier_data` Docker volume stores:
- ZeroTier identity (`identity.public`, `identity.secret`)
- Controller database (network definitions, member assignments)
- Auth token

**Do not delete this volume** unless you intend to reset the controller. Losing it means losing all member assignments and requiring nodes to be re-authorized.

Back up the volume periodically:

```bash
docker run --rm -v zerotier_data:/data -v $(pwd):/backup \
  alpine tar czf /backup/zerotier-data-backup.tar.gz /data
```

---

## Updating

```bash
docker compose pull
docker compose up -d
```

The containers will restart with the new image. Source routing rules are reinstalled automatically by the entrypoint script.

---

## Container Details

The border router image is built from `deploy/bird-border/Dockerfile`:

- **Base**: `debian:12-slim`
- **Packages**: `bird2`, `iproute2`, `gettext-base`, `curl`
- **Capabilities required**: `NET_ADMIN` (routing rules), `SYS_ADMIN` (ZeroTier tun/tap)
- **Device**: `/dev/net/tun` (ZeroTier overlay interface)
- **Network mode**: `host` (required for BGP direct peering)
