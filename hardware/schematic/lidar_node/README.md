# LiDAR Node — Hardware Schematic Reference Design

**Project:** AEGIS-MESH
**Node Class:** LidarNode — High-Resolution 3D Geometry and Tracking
**Status:** Reference Design — All Variants. No component is locked.
**License:** CERN-OHL-S v2

---

## 1. Connectivity Architecture — Mesh Radio Only

**The LiDAR node has NO WiFi and NO cellular connectivity.**

All LiDAR node data transmits via mesh radio (PoE Ethernet preferred for high-bandwidth variants, or BLE/Thread for lower-bandwidth variants) to the edge controller exclusively. The edge controller is the sole external network gateway.

```
LiDAR Node Sensors
       │
       │  Mesh Radio (PoE Ethernet / BLE / Thread)
       ▼
Edge Controller
       │
       ├──► WiFi → Companion App (local)
       └──► Cellular → Companion App (remote)
```

**Data flow for all variants:**
```
Sensors (LiDAR, Camera, Radar, etc.)
       │
       ▼
On-Node Processing (Optional)
       │
       ▼
Mesh Radio (PoE/BLE/Thread)
       │
       ▼
Edge Controller
       │
       ├──► Local Storage (SD/NVMe)
       └──► WiFi/Cellular → Companion App
```

**The LiDAR node never connects directly to the companion app.** All app-facing connectivity routes through the edge controller.

---

## 2. Purpose

