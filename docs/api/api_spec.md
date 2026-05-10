# AEGIS-MESH API Specification

**Base URL:** `http://aegis-mesh.local` (local network) or `https://<configured-address>` (if remote access enabled)
**Protocol:** REST/JSON for configuration and status; WebSocket for real-time streams; RTSP/WebSocket for media
**Authentication:** Bearer token (user-configured). Required for all endpoints. TLS recommended for remote access.

---

## Architecture Overview

The AEGIS-MESH API is served by the **Edge Controller** — the sole external network gateway for the entire sensing mesh. All sensor nodes communicate via mesh radio to the edge controller, which aggregates data, runs fusion algorithms, and serves the companion app.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    AEGIS-MESH API Architecture                              │
│                                                                              │
│  Always-On Nodes ──┐                                                         │
│  LiDAR Nodes───────┤── Mesh Radio ──► Edge Controller ──► HTTP/WS API      │
│  Identity Nodes────┘  (BLE/Thread/PoE)       │                               │
│                                              ├──► WiFi ──► Companion App   │
│                                              ├──► Cellular ──► Remote App │
│                                              └──► BLE ──► Local App       │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Authentication

All endpoints require bearer token authentication:

```
Authorization: Bearer <user_configured_token>
```

**Token configuration (`aegis-mesh.toml`):**

```toml
[companion_app]
enabled = true
token = "your_secure_token_here"
enable_remote_access = false
remote_access_token = "separate_token_for_remote"
```

---

## Node Management

### `GET /api/nodes`

Returns all registered nodes with health status, capabilities, and configuration.

**Response:**

```json
{
  "nodes": [
    {
      "id": "ceiling_living_room_north",
      "class": "LidarNode",
      "variant": "B",
      "position_m": [2.0, 7.5, 2.8],
      "zone_id": "living_room",
      "health": {
        "status": "Online",
        "last_seen_ms": 42,
        "uptime_hours": 168.5
      },
      "sensors": {
        "primary": ["SolidStateLidar", "MmWaveRadar"],
        "optional": ["Acoustic"],
        "camera": false
      },
      "connectivity": {
        "transport": "poe",
        "latency_ms": 12,
        "signal_strength_dbm": null
      },
      "storage": {
        "type": "none",
        "capacity_mb": 0,
        "used_mb": 0
      },
      "battery_percent": null
    },
    {
      "id": "identity_front_door",
      "class": "IdentityNode",
      "variant": "C",
      "position_m": [0.0, 3.2, 1.5],
      "zone_id": "front_entry",
      "health": {
        "status": "Online",
        "last_seen_ms": 15,
        "uptime_hours": 72.3
      },
      "sensors": {
        "primary": ["IdentificationSensor"],
        "camera": true
      },
      "connectivity": {
        "transport": "ble",
        "latency_ms": 35,
        "signal_strength_dbm": -62
      },
      "storage": {
        "type": "sd_card",
        "capacity_mb": 64000,
        "used_mb": 2340
      },
      "battery_percent": null,
      "privacy": {
        "hardware_switch_installed": true,
        "hardware_switch_state": "enabled",
        "software_state": "enabled",
        "data_handling_mode": "local_storage"
      }
    }
  ],
  "total_count": 8,
  "online_count": 8
}
```

### `GET /api/nodes/{id}`

Detailed information for a single node.

**Response:**

```json
{
  "id": "identity_front_door",
  "class": "IdentityNode",
  "variant": "C",
  "position_m": [0.0, 3.2, 1.5],
  "zone_id": "front_entry",
  "health": {
    "status": "Online",
    "last_seen_ms": 15,
    "uptime_hours": 72.3,
    "temperature_c": 38.2,
    "cpu_load_percent": 12,
    "memory_used_mb": 45,
    "memory_total_mb": 256
  },
  "sensors": {
    "primary": ["IdentificationSensor"],
    "camera": true,
    "camera_resolution": "1080p",
    "camera_fov_deg": 90
  },
  "connectivity": {
    "transport": "ble",
    "protocol": "mesh",
    "latency_ms": 35,
    "signal_strength_dbm": -62,
    "packets_sent": 15234,
    "packets_received": 15230,
    "packet_loss_percent": 0.03
  },
  "storage": {
    "type": "sd_card",
    "capacity_mb": 64000,
    "used_mb": 2340,
    "available_mb": 61660,
    "write_speed_mbps": 12.5
  },
  "privacy": {
    "hardware_switch_installed": true,
    "hardware_switch_state": "enabled",
    "software_state": "enabled",
    "data_handling_mode": "local_storage"
  },
  "firmware": {
    "version": "v1.2.0",
    "build_hash": "a3f4b2c9",
    "last_updated": "2024-01-15T10:00:00Z"
  },
  "calibration": {
    "state": "Calibrated",
    "confidence": 0.95,
    "last_calibrated": "2024-01-20T14:30:00Z"
  }
}
```

### `GET /api/nodes/{id}/health`

Health status summary for a single node.

**Response:**

```json
{
  "node_id": "identity_front_door",
  "status": "Online",
  "last_seen_ms": 15,
  "uptime_hours": 72.3,
  "health_checks": [
    {"name": "SensorEnumeration", "status": "Pass"},
    {"name": "MeshConnectivity", "status": "Pass"},
    {"name": "StorageWrite", "status": "Pass"},
    {"name": "CameraInit", "status": "Pass"}
  ]
}
```

### `GET /api/nodes/{id}/coverage`

Coverage metrics for a node within its assigned zone.

**Response:**

