---
title: Operating a Node
description: Node operator responsibilities including updates, service deployment, firewall configuration, and monitoring.
topics: [node-operations, maintenance, systemd, service-deployment, firewall, resource-monitoring]
related:
  - docs/deployment/mesh-nodes.md
  - docs/operations/monitoring.md
  - docs/ai-agents/observation-api.md
---

# Operating a Node

This page covers ongoing responsibilities and best practices for operating a mesh node in 44Mesh.

---

## Node Operator Responsibilities

As a node operator, you are responsible for:

- Keeping the node online and connected to the mesh
- Updating the ZeroTier client when new versions are released
- Securing access to the host running the node
- Ensuring services running on the node are properly configured and secured

---

## Keeping the Node Running

### Restart Policy

The Docker Compose stack includes `restart: unless-stopped` (or equivalent). Ensure the Docker service starts on boot:

```bash
sudo systemctl enable docker
sudo systemctl start docker
```

### Auto-Start on Boot

```bash
cd deploy/zerotier
docker compose up -d
```

To ensure the compose stack starts with Docker, you can add a systemd unit:

```ini
# /etc/systemd/system/44mesh-node.service
[Unit]
Description=44Mesh Node
After=docker.service
Requires=docker.service

[Service]
Type=oneshot
RemainAfterExit=yes
WorkingDirectory=/path/to/deploy/zerotier
ExecStart=/usr/bin/docker compose up -d
ExecStop=/usr/bin/docker compose down
TimeoutStartSec=0

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl enable 44mesh-node
sudo systemctl start 44mesh-node
```

---

## Updating

Pull the latest image and restart:

```bash
cd deploy/zerotier
docker compose pull
docker compose up -d
```

Source routing rules are reinstalled by the ZeroTier fork automatically when the daemon reconnects.

---

## Node Identity

By default, ZeroTier generates a new identity on each fresh container start. To preserve your node ID:

1. Note your current identity:
   ```bash
   docker exec zerotier cat /var/lib/zerotier-one/identity.public
   docker exec zerotier cat /var/lib/zerotier-one/identity.secret
   ```
2. Store in `.env`:
   ```bash
   ZEROTIER_IDENTITY_PUBLIC=<value>
   ZEROTIER_IDENTITY_SECRET=<value>
   ```

With a persistent identity, your node ID remains the same across restarts and you don't need to be re-authorized after updates.

---

## Running Services

Mesh nodes can run any service. The service should bind to the node's public mesh IP (or `0.0.0.0`):

```bash
# Example: web server
python3 -m http.server 8080

# Example: bind to specific mesh IP
python3 -m http.server 8080 --bind 44.x.y.z
```

### Firewall Recommendations

Allow only the ports your services use:

```bash
# Allow specific service ports
iptables -A INPUT -p tcp --dport 8080 -j ACCEPT

# Allow ZeroTier
iptables -A INPUT -p udp --dport 9993 -j ACCEPT

# Allow established connections
iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

# Drop everything else
iptables -A INPUT -j DROP
```

---

## Monitoring Your Node

```bash
# Node status
docker exec zerotier zerotier-cli info

# Network status and IP assignment
docker exec zerotier zerotier-cli listnetworks

# Connected peers
docker exec zerotier zerotier-cli listpeers

# Container health
docker compose ps
```

Set up basic alerting to notify you if the container stops.

---

## Resource Usage

The ZeroTier client has minimal resource requirements:

| Resource | Typical Usage |
|----------|--------------|
| CPU | < 1% at idle |
| RAM | ~50–100 MB |
| Network | ~1 KB/s overhead (keepalives) |
| Disk | ~100 MB for the container image |

The AlterMundi fork does not significantly increase resource usage over the standard ZeroTier client.

---

## Sensor and AI Agent Workloads

If your node hosts sensors or AI agent endpoints, additional considerations apply:

- Expose your capabilities via the observation API (see [Observation API](../ai-agents/observation-api.md))
- Ensure the endpoint is accessible on the public IP
- Consider rate limiting to prevent overload from automated queries
- Document your sensor capabilities so agents can discover them

See the [AI Agents](../ai-agents/overview.md) section for the data model and API conventions.
