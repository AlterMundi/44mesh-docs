---
title: Example Agent
description: Complete Python anomaly detection agent that discovers nodes, collects temperature data, and detects statistical outliers.
topics: [example-agent, python, anomaly-detection, node-discovery, data-collection]
related:
  - docs/ai-agents/overview.md
  - docs/ai-agents/data-model.md
  - docs/ai-agents/observation-api.md
  - docs/ai-agents/agent-discovery.md
---

# Example Agent

This page walks through a complete example of an AI agent that discovers nodes on a 44Mesh network, queries sensor data, and detects anomalies.

---

## What the Agent Does

1. Discovers all active nodes in the mesh by querying a known IP range
2. Identifies which nodes have temperature sensors
3. Fetches the latest readings from each sensor node
4. Detects anomalies (readings more than 2 standard deviations from the mean)
5. Prints an alert for any anomalous nodes

---

## Full Example

```python
import requests
import ipaddress
import statistics
from concurrent.futures import ThreadPoolExecutor, as_completed
from datetime import datetime, timezone

# ─── Configuration ────────────────────────────────────────────────────────────

MESH_RANGE = "44.30.127.0/24"   # Your mesh's IP block
TIMEOUT = 5                      # Seconds to wait per node
MAX_WORKERS = 50                 # Concurrent node probes
ANOMALY_THRESHOLD = 2.0          # Standard deviations

# ─── Discovery ────────────────────────────────────────────────────────────────

def get_node_capabilities(ip: str) -> dict | None:
    """Query a node's capabilities. Returns None if unreachable."""
    try:
        r = requests.get(f"http://{ip}/capabilities", timeout=TIMEOUT)
        if r.status_code == 200:
            data = r.json()
            data["_ip"] = ip
            return data
    except Exception:
        pass
    return None


def discover_nodes(mesh_range: str) -> list[dict]:
    """Probe all IPs in the mesh range and return responding nodes."""
    network = ipaddress.ip_network(mesh_range)
    hosts = [str(h) for h in network.hosts()]

    print(f"Probing {len(hosts)} addresses in {mesh_range}...")

    nodes = []
    with ThreadPoolExecutor(max_workers=MAX_WORKERS) as executor:
        futures = {executor.submit(get_node_capabilities, ip): ip for ip in hosts}
        for future in as_completed(futures):
            result = future.result()
            if result:
                nodes.append(result)

    print(f"Found {len(nodes)} active nodes")
    return nodes


def filter_by_capability(nodes: list[dict], capability_type: str) -> list[dict]:
    """Return nodes that have the given capability."""
    return [
        n for n in nodes
        if any(c["type"] == capability_type for c in n.get("capabilities", []))
    ]

# ─── Data Collection ──────────────────────────────────────────────────────────

def get_latest_reading(node: dict, sensor_type: str) -> dict | None:
    """Get the most recent observation of a given sensor type from a node."""
    ip = node["_ip"]
    try:
        r = requests.get(
            f"http://{ip}/observations/latest",
            timeout=TIMEOUT
        )
        if r.status_code == 200:
            return r.json().get("latest", {}).get(sensor_type)
    except Exception:
        pass
    return None


def collect_readings(nodes: list[dict], sensor_type: str) -> list[dict]:
    """Collect latest readings from all nodes with the given sensor type."""
    readings = []

    def fetch(node):
        reading = get_latest_reading(node, sensor_type)
        if reading:
            return {
                "node_ip": node["_ip"],
                "location": node.get("location", {}),
                "timestamp": reading["timestamp"],
                "value": reading["value"],
                "unit": reading["unit"],
                "confidence": reading.get("confidence", 1.0)
            }
        return None

    with ThreadPoolExecutor(max_workers=MAX_WORKERS) as executor:
        results = list(executor.map(fetch, nodes))

    return [r for r in results if r is not None]

# ─── Anomaly Detection ────────────────────────────────────────────────────────

def detect_anomalies(readings: list[dict], threshold: float = ANOMALY_THRESHOLD) -> list[dict]:
    """
    Flag readings that deviate more than `threshold` standard deviations
    from the mean.
    """
    if len(readings) < 3:
        print("Not enough readings for anomaly detection (need at least 3)")
        return []

    values = [r["value"] for r in readings]
    mean = statistics.mean(values)
    stdev = statistics.stdev(values)

    if stdev == 0:
        return []  # All readings are identical

    anomalies = []
    for r in readings:
        z_score = abs(r["value"] - mean) / stdev
        if z_score > threshold:
            anomalies.append({**r, "z_score": round(z_score, 2), "mean": mean, "stdev": stdev})

    return anomalies

# ─── Main ─────────────────────────────────────────────────────────────────────

def main():
    print(f"44Mesh Anomaly Detector — {datetime.now(timezone.utc).isoformat()}")
    print("=" * 60)

    # Step 1: Discover active nodes
    all_nodes = discover_nodes(MESH_RANGE)

    # Step 2: Filter to temperature sensor nodes
    temp_nodes = filter_by_capability(all_nodes, "temperature_sensor")
    print(f"Nodes with temperature sensors: {len(temp_nodes)}")

    if not temp_nodes:
        print("No temperature sensor nodes found. Exiting.")
        return

    # Step 3: Collect readings
    print("Collecting temperature readings...")
    readings = collect_readings(temp_nodes, "temperature_sensor")
    print(f"Collected {len(readings)} readings")

    # Print all readings
    print("\nCurrent readings:")
    for r in sorted(readings, key=lambda x: x["value"], reverse=True):
        loc = r["location"].get("description", r["node_ip"])
        print(f"  {r['value']:6.1f} {r['unit']}  —  {loc}  ({r['node_ip']})")

    # Step 4: Detect anomalies
    anomalies = detect_anomalies(readings)

    if not anomalies:
        print("\nNo anomalies detected.")
    else:
        print(f"\n⚠ {len(anomalies)} anomaly(ies) detected:")
        for a in anomalies:
            loc = a["location"].get("description", a["node_ip"])
            print(
                f"  ALERT: {a['node_ip']} ({loc})\n"
                f"    Value: {a['value']} {a['unit']}\n"
                f"    Mean: {a['mean']:.1f}, StdDev: {a['stdev']:.1f}, Z-score: {a['z_score']}\n"
                f"    Timestamp: {a['timestamp']}"
            )

if __name__ == "__main__":
    main()
```