```json
{
  "node_id": "ceiling_living_room_north",
  "zone_id": "living_room",
  "coverage": {
    "observable_voxels": 12450,
    "total_zone_voxels": 15600,
    "coverage_fraction": 0.80,
    "average_los_quality": 0.85
  },
  "occlusions": [
    {
      "region": "behind_couch",
      "voxels_occluded": 850,
      "occlusion_type": "furniture"
    }
  ]
}
```

### `GET /api/nodes/{id}/stream`

Open a live media stream from a camera-capable node (Identity nodes, LiDAR Variant C/D, Always-On Variant D).

**Response:** Redirects to RTSP stream or opens WebSocket stream.

**Query parameters:**
- `format`: `rtsp` | `websocket` (default: `rtsp`)
- `quality`: `low` | `medium` | `high` | `raw`

**RTSP URL:** `rtsp://aegis-mesh.local:9090/{node_id}`

**WebSocket:** `ws://aegis-mesh.local:9091/{node_id}`

### `GET /api/nodes/{id}/panorama`

Open a 360° panoramic stream from a LiDAR Variant D node with 360° camera array.

**Response:** Redirects to 360° panorama stream.

**Query parameters:**
- `format`: `equirectangular` | `spherical` | `cubemap`
- `resolution`: `2k` | `4k`
- `quality`: `medium` | `high`

---

## Zone Management

### `GET /api/zones`

All defined zones with bounds, assigned nodes, and activity summary.

**Response:**

```json
{
  "zones": [
    {
      "id": "living_room",
      "name": "Living Room",
      "bounds_m": {
        "min": [0.0, 0.0, 0.0],
        "max": [5.0, 6.0, 2.7]
      },
      "assigned_nodes": ["ceiling_living_room_north", "ceiling_living_room_south"],
      "current_occupancy": {
        "count": 1,
        "entities": ["trk_0042"]
      },
      "coverage_fraction": 0.95
    }
  ],
  "total_count": 6
}
```

### `POST /api/zones`

Create a new zone.

**Request:**

```json
{
  "id": "garage",
  "name": "Garage",
  "bounds_m": {
    "min": [8.0, 0.0, 0.0],
    "max": [12.0, 6.0, 3.0]
  },
  "type": "volume",
  "assigned_nodes": ["lidar_garage_center"],
  "alert_rules": {
    "on_entry": true,
    "on_exit": false,
    "on_motion": true
  }
}
```

### `GET /api/zones/{id}`

Detailed information for a single zone.

### `PUT /api/zones/{id}`

Update zone configuration.

### `DELETE /api/zones/{id}`

Delete a zone.

### `GET /api/zones/{id}/tracks`

Active tracks within a zone.

**Response:**

```json
{
  "zone_id": "living_room",
  "tracks": [
    {
      "track_id": "trk_0042",
      "class": "HumanWalking",
      "position_m": [3.2, 4.1, 0.9],
      "velocity_ms": [0.8, 1.2, 0.0],
      "confidence": 0.91,
      "time_in_zone_ms": 45000,
      "entry_position_m": [1.0, 0.5, 0.9]
    }
  ],
  "occupancy_count": 1,
  "timestamp_ns": 1706123456789
}
```

### `GET /api/zones/{id}/heatmap`

Activity heatmap for a zone over a specified time range.

**Query parameters:**
- `start`: ISO timestamp
- `end`: ISO timestamp
- `resolution`: `hour` | `day` | `week`

**Response:**

```json
{
  "zone_id": "living_room",
  "heatmap": [
    {"time_bucket": "2024-01-25T14:00:00Z", "occupancy_minutes": 45},
    {"time_bucket": "2024-01-25T15:00:00Z", "occupancy_minutes": 120},
    {"time_bucket": "2024-01-25T16:00:00Z", "occupancy_minutes": 90}
  ]
}
```

---

## Track Management

### `GET /api/tracks`

All currently active tracks in the home frame.

**Query parameters:**
- `zone`: Filter by zone ID
- `class`: Filter by track class
- `min_confidence`: Minimum confidence threshold

**Response:**

```json
{
  "tracks": [
    {
      "id": "trk_0042",
      "class": "HumanWalking",
      "classification_source": "radar_micro_doppler",
      "position_m": [3.2, 4.1, 0.9],
      "velocity_ms": [0.8, 1.2, 0.0],
      "prediction_centers": [
        {"time_offset_ms": 100, "position_m": [3.28, 4.22, 0.9]},
        {"time_offset_ms": 200, "position_m": [3.36, 4.34, 0.9]},
        {"time_offset_ms": 300, "position_m": [3.44, 4.46, 0.9]}
      ],
      "confidence": 0.91,
      "anomaly_flags": [],
      "current_zone": "living_room",
      "track_age_ms": 45000,
      "timestamp_ns": 1706123456789
    }
  ],
  "total_count": 2,
  "timestamp_ns": 1706123456789
}
```

### `GET /api/tracks/{id}`

Detailed information for a single track.

**Response:**

