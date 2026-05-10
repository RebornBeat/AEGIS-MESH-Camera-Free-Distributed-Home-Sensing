# Edge Controller — Primary Compute and Network Hub

**Project:** AEGIS-MESH
**Node Position:** Central hub (home server closet, shelf, or dedicated enclosure)
**Primary Role:** Compute hub, fusion center, **sole external network gateway**, API server, recording manager, SLAM processor
**Status:** Reference Design — All Variants
**License:** CERN-OHL-S v2

---

## 1. Role in the Mesh — The Sole External Gateway

### 1.1 Critical Architectural Constraint

**Only the Edge Controller connects to external networks.** This is not a configuration choice — it is an architectural constraint enforced at both hardware and firmware levels.

**All other nodes (Always-On, LiDAR, Identity) communicate exclusively via mesh radio to the edge controller.** They have no WiFi, no cellular, and no direct connection to the companion app.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    AEGIS-MESH Connectivity Model                             │
│                                                                              │
│  Always-On Nodes ──┐                                                         │
│  LiDAR Nodes───────┤── Mesh Radio ──► Edge Controller ──► WiFi ──► App      │
│  Identity Nodes────┘  (BLE/Thread/PoE)       │                               │
│                                              ├──► Cellular ──► Remote App    │
│                                              └──► BLE direct ──► Local App  │
│                                                                              │
│  ⚠️  EDGE CONTROLLER IS THE ONLY NODE WITH WiFi / Cellular / BLE (to app)   │
│  ⚠️  ALL OTHER NODES = MESH RADIO ONLY (no external connectivity)            │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 1.2 Why This Architecture?

| Constraint | WiFi on Sensing Node | BLE on Sensing Node | Mesh Radio on Sensing Node |
|------------|---------------------|---------------------|---------------------------|
| **Power** | 300-1000+ mW active | 5-15 mW active | 5-30 mW active |
| **Heat** | Significant thermal load | Minimal | Minimal |
| **Form factor impact** | Requires larger enclosure, antenna space | Minimal | Minimal |
| **Installation flexibility** | Requires WiFi coverage at every node | Requires proximity to gateway | Flexible placement |
| **Cost per node** | Higher | Lower | Lower |

**Conclusion:** WiFi/cellular on every sensing node would increase power, thermal load, cost, and installation complexity without meaningful benefit. Concentrating external connectivity at the edge controller is the correct architecture.

### 1.3 Primary Functions

| Function | Description |
|----------|-------------|
| **Compute Hub** | Runs fusion algorithms (`aegis-fusion`), predictive tracking (`aegis-tracking`), SLAM (`aegis-slam`), and all system coordination |
| **Fusion Center** | Aggregates detections from all nodes, performs multi-node fusion using JPDA and Covariance Intersection |
| **SLAM Processor** | Builds dense 3D world model from LiDAR + camera data (Variants B/C) |
| **API Server** | Runs embedded HTTP/WebSocket server for companion app |
| **Recording Manager** | Aggregates recording events from all nodes, manages storage, generates legal exports |
| **WiFi Gateway** | Primary companion app connection, media streaming, remote access |
| **Cellular Gateway** | Optional LTE/5G for remote access when home network unavailable |
| **BLE Direct** | Fallback companion app connection (alerts and metadata only) |
| **Time Master** | Synchronizes clocks across all mesh nodes |

---

## 2. Physical Characteristics

### 2.1 Form Factor Options

| Form Factor | Description | Thermal Management | Use Case |
|-------------|-------------|-------------------|----------|
| **Shelf Unit** | Small box on shelf or in closet | Passive convection | Residential deployment |
| **Wall Mount** | Enclosure with mounting bracket | Passive convection | Limited space, utility room |
| **Rack Mount** | 1U or 2U rack enclosure | Active cooling fans | Commercial/industrial deployment |
| **Outdoor Enclosure** | IP66 rated, temperature controlled | Active cooling, heater | Exterior deployment, remote locations |

### 2.2 Power Requirements

| Variant | Power Source | Power Draw (Active) | Notes |
|---------|--------------|---------------------|-------|
| A — SBC | USB-C PD or 5V barrel | 5-15 W | No active cooling required |
| B — ARM with NPU | 12V barrel or PoE+ | 10-25 W | May require active cooling under load |
| C — x86 Mini PC | 12-19V barrel | 15-65 W | Active cooling mandatory |

### 2.3 Environmental

| Parameter | Specification |
|-----------|---------------|
| Operating temperature | 0°C to +45°C (indoor), -20°C to +50°C (outdoor variant) |
| Humidity | 10-90% non-condensing |
| Enclosure rating | IP20 (indoor), IP66 (outdoor variant) |
| Cooling | Passive for SBC, active for high-performance variants |

---

## 3. Hardware Variants

### Variant A — SBC Class (Raspberry Pi 4/5)

**Target Use Case:** Residential deployment, cost-conscious, standard sensing load

**Compute Platform:**

