# Hardware — AEGIS-MESH

This directory contains the hardware reference designs for AEGIS-MESH's distributed residential sensing mesh. The architecture provides geometry-driven distributed perception for residential awareness, fusing LiDAR, mmWave radar, acoustic sensing, environmental sensing, and optional visual capture.

**All designs are sensing-only.** No effector or actuator designs are present in this directory. The project maintains a strict separation between sensing and identification functions, with user-configurable privacy controls at every layer.

---

## Scope and Purpose

AEGIS-MESH hardware provides:

- **Distributed residential perception** via mesh network of sensor nodes
- **Multi-modal sensing** (LiDAR, mmWave radar, acoustic, environmental, optional cameras)
- **Centralized compute and gateway** at the edge controller
- **User-configured data handling** with no system-imposed restrictions
- **Privacy-by-design architecture** realized at the hardware level

**Reference design status:** All specifications are research-grade reference designs. Components are candidate options, not locked selections. Multiple options exist at each functional block to support different deployment scenarios and cost targets.

---

## Critical Architectural Constraint: Edge Controller as Sole External Gateway

**Only the Edge Controller has WiFi, Cellular, and direct BLE connection to the companion app.**

**All other nodes (Always-On, LiDAR, Identity) communicate exclusively via mesh network to the edge controller.** They have no WiFi, no cellular, and no direct connection to external networks.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      AEGIS-MESH Connectivity Model                          │
│                                                                              │
│  Always-On Nodes ──┐                                                         │
│  LiDAR Nodes───────┤── Mesh Radio ──► Edge Controller ──► WiFi ──► App     │
│  Identity Nodes────┘  (BLE/Thread/PoE)       │                              │
│                                              ├──► Cellular ──► Remote App  │
│                                              └──► BLE direct ──► Local App│
│                                                                              │
│  ⚠️  EDGE CONTROLLER IS THE ONLY NODE WITH WiFi / Cellular / External BLE   │
│  ⚠️  ALL OTHER NODES = MESH ONLY (BLE/Thread/PoE to edge, nothing external) │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Why This Architecture?

| Constraint | WiFi on Sensor Node | BLE/Thread on Sensor Node |
|------------|---------------------|---------------------------|
| **Power** | 300-1000+ mW active | 5-50 mW active |
| **Heat** | Significant thermal load | Minimal |
| **Form factor impact** | Requires larger antenna, clearance | Minimal antenna space |
| **Installation flexibility** | Requires WiFi coverage at each sensor location | Only edge controller needs WiFi |
| **Determinism** | Variable latency, contention | Consistent, schedulable |
| **Cost** | WiFi module per node | Single WiFi at edge controller |

**Conclusion:** WiFi on distributed sensor nodes increases cost, power, thermal load, and installation complexity. The centralized gateway model (edge controller) provides external connectivity while maintaining low-power, low-cost sensor nodes that can be placed anywhere without WiFi coverage requirements.

---

## Transport Layer Hierarchy

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         BANDWIDTH HIERARCHY                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│  BLE / Thread (500 Kbps - 2 Mbps)                                            │
│  → Control plane (configuration, commands, health monitoring)               │
│  → Metadata (detections, classification results, track updates)             │
│  → Low-bandwidth sensor data                                                 │
│  → Always-on, lowest power (5-50 mW)                                         │
│  → Primary mesh transport for all sensor nodes                               │
├─────────────────────────────────────────────────────────────────────────────┤
│  PoE Ethernet (100 Mbps - 1 Gbps)                                            │
│  → High-bandwidth sensor data (LiDAR point clouds, multi-camera streams)    │
│  → SLAM data transfer                                                        │
│  → Continuous streaming when required                                        │
│  → Power + data in single cable                                              │
│  → Primary transport for LiDAR nodes (especially 360° camera variants)       │
├─────────────────────────────────────────────────────────────────────────────┤
│  WiFi (50-300+ Mbps) — EDGE CONTROLLER ONLY                                 │
│  → Primary companion app connection                                          │
│  → High-bandwidth media streaming                                            │
│  → SLAM data transfer to app                                                 │
│  → Local network integration (Home Assistant, OpenHAB, etc.)                 │
│  → Power: 300-500 mW when streaming                                          │
├─────────────────────────────────────────────────────────────────────────────┤
│  Cellular (Variable by module) — EDGE CONTROLLER ONLY                        │
│  → LTE Cat 1: 5-10 Mbps — alerts, metadata, compressed clips                │
│  → LTE Cat 4: 50-150 Mbps — moderate streaming                              │
│  → 5G: 100-1000+ Mbps — live streaming, remote access                        │
│  → Used when WiFi unavailable (remote access, away from home)               │
│  → Power: 150-1200 mW depending on module and activity                       │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Supported Cellular Modules (Edge Controller Only)

