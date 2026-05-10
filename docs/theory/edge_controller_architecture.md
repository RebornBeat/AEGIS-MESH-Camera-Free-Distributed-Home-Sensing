# Edge Controller Architecture — AEGIS-MESH

**Project:** AEGIS-MESH
**Domain:** Central compute and network gateway for distributed residential sensing
**Implementation:** `crates/aegis-edge-controller/`, `crates/aegis-api/`, `crates/aegis-storage/`
**Status:** Reference Design

---

## 1. Purpose

This document specifies the architecture of the AEGIS-MESH Edge Controller — the central compute hub, network gateway, fusion engine, SLAM processor, and API server for the distributed residential sensing mesh.

**The Edge Controller is to AEGIS-MESH what the Belt Node is to SENTINEL-WEAR.**

**Critical architectural principle:** The Edge Controller is the **sole node with external network connectivity** in the AEGIS-MESH system. All other nodes (Always-On, LiDAR, Identity) communicate exclusively via mesh radio (BLE, Thread, or PoE Ethernet) to the Edge Controller. They have no WiFi, no cellular, and no direct connection to external networks.

---

## 2. Role in the System

| Function | Edge Controller | Notes |
|----------|-----------------|-------|
| **Compute Hub** | ✅ Yes | Runs all fusion, tracking, SLAM, and coordination |
| **Fusion Center** | ✅ Yes | Multi-node detection fusion (CI + JPDA), PentaTrack integration |
| **SLAM Processor** | ✅ Yes | Dense 3D reconstruction from LiDAR + cameras |
| **API Server** | ✅ Yes | HTTP/WebSocket server for companion app |
| **Recording Manager** | ✅ Yes | Aggregates recordings, integrity chains, legal export |
| **WiFi Gateway** | ✅ Yes | Primary external connectivity |
| **Cellular Gateway** | ✅ Optional | Remote access via LTE/5G SIM |
| **Mesh Hub** | ✅ Yes | Coordinates all inter-node communication |
| **Time Master** | ✅ Yes | Provides clock reference for all nodes |
| **Policy Engine** | ✅ Yes | Privacy-First / Security-First / Away / Silent modes |
| **Power Source** | AC powered | Not battery-constrained (unlike SENTINEL-WEAR belt) |

---

## 3. Critical Architectural Constraint: Sole External Gateway

### 3.1 The Principle

**Only the Edge Controller has WiFi, Cellular, and direct external network connectivity.**

**All other nodes (Always-On, LiDAR, Identity) communicate exclusively via mesh radio to the Edge Controller.** They have no WiFi, no cellular, and no direct connection to the companion app or external services.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    AEGIS-MESH Connectivity Model                            │
│                                                                             │
│  Always-On Nodes ──┐                                                        │
│  LiDAR Nodes───────┤── Mesh Radio ──► Edge Controller ──► WiFi ──► App     │
│  Identity Nodes────┘  (BLE/Thread/PoE)       │                              │
│                                              ├──► Cellular ──► Remote App  │
│                                              └──► BLE direct ──► Local App │
│                                                                             │
│  ⚠️  EDGE CONTROLLER IS THE ONLY NODE WITH WiFi / Cellular / External BLE   │
│  ⚠️  ALL OTHER NODES = MESH RADIO ONLY (no external connectivity)           │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 3.2 Why This Architecture?

| Constraint | WiFi on Sensor Node | BLE/Mesh on Sensor Node |
|------------|---------------------|-------------------------|
| **Power** | 300-1000+ mW active | 5-15 mW active |
| **Heat** | Significant thermal load | Minimal |
| **Complexity** | Requires TCP/IP stack, security | Simple protocol |
| **Cost** | Higher | Lower |
| **Placement flexibility** | Requires WiFi coverage | Mesh self-organizes |
| **Security surface** | Direct internet exposure | Isolated from external |

**Conclusion:** WiFi and cellular are appropriate only at the Edge Controller, where AC power, larger enclosure, and centralized security management are acceptable.

---