| Component | Specification |
|-----------|---------------|
| SoC | Broadcom BCM2711 (Pi 4) or BCM2712 (Pi 5) |
| CPU | Quad-core Cortex-A72 @ 1.5 GHz (Pi 4) or Cortex-A76 @ 2.4 GHz (Pi 5) |
| RAM | 4 GB / 8 GB options |
| Storage | microSD or NVMe via HAT (Pi 5) |
| WiFi | 802.11ac (built-in) |
| Bluetooth | BLE 5.0 (built-in) |
| Ethernet | Gigabit |
| USB | USB 3.0 ports for cellular module |
| Power | USB-C PD (5V 3A) |

**Capabilities:**

| Capability | Support Level |
|------------|---------------|
| Sparse tracking (PentaTrack) | ✅ Full support |
| Multi-node fusion | ✅ Full support (up to 20 nodes) |
| Dense SLAM | ⚠️ Limited (2-4 LiDAR + camera nodes) |
| 360° stitching | ⚠️ Limited quality (CPU-bound) |
| Multiple video streams | ⚠️ 2-4 streams at 1080p |
| Neural inference | ❌ Not practical (no NPU) |

**Recommended for:**
- Homes with 5-15 nodes
- Sparse tracking as primary mode
- Limited SLAM requirements
- Budget-conscious deployment

**Estimated cost:** $50-100 (SBC) + $20-50 (enclosure, power)

---

### Variant B — ARM with NPU (Jetson Nano/Orin Class)

**Target Use Case:** Full-featured residential, commercial, industrial

**Compute Platform Options:**

| Option | SoC | CPU | GPU | NPU | RAM | Power |
|--------|-----|-----|-----|-----|-----|-------|
| Jetson Nano | Tegra X1 | 4× Cortex-A57 | 128-core Maxwell | — | 4 GB | 10-15 W |
| Jetson Orin Nano | Orin | 6× Cortex-A78 | Ampere | 40 TOPS | 8 GB | 15-25 W |
| Jetson Orin NX | Orin | 8× Cortex-A78 | Ampere | 100 TOPS | 16 GB | 20-40 W |
| Raspberry Pi 5 + Coral TPU | BCM2712 + Edge TPU | 4× Cortex-A76 | VideoCore VII | 4 TOPS | 8 GB | 10-20 W |

**Capabilities:**

| Capability | Support Level |
|------------|---------------|
| Sparse tracking (PentaTrack) | ✅ Full support |
| Multi-node fusion | ✅ Full support (up to 50+ nodes) |
| Dense SLAM | ✅ Full support |
| 360° stitching | ✅ Full support (GPU-accelerated) |
| Multiple video streams | ✅ 8-16 streams at 1080p |
| Neural inference | ✅ Full support (NPU-accelerated) |
| Real-time object recognition | ✅ Full support |

**Additional Features:**
- Hardware video encoding/decoding (NVENC/NVDEC)
- CUDA support for custom algorithms
- On-device model training (Orin variants)

**Recommended for:**
- Homes with 10-30 nodes
- Dense SLAM as standard mode
- Multiple 360° camera streams
- Commercial/industrial deployment
- Real-time neural inference requirements

**Estimated cost:** $200-500 (module/carrier) + $50-100 (enclosure, cooling)

---

### Variant C — x86 Mini PC

**Target Use Case:** Maximum flexibility, multi-tenant, commercial, research

**Compute Platform Options:**

| Option | CPU | RAM | Storage | Power |
|--------|-----|-----|---------|-------|
| Intel NUC 11/12 | i5/i7, 4-6 cores | 16-64 GB | NVMe SSD | 15-35 W |
| Intel NUC Pro | i5/i7, 6-8 cores | 32-128 GB | Dual NVMe | 35-65 W |
| AMD Mini PC | Ryzen 5/7 | 16-64 GB | NVMe SSD | 25-45 W |
| Custom 1U Server | Xeon/EPYC | 64-256 GB | RAID NVMe | 45-150 W |

**Capabilities:**

| Capability | Support Level |
|------------|---------------|
| Sparse tracking | ✅ Full support |
| Multi-node fusion | ✅ Unlimited |
| Dense SLAM | ✅ Full support (highest quality) |
| 360° stitching | ✅ Full support (CPU/GPU) |
| Multiple video streams | ✅ Unlimited |
| Neural inference | ✅ Full support (with GPU add-on) |
| Docker/Kubernetes | ✅ Full support |
| Multi-tenant | ✅ Multiple AEGIS-MESH instances |
| External GPU | ✅ Via Thunderbolt/PCIe |

**Additional Features:**
- Full Linux distribution flexibility
- Docker containerization
- RAID storage for reliability
- Hardware TPM for secure boot
- BMC for remote management (server variants)

**Recommended for:**
- Large-scale residential (30+ nodes)
- Commercial/industrial deployment
- Multi-property management
- Research platform
- Maximum flexibility requirements

**Estimated cost:** $300-800 (mini PC) + $100-200 (enclosure, UPS)

---

### Variant D — PoE-Powered Industrial

