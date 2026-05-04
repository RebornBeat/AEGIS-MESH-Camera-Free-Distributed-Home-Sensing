# AEGIS-MESH — Distributed Residential Sensing Research Platform

**A privacy-first, geometry-driven distributed perception mesh for residential awareness. AEGIS-MESH fuses LiDAR, mmWave radar, and neuromorphic sensing to answer critical awareness questions—presence, occupancy, anomaly, and identity—without relying on continuous video surveillance.**

[![License: MIT](https://img.shields.io/badge/Code-MIT-blue.svg)](#license)
[![License: CERN OHL](https://img.shields.io/badge/Hardware-CERN%20OHL--S%20v2-green.svg)](#license)
[![Privacy-First](https://img.shields.io/badge/Architecture-Privacy%20By%20Design-green.svg)](#privacy-architecture)
[![Status: Research](https://img.shields.io/badge/Status-Research%20%2F%20Educational-orange.svg)](#scope)

---

## Scope

AEGIS-MESH is a **research and education platform** for studying distributed residential perception. It provides the architecture, firmware reference implementations, software stack, and protocol specifications for building a privacy-preserving sensor mesh.

**Key Thesis:** *Most actionable residential awareness needs—presence, tracking, fall detection, and intrusion—can be solved with higher reliability and better privacy using a distributed mesh of non-imaging sensors than with traditional cameras.*

**Intended Audience:** Hobbyists, makers, privacy researchers, and educators.

**Disclaimers:**
*   **Not a certified life-safety product.** It is not UL-listed or CE-marked for safety functions.
*   **Not a substitute for smoke/CO detectors** or monitored alarm systems.
*   **Not a surveillance system.** The architecture prohibits persistent video recording by default.

---

## Architecture Overview

AEGIS-MESH moves away from the "symmetric grid" approach of placing identical sensors everywhere. Instead, it employs a **Hybrid Node Strategy** that optimizes for cost, power, and information density.

### The Hybrid Node Strategy

Different areas of a home have different sensing requirements. AEGIS-MESH uses three distinct node classes:

1.  **Choke Point Nodes (Low Cost / High Value)**
    *   **Location:** Doorways, hallways, stair entries.
    *   **Hardware:** Cheap sensors (PIR + 1D Time-of-Flight or simple mmWave presence).
    *   **Role:** Binary logic—"Did something pass?" These are information bottlenecks. Knowing who passed where provides immense tracking value with minimal hardware cost.

2.  **Volume Nodes (Rich Sensing)**
    *   **Location:** Living rooms, bedrooms, open kitchens.
    *   **Hardware:** Solid-State LiDAR + mmWave Radar + Microphone Arrays.
    *   **Role:** Tracking *position* and *geometry* within the space. They handle the "Where exactly are they?" and "What is the shape of the room?" questions.

3.  **Identity Nodes (Optional / Opt-In)**
    *   **Location:** Entry points or specific verification zones.
    *   **Hardware:** Low-resolution camera or biometric sensor.
    *   **Role:** Strictly **Identification Only**. These nodes are hardware-isolated (see [Identity Layer](#identity-layer-opt-in)).

---

## Sensing Stack

The sensing stack is layered to balance power consumption against information fidelity.

```
┌──────────────────────────────────────────────────────────────┐
│                   AEGIS-MESH Sensing Stack                   │
├──────────────────────────────────────────────────────────────┤
│  Always-On Layer (Low Power)                                 │
│  ────────────────────────────                                │
│  mmWave radar (presence, gross motion, through-wall)         │
│  PIR (binary motion trigger at choke points)                 │
│  Acoustic (Full Echo profiles for material/event classification) │
├──────────────────────────────────────────────────────────────┤
│  Event-Triggered Layer (High Fidelity)                       │
│  ───────────────────────────────────────                     │
│  Solid-state LiDAR (geometry, precise tracking)              │
│  Neuromorphic Event Sensors (microsecond motion detection)   │
├──────────────────────────────────────────────────────────────┤
│  Identity Layer (Opt-In Only)                                │
│  ───────────────────────────                                 │
│  Local Face Recognition / Biometrics (off-network, isolated) │
│  Hardware Kill Switch REQUIRED on all Identity Nodes         │
└──────────────────────────────────────────────────────────────┘
```

### Sensor Fusion & Physics

A core research contribution of AEGIS-MESH is the fusion of complementary modalities:

*   **LiDAR vs. Full Echo Acoustic:**
    *   **LiDAR** provides high-resolution surface geometry but fails in smoke, fog, or steam.
    *   **Full Echo Acoustic** analyzes the *entire* reflected sound waveform. It penetrates smoke/fog and reveals **material density** (e.g., distinguishing a glass break from a plastic drop) and **internal structure**.
    *   **Fusion Logic:** The system uses LiDAR for shape and Acoustic for material classification/robustness.

*   **Event-Based Triggering:**
    *   Neuromorphic event cameras have microsecond latency but lower spatial resolution. They act as a "trigger," waking up the LiDAR for a high-res scan only when rapid motion is detected, saving power and optimizing bandwidth.

---

## Identity Layer (Opt-In)

Cameras are strictly **excluded** from the primary sensing mesh. They are used *only* for user-initiated identification and are subject to strict architectural isolation:

1.  **Purpose:** Identification (Who is this?), never Sensing (Where is this?).
2.  **Isolation:** Identity nodes run on separate physical hardware links from the mesh.
3.  **Hardware Kill Switch:** Every Identity Node must have a physical switch that cuts power to the imaging sensor.
4.  **Privacy:** Biometric processing (face/voice) happens 100% on-device. No facial embeddings or video streams leave the node.

---

## Auto-Calibration and Self-Mapping

Manual coordinate entry is a barrier to adoption. AEGIS-MESH includes a **Walk-Through Auto-Calibration** mode:

1.  **Activation:** User enables "Calibration Mode" in the companion app.
2.  **The Walk:** The user walks a defined path through the home (30-60 seconds).
3.  **Trilateration:** Multiple nodes detect the user simultaneously. The system uses the user's continuous motion as a known reference signal to trilaterate the exact position of every node.
4.  **Occlusion Mapping:** As the user walks, the system records which nodes can "see" which areas, automatically generating an empirical occlusion map.
5.  **Dynamic Remapping:** If furniture is moved, the system detects new blind spots during normal operation and flags them for the user, suggesting placement adjustments.

*See `software/edge_controller/calibration.py` for implementation details.*

---

## Node Placement (Geometry-Driven)

The system optimizes for **resilience to layout change** rather than brute-force redundancy.

1.  **One-time mapping pass:** A mobile LiDAR (or temporary node) scans the empty home.
2.  **Coverage Solver:** An algorithm computes the minimum set of fixed-node placements to observe every voxel of interest.
3.  **Layered placement:**
    *   **Ceiling-corner nodes** (primary geometry, ~70% of coverage).
    *   **Low-angle nodes** (20-50 cm above floor) to catch under-furniture occlusions.
    *   **Choke-point nodes** at doorways/hallways.

---

## Privacy Architecture

AEGIS-MESH is designed to be private by architecture, not just by policy.

*   **No Cloud by Default.** All processing is local on the edge controller (Jetson-class or mini-PC).
*   **No Raw Sensor Egress.** Only metadata (track positions, event types) leaves the node. No point clouds, audio waveforms, or images traverse the mesh network.
*   **Audit Logs.** Every metadata egress event is logged in an append-only local log.
*   **Reproducible Builds.** Users can verify their nodes run the published code.

---

## Repository Layout

This repository follows a Full Stack Hardware Documentation Framework.

```
aegis-mesh/
├── docs/
│   ├── guides/
│   │   ├── getting_started.md
│   │   ├── node_placement_guide.md
│   │   └── adaptive_remap.md
│   ├── theory/
│   │   ├── sensor_fusion.md           # LiDAR vs Acoustic Physics
│   │   ├── occlusion_driven_placement.md
│   │   ├── privacy_threat_model.md
│   │   └── why_no_cameras.md          # Architectural argument
│   ├── api/
│   └── assets/
├── hardware/
│   ├── schematic/
│   │   ├── always_on_node/            # mmWave + PIR + MCU + audio
│   │   ├── lidar_node/                # solid-state LiDAR + event sensor
│   │   └── identity_node/             # Camera/Biometric (Opt-In, Isolated)
│   ├── gerbers/
│   ├── bom/
│   ├── assembly/
│   ├── testing/
│   └── hardware_config.md
├── firmware/
│   ├── src/
│   │   ├── drivers/
│   │   ├── logic/                     # Event detection, classification
│   │   └── main.c
│   ├── bootloader/
│   ├── tools/
│   ├── platformio.ini
│   └── firmware.md
├── software/
│   ├── edge_controller/
│   │   ├── main.py
│   │   ├── calibration.py             # Walk-through auto-calibration
│   │   └── penta_track_integration.py
│   ├── desktop/                       # Electron config / monitoring app
│   ├── mobile/                        # iOS / Android local app
│   ├── web/                           # Local web dashboard
│   ├── cli/                           # Power-user tools
│   ├── sdk/                           # Python / JS bindings
│   ├── protocol/
│   │   └── protocol_spec.md           # Mesh protocol (signed, local-only)
│   └── software.md
├── mechanical/
│   ├── cad/                           # Wall / ceiling / corner enclosures
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
│   ├── compliance.md
│   ├── research_ethics.md
│   └── tos_compliance.md
├── media/
├── README.md
├── LICENSE
└── CHANGELOG.md
```

---

## Quick Start

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

The edge controller will auto-discover nodes, prompt for a **walk-through calibration**, and produce a placement report.

---

## License

This project uses a split license to protect the open-source nature of the work while ensuring hardware reciprocity:

*   **Code (Software/Firmware):** [MIT License](LICENSE).
*   **Hardware (Schematics/PCB):** [CERN Open Hardware Licence Version 2 - Strongly Reciprocal](LICENSE). This ensures that anyone manufacturing hardware based on these designs must release their modifications.
*   **Documentation:** [CC BY 4.0](LICENSE).