## 4. Transport Layer Hierarchy

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         BANDWIDTH HIERARCHY                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│  Mesh Radio (BLE 5.x / Thread / PoE Ethernet)                               │
│  → Control plane (configuration, commands, health)                          │
│  → Detection metadata (position, classification, confidence)                 │
│  → Sensor data streams (compressed)                                          │
│  → Point clouds from LiDAR nodes                                            │
│  → 360° camera streams from LiDAR Variant D                                  │
│  BLE: ~500 Kbps - 2 Mbps practical                                          │
│  Thread: ~250 Kbps max                                                       │
│  PoE Ethernet: 100 Mbps - 1 Gbps (LiDAR + 360° camera nodes)                │
├─────────────────────────────────────────────────────────────────────────────┤
│  WiFi (50-300+ Mbps) — EDGE CONTROLLER ONLY                                  │
│  → Primary companion app connection                                          │
│  → High-bandwidth streaming (360° video, SLAM data)                          │
│  → Multiple simultaneous camera streams                                      │
│  → Remote access (when enabled)                                             │
│  → Power: 300-500 mW when streaming                                          │
├─────────────────────────────────────────────────────────────────────────────┤
│  Cellular (Variable by module) — EDGE CONTROLLER ONLY                        │
│  → LTE Cat 1: 5-10 Mbps — alerts, metadata, compressed clips                │
│  → LTE Cat 4: 50-150 Mbps — moderate streaming                              │
│  → 5G: 100-1000+ Mbps — suitable for 360° live relay                         │
│  → Used when WiFi unavailable (remote access, away from home)               │
│  → Power: 150-1200 mW depending on module and activity                       │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 5. Hardware Variants

### 5.1 Variant A — SBC (Raspberry Pi 4/5 Class)

**Target:** Budget deployment, basic capability

| Component | Specification |
|-----------|---------------|
| Compute | ARM Cortex-A72/A76, Quad-core |
| RAM | 4-8 GB |
| Storage | SD card or NVMe (via USB adapter) |
| WiFi | Built-in 802.11ac |
| Ethernet | Built-in Gigabit |
| Cellular | USB module (optional) |
| Power | 5V/3A USB-C or PoE HAT |
| Typical Power | 3-7 W |
| Enclosure | Small desktop or wall-mount |

**Capabilities:**
- ✅ Sparse tracking (PentaTrack)
- ✅ Basic API server
- ✅ Recording management
- ⚠️ SLAM limited (slow, may require reduced resolution)
- ⚠️ 2-4 camera streams
- ❌ Real-time 360° stitching challenging

**Cost:** $50-150

**Deployment scenarios:**
- Single apartment or small home
- Research prototype
- Budget-conscious deployment

### 5.2 Variant B — ARM with NPU (Jetson Nano/Orin Class)

**Target:** Standard production deployment

| Component | Specification |
|-----------|---------------|
| Compute | ARM + NVIDIA GPU + NPU |
| RAM | 4-16 GB |
| Storage | NVMe SSD |
| WiFi | Built-in or external |
| Ethernet | Gigabit |
| Cellular | M.2 or USB module (optional) |
| Power | 5V/4A or PoE 802.3at |
| Typical Power | 7-15 W |
| Enclosure | Desktop or rack-mount |

**Capabilities:**
- ✅ Full SLAM processing
- ✅ 360° camera stitching
- ✅ 8+ camera streams
- ✅ Real-time neural inference
- ✅ Dense world model

**Cost:** $200-600

**Deployment scenarios:**
- Full-size home
- Multi-camera deployment
- Dense SLAM required
- Production security deployment

### 5.3 Variant C — x86 Mini PC

**Target:** Maximum flexibility, commercial deployment

| Component | Specification |
|-----------|---------------|
| Compute | Intel/AMD x86, 4-8 cores |
| RAM | 8-32 GB |
| Storage | NVMe SSD (250 GB - 2 TB) |
| WiFi | PCIe or USB card |
| Ethernet | Dual Gigabit or 2.5 GbE |
| Cellular | USB module |
| Power | 19V/65W external adapter |
| Typical Power | 10-25 W |
| Enclosure | Rack-mount or wall-mount |

**Capabilities:**
- ✅ Maximum compute flexibility
- ✅ Docker containers for modularity
- ✅ Unlimited SLAM capability
- ✅ Multiple simultaneous 360° streams
- ✅ Full neural inference
- ✅ Long-term storage

**Cost:** $300-1000

**Deployment scenarios:**
- Large home or multi-building
- Commercial property
- Long-term storage requirement
- Professional installation

### 5.4 Variant D — Industrial Grade

**Target:** Harsh environments, commercial/industrial deployment

| Component | Specification |
|-----------|---------------|
| Compute | ARM or x86, industrial-rated |
| RAM | 4-16 GB |
| Storage | Industrial SSD, extended temperature |
| WiFi | Industrial module |
| Ethernet | Industrial Gigabit with surge protection |
| Cellular | Industrial LTE/5G module |
| Power | Wide input (9-36V), surge protection |
| Enclosure | DIN-rail or NEMA-rated |
| Temperature | -20°C to +70°C operating |
| Typical Power | 5-15 W |