**Target Use Case:** Industrial environments, commercial buildings, locations with PoE infrastructure

**Compute Platform:**

| Component | Specification |
|-----------|---------------|
| MCU + SBC combo | STM32H7 (real-time mesh coordination) + Linux SBC (compute) |
| Power | PoE 802.3at (25.5 W) or PoE++ (51 W) |
| Enclosure | IP66, industrial rated |
| Temperature | -20°C to +60°C operating |
| Cooling | Sealed enclosure, conduction cooling |

**Additional Features:**
- Industrial I/O (Modbus, CAN bus for building integration)
- Conformal coating on PCB
- Wide input voltage tolerance
- Watchdog timer with auto-restart

**Recommended for:**
- Factory floor monitoring
- Warehouse security
- Commercial building automation
- Outdoor installation

---

## 4. Connectivity Architecture

### 4.1 WiFi Interface

**The primary companion app connection.**

| Specification | Value |
|---------------|-------|
| Standard | 802.11 a/b/g/n/ac/ax (depending on SBC) |
| Purpose | Companion app, media streaming, SLAM data transfer, API access |
| Frequency bands | 2.4 GHz and 5 GHz |
| Recommendation | **Prefer 5 GHz band** to avoid BLE interference |
| Power draw | 300-500 mW when streaming |

**WiFi is present ONLY on the edge controller.** No sensing node has WiFi capability.

### 4.2 Cellular / SIM Interface

**The secondary external connection for remote access.**

**Purpose:**
- Remote access when home network unavailable
- Away-mode alerts
- Emergency notifications
- Live streaming over cellular (with 5G module)

**Hardware Interface:**

| Interface | Description |
|-----------|-------------|
| SIM slot | nano-SIM (4FF) or eSIM footprint |
| Module connection | UART (Cat 1/4) or USB 3.x (5G) |
| Antenna | U.FL connector for external cellular antenna |
| Optional GNSS | From cellular module if supported |

**Supported Cellular Modules:**

| Module | Technology | Speed | Interface | Power (Active) | Use Case |
|--------|------------|-------|-----------|----------------|----------|
| Quectel EC21 | LTE Cat 1 | 5-10 Mbps | UART | 150-300 mA | Alerts, metadata, minimal data |
| Quectel EC25 | LTE Cat 4 | 50-150 Mbps | UART/USB | 400-600 mA | Compressed video clips |
| Quectel RM502Q-AE | 5G Sub-6GHz | 100-1000+ Mbps | USB 3.x | 800-1200 mA | Live video streaming, 360° relay |
| Quectel EG912Y-GL | LTE Cat 12 | 300+ Mbps | USB | 500-800 mA | Multi-carrier, eSIM support |
| Sierra Wireless RV55 | LTE/5G | Variable | USB | Variable | Industrial grade |
| Telit ME910G1 | LTE Cat M1 | 1 Mbps | UART | 100-200 mA | Low-power IoT mode |

**SIM Interface Signals:**

| Signal | Description |
|--------|-------------|
| `SIM_CLK` | SIM card clock |
| `SIM_DATA` | SIM card data |
| `SIM_RST` | SIM card reset |
| `SIM_VCC` | SIM card power (1.8V or 3.0V selectable) |
| `SIM_DET` | SIM card present detect |

**Antenna Requirements:**
- External antenna required (cellular frequencies 700 MHz - 3.5 GHz, 24-47 GHz for 5G mmWave)
- U.FL connector on PCB
- Antenna mounted on enclosure exterior or connected via bulkhead connector

**Cellular is present ONLY on the edge controller.** No sensing node has cellular capability.

### 4.3 Bluetooth / BLE

**Three distinct roles:**

**Role 1: Mesh Radio to Sensing Nodes (via Thread or custom protocol)**
- Not BLE for this role (typically Thread/802.15.4 or proprietary mesh)
- BLE may be used if nodes are BLE-based

**Role 2: Direct Companion App Connection**
- BLE 5.x connection to companion app
- Fallback when WiFi unavailable
- Bandwidth limited: alerts, metadata, configuration only
- No video streaming over BLE

**Role 3: Device Commissioning**
- Initial node pairing
- Firmware updates (for small firmware images)

**Configuration:**

```toml
[connectivity.bluetooth]
ble_direct_enabled = true
ble_advertising_name = "aegis-mesh"
ble_pairing_mode = "secure"
```

### 4.4 Ethernet

**Primary wired interfaces:**

| Interface | Speed | Purpose |
|-----------|-------|---------|
| Primary Ethernet | 1 Gbps | Network backbone, high-bandwidth node communication |
| Secondary Ethernet (optional) | 1 Gbps | PoE aggregator, dedicated media network |
| PoE Input | 802.3af/at | Power + data from PoE switch |

**For LiDAR nodes with PoE:** PoE Ethernet from LiDAR nodes terminates at the edge controller (either via switch or directly). The edge controller aggregates all PoE node data.

### 4.5 USB