| Module | Technology | Speed | Interface | Use Case |
|--------|------------|-------|-----------|----------|
| Quectel EC21 | LTE Cat 1 | 5-10 Mbps | UART | Alerts, metadata only |
| Quectel EC25 | LTE Cat 4 | 50-150 Mbps | UART/USB | Compressed video clips |
| Quectel RM502Q-AE | 5G Sub-6GHz | 100+ Mbps | USB 3.x | Live streaming |
| Quectel EG912Y-GL | LTE Cat 12 | 300+ Mbps | USB | Multi-carrier, eSIM |

---

## Node Classes Overview

| Node Class | Sensing Foundation | Camera | Mesh Transport | External Network |
|------------|-------------------|--------|----------------|------------------|
| **Always-On** (Var A-C) | mmWave + PIR + Acoustic | None | BLE/Thread | None |
| **Always-On** (Var D) | mmWave + Acoustic + Camera | User-configured | BLE/Thread | None |
| **LiDAR** (Var A-B) | LiDAR + Event + optional Radar | None | PoE/BLE | None |
| **LiDAR** (Var C) | LiDAR + Event + Camera | User-configured | PoE | None |
| **LiDAR** (Var D) | LiDAR + Event + 360° Camera | User-configured | PoE | None |
| **Identity** | Configurable biometric | User-configured | BLE/Thread/PoE | None |
| **Edge Controller** | mmWave + IMU + Env | None | WiFi/Cellular/BLE | **Sole gateway** |

---

## Hardware Variants

### Always-On Node

**Purpose:** Continuously-powered sensing backbone. Provides presence detection, acoustic classification, and environmental monitoring.

**Variants:**

| Variant | Sensors | Form Factor | Use Case |
|---------|---------|-------------|----------|
| **A — Choke Point** | PIR + 1D ToF + MCU + Mesh Radio | Slim, wall-mount | Doorways, hallways |
| **B — Volume Presence** | mmWave Radar + 4-element Mic Array + Env + MCU + Mesh | Ceiling-corner mount | Living rooms, bedrooms |
| **C — Volume + Enhanced Acoustic** | All of B + 6-element Mic Array | Same as B | Enhanced acoustic analysis |
| **D — Volume + Camera** | All of B + Camera Module | Same as B | Visual capture (user-configured) |

**Configuration options (all variants):**

| Option | Values | Notes |
|--------|--------|-------|
| Camera (Var D) | None / Forward-facing | PCB has footprint, module optionally populated |
| UWB | Disabled / Enabled | Optional precision timing |
| Local storage | microSD / Internal flash | Optional raw data storage |
| Power | USB-C / PoE / Battery | Depends on installation location |

### LiDAR Node

**Purpose:** High-resolution geometric sensing for volume tracking, 3D mapping, and SLAM.

**Variants:**

| Variant | Sensors | Bandwidth | Use Case |
|---------|---------|-----------|----------|
| **A — Basic** | LiDAR + Event Camera | PoE/BLE | Standard geometry |
| **B — Full** | LiDAR + Event + mmWave Radar | PoE/BLE | Fog/smoke robustness |
| **C — + Camera** | All of B + Camera | PoE | Visual SLAM |
| **D — + 360° Camera** | LiDAR + Event + 4-8 Camera Array + Radar | PoE | Complete 360° coverage |

**360° Camera Array Configuration:**