```json
{
  "id": "trk_0042",
  "class": "HumanWalking",
  "classification_source": "radar_micro_doppler",
  "position_m": [3.2, 4.1, 0.9],
  "velocity_ms": [0.8, 1.2, 0.0],
  "acceleration_ms2": [0.0, 0.0, 0.0],
  "prediction_centers": [
    {"time_offset_ms": 100, "position_m": [3.28, 4.22, 0.9]},
    {"time_offset_ms": 200, "position_m": [3.36, 4.34, 0.9]},
    {"time_offset_ms": 300, "position_m": [3.44, 4.46, 0.9]},
    {"time_offset_ms": 500, "position_m": [3.60, 4.70, 0.9]}
  ],
  "drift_profile": "human_walking",
  "confidence": 0.91,
  "confidence_history": [
    {"timestamp_ns": 1706123416789, "confidence": 0.75},
    {"timestamp_ns": 1706123426789, "confidence": 0.82},
    {"timestamp_ns": 1706123436789, "confidence": 0.88},
    {"timestamp_ns": 1706123446789, "confidence": 0.91}
  ],
  "anomaly_flags": [],
  "trajectory_history": [
    {"timestamp_ns": 1706123416789, "position_m": [1.0, 0.5, 0.9]},
    {"timestamp_ns": 1706123426789, "position_m": [1.8, 1.5, 0.9]},
    {"timestamp_ns": 1706123436789, "position_m": [2.5, 2.8, 0.9]},
    {"timestamp_ns": 1706123446789, "position_m": [3.2, 4.1, 0.9]}
  ],
  "current_zone": "living_room",
  "zones_visited": ["front_entry", "hallway", "living_room"],
  "track_age_ms": 45000,
  "detection_sources": ["lidar_living_room_north", "lidar_living_room_south"],
  "identity_result": null,
  "timestamp_ns": 1706123456789
}
```

### `GET /api/tracks/stream`

WebSocket endpoint. Emits track events in real time.

**Event types:**
- `TrackCreated`: New track initialized
- `TrackUpdated`: Position/classification update
- `TrackLost`: Track lost or removed
- `TrackAnomaly`: Anomaly flag raised

**WebSocket message format:**

```json
{
  "event_type": "TrackUpdated",
  "timestamp_ns": 1706123456789,
  "track": {
    "id": "trk_0042",
    "class": "HumanWalking",
    "position_m": [3.2, 4.1, 0.9],
    "velocity_ms": [0.8, 1.2, 0.0],
    "confidence": 0.91,
    "current_zone": "living_room"
  }
}
```

---

## Policy Management

### `GET /api/policy`

Current policy mode and active rules.

**Response:**

```json
{
  "current_mode": "SecurityFirst",
  "available_modes": ["PrivacyFirst", "SecurityFirst", "Away", "Silent"],
  "rules": {
    "identity_layer": {
      "activation": "triggered",
      "triggers": [
        {"type": "UnknownPerson", "zones": ["driveway", "front_entry"]},
        {"type": "MotionInZone", "zones": ["restricted_room"]}
      ]
    },
    "recording": {
      "mode": "on_detection",
      "continuous_in_away_mode": true
    },
    "alerts": {
      "haptic_enabled": true,
      "audio_enabled": false,
      "app_notification": true
    }
  },
  "mode_since": "2024-01-25T14:23:00Z",
  "auto_away_after_minutes": 30
}
```

### `POST /api/policy/mode`

Set policy mode.

**Request:**

```json
{
  "mode": "SecurityFirst",
  "triggers": [
    {"type": "UnknownPerson", "zones": ["driveway", "front_entry"]},
    {"type": "MotionInZone", "zones": ["restricted_room"]}
  ]
}
```

**Response:**

```json
{
  "previous_mode": "PrivacyFirst",
  "new_mode": "SecurityFirst",
  "effective_at": "2024-01-25T14:30:00Z",
  "changes_applied": {
    "identity_layer_activated": true,
    "sensitivity_increased": true
  }
}
```

### `PUT /api/policy/rules`

Update specific policy rules without changing mode.

**Request:**

```json
{
  "identity_layer": {
    "triggers": [
      {"type": "UnknownPerson", "zones": ["all"]}
    ]
  },
  "alerts": {
    "haptic_enabled": true,
    "audio_enabled": false
  }
}
```

---

## Connectivity Management

### `GET /api/connectivity`

Current connectivity status for the edge controller.

**Response:**

```json
{
  "wifi": {
    "enabled": true,
    "connected": true,
    "ssid": "HomeNetwork",
    "rssi_dbm": -55,
    "ip_address": "192.168.1.100",
    "mac_address": "AA:BB:CC:DD:EE:FF"
  },
  "cellular": {
    "enabled": true,
    "connected": false,
    "module": "Quectel EC25",
    "sim_present": true,
    "carrier": null,
    "signal_strength_dbm": null,
    "technology": null,
    "ip_address": null,
    "data_used_mb": 0
  },
  "bluetooth": {
    "enabled": true,
    "paired_devices": 0
  },
  "ethernet": {
    "connected": true,
    "ip_address": "192.168.1.101",
    "speed_mbps": 1000
  },
  "mesh_nodes": {
    "total": 8,
    "online": 8,
    "transports": {
      "ble": 5,
      "thread": 0,
      "poe": 3
    }
  }
}
```

### `GET /api/connectivity/wifi`

Detailed WiFi status.

**Response:**

```json
{
  "enabled": true,
  "connected": true,
  "ssid": "HomeNetwork",
  "rssi_dbm": -55,
  "ip_address": "192.168.1.100",
  "subnet_mask": "255.255.255.0",
  "gateway": "192.168.1.1",
  "dns_servers": ["192.168.1.1", "8.8.8.8"],
  "mac_address": "AA:BB:CC:DD:EE:FF",
  "channel": 6,
  "band": "2.4ghz",
  "prefer_5ghz": true
}
```

### `POST /api/connectivity/wifi/configure`

Update WiFi configuration.

**Request:**

```json
{
  "ssid": "NewNetwork",
  "password": "new_password",
  "prefer_5ghz": true
}
```

### `GET /api/connectivity/cellular`

Detailed cellular/SIM status.

**Response:**

