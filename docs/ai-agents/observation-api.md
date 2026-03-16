# Observation API

The observation API is the standard interface through which AI agents query sensor data from mesh nodes. This page defines the endpoints, parameters, and response formats.

---

## Base URL

All endpoints are relative to the node's public IP:

```
http://<node-public-ip>/
```

No authentication is required by default (nodes are publicly accessible). Node operators may add authentication for sensitive data.

---

## Endpoints

### GET /capabilities

Returns the node's identity and the list of capabilities it exposes.

**Request**:
```
GET http://44.30.127.42/capabilities
```

**Response** `200 OK`:
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
    }
  ],
  "api_version": "1",
  "observations_endpoint": "/observations"
}
```

---

### GET /observations

Returns recent or historical observations from this node.

**Request**:
```
GET http://44.30.127.42/observations
```

**Query Parameters**:

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `sensor_type` | string | all | Filter by sensor type |
| `since` | ISO 8601 | last 1 hour | Return observations after this time |
| `until` | ISO 8601 | now | Return observations before this time |
| `limit` | integer | 100 | Maximum number of results |
| `offset` | integer | 0 | Pagination offset |

**Example**:
```
GET /observations?sensor_type=temperature_sensor&since=2024-03-15T00:00:00Z&limit=50
```

**Response** `200 OK`:
```json
{
  "node_id": "a1b2c3d4e5",
  "node_ip": "44.30.127.42",
  "count": 2,
  "observations": [
    {
      "id": "obs_001",
      "timestamp": "2024-03-15T14:30:00Z",
      "sensor_type": "temperature_sensor",
      "value": 23.4,
      "unit": "celsius",
      "confidence": 0.97,
      "location": {"lat": -34.6037, "lon": -58.3816}
    }
  ],
  "pagination": {
    "limit": 50,
    "offset": 0,
    "total": 1440
  }
}
```

---

### GET /observations/latest

Returns the most recent observation for each sensor type.

**Request**:
```
GET http://44.30.127.42/observations/latest
```

**Response** `200 OK`:
```json
{
  "node_id": "a1b2c3d4e5",
  "node_ip": "44.30.127.42",
  "latest": {
    "temperature_sensor": {
      "id": "obs_001",
      "timestamp": "2024-03-15T14:30:00Z",
      "value": 23.4,
      "unit": "celsius",
      "confidence": 0.97
    },
    "humidity_sensor": {
      "id": "obs_002",
      "timestamp": "2024-03-15T14:30:00Z",
      "value": 65.2,
      "unit": "percent_rh",
      "confidence": 0.95
    }
  }
}
```

---

### GET /observations/{id}

Returns a specific observation by ID.

**Request**:
```
GET http://44.30.127.42/observations/obs_001
```

**Response** `200 OK`: Single observation object (see [Data Model](data-model.md)).

---

### POST /actions (Optional)

Triggers an action on the node. This endpoint is optional and node-specific.

**Request**:
```json
{
  "action": "capture_image",
  "parameters": {
    "resolution": "1920x1080"
  }
}
```

**Response** `202 Accepted`:
```json
{
  "action_id": "act_xyz789",
  "status": "pending",
  "estimated_completion_seconds": 5
}
```

---

## Implementation Reference

Here is a minimal Python implementation of the observation API using Flask:

```python
from flask import Flask, jsonify, request
from datetime import datetime, timezone
import uuid

app = Flask(__name__)

# In a real implementation, these come from actual sensors
observations_store = []

NODE_INFO = {
    "node_id": "a1b2c3d4e5",
    "node_ip": "44.30.127.42",
    "location": {"lat": -34.6037, "lon": -58.3816, "description": "Buenos Aires, AR"},
    "capabilities": [
        {
            "type": "temperature_sensor",
            "unit": "celsius",
            "update_interval_seconds": 60,
            "history_available": True
        }
    ],
    "api_version": "1",
    "observations_endpoint": "/observations"
}

@app.route("/capabilities")
def capabilities():
    return jsonify(NODE_INFO)

@app.route("/observations")
def observations():
    sensor_type = request.args.get("sensor_type")
    limit = int(request.args.get("limit", 100))
    offset = int(request.args.get("offset", 0))

    filtered = observations_store
    if sensor_type:
        filtered = [o for o in filtered if o["sensor_type"] == sensor_type]

    page = filtered[offset:offset + limit]

    return jsonify({
        "node_id": NODE_INFO["node_id"],
        "node_ip": NODE_INFO["node_ip"],
        "count": len(page),
        "observations": page,
        "pagination": {"limit": limit, "offset": offset, "total": len(filtered)}
    })

@app.route("/observations/latest")
def observations_latest():
    latest = {}
    for obs in reversed(observations_store):
        if obs["sensor_type"] not in latest:
            latest[obs["sensor_type"]] = obs

    return jsonify({
        "node_id": NODE_INFO["node_id"],
        "node_ip": NODE_INFO["node_ip"],
        "latest": latest
    })

def record_observation(sensor_type, value, unit, confidence=1.0):
    obs = {
        "id": f"obs_{uuid.uuid4().hex[:8]}",
        "timestamp": datetime.now(timezone.utc).isoformat(),
        "sensor_type": sensor_type,
        "value": value,
        "unit": unit,
        "confidence": confidence,
        "location": NODE_INFO["location"]
    }
    observations_store.append(obs)
    return obs

if __name__ == "__main__":
    # Seed with a sample reading
    record_observation("temperature_sensor", 23.4, "celsius", 0.97)
    app.run(host="0.0.0.0", port=8080)
```

Run it on a mesh node and it becomes immediately queryable from any AI agent at `http://44.x.y.z:8080/`.

---

## Error Responses

| Code | Meaning |
|------|---------|
| `200` | Success |
| `400` | Bad request (invalid parameters) |
| `404` | Not found (observation ID doesn't exist) |
| `429` | Rate limited |
| `500` | Internal server error |

Error response body:
```json
{
  "error": "invalid_parameter",
  "message": "sensor_type must be one of: temperature_sensor, humidity_sensor",
  "timestamp": "2024-03-15T14:30:00Z"
}
```