| Interface | Purpose |
|-----------|---------|
| USB-C | Companion app connection (PC), firmware updates, charging |
| USB 3.x | Cellular module connection, external storage |

### 4.6 Connectivity Summary Table

| Interface | Direction | Purpose | Bandwidth |
|-----------|-----------|---------|-----------|
| WiFi | To companion app | Primary app connection, streaming | 50-300+ Mbps |
| Cellular | To companion app | Remote access, alerts | 5-1000+ Mbps |
| BLE | To companion app | Direct connection, fallback | 1-2 Mbps |
| Ethernet | To sensing nodes | PoE LiDAR nodes, backbone | 1 Gbps |
| USB | To PC/devices | Admin, updates, storage | 5-10 Gbps |

---

## 5. Compute Architecture

### 5.1 Software Stack

All edge controller variants run a common Rust-based software stack:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         AEGIS-MESH Edge Controller Stack                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                    Companion App Interface                           │    │
│  │                                                                      │    │
│  │  HTTP/REST API        WebSocket Events        Media Streams         │    │
│  │  (aegis-api)          (aegis-api)              (RTSP/WebSocket)      │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                              │                                               │
│                              ▼                                               │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                    Core Processing Layer                             │    │
│  │                                                                      │    │
│  │  Multi-Node Fusion     PentaTrack Tracking    SLAM Engine           │    │
│  │  (aegis-fusion)        (aegis-tracking)        (aegis-slam)         │    │
│  │                                                                      │    │
│  │  Placement Optimizer   Calibration Engine    Policy Engine          │    │
│  │  (aegis-placement)     (aegis-calibration)    (aegis-policy)        │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                              │                                               │
│                              ▼                                               │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                    Mesh Protocol Layer                               │    │
│  │                                                                      │    │
│  │  Message Signing       Time Synchronization    Health Monitoring    │    │
│  │  (HMAC-SHA256)         (Time Master)            (Heartbeat)          │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                              │                                               │
│                              ▼                                               │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                    Storage Layer                                     │    │
│  │                                                                      │    │
│  │  Recording Manager     Integrity Chain          Retention Policy     │    │
│  │  (aegis-storage)       (SHA-256 chain)           (user-configured)   │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 5.2 Crate Mapping

| Crate | Responsibility | Resource Requirement |
|-------|----------------|----------------------|
| `aegis-core` | Shared types, events, config | Minimal |
| `aegis-perception` | Per-node sensor pipelines | Low |
| `aegis-fusion` | Multi-node detection fusion | Medium |
| `aegis-tracking` | PentaTrack bridge | Medium |
| `aegis-slam` | Dense SLAM | High (GPU/NPU recommended) |
| `aegis-placement` | Node placement optimization | Medium (runs at config time) |
| `aegis-calibration` | Calibration algorithms | Low |
| `aegis-policy` | Policy engine | Low |
| `aegis-mesh-protocol` | Mesh protocol implementation | Low |
| `aegis-storage` | Recording management | Medium |
| `aegis-api` | REST/WebSocket API server | Medium |
| `aegis-alerts` | Alert routing | Low |
| `aegis-edge-controller` | Main binary | — |

### 5.3 Resource Requirements by Variant

| Resource | Variant A (SBC) | Variant B (NPU) | Variant C (x86) |
|----------|-----------------|-----------------|------------------|
| **CPU cores** | 4 | 6-8 | 6-16 |
| **RAM** | 4-8 GB | 8-16 GB | 16-128 GB |
| **Storage** | microSD (64 GB+) | NVMe SSD (256 GB+) | NVMe RAID (1 TB+) |
| **GPU** | None / VideoCore | Ampere (CUDA) | Optional discrete |
| **NPU** | None | 40-100 TOPS | Optional add-on |
| **Simultaneous video streams** | 2-4 | 8-16 | Unlimited |
| **SLAM quality** | Limited | Full | Maximum |
| **Recommended node count** | 5-15 | 10-30 | 30+ |

---

## 6. Mesh Radio Hub Function

### 6.1 Role as Mesh Coordinator

The edge controller serves as the mesh coordinator for all sensing nodes:

**Functions:**
- **Time Master:** All nodes synchronize to the edge controller's clock
- **Message Router:** Routes mesh messages between nodes and to the API layer
- **Heartbeat Monitor:** Tracks node health via periodic heartbeats
- **Discovery Server:** New nodes register with the edge controller
- **Configuration Distribution:** Pushes config changes to nodes

**Time Synchronization:**

```
Edge Controller (Time Master)
       │
       ├── Periodic timestamp broadcast (every 100 ms)
       │
       └── Nodes track clock offset using ClockOffsetEstimator
           (PTP-style four-timestamp exchange)
```

**Heartbeat Protocol:**

```toml
[mesh.heartbeat]
interval_ms = 100                # Every 100 ms
missed_heartbeat_threshold = 3   # Flag offline after 3 missed
auto_rejoin_timeout_s = 30       # Auto-rejoin after 30 seconds
```

### 6.2 Mesh Protocol Implementation

**Supported mesh transports:**