```json
{
  "enabled": true,
  "connected": false,
  "module": "Quectel EC25",
  "module_firmware": "EC25EFAR06A03M4G",
  "sim_present": true,
  "sim_type": "nano_sim",
  "carrier": null,
  "signal_strength_dbm": null,
  "signal_quality": null,
  "technology": null,
  "ip_address": null,
  "apn": "carrier.apn",
  "imei": "861234567890123",
  "imsi": null,
  "data_used_mb": 0,
  "data_limit_mb": 1000,
  "registered": false
}
```

### `POST /api/connectivity/cellular/toggle`

Enable or disable cellular module.

**Request:**

```json
{
  "enabled": true
}
```

### `GET /api/connectivity/cellular/data-usage`

Current billing period data usage.

**Response:**

```json
{
  "billing_period_start": "2024-01-01T00:00:00Z",
  "billing_period_end": "2024-01-31T23:59:59Z",
  "data_used_mb": 234,
  "data_limit_mb": 1000,
  "data_remaining_mb": 766,
  "daily_average_mb": 7.8,
  "projected_usage_mb": 241
}
```

### `GET /api/connectivity/mesh`

Mesh network status for all nodes.

**Response:**

```json
{
  "protocol": "ble_thread_hybrid",
  "edge_controller_address": "192.168.1.100:8080",
  "heartbeat_interval_ms": 100,
  "nodes": [
    {
      "node_id": "pendant_entry",
      "transport": "ble",
      "latency_ms": 35,
      "signal_strength_dbm": -62,
      "last_heartbeat_ms": 45
    },
    {
      "node_id": "lidar_living_room_north",
      "transport": "poe",
      "latency_ms": 12,
      "signal_strength_dbm": null,
      "last_heartbeat_ms": 8
    }
  ],
  "time_sync": {
    "method": "ble_connection_events",
    "offset_ns": 125000,
    "drift_ns_per_sec": 3
  }
}
```

---

## Calibration

### `GET /api/calibration/status`

Current calibration state for all nodes.

**Response:**

```json
{
  "session_id": null,
  "overall_status": "calibrated",
  "nodes": [
    {
      "id": "ceiling_living_room_north",
      "calibration_state": "Calibrated",
      "confidence": 0.93,
      "last_calibrated": "2024-01-25T14:23:00Z",
      "position_m": [2.0, 7.5, 2.8],
      "orientation_quat": [0.0, 0.0, 0.0, 1.0]
    },
    {
      "id": "identity_front_door",
      "calibration_state": "Uncalibrated",
      "confidence": null,
      "last_calibrated": null,
      "position_m": null,
      "orientation_quat": null
    }
  ],
  "voxel_map_version": 12,
  "last_remap": "2024-01-26T09:00:00Z"
}
```

### `POST /api/calibration/start`

Begin a walk-through calibration session.

**Request:**

```json
{
  "mode": "walk_through",
  "calibration_duration_seconds": 60
}
```

**Response:**

```json
{
  "session_id": "cal_20240126_103000",
  "status": "collecting",
  "started_at": "2024-01-26T10:30:00Z",
  "expected_completion": "2024-01-26T10:31:00Z",
  "nodes_detected": 3,
  "nodes_remaining": 5
}
```

### `GET /api/calibration/{session_id}/status`

Progress of a calibration session.

**Response:**

```json
{
  "session_id": "cal_20240126_103000",
  "status": "collecting",
  "started_at": "2024-01-26T10:30:00Z",
  "elapsed_seconds": 30,
  "remaining_seconds": 30,
  "nodes": [
    {
      "id": "ceiling_living_room_north",
      "state": "collecting",
      "samples_collected": 152,
      "confidence": 0.75
    },
    {
      "id": "identity_front_door",
      "state": "uncalibrated",
      "samples_collected": 0,
      "confidence": null
    }
  ],
  "recommended_action": "Continue walking through all zones"
}
```

### `POST /api/calibration/{session_id}/complete`

Complete and apply calibration.

**Response:**

```json
{
  "session_id": "cal_20240126_103000",
  "status": "complete",
  "completed_at": "2024-01-26T10:31:00Z",
  "nodes_calibrated": 7,
  "nodes_failed": 1,
  "overall_confidence": 0.89,
  "voxel_map_updated": true,
  "new_blind_spots": [
    {"zone": "garage", "region": "behind_shelves", "voxels": 150}
  ]
}
```

### `POST /api/calibration/remap`

Trigger immediate dynamic remap.

**Response:**

```json
{
  "remap_id": "remap_20240126_103500",
  "status": "complete",
  "scan_duration_ms": 1500,
  "changes_detected": true,
  "changes": [
    {
      "type": "furniture_moved",
      "zone": "living_room",
      "description": "Couch moved 0.5m east",
      "affected_voxels": 450
    }
  ],
  "coverage_impact": {
    "previous_coverage": 0.95,
    "new_coverage": 0.92,
    "new_blind_spots": 2
  },
  "recommendations": [
    "Consider adding choke-point node at hallway entry"
  ]
}
```

### `GET /api/calibration/voxel-map`

Download the current voxel map.

**Query parameters:**
- `format`: `binary` | `json`

**Response (binary):** Compressed voxel grid data

**Response (JSON):**

```json
{
  "version": 12,
  "resolution_m": 0.25,
  "bounds_m": {
    "min": [0.0, 0.0, 0.0],
    "max": [15.0, 12.0, 3.0]
  },
  "voxels": [
    {"position": [2, 3, 0], "class": "open"},
    {"position": [2, 3, 1], "class": "obstruction"},
    {"position": [2, 3, 2], "class": "open"}
  ]
}
```

---

## Identity Layer

### `GET /api/identity/status`

Status of all identity nodes.

**Response:**

