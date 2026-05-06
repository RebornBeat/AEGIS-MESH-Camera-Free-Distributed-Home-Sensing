# AEGIS-MESH — Distributed Residential Sensing Research Platform

**A geometry-driven distributed perception mesh for residential awareness. AEGIS-MESH fuses LiDAR, mmWave radar, neuromorphic sensing, and optional visual capture to answer awareness questions — presence, occupancy, anomaly, geometry, and identity — with full user control over data capture, storage, and retention.**

[![License: MIT](https://img.shields.io/badge/Code-MIT-blue.svg)](#license)
[![License: CERN OHL](https://img.shields.io/badge/Hardware-CERN%20OHL--S%20v2-green.svg)](#license)
[![Status: Research](https://img.shields.io/badge/Status-Research%20%2F%20Educational-orange.svg)](#scope)

---

## Scope

AEGIS-MESH is a **research and education platform** studying distributed residential perception. It provides architecture, firmware reference implementations, software stack, and protocol specifications for building a high-fidelity sensor mesh.

**Key Thesis:** *Most actionable residential awareness needs — presence, tracking, fall detection, intrusion, geometry — can be solved with higher reliability using a distributed mesh of complementary sensors. The sensing backbone is camera-free by architecture. Visual capture is an additive configurable layer, not the foundation.*

**Intended Audience:** Hobbyists, makers, privacy researchers, educators, and integrators.

**Disclaimers:**
- Not a certified life-safety product (not UL-listed or CE-marked for safety functions).
- Not a substitute for smoke/CO detectors or monitored alarm systems.
- User assumes responsibility for validating configurations for their jurisdiction and use case.

---

## Architecture Overview

### The Core Thesis: Sensing vs. Identification

AEGIS-MESH separates **Sensing** (continuous state observation) from **Identification** (discrete entity recognition):

**Sensing (Continuous):** Monitoring state, motion, position, and behavior. Requires high temporal resolution, robustness to environment (lighting, fog, occlusion), and low latency. Best performed by non-imaging sensors.

**Identification (Configurable):** Determining *who* a specific individual is. Traditionally requires imagery but is an episodic or continuous query depending on user configuration. AEGIS-MESH supports both — episodic identification triggered by the sensing layer, or continuous visual recording if the user configures it.

### The Hybrid Node Strategy

Three node classes compose the mesh:

**1. Choke Point Nodes (Low Cost / High Value)**
- Location: Doorways, hallways, stair entries
- Hardware: PIR + 1D ToF or simple mmWave presence
- Role: Binary detection — "Did something pass?"

**2. Volume Nodes (Rich Sensing)**
- Location: Living rooms, bedrooms, open kitchens
- Hardware: Solid-state LiDAR + mmWave Radar + Microphone Arrays
- Role: Position and geometry tracking

**3. Identity Nodes (Configurable)**
- Location: Entry points or verification zones
- Hardware: Configurable visual/biometric sensor (all specifications research-determined)
- Role: Identification — "Who is this?"
- Mode A — Passive: Triggered only when the sensing layer detects an anomaly
- Mode B — Active: Continuous visual recording per user configuration
- Privacy controls: Software-based (scheduling, activity triggers) and/or optional hardware switches per user preference

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
│  360° multi-camera arrays on Identity Nodes (research variant)               │
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

## Node Classes — All Variants

### Always-On Node

**Variant A — Choke Point**
- PIR + 1D ToF + MCU + Mesh Radio
- Lowest power, lowest cost
- Binary detection at doorways

**Variant B — Volume Presence**
- mmWave Radar + Microphone Array + Environmental + MCU + Mesh Radio
- Full presence and acoustic classification
- No camera, no LiDAR

**Variant C — Volume Presence + Acoustic**
- All of Variant B + higher-density microphone array (6-element)
- Enhanced acoustic material classification and DOA

**Variant D — Volume Presence + Optional Camera**
- All of Variant B + optional camera module (user-configured)
- Camera connected via configurable interface (MIPI CSI, USB)
- Local storage on-node SD card
- Live streaming to companion app

**All always-on variants:**
- Camera-free primary sensing architecture
- Acoustic processing on-MCU
- Raw audio storage and streaming user-configurable
- No mandatory camera prohibition — if user adds camera to any variant, it is fully supported

### LiDAR Node

**Variant A — Basic LiDAR**
- Solid-state LiDAR (Livox MID-360 class) + event camera (trigger) + MCU + Mesh
- Event-triggered activation for power efficiency

**Variant B — Full LiDAR**
- Solid-state LiDAR + event camera + mmWave radar + MCU + Mesh
- Dual modality for fog/smoke robustness

**Variant C — LiDAR + Camera**
- All of Variant B + conventional camera (high resolution, configurable)
- Full SLAM capability (LiDAR + visual odometry from camera)
- 360° camera array variant available (4 cameras covering hemisphere)

**Variant D — LiDAR + 360° Camera**
- All of Variant B + 4–8 camera array covering full 360° hemisphere
- Room-scale 360° video capture at ceiling mount position
- Dense SLAM with 360° visual coverage
- This is the most capable and most expensive variant
- Best suited for open-plan spaces requiring full room coverage

### Identity Node

**Variant A — Metadata-Only (Default)**
- Configurable camera/biometric sensor
- On-device inference
- Metadata transmission only (classification tags)
- No local storage by default

**Variant B — Local Storage**
- All of Variant A + SD card
- Raw video, clips, and metadata stored locally
- User accesses recordings via companion app or physical SD removal

**Variant C — App Streaming + Storage**
- All of Variant B + Wi-Fi streaming
- Live footage available in companion app
- Full recordings library

**Variant D — Multi-Camera Array**
- Multiple cameras at configurable angles
- 360° coverage of entry point or verification zone
- All camera streams stored and available

**Privacy controls (all user-configured — none mandatory):**
- Hardware power switch (user-installed optional — cuts camera power rail)
- Software activity scheduling (camera active only during away mode, etc.)
- Trigger-based activation (sensing layer triggers camera)
- Continuous recording
- Any combination of the above

**No mandatory hardware kill switch requirement.** The architecture supports hardware switches for users who want them. It also fully supports software-only privacy controls for users who prefer flexibility. Jurisdiction-specific legal requirements are the deployer's responsibility.

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
│   └── src/bin/
│       ├── always_on_node.rs
│       ├── lidar_node.rs
│       └── identity_node.rs
├── hardware/                    # PCB schematics, BOMs, test jig
│   ├── schematic/
│   │   ├── always_on_node/      # All variants
│   │   ├── lidar_node/          # All variants including 360° camera
│   │   └── identity_node/       # All variants, optional hardware switch
│   ├── testing/
│   │   └── test_jig_pcb/
│   └── hardware_config.md
├── apps/                        # Companion app architecture and API spec
├── scenarios/                   # Deployment scenarios (TOML)
├── docs/
│   ├── guides/
│   │   ├── getting_started.md
│   │   ├── node_placement_guide.md
│   │   └── adaptive_remap.md
│   ├── theory/
│   │   ├── sensing_vs_identification.md
│   │   ├── sensor_fusion.md
│   │   ├── occlusion_driven_placement.md
│   │   ├── privacy_threat_model.md
│   │   ├── coverage_geometry.md
│   │   ├── mast_elevation_analysis.md
│   │   ├── future_research.md
│   │   └── sensing_vs_identification.md
│   ├── api/
│   │   └── api_spec.md
│   └── protocol/
│       └── protocol_spec.md
├── mechanical/
│   ├── cad/
│   ├── stl/
│   ├── drawings/
│   └── mechanical_spec.md
├── production/
│   ├── sop/
│   └── quality/
├── legal/
│   ├── compliance.md
│   ├── research_ethics.md
│   └── tos_compliance.md
├── media/
├── README.md
├── LICENSE
└── CHANGELOG.md
```

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

## World Model Architecture

AEGIS-MESH supports both sparse probabilistic and dense SLAM world models.

### Sparse World Model (Mode A — Always Available)

**Technology:** OMNI-SENSE → PentaTrack → Zone-anchored tracks

**Output:** Per-zone tracked entities with position, velocity, trajectory, classification, and anomaly flags.

**Visualization:** Floor-plan overlay with detection blobs, velocity vectors, zone alerts.

**Use cases:** All real-time alerting, occupancy monitoring, presence detection.

### Dense SLAM World Model (Mode B — Hardware Dependent)

**Technology:** LiDAR point clouds + camera visual odometry → SLAM → 3D mesh + object recognition

**Output:** Full 3D geometry of monitored spaces, objects classified and tracked in 3D context.

**Visualization:** 3D walkthrough of the captured environment with entity trajectories and event annotations.

**Use cases:** Forensic review, legal evidence preparation, architecture audit, precise occupancy mapping.

**Hardware requirement:** Edge controller with Linux and sufficient compute (Raspberry Pi 4 class minimum). Point cloud + camera data from LiDAR + camera node variants.

---

## Auto-Calibration and Self-Mapping

**Walk-Through Auto-Calibration:**
1. User enables Calibration Mode in companion app.
2. User walks a defined path through the home (30–60 seconds).
3. Multiple nodes detect the user simultaneously. System trilaterates exact node positions.
4. Occlusion map generated automatically — records which nodes observe which areas.
5. Dynamic remapping: if furniture moves, new blind spots flagged and placement adjustments suggested.

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

## Data Storage and Companion App

### Storage Architecture

All node classes support local data storage:
- microSD card (removable, up to 2 TB) — raw video, point clouds, audio, metadata
- Internal flash — logs, metadata, compressed clips
- Edge controller storage — aggregated from all nodes
- Network attached storage (NAS) — optional via edge controller

### Data Handling Configuration (`aegis-mesh.toml`)

```toml
[data_handling]
store_raw_video = false              # Enable raw video storage from Identity/LiDAR nodes
store_raw_audio = false              # Enable raw audio storage from acoustic nodes
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
- 360° panoramic view from nodes with multi-camera arrays
- 3D world model viewer (dense SLAM mode)
- Full recordings library with playback, search, and timeline
- Gait analytics and event history
- Zone activity statistics and heatmaps
- System configuration and node management
- Legal export with cryptographic integrity chain

See `apps/README.md` for full API and architecture.

---

## Privacy and Connectivity Architecture

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

## Node Placement Geometry

**Geometry-driven placement (not brute-force redundancy):**

1. One-time mapping pass (mobile LiDAR scan or temporary node)
2. Coverage solver: minimum node placements to observe every voxel of interest
3. Layered placement:
   - Ceiling-corner nodes (primary geometry, ~70% of coverage)
   - Low-angle nodes (20–50 cm above floor) for under-furniture coverage
   - Choke-point nodes at doorways/hallways

---

## Time Synchronization

All nodes share clock via:
- Mesh protocol timestamp broadcast (edge controller as time master)
- Per-node `omni-sense-time::ClockOffsetEstimator` for drift tracking
- Heartbeat at 100 ms intervals; nodes missing 3 consecutive heartbeats flagged offline

---

## Configuration

```toml
[deployment]
building_floor_count = 1
floor_height_m = 2.7
total_area_m2 = 120.0

[policy]
default_mode = "PrivacyFirst"
auto_away_after_minutes = 30
alert_cooldown_s = 60.0

[mesh_network]
protocol = "wifi"
edge_controller_address = "192.168.1.100:8080"
heartbeat_interval_ms = 100
message_signing = true

[data_handling]
store_raw_video = false
store_raw_audio = false
retention_days = 30
enable_app_streaming = false
recording_trigger = "on_detection"

[companion_app]
enabled = true
api_port = 8080
media_port = 9090
enable_remote_access = false
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