---

## Running the Agent

Install dependencies:

```bash
pip install requests
```

Run from any machine with Internet access (nodes are publicly reachable):

```bash
python3 agent.py
```

Example output:

```
44Mesh Anomaly Detector — 2024-03-15T14:30:00Z
============================================================
Probing 254 addresses in 44.30.127.0/24...
Found 12 active nodes
Nodes with temperature sensors: 8
Collecting temperature readings...
Collected 8 readings

Current readings:
   42.3 celsius  —  Patagonia, AR (44.30.127.18)
   25.1 celsius  —  Buenos Aires, AR (44.30.127.42)
   24.8 celsius  —  Mendoza, AR (44.30.127.31)
   24.2 celsius  —  Córdoba, AR (44.30.127.27)
   23.9 celsius  —  Rosario, AR (44.30.127.55)
   23.4 celsius  —  Santa Fe, AR (44.30.127.63)
   22.8 celsius  —  Tucumán, AR (44.30.127.71)
   21.1 celsius  —  Bariloche, AR (44.30.127.88)

⚠ 1 anomaly(ies) detected:
  ALERT: 44.30.127.18 (Patagonia, AR)
    Value: 42.3 celsius
    Mean: 25.7, StdDev: 6.2, Z-score: 2.68
    Timestamp: 2024-03-15T14:28:00Z
```

---

## Extending the Agent

### Continuous Monitoring

```python
import time

while True:
    all_nodes = discover_nodes(MESH_RANGE)
    temp_nodes = filter_by_capability(all_nodes, "temperature_sensor")
    readings = collect_readings(temp_nodes, "temperature_sensor")
    anomalies = detect_anomalies(readings)

    if anomalies:
        # Send to alerting system, write to database, notify Slack, etc.
        handle_anomalies(anomalies)

    time.sleep(300)  # Check every 5 minutes
```

### Multi-Sensor Analysis

The same pattern works for any sensor type. Combine data from multiple sensors for richer analysis:

```python
humidity_nodes = filter_by_capability(all_nodes, "humidity_sensor")
humidity_readings = collect_readings(humidity_nodes, "humidity_sensor")

# Correlate temperature and humidity by location
# (match by node_ip or coordinates)
```

### Storing to a Time Series Database

```python
from influxdb_client import InfluxDBClient, Point

client = InfluxDBClient(url="http://influxdb:8086", token="...", org="44mesh")
write_api = client.write_api()

for r in readings:
    point = (
        Point("sensor_observation")
        .tag("node_ip", r["node_ip"])
        .tag("sensor_type", "temperature_sensor")
        .tag("location", r["location"].get("description", "unknown"))
        .field("value", r["value"])
        .field("confidence", r["confidence"])
    )
    write_api.write(bucket="mesh-observations", record=point)
```