```json
{
  "nodes": [
    {
      "id": "identity_front_door",
      "state": "enabled",
      "hardware_switch_installed": true,
      "hardware_switch_state": "enabled",
      "software_state": "enabled",
      "data_handling_mode": "local_storage",
      "last_identification": "2024-01-26T10:00:00Z",
      "storage": {
        "capacity_mb": 64000,
        "used_mb": 2340,
        "recording_count": 156
      }
    },
    {
      "id": "identity_back_door",
      "state": "disabled",
      "hardware_switch_installed": false,
      "hardware_switch_state": null,
      "software_state": "disabled",
      "data_handling_mode": "metadata_only",
      "last_identification": null,
      "storage": {
        "capacity_mb": 0,
        "used_mb": 0,
        "recording_count": 0
      }
    }
  ],
  "total_nodes": 2,
  "enabled_count": 1
}
```

### `GET /api/identity/{id}/status`

Detailed status for a single identity node.

### `POST /api/identity/{id}/enable`

Enable an identity node.

**Precondition:** If hardware switch is installed, it must be in `enabled` position.

**Response:**

```json
{
  "node_id": "identity_front_door",
  "previous_state": "disabled",
  "new_state": "enabled",
  "enabled_at": "2024-01-26T10:30:00Z",
  "data_handling_mode": "local_storage"
}
```

**Error response (hardware switch disabled):**

```json
{
  "error": "HardwareSwitchDisabled",
  "message": "Cannot enable identity node: hardware power switch is in disabled position",
  "node_id": "identity_front_door",
  "hardware_switch_state": "disabled"
}
```

### `POST /api/identity/{id}/disable`

Disable an identity node.

**Response:**

```json
{
  "node_id": "identity_front_door",
  "previous_state": "enabled",
  "new_state": "disabled",
  "disabled_at": "2024-01-26T10:30:00Z"
}
```

### `POST /api/identity/{id}/data-mode`

Set data handling mode for an identity node.

**Request:**

```json
{
  "mode": "local_storage",
  "store_raw": true,
  "retention_days": 30,
  "enable_streaming": false
}
```

**Modes:**
- `metadata_only`: Classification tags only
- `local_storage`: Raw media stored locally
- `app_streaming`: Live stream to companion app
- `full_recording`: Continuous capture with integrity chain

---

## Recordings Management

### `GET /api/recordings`

List all stored recordings.

**Query parameters:**
- `node_id`: Filter by node
- `zone_id`: Filter by zone
- `trigger`: Filter by trigger type
- `start`: ISO timestamp (range start)
- `end`: ISO timestamp (range end)
- `class`: Filter by detection class
- `limit`: Maximum results (default: 100)
- `offset`: Pagination offset

**Response:**

```json
{
  "recordings": [
    {
      "id": "rec_20240126_103000",
      "node_id": "identity_front_door",
      "node_class": "IdentityNode",
      "zone_id": "front_entry",
      "start_timestamp": "2024-01-26T10:30:00.000Z",
      "end_timestamp": "2024-01-26T10:30:42.500Z",
      "duration_s": 42.5,
      "trigger": "UnknownPerson",
      "detection_class": "UnknownPerson",
      "confidence": 0.87,
      "modalities": ["video", "audio"],
      "format": "h264_mp4",
      "file_size_mb": 125,
      "file_path": "/recordings/identity_front_door/2024/01/26/rec_103000.mp4",
      "has_integrity_chain": true,
      "sha256": "a3f4b2c9d8e1f7a2b5c8d3e6f9a0b1c2d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8"
    }
  ],
  "total_count": 156,
  "total_size_mb": 2340,
  "page": {
    "limit": 100,
    "offset": 0,
    "has_more": true
  }
}
```

### `GET /api/recordings/{id}`

Download a recording file.

**Query parameters:**
- `format`: `original` | `mp4` | `webm` (default: `original`)

**Response:** Binary file stream with appropriate content-type.

### `GET /api/recordings/{id}/meta`

Recording metadata and integrity manifest.

**Response:**

```json
{
  "id": "rec_20240126_103000",
  "node_id": "identity_front_door",
  "node_class": "IdentityNode",
  "zone_id": "front_entry",
  "start_timestamp": "2024-01-26T10:30:00.000Z",
  "end_timestamp": "2024-01-26T10:30:42.500Z",
  "duration_s": 42.5,
  "trigger": "UnknownPerson",
  "detection_class": "UnknownPerson",
  "confidence": 0.87,
  "modalities": ["video", "audio"],
  "format": "h264_mp4",
  "resolution": "1080p",
  "frame_rate": 30,
  "file_size_mb": 125,
  "file_path": "/recordings/identity_front_door/2024/01/26/rec_103000.mp4",
  "integrity": {
    "sha256": "a3f4b2c9d8e1f7a2b5c8d3e6f9a0b1c2d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8",
    "computed_at": "2024-01-26T10:30:45.000Z",
    "chain_hash": "b7e8d1f3a2c4e5f6b7a8c9d0e1f2a3b4",
    "algorithm": "SHA-256"
  },
  "firmware_version": "v1.2.0",
  "hardware_variant": "identity_node_v1",
  "chain_of_custody": [
    {
      "event": "created",
      "timestamp": "2024-01-26T10:30:00.000Z",
      "node_id": "identity_front_door"
    },
    {
      "event": "integrity_computed",
      "timestamp": "2024-01-26T10:30:45.000Z",
      "edge_controller": "edge_main"
    }
  ]
}
```

### `DELETE /api/recordings/{id}`

Delete a recording.

**Response:**

```json
{
  "id": "rec_20240126_103000",
  "deleted": true,
  "deleted_at": "2024-01-26T10:35:00Z",
  "storage_freed_mb": 125
}
```