| Transport | Speed | Range | Power (node side) | Use Case |
|-----------|-------|-------|-------------------|----------|
| BLE 5.x / Thread | 1-2 Mbps | 10-30 m | 5-15 mW | Standard nodes |
| PoE Ethernet | 1 Gbps | Cable | Powered | LiDAR nodes |
| 802.15.4 (Thread) | 250 Kbps | 30-100 m | 20-50 mW | Extended range |
| Wi-Fi mesh (ESP32 nodes) | 50+ Mbps | 30-50 m | 300-500 mW | High-bandwidth nodes |

**Mesh protocol features:**
- HMAC-SHA256 message signing
- Sequence numbers (replay protection)
- Encryption (AES-128) for sensitive data
- Priority queuing (alerts > detections > telemetry)

---

## 7. Storage Architecture

### 7.1 Local Storage Options

| Storage Type | Capacity | Interface | Use Case |
|--------------|----------|-----------|----------|
| microSD card | Up to 2 TB | SDIO | Basic deployment, removable |
| NVMe SSD | 256 GB - 4 TB | PCIe | High-performance, variants B/C |
| SATA SSD | 1-8 TB | SATA | Bulk storage, server variants |
| USB storage | Variable | USB 3.x | Backup, archive, portable |
| Network storage (NAS) | Variable | Ethernet | Offload, multi-controller |

### 7.2 Storage Configuration

```toml
[data]
# Storage targets
storage_target = "nvme_ssd"       # "sd_card" | "nvme_ssd" | "sata_ssd" | "network_path"

# Retention
retention_days = 90               # 0 = keep forever
max_storage_mb = 500000           # 500 GB limit (0 = unlimited)

# Data types to store
store_raw_video = true
store_raw_audio = false
store_raw_sensor_data = false
store_slam_maps = true

# Recording triggers
continuous_recording = false
recording_trigger = "on_detection"  # "always" | "on_detection" | "on_alert" | "manual"

# Integrity
include_integrity_chain = true
```

### 7.3 Storage Performance Requirements

| Use Case | Minimum Write Speed | Recommended |
|----------|---------------------|-------------|
| Metadata only | 10 MB/s | microSD Class 10 |
| Single 1080p video | 30 MB/s | microSD UHS-I / NVMe |
| Multiple 1080p videos | 100 MB/s | NVMe SSD |
| 4K video | 150 MB/s | NVMe SSD |
| 360° 4K panoramic | 200+ MB/s | NVMe SSD |

---

## 8. Power Architecture

### 8.1 Power Requirements by Variant

| Variant | Idle Power | Active Power | Peak Power |
|---------|------------|--------------|------------|
| A — SBC | 3-5 W | 8-15 W | 20 W |
| B — NPU | 5-10 W | 15-30 W | 45 W |
| C — x86 | 10-20 W | 25-60 W | 100 W |
| D — Industrial | 5-12 W | 12-25 W | 35 W |

### 8.2 Power Sources

| Source | Applicable Variants | Notes |
|--------|---------------------|-------|
| USB-C PD | A, B | Simple installation |
| 12V barrel | B, C | Higher power capacity |
| PoE 802.3at | A, D | 25.5 W maximum |
| PoE++ (802.3bt) | B, D | 51 W (industrial variants) |
| AC adapter | C (x86) | Dedicated power supply |

### 8.3 Battery Backup (Optional)

For deployments requiring uninterruptible operation:

| Backup Type | Capacity | Runtime (typical) | Applicability |
|-------------|----------|-------------------|---------------|
| Internal UPS (mini UPS HAT) | 5000-10000 mAh | 30-60 minutes | Variant A |
| External UPS | 100-500 Wh | 2-8 hours | All variants |
| Generator | Unlimited | Continuous | Critical infrastructure |

```toml
[power.backup]
enabled = false
type = "external_ups"             # "internal" | "external_ups" | "generator"
shutdown_on_battery = true
shutdown_threshold_percent = 15
grace_period_s = 300              # 5 minutes before shutdown
```

---

## 9. Thermal Management

### 9.1 Thermal Requirements by Variant

| Variant | TDP | Cooling Requirement | Typical Configuration |
|---------|-----|---------------------|----------------------|
| A — SBC | 5-15 W | Passive | Heatsink, ventilated enclosure |
| B — NPU | 15-35 W | Active (recommended) | Small fan, heatsink |
| C — x86 | 35-100 W | Active (mandatory) | Multiple fans, possibly liquid |
| D — Industrial | 10-25 W | Passive or sealed | Conduction to enclosure |

### 9.2 Thermal Monitoring

```toml
[thermal]
monitoring_enabled = true
sensor_type = "ntc_thermistor"

[thermal.thresholds]
warning_c = 45                    # Log warning
throttle_c = 55                   # Reduce SLAM frame rate
critical_c = 65                   # Shutdown

[thermal.throttle_actions]
reduce_slam_rate = true
reduce_video_quality = true
disable_360_stitching = true
```

### 9.3 Cooling Solutions