The LiDAR Node provides high-resolution geometric sensing for volume tracking, 3D mapping, and SLAM integration. It activates on-demand (triggered by the Always-On Node's motion events) or continuously at low duty cycle for dynamic remap scanning.

Four variants range from a basic event-triggered LiDAR setup to a full 360° camera array with SLAM capability.

---

## 3. Node Variants

### Variant A — Basic Event-Triggered LiDAR

**Sensors:**
- Solid-state LiDAR (ceiling-mount)
- Event camera (fast-motion trigger — wakes LiDAR on microsecond-latency detection)

**Connectivity:** PoE Ethernet or BLE/Thread mesh to edge controller

**Purpose:** Cost-effective high-fidelity geometry. LiDAR activates only when event camera detects motion. Reduces average power consumption dramatically.

**Use case:** Standard residential room coverage where continuous LiDAR is unnecessary.

**Data flow:**
```
Event Camera → LiDAR Wake → Point Cloud → Mesh (BLE/Thread) → Edge Controller
```

### Variant B — Full LiDAR + Radar

**Sensors:**
- Solid-state LiDAR
- Event camera
- mmWave FMCW radar

**Connectivity:** PoE Ethernet or BLE/Thread mesh to edge controller

**Purpose:** Dual modality for fog/smoke/steam robustness. Radar continues to detect in conditions that degrade LiDAR. Provides velocity from Doppler alongside LiDAR geometry.

**Use case:** Kitchens (steam), bathrooms (steam), rooms with smoke or variable visibility.

**Data flow:**
```
LiDAR + Radar Point Clouds → Mesh (PoE/BLE) → Edge Controller → Fusion
```

### Variant C — LiDAR + Camera

**Sensors:**
- Solid-state LiDAR
- Event camera
- Conventional camera (configurable resolution)
- mmWave radar (optional)

**Connectivity:** PoE Ethernet (recommended for camera bandwidth)

**Purpose:** Enables visual SLAM (LiDAR provides depth; camera provides visual odometry and texture). Supports dense 3D reconstruction with RGB texture. Camera data handling fully user-configured.

**Use case:** Users wanting full SLAM capability with textured 3D maps and visual recording.

**Data flow:**
```
LiDAR Point Clouds → PoE → Edge Controller → SLAM
Camera Video → PoE → Edge Controller → Storage/Stream
                               │
                               └──► WiFi → Companion App
```

### Variant D — LiDAR + 360° Camera Array

**Sensors:**
- Solid-state LiDAR (ceiling mount)
- Event camera
- 4–8 camera array (full 360° horizontal coverage of room)
- mmWave radar

**Connectivity:** PoE Ethernet 802.3at (required for high bandwidth)

**Purpose:** Room-scale 360° video capture at ceiling position combined with LiDAR geometry. Enables:
- Full 360° visual recording of the room
- Dense SLAM with complete visual coverage (no blind spots from single-camera angle)
- Occupancy mapping with full visual context

**Use case:** Users wanting the most comprehensive room-scale documentation and SLAM capability. Equivalent to a full-room 360° security camera system with added LiDAR depth.

**Data flow:**
```
4–8 Camera Streams → On-Node ISP (Optional) → PoE → Edge Controller
                                                      │
                                                      ├──► Stitching
                                                      ├──► SLAM
                                                      ├──► Storage
                                                      └──► WiFi → Companion App
```

---

## 4. 360° Camera Array Architecture (Variant D)

### 4.1 Physical Layout

The 360° camera array consists of 4–8 camera modules distributed around the LiDAR node enclosure at ceiling height.

| Configuration | Cameras | Angular Spacing | FoV Required per Camera |
|---------------|---------|-----------------|-------------------------|
| Minimal | 4 | 90° | ≥ 100° |
| Standard | 6 | 60° | ≥ 70° |
| Dense | 8 | 45° | ≥ 55° |
| Ultra | 12 | 30° | ≥ 40° |

### 4.2 Camera Module Options

| Option | Resolution | Interface | Notes |
|--------|------------|-----------|-------|
| OV2640 | 2 MP | MIPI/DVP | Low cost, wide-angle available |
| OV5640 | 5 MP | MIPI | Autofocus option |
| IMX219 | 8 MP | MIPI CSI-2 | Raspberry Pi compatible, fisheye available |
| IMX477 | 12 MP | MIPI CSI-2 | High quality, for premium installations |

### 4.3 Wired Internal Camera Bus (Recommended Architecture)

All cameras connect via internal MIPI CSI-2 bus to a central vision processor or multiplexer on the LiDAR node:

```
Camera 0 ─┐
Camera 1 ─┤
Camera 2 ─┼── MIPI CSI-2 Multiplexer ── Vision Processor / ISP
Camera 3 ─┤        (or direct to SoM)
Camera 4 ─┤              │
Camera 5 ─┤              ├──► Local Stitching (GPU/ISP)
Camera 6 ─┤              └──► PoE → Edge Controller
Camera 7 ─┘
```

**Benefits of wired internal bus:**
- Maximum internal bandwidth (no wireless constraint)
- Hardware-synchronized capture (FSYNC)
- Single encoded stream to edge controller
- Reduced latency

### 4.4 Hardware Synchronization (FSYNC)

**Critical requirement:** All cameras must capture simultaneously for clean stitching.

**Implementation:**
- Single FSYNC GPIO line from vision processor to all camera FSYNC pins
- All cameras trigger on same rising edge
- PCB trace length matching: target < 10 ns skew between any two cameras
- Without FSYNC: cameras drift at frame rate, causing visible stitching artifacts

**FSYNC signal specification:**
```
FSYNC GPIO (from vision processor)
    │
    ├──► Camera 0 FSYNC
    ├──► Camera 1 FSYNC
    ├──► Camera 2 FSYNC
    ├──► Camera 3 FSYNC
    ├──► Camera 4 FSYNC
    ├──► Camera 5 FSYNC
    ├──► Camera 6 FSYNC
    └──► Camera 7 FSYNC
```

### 4.5 Image Stitching Pipeline

**Option A: On-Node Stitching**
- Vision processor (Allwinner V851 or similar) stitches all cameras to single 360° stream
- Single encoded stream transmitted via PoE to edge controller
- Reduces bandwidth on link
- Requires more compute on node

**Option B: Edge Controller Stitching**
- All camera streams transmitted individually via PoE to edge controller
- Edge controller stitches 360° panorama
- Higher bandwidth on link
- Simpler node design

**Option C: Hybrid / Progressive**
- Low-res stitched baseline from node
- Higher-res keyframes transmitted progressively
- Edge controller accumulates and improves quality over time

### 4.6 Bandwidth Requirements

| Stream Type | Bandwidth | Transport |
|-------------|-----------|-----------|
| 8 cameras × QVGA MJPEG | 0.5-1 Mbps | BLE sufficient |
| 8 cameras × VGA H.264 | 2-4 Mbps | BLE/Thread sufficient |
| Stitched 360° at 2K H.264 | 3-4 Mbps | PoE (standard) |
| Stitched 360° at 4K H.264 | 8-15 Mbps | PoE (standard) |
| Raw 8× 1080p streams | 50-100 Mbps | PoE required |

**For Variant D, PoE Ethernet is mandatory.** The bandwidth of 8 simultaneous camera streams requires wired transport.

### 4.7 Calibration

**Per-camera calibration (intrinsic):**
- Lens distortion correction
- Focal length and optical center
- Stored in node flash or edge controller

**Extrinsic calibration (inter-camera):**
- Relative position of each camera in the 360° array
- Seam placement and overlap regions
- Loaded at boot

**Stitching quality verification:**
- Straight lines continuous across boundaries
- Seam visibility < 3 pixels at 1080p output
- No distortion at camera transition zones

---

## 5. Candidate Component Set (Test Variants — Not Locked)

### 5.1 MCU / Compute Platform

| Variant | MCU Option | Notes |
|---------|------------|-------|
| A | STM32H743 (Cortex-M7, 480 MHz) | Point cloud processing + event stream |
| B | NXP i.MX RT1060 (Cortex-M7, 600 MHz) | Camera interface capable |
| C | Raspberry Pi CM4 | Linux-capable, handles SLAM + camera streams |
| D | Raspberry Pi CM4 with GPU | Handles stitching for 360° array |

### 5.2 Solid-State LiDAR Module

| Option | Range | Points/sec | Interface | Notes |
|--------|-------|------------|-----------|-------|
| Livox MID-360 | 40 m | 200k | Ethernet | Non-repetitive scan, preferred ceiling |
| Robosense BPearl | 30 m | 120k | Ethernet/RS-485 | Hemispherical FoV |
| Ouster OS0-128 | 50 m | 128-beam | Ethernet | Highest density |
| Terabee 3Dcam | 2 m | — | USB | RGBD ToF, sub-room/close-range |

### 5.3 Event Camera

| Option | Resolution | Interface | Notes |
|--------|------------|-----------|-------|
| Prophesee EVK4 (IMX636) | 1280×720 | USB | Evaluation/research |
| iniVation DAVIS346 | 346×260 | USB | Combined frame+events |
| Prophesee Metavision EVK3 | 640×480 | USB | Lower power |

### 5.4 Camera Modules (Variants C and D)

**Single camera (Variant C):**

| Module | Resolution | Interface | Notes |
|--------|------------|-----------|-------|
| OV5640 | 5 MP | MIPI/DVP | Autofocus option |
| IMX219 | 8 MP | MIPI CSI-2 | Raspberry Pi compatible |
| IMX477 | 12 MP | MIPI CSI-2 | High quality |

**360° array (Variant D):**

| Module | Resolution | Notes |
|--------|------------|-------|
| OV2640 | 2 MP | Low cost, wide-angle/fisheye available |
| IMX219 | 8 MP | With fisheye lens |
| OV5640 | 5 MP | Autofocus, with wide-angle |

### 5.5 Optional mmWave Radar (Variants B, C, D)

| Module | Band | Interface | Notes |
|--------|------|-----------|-------|
| TI IWR6843AOP | 60 GHz | SPI | Integrated antenna |
| Acconeer XR112 | 60 GHz | SPI | Ultra-low power |

### 5.6 Mesh Radio / Network Interface

| Transport | Bandwidth | Use Case |
|-----------|-----------|----------|
| PoE Ethernet 802.3af | 100 Mbps | Variants A, B, C |
| PoE Ethernet 802.3at | 100 Mbps + 25.5W power | Variant D (360° camera) |
| BLE 5.x | 1-2 Mbps | Variants A, B (if no camera) |
| Thread | 250 Kbps | Low-bandwidth fallback |

**For Variants C and D with cameras, PoE is strongly recommended.** The bandwidth for camera data requires wired transport.

### 5.7 Local Storage

| Storage Type | Capacity | Interface | Use Case |
|--------------|----------|-----------|----------|
| microSD | Up to 2 TB | SPI/SDIO | Point clouds, video, SLAM maps |
| QSPI NOR Flash | 8-128 MB | QSPI | Logs, metadata, compressed clips |
| On-node NVMe | 128 GB-1 TB | PCIe | Variant D with local SLAM |

### 5.8 Power Supply

| Source | Use Case |
|--------|----------|
| PoE 802.3af (15.4 W) | Variants A, B, C |
| PoE 802.3at (25.5 W) | Variant D (360° camera array) |

---

## 6. Node Activation Strategy

### 6.1 Event-Triggered Activation (Variants A, B)

```
Event Camera (always on, ~200 mW)
       │
       │ Fast-motion event (< 1 ms latency)
       ▼
LiDAR Controller (MCU)
       │ Wakes LiDAR from sleep
       ▼
LiDAR active (1–3 scan cycles: 100–300 ms)
       │
       ▼
Point Cloud → Cluster → Centroid extraction → DetectionEvent
       │
       ▼
Mesh Radio → Edge Controller (< 10 ms transport)
       │
       ▼
LiDAR returns to sleep (if no continued motion)
```

### 6.2 Continuous Operation (Variants C, D with recording)

For Variants C and D with continuous recording configured:
- Cameras remain active per data handling policy
- LiDAR activation strategy still applies for event-triggered scans
- Edge controller manages continuous recording

---

## 7. SLAM Integration

### 7.1 LiDAR-Based SLAM (All Variants)

Point cloud from solid-state LiDAR provides precise geometry. Frame-to-frame ICP or LOAM-style odometry builds the 3D map.

**Stack (on edge controller):**
- Backend: LIO-SAM or FAST-LIO2
- Dense reconstruction: Open3D
- Loop closure: Scan context descriptors

### 7.2 Visual SLAM (Variant C)

LiDAR provides depth; camera provides visual features for loop closure and map consistency.

**Stack (on edge controller):**
- Visual odometry: ORB-SLAM3 or VINS-Mono
- Dense reconstruction: OpenMVS
- Object recognition: YOLOv8-nano

### 7.3 360° Visual SLAM (Variant D)

Eight cameras covering full horizontal FoV. No blind spots. Provides complete visual context for every voxel in the map.

**Capabilities:**
- Textured 3D reconstruction of the room
- Complete occupancy history with visual context
- Forensic review: any past moment reconstructed in 3D with full visual coverage
- No orientation-dependent blind spots

**Stack (on edge controller):**
- Multi-camera SLAM: ORB-SLAM3 multi-camera extension or custom
- Dense reconstruction: OpenMVS with multi-camera input
- 360° panorama: Real-time stitching for live view

---

## 8. Data Handling — Fully User Configured

All data from this node is user-configured. No system-imposed restrictions.

### 8.1 Configuration (`aegis-mesh.toml`)

```toml
[data]
# Raw sensor data storage
store_raw_point_clouds = true
store_raw_video = false              # For Variants C, D
store_raw_sensor_data = false

# Storage target
storage_target = "edge_controller"   # "edge_controller" | "node_local_sd" | "network"

# Retention
retention_days = 30
continuous_recording = false
recording_trigger = "on_detection"
max_storage_mb = 0

# Streaming
enable_app_streaming = false
stream_quality = "high"

# Integrity
include_integrity_chain = true

# SLAM
enable_slam = true                   # Requires Linux-class edge controller
slam_mode = "sparse"                 # "sparse" | "dense"
```

### 8.2 Data Modes by Variant

**Variant A (Basic):**
- Point clouds on detection
- No camera data
- Data flow: LiDAR → Mesh → Edge Controller

**Variant B (LiDAR + Radar):**
- Point clouds + radar velocity
- No camera data
- Data flow: LiDAR + Radar → Mesh → Edge Controller

**Variant C (LiDAR + Camera):**
- Point clouds + video (if configured)
- Video storage/streaming per configuration
- Data flow: LiDAR + Camera → PoE → Edge Controller → Storage/Stream

**Variant D (LiDAR + 360° Camera):**
- Point clouds + 360° video
- 360° panorama recording and streaming
- Data flow: LiDAR + 8 Cameras → PoE → Edge Controller → Stitch/Store/Stream

---

## 9. Camera Privacy Controls — User Configured

For Variants C and D with cameras, all privacy controls are user-selected. None are mandatory by architecture.

### 9.1 Hardware Options (User-Installed)

**Optional hardware power switch:**
- Physical switch on `V_CAM` rail
- Cuts camera power when set to DISABLED
- No software override
- PCB provides MOSFET footprint + switch footprint

### 9.2 Software Controls

**Activity scheduling:**
- Camera active only during specific hours
- Or: only when sensing layer detects relevant events

**Detection-triggered activation:**
- Camera activates when Always-On node detects motion in zone
- Camera deactivates after configurable timeout

**Continuous recording:**
- All cameras active continuously
- Per retention and storage configuration

### 9.3 Configuration

```toml
[privacy.camera]
hardware_switch_installed = false
software_control_enabled = true
default_state = "disabled"
schedule = []                        # Time ranges when enabled
trigger_on_detection = true
```

---

## 10. Power Architecture

### 10.1 Power Budget by Variant

| Component | Active Power | Peak Power |
|-----------|--------------|------------|
| LiDAR module | 5-10 W | 15 W |
| Event camera | 0.2-0.5 W | 1 W |
| MCU | 0.1-0.5 W | 1 W |
| Mesh radio (BLE/Thread) | 0.05-0.2 W | 0.5 W |
| PoE Ethernet | 0.1-0.3 W | 0.5 W |
| mmWave radar (optional) | 0.5-1 W | 2 W |
| Camera (per module) | 0.1-0.5 W | 1 W |

### 10.2 Total Power by Variant

| Variant | Sleep | Active | Peak | PoE Class |
|---------|-------|--------|------|-----------|
| A (Basic) | 0.3 W | 6-11 W | 17 W | 802.3af |
| B (LiDAR + Radar) | 0.4 W | 7-12 W | 18 W | 802.3af |
| C (LiDAR + Camera) | 0.5 W | 6-12 W | 18 W | 802.3af |
| D (LiDAR + 360°) | 0.8 W | 6-14 W | 25 W | 802.3at |

**Variant D requires PoE 802.3at (Class 4+, 25.5 W).**

### 10.3 Power States

| State | LiDAR | Cameras | Radio | Wake Source |
|-------|-------|---------|-------|-------------|
| Active | Full power | Active | Full | — |
| Idle | Sleep | Off | Connected | Event camera trigger |
| Sleep | Off | Off | Advertising | Mesh command, PoE wake |

---

## 11. Interface Summary

### 11.1 Standard Interfaces (All Variants)

| Interface | Signal Names | Connection |
|-----------|--------------|------------|
| LiDAR | `ETH_TX+/-`, `ETH_RX+/-` | Ethernet to mesh/edge |
| Event Camera | `USB_D+/-` or `SPI_MOSI/MISO/CLK/CS` | USB or SPI |
| mmWave Radar | `SPI_MOSI/MISO/CLK/CS`, `RADAR_IRQ` | SPI |
| Mesh Radio | Integrated MCU or `BLE_TX/RX` | BLE/Thread |
| Debug | `SWD_CLK/DATA`, `UART_TX` | 4-pin header |
| Power | `PoE_VIN+/-` | PoE or DC input |

### 11.2 Camera Interfaces (Variants C, D)

| Signal | Description | Routing Notes |
|--------|-------------|---------------|
| `CAM_n_MIPI_CLK+/-` | MIPI CSI-2 clock per camera n | 100 Ω differential, length-match 2 mil |
| `CAM_n_MIPI_D0+/-` | MIPI CSI-2 data per camera n | 100 Ω differential |
| `CAM_FSYNC` | Hardware sync broadcast | GPIO to all camera FSYNC pins |
| `CAM_PWDN_n` | Power-down per camera | GPIO per camera |

### 11.3 Variant D Additional Interfaces

| Signal | Description |
|--------|-------------|
| `CAM_FSYNC` | Hardware sync to all 4-8 cameras |
| `CAM_SEL[0:2]` | Camera multiplexer select (if used) |
| `ISP_MIPI_CLK+/-` | Vision processor to MCU/SOM |
| `ISP_MIPI_D+/-` | Stitched stream to MCU/SOM |

---

## 12. Testing Strategy

### 12.1 All Variants

1. **Power-on self-test** — MCU boots, CRC verified
2. **Sensor enumeration** — LiDAR, event camera respond
3. **Mesh connectivity** — Node appears on mesh network, ping to edge controller < 50 ms
4. **Firmware version** — Matches current build

### 12.2 LiDAR Testing

1. **Detection range** — Human at 2 m, 5 m, 8 m from ceiling. Point cloud density and centroid accuracy
2. **Geometry accuracy** — 50×50×50 cm box at 3 m. Reported vs. true dimensions within ±2 cm
3. **Event trigger latency** — Hand motion to LiDAR wake < 5 ms
4. **Steam/fog immunity** — Graceful degradation, no false detections

### 12.3 Camera Testing (Variants C, D)

1. **Camera initialization** — All cameras enumerate
2. **Data handling mode verification** — Storage, streaming per configuration
3. **FSYNC synchronization** — All cameras capture within ±1 ms

### 12.4 360° Array Testing (Variant D)

1. **All N cameras initialize** — Within 5 seconds
2. **FSYNC verified** — All cameras trigger within ±1 ms
3. **Stitching quality** — Straight lines continuous across boundaries; seam < 3 px at 1080p
4. **Full recording** — Point cloud + 360° video stored on edge controller SD
5. **Companion app access** — 360° panorama accessible within 5 seconds via edge controller WiFi

---

## 13. Directory Structure

```
lidar_node/
├── variant_a_basic/
│   ├── lidar_node_basic.kicad_pro
│   ├── lidar_node_basic.kicad_sch
│   └── gerbers/
├── variant_b_lidar_radar/
│   ├── lidar_node_radar.kicad_pro
│   └── gerbers/
├── variant_c_lidar_camera/
│   ├── lidar_node_camera.kicad_pro
│   ├── lidar_node_camera.kicad_sch
│   └── gerbers/
├── variant_d_lidar_360_camera/
│   ├── lidar_360_main.kicad_pro
│   ├── lidar_360_main.kicad_sch
│   ├── lidar_360_camera_array.kicad_sch
│   ├── lidar_360_isp.kicad_sch
│   ├── stitching_calibration.json
│   └── gerbers/
├── bom/
│   ├── lidar_node_bom.csv
│   └── camera_array_bom.csv
└── README.md
```
