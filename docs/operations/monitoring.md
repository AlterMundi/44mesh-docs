---
title: Monitoring
description: Health checks, BGP session status, routing verification, connectivity tests, and log inspection commands.
topics: [monitoring, health-checks, bgp-status, routing-verification, connectivity-testing, logging]
related:
  - docs/operations/troubleshooting.md
  - docs/operations/security.md
  - docs/deployment/border-router.md
---

# Monitoring

This page covers key metrics, health checks, and commands for monitoring a running 44Mesh network.

---

## Container Health

All 44Mesh containers include health checks. Check container status:

```bash
# Border router
cd deploy/bird-border
docker compose ps

# Mesh node
cd deploy/zerotier
docker compose ps
```

Health check details:
- **ZeroTier controller/node**: checks `zerotier-cli info` responds successfully
- **BIRD2**: checks `birdc show status` responds successfully

---

## BGP Session Status

Check the BGP session on the border router:

```bash
docker exec bird-border birdc show protocols
```

Key fields to watch:

| Field | Healthy Value |
|-------|--------------|
| `isp` protocol state | `Established` |
| Since | Should not be constantly resetting (indicates flapping) |
| Info | `Established` for the BGP neighbor |

```bash
# More detail on the BGP session
docker exec bird-border birdc "show protocols all isp"
```

Watch for BGP route changes:

```bash
docker exec bird-border birdc "show route count"
```

---

## Routing Table Verification

Verify your mesh prefix is being announced:

```bash
# What BIRD is exporting to the ISP
docker exec bird-border birdc "show route export isp"

# Full BIRD routing table
docker exec bird-border birdc show route

# Kernel routing table on the host
ip route show
ip route show table 123
```

---

## ZeroTier Network Status

On the border router / controller host:

```bash
TOKEN=$(docker exec zerotier cat /var/lib/zerotier-one/authtoken.secret)
NETWORK_ID=<your-network-id>

# List all members and their status
curl -s -H "X-ZT1-Auth: ${TOKEN}" \
  "http://localhost:9993/controller/network/${NETWORK_ID}/member" | python3 -m json.tool
```

On any node:

```bash
# Node status and version
docker exec zerotier zerotier-cli info

# Network membership and IP assignment
docker exec zerotier zerotier-cli listnetworks

# Connected peers
docker exec zerotier zerotier-cli listpeers
```

---

## Source Routing Verification

On the border router:

```bash
# Policy routing rules
ip rule show

# Table 123 (mesh → ISP forwarding)
ip route show table 123
```

Expected:
```
32765:  from 44.x.y.0/24 lookup 123
```
```
default via <ISP_IP>
```

On a mesh node:

```bash
# Per-node policy routing rule
ip rule show

# Per-node routing table
ip route show table <table-number>
```

---

## Connectivity Tests

### Outbound from a mesh node

```bash
# Test that traffic from the public IP routes correctly
docker exec zerotier curl --interface <mesh-node-ip> https://ifconfig.me
```

The response should be the border router's public IP (not the node's underlying Internet IP).

### Inbound to a mesh node

From any Internet-connected host:

```bash
# Run a service on the node first
docker exec zerotier python3 -m http.server 8080

# Test from outside
curl http://<mesh-node-public-ip>:8080
```

### Node-to-node connectivity

Mesh nodes can communicate directly over ZeroTier:

```bash
# From node A, ping node B
docker exec zerotier ping <node-b-mesh-ip>
```

---

## Log Monitoring

### BIRD logs

```bash
docker exec bird-border cat /var/log/bird/bird.log
# or follow live:
docker logs -f bird-border
```

### ZeroTier logs

```bash
docker logs -f zerotier
```

### ZeroTier UI logs

```bash
docker logs -f zerotier-ui
```

---

## Key Metrics to Watch

| Metric | What to Check |
|--------|--------------|
| BGP session uptime | Should be stable; frequent resets indicate a problem |
| BGP prefix count | Your mesh prefix should always be announced |
| ZeroTier node count | Number of authorized members |
| Source routing rules | Table 123 should always have a default route |
| Node IP assignments | Each authorized node should have an IP |

---

## Automated Health Checks

The GitHub Actions deployment pipeline verifies health after deployment:

```bash
# Wait for zerotier healthy + bird-border running (5min timeout)
# Waits for zerotier-ui healthy (2min timeout)
# Shows BIRD status in deployment summary
```

For production monitoring, consider integrating with a monitoring system (Prometheus, Grafana, Zabbix) using BIRD's built-in metrics or custom scripts that wrap the `birdc` commands.
