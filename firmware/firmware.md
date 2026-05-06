# Firmware Architecture — AEGIS-MESH

**Version:** 0.1 | **Status:** Research | **Target:** `no_std` (ARM Cortex-M / RISC-V)

---

## 1. Purpose

This document specifies the firmware architecture for AEGIS-MESH sensing nodes. Three node classes (AlwaysOn, LiDAR, Identity) each have a dedicated firmware binary compiled for the node's MCU target.

**Firmware responsibilities:**
- Drive all sensors on the node (mmWave, acoustic, LiDAR, event camera, cameras, environmental).
- Perform on-node processing to reduce mesh bandwidth.
- Manage local storage (SD card, internal flash) per user data handling configuration.
- Communicate with the edge controller via the mesh radio.
- For Always-On Variant D and all camera-capable nodes: manage camera capture per user data handling policy.

**Scope:** Sensing-and-information only. No effectors. No actuation. Data handling is user-configured — firmware enforces the user's configuration without imposing additional restrictions.

---

## 2. Target Platforms

**Primary Targets:**
- ARM Cortex-M33 (STM32U5, nRF5340)
- ARM Cortex-M7 (STM32H7, NXP i.MX RT1060)
- RISC-V (ESP32-S3 for Wi-Fi-enabled variants)

**Resource Profile (Minimum):**
- Flash: 512 KB (1 MB preferred)
- RAM: 256 KB
- Storage: microSD card (for recording-capable nodes)

---

## 3. Workspace Structure

```
firmware/
├── Cargo.toml
├── build.rs
├── memory.x
├── firmware.md
├── README.md
└── src/
    ├── lib.rs
    ├── bin/
    │   ├── always_on_node.rs       # Variants A–C (no camera)
    │   ├── always_on_node_cam.rs   # Variant D (with camera)
    │   ├── lidar_node.rs           # Variants A–B (no camera)
    │   ├── lidar_node_cam.rs       # Variant C (with camera)
    │   ├── lidar_node_360.rs       # Variant D (360° camera array)
    │   └── identity_node.rs        # All identity variants
    ├── drivers/
    │   ├── mod.rs
    │   ├── mmwave.rs
    │   ├── lidar.rs
    │   ├── event_sensor.rs
    │   ├── acoustic.rs
    │   ├── pir.rs
    │   ├── tof.rs
    │   ├── environmental.rs
    │   ├── camera.rs               # Used by cam variants
    │   ├── camera_360.rs           # Used by lidar_node_360
    │   └── storage.rs
    ├── logic/
    │   ├── mod.rs
    │   ├── presence_detection.rs
    │   ├── recording_manager.rs
    │   └── power_management.rs
    └── mesh_radio/
        ├── mod.rs
        └── protocol.rs
```

---

## 4. Node Binary Descriptions

### `always_on_node.rs` (Variants A–C)

Camera-free. No camera driver linked. Handles: PIR, ToF, mmWave radar, microphone array, environmental sensor. These variants are camera-free at the compile-time dependency level.

### `always_on_node_cam.rs` (Variant D)

All of the above plus camera driver. Camera data handling per `aegis-mesh.toml`. Optional hardware power switch support.

### `lidar_node.rs` (Variants A–B)

LiDAR + event camera + optional mmWave. No conventional camera. Event camera provides microsecond-latency LiDAR trigger.

### `lidar_node_cam.rs` (Variant C)

All of Variant A/B plus conventional camera. Camera provides visual odometry for SLAM.

### `lidar_node_360.rs` (Variant D)

All of Variant A/B plus 360° camera array. Handles: camera multiplexing, hardware sync signal generation, frame capture, belt-node transmission pipeline.

### `identity_node.rs`

Configurable identification sensor. Reads data handling mode from config. Supports: metadata-only, local storage, app streaming, full recording. Optional hardware power switch support.

---

## 5. Driver Layer

**`camera.rs` (single camera):**
- MIPI CSI-2 or DVP interface.
- Configured per data handling mode.
- Hardware power switch check at boot (if pin is configured in hardware config).
- Operates per data handling policy:
  - Metadata-only: inference on-MCU.
  - Local storage: frames to SD card.
  - App streaming: encoded stream to edge controller over mesh radio (then Wi-Fi relay to app).
  - Full recording: continuous capture with integrity chain.

**`camera_360.rs` (360° camera array):**
- Multiplexes N camera MIPI lanes.
- Generates FSYNC hardware sync pulse to all cameras simultaneously.
- Loads calibration JSON from flash at boot.
- Streams raw or compressed frames to edge controller.
- Edge controller handles stitching (belt-equivalent for AEGIS-MESH = edge controller compute).

**`storage.rs`:**
- microSD SPI interface via `embedded-sdmmc`.
- FAT32 filesystem.
- `open_recording()`, `write_frame()`, `close_recording()`, `list_recordings()`, `delete_recording()`.
- SHA-256 hash computation on completed recordings.
- Integrity manifest generation and storage.
- Retention enforcement per `max_storage_mb`.

---

## 6. Data Handling

All data handling is user-configured in `aegis-mesh.toml`. No system-imposed restrictions.

```toml
[data_handling]
store_raw_video = false
store_raw_audio = false
store_raw_sensor_data = false
recording_trigger = "on_detection"
retention_days = 90
max_storage_mb = 0

[identity_node.data]
mode = "metadata_only"
store_raw = false
```

**Camera data handling modes:** metadata-only, local_storage, app_streaming, full_recording — as detailed in `hardware/schematic/identity_node/README.md`.

**Optional hardware power switch:** If user installs a hardware power switch on the camera rail, firmware reads the GPIO at boot. If DISABLED, camera is not initialized. If not installed, software config is the sole control.

---

## 7. Build System

```toml
[workspace]
members = ["."]

[package]
name = "aegis-mesh-firmware"
version = "0.1.0"
edition = "2021"

[dependencies]
cortex-m = "0.7"
cortex-m-rt = "0.7"
embedded-hal = "1.0"
panic-halt = "0.2"
embedded-sdmmc = "0.7"

[features]
default = []
camera = []
camera_360 = ["camera"]
hw_camera_switch = []

[[bin]]
name = "always_on_node"
path = "src/bin/always_on_node.rs"

[[bin]]
name = "always_on_node_cam"
path = "src/bin/always_on_node_cam.rs"
required-features = ["camera"]

[[bin]]
name = "lidar_node"
path = "src/bin/lidar_node.rs"

[[bin]]
name = "lidar_node_cam"
path = "src/bin/lidar_node_cam.rs"
required-features = ["camera"]

[[bin]]
name = "lidar_node_360"
path = "src/bin/lidar_node_360.rs"
required-features = ["camera_360"]

[[bin]]
name = "identity_node"
path = "src/bin/identity_node.rs"
required-features = ["camera"]
```

---

## 8. Legal & Compliance

- Sensing-only effectors: no actuation path in any binary.
- Data handling: user-configured. No restrictions imposed.
- Recording laws: deployer's responsibility.
- Radio: FCC Part 15, CE RED compliance required for mesh radio module.
- Battery: hardware overcharge/overdischarge protection required; firmware provides secondary cutoff.