**Capabilities:** Same as Variant B or C (depending on compute)

**Cost:** $500-2000

**Deployment scenarios:**
- Industrial facility
- Outdoor-protected location
- High-reliability requirement
- Professional security deployment

---

## 6. Connectivity Architecture Detail

### 6.1 WiFi Interface

| Specification | Value |
|---------------|-------|
| Standard | 802.11 a/b/g/n/ac/ax (depending on hardware) |
| Purpose | Primary companion app connection, media streaming |
| Frequency | 5 GHz preferred (avoids 2.4 GHz conflict with BLE mesh) |
| Security | WPA3, optional TLS for API |
| Power | 300-500 mW when streaming |

**Configuration:**

```toml
[connectivity.wifi]
enabled = true
prefer_5ghz = true
ssid = ""                          # Set at runtime for security
password = ""                      # Set at runtime
country_code = "US"
channel = "auto"
tx_power_dbm = 20
```

**WiFi is present ONLY on the Edge Controller.** No sensor node has WiFi capability.

### 6.2 Cellular / SIM Interface

The Edge Controller supports optional LTE/5G for remote access when home WiFi is unavailable.

**Physical interface:**
- nano-SIM slot (4FF) or eSIM footprint
- Cellular module via USB (variants A, C) or M.2 (variants B, D)
- U.FL antenna connector for external cellular antenna

**Supported Modules:**

| Module | Technology | Speed | Interface | Power (Active) | Use Case |
|--------|------------|-------|-----------|----------------|----------|
| Quectel EC21 | LTE Cat 1 | 5-10 Mbps | UART | 150-300 mA | Alerts, metadata |
| Quectel EC25 | LTE Cat 4 | 50-150 Mbps | UART/USB | 400-600 mA | Compressed clips |
| Quectel RM502Q-AE | 5G Sub-6GHz | 100+ Mbps | USB 3.x | 800-1200 mA | Live 360° streaming |
| Quectel EG912Y-GL | LTE Cat 12 | 300+ Mbps | USB | 500-800 mA | Multi-carrier eSIM |
| Sierra Wireless RV55 | LTE/5G | Variable | USB | Variable | Industrial grade |

**SIM Interface Signals:**

| Signal | Description |
|--------|-------------|
| `SIM_CLK` | SIM card clock |
| `SIM_DATA` | SIM card data |
| `SIM_RST` | SIM card reset |
| `SIM_VCC` | SIM card power (1.8V or 3.0V selectable) |
| `SIM_DET` | SIM card present detect |

**Configuration:**

```toml
[connectivity.cellular]
enabled = false
sim_type = "nano_sim"            # "nano_sim" | "esim"
apn = "your.carrier.apn"
preferred_bands = []              # Empty = carrier default
fallback_to_wifi = true           # Prefer WiFi when available
always_on_cellular = false        # Keep cellular active even with WiFi
alert_via_cellular = true         # Send alerts via cellular when active
stream_via_cellular = false       # Stream video via cellular (data cost)
emergency_contact = ""            # Phone/endpoint for emergency alerts
```

**Cellular is present ONLY on the Edge Controller.** No sensor node has cellular capability.

### 6.3 Bluetooth (BLE)

The Edge Controller's BLE radio serves two purposes:

1. **Mesh coordination:** Communicates with BLE-only sensor nodes
2. **Direct app connection:** Companion app can connect via BLE when WiFi unavailable

**Limitations of BLE direct connection:**
- No video streaming
- No SLAM data transfer
- Metadata and alerts only

**Configuration:**

```toml
[connectivity.bluetooth]
ble_direct_enabled = true
ble_advertising_name = "aegis-mesh"
ble_mesh_coordinator = true
```

### 6.4 Ethernet

**Primary for:**
- PoE-powered sensor nodes (LiDAR Variant C/D)
- High-bandwidth wired backhaul for fixed installations
- Direct PC companion app connection

**Configuration:**

```toml
[connectivity.ethernet]
enabled = true
static_ip = ""                    # Empty = DHCP
gateway = ""
dns_servers = []
```

### 6.5 USB-C

**Purposes:**
- PC companion app connection
- Firmware updates
- Charging (for battery-backup variants)
- Debug access

---

## 7. Mesh Network Architecture

### 7.1 Mesh Protocol Stack

The Edge Controller coordinates all inter-node communication via the mesh protocol:

```
┌─────────────────────────────────────────────────────────────────┐
│                    MESH PROTOCOL STACK                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Application Layer                                               │
│  ├── Detection events (per-node sensor detections)              │
│  ├── Track updates (fused entity positions)                     │
│  ├── Anomaly events (fall, intrusion, acoustic)                 │
│  ├── Configuration commands                                     │
│  ├── Health/heartbeat                                           │
│  ├── Recording notifications                                    │
│  └── SLAM data (point clouds, keyframes)                        │
│                                                                  │
│  Transport Layer                                                 │
│  ├── BLE 5.x (low-power nodes)                                  │
│  ├── Thread (IPv6-based mesh)                                   │
│  └── PoE Ethernet (high-bandwidth nodes)                        │
│                                                                  │
│  Security Layer                                                  │
│  ├── HMAC-SHA256 message signing                                │
│  ├── Sequence numbers (replay protection)                       │
│  └── Per-node authentication keys                               │
│                                                                  │
│  Physical Layer                                                  │
│  ├── 2.4 GHz radio (BLE/Thread)                                 │
│  └── Ethernet PHY (PoE nodes)                                   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 7.2 Mesh Message Types

```rust
pub enum MeshMessage {
    // Detection and tracking
    Detection(DetectionEvent),
    TrackUpdate(TrackState),
    AnomalyEvent(AnomalyType),
    
    // Sensor data (compressed)
    PointCloudChunk {
        node_id: NodeId,
        chunk_id: u32,
        total_chunks: u32,
        data: Vec<u8>,
    },
    
    // Camera/identity data
    RecordingAvailable {
        node_id: NodeId,
        recording_path: String,
        timestamp: u64,
        duration_s: f32,
        trigger: String,
    },
    MediaStreamAvailable {
        node_id: NodeId,
        stream_endpoint: String,
        format: String,
    },
    
    // Configuration
    NodeConfig(NodeConfigUpdate),
    CalibrationCommand(CalibrationPhase),
    
    // Health
    NodeHeartbeat {
        node_id: NodeId,
        battery_percent: u8,
        health_flags: u16,
        timestamp: u64,
    },
    
    // Security
    TimeSync {
        master_timestamp_us: u64,
    },
}
```

### 7.3 Time Synchronization

The Edge Controller is the time master for all nodes:

| Mechanism | Accuracy | Use Case |
|-----------|----------|----------|
| BLE connection events | ~1 ms | Standard operation |
| Mesh timestamp broadcast | ~100 µs | Improved sync |
| NTP/PTP over Ethernet | < 1 ms | PoE nodes |
| Optional UWB | < 1 µs | SLAM-precision sync |

**Configuration:**

```toml
[mesh.time_sync]
mode = "connection_event"         # "connection_event" | "broadcast" | "uwb"
sync_interval_ms = 100
max_drift_us = 500
```

---

## 8. Compute Architecture

### 8.1 Software Stack

```
┌─────────────────────────────────────────────────────────────────┐
│                    EDGE CONTROLLER SOFTWARE STACK               │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  aegis-edge-controller (main binary)                            │
│  ├── Runs on Linux (systemd service)                           │
│  └── Coordinates all other crates                               │
│                                                                  │
│  Core Crates:                                                    │
│  ├── aegis-core          # Shared types, events, config         │
│  ├── aegis-perception    # Per-node sensor pipelines            │
│  ├── aegis-fusion        # Multi-node detection fusion          │
│  ├── aegis-tracking      # PentaTrack bridge                    │
│  ├── aegis-slam          # Dense SLAM, 360° stitching           │
│  ├── aegis-policy        # Policy engine (Privacy/Security/Away)│
│  ├── aegis-alerts        # Alert classification and routing     │
│  ├── aegis-storage       # Recording management, retention      │
│  ├── aegis-api           # HTTP/WebSocket server                │
│  ├── aegis-mesh-protocol # Mesh protocol implementation         │
│  └── aegis-calibration   # Calibration algorithms               │
│                                                                  │
│  External Dependencies:                                          │
│  ├── OMNI-SENSE          # Sensor abstraction, fusion           │
│  ├── PentaTrack          # Predictive tracking                  │
│  └── System libraries    # OpenCV, PCL, SLAM frameworks         │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 8.2 Processing Pipeline

