<!--
SPDX-FileCopyrightText: 2024 AlterMundi <docs@44mesh.net>

SPDX-License-Identifier: CC-BY-SA-4.0
-->

# Agent Discovery

Discovery is how AI agents find out which nodes exist on the network, what capabilities they offer, and how to query them.

---

## Discovery Approaches

### 1. Known Address Range

If an agent knows the mesh's IP block (e.g., `44.30.127.0/24`), it can scan the range to find active nodes:

```python
import ipaddress
import requests
from concurrent.futures import ThreadPoolExecutor

def probe_node(ip):
    try:
        r = requests.get(f"http://{ip}/capabilities", timeout=2)
        if r.status_code == 200:
            return {"ip": ip, "capabilities": r.json()}
    except Exception:
        pass
    return None

network = ipaddress.ip_network("44.30.127.0/24")
hosts = [str(h) for h in network.hosts()]

with ThreadPoolExecutor(max_workers=50) as executor:
    results = list(executor.map(probe_node, hosts))

active_nodes = [r for r in results if r is not None]
```

This gives you a list of nodes that respond to the capabilities endpoint.

### 2. Controller API

The ZeroTier controller knows about all authorized members. Query it to get the current member list:

```bash
TOKEN=<controller-auth-token>
NETWORK_ID=<network-id>

curl -H "X-ZT1-Auth: ${TOKEN}" \
  "http://<border-router-host>:9993/controller/network/${NETWORK_ID}/member"
```

Response includes each member's assigned IPs. An agent with access to the controller API can enumerate all nodes without scanning.

### 3. Registry (Planned)

A future network-wide registry will allow agents to register themselves and their capabilities, and query the registry to find other agents and nodes. This will remove the need for scanning or controller API access.

---

## Capabilities Endpoint

Each node that wants to participate as a sensor or agent endpoint should expose a `/capabilities` endpoint:

```
GET http://<node-ip>/capabilities
```

Response:

```json
{
  "node_id": "a1b2c3d4e5",
  "node_ip": "44.30.127.42",
  "location": {
    "lat": -34.6037,
    "lon": -58.3816,
    "description": "Buenos Aires, AR"
  },
  "capabilities": [
    {
      "type": "temperature_sensor",
      "unit": "celsius",
      "update_interval_seconds": 60,
      "history_available": true
    },
    {
      "type": "camera",
      "resolution": "1920x1080",
      "fov_degrees": 120,
      "capture_on_demand": true
    }
  ],
  "api_version": "1",
  "observations_endpoint": "/observations",
  "actions_endpoint": "/actions"
}
```

---

## Discovery Workflow

A typical agent discovery workflow:

```python
import requests

# Step 1: Get list of node IPs (from scan or controller)
node_ips = ["44.30.127.10", "44.30.127.20", "44.30.127.30"]

# Step 2: Query capabilities from each node
nodes = []
for ip in node_ips:
    try:
        r = requests.get(f"http://{ip}/capabilities", timeout=5)
        if r.status_code == 200:
            nodes.append(r.json())
    except Exception as e:
        print(f"Node {ip} unreachable: {e}")

# Step 3: Filter by needed capability
temperature_nodes = [
    n for n in nodes
    if any(c["type"] == "temperature_sensor" for c in n["capabilities"])
]

print(f"Found {len(temperature_nodes)} temperature sensor nodes")
for node in temperature_nodes:
    print(f"  {node['node_ip']} at {node['location']['description']}")
```

---

## Capability Types

Standard capability types (extensible):

| Type | Description |
|------|-------------|
| `temperature_sensor` | Ambient temperature measurement |
| `humidity_sensor` | Relative humidity measurement |
| `pressure_sensor` | Atmospheric pressure |
| `radiation_sensor` | Ionizing radiation (Geiger counter) |
| `sdr_receiver` | Software-defined radio receiver |
| `camera` | Still or video camera |
| `microphone` | Audio capture |
| `gps` | GPS position data |
| `wind_sensor` | Wind speed and direction |
| `compute` | General-purpose compute availability |

Nodes can define custom capability types using a namespaced string (`org.myproject.custom_sensor`).

---

## Caching Discovery Results

Node capabilities don't change often. Cache them with a reasonable TTL:

```python
from datetime import datetime, timedelta

class NodeRegistry:
    def __init__(self, ttl_minutes=10):
        self.nodes = {}
        self.ttl = timedelta(minutes=ttl_minutes)

    def get(self, ip):
        entry = self.nodes.get(ip)
        if entry and datetime.now() - entry["fetched_at"] < self.ttl:
            return entry["data"]
        return None

    def set(self, ip, data):
        self.nodes[ip] = {"data": data, "fetched_at": datetime.now()}
```

---

## Agent Self-Registration

Agents that want to be discoverable by other agents should also expose a `/capabilities` endpoint describing their role:

```json
{
  "node_id": "agent-anomaly-detector-001",
  "node_ip": "44.30.127.99",
  "type": "agent",
  "role": "anomaly_detector",
  "monitors": ["temperature_sensor", "radiation_sensor"],
  "alert_endpoint": "/alerts",
  "status_endpoint": "/status"
}
```

This allows coordination between agents without central orchestration.
