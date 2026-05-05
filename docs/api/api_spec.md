# AEGIS-MESH API Specification

**Base URL:** `http://aegis-mesh.local` (local network) or `https://<configured-address>` (if remote access enabled)
**Protocol:** REST/JSON for configuration and status; WebSocket for real-time streams
**Authentication:** Configurable; default is local-network-only (no auth). HMAC tokens when remote access is enabled.

---

## Node Management

### `GET /api/nodes`

Returns all registered nodes with health status.

**Response:**
```json
{
  "nodes": [
    {
      "id": "ceiling_living_room_north",
      "class": "LidarNode",
      "position_m": [2.0, 7.5, 2.8],
      "health": "Online",
      "sensors": ["SolidStateLidar", "MmWaveRadar", "Acoustic"],
      "battery_percent": null,
      "last_seen_ms": 42
    }
  ]
}
```

### `GET /api/nodes/{id}/health`

Detailed health for one node.

### `GET /api/nodes/{id}/coverage`

Coverage metrics for one node: observable voxel count, coverage fraction of assigned zone.

---

## Zone Management

### `GET /api/zones`

All defined zones with bounds and assigned nodes.

### `GET /api/zones/{id}/tracks`

Active tracks within a zone with positions and classifications.

---

## Track Management

### `GET /api/tracks`

All currently active tracks in the home frame.

**Response:**
```json
{
  "tracks": [
    {
      "id": "trk_0042",
      "class": "HumanWalking",
      "position_m": [3.2, 4.1, 0.9],
      "velocity_ms": [0.8, 1.2, 0.0],
      "confidence": 0.91,
      "timestamp_ns": 1706123456789
    }
  ]
}
```

### `GET /api/tracks/stream`

WebSocket endpoint. Emits `TrackUpdate` events in real time as tracks are created, updated, and removed.

---

## Policy Management

### `GET /api/policy`

Current policy mode and active rules.

### `POST /api/policy/mode`

Set policy mode.

**Request:**
```json
{
  "mode": "SecurityFirst",
  "triggers": [
    {"type": "TrackClass", "class": "UnknownPerson", "zone": "driveway"},
    {"type": "MotionInZone", "zone": "restricted_room"}
  ]
}
```

**Modes:** `PrivacyFirst` | `SecurityFirst` | `Custom`

---

## Calibration

### `GET /api/calibration/status`

Current calibration state for all nodes.

**Response:**
```json
{
  "nodes": [
    {
      "id": "ceiling_living_room_north",
      "calibration_state": "Calibrated",
      "confidence": 0.93,
      "last_calibrated": "2024-01-25T14:23:00Z"
    },
    {
      "id": "choke_front_door",
      "calibration_state": "Uncalibrated",
      "confidence": null
    }
  ]
}
```

### `POST /api/calibration/start`

Begin a walk-through calibration session. Returns session ID.

### `GET /api/calibration/{session_id}/status`

Progress of a calibration session.

### `POST /api/calibration/remap`

Trigger immediate dynamic remap (compare current geometry to baseline).

---

## Identity Layer

### `GET /api/identity/status`

Status of all identity nodes (kill-switch state, activation state).

### `POST /api/identity/enable`

Enable an identity node (requires kill-switch to be in enabled position — verified by hardware GPIO read).

**Request:** `{"node_id": "identity_front_door"}`

### `POST /api/identity/disable`

Disable an identity node (puts it in dormant state).

---

## Audit

### `GET /api/audit`

Recent audit log entries (paginated).

**Query params:** `?since=<timestamp>&limit=<n>`

### `GET /api/audit/stream`

WebSocket endpoint. Emits audit log entries in real time as they are written.

---

## Coverage

### `GET /api/coverage/map`

Current coverage map as a voxel grid. Returns compressed binary format (content-type: application/octet-stream).

### `GET /api/coverage/blind-spots`

List of currently unobserved voxels that are within the "important" region set.

### `GET /api/coverage/placement-report`

Current placement quality report as HTML.