```
┌─────────────────────────────────────────────────────────────────┐
│                    PROCESSING PIPELINE                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Input Sources:                                                  │
│  ├── Always-On nodes → Detection events (BLE/Thread)           │
│  ├── LiDAR nodes → Point clouds (PoE/Thread)                   │
│  ├── Identity nodes → Classification results (BLE)             │
│  └── Environmental nodes → Atmospheric data (BLE)              │
│                                                                  │
│  Fusion Layer (aegis-fusion):                                   │
│  ├── ZoneFuser: Per-zone detection aggregation                  │
│  ├── JPDA: Multi-node association                              │
│  └── Covariance Intersection: Cross-node track fusion          │
│                                                                  │
│  Tracking Layer (aegis-tracking):                               │
│  ├── PentaTrack bridge: Predictive center-field                │
│  ├── Track management: Create, update, delete tracks           │
│  └── Anomaly detection: Deviation from predicted path          │
│                                                                  │
│  SLAM Layer (aegis-slam) — Optional:                            │
│  ├── Point cloud processing                                    │
│  ├── Visual odometry (from LiDAR + camera nodes)               │
│  ├── Loop closure detection                                    │
│  └── Dense mesh generation                                     │
│                                                                  │
│  Output:                                                         │
│  ├── Track state → World model                                  │
│  ├── Anomaly events → Alert system                             │
│  ├── 360° panorama → API media server                          │
│  └── SLAM map → Storage + API                                  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 8.3 SLAM Processing

**Supported SLAM frameworks:**

| Framework | Use Case | Hardware Requirement |
|-----------|----------|---------------------|
| FAST-LIO2 | LiDAR-only SLAM | Any variant |
| LIO-SAM | LiDAR-inertial odometry | Variant B+ |
| ORB-SLAM3 | Visual + LiDAR fusion | Variant B+ with NPU |
| RTAB-Map | RGB-D SLAM | Variant B+ |

**Configuration:**

```toml
[slam]
enabled = false
framework = "fast_lio2"           # "fast_lio2" | "lio_sam" | "orb_slam3"
resolution_m = 0.05               # Voxel size
loop_closure_enabled = true
keyframe_interval_s = 0.5
max_map_size_mb = 1024

[slam.camera_integration]
use_lidar_variant_c_cameras = true
use_lidar_variant_d_360 = true
stitch_360_on_edge_controller = true
```

---

## 9. Storage Architecture

### 9.1 Storage Hierarchy

| Storage Type | Capacity | Purpose |
|--------------|----------|---------|
| System SD card / NVMe | 32-128 GB | OS, logs, configuration |
| Data storage | 250 GB - 8 TB | Recordings, SLAM maps, evidence |
| Network storage | Variable | Backup, archive |

### 9.2 Recording Management

**Sources:**
- All sensor nodes send `RecordingAvailable` events
- Edge controller aggregates and indexes recordings
- Integrity chain computed and stored

**Data types:**

| Data Type | Source | Typical Size |
|-----------|--------|---------------|
| Detection metadata | All nodes | KB per event |
| Point cloud clips | LiDAR nodes | MB per clip |
| Video clips | LiDAR C/D, Identity nodes | MB-GB per clip |
| 360° panorama | LiDAR Variant D | GB per session |
| SLAM map | Edge controller | MB per room |

**Configuration:**

```toml
[data]
store_raw_video = false
store_raw_audio = false
store_raw_point_clouds = true
storage_target = "nvme_ssd"       # "sd_card" | "internal_flash" | "nvme_ssd" | "network_path"
retention_days = 30
max_storage_mb = 512000           # 500 GB
continuous_recording = false
recording_trigger = "on_detection"
include_integrity_chain = true

[data.integrity]
algorithm = "sha256"
manifest_format = "json"
```

### 9.3 Legal Evidence Export

**Export package includes:**
- Raw recording file
- Integrity manifest (JSON)
- Metadata (node ID, timestamp, trigger)
- Chain hash for cryptographic proof

**Export format:**

```json
{
  "export_version": "1.0",
  "node_id": "lidar_node_01",
  "node_class": "LidarNode",
  "zone_id": "living_room",
  "recording_start": "2024-03-15T14:30:22.004Z",
  "recording_duration_s": 42.3,
  "format": "h264_mp4",
  "modalities": ["video", "point_cloud"],
  "trigger": "on_detection",
  "file_sha256": "a3f4b2c9d8e1f7a2b5c8d3e6f9a0b1c2d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8",
  "firmware_version": "v1.0.0",
  "hardware_variant": "lidar_variant_c",
  "edge_controller_version": "v2.1.0",
  "export_timestamp": "2024-03-16T09:00:00.000Z",
  "chain_hash": "b7e8d1f3a2c4e5f6b7a8c9d0e1f2a3b4"
}
```

---

## 10. API Server Architecture

### 10.1 Server Stack

| Component | Technology | Purpose |
|-----------|------------|---------|
| HTTP framework | Axum (Rust) | REST API |
| WebSocket | Axum | Real-time events |
| Media server | RTSP/WebSocket | Video streaming |
| Authentication | Bearer token | API security |
| TLS | Rustls | Encrypted connections |

### 10.2 REST API Endpoints

**Node Management:**

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/nodes` | All nodes with health status |
| GET | `/api/nodes/{id}` | Single node details |
| POST | `/api/nodes/{id}/config` | Update node configuration |
| GET | `/api/nodes/{id}/health` | Detailed health metrics |

