# LiDAR Node — Hardware Schematic Reference Design

**Project:** AEGIS-MESH
**Node Class:** LidarNode — High-Resolution 3D Geometry and Tracking
**Status:** Reference Design — All Variants. No component is locked.
**License:** CERN-OHL-S v2

---

## 1. Purpose

The LiDAR Node provides high-resolution geometric sensing for volume tracking, 3D mapping, and SLAM integration. It activates on-demand (triggered by the Always-On Node's motion events) or continuously at low duty cycle for dynamic remap scanning.

Four variants range from a basic event-triggered LiDAR setup to a full 360° camera array with SLAM capability.

---

## 2. Node Variants

### Variant A — Basic Event-Triggered LiDAR

**Sensors:**
- Solid-state LiDAR (ceiling-mount)
- Event camera (fast-motion trigger — wakes LiDAR on microsecond-latency detection)

**Purpose:** Cost-effective high-fidelity geometry. LiDAR activates only when event camera detects motion. Reduces average power consumption dramatically.

**Use case:** Standard residential room coverage where continuous LiDAR is unnecessary.

### Variant B — Full LiDAR + Radar

**Sensors:**
- Solid-state LiDAR
- Event camera
- mmWave FMCW radar

**Purpose:** Dual modality for fog/smoke/steam robustness. Radar continues to detect in conditions that degrade LiDAR. Provides velocity from Doppler alongside LiDAR geometry.

**Use case:** Kitchens (steam), bathrooms (steam), rooms with smoke or variable visibility.

### Variant C — LiDAR + Camera

**Sensors:**
- Solid-state LiDAR
- Event camera
- Conventional camera (configurable resolution)
- mmWave radar (optional)

**Purpose:** Enables visual SLAM (LiDAR provides depth; camera provides visual odometry and texture). Supports dense 3D reconstruction with RGB texture. Camera data handling fully user-configured.

**Use case:** Users wanting full SLAM capability with textured 3D maps and visual recording.

### Variant D — LiDAR + 360° Camera Array

**Sensors:**
- Solid-state LiDAR (ceiling mount)
- Event camera
- 4–8 camera array (full 360° horizontal coverage of room)
- mmWave radar

**Purpose:** Room-scale 360° video capture at ceiling position combined with LiDAR geometry. Enables:
- Full 360° visual recording of the room
- Dense SLAM with complete visual coverage (no blind spots from single-camera angle)
- Occupancy mapping with full visual context

**Use case:** Users wanting the most comprehensive room-scale documentation and SLAM capability. Equivalent to a full-room 360° security camera system with added LiDAR depth.

---

## 3. Candidate Component Set (Test Variants — Not Locked)

### MCU

- **Variant A:** STM32H743 (Cortex-M7, 480 MHz, point cloud processing + event stream)
- **Variant B:** NXP i.MX RT1060 (Cortex-M7, 600 MHz, camera interface)
- **Variant C:** Raspberry Pi CM4 (Linux-capable, handles dense SLAM for Variants C/D)

### Solid-State LiDAR Module

- **Variant A:** Livox MID-360 (non-repetitive scan, 40 m, 200k pts/s, Ethernet) — preferred ceiling mount
- **Variant B:** Robosense BPearl (hemispherical FoV, 30 m, 120k pts/s, Ethernet/RS-485)
- **Variant C:** Ouster OS0-128 (128-beam, 50 m, highest density, Ethernet)
- **Variant D:** Terabee 3Dcam (RGBD ToF, 2 m, USB) — sub-room / close-range nodes

### Event Camera

- **Variant A:** Prophesee EVK4 (IMX636, 1280×720, USB) — evaluation/research
- **Variant B:** iniVation DAVIS346 (346×260, USB, combined frame+events)
- **Variant C:** Prophesee Metavision EVK3 (640×480, USB, lower power)

### Camera Module (Variants C and D)

**Single camera (Variant C):**
- **Variant A:** OmniVision OV5640 (5 MP, autofocus, MIPI/DVP)
- **Variant B:** Sony IMX219 (8 MP, CSI-2)
- **Variant C:** Sony IMX477 (12 MP, high quality) — for premium installations

**360° Camera Array (Variant D):**
- 4 cameras at 90° spacing → each needs ≥ 100° FoV
- 6 cameras at 60° spacing → each needs ≥ 70° FoV
- 8 cameras at 45° spacing → each needs ≥ 55° FoV

Recommended camera per position: OmniVision OV2640 (2 MP, wide-angle optics) for initial research; Sony IMX219 with fisheye for higher quality.

**Image stitching for Variant D:**
- Raspberry Pi CM4 belt equivalent on-node handles stitching via GPU
- Or: individual camera streams transmitted to edge controller for stitching
- Or: companion app handles stitching for maximum quality

### Optional mmWave Radar (Variants B, C, D)

- TI IWR6843AOP (as Always-On Node)
- Acconeer XR112 (lower power)

### Local Storage

- microSD card (SPI, up to 2 TB) — point clouds, video recordings, SLAM maps
- Internal flash — logs, compressed metadata

### Power Supply

- **PoE 802.3af/at preferred** for ceiling-mounted nodes.
- Variant D (full 360° camera): may require PoE 802.3at (25.5 W) — check total draw.

---

## 4. Node Activation Strategy

```
Event Camera (always on, low power: ~200 mW)
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

For Variant D with continuous recording configured, cameras remain active and record per data handling policy. LiDAR activation strategy still applies.

---

## 5. SLAM Integration

Variants C and D enable full SLAM operation:

**LiDAR-based SLAM:** Point cloud from solid-state LiDAR provides precise geometry. Frame-to-frame ICP or LOAM-style odometry builds the 3D map.

**Visual SLAM (LiDAR + Camera):** Camera provides visual feature tracking for loop closure and map consistency. Dense reconstruction via OpenMVS or similar.

**360° Visual SLAM (Variant D):** Eight cameras covering full horizontal FoV. No blind spots. Provides complete visual context for every voxel in the map. Enables:
- Textured 3D reconstruction of the room
- Complete occupancy history with visual context
- Forensic review: any past moment reconstructed in 3D with full visual coverage

**SLAM stack (on Raspberry Pi CM4 class node or edge controller):**
- Backend: ORB-SLAM3 or LIO-SAM
- Dense reconstruction: OpenMVS
- Object recognition: YOLO variant on edge controller

---

## 6. Data Handling

All data from this node is user-configured:

**Point clouds:** Store raw LiDAR point clouds for SLAM and forensic analysis. Size: ~1–5 MB per scan at 200k pts/s.

**Video (Variants C, D):** Store raw or compressed video per `aegis-mesh.toml`. Stream to companion app when `enable_app_streaming = true`.

**360° panorama (Variant D):** Store stitched equirectangular recordings. Available for VR headset playback. Stream live to companion app.

**Metadata-only mode:** Default. Transmit only detection centroids and classification hints over mesh.

---

## 7. Power Architecture

| Component | Typical Power | Peak Power |
|---|---|---|
| LiDAR module | 5–10 W active, 0.1 W sleep | 15 W |
| Event camera | 0.2–0.5 W | 1 W |
| MCU | 0.1–0.3 W | 0.5 W |
| Mesh radio | 0.05–0.2 W | 0.5 W |
| Camera (per module, Variant C) | 0.1–0.5 W | 1 W |
| 8-camera array (Variant D) | 0.8–4 W | 8 W |
| **Total Variant A (event trigger)** | **~0.4–11 W** | **~17 W** |
| **Total Variant D (full active)** | **~6–14 W** | **~25 W** |

Variant D requires PoE 802.3at (Class 4+).

---

## 8. Testing Strategy

1. **Detection range:** Human at 2 m, 5 m, 8 m from ceiling. Point cloud density and centroid accuracy.
2. **Geometry accuracy:** 50×50×50 cm box at 3 m. Reported vs. true dimensions.
3. **Event-trigger:** Hand motion → LiDAR wake within 5 ms.
4. **Steam/fog immunity:** Graceful degradation, no false detections.
5. **Power profile:** All states measured per variant.
6. **SLAM quality (Variants C/D):** Closed-loop room traversal; map closure error < 5 cm.
7. **360° stitching (Variant D):** Straight lines continuous across camera boundaries. Seam visible < 3 pixels at 1080p output.

---

## 9. Directory Structure

```
lidar_node/
├── variant_a_basic/
│   ├── lidar_node_basic.kicad_pro
│   └── gerbers/
├── variant_b_lidar_radar/
│   └── gerbers/
├── variant_c_lidar_camera/
│   └── gerbers/
├── variant_d_lidar_360_camera/
│   ├── lidar_360_main.kicad_pro
│   ├── lidar_360_main.kicad_sch
│   ├── lidar_360_camera_array.kicad_sch
│   ├── stitching_calibration.json
│   └── gerbers/
├── bom/
└── README.md
```