### `GET /api/recordings/{id}/thumbnail`

Get a thumbnail image for a recording.

**Query parameters:**
- `time_s`: Timestamp within recording (default: middle)
- `size`: `small` | `medium` | `large`

**Response:** JPEG image binary.

---

## Legal Export

### `GET /api/export/{recording_id}`

Export a recording with full integrity chain.

**Query parameters:**
- `format`: `zip` | `tar`
- `include_metadata`: `true` | `false`

**Response:** ZIP or TAR archive containing:
- Recording file(s)
- `integrity_manifest.json`
- `chain_of_custody.json`
- `export_report.json`

### `POST /api/export/bulk`

Export multiple recordings in a batch.

**Request:**

```json
{
  "recording_ids": ["rec_001", "rec_002", "rec_003"],
  "format": "zip",
  "include_metadata": true,
  "include_integrity_chain": true
}
```

**Response:**

```json
{
  "export_id": "export_20240126_103500",
  "status": "processing",
  "recording_count": 3,
  "estimated_size_mb": 375,
  "estimated_completion": "2024-01-26T10:35:30Z",
  "download_url": null
}
```

### `GET /api/export/{export_id}/status`

Check status of a bulk export.

**Response:**

```json
{
  "export_id": "export_20240126_103500",
  "status": "complete",
  "recording_count": 3,
  "total_size_mb": 380,
  "download_url": "/api/export/export_20240126_103500/download",
  "expires_at": "2024-01-26T11:35:00Z"
}
```

### `GET /api/export/{export_id}/download`

Download the completed export archive.

### `GET /api/export/{export_id}/report`

Generate a formatted PDF report for legal submission.

**Response:** PDF document with:
- Cover page with export metadata
- Table of recordings
- Integrity verification summary
- Chain of custody log
- Technical appendix

---

## World Model

### `GET /api/world-model`

Current world model state.

**Response:**

```json
{
  "mode": "sparse_and_dense",
  "sparse_available": true,
  "dense_available": true,
  "slam_status": {
    "active": true,
    "mode": "dense",
    "map_size_mb": 45,
    "entities_tracked": 2
  },
  "last_updated_ns": 1706123456789000
}
```

### `GET /api/world-model/3d`

Get the dense 3D SLAM map (if available).

**Query parameters:**
- `format`: `mesh` | `pointcloud` | `voxel`
- `resolution`: `low` | `medium` | `high`

**Response:**

```json
{
  "format": "mesh",
  "bounds_m": {
    "min": [0.0, 0.0, 0.0],
    "max": [15.0, 12.0, 3.0]
  },
  "mesh_url": "/api/world-model/3d/mesh.obj",
  "pointcloud_url": "/api/world-model/3d/points.ply",
  "texture_url": "/api/world-model/3d/texture.jpg",
  "object_count": 45,
  "last_updated": "2024-01-26T10:30:00Z"
}
```

### `GET /api/world-model/3d/mesh`

Download the 3D mesh (OBJ format).

### `GET /api/world-model/3d/pointcloud`

Download the point cloud (PLY format).

### `GET /api/world-model/replay/{timestamp_ms}`

Get world model state at a past timestamp.

**Response:**

```json
{
  "timestamp_ns": 1706120000000,
  "tracks": [
    {
      "id": "trk_0042",
      "class": "HumanWalking",
      "position_m": [3.2, 4.1, 0.9],
      "velocity_ms": [0.8, 1.2, 0.0]
    }
  ],
  "zone_occupancy": {
    "living_room": 1,
    "kitchen": 0
  },
  "slam_snapshot_url": "/api/world-model/replay/1706120000000/snapshot.ply"
}
```

### `GET /api/world-model/entities`

Get all entities currently in the world model.

**Query parameters:**
- `class`: Filter by entity class
- `zone`: Filter by zone

**Response:**

```json
{
  "entities": [
    {
      "id": "ent_0001",
      "type": "furniture",
      "class": "Couch",
      "position_m": [3.0, 2.5, 0.4],
      "dimensions_m": [2.0, 0.8, 0.7],
      "zone_id": "living_room",
      "first_seen": "2024-01-01T00:00:00Z",
      "last_updated": "2024-01-26T10:30:00Z"
    },
    {
      "id": "ent_0002",
      "type": "furniture",
      "class": "DiningTable",
      "position_m": [7.0, 5.0, 0.75],
      "dimensions_m": [1.6, 0.9, 0.75],
      "zone_id": "dining",
      "first_seen": "2024-01-01T00:00:00Z",
      "last_updated": "2024-01-26T10:30:00Z"
    }
  ],
  "total_count": 45
}
```

---

## 360° Panorama (LiDAR Variant D Nodes)

### `GET /api/nodes/{id}/panorama/latest`

Get the latest stitched 360° panorama from a LiDAR Variant D node.

**Query parameters:**
- `format`: `jpeg` | `png`
- `resolution`: `2k` | `4k`

**Response:** Equirectangular panorama image.

### `GET /api/nodes/{id}/panorama/stream`

WebSocket endpoint for live 360° panorama stream.

**WebSocket message format:**

```json
{
  "timestamp_ns": 1706123456789000,
  "width": 3840,
  "height": 1920,
  "format": "jpeg",
  "data": "<base64_encoded_frame>"
}
```

### `GET /api/recordings/{id}/panorama`

Access a 360° panorama recording.

**Response:**

```json
{
  "recording_id": "rec_360_20240126",
  "node_id": "lidar_main_room",
  "start_timestamp": "2024-01-26T10:00:00Z",
  "end_timestamp": "2024-01-26T10:05:00Z",
  "duration_s": 300,
  "resolution": "4k",
  "format": "equirectangular_mp4",
  "file_size_mb": 850,
  "download_url": "/api/recordings/rec_360_20240126/download"
}
```

