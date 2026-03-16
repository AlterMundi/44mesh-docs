# Troubleshooting

---

## BGP Issues

### BGP Session Won't Establish

**Symptom**: `birdc show protocols` shows the `isp` protocol in `Active` or `Connect` state, never reaching `Established`.

**Diagnostic steps**:

```bash
# Check BIRD status
docker exec bird-border birdc show protocols all isp

# Check if port 179 is reachable from the ISP peer
# (run from the ISP side, or use a port scanner)
nc -zv <BORDER_ROUTER_IP> 179

# Check firewall
iptables -L INPUT -n -v | grep 179
```

**Common causes**:

| Cause | Fix |
|-------|-----|
| Port 179 blocked by firewall | Allow TCP 179 from the ISP peer IP |
| Wrong peer IP in `.env` | Verify `ISP_IP` matches ISP's BGP peer IP |
| Wrong AS number | Verify `ISP_AS` and `BORDER_ROUTER_AS` match ISP configuration |
| `BORDER_ROUTER_IP` not assigned to host interface | Run `ip addr show` and verify the IP exists |
| ISP hasn't configured their end | Contact ISP to verify their BGP configuration |

---

### BGP Session Established but Prefix Not Announced

**Symptom**: Session shows `Established` but the mesh prefix doesn't appear in `show route export isp`.

**Diagnostic steps**:

```bash
docker exec bird-border birdc show route
docker exec bird-border birdc "show route export isp"
docker exec bird-border birdc "show route for <MESH_ADDRESS_RANGE>"
```

**Common causes**:

| Cause | Fix |
|-------|-----|
| Static route for mesh range missing | Check that `MESH_ADDRESS_RANGE` is set correctly in `.env` |
| Export filter mismatch | Verify the BIRD config: `export where source = RTS_STATIC` |
| BIRD config not regenerated | Restart the container to regenerate config from template |

---

### BGP Flapping

**Symptom**: BGP session repeatedly connects and disconnects.

**Diagnostic steps**:

```bash
docker logs bird-border | grep -i bgp
docker exec bird-border birdc show protocols all isp
```

**Common causes**:

| Cause | Fix |
|-------|-----|
| Hold timer too short | Increase `BGP_HOLD_TIME` (recommended: 90s minimum) |
| Network instability | Check the link between border router and ISP |
| Container restarting | Check `docker compose ps` for restart counts |

---

## ZeroTier Issues

### Node Not Joining the Network

**Symptom**: `zerotier-cli listnetworks` shows `REQUESTING_CONFIGURATION` or nothing.

**Diagnostic steps**:

```bash
docker exec zerotier zerotier-cli info
docker exec zerotier zerotier-cli listnetworks
docker logs zerotier
```

**Common causes**:

| Cause | Fix |
|-------|-----|
| Wrong network ID | Verify `ZT_NETWORK_ID` in `.env` is 16 hex characters |
| Port 9993/UDP blocked | Check firewall; ZeroTier needs outbound UDP 9993 |
| Controller unreachable | Verify the border router is running and `zerotier-cli info` responds |

---

### Node Joined but Not Authorized

**Symptom**: `zerotier-cli listnetworks` shows `ACCESS_DENIED`.

**Fix**: Authorize the node via the UI or API:

```bash
TOKEN=$(docker exec zerotier cat /var/lib/zerotier-one/authtoken.secret)
curl -X POST \
  -H "X-ZT1-Auth: ${TOKEN}" \
  -d '{"authorized": true}' \
  "http://localhost:9993/controller/network/<network-id>/member/<node-id>"
```

---

### Node Has No IP Assignment

**Symptom**: Node shows `OK` in `listnetworks` but no IP is assigned.

**Diagnostic steps**:

```bash
# Check the network's IP pool configuration
TOKEN=$(docker exec zerotier cat /var/lib/zerotier-one/authtoken.secret)
curl -H "X-ZT1-Auth: ${TOKEN}" \
  "http://localhost:9993/controller/network/<network-id>" | python3 -m json.tool
```