| Solution | Noise Level | Maintenance | Cost |
|----------|-------------|-------------|------|
| Passive heatsink | Silent | None | $5-20 |
| Small fan (40mm) | Low (20-30 dB) | Occasional cleaning | $10-30 |
| Multiple fans | Medium (35-45 dB) | Regular cleaning | $20-50 |
| Liquid cooling | Low (pump only) | Annual check | $50-150 |

---

## 10. API Server Architecture

### 10.1 Embedded Server

The edge controller runs an embedded HTTP/WebSocket server:

**Framework:** Rust `axum` or equivalent async web framework

**Ports:**

| Port | Protocol | Purpose |
|------|----------|---------|
| 8080 | HTTP | REST API |
| 8080 | WebSocket | Real-time event stream |
| 9090 | RTSP | Media streams |
| 9091 | WebSocket (H.264) | Browser-compatible video |
| 9092 | WebSocket (360°) | 360° panoramic streams |
| 9093 | WebSocket (audio) | Audio streams |

### 10.2 REST API Endpoints

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/nodes` | List all nodes with health status |
| GET | `/api/nodes/{id}` | Single node details |
| GET | `/api/zones` | List all zones |
| GET | `/api/zones/{id}` | Single zone details |
| GET | `/api/tracks` | Current active tracks |
| GET | `/api/tracks/stream` | WebSocket: live track updates |
| GET | `/api/alerts` | Alert history (paginated) |
| GET | `/api/alerts/stream` | WebSocket: live alerts |
| POST | `/api/alerts/emergency` | Trigger emergency contact notification |
| GET | `/api/recordings` | List stored recordings |
| GET | `/api/recordings/{id}` | Download recording |
| GET | `/api/recordings/{id}/meta` | Integrity manifest |
| DELETE | `/api/recordings/{id}` | Delete recording |
| GET | `/api/export/{id}` | Export recording with integrity chain |
| GET | `/api/nodes/{id}/stream` | Open live media stream |
| GET | `/api/world-model` | Current world model state |
| GET | `/api/world-model/3d` | 3D SLAM mesh (if available) |
| GET | `/api/world-model/replay/{timestamp}` | World model at past timestamp |
| GET | `/api/analytics/zones` | Zone activity statistics |
| GET | `/api/analytics/coverage` | Coverage analysis |
| GET | `/api/config` | Current configuration |
| POST | `/api/config` | Update configuration |
| POST | `/api/policy/mode` | Change policy mode |
| GET | `/api/connectivity` | Current connectivity status |
| GET | `/api/connectivity/cellular` | Cellular status and signal |
| POST | `/api/connectivity/cellular/toggle` | Enable/disable cellular |
| GET | `/api/health` | System health summary |

### 10.3 WebSocket Event Types

| Path | Event Types |
|------|-------------|
| `/ws/events` | All mesh events |
| `/ws/detections` | DetectionEvent |
| `/ws/alerts` | AlertEvent |
| `/ws/tracks` | TrackUpdateEvent |
| `/ws/world-model` | WorldModelUpdate |
| `/ws/nodes` | NodeHealthEvent |

---

## 11. Recording Management

### 11.1 Recording Sources

The edge controller aggregates recordings from multiple sources:

| Source | Storage Location | Content |
|--------|------------------|---------|
| Identity nodes | Node SD card → Edge aggregated | Raw video, clips, images |
| LiDAR nodes (with camera) | Node SD card → Edge aggregated | Point clouds, video |
| Always-On nodes | Node SD card | Audio clips, sensor data |
| Edge controller local | Internal storage | SLAM maps, world model |

### 11.2 Recording Lifecycle

```
Detection Event
       │
       ▼
Recording Triggered (per configuration)
       │
       ▼
Node captures to local SD card
       │
       ▼
Node sends RecordingAvailable event
       │
       ▼
Edge controller indexes recording
       │
       ▼
Edge controller computes SHA-256
       │
       ▼
Edge controller adds to integrity chain
       │
       ▼
Recording available via API
       │
       ▼
Retention policy enforcement (daily)
```

### 11.3 Integrity Chain

**Format:**

```json
{
  "export_version": "1.0",
  "node_id": "identity_node_entry",
  "node_class": "IdentityNode",
  "zone_id": "front_entry",
  "recording_start_utc": "2024-03-15T14:30:22.004Z",
  "recording_duration_s": 42.3,
  "format": "h264_mp4",
  "modalities": ["video"],
  "file_sha256": "a3f4b2c9d8e1f7a2b5c8d3e6f9a0b1c2d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8",
  "firmware_version": "v1.2.0",
  "hardware_variant": "identity_node_v1",
  "export_timestamp_utc": "2024-03-16T09:00:00.000Z",
  "chain_hash": "e3f5a7b9c1d2e4f6a8b0c2d4e6f8a0b2"
}
```

**Chain construction:**

```
chain_hash_n = SHA256(chain_hash_{n-1} || file_sha256_n || timestamp_n)
```

---

## 12. SLAM Processing

### 12.1 SLAM Requirements

**Dense SLAM requires:**
- LiDAR node with camera (Variant C or D)
- Sufficient edge controller compute (Variant B or C)

| SLAM Mode | Compute Requirement | Edge Variant |
|-----------|---------------------|--------------|
| Single room | 2-4 TOPS | Variant B (NPU) |
| Multi-room | 8-20 TOPS | Variant B (high-end) |
| Large building | 20+ TOPS | Variant C (x86 + GPU) |

### 12.2 SLAM Pipeline

```
LiDAR Point Clouds ──┐
                     │
