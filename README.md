# AEGIS-MESH — Distributed Residential Sensing Research Platform

**A geometry-driven distributed perception mesh for residential awareness. AEGIS-MESH fuses LiDAR, mmWave radar, neuromorphic sensing, and optional visual capture to answer awareness questions — presence, occupancy, anomaly, geometry, and identity — with full user control over data capture, storage, and retention.**

[![License: MIT](https://img.shields.io/badge/Code-MIT-blue.svg)](#license)
[![License: CERN OHL](https://img.shields.io/badge/Hardware-CERN%20OHL--S%20v2-green.svg)](#license)
[![Status: Research](https://img.shields.io/badge/Status-Research%20%2F%20Educational-orange.svg)](#scope-and-disclaimers)
[![Sensing-Only Effectors](https://img.shields.io/badge/Effectors-Out%20of%20Scope-orange.svg)](#scope-and-disclaimers)

---

## Scope and Disclaimers

AEGIS-MESH is a **research and education platform** studying distributed residential perception. It provides architecture, firmware reference implementations, software stack, and protocol specifications for building a high-fidelity sensor mesh.

**Key Thesis:** *Most actionable residential awareness needs — presence, tracking, fall detection, intrusion, geometry — can be solved with higher reliability using a distributed mesh of complementary sensors. The sensing backbone is camera-free by architecture. Visual capture is an additive configurable layer, not the foundation.*

**Intended Audience:** Hobbyists, makers, privacy researchers, educators, integrators, and home automation enthusiasts.

**Disclaimers:**
- Not a certified life-safety product (not UL-listed or CE-marked for safety functions).
- Not a substitute for smoke/CO detectors or monitored alarm systems.
- User assumes responsibility for validating configurations for their jurisdiction and use case.
- Recording laws vary by jurisdiction — users are responsible for compliance with applicable consent and surveillance regulations.

---

## What the Project Actually Builds

A **distributed residential sensor mesh** installed in a home or building that continuously maintains a 3D awareness field. Sensors are mounted at fixed positions (ceiling corners, doorways, hallways) and together build a complete perception mesh of the environment. The system supports two parallel world-model modes simultaneously:

**Mode A — Sparse Probabilistic World Model:** PentaTrack-based, low-power, event-driven. Fast, efficient. Answers: "Something is here, moving this way, at this speed."

**Mode B — Dense SLAM World Map:** Full 3D reconstruction from LiDAR + camera arrays. Geometry-complete. Answers: "This is what the environment looks like, with objects identified and tracked." Requires higher compute (edge controller Linux variant).

Both modes run simultaneously when hardware supports it. Mode A always runs. Mode B activates based on hardware configuration and user preference.

```
                         ┌─────────────────────────────────┐
                         │        LiDAR Node (360°)         │
                         │  Solid-state LiDAR + Event Cam  │
                         │  + 360° Camera Array + Radar    │
                         │  Ceiling mount, full room cover │
                         └──────────────┬──────────────────┘
                                        │
             ┌──────────────────────────┼────────────────────────────┐
             │                          │                            │
   ┌─────────▼────────────┐  ┌──────────▼──────────────┐  ┌─────────▼──────────┐
   │  Always-On Node      │  │  Always-On Node          │  │  Always-On Node    │
   │  — Volume Presence   │  │  — Volume Presence       │  │  — Choke Point     │
   │  mmWave+Acoustic+Env │  │  mmWave+Acoustic+Env     │  │  PIR+ToF           │
   │  + optional camera   │  │  + optional camera       │  │  Binary detection  │
   └─────────────────────┘  └─────────────────────────┘  └────────────────────┘
                                        │
                                        │
                                        ▼
                             ┌──────────────────────┐
                             │   Edge Controller    │
                             │   PRIMARY HUB         │
                             │   SOLE EXTERNAL GATEWAY│
                             │   Compute + Storage   │
                             │   WiFi + Cellular     │
                             │   Mesh Hub + API      │
                             │   SLAM Processor      │
                             └──────────┬───────────┘
                                        │
                          ┌─────────────┴──────────────┐
                          │                            │
             ┌────────────▼────────┐     ┌────────────▼────────┐
             │  Identity Node      │     │  LiDAR Node          │
             │  Configurable       │     │  Solid-state LiDAR   │
             │  Visual/Biometric   │     │  + Event + Radar     │
             │  Entry point        │     │  + optional camera   │
             └─────────────────────┘     └─────────────────────┘
```

---

## Connectivity Architecture — The Edge Controller as Sole External Gateway

### Critical Architectural Principle

**Only the Edge Controller connects to external networks.** This is not a configuration choice — it is an architectural constraint enforced at both hardware and firmware levels.

**All other nodes (always-on, LiDAR, identity) communicate exclusively via mesh radio (BLE/Thread/PoE) to the edge controller.** They have no WiFi, no cellular, and no direct connection to the companion app.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      AEGIS-MESH Connectivity Model                          │
│                                                                              │
│  Always-On Nodes ──┐                                                         │
│  LiDAR Nodes───────┤                                                         │
│  Identity Nodes────┼── Mesh Radio ──► Edge Controller ──► WiFi ──► Companion App│
│                     │  (BLE/Thread/PoE)       │                               │
│                     │                         ├──► Cellular ──► Remote App   │
│                     │                         └──► BLE direct ──► Local App │
│                                                                              │
│  ⚠️  EDGE CONTROLLER IS THE ONLY NODE WITH WiFi / Cellular / BLE (to external)│
│  ⚠️  ALL OTHER NODES = MESH RADIO ONLY (to edge controller, nothing external)│
└─────────────────────────────────────────────────────────────────────────────┘
```

### Why This Architecture?

| Constraint | WiFi on Sensor Node | BLE/Thread on Sensor Node | PoE on Sensor Node |
|------------|---------------------|---------------------------|---------------------|
| **Power** | 300-1000+ mW active | 5-15 mW active | Powered by Ethernet (no battery) |
| **Heat** | Significant thermal load | Minimal | None (external power) |
| **Installation complexity** | Requires power outlet nearby | Battery or low-power wired | Requires Ethernet run |
| **Determinism** | Variable latency | Consistent, schedulable | Excellent |
| **Coverage reliability** | Subject to WiFi dead zones | Mesh covers dead zones | Wired reliability |

**Conclusion:** WiFi on wall/ceiling-mounted sensor nodes adds cost, complexity, and failure modes. Mesh radio (BLE/Thread) or PoE provides reliable, deterministic communication to the edge controller, which handles all external connectivity.

---

## Transport Layer Hierarchy

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         BANDWIDTH HIERARCHY                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│  BLE / Thread (500 Kbps - 2 Mbps practical)                                  │
│  → Control plane (configuration, commands, health)                          │
│  → Metadata (detections, classification results, track updates)             │
│  → Low-bandwidth sensor data                                                 │
│  → Always-on, lowest power (5-15 mW)                                         │
│  → Present on ALL nodes                                                      │
├─────────────────────────────────────────────────────────────────────────────┤
│  PoE Ethernet (100 Mbps - 1 Gbps)                                            │
│  → High-bandwidth continuous (LiDAR point clouds, 360° camera streams)       │
│  → Simultaneous multi-stream capability                                      │
│  → Zero wireless interference                                               │
│  → Power + data over single cable                                            │
│  → LiDAR Variant D preferred transport                                       │
├─────────────────────────────────────────────────────────────────────────────┤
│  WiFi (50-300+ Mbps) — EDGE CONTROLLER ONLY                                 │
│  → Companion app connection (primary)                                        │
│  → Media streaming to app                                                    │
│  → SLAM data transfer to app                                                 │
│  → Remote access (when enabled)                                              │
│  → Power: 300-500 mW when streaming                                          │
├─────────────────────────────────────────────────────────────────────────────┤
│  Cellular (Variable by module) — EDGE CONTROLLER ONLY                       │
│  → LTE Cat 1: 5-10 Mbps — alerts, metadata, compressed clips                │
│  → LTE Cat 4: 50-150 Mbps — good for moderate streaming                     │
│  → 5G: 100-1000+ Mbps — suitable for 360° live relay                         │
│  → Used when WiFi unavailable (remote property, backup connectivity)         │
│  → Power: 150-1200 mW depending on module and activity                       │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Supported Cellular Modules (Edge Controller Only)

| Module | Technology | Speed | Interface | Use Case |
|--------|------------|-------|-----------|----------|
| Quectel EC21 | LTE Cat 1 | 5-10 Mbps | UART | Alerts, metadata only |
| Quectel EC25 | LTE Cat 4 | 50-150 Mbps | UART/USB | Compressed video clips |
| Quectel RM502Q-AE | 5G Sub-6GHz | 100+ Mbps | USB 3.x | Live 360° streaming |
| Quectel EG912Y-GL | LTE Cat 12 | 300+ Mbps | USB | Multi-carrier, eSIM |

---

## Node Architecture — All Variants

### Always-On Node

The continuously-powered sensing backbone of AEGIS-MESH.

#### Hardware Variants (Different PCB Designs)

**Variant A — Choke Point (Minimal)**
- PIR + 1D ToF + MCU + Mesh Radio (BLE/Thread)
- Lowest power, lowest cost
- Binary detection at doorways
- Ceiling or wall mount
- ~30-50 g, battery or wired power

**Variant B — Volume Presence (Primary)**
- mmWave Radar + 4-element Microphone Array + Environmental Sensors + MCU + Mesh Radio
- Full presence detection + acoustic classification
- No camera, no LiDAR
- ~50-80 g, wired power preferred

**Variant C — Volume + Enhanced Acoustic**
- All of Variant B + 6-element microphone array
- Enhanced 3D DOA and material classification
- ~60-90 g

**Variant D — Volume + Optional Camera**
- All of Variant B + configurable camera module
- Camera data handling fully user-configured
- Camera encoded on-node → mesh radio → edge controller → WiFi → companion app
- ~70-100 g

#### Configuration Options (Same PCB, Different Population)

For Variants B, C, D, these are configuration options (not separate hardware variants):

| Option | Values | Notes |
|--------|--------|-------|
| Camera | None / Forward-facing / Wide-angle | PCB has footprint, module optionally populated |
| Mesh radio | BLE / Thread / Both | Selection at assembly or firmware config |
| Environmental sensors | Basic / Extended | Different population |
| Storage | None / SD card footprint | For raw audio/sensor data |

### LiDAR Node

High-resolution geometric sensing for volume tracking, 3D mapping, and SLAM.

#### Hardware Variants

**Variant A — Basic LiDAR**
- Solid-state LiDAR (Livox MID-360 class) + event camera (trigger) + MCU + Mesh Radio
- Event-triggered activation for power efficiency
- No conventional camera
- Ceiling or wall mount
- PoE preferred for power + data

**Variant B — Full LiDAR + Radar**
- Solid-state LiDAR + event camera + mmWave radar + MCU + Mesh Radio (PoE preferred)
- Dual modality for fog/smoke robustness
- No conventional camera
- Ceiling mount recommended

**Variant C — LiDAR + Camera**
- All of Variant B + conventional camera (configurable resolution)
- Camera provides visual SLAM odometry + recording
- Full SLAM capability (LiDAR + visual)
- PoE for power + high-bandwidth data

**Variant D — LiDAR + 360° Camera Array**
- Solid-state LiDAR + event camera + 4–8 camera array + mmWave radar
- Room-scale 360° video capture from ceiling position
- High bandwidth: requires PoE Ethernet to edge controller
- Full dense SLAM with 360° visual coverage
- This is the most capable and most expensive variant
- Best suited for open-plan spaces, main living areas, atriums

#### Camera Coverage Configurations (Variant D)

| Config | Cameras | Angular Spacing | FoV Needed | Coverage |
|--------|---------|-----------------|------------|----------|
| Minimal | 4 | 90° | ≥ 100° | 360° with overlap |
| Standard | 6 | 60° | ≥ 70° | 360° good overlap |
| Dense | 8 | 45° | ≥ 55° | 360° excellent overlap |

#### 360° Camera Stitching Architecture

**Wired Internal Camera Bus:**
- All cameras connect via MIPI CSI-2 internal bus to central vision processor
- Hardware FSYNC synchronization across all cameras
- Stitching performed on edge controller (recommended) or on-node vision processor

**Bandwidth over PoE:**
- 4 cameras × 1080p H.264: ~8-12 Mbps (easily handled)
- 8 cameras × 720p H.264: ~10-15 Mbps (easily handled)
- 8 cameras × 1080p H.264: ~20-30 Mbps (within GigE PoE capacity)

### Identity Node

The configurable identification layer at entry points.

#### Hardware Variants

**Variant A — Metadata-Only (Default)**
- Configurable camera/biometric sensor
- On-device inference
- Metadata transmission only (classification tags)
- No local storage by default
- ~50-80 g

**Variant B — Local Storage**
- All of Variant A + SD card
- Raw video, clips, and metadata stored locally
- User accesses recordings via companion app or physical SD removal
- ~60-90 g

**Variant C — App Streaming + Storage**
- All of Variant B + streaming capability
- Live footage available in companion app
- Full recordings library
- ~70-100 g

**Variant D — Multi-Camera Array**
- Multiple cameras at configurable angles
- 360° coverage of entry point or verification zone
- All camera streams stored and available
- Best for high-traffic entries or commercial deployments
- ~100-150 g

#### Privacy Controls (User-Configured — None Mandatory)

**Option A — Hardware Power Switch (user-installed):**
- Physical switch on `V_CAM` power rail
- PCB provides footprint + MOSFET
- When DISABLED: sensor physically unpowered
- Firmware reads `CAM_HW_SW_GPIO` at boot if pin populated

**Option B — Software-Only Control:**
- MCU controls camera power via MOSFET GPIO
- Scheduling, detection-triggered, or manual activation

**No mandatory switch requirement.** Users configure per applicable laws in their jurisdiction.

#### Data Handling Modes (All User-Configured)

| Mode | Description | Camera Data |
|------|-------------|-------------|
| Metadata-only | On-device inference, transmit tags only | Not stored or transmitted |
| Local storage | Raw frames written to SD card | Stored on node SD |
| App streaming | Live footage to companion app | Streamed via edge controller |
| Full recording | All data captured and stored | Stored + available for export |

---

## Edge Controller — Central Compute Hub

### Role in the Architecture

The edge controller is AEGIS-MESH's equivalent to SENTINEL-WEAR's belt node. It is the **central compute and network hub** for the entire sensing mesh.

| Function | Edge Controller |
|----------|-----------------|
| Compute hub | Runs fusion, SLAM, policy engine |
| Fusion center | Combines data from all nodes |
| SLAM processor | Builds dense 3D world model |
| API server | Serves companion app |
| Recording manager | Aggregates and stores recordings |
| WiFi gateway | Primary path to companion app |
| Cellular gateway | Remote access (optional) |
| Mesh hub | Coordinates all sensor nodes |
| Time master | Provides clock synchronization |

### Hardware Variants

**Variant A — SBC Class (Raspberry Pi 4/5)**
- ARM Cortex-A72/A76, 4-8 GB RAM
- SD card or NVMe storage
- Built-in WiFi 802.11ac
- Optional cellular USB module
- Cost: $50-100
- Capability: Sparse tracking, 2-4 camera streams, basic SLAM
- Suitable for: Basic residential deployment

**Variant B — ARM with NPU (Jetson Nano/Orin Class)**
- ARM + NVIDIA GPU + NPU
- 4-16 GB RAM
- NVMe storage
- Built-in WiFi or external
- Optional cellular via USB/M.2
- Cost: $200-500
- Capability: Full SLAM, 8+ camera streams, 360° processing, neural inference
- Suitable for: Full capability deployment, larger homes

**Variant C — x86 Mini PC**
- Intel/AMD x86 processor
- 8-32 GB RAM
- NVMe SSD
- External WiFi card
- Optional cellular via USB
- Cost: $300-800
- Capability: Maximum flexibility, Docker, full SLAM, unlimited streams
- Suitable for: Commercial deployment, multi-building

### Key Difference from Wearable Belt Node

| Aspect | AEGIS-MESH Edge Controller | SENTINEL-WEAR Belt Node |
|--------|---------------------------|-------------------------|
| Power source | AC wall power (unlimited) | Battery (constrained) |
| Form factor | Shelf/rack mounted | Wearable |
| Thermal management | Active cooling possible | Limited by skin contact |
| Compute capacity | High (can run full SLAM 24/7) | Limited by battery |

---

## Sensing Stack

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                         AEGIS-MESH Sensing Stack                             │
├──────────────────────────────────────────────────────────────────────────────┤
│  Always-On Layer (Low Power)                                                 │
│  mmWave radar (presence, gross motion, through-wall, micro-Doppler)          │
│  PIR (binary motion trigger at choke points)                                 │
│  Acoustic (Full Echo profiles: material/event classification, DOA)           │
│  1D ToF (choke-point ranging)                                                │
│  Environmental (temperature, humidity, pressure, VOC, PM2.5)                 │
├──────────────────────────────────────────────────────────────────────────────┤
│  Event-Triggered High-Fidelity Layer                                         │
│  Solid-state LiDAR (3D point cloud, geometry, tracking at 10–50 Hz)          │
│  Neuromorphic Event Cameras (microsecond-latency motion trigger)             │
├──────────────────────────────────────────────────────────────────────────────┤
│  Visual Capture Layer (User-Configured — Any Node)                           │
│  Conventional cameras at any resolution and configuration                    │
│  Continuous recording, activity-triggered recording, or metadata-only        │
│  Live streaming to companion app                                              │
│  Full raw video storage for evidence, review, and SLAM                        │
│  360° multi-camera arrays on LiDAR Variant D                                 │
└──────────────────────────────────────────────────────────────────────────────┘
```

### Sensor Fusion and Physics

**LiDAR vs. Full Echo Acoustic:**
- LiDAR provides high-resolution surface geometry but fails in smoke, fog, steam.
- Full Echo Acoustic analyzes the entire reflected sound waveform. It penetrates smoke/fog and reveals material density (glass break vs. plastic drop, wood vs. ceramic).
- Together: LiDAR for shape, Acoustic for material classification and robustness.

**Event-Based Triggering:**
- Event cameras have microsecond latency, acting as a trigger to wake LiDAR for a high-res scan only when rapid motion is detected.
- Saves power. Optimizes bandwidth. Provides microsecond-scale detection timestamps.

---

## World Model Architecture

### Mode A — Sparse Probabilistic (Always Available)

**Technology:** OMNI-SENSE → PentaTrack → Zone-anchored tracks

**Output:** Per-zone tracked entities with position, velocity, trajectory, classification, and anomaly flags.

**Visualization:** Floor-plan overlay with detection blobs, velocity vectors, zone alerts.

**Use cases:** All real-time alerting, occupancy monitoring, presence detection, fall detection.

### Mode B — Dense SLAM (Hardware Dependent)

**Technology:** LiDAR point clouds + camera visual odometry → SLAM → 3D mesh + object recognition

**Output:** Full 3D geometry of monitored spaces, objects classified and tracked in 3D context.

**Visualization:** 3D walkthrough of the captured environment with entity trajectories and event annotations.

**Use cases:** Forensic review, legal evidence preparation, architecture audit, precise occupancy mapping, renovation planning.

**Hardware requirement:** Edge controller with Linux and sufficient compute. Point cloud + camera data from LiDAR + camera node variants.

---

## 360° LiDAR Node SLAM Integration

### Stationary Sensor SLAM Characteristics

Unlike mobile SLAM (SENTINEL-WEAR), AEGIS-MESH sensors are fixed in place:

**Advantages:**
- Sensors don't move — map is built once per room
- Loop closure when revisiting areas not needed
- Multi-room fusion through doorway overlap
- Persistent map storage — reload at boot

**Challenges:**
- Furniture moves over time → dynamic remap required
- Multiple LiDAR nodes → need coordinate frame alignment
- Doorway transitions → handoff between node coverage zones

### Multi-Room SLAM Architecture

```
Room A (LiDAR Node A) ←─── Doorway overlap ───→ Room B (LiDAR Node B)
        │                                         │
        │                                         │
        ▼                                         ▼
   Room A Map                              Room B Map
        │                                         │
        └────────────── Edge Controller ──────────┘
                              │
                              ▼
                    Unified World Model
                    (Zoned, multi-room)
```

**Edge controller maintains:**
- Per-room SLAM maps
- Zone boundaries and transitions
- Unified world model for companion app visualization
- Cross-room entity tracking

---

## Policy Engine

Four operational modes:

| Mode | Always-On / LiDAR | Identity Node |
|---|---|---|
| **Privacy-First (default)** | Active, metadata-only | Off or metadata-only |
| **Security-First** | Active | Activates on trigger events |
| **Away** | Maximum sensitivity | Activates on trigger events; continuous recording if configured |
| **Silent** | Record-only, no alerts | Follows configured recording policy |

**Mode transitions:** Via companion app or API. All modes and Identity Node activation are user-configured. No mode automatically activates cameras without user having enabled that configuration.

---

## Deployment Paths — Recommended Configurations

### Path 1 — Basic Home Awareness

**Target:** Small apartment or house, basic presence detection, minimal cost.

| Node Type | Count | Variant |
|-----------|-------|---------|
| Choke Point | 2-4 | Always-On A |
| Volume Presence | 1-2 | Always-On B |
| Edge Controller | 1 | Variant A (SBC) |

**Total nodes:** 3-7
**Cost:** $200-500
**Capabilities:** Presence detection, choke-point tracking, basic alerts

### Path 2 — Standard Residential Security

**Target:** Typical 3-4 bedroom home, comprehensive coverage.

| Node Type | Count | Variant |
|-----------|-------|---------|
| Choke Point | 4-6 | Always-On A |
| Volume Presence | 3-5 | Always-On B |
| LiDAR (main areas) | 1-2 | LiDAR B |
| Identity (entries) | 1-2 | Identity A |
| Edge Controller | 1 | Variant B (ARM NPU) |

**Total nodes:** 10-16
**Cost:** $800-1500
**Capabilities:** Full presence tracking, acoustic alerts, entry identification, SLAM in main areas

### Path 3 — Premium Residential Security

**Target:** Large home, comprehensive coverage, 360° capture in main areas.

| Node Type | Count | Variant |
|-----------|-------|---------|
| Choke Point | 6-10 | Always-On A |
| Volume Presence | 4-8 | Always-On B/D |
| LiDAR 360° (main rooms) | 2-4 | LiDAR D |
| Identity (entries) | 2-4 | Identity B/C |
| Edge Controller | 1 | Variant B or C |

**Total nodes:** 15-28
**Cost:** $2000-5000
**Capabilities:** Full 360° coverage, dense SLAM throughout, comprehensive evidence capture

### Path 4 — Commercial / Multi-Building

**Target:** Small business, multi-building property, high-traffic areas.

| Node Type | Count | Variant |
|-----------|-------|---------|
| Choke Point | 10-20 | Always-On A |
| Volume Presence | 5-10 | Always-On B/D |
| LiDAR 360° | 3-6 | LiDAR D |
| Identity | 3-6 | Identity C/D |
| Edge Controller | 1-2 | Variant C (x86) |

**Total nodes:** 25-50+
**Cost:** $5000-15000+
**Capabilities:** Enterprise-grade coverage, full audit trail, multi-building integration

---

## Coverage Geometry Analysis

AEGIS-MESH includes a full coverage analysis engine:

- **Horizon calculation:** Earth-curvature-corrected sensor horizon per node height.
- **Shadow cone analysis:** Identifies blind zones beneath each mast/node.
- **LOS ray marching:** Per-voxel line-of-sight quality from each node.
- **Placement optimizer:** Greedy submodular solver for optimal placement.
- **Pareto frontier:** Coverage vs. cost optimization for any budget.
- **Robustness analysis:** Coverage degradation curves under node failure scenarios.

```bash
cargo run -p aegis-edge-controller -- analyze-coverage --scenario scenarios/my_home.toml
```

---

## Auto-Calibration and Self-Mapping

**Walk-Through Auto-Calibration:**
1. User enables Calibration Mode in companion app.
2. User walks a defined path through the home (30–60 seconds).
3. Multiple nodes detect the user simultaneously. System trilaterates exact node positions.
4. Occlusion map generated automatically — records which nodes observe which areas.
5. Dynamic remapping: if furniture moves, new blind spots flagged and placement adjustments suggested.

---

## Node Placement Strategy

**Layered placement:**

**Layer 1 — Ceiling-Corner Nodes (Primary, ~70% of coverage)**
- Position: Room corners, ceiling height
- Provides: Wide field of view, minimal obstruction
- Covers: Most floor area

**Layer 2 — Low-Angle Nodes (Occlusion breakers)**
- Position: 20-50 cm above floor, wall-mounted
- Provides: Under-furniture coverage, pet tracking
- Covers: Areas shadowed from ceiling

**Layer 3 — Choke-Point Nodes (High value, low cost)**
- Position: Doorways, hallway entries
- Provides: Entry/exit detection, traffic counting
- Covers: Critical transition points

---

## Time Synchronization

All nodes share clock via:
- Mesh protocol timestamp broadcast (edge controller as time master)
- Per-node `omni-sense-time::ClockOffsetEstimator` for drift tracking
- Heartbeat at 100 ms intervals; nodes missing 3 consecutive heartbeats flagged offline

---

## Data Storage and Companion App

### Storage Architecture

All node classes support local data storage:
- microSD card (removable, up to 2 TB) — raw video, point clouds, audio, metadata
- Internal flash — logs, metadata, compressed clips
- Edge controller storage — aggregated from all nodes
- Network attached storage (NAS) — optional via edge controller

### Data Handling Configuration

```toml
[data_handling]
store_raw_video = false              # Enable raw video storage
store_raw_audio = false              # Enable raw audio storage
store_raw_sensor_data = false        # Enable raw radar/LiDAR data storage
storage_target = "internal"          # "sd_card" | "internal" | "network_path" | "companion_app"
retention_days = 30                  # 0 = keep forever
enable_app_streaming = false         # Stream to companion app
continuous_recording = false         # Record continuously vs on-trigger
recording_trigger = "on_detection"   # "always" | "on_detection" | "on_alert" | "manual"
max_storage_mb = 0                   # 0 = unlimited
```

### Companion App

Mobile (iOS/Android) and desktop (Windows/macOS/Linux) apps connect to the edge controller's embedded API server.

**Features:**
- Live detection map (floor-plan overlay with zone activity)
- Live video/audio streams from any configured node
- 360° panoramic view from LiDAR Variant D cameras
- 3D world model viewer (dense SLAM mode)
- Full recordings library with playback, search, and timeline
- Zone activity statistics and heatmaps
- System configuration and node management
- Legal export with cryptographic integrity chain

---

## Privacy and Security

### Local-First, Cloud-Ready

- Operates fully offline. No internet required.
- API-ready: Exposes secure endpoints for Home Assistant, OpenHAB, or custom dashboards.
- User-controlled egress: default configuration blocks external traffic; user must explicitly enable remote access.

### Security

- Message signing: HMAC-SHA256 on all mesh protocol messages.
- Audit logs: All data access and configuration changes logged locally.
- Reproducible builds: Users can verify nodes run published code.
- TLS optional for local API, recommended for remote access.

### Privacy by Architecture — Sensing Layer

The Always-On and LiDAR node firmware binaries have no dependency on `omni-sense-drivers-camera`. These nodes cannot produce imagery — the privacy property is structural and compile-time enforced. Users who want camera capability on these nodes must explicitly add the camera dependency and configure it.

### Privacy by Configuration — Identity Layer

Identity nodes support any data handling mode the user configures. There is no system-imposed restriction on what users capture on their own property. Jurisdiction-specific recording consent laws are the deployer's responsibility.

---

## Software Architecture

### Crate Structure

```
aegis-mesh/
├── crates/
│   ├── aegis-core/              # Shared types, node/zone/event definitions
│   ├── aegis-perception/        # Per-node sensor pipelines (OMNI-SENSE based)
│   ├── aegis-fusion/            # Multi-node detection fusion (CI + JPDA)
│   ├── aegis-tracking/          # PentaTrack bridge, track management
│   ├── aegis-slam/              # Dense SLAM, 360° stitching, world model
│   ├── aegis-placement/         # Node placement optimization (coverage solver)
│   ├── aegis-calibration/       # Walk-through and automated calibration
│   ├── aegis-policy/            # Policy engine (Privacy-First / Security-First / Away)
│   ├── aegis-mesh-protocol/     # Mesh network protocol, message signing
│   ├── aegis-storage/           # Recording management, retention, export
│   ├── aegis-api/               # Companion app REST/WebSocket/media server
│   ├── aegis-alerts/            # Alert classification and routing
│   └── aegis-edge-controller/   # Main binary: runs on the edge controller
├── firmware/                    # Embedded firmware for each node class
├── hardware/                    # PCB schematics, BOMs, test jig
├── apps/                        # Companion app architecture and API spec
├── scenarios/                   # Deployment scenarios (TOML)
├── docs/
│   ├── guides/
│   ├── theory/
│   ├── api/
│   └── protocol/
├── mechanical/
├── production/
├── legal/
├── media/
├── README.md
├── LICENSE
└── CHANGELOG.md
```

---

## Getting Started

```bash
git clone https://github.com/ungatedminds/aegis-mesh
cd aegis-mesh
cargo build --workspace
cargo test --workspace
cargo run -p aegis-edge-controller -- analyze-coverage --scenario scenarios/example_home.toml
cargo run -p aegis-edge-controller -- --simulated --scenario scenarios/example_home.toml
cd firmware && cargo build --target thumbv7em-none-eabihf --bin always_on_node --release
```

---

## License

MIT for code. CERN-OHL-S v2 for hardware. CC BY 4.0 for documentation.