| Config | Cameras | Angular Spacing | FoV Needed per Camera |
|--------|---------|-----------------|------------------------|
| Minimal | 4 | 90° | ≥ 100° |
| Standard | 6 | 60° | ≥ 70° |
| Dense | 8 | 45° | ≥ 55° |
| Ultra | 12 | 30° | ≥ 40° |

**Configuration options:**

| Option | Values | Notes |
|--------|--------|-------|
| Camera count | 4 / 6 / 8 / 12 | Variant D configuration |
| Stitching location | Edge Controller / Node | Edge controller recommended |
| SLAM mode | Sparse / Dense | User-configured |
| Power | PoE 802.3af/at | Required for 360° variants |

### Identity Node

**Purpose:** Configurable visual/biometric identification at entry points or verification zones.

**Variants:**

| Variant | Capability | Storage | Notes |
|---------|------------|---------|-------|
| **A — Standard** | Single configurable sensor | Optional SD | Entry verification |
| **B — Multi-Camera** | 2-4 cameras at configured angles | SD card | 360° entry coverage |
| **C — Outdoor/Weatherproof** | IP66/IP67 enclosure | SD card | Outdoor gate entry |
| **D — Extended Compute** | On-node inference accelerator | SD card + Internal | Edge processing |

**Configuration options:**

| Option | Values | Notes |
|--------|--------|-------|
| Hardware kill switch | Installed / Not installed | User preference |
| Data mode | Metadata-only / Local storage / Streaming / Full recording | User-configured |
| Sensor type | Visual / Depth / Biometric | Configurable per deployment |

### Edge Controller

**Purpose:** Central compute hub, mesh coordinator, sole external gateway, SLAM processor, API server.

**The edge controller is to AEGIS-MESH what the belt node is to SENTINEL-WEAR.**

**Variants:**

| Variant | Compute | RAM | Storage | Use Case |
|---------|---------|-----|---------|----------|
| **A — SBC** | Raspberry Pi 4/5 class | 4-8 GB | SD card | Basic deployment |
| **B — ARM with NPU** | Jetson Nano/Orin class | 4-16 GB | NVMe | Full SLAM, 8+ cameras |
| **C — x86 Mini PC** | Intel/AMD x86 | 8-32 GB | NVMe SSD | Maximum flexibility |
| **D — Industrial** | Industrial-grade SBC | 4-16 GB | Industrial storage | Harsh environments |

**Edge Controller Responsibilities:**

| Function | Description |
|----------|-------------|
| **Mesh Hub** | Coordinates all sensor nodes via BLE/Thread/PoE |
| **Fusion Engine** | Runs `aegis-fusion`, `aegis-tracking` (PentaTrack bridge) |
| **SLAM Processor** | Dense 3D reconstruction from LiDAR + cameras |
| **API Server** | REST/WebSocket for companion app |
| **Recording Manager** | Aggregates recordings, integrity chains, exports |
| **WiFi Gateway** | Primary connection to companion app |
| **Cellular Gateway** | Remote access, away-mode alerts |
| **BLE Direct** | Fallback local connection to app |

---

