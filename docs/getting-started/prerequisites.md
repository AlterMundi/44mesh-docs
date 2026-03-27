<!--
SPDX-FileCopyrightText: 2024 AlterMundi <docs@44mesh.net>

SPDX-License-Identifier: CC-BY-SA-4.0
-->

# Prerequisites

Before deploying 44Mesh, you need to gather network resources and prepare the host system. The requirements differ depending on your role: running a **border router** (full participation with public routing) or deploying **mesh nodes only** (joining an existing network).

---

## Border Router Prerequisites

### Network Resources

| Resource | Required | Notes |
|----------|----------|-------|
| **AS Number** | Yes | Obtained from an RIR (ARIN, LACNIC, RIPE, APNIC, AFRINIC) or borrowed from a sponsor |
| **Public IPv4 Block** | Yes | `/24` or larger recommended; can be from RIR, AMPRNet (44/8), or leased |
| **BGP Peer** | Yes | An ISP or IXP willing to establish a BGP session with your AS |
| **BGP Peering IP** | Yes | A link-local or dedicated IP for the BGP session (assigned by your ISP) |

!!! note "AMPRNet allocations"
    If you are an amateur radio operator, you can apply for a `/24` from the [AMPRNet](https://www.ampr.org/) 44/8 block. This is free but requires an amateur radio license.

### Host System

| Requirement | Details |
|-------------|---------|
| **Operating System** | Linux (any modern distribution) |
| **Docker** | Version 20.10 or later |
| **Docker Compose** | Version 2.x (`docker compose` plugin, not `docker-compose`) |
| **IP forwarding** | Must be enabled: `net.ipv4.ip_forward=1` |
| **Secondary IP** | A secondary IP in your BGP peering range must be assigned to a host interface |
| **Port 179/TCP** | Open for BGP; must be reachable from your ISP's BGP peer |
| **Port 9993/UDP** | Open for ZeroTier peer connectivity |

#### Enable IP Forwarding

```bash
echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf
sysctl -p
```

#### Add Secondary IP for BGP Peering

The border router needs a secondary IP address on a host interface for BGP peering. The method depends on your system:

=== "NetworkManager (nmcli)"
    ```bash
    nmcli con mod <connection-name> +ipv4.addresses <border-router-ip>/32
    nmcli con up <connection-name>
    ```

=== "systemd-networkd"
    Create `/etc/systemd/network/10-border.conf`:
    ```ini
    [Match]
    Name=eth0

    [Network]
    Address=<border-router-ip>/32
    ```
    Then: `systemctl restart systemd-networkd`

=== "interfaces.d (Debian/Ubuntu)"
    Add to `/etc/network/interfaces.d/border-router`:
    ```
    auto eth0:0
    iface eth0:0 inet static
        address <border-router-ip>
        netmask 255.255.255.255
    ```

---

## Mesh Node Prerequisites

If you are joining an existing 44Mesh network (border router is already running):

| Requirement | Details |
|-------------|---------|
| **Operating System** | Linux (any modern distribution); arm64 and arm/v7 supported |
| **Docker** | Version 20.10 or later |
| **Docker Compose** | Version 2.x |
| **Port 9993/UDP** | Outbound UDP access required for ZeroTier |
| **ZeroTier Network ID** | Provided by the network operator |

!!! tip "Raspberry Pi"
    Mesh nodes work on Raspberry Pi (arm64 or arm/v7). The Docker images are built for all three architectures.

---

## Development / Testing Prerequisites

For local testing with the mock ISP (no real BGP peer needed):

| Requirement | Details |
|-------------|---------|
| **Docker** | Version 20.10 or later |
| **Docker Compose** | Version 2.x |
| **IP forwarding** | Must be enabled |
| **Two hosts** | Or two network namespaces — one for border router, one for mock ISP |

The mock ISP simulates a BGP peer using RFC 5737 test prefixes (`192.0.2.0/24`, `198.51.100.0/24`, `203.0.113.0/24`), so no real IP resources are required for testing.

---

## What You Will Configure

Once prerequisites are met, you will configure the following environment variables:

```bash
# Border Router
BORDER_ROUTER_AS=65000           # Your BGP AS number
BORDER_ROUTER_IP=192.0.2.1       # Your BGP peering IP (secondary IP on host)
ISP_IP=192.0.2.254               # Your ISP's BGP peer IP
ISP_AS=65001                     # Your ISP's AS number
MESH_ADDRESS_RANGE=44.x.y.0/24  # Your public IP block
BGP_HOLD_TIME=90
BGP_KEEPALIVE_TIME=30

# Mesh Nodes
ZT_NETWORK_ID=<16-hex-chars>     # ZeroTier network ID
```

See the [Quickstart](quickstart.md) for the full deployment sequence.