Camera Keyframes ────┼──► SLAM Backend ──► 3D Mesh ──► World Model
                     │
IMU Data ────────────┘
```

**SLAM algorithms supported:**
- LIO-SAM (LiDAR-Inertial Odometry)
- FAST-LIO2
- ORB-SLAM3 (with camera)
- RTAB-Map

### 12.3 SLAM Configuration

```toml
[slam]
enabled = true
algorithm = "lio_sam"             # "lio_sam" | "fast_lio" | "orb_slam3" | "rtab_map"

[slam.lidar_nodes]
required_nodes = ["lidar_main_room", "lidar_kitchen"]

[slam.camera_nodes]
required_nodes = ["lidar_main_room"]   # LiDAR variant C/D with camera

[slam.output]
store_dense_mesh = true
mesh_resolution_m = 0.05          # 5 cm
keyframe_interval_s = 1.0
```

---

## 13. Cellular Module Integration

### 13.1 Hardware Integration

**Physical interface:**

| Signal | MCU Pin | Description |
|--------|---------|-------------|
| `CELL_TX` | UART TX | Data to cellular module |
| `CELL_RX` | UART RX | Data from cellular module |
| `CELL_RST` | GPIO | Module reset |
| `CELL_PWR` | GPIO | Power enable |
| `CELL_STATUS` | GPIO | Module status indicator |
| `USB_D+/-` | USB | USB data for high-speed modules |

**Antenna:**
- External antenna required
- U.FL connector on PCB
- Mount on enclosure exterior

**SIM:**
- nano-SIM slot (4FF) on enclosure exterior for easy access
- Or eSIM (soldered, multi-carrier)

### 13.2 Power Considerations

| Module | Active Current | Peak Current | Recommended Power Supply |
|--------|---------------|--------------|--------------------------|
| EC21 (LTE Cat 1) | 150-300 mA | 500 mA | 5V 2A sufficient |
| EC25 (LTE Cat 4) | 400-600 mA | 1.5 A | 5V 3A recommended |
| RM502Q-AE (5G) | 800-1200 mA | 2.5 A | Dedicated 5V 5A rail |

**Design note:** For 5G modules, provide dedicated power rail with sufficient headroom for transmit bursts.

### 13.3 Cellular Configuration

```toml
[connectivity.cellular]
enabled = false
sim_type = "nano_sim"            # "nano_sim" | "esim"
apn = "your.carrier.apn"
fallback_to_wifi = true          # Prefer WiFi when available
always_on_cellular = false       # Keep cellular active even with WiFi

# Data usage controls
alert_via_cellular = true        # Send alerts via cellular when active
stream_via_cellular = false      # Stream video over cellular (data cost)
compressed_clips_via_cellular = true  # Send compressed clips

# Emergency contact
emergency_contact = ""           # Phone/endpoint for emergency alerts
emergency_trigger = "manual"     # "manual" | "fall_detected" | "intrusion_detected"
```

---

## 14. Security Architecture

### 14.1 Physical Security

| Feature | Specification |
|---------|---------------|
| Tamper detection | Optional tamper switch on enclosure |
| Secure boot | TPM or MCU secure boot (variant dependent) |
| Encrypted storage | LUKS or equivalent for SSD |
| Key storage | TPM or secure element |

### 14.2 Network Security

| Feature | Default |
|---------|---------|
| API authentication | Bearer token (user-set) |
| TLS | Optional for local, mandatory for remote |
| Message signing | HMAC-SHA256 on all mesh messages |
| Firewall | Built-in (Linux iptables/nftables) |

### 14.3 Configuration

```toml
[security]
api_authentication = true
api_token = ""                   # Set by user
tls_enabled = false              # Enable for remote access
tls_cert_path = "/etc/aegis/tls/cert.pem"
tls_key_path = "/etc/aegis/tls/key.pem"

