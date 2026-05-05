# Getting Started — AEGIS-MESH

---

## Prerequisites

- Rust 1.75+ with the stable toolchain (`rustup update stable`)
- AEGIS-MESH repository cloned
- Either: physical sensor nodes (see `hardware/`) OR: no hardware needed for simulation mode

---

## Step 1: Build

```bash
git clone https://github.com/<user>/aegis-mesh.git
cd aegis-mesh
cargo build --release
```

The build produces:
- `target/release/aegis-edge-controller` — the main process that runs on your edge device (Jetson, Raspberry Pi, home server)
- Firmware binaries in `firmware/target/` (requires embedded target setup, see below)

---

## Step 2: Run in Simulation Mode

The easiest way to start. No hardware required.

```bash
cargo run --bin aegis-edge-controller -- \
  --config scenarios/single_room.toml \
  --mode simulation
```

The edge controller starts, loads the scenario, initializes simulated sensors (via OMNI-SENSE simulation implementations), runs the perception pipeline, and serves the local dashboard at `http://localhost:7171`.

Open `http://localhost:7171` to see:
- A floor-plan view with simulated tracks
- Coverage map showing which areas each node observes
- Event log showing detections and alerts

---

## Step 3: Walk-Through Calibration

In simulation mode, calibration is pre-computed from the scenario geometry. On real hardware:

```bash
# Start calibration mode
curl -X POST http://localhost:7171/api/calibration/start

# Walk through your home at normal pace (30–60 seconds)
# The system detects your movement and trilaterates node positions

# Check calibration status
curl http://localhost:7171/api/calibration/status

# When confidence > 0.85 for all nodes, calibration is complete
```

After calibration, the system generates:
- A 3D node placement map
- An occlusion map showing coverage and blind spots
- A placement report with suggestions if coverage gaps exist

---

## Step 4: Configure Policy Mode

Default is Privacy-First mode: Identity nodes are dormant; all tracks are anonymous geometry.

```bash
# Switch to Security-First mode
curl -X POST http://localhost:7171/api/policy/mode \
  -H "Content-Type: application/json" \
  -d '{"mode": "SecurityFirst", "triggers": ["UnknownObjectInDriveway", "MotionInRestrictedZone"]}'

# Return to Privacy-First
curl -X POST http://localhost:7171/api/policy/mode \
  -d '{"mode": "PrivacyFirst"}'
```

---

## Step 5: Flash Node Firmware

Requires the embedded Rust toolchain (`rustup target add thumbv7em-none-eabihf`).

```bash
# Flash an always-on node
cd firmware
cargo build --bin always_on_node --release --target thumbv7em-none-eabihf
# Flash with your programmer (OpenOCD, probe-rs, etc.)
probe-rs download target/thumbv7em-none-eabihf/release/always_on_node --chip <your-chip>
```

---

## Step 6: Node Discovery

When real nodes boot, they broadcast on the mesh network. The edge controller auto-discovers them:

```bash
curl http://localhost:7171/api/nodes
# Returns list of discovered nodes with health status
```

Newly discovered nodes are in "uncalibrated" state. Run a calibration walk to establish their positions.

---

## Scenario File Format

Scenarios are TOML files that configure the simulation or provide the home geometry for real deployments:

```toml
[region]
bounds_m = [10.0, 8.0, 3.0]    # x, y, z bounds in meters
voxel_resolution = 0.25

[[nodes]]
id = "ceiling_living_room_north"
class = "LidarNode"
position_m = [2.0, 7.5, 2.8]
sensors = ["SolidStateLidar", "MmWaveRadar", "Acoustic"]

[[nodes]]
id = "choke_front_door"
class = "AlwaysOnNode"
position_m = [0.0, 4.0, 1.5]
sensors = ["Pir", "MmWavePresence"]

[atmosphere]
profile = "Indoor"
```
