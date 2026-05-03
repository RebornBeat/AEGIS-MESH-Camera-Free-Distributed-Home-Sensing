# 2. AEGIS-MESH — Camera-Free Distributed Home Sensing

**A privacy-first distributed perception mesh for residential awareness using LiDAR, mmWave radar, and neuromorphic event sensing — no traditional cameras, no recorded video.**

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](#license)
[![Privacy-First](https://img.shields.io/badge/Privacy-First-green.svg)](#privacy-architecture)
[![Status: Research](https://img.shields.io/badge/Status-Research%20%2F%20Educational-orange.svg)](#scope)

---

## Scope

AEGIS-MESH is a **research and education platform** for studying camera-free residential perception. It is built around a single thesis: *most "smart home security" needs — presence, occupancy, motion, fall detection, intrusion, pet vs. person, abnormal activity — can be solved with a distributed mesh of non-imaging sensors, with strictly better privacy properties than any camera-based system.*

This repository provides the architecture, firmware reference implementation, software stack, and protocol specifications for building such a mesh. It is intended for hobbyists, makers, privacy researchers, and educators. It is **not** a certified life-safety product. It is not UL-listed. It is not a substitute for a monitored alarm or smoke / CO detection. Nothing in this repository should be deployed as the sole protection for any person or property.

---

## Why "No Cameras"

Cameras solve identification well and almost everything else poorly:

- They record imagery of people in their own homes — a permanent privacy liability.
- They fail in darkness without IR illumination (which itself is detectable).
- They fail behind furniture, in steam, in smoke, in dust.
- They generate enormous data volumes that demand cloud processing — which means private home video leaving the house.
- They invite a class of network attacks (compromised camera = inside-the-home surveillance device).

A non-imaging sensor mesh — LiDAR for geometry, mmWave for through-fabric motion, event sensors for fast change detection, acoustic for material/event classification — can answer almost every actionable question ("is someone home, where, doing what, are they OK") *without ever forming an image of a person*. The data that leaves a node is metadata: *"motion event in zone 3, human-class, walking, 1.2 m/s, 0.94 confidence."* No pixels.

---

## Sensing Stack

```
┌──────────────────────────────────────────────────────────────┐
│                   AEGIS-MESH Sensing Stack                   │
├──────────────────────────────────────────────────────────────┤
│  Always-On Layer (cheap, low power)                          │
│  ──────────────────────────────────                          │
│  mmWave radar (presence, gross motion, through-wall)         │
│  PIR (corroborating)                                         │
│  Acoustic anomaly (glass break, smoke alarm chirp, fall      │
│    impact signature, pet vs. person vocalization)            │
├──────────────────────────────────────────────────────────────┤
│  Event-Triggered Layer (activated on motion)                 │
│  ───────────────────────────────────────────                 │
│  Solid-state LiDAR node (room geometry, person tracking)     │
│  Neuromorphic event sensor (microsecond change detection,    │
│    insect / fast-motion / sudden-event classification)       │
├──────────────────────────────────────────────────────────────┤
│  Optional Identity Layer (opt-in, never default)             │
│  ───────────────────────────────────────────────             │
│  Local face recognition node (off-network, on-device)        │
│  Voice biometrics (wake-word + speaker ID, on-device)        │
│  Both layers are physically isolated from the always-on mesh │
│  and can be disabled at the hardware switch level.           │
└──────────────────────────────────────────────────────────────┘
```

The key architectural rule: **identity-capable sensors are physically separate, opt-in, and air-gapped from the always-on mesh.** An attacker compromising the always-on mesh learns occupancy patterns; they do not learn faces or voices.

---

## Node Placement (Geometry-Driven)

AEGIS-MESH does not blanket a home in mirrored sensors. It uses an *occlusion-driven* placement model:

1. **One-time mapping pass.** A mobile LiDAR (handheld or temporary node) scans the empty home and produces a 3D voxel map.
2. **Coverage solver.** A solver computes the minimum set of fixed-node placements that observe every voxel of interest from at least one good angle.
3. **Layered placement.**
   - **Ceiling-corner nodes** (primary geometry, ~70% of coverage).
   - **Low-angle nodes** at 20–50 cm above floor along walls (catch under-furniture occlusions).
   - **Choke-point nodes** at doorways, hallways, stairs (highest information density per node).
   - **Optional upward nodes** only where ceiling-level activity matters.
4. **Adaptive remap.** Periodic low-frequency LiDAR rescans detect furniture changes and update the occlusion map. New blind spots are flagged in the companion app with placement suggestions.

The system optimizes for *resilience to layout change* — not for brute-force redundancy. A typical 3-bedroom home runs on ~6–10 fixed nodes plus 2–3 choke-point nodes.

---

## PentaTrack Integration

Every fixed node feeds bounding-box detections into [PentaTrack](https://github.com/RebornBeat/PentaTrack) running on the local edge controller. PentaTrack handles:

- Predictive multi-center tracking across the mesh.
- Object-type awareness (`pedestrian`, `pet`, `vehicle-near-house`) with drift profiles.
- Anomaly detection (fall = vertical drift exceeding human walking profile; intrusion = pedestrian drift profile in a no-occupancy time window).
- Cross-node track handoff as people move between rooms.

The companion app surfaces tracks as anonymous geometric figures on a floor plan — never as imagery.

---

## Privacy Architecture

- **No cloud by default.** All processing is local on the edge controller (Jetson-class or mini-PC).
- **No raw sensor egress.** Only metadata — never point clouds, never event streams, never audio waveforms — leaves the home network unless the user explicitly enables a remote feature.
- **Hardware kill switches** on every identity-capable sensor.
- **Open audit log.** Every metadata egress event is logged in an append-only local log the user can inspect.
- **Reproducible firmware builds** so users can verify their nodes run the published code.

---

## Repository Layout (Full Stack Framework)

```
aegis-mesh/
├── docs/
│   ├── guides/
│   │   ├── getting_started.md
│   │   ├── node_placement_guide.md
│   │   └── adaptive_remap.md
│   ├── theory/
│   │   ├── why_no_cameras.md
│   │   ├── occlusion_driven_placement.md
│   │   └── privacy_threat_model.md
│   ├── api/
│   └── assets/
├── hardware/
│   ├── schematic/
│   │   ├── always_on_node/      # mmWave + PIR + MCU + audio
│   │   └── lidar_node/          # solid-state LiDAR + event sensor
│   ├── gerbers/
│   ├── bom/
│   ├── assembly/
│   ├── datasheets/
│   ├── testing/
│   └── hardware_config.md
├── firmware/
│   ├── src/
│   │   ├── drivers/
│   │   ├── logic/               # event detection, classification
│   │   └── main.c
│   ├── bootloader/
│   ├── tools/
│   ├── platformio.ini
│   └── firmware.md
├── software/
│   ├── edge_controller/         # Local fusion + PentaTrack runner
│   ├── desktop/                 # Electron config / monitoring app
│   ├── mobile/                  # iOS / Android local app (LAN-only by default)
│   ├── web/                     # Local web dashboard
│   ├── cli/                     # Power-user tools
│   ├── sdk/                     # Python / JS bindings
│   ├── protocol/
│   │   └── protocol_spec.md     # Mesh protocol (signed, local-only)
│   └── software.md
├── mechanical/
│   ├── cad/                     # Wall / ceiling / corner enclosures
│   ├── stl/
│   ├── drawings/
│   └── mechanical_spec.md
├── production/
│   ├── sop/
│   ├── quality/
│   └── packaging/
├── legal/
│   ├── certifications/
│   ├── licenses/
│   └── compliance.md            # FCC for mmWave + RoHS
├── media/
├── README.md
├── LICENSE
└── CHANGELOG.md
```

---

## Quick Start (Reference Build)

```bash
# Clone
git clone https://github.com/<user>/aegis-mesh.git
cd aegis-mesh

# Flash a node
cd firmware
pio run -e always_on_node -t upload

# Run the edge controller
cd ../software/edge_controller
docker compose up

# Open the local dashboard
open http://aegis-mesh.local
```

The edge controller will auto-discover nodes on the local network, prompt for a one-time LiDAR mapping pass, and produce a placement report.

---

## What This Is Not

- It is not a UL/CE-certified life-safety system.
- It is not a substitute for smoke / CO / fire alarms.
- It is not a substitute for a monitored alarm.
- It does not call emergency services automatically (no autodial).
- It is a research and education platform.

---

## License

MIT for code. CERN-OHL-S v2 for hardware schematics. CC BY 4.0 for documentation.