**Zone Management:**

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/zones` | All zones |
| POST | `/api/zones` | Create zone |
| PUT | `/api/zones/{id}` | Update zone |
| DELETE | `/api/zones/{id}` | Delete zone |
| GET | `/api/zones/{id}/coverage` | Coverage analysis for zone |

**Detection and Tracking:**

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/tracks` | Current tracks |
| GET | `/api/tracks/{id}` | Single track details |
| GET | `/api/tracks/history` | Track history (paginated) |
| GET | `/api/detections/recent` | Recent detection events |

**World Model:**

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/world-model` | Current world model state |
| GET | `/api/world-model/3d` | SLAM mesh (if available) |
| GET | `/api/world-model/replay/{timestamp}` | World model at past time |

**360° and Media:**

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/nodes/{id}/stream` | Open video stream |
| GET | `/api/nodes/{id}/panorama` | 360° panorama stream |
| GET | `/api/nodes/{id}/pointcloud` | Live point cloud stream |

**Recordings:**

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/recordings` | List recordings |
| GET | `/api/recordings/{id}` | Download recording |
| GET | `/api/recordings/{id}/manifest` | Integrity manifest |
| DELETE | `/api/recordings/{id}` | Delete recording |
| GET | `/api/export/{id}` | Export with integrity chain |

**Analytics:**

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/analytics/zones` | Zone activity statistics |
| GET | `/api/analytics/occupancy` | Occupancy history |
| GET | `/api/analytics/coverage` | Coverage metrics |

**Connectivity:**

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/connectivity` | Current connectivity status |
| GET | `/api/connectivity/cellular` | Cellular status and signal |
| POST | `/api/connectivity/cellular/toggle` | Enable/disable cellular |

**Configuration:**

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/config` | Current configuration |
| POST | `/api/config` | Update configuration |
| GET | `/api/policy` | Current policy mode |
| POST | `/api/policy/mode` | Set policy mode |

**Calibration:**

| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/calibration/start` | Begin calibration |
| GET | `/api/calibration/status` | Calibration progress |
| POST | `/api/calibration/complete` | Finish calibration |

### 10.3 WebSocket Endpoints

| Path | Description |
|------|-------------|
| `/ws/events` | All mesh events |
| `/ws/detections` | Live detections |
| `/ws/alerts` | Live alerts |
| `/ws/tracks` | Track updates |
| `/ws/world-model` | World model updates |

### 10.4 Media Streaming

| Protocol | Port | Description |
|----------|------|-------------|
| RTSP | 9090 | Per-node video streams |
| WebSocket H.264 | 9091 | Browser-compatible video |
| WebSocket 360° | 9092 | 360° panorama from LiDAR Variant D |
| WebSocket Point Cloud | 9093 | Live point cloud data |

---

## 11. 360° Camera Integration

### 11.1 LiDAR Variant D Processing

LiDAR Variant D nodes transmit raw camera streams to the Edge Controller via PoE. The Edge Controller handles:

1. **Stitching:** Combines N camera frames into equirectangular 360° panorama
2. **Encoding:** H.264/H.265 encoding for streaming and storage
3. **SLAM:** Uses 360° visual features for loop closure
4. **Storage:** Records 360° video with SLAM pose annotations

### 11.2 Stitching Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    360° STITCHING PIPELINE                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Input (from LiDAR Variant D node via PoE):                    │
│  ├── Camera 0 stream (0°)                                       │
│  ├── Camera 1 stream (45°)                                      │
│  ├── ...                                                        │
│  └── Camera 7 stream (315°)                                     │
│                                                                  │
│  Processing (Edge Controller):                                  │
│  ├── Load calibration (intrinsics + extrinsics)                │
│  ├── Per-frame alignment check                                 │
│  ├── Feature matching between adjacent cameras                  │
│  ├── Seam optimization                                         │
│  ├── Equirectangular projection                                │
│  └── H.264 encoding                                            │
│                                                                  │
│  Output:                                                         │
│  ├── Live 360° stream (WebSocket 9092)                         │
│  ├── Recording (NVMe/SD)                                       │
│  ├── SLAM visual features                                      │
│  └── Panorama frame files (for legal export)                   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 11.3 Configuration

```toml
[360_processing]
enabled = true
stitch_on_edge = true             # vs. on-node stitching
output_resolution = "3840x1920"   # 4K equirectangular
encoding = "h264"                 # "h264" | "h265"
quality = "high"                  # "low" | "medium" | "high"
keyframe_interval_s = 0.5
calibration_file = "/etc/aegis/360_calibration.json"
```

---

## 12. Power and Thermal

### 12.1 Power Architecture

| Variant | Power Source | Typical Draw | Peak Draw |
|---------|--------------|--------------|-----------|
| A (SBC) | USB-C 5V/3A or PoE | 3-7 W | 15 W |
| B (NPU) | 5V/4A or PoE 802.3at | 7-15 W | 25 W |
| C (x86) | 19V/65W adapter | 10-25 W | 65 W |
| D (Industrial) | 9-36V wide input | 5-15 W | 25 W |

### 12.2 Thermal Management

**Unlike SENTINEL-WEAR's belt node, the Edge Controller is NOT constrained by skin-contact temperature.**

However, thermal management is still important for reliability:

| Measure | Application |
|---------|--------------|
| Passive heatsinks | All variants |
| Active cooling (fan) | Variants B, C for sustained high load |
| Thermal throttling | Firmware monitors temperature, reduces SLAM rate if needed |
| Ambient rating | 0-50°C operating (indoor) |

### 12.3 Uninterruptible Power Supply (Optional)

For critical deployments, the Edge Controller can be connected to a UPS:

```toml
[power.ups]
enabled = false
type = "usb_hid"                  # "usb_hid" | "apcupsd" | "network"
shutdown_delay_s = 60
shutdown_command = "systemctl poweroff"
```

---

## 13. Security Architecture

### 13.1 Network Security

| Measure | Implementation |
|---------|----------------|
| WiFi encryption | WPA3-Personal or WPA2-Enterprise |
| API authentication | Bearer token (user-configured) |
| TLS | Optional for local, mandatory for remote |
| Mesh message signing | HMAC-SHA256 |
| Sequence numbers | Replay attack prevention |

### 13.2 Access Control

```toml
[security]
api_authentication = "bearer"     # "none" | "bearer"
bearer_token = ""                  # Set by user
tls_enabled = false                # Enable for remote access
tls_cert_path = "/etc/aegis/cert.pem"
tls_key_path = "/etc/aegis/key.pem"

