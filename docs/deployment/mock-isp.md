---
title: Mock ISP (Testing)
description: Simulated BGP peer for local development and testing using RFC 5737 test prefixes.
topics: [mock-isp, testing, bgp, bird2, local-development, rfc-5737]
related:
  - docs/getting-started/quickstart.md
  - docs/network/bgp.md
  - docs/deployment/border-router.md
---

# Mock ISP (Testing)

The mock ISP component simulates an upstream BGP peer for local development and testing. It allows you to test the full 44Mesh stack — including BGP session establishment and route propagation — without a real ISP connection.

---

## What It Does

The mock ISP runs a BIRD2 instance that:
- Establishes an eBGP session with your border router
- Announces three RFC 5737 test prefixes (`192.0.2.0/24`, `198.51.100.0/24`, `203.0.113.0/24`)
- Accepts your mesh range (`MESH_ADDRESS_RANGE`) from the border router
- Simulates normal ISP behavior without requiring real BGP peering

RFC 5737 addresses are documentation ranges and are not routed on the Internet, making them safe to use in testing environments.

---

## When to Use It

| Scenario | Use Mock ISP? |
|----------|---------------|
| Local development | Yes |
| Testing BGP configuration before connecting to a real ISP | Yes |
| CI/CD pipelines | Yes (if testing routing) |
| Production | No — use a real ISP or IXP |

---

## Host Requirements

The mock ISP must be able to reach the border router's BGP peering IP (`BORDER_ROUTER_IP`). It can run:
- On the same host as the border router (different process)
- On a separate host connected to the same network

---

## Configuration

```bash
cd deploy/rpi-isp
cp .env.example .env
```

```bash
# Mock ISP AS number (must differ from border router AS)
ISP_AS=65001

# Mock ISP's BGP IP (must match ISP_IP in border router .env)
ISP_IP=203.0.113.1

# Border router's BGP IP
BORDER_ROUTER_IP=203.0.113.2

# Border router's AS number
BORDER_ROUTER_AS=65000

# Your mesh range (accepted from the border router)
MESH_ADDRESS_RANGE=44.30.127.0/24

# Test prefixes announced by the mock ISP
TEST_PREFIX_1=192.0.2.0/24
TEST_PREFIX_2=198.51.100.0/24
TEST_PREFIX_3=203.0.113.0/24

# BGP timers
BGP_HOLD_TIME=90
BGP_KEEPALIVE_TIME=30
```

The `ISP_IP` and `BORDER_ROUTER_IP` here must match the corresponding values in `deploy/bird-border/.env`.

---

## Deploy

```bash
docker compose up -d
```

The entrypoint generates the BIRD config from the template, validates the syntax, then starts the daemon.

---

## Verify

On the mock ISP host:

```bash
docker exec rpi-isp birdc show protocols
docker exec rpi-isp birdc show route
```

The `border` protocol should show `Established`.

On the border router host:

```bash
docker exec bird-border birdc show protocols
docker exec bird-border birdc show route
```

You should see:
- The `isp` protocol in `Established` state
- The three test prefixes received from the mock ISP
- Your mesh range in the export table

---

## Generated BIRD Configuration

The mock ISP generates a config equivalent to:

```
router id <ISP_IP>;

protocol device {}

protocol static {
  route <TEST_PREFIX_1> blackhole;
  route <TEST_PREFIX_2> blackhole;
  route <TEST_PREFIX_3> blackhole;
}

protocol bgp border {
  local <ISP_IP> as <ISP_AS>;
  neighbor <BORDER_ROUTER_IP> as <BORDER_ROUTER_AS>;
  hold time <BGP_HOLD_TIME>;
  keepalive time <BGP_KEEPALIVE_TIME>;

  ipv4 {
    import where net = <MESH_ADDRESS_RANGE>;
    export where source = RTS_STATIC;
  };
}
```

The import filter restricts what the mock ISP accepts from the border router to only your mesh range, simulating a real ISP that only learns your announced prefix.

---

## End-to-End Test Procedure

1. Start the border router (`deploy/bird-border/`)
2. Start the mock ISP (`deploy/rpi-isp/`)
3. Verify BGP session is established on both sides
4. Deploy a mesh node (`deploy/zerotier/`) and authorize it
5. From the mesh node, test outbound routing:
   ```bash
   curl --interface 44.x.y.z https://ifconfig.me
   ```
6. Verify the mesh prefix is visible in the mock ISP's routing table:
   ```bash
   docker exec rpi-isp birdc "show route for 44.x.y.0/24"
   ```

---

## Container Details

- **Base**: `debian:12-slim` (same as border router)
- **Packages**: `bird2`, `iproute2`, `gettext-base`
- **Network mode**: `host`
- **Capabilities**: `NET_ADMIN`
- **Architectures**: amd64, arm64, arm/v7
