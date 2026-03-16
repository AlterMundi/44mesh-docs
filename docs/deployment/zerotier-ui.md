---
title: ZeroTier UI
description: Web interface deployment for managing ZeroTier networks, authorizing nodes, and configuring ingressNodeV4.
topics: [zerotier-ui, web-interface, node-authorization, ingress-node, ip-pool-management]
related:
  - docs/deployment/border-router.md
  - docs/deployment/mesh-nodes.md
  - docs/operations/security.md
---

# ZeroTier UI

The ZeroTier UI is an optional web interface for managing the 44Mesh network. It provides a graphical alternative to the controller API for authorizing nodes, managing IP assignments, and configuring the `ingressNodeV4` setting.

---

## What Gets Deployed

`docker compose up` in `deploy/zerotier-ui/` starts a single container:

| Container | Image | Purpose |
|-----------|-------|---------|
| `zerotier-ui` | `buzondefede/44mesh-zerotier-ui` | Web interface for ZeroTier controller |

The container reads the controller auth token from the shared `zerotier_data` volume, so it must run on the same host as the border router.

---

## Prerequisites

The border router must be running and its `zerotier_data` volume must exist before deploying the UI.

---

## Configuration

```bash
cd deploy/zerotier-ui
cp .env.example .env
```

```bash
# Admin password for the web interface
ZTNCUI_PASSWD=your-secure-password

# HTTP port (default: 3180)
ZTNCUI_HTTP_PORT=3180

# HTTPS port (default: 3443)
ZTNCUI_HTTPS_PORT=3443

# Bind to all interfaces (set to true for remote access)
ZTNCUI_HTTP_ALL_INTERFACES=true

# Optional: set a session secret for persistent sessions across restarts
# SESSION_SECRET=your-session-secret
```

!!! warning "Password security"
    `ZTNCUI_PASSWD` is stored as plaintext in the `.env` file. Restrict file permissions (`chmod 600 .env`) and use a firewall to limit access to the UI port.

---

## Deploy

```bash
docker compose up -d
```

Access the UI at `http://<host-ip>:3180`.

Log in with the password set in `ZTNCUI_PASSWD`.

---

## Usage

### Viewing Networks

The home page lists all networks managed by the controller. Click a network to view its members and configuration.

### Authorizing Nodes

1. Navigate to the network
2. Find the pending member in the members list
3. Click **Authorize** to grant the node access
4. The node receives an IP from the pool and source routing is installed automatically

### Setting ingressNodeV4

The AlterMundi fork of the UI adds a dedicated button for setting the `ingressNodeV4` parameter:

1. Navigate to the network configuration page
2. Find the **Ingress Node** section
3. Enter the border router's assigned ZeroTier IP
4. Click **Set Ingress Node**

This distributes the ingress node address to all current and future members.

### Managing IP Pools

The IP assignment pool can be adjusted in the network configuration:
- Set the start and end of the pool
- Individual members can have specific IPs pinned

---

## Security Considerations

The web interface has direct access to the ZeroTier controller API. Treat it as a privileged management interface:

- Place it behind a reverse proxy with TLS in production
- Use a firewall to restrict access to the UI port
- Generate a strong, random password
- Consider binding to a specific interface rather than all interfaces if not needed externally

### Using a Reverse Proxy

For production use, place the UI behind Nginx or Caddy:

```nginx
server {
    listen 443 ssl;
    server_name zerotier-ui.yourdomain.com;

    ssl_certificate /path/to/cert.pem;
    ssl_certificate_key /path/to/key.pem;

    location / {
        proxy_pass http://127.0.0.1:3180;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

In this setup, set `ZTNCUI_HTTP_ALL_INTERFACES=false` and bind only to localhost.

---

## Container Details

- **Base**: Node.js 18 on Debian Bookworm
- **Source**: AlterMundi fork of `ztncui` (`v0.0.13-altermundi`)
- **Auth token**: Read from `/var/lib/zerotier-one/authtoken.secret` (shared volume)
- **Runs as**: Unprivileged `ztncui` user (via `gosu`)
- **Volumes**: `zerotier_data` shared with border router (read-only access to auth token)