[security.audit]
enabled = true
log_path = "/var/log/aegis/audit.log"
retention_days = 90
```

### 13.3 Update Security

- Firmware updates signed with Ed25519
- User must approve updates (not automatic)
- Verification before installation
- Rollback capability

---

## 14. Companion App Integration

### 14.1 Connection Models

| Connection | Primary Use | Bandwidth |
|------------|-------------|-----------|
| WiFi (local) | Primary | Full (50-300 Mbps) |
| Cellular (remote) | Remote access | Variable (5-1000 Mbps) |
| USB (direct) | Admin, firmware update | Full |
| BLE (fallback) | Minimal access | Metadata only |

### 14.2 App Features Served by Edge Controller

- Live detection map (floor-plan overlay)
- Live camera/360° streams
- Recording library
- 3D world model viewer
- Analytics dashboards
- Configuration management
- Legal export

See `apps/README.md` for full companion app specification.

---

## 15. Policy Engine

### 15.1 Operational Modes

| Mode | Always-On / LiDAR | Identity Nodes | Use Case |
|------|-------------------|----------------|----------|
| **Privacy-First** | Active, metadata-only | Off or metadata-only | Default |
| **Security-First** | Active | Activates on trigger | Home alone, away |
| **Away** | Maximum sensitivity | Activates on trigger | Extended absence |
| **Silent** | Record-only, no alerts | Configured recording | Covert monitoring |

### 15.2 Mode Transitions

```toml
[policy]
default_mode = "PrivacyFirst"
auto_away_after_minutes = 30      # Switch to Away after inactivity
return_to_default_after_minutes = 5
```

### 15.3 Identity Node Activation Policy

```toml
[policy.identity]
activation_trigger = "on_unknown_detection"  # "never" | "on_unknown_detection" | "on_any_motion" | "always"
deactivation_delay_s = 60
require_manual_enable = false
```

---

## 16. Calibration

### 16.1 Walk-Through Calibration

**Process:**
1. User enables calibration mode in companion app
2. User walks through home at normal pace (30-60 seconds)
3. Multiple nodes detect user simultaneously
4. System trilaterates node positions
5. Occlusion map generated
6. Coverage analysis performed

**Output:**
- Node position map
- Occlusion/coverage map
- Placement suggestions

### 16.2 360° Camera Calibration

For LiDAR Variant D nodes:

1. Place calibration checkerboard at multiple positions
2. System captures images from all cameras
3. Intrinsics and extrinsics computed
4. Stitching calibration stored

---

## 17. Testing

### 17.1 Test Points

| Test Point | Signal | Description |
|------------|--------|-------------|
| `SIM_VCC` | Power | SIM card power rail |
| `CELLULAR_PWR` | Power | Cellular module power |
| `WIFI_RST` | GPIO | WiFi module reset |
| `CELLULAR_STATUS` | GPIO | Cellular module status |
| `MESH_RADIO_TEST` | RF | Mesh radio test point |
| `SWD_CLK/DATA` | Debug | SWD interface |
| `UART_TX` | Debug | Console output |

### 17.2 Factory Tests

1. Power-on self-test
2. Rail verification (all within ±5%)
3. WiFi association
4. Cellular module AT command response
5. Mesh radio ping to test node
6. API server response
7. Recording write/read test
8. 360° stitching test (if applicable)

---

## 18. Mechanical

### 18.1 Enclosure Options

| Variant | Form Factor | Mounting |
|---------|-------------|----------|
| A (SBC) | Small desktop | Shelf, wall-mount |
| B (NPU) | Desktop | Shelf, rack ears |
| C (x86) | Mini-tower or rack | Rack-mount, wall-mount |
| D (Industrial) | DIN-rail or NEMA | DIN rail, NEMA enclosure |

### 18.2 Antenna Placement

```
Edge Controller Enclosure (Top View)
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│   [Cellular Antenna]                    [WiFi Antenna]         │
│   (External connector)                  (External connector)   │
│                                                                 │
│   ┌─────────────────────────────────────────────────────┐      │
│   │              Internal Electronics                   │      │
│   │                                                     │      │
│   │   [Mesh Radio Antenna]    [Mesh Radio Antenna]     │      │
│   │   (Internal PCB)          (Internal PCB)            │      │
│   │                                                     │      │
│   └─────────────────────────────────────────────────────┘      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 19. Configuration Reference

```toml
# aegis-mesh.toml

[deployment]
building_name = "My Home"
building_type = "single_family"     # "apartment" | "single_family" | "commercial"
total_area_m2 = 180
floor_count = 1
floor_height_m = 2.7

[connectivity]
wifi_enabled = true
wifi_prefer_5ghz = true

[connectivity.cellular]
enabled = false
sim_type = "nano_sim"
apn = ""
fallback_to_wifi = true
always_on_cellular = false
alert_via_cellular = true
stream_via_cellular = false

[connectivity.bluetooth]
ble_direct_enabled = true

[mesh]
protocol = "thread"                # "ble" | "thread"
heartbeat_interval_ms = 100
message_signing = true

[mesh.time_sync]
mode = "connection_event"
sync_interval_ms = 100

[data]
store_raw_video = false
store_raw_audio = false
store_raw_point_clouds = true
storage_target = "nvme_ssd"
retention_days = 30
max_storage_mb = 512000
include_integrity_chain = true

[slam]
enabled = false
framework = "fast_lio2"
resolution_m = 0.05

[360_processing]
enabled = false
stitch_on_edge = true
output_resolution = "3840x1920"

[policy]
default_mode = "PrivacyFirst"
auto_away_after_minutes = 30

[security]
api_authentication = "bearer"
bearer_token = ""
tls_enabled = false

[companion_app]
enabled = true
api_port = 8080
media_port = 9090
enable_remote_access = false
```

---

## 20. Summary

**The Edge Controller is the brain and gateway of AEGIS-MESH:**

| Aspect | Implementation |
|--------|-----------------|
| **Sole external gateway** | WiFi + Cellular + External BLE |
| **Compute hub** | Runs all fusion, SLAM, API |
| **Storage manager** | Recordings, evidence, SLAM maps |
| **Mesh coordinator** | Time master, message routing |
| **Policy engine** | Privacy/Security/Away modes |
| **API server** | Companion app interface |
| **Power** | AC-powered (not battery constrained) |
| **Form factor** | Fixed installation (not wearable) |

**Key architectural principle:** All external connectivity is concentrated at the Edge Controller. Sensor nodes have NO external network access — they communicate only via mesh radio to the Edge Controller.

---

**End of Document**