[security.mesh]
message_signing = true
encryption_enabled = true
```

---

## 15. Mechanical Specifications

### 15.1 Enclosure Dimensions

| Form Factor | Dimensions | Volume | Mounting |
|-------------|------------|--------|----------|
| Shelf unit | 150 × 100 × 40 mm | 0.6 L | Rubber feet |
| Wall mount | 180 × 120 × 45 mm | 1.0 L | 4× wall anchors |
| 1U rack | 482 × 200 × 44 mm | 4.2 L | Rack rails |
| Outdoor | 200 × 150 × 80 mm | 2.4 L | Pole/wall mount |

### 15.2 Ventilation

| Variant | Ventilation Type | Airflow |
|---------|------------------|---------|
| A — SBC | Passive slots | Natural convection |
| B — NPU | Active fan | 20-40 CFM |
| C — x86 | Multiple fans | 40-100 CFM |
| D — Industrial | Sealed (IP66) | Conduction cooling |

### 15.3 LED Indicators

| LED | Color | Function |
|-----|-------|----------|
| Power | Green | Power on |
| Mesh Activity | Blue | Mesh traffic |
| Alert | Red | Active alert |
| Storage | Yellow | Storage activity |
| Cellular | Cyan | Cellular active |

---

## 16. Installation

### 16.1 Typical Residential Installation

1. Mount edge controller in central location (utility closet, shelf)
2. Connect to home router via Ethernet (or use WiFi)
3. Install optional cellular SIM and antenna
4. Connect power
5. Access companion app at `aegis-mesh.local` or configured IP
6. Run walk-through calibration

### 16.2 PoE Aggregator Configuration (for LiDAR nodes)

If using PoE LiDAR nodes:

```
PoE Switch
    │
    ├── Edge Controller (management port)
    │
    ├── LiDAR Node 1 (PoE port 1)
    │
    ├── LiDAR Node 2 (PoE port 2)
    │
    └── ... (additional LiDAR nodes)
```

The edge controller receives all LiDAR data via the switch's management network.

---

## 17. Configuration Reference

```toml
# aegis-mesh.toml — Edge Controller Configuration

[deployment]
building_name = "Home"
floor_count = 1
total_area_m2 = 150.0

[connectivity]
wifi_enabled = true
wifi_ssid = "YourHomeNetwork"
wifi_password = ""               # Set via secure method
wifi_prefer_5ghz = true

[connectivity.cellular]
enabled = false
sim_type = "nano_sim"
apn = ""
fallback_to_wifi = true
always_on_cellular = false
alert_via_cellular = true
stream_via_cellular = false
emergency_contact = ""

[mesh_network]
protocol = "thread"             # "thread" | "ble" | "wifi_mesh"
edge_controller_address = "192.168.1.100:8080"
heartbeat_interval_ms = 100
message_signing = true

[data]
store_raw_video = true
store_raw_audio = false
store_raw_sensor_data = false
storage_target = "nvme_ssd"
retention_days = 90
max_storage_mb = 500000
include_integrity_chain = true

[slam]
enabled = true
algorithm = "lio_sam"

[policy]
default_mode = "PrivacyFirst"
auto_away_after_minutes = 30
alert_cooldown_s = 60.0

[companion_app]
enabled = true
api_port = 8080
media_port = 9090
enable_remote_access = false
remote_access_token = ""

[thermal]
monitoring_enabled = true
warning_threshold_c = 45
throttle_threshold_c = 55
critical_threshold_c = 65

[security]
api_authentication = true
tls_enabled = false
```

---

## 18. Testing and Validation

### 18.1 Hardware Validation

| Test | Method | Acceptance Criteria |
|------|--------|---------------------|
| Power rails | Multimeter | All rails within ±5% |
| Temperature | Thermal camera | Idle < 40°C, load < 55°C |
| WiFi connectivity | Speed test | > 50 Mbps sustained |
| Cellular connectivity | AT command | Module responds OK |
| API server | HTTP request | All endpoints return valid response |
| Storage write | FIO benchmark | > Minimum required speed |

### 18.2 System Integration Tests

| Test | Description |
|------|-------------|
| Node discovery | Power on node; verify edge detects within 30 s |
| Time sync | Verify node clocks within 1 ms of edge |
| Heartbeat | Power off node; verify edge flags offline within 300 ms |
| Recording | Trigger detection; verify recording indexed within 5 s |
| Alert routing | Inject alert; verify correct routing |
| SLAM (if enabled) | Walk through space; verify map updates |

### 18.3 Cellular Tests

| Test | Method |
|------|--------|
| SIM detection | Verify `SIM_DET` GPIO reads present |
| Module power on | AT command returns OK within 5 s |
| Network registration | Module reports registered within 60 s |
| Data connectivity | Ping external host via cellular |
| Alert delivery | Inject alert; verify delivery to cellular endpoint |

---

## 19. Summary

The AEGIS-MESH Edge Controller is the central hub of the distributed sensing mesh, providing:

| Function | Implementation |
|----------|---------------|
| **Sole external gateway** | WiFi, cellular, BLE direct |
| **Compute hub** | Multi-node fusion, tracking, SLAM |
| **API server** | REST + WebSocket for companion app |
| **Recording manager** | Storage, retention, integrity chains |
| **Mesh coordinator** | Time sync, health monitoring, discovery |

**Variant selection guide:**

| Scenario | Recommended Variant |
|----------|-------------------|
| Small home (5-15 nodes) | A — SBC |
| Large home (10-30 nodes) | B — NPU |
| Multi-property, commercial | C — x86 |
| Industrial, outdoor | D — Industrial |

---

**End of Document**