**Common causes**:

| Cause | Fix |
|-------|-----|
| IP pool not configured | Set `ipAssignmentPools` in the network config |
| IP pool exhausted | Expand the pool or deauthorize unused members |
| `v4AssignMode` not set to `zt` | Enable ZeroTier-managed IP assignment |

---

### Source Routing Not Installed

**Symptom**: Node has an IP but outbound traffic from the public IP doesn't use the border router.

**Diagnostic steps**:

```bash
# Check if routing rules exist
ip rule show

# Check if the per-node routing table exists
ip route show table <table-number>

# Test which path traffic takes
ip route get 8.8.8.8 from <mesh-node-ip>
```

**Common causes**:

| Cause | Fix |
|-------|-----|
| Node is not running the AlterMundi fork | Check the image source; must be `buzondefede/44mesh-zerotier` |
| `allowDefault=0` on the node | Check `network.local.conf`; must be `allowDefault=1` |
| `ingressNodeV4` not set on the network | Set via API or ZeroTier UI |

---

## Connectivity Issues

### Node Not Reachable from Internet

**Symptom**: Cannot reach a service on the mesh node from outside.

**Diagnostic steps**:

```bash
# Verify the node has a public IP
docker exec zerotier zerotier-cli listnetworks

# Verify BGP is announcing the prefix
docker exec bird-border birdc "show route export isp"

# Verify the service is listening on the right IP/port
ss -tlnp | grep <port>

# Check routing from the border router to the node
ip route get <mesh-node-ip>
```

**Common causes**:

| Cause | Fix |
|-------|-----|
| BGP not established | Fix BGP session first |
| Service not listening on public IP | Bind service to `0.0.0.0` or the specific mesh IP |
| Firewall on the node | Allow the service port in the node's firewall |
| No route from border router to node | Verify ZeroTier overlay is connected |

---

### No Internet Access from Mesh Node

**Symptom**: Node can reach other mesh nodes but not the Internet.

**Diagnostic steps**:

```bash
# Test from the public IP
curl --interface <mesh-ip> https://ifconfig.me

# Test from the node's native IP (should work if Internet is available)
curl https://ifconfig.me

# Check source routing
ip rule show
ip route show table <table-number>
```

**Common causes**:

| Cause | Fix |
|-------|-----|
| `ingressNodeV4` not set | Set the ingress node in network config |
| Border router not forwarding | Verify `net.ipv4.ip_forward=1` on the border router host |
| Source routing on border router missing | Restart the border router container |
| BGP not receiving default route | Verify ISP is advertising a default route |

---

## ZeroTier UI Issues

### UI Can't Connect to Controller

**Symptom**: UI shows connection error or blank network list.

**Diagnostic steps**:

```bash
docker logs zerotier-ui

# Check the auth token
docker exec zerotier-ui cat /var/lib/zerotier-one/authtoken.secret
```

**Common causes**:

| Cause | Fix |
|-------|-----|
| `zerotier_data` volume not shared | Both services must reference the same named volume |
| Border router container not running | Start bird-border first |
| Auth token not yet available | Wait for zerotier container to fully start |

---

## Useful Diagnostic Commands Summary

```bash
# BGP
docker exec bird-border birdc show protocols
docker exec bird-border birdc show route
docker exec bird-border birdc "show route export isp"

# ZeroTier (node)
docker exec zerotier zerotier-cli info
docker exec zerotier zerotier-cli listnetworks
docker exec zerotier zerotier-cli listpeers

# Controller API
TOKEN=$(docker exec zerotier cat /var/lib/zerotier-one/authtoken.secret)
curl -H "X-ZT1-Auth: $TOKEN" http://localhost:9993/controller/network

# Source routing
ip rule show
ip route show table 123
ip route get 8.8.8.8 from <mesh-ip>

# Container health
docker compose ps
docker compose logs -f
```
