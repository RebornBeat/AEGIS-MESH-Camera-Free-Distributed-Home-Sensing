# AEGIS-MESH Companion App Architecture

**Status:** Specification
**Components:** Mobile app (iOS / Android), Desktop app (Windows / macOS / Linux)
**Server:** Embedded HTTP/WebSocket server on edge controller

---

## 1. Overview

The AEGIS-MESH companion app connects to the edge controller's embedded API server over the local network or optionally over a remote encrypted channel. It provides full visibility into the distributed residential sensing mesh, access to all recorded data, live feeds from any node, dense world model review, and complete system configuration.

The app communicates exclusively with the user's own edge controller. No data is transmitted to any third party.

---

## 2. Mobile App (iOS / Android)

### Features

**Live Detection Map:**
- Interactive floor-plan overlay with all node positions, active zones, and real-time detections.
- Entity blobs with velocity vectors and PentaTrack prediction arcs.
- Zone alerts shown as colored overlays.
- Toggle between floor levels.

**Live Feeds:**
- On-demand live video, audio, or point cloud stream from any configured node.
- 360° panoramic view from nodes with multi-camera arrays.
- LiDAR point cloud live view.

**Recordings Library:**
- Browse all stored recordings: video, audio, point clouds, acoustic events.
- Filter by node, zone, trigger, date, classification.
- Full playback with timeline scrubbing.
- Download recordings to device.

**Alert History:**
- Full log of detections, zone alerts, acoustic events, occupancy events.
- Each alert links to associated recording when captured.

**System Status:**
- Node health and battery status.
- Coverage map (which zones are covered by which nodes).
- Calibration status.

**Configuration:**
- Node management, zone setup, policy mode, data handling settings.

**Remote Access:**
- When configured, access from outside home network via user-set token.

**Legal Export:**
- Export any recording with cryptographic integrity manifest.
- Suitable for insurance, legal proceedings, or personal records.

---

## 3. Desktop App (Windows / macOS / Linux)

### Additional Features

**Full Recording Management:**
- Multi-node timeline. Tag, search, annotate, bulk export.
- Incident grouping: link multiple clips from different nodes covering the same event.

**Dense SLAM 3D World Viewer:**
- Interactive 3D walkthrough of any captured space.
- Timeline scrub to replay past state.
- Entity trajectory overlays on 3D geometry.

**Analytics Dashboard:**
- Zone activity heatmaps and histograms.
- Presence/absence patterns over time.
- Node coverage analysis.
- Acoustic event history.

**Configuration Manager:**
- Full TOML editing with validation.
- Coverage geometry analysis tools.
- Node placement optimizer output viewer.

**Legal Export Suite:**
- Export with full SHA-256 integrity chain.
- Generate formatted report for legal or insurance submission.
- Batch export for date ranges across all nodes.

**Backup and Archive:**
- Schedule automatic backup to NAS or external drive.
- Archive management.

---

## 4. Edge Controller API Reference

### Authentication

```
Authorization: Bearer <user_configured_token>
```

### REST API

| Method | Path | Description |
|---|---|---|
| GET | `/api/nodes` | All nodes with class, health, zone coverage |
| GET | `/api/zones` | All zone definitions |
| GET | `/api/alerts` | Alert history (paginated, filterable) |
| GET | `/api/recordings` | All recordings (paginated, filterable) |
| GET | `/api/recordings/{id}` | Download recording file |
| GET | `/api/recordings/{id}/meta` | Integrity manifest |
| DELETE | `/api/recordings/{id}` | Delete recording |
| GET | `/api/export/{id}` | Export with integrity chain (ZIP) |
| GET | `/api/nodes/{id}/stream` | Open live media stream |
| GET | `/api/nodes/{id}/pointcloud` | Open live point cloud stream |
| GET | `/api/world-model` | Current world model (sparse + dense availability) |
| GET | `/api/world-model/3d` | Current 3D SLAM mesh (if available) |
| GET | `/api/world-model/replay/{timestamp_ms}` | World model at past timestamp |
| GET | `/api/analytics/zones` | Zone activity statistics |
| GET | `/api/analytics/coverage` | Coverage analysis results |
| GET | `/api/analytics/detections` | Detection history for time range |
| POST | `/api/config` | Update configuration |
| GET | `/api/config` | Read current configuration |
| POST | `/api/policy/mode` | Set policy mode |
| POST | `/api/calibration/walk-through/start` | Start walk-through calibration |
| GET | `/api/health` | System health |
| GET | `/api/placement/analysis` | Coverage placement analysis results |

### WebSocket Streams

| Path | Description |
|---|---|
| `/ws/events` | All live MeshEvent types |
| `/ws/detections` | Live detection events only |
| `/ws/alerts` | Live zone alert events |
| `/ws/occupancy` | Live occupancy change events |
| `/ws/nodes` | Live node health updates |

### Media Streaming

| Protocol | Port | Description |
|---|---|---|
| RTSP | 9090 | Per-node video (Identity/LiDAR with camera) |
| WebSocket H.264 | 9091 | Browser/app compatible stream |
| WebSocket 360° | 9092 | 360° panoramic from multi-camera nodes |
| WebSocket Point Cloud | 9093 | Live LiDAR point cloud (binary) |

---

## 5. Integrity Manifest Format

```json
{
  "export_version": "1.0",
  "node_id": "identity_node_entry",
  "node_class": "IdentityNode",
  "zone_id": "front_entry",
  "recording_start_utc": "2024-03-15T14:30:22.004Z",
  "recording_duration_s": 12.8,
  "format": "h264_mp4",
  "modalities": ["video"],
  "trigger": "on_detection",
  "file_sha256": "c9d4e2f8a1b3c5d7e9f0a2b4c6d8e0f2a4b6c8d0e2f4a6b8c0d2e4f6a8b0c2d4",
  "firmware_version": "v1.2.0",
  "hardware_variant": "identity_node_v1",
  "export_timestamp_utc": "2024-03-16T09:00:00.000Z",
  "chain_hash": "e3f5a7b9c1d2e4f6a8b0c2d4e6f8a0b2"
}
```

---

## 6. Data Handling Configuration (`aegis-mesh.toml`)

```toml
[data_handling]
store_raw_video = false
store_raw_audio = false
store_raw_sensor_data = false
storage_target = "internal"          # "sd_card" | "internal" | "network_path"
retention_days = 90
enable_app_streaming = false
stream_quality = "medium"
continuous_recording = false
recording_trigger = "on_detection"
max_storage_mb = 0
include_integrity_chain = true

[companion_app]
enabled = true
bind_address = "0.0.0.0"
api_port = 8080
media_port = 9090
enable_remote_access = false
remote_access_token = ""
```

---

## 7. Security

- All endpoints require user-set bearer token.
- TLS recommended for remote access.
- No external services required. No telemetry.
- App communicates exclusively with the user's own edge controller.
