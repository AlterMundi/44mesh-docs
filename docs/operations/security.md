# Security

This page covers the security model of 44Mesh and recommendations for hardening a deployment.

---

## Container Capabilities

44Mesh containers require elevated Linux capabilities that are not available to standard Docker containers.

### Capabilities Used

| Capability | Container | Reason |
|-----------|-----------|--------|
| `NET_ADMIN` | All | Installing routing rules (`ip rule`, `ip route`), configuring interfaces |
| `SYS_ADMIN` | ZeroTier only | Creating and managing tun/tap devices, network namespace operations |

These capabilities are explicitly justified in the project documentation. They are required for the core networking functionality and cannot be reduced without changing the architecture.

!!! note "No host root access"
    The containers do not run as the host root user. The capabilities are granted specifically for networking operations within the container's namespace.

### `/dev/net/tun`

ZeroTier requires access to the `/dev/net/tun` device to create its virtual network interface. This is standard for VPN and overlay networking tools.

---

## Network Exposure

### Border Router

| Port | Protocol | Exposure | Notes |
|------|----------|----------|-------|
| `179` | TCP | Internet | BGP — must be reachable from your ISP's BGP peer |
| `9993` | UDP | Internet | ZeroTier peer communication |
| `9993` | TCP | Localhost only | ZeroTier local API |

Restrict firewall rules so that port 179 is only reachable from your ISP's peering IP:

```bash
# Example iptables rule
iptables -A INPUT -p tcp --dport 179 -s <ISP_PEER_IP> -j ACCEPT
iptables -A INPUT -p tcp --dport 179 -j DROP
```

### ZeroTier UI

| Port | Protocol | Exposure | Notes |
|------|----------|----------|-------|
| `3180` | TCP | Configurable | HTTP web interface |
| `3443` | TCP | Configurable | HTTPS web interface |

!!! danger "Restrict UI access"
    The ZeroTier UI has full control over the network (authorizing members, changing IP assignments, setting `ingressNodeV4`). Restrict access to trusted networks only. Do not expose it to the public Internet without authentication measures beyond the password.

---

## Credentials Management

### ZeroTier Auth Token

The controller generates an auth token stored in the `zerotier_data` volume at `/var/lib/zerotier-one/authtoken.secret`. This token grants full access to the controller API.

- The token is automatically shared with the ZeroTier UI via the shared volume
- Treat it as a sensitive credential
- Back up the volume securely

### ZeroTier UI Password

`ZTNCUI_PASSWD` is stored plaintext in the `.env` file.

Recommendations:
- Use a strong, random password
- Set restrictive file permissions: `chmod 600 .env`
- Do not commit `.env` files to version control (enforced by `.gitignore`)
- Consider using Docker secrets or environment variable injection from a secrets manager

### Node Identity Keys

If you pre-seed ZeroTier node identities using `ZEROTIER_IDENTITY_SECRET`, treat these as private keys:
- Never commit to version control
- Use secrets management (Vault, AWS Secrets Manager, etc.)
- Rotate if compromised — authorized node ID changes if the key changes, requiring re-authorization

---

## TLS for the Web UI

For production deployments, place the ZeroTier UI behind a reverse proxy with TLS:

```nginx
server {
    listen 443 ssl http2;
    server_name zerotier-ui.yourdomain.com;

    ssl_certificate /etc/letsencrypt/live/yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/yourdomain.com/privkey.pem;

    location / {
        proxy_pass http://127.0.0.1:3180;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Set `ZTNCUI_HTTP_ALL_INTERFACES=false` and bind only to localhost when using a reverse proxy.

---

## BGP Security

### Prefix Filtering

The BIRD configuration includes an export filter that restricts announcements to your mesh range only:

```
export where source = RTS_STATIC;
```

This prevents accidental announcement of routes that are not yours.

For additional protection, consider configuring RPKI (Resource Public Key Infrastructure) validation with your ISP. RPKI allows the ISP to verify that you are authorized to announce your prefix.

### BGP Session Authentication

TCP-MD5 authentication can be added to the BGP session to prevent session hijacking:

```bird
protocol bgp isp {
  password "your-bgp-session-password";
  ...
}
```

Coordinate the password with your ISP.

---

## ZeroTier Network Security

The ZeroTier network should be configured as **private** (`"private": true`), which requires explicit authorization for each node. Public networks allow any node to join without authorization — never use a public network in production.

Node authorization should be reviewed periodically. Deauthorize nodes that are no longer active.

---

## Firewall Recommendations

On the border router host:

```bash
# Allow BGP from ISP peer only
iptables -A INPUT -p tcp --dport 179 -s <ISP_IP> -j ACCEPT
iptables -A INPUT -p tcp --dport 179 -j DROP

# Allow ZeroTier
iptables -A INPUT -p udp --dport 9993 -j ACCEPT

# Allow ZeroTier UI only from management network
iptables -A INPUT -p tcp --dport 3180 -s <management-cidr> -j ACCEPT
iptables -A INPUT -p tcp --dport 3180 -j DROP

# Allow IP forwarding for mesh traffic
iptables -A FORWARD -j ACCEPT
```

On mesh nodes:

```bash
# Allow ZeroTier
iptables -A INPUT -p udp --dport 9993 -j ACCEPT

# Allow services you're running
iptables -A INPUT -p tcp --dport 8080 -j ACCEPT

# Drop everything else
iptables -A INPUT -j DROP
```