## Core Node Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                      AEGIS-MESH Sensor Node                       │
├──────────────────────────────────────────────────────────────────┤
│  MCU Core                                                         │
│  - ARM Cortex-M4F / M7 (configurable per node type)              │
│  - FPU required for signal processing                             │
│  - DSP extensions preferred                                       │
├──────────────────────────────────────────────────────────────────┤
│  Sensing Interfaces (Populated per Node Type)                     │
│  - mmWave Radar (SPI/UART)                                        │
│  - Solid-State LiDAR (Ethernet/SPI)                               │
│  - Event Camera (MIPI/SPI parallel)                               │
│  - Microphone Array (I2S/PDM)                                     │
│  - PIR Sensor (GPIO)                                              │
│  - 1D ToF (I2C/SPI)                                               │
│  - Environmental (I2C)                                            │
│  - Camera (MIPI CSI-2 / DVP) — user-configured                    │
├──────────────────────────────────────────────────────────────────┤
│  Storage Interface (Per Configuration)                             │
│  - microSD card (SPI, up to 2 TB)                                 │
│  - QSPI NOR Flash — metadata, logs                                │
├──────────────────────────────────────────────────────────────────┤
│  Mesh Radio Interface — ALL SENSOR NODES                          │
│  - BLE 5.x / Thread (mandatory, integrated or module)             │
│  - PoE Ethernet (LiDAR nodes)                                     │
│  - NO WiFi, NO Cellular                                           │
├──────────────────────────────────────────────────────────────────┤
│  Debug Interface                                                  │
│  - SWD (Serial Wire Debug)                                        │
│  - UART Console                                                   │
└──────────────────────────────────────────────────────────────────┘
```

```
┌──────────────────────────────────────────────────────────────────┐
│                      Edge Controller                              │
├──────────────────────────────────────────────────────────────────┤
│  Compute Core                                                     │
│  - ARM Cortex-A72/A76 (SBC variants)                              │
│  - x86 (mini PC variants)                                         │
│  - 4-32 GB RAM                                                    │
│  - Linux OS (full crate stack)                                    │
├──────────────────────────────────────────────────────────────────┤
│  Mesh Hub Interfaces                                              │
│  - BLE 5.x (to sensor nodes)                                      │
│  - Thread (to sensor nodes)                                       │
│  - PoE Switch (to LiDAR nodes)                                    │
├──────────────────────────────────────────────────────────────────┤
│  External Network Interfaces — EDGE CONTROLLER ONLY              │
│  - WiFi 802.11ac/ax (built-in or external)                        │
│  - Cellular LTE/5G module (nano-SIM or eSIM)                      │
│  - BLE 5.x (direct to companion app)                              │
│  - USB-C (PC companion + charging)                                │
├──────────────────────────────────────────────────────────────────┤
│  Storage                                                          │
│  - microSD (basic variants)                                       │
│  - NVMe SSD (high-performance variants)                           │
│  - Network storage (optional)                                     │
├──────────────────────────────────────────────────────────────────┤
│  Sensing (Optional)                                               │
│  - mmWave Radar (downward)                                        │
│  - IMU (reference orientation)                                    │
│  - Environmental sensors                                          │
└──────────────────────────────────────────────────────────────────┘
```

---

## Privacy Architecture

### Hardware-Level Commitments

The privacy-by-design architecture is realized at the hardware level, not just through firmware or configuration:

**1. Always-On Nodes (Variants A-C) are camera-free at the firmware dependency level.**

These variants do not link `omni-sense-drivers-camera`. The firmware binary for Variants A-C has no camera driver capability. Users cannot enable cameras on these variants through configuration — the capability is architecturally absent.

**2. LiDAR Nodes (Variants A-B) are camera-free at the firmware dependency level.**

Same architectural commitment as Always-On Variants A-C.

**3. Identity Nodes and camera-capable variants support user-configured privacy controls.**

For Identity Nodes and Always-On/LiDAR variants with cameras:
- **Optional hardware kill switch** (user-installed, physically cuts camera power rail)
- **Software privacy controls** (scheduling, detection-triggered, manual)
- **Data handling modes** (metadata-only, local storage, streaming, full recording)

### No Mandatory Hardware Kill Switch Requirement

The architecture supports hardware kill switches for users who want unconditional physical camera disable. It also supports software-only privacy controls for users who prefer flexibility. Both are valid configurations.

The system does not mandate either approach. Jurisdiction-specific legal requirements are the deployer's responsibility.

### Data Handling — Fully User Configured

All data storage, streaming, and retention is user-configured. No system-imposed restrictions.

| Mode | Description | Use Case |
|------|-------------|----------|
| Metadata-only | Classification tags only, no raw data | Minimum footprint |
| Local storage | Raw data on SD/internal flash | Personal records, evidence |
| Streaming | Live to companion app | Real-time monitoring |
| Full recording | All data captured with integrity chain | Legal documentation |

---

## Test Pad Standard

### Sensor Nodes (12-Pad)

| Pad | Signal | Description |
|---|---|---|
| 1 | `VCC_BATT` | Battery/input power |
| 2 | `GND` | Ground |
| 3 | `CHRG_IN+` | Charging positive |
| 4 | `CHRG_IN-` | Charging negative |
| 5 | `MESH_RF_TEST` | Mesh radio test point |
| 6 | `MCU_BOOT` | Boot mode select |
| 7 | `MCU_RESET` | MCU reset |
| 8 | `CAM_HW_SW_TEST` | Camera switch GPIO (if installed) |
| 9 | `LED_STATUS` | Status LED |
| 10 | `SWD_CLK` | SWD clock |
| 11 | `SWD_DATA` | SWD data |
| 12 | `UART_TX` | Debug UART |

### Edge Controller Additional Pads

| Pad | Signal | Description |
|---|---|---|
| 13 | `CELLULAR_PWR` | Cellular module power |
| 14 | `SIM_VCC` | SIM card power |
| 15 | `WIFI_RST` | WiFi reset |
| 16 | `CELLULAR_STATUS` | Cellular status GPIO |
| 17 | `USB_D+` | USB data plus |
| 18 | `USB_D-` | USB data minus |

---

## Power Architecture

### Sensor Nodes

| Node Variant | Sleep (mW) | Active (mW) | Peak (mW) |
|--------------|------------|-------------|-----------|
| Always-On A (PIR) | 10-50 | 100-300 | 500 |
| Always-On B (volume) | 50-150 | 500-1000 | 1500 |
| Always-On D (+ camera) | 60-200 | 600-1500 | 2500 |
| LiDAR A (event) | 200-500 | 2000-5000 | 8000 |
| LiDAR D (360° camera) | 400-1000 | 6000-14000 | 25000 |

### Edge Controller

| Variant | Idle (W) | Active (W) | Peak (W) |
|---------|----------|------------|-----------|
| A — SBC | 2-4 | 5-10 | 15 |
| B — ARM NPU | 3-5 | 8-15 | 25 |
| C — x86 | 5-10 | 15-30 | 50 |

---

## Storage Architecture

### Sensor Node Storage

| Type | Capacity | Interface | Use Case |
|------|----------|-----------|----------|
| microSD | Up to 2 TB | SPI | Raw video, audio, sensor data |
| QSPI NOR Flash | 8-128 MB | QSPI | Metadata, logs, model weights |
| Internal MCU Flash | 512 KB - 2 MB | Internal | Firmware, critical config |

### Edge Controller Storage

| Type | Capacity | Interface | Use Case |
|------|----------|-----------|----------|
| microSD | Up to 2 TB | SDIO | Recordings, SLAM maps (basic variants) |
| NVMe SSD | 128 GB - 2 TB | PCIe | Full deployment (high-performance variants) |
| Network storage | Variable | Ethernet | Backup, archive |

---

## 360° Camera Architecture (LiDAR Variant D)

### Wired Internal Camera Bus

All cameras connect via internal MIPI CSI-2 bus to a central vision processor on the node or transmit via PoE to the edge controller for stitching.

**Recommended architecture:** Edge controller handles stitching

- All N camera streams transmitted via PoE to edge controller
- Edge controller (Linux SoM) runs stitching with GPU acceleration
- Single 360° panorama stream served to companion app
- Full-resolution recording on edge controller storage

### FSYNC Hardware Synchronization

- Single FSYNC GPIO line from vision processor/edge controller
- Connects to FSYNC input of all cameras simultaneously
- Target: < 10 ns skew between any two cameras
- Required for clean stitching without artifacts

### Bandwidth Considerations

| Stream Type | Bandwidth | PoE Capability |
|-------------|-----------|----------------|
| 8 cameras × VGA H.264 | ~2-4 Mbps | ✅ Easily handled |
| Stitched 360° at 2K H.264 | ~3-4 Mbps | ✅ Handled |
| Stitched 360° at 4K H.264 | ~8-15 Mbps | ✅ Handled (PoE 100 Mbps+) |

---

## SLAM Architecture

### Sparse World Model (Always Available)

- **Technology:** OMNI-SENSE → PentaTrack → Zone-anchored tracks
- **Output:** Per-zone tracked entities with position, velocity, classification
- **Visualization:** Floor-plan overlay with detection blobs, velocity vectors
- **Hardware:** All edge controller variants

### Dense SLAM World Model (Hardware Dependent)

- **Technology:** LiDAR point clouds + camera visual odometry → SLAM → 3D mesh
- **Output:** Full 3D geometry with object recognition
- **Visualization:** 3D walkthrough with entity trajectories
- **Hardware:** Edge Controller Variant B or C (GPU/NPU required)

### Multi-Room SLAM

- Nodes hand off tracking across room boundaries
- Global consistency maintained via loop closure
- Map segments per room with automatic labeling
- User can export/delete specific rooms

---

## Mechanical Specifications

### Enclosure Types

| Node Type | Mounting | IP Rating | Notes |
|-----------|----------|-----------|-------|
| Always-On A | Wall-mount | IP54 | Choke-point position |
| Always-On B-D | Ceiling-corner | IP54 | Volume coverage |
| LiDAR A-C | Ceiling | IP54 | Geometry sensing |
| LiDAR D | Ceiling center | IP54 | 360° coverage |
| Identity | Wall-mount | IP54 (IP66 outdoor variant) | Entry verification |
| Edge Controller | Shelf/rack/wall | IP54 | Indoor only |

### Materials

**Skin-contact surfaces (if applicable):**
- Grade 5 titanium (Ti-6Al-4V)
- 316L surgical stainless steel
- Medical-grade silicone (ISO 10993)

**Enclosure materials:**
- Powder-coated aluminum
- ABS plastic
- UV-resistant polycarbonate

---

## Testing

### Production Test Jig

The test jig validates all node variants before deployment. See `hardware/testing/test_jig_pcb/` for full specification.

**Tests for all nodes:**
1. Power-on self-test
2. Rail verification
3. Sensor enumeration
4. Mesh radio connectivity
5. Firmware version check

**Tests for edge controller only:**
1. WiFi AP association
2. API server endpoints
3. Cellular/SIM detection and AT command
4. Recording manager

**Tests for camera-capable variants:**
1. Camera initialization
2. Data handling mode verification
3. Optional hardware switch test (if installed)

---

## Regulatory Compliance

All hardware designs conform to:

| Domain | Standards |
|--------|-----------|
| RF emission | FCC Part 15, CE RED, ISED RSS |
| Electrical safety | UL 62368-1, IEC 62368-1 |
| Radio module | Pre-certified BLE/Thread modules |
| EMC | FCC Part 15 Subpart B, EN 55032 |

---

## Directory Structure

```
hardware/
├── schematic/
│   ├── always_on_node/
│   │   ├── variant_a_chokepoint/
│   │   ├── variant_b_volume/
│   │   ├── variant_c_enhanced_acoustic/
│   │   ├── variant_d_volume_camera/
│   │   └── bom/
│   ├── lidar_node/
│   │   ├── variant_a_basic/
│   │   ├── variant_b_full/
│   │   ├── variant_c_lidar_camera/
│   │   ├── variant_d_lidar_360_camera/
│   │   └── bom/
│   ├── identity_node/
│   │   ├── variant_a_standard/
│   │   ├── variant_b_multi_camera/
│   │   ├── variant_c_outdoor/
│   │   └── bom/
│   ├── edge_controller/
│   │   ├── variant_a_sbc/
│   │   ├── variant_b_arm_npu/
│   │   ├── variant_c_x86/
│   │   └── bom/
│   └── README.md
├── testing/
│   └── test_jig_pcb/
├── hardware_config.md
└── README.md
```

---

## Configuration Reference

See `hardware_config.md` for complete interface specifications, pin maps, power domains, and configuration options.

---

## Legal and Ethics

All designs are research reference designs. Not certified products. Not life-safety devices. Users are responsible for:

- Regulatory compliance in their jurisdiction
- Installation safety
- Recording consent laws (two-party consent states, GDPR, etc.)
- Data protection compliance

See `legal/compliance.md`, `legal/research_ethics.md`, and `legal/tos_compliance.md` for the project's full legal posture.