### `GET /api/nodes/{id}/panorama/calibration`

Get camera calibration data for 360° stitching.

**Response:**

```json
{
  "node_id": "lidar_main_room",
  "camera_count": 8,
  "calibration_date": "2024-01-15T10:00:00Z",
  "calibration_quality": 0.94,
  "cameras": [
    {
      "index": 0,
      "angle_deg": 0,
      "intrinsics": {
        "fx": 1200.5,
        "fy": 1200.5,
        "cx": 640.0,
        "cy": 360.0,
        "distortion": [0.02, -0.01, 0.0, 0.0, 0.0]
      },
      "extrinsics": {
        "rotation": [[1, 0, 0], [0, 1, 0], [0, 0, 1]],
        "translation": [0.0, 0.0, 0.0]
      }
    }
  ],
  "stitching_params": {
    "blend_width_px": 20,
    "exposure_compensation": true
  }
}
```

---

## Coverage Analysis

### `GET /api/coverage/map`

Get the current coverage voxel map.

**Response:** Binary compressed voxel grid.

### `GET /api/coverage/analysis`

Detailed coverage analysis.

**Response:**

```json
{
  "total_volume_m3": 486.0,
  "observed_volume_m3": 462.0,
  "coverage_fraction": 0.95,
  "blind_spots": [
    {
      "zone_id": "garage",
      "volume_m3": 8.5,
      "reason": "No line-of-sight from existing nodes",
      "suggested_placement": {
        "position_m": [10.0, 3.0, 2.5],
        "recommended_node_class": "AlwaysOnNode",
        "estimated_coverage_gain": 0.07
      }
    }
  ],
  "coverage_by_zone": [
    {"zone_id": "living_room", "coverage": 0.97},
    {"zone_id": "kitchen", "coverage": 0.95},
    {"zone_id": "garage", "coverage": 0.88}
  ],
  "node_contributions": [
    {"node_id": "lidar_living_room_north", "voxels_observed": 12450},
    {"node_id": "lidar_living_room_south", "voxels_observed": 11800}
  ]
}
```

### `GET /api/coverage/blind-spots`

List all blind spots.

**Response:**

```json
{
  "blind_spots": [
    {
      "id": "bs_001",
      "zone_id": "garage",
      "center_m": [10.0, 3.0, 0.5],
      "volume_m3": 8.5,
      "reason": "Occluded by shelving unit",
      "nearby_nodes": ["lidar_garage_center"],
      "suggested_fix": "Add choke-point node at garage side door"
    }
  ],
  "total_blind_spot_volume_m3": 24.0,
  "percentage_of_total": 0.05
}
```

### `GET /api/coverage/placement-report`

HTML placement quality report.

**Response:** HTML document with interactive 3D coverage visualization.

### `POST /api/coverage/analyze`

Run placement analysis for a scenario.

**Request:**

```json
{
  "budget": 12,
  "min_coverage_fraction": 0.95,
  "prioritize_zones": ["living_room", "front_entry"]
}
```

**Response:**

```json
{
  "analysis_id": "analysis_20240126",
  "status": "complete",
  "current_coverage": 0.95,
  "target_coverage": 0.95,
  "current_node_count": 10,
  "suggested_nodes": [
    {
      "position_m": [10.0, 3.0, 2.5],
      "class": "AlwaysOnNode",
      "estimated_coverage_gain": 0.07,
      "estimated_cost_usd": 45,
      "priority": "high"
    }
  ],
  "pareto_frontier": [
    {"node_count": 8, "coverage": 0.88},
    {"node_count": 10, "coverage": 0.95},
    {"node_count": 12, "coverage": 0.97},
    {"node_count": 15, "coverage": 0.99}
  ]
}
```

---

## Alerts

### `GET /api/alerts`

Recent alert history.

**Query parameters:**
- `zone`: Filter by zone
- `class`: Filter by alert class
- `since`: ISO timestamp
- `limit`: Maximum results

**Response:**

```json
{
  "alerts": [
    {
      "id": "alert_20240126_103000",
      "timestamp": "2024-01-26T10:30:00Z",
      "alert_class": "UnknownPerson",
      "zone_id": "front_entry",
      "track_id": "trk_0042",
      "confidence": 0.87,
      "trigger_source": "identity_front_door",
      "data_available": {
        "recording_id": "rec_20240126_103000",
        "image_thumbnail": "/api/recordings/rec_20240126_103000/thumbnail"
      },
      "acknowledged": false,
      "acknowledged_at": null
    }
  ],
  "total_count": 45,
  "unacknowledged_count": 3
}
```

### `POST /api/alerts/{id}/acknowledge`

Acknowledge an alert.

**Response:**

```json
{
  "id": "alert_20240126_103000",
  "acknowledged": true,
  "acknowledged_at": "2024-01-26T10:35:00Z"
}
```

### `GET /api/alerts/stream`

WebSocket endpoint for real-time alerts.

---

## Audit

### `GET /api/audit`

Audit log entries.

**Query parameters:**
- `since`: ISO timestamp
- `until`: ISO timestamp
- `node_id`: Filter by node
- `event_type`: Filter by event type
- `limit`: Maximum results

**Response:**

```json
{
  "entries": [
    {
      "id": "audit_20240126_103000",
      "timestamp": "2024-01-26T10:30:00Z",
      "event_type": "IdentityNodeActivated",
      "node_id": "identity_front_door",
      "user": "system",
      "details": {
        "trigger": "UnknownPerson detection",
        "zone": "front_entry"
      }
    },
    {
      "id": "audit_20240126_094500",
      "timestamp": "2024-01-26T09:45:00Z",
      "event_type": "PolicyModeChanged",
      "user": "homeowner",
      "details": {
        "from": "PrivacyFirst",
        "to": "SecurityFirst"
      }
    }
  ],
  "total_count": 234,
  "page": {
    "limit": 100,
    "offset": 0,
    "has_more": true
  }
}
```

### `GET /api/audit/stream`

WebSocket endpoint for real-time audit events.

---

## System Management

### `GET /api/health`

System health summary.

**Response:**

```json
{
  "status": "healthy",
  "uptime_hours": 168.5,
  "edge_controller": {
    "cpu_load_percent": 12,
    "memory_used_mb": 450,
    "memory_total_mb": 2048,
    "temperature_c": 42.3,
    "disk_used_mb": 2340,
    "disk_total_mb": 64000
  },
  "mesh": {
    "nodes_online": 8,
    "nodes_total": 8
  },
  "storage": {
    "recordings_count": 156,
    "storage_used_mb": 2340,
    "storage_available_mb": 61660
  }
}
```

### `GET /api/system/info`

System information.

**Response:**

```json
{
  "firmware_version": "v1.2.0",
  "build_hash": "a3f4b2c9",
  "build_date": "2024-01-15T10:00:00Z",
  "edge_controller_model": "Raspberry Pi CM4",
  "edge_controller_ram_mb": 4096,
  "protocol_version": "2.1",
  "configuration_version": 45
}
```

### `POST /api/system/restart`

Restart the edge controller (requires confirmation).

**Request:**

```json
{
  "confirm": true
}
```

### `POST /api/system/shutdown`

Shutdown the edge controller (requires confirmation).

---

## Configuration

### `GET /api/config`

Get current system configuration.

**Response:**

```json
{
  "deployment": {
    "building_floor_count": 1,
    "floor_height_m": 2.7,
    "total_area_m2": 120.0
  },
  "policy": {
    "default_mode": "PrivacyFirst",
    "auto_away_after_minutes": 30,
    "alert_cooldown_s": 60.0
  },
  "data_handling": {
    "store_raw_video": false,
    "store_raw_audio": false,
    "store_raw_sensor_data": false,
    "storage_target": "sd_card",
    "retention_days": 30,
    "enable_app_streaming": false,
    "continuous_recording": false,
    "recording_trigger": "on_detection",
    "max_storage_mb": 0
  },
  "connectivity": {
    "cellular_enabled": false,
    "cellular_apn": "",
    "fallback_to_wifi": true
  },
  "companion_app": {
    "enabled": true,
    "api_port": 8080,
    "media_port": 9090,
    "enable_remote_access": false
  }
}
```

### `PUT /api/config`

Update system configuration.

**Request:** Partial or full configuration TOML.

**Response:**

```json
{
  "updated": true,
  "updated_fields": ["policy.default_mode", "data_handling.store_raw_video"],
  "restart_required": false,
  "validation_errors": []
}
```

### `GET /api/config/validate`

Validate configuration without applying.

---

## WebSocket Streams

### `/ws/events`

All live events from the sensing mesh.

**Event types:**
- `DetectionEvent`
- `TrackUpdate`
- `ZoneAlert`
- `IdentityEvent`
- `RecordingAvailable`
- `MediaStreamAvailable`
- `SensorDataAvailable`
- `CalibrationUpdate`
- `HealthUpdate`

### `/ws/detections`

Live detection events only.

### `/ws/alerts`

Live alert events only.

### `/ws/occupancy`

Zone occupancy changes.

### `/ws/nodes`

Node health and connectivity updates.

---

## Media Streaming Ports

| Protocol | Port | Path | Description |
|---|---|---|---|
| RTSP | 9090 | `rtsp://aegis-mesh.local:9090/{node_id}` | Per-node video stream |
| WebSocket H.264 | 9091 | `ws://aegis-mesh.local:9091/{node_id}` | Browser-compatible stream |
| WebSocket 360° | 9092 | `ws://aegis-mesh.local:9092/{node_id}` | 360° panorama stream |
| WebSocket Point Cloud | 9093 | `ws://aegis-mesh.local:9093/{node_id}` | LiDAR point cloud (binary) |

---

## Error Responses

All endpoints return standard error format:

```json
{
  "error": "ValidationError",
  "message": "Invalid node_id format",
  "details": {
    "field": "node_id",
    "expected": "string matching pattern ^[a-z0-9_]+$",
    "received": "Front Door"
  },
  "request_id": "req_abc123"
}
```

**Common error codes:**
- `400 Bad Request` — Invalid request format
- `401 Unauthorized` — Missing or invalid token
- `403 Forbidden` — Token valid but insufficient permissions
- `404 Not Found` — Resource not found
- `409 Conflict` — State conflict (e.g., hardware switch disabled)
- `422 Unprocessable Entity` — Validation failed
- `500 Internal Server Error` — Server error
- `503 Service Unavailable` — SLAM or other service not running

---

## Rate Limiting

| Endpoint Category | Rate Limit |
|-------------------|------------|
| Configuration | 10 requests/minute |
| Recordings list | 60 requests/minute |
| Media streams | 5 concurrent streams |
| WebSocket | 10 concurrent connections |

---

## Versioning

The API is versioned via URL path: `/api/v1/...` (future). Current version is implied.

All changes are backward-compatible. Breaking changes require new major version.

---

## OpenAPI Specification

Full OpenAPI 3.0 specification available at: `GET /api/openapi.yaml`

---

**End of AEGIS-MESH API Specification**
