# Identity Node — Hardware Schematic Reference Design

**Project:** AEGIS-MESH
**Node Class:** IdentityNode — Configurable Visual / Biometric Identification
**Status:** Reference Design — All Variants
**License:** CERN-OHL-S v2

---

## 1. Connectivity Architecture

**The identity node has NO WiFi and NO cellular connectivity.**

All identity node data transmits via mesh radio to the edge controller exclusively. The edge controller is the sole external network gateway.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        IDENTITY NODE CONNECTIVITY                          │
│                                                                              │
│  Identification Sensor                                                      │
│       │                                                                      │
│       ▼                                                                      │
│  MCU (Identity Processing)                                                  │
│       │                                                                      │
│       ▼                                                                      │
│  Mesh Radio (BLE / Thread / PoE)                                            │
│       │                                                                      │
│       └──► Edge Controller (sole gateway)                                   │
│                 │                                                            │
│                 ├──► WiFi ──► Companion App (local network)                │
│                 ├──► Cellular ──► Remote App / Emergency Contact           │
│                 └──► BLE Direct ──► Companion App (fallback)               │
│                                                                              │
│  ⚠️  IDENTITY NODE = MESH RADIO ONLY (no WiFi, no cellular)                 │
│  ⚠️  ALL EXTERNAL CONNECTIVITY THROUGH EDGE CONTROLLER                       │
└─────────────────────────────────────────────────────────────────────────────┘
```

**Data flow for camera data:**
```
Camera → MCU → Encode (if streaming) → Mesh Radio → Edge Controller → WiFi → Companion App
                                                                              │
                                                                              └──► SD Card (recording)
```

---

## 2. Purpose

The Identity Node is the configurable identification layer of AEGIS-MESH. It activates based on user-configured policy — either automatically triggered by the sensing layer when anomalous activity is detected, or running continuously per the user's data handling configuration.

**What it does:**
- Captures visual or biometric data from a configurable sensor.
- Processes locally for classification (metadata-only mode) or stores/streams the full capture (per user configuration).
- Transmits classification results and/or raw media to the edge controller per the user's data handling policy.
- All data handling modes are available: metadata-only, local recording, app streaming, continuous capture.

**Physical isolation from sensing mesh:**
- Separate hardware from Always-On and LiDAR nodes.
- Optional separate power rail controllable by a hardware switch (user-installed if desired).
- Separate MCU that does not share bus with sensing mesh MCU.

---

## 3. Core Architecture

```
Power Supply (5V USB / Battery / PoE)
       │
       ├──[ Optional Hardware Power Switch (user-installed) ]────────────┐
       │         │                                                       │
       │    Cuts V_SENSOR rail                             GPIO read by MCU (if installed)
       │    (if switch present and DISABLED)                             │
       │         │                                                       │
       ▼         ▼                                                       │
  ┌─────────┐  V_SENSOR ──────────────────────────────────────────────  │
  │ MCU     │◄──── CAMERA_HW_SW_GPIO (LOW = disabled, if pin present) ◄─┘
  └────┬────┘
       │
       │ (CSI / DVP / USB per sensor type)
       ▼
┌─────────────────────┐
│ Identification      │
│ Sensor (Configurable│
│ — see Section 5)    │
└─────────────────────┘
       │
       ▼
┌─────────────────────┐
│ Local Storage       │   SD card or internal flash
│ (Optional — per     │   Raw video, clips, images
│  data config)       │
└─────────────────────┘
       │
       ▼
┌─────────────────────┐
│ Mesh Radio          │   To edge controller ONLY
│ (BLE / Thread / PoE)│   Transmits per configured data handling mode:
│                     │   - Metadata only (classification tags)
│                     │   - Recording availability events
│                     │   - Live stream relay via edge controller
└─────────────────────┘
```

---

## 4. Data Handling — Fully User Configurable

All data handling is configured in `aegis-mesh.toml`. The architecture supports any combination. **No restrictions are imposed by the system.**

### 4.1 Mode A — Metadata-Only (Default)

- On-device inference produces classification tags.
- Only `IdentificationEvent` metadata transmitted over mesh.
- Example: `{"class": "KnownResident", "confidence": 0.94}`
- No raw frames stored or transmitted.
- Minimum bandwidth and storage footprint.
- Lowest power consumption for camera-enabled operation.

### 4.2 Mode B — Local Storage

- Raw video, images, or clips written to on-node SD card or internal flash.
- Available for review via AEGIS-MESH companion app (through edge controller API) or direct SD card access.
- Suitable for security investigations, legal evidence, system calibration, incident review.
- Storage capacity and retention period user-configured.
- `RecordingAvailable` mesh event sent to edge controller on completion.

### 4.3 Mode C — App Streaming

- Raw or compressed video/audio streamed to the AEGIS-MESH companion app.
- Data path: Camera → MCU encode → Mesh Radio → Edge Controller → WiFi → Companion App
- Users view live footage and review recordings from mobile or desktop.
- Simultaneous local storage available if configured.
- Edge controller handles the relay from mesh radio to WiFi.

### 4.4 Mode D — Continuous Recording

- All sensor data captured continuously or on-trigger.
- Recordings carry embedded integrity metadata (timestamp, node ID, cryptographic hash chain) for legal use.
- Users export for legal proceedings, insurance claims, or personal records.
- Full audit trail with cryptographic integrity.

### 4.5 Configuration (`aegis-mesh.toml`)

```toml
[identity_node.data]
# Data handling mode
mode = "metadata_only"               # "metadata_only" | "local_storage" | "app_streaming" | "full_recording"

# Storage settings
store_raw = false                    # Enable raw frame/video storage
storage_target = "sd_card"          # "sd_card" | "internal_flash"
retention_days = 90                  # 0 = keep forever
continuous_recording = false
recording_trigger = "on_detection"   # "always" | "on_detection" | "on_alert" | "manual"
max_storage_mb = 0                   # 0 = unlimited

# Streaming settings
enable_streaming = false
stream_quality = "high"              # "low" | "medium" | "high" | "raw"
stream_codec = "h264"                # "h264" | "h265" | "mjpeg"

# Integrity chain
include_integrity_chain = true      # SHA-256 hash chain for legal evidence
```

---

## 5. Privacy Architecture — User Configured

The privacy model is **hardware-first, user-configured second**.

### 5.1 Hardware Kill Switch (Optional — User-Installed)

**Purpose:** Guaranteed physical camera disable without software dependency.

**Implementation:**
- Physical slider or toggle that cuts the `V_SENSOR` power rail.
- When DISABLED: sensor is physically unpowered; no capture occurs regardless of software configuration.
- When ENABLED: all software data handling modes are available.
- **This is an option the user can add — it is not architecturally required.**
- Some users want guaranteed physical camera disable without software interaction; this enables that.
- Other users prefer software-only control; no switch is needed.

**Hardware implementation:**
- PCB provides MOSFET footprint on `V_SENSOR` rail.
- Switch footprint provided for standard SMD or through-hole switches.
- MCU GPIO `CAMERA_HW_SW_GPIO` reads switch state at boot if pin is populated.
- If DISABLED detected: firmware does not initialize camera, reports `CameraDisabled` status.

### 5.2 Software-Only Control (Alternative to Hardware Switch)

- MCU controls `V_SENSOR` via MOSFET GPIO without physical switch.
- User configures scheduling, detection-triggered activation, or manual control.
- Software can be configured in `aegis-mesh.toml`.

### 5.3 Software Privacy Controls

**Scheduling:**
```toml
[identity_node.privacy]
# Camera active only during configured hours
schedule_enabled = false
schedule_on = "08:00"
schedule_off = "22:00"
```

**Detection-Triggered Mode:**
- Sensing layer triggers Identity Node only when anomaly criteria are met.
- Configuration in edge controller policy engine.

**Always-On Mode:**
- Identity Node captures continuously per storage configuration.

### 5.4 No Jurisdictional Restrictions

- The system does not impose any jurisdiction-specific data limits.
- Users are responsible for compliance with applicable recording, privacy, and data protection laws in their jurisdiction.
- All data handling is user choice.

### 5.5 Privacy Architecture Summary

| Control Type | Mechanism | Override Possible? |
|--------------|-----------|---------------------|
| Hardware kill switch (if installed) | Physical power cut | No — hardware enforcement |
| Software disable | MCU GPIO + MOSFET | Yes — software controlled |
| Scheduling | Firmware timer | Yes — configurable |
| Detection-triggered | Edge controller policy | Yes — policy dependent |
| Continuous capture | User configuration | Yes — always configurable |

---

## 6. Schematic Blocks

### 6.1 Power Supply

- **Input:** 5V DC (USB-C, screw terminal, or PoE splitter).
- **Main rail (`V_MAIN`):** 3.3V LDO for MCU and digital peripherals.
- **Sensor rail (`V_SENSOR`):** Secondary power rail for identification sensor.
  - Can be gated by optional user-installed hardware switch.
  - Can also be software-controlled via MCU GPIO → MOSFET for software-only privacy control.

```
V_MAIN (5V) → LDO → V_MCU (3.3V)
              │
              └── Optional Switch → V_SENSOR (3.3V/2.8V)
```

### 6.2 Camera Interface

- **Connector:** 24-pin FPC (compatible with standard camera modules).
- **Modular:** Any compatible camera module can be connected.
- **Route:** MIPI CSI-2 traces with 100 Ω differential impedance, length-matched within 2 mil.
- **Power:** Camera module powered from `V_SENSOR` (optionally gated).

### 6.3 Local Storage

- **Primary:** microSD card slot (SPI mode, up to 2 TB).
- **Secondary:** On-board QSPI NOR Flash (8–128 MB) for metadata, logs, model weights.
- **Purpose:** Raw video, image clips, acoustic recordings, integrity manifests.
- **Removable:** SD card allows user to physically retrieve recordings without network access.
- **Recommended for:** Legal evidence applications, high-volume recording deployments.

### 6.4 Mesh Communication

**The identity node uses mesh radio ONLY — no WiFi, no cellular.**

| Transport | Bandwidth | Use Case |
|-----------|-----------|----------|
| BLE 5.x | 500 Kbps - 2 Mbps | Metadata, alerts, configuration |
| Thread | Similar to BLE | Alternative mesh protocol |
| PoE Ethernet | 10-100 Mbps | High-bandwidth camera streaming (Variant C/D) |

**All data routes to edge controller first, then to companion app.**

---

## 7. Mesh Communication Details

### 7.1 Metadata-Only Transmission

```json
{
  "type": "IdentificationEvent",
  "node_id": "identity_node_01",
  "timestamp_ms": 1710509422004,
  "classification": "KnownResident",
  "confidence": 0.94,
  "zone_id": "front_entry"
}
```

### 7.2 Recording Available Event

When a recording is completed and written to local storage:

```json
{
  "type": "RecordingAvailable",
  "node_id": "identity_node_01",
  "recording_path": "identity_node_01/clip_20240315_143022.mp4",
  "start_timestamp_ms": 1710509422004,
  "duration_ms": 5200,
  "trigger": "on_detection",
  "size_bytes": 4523108,
  "format": "h264_mp4",
  "modalities": ["video"],
  "sha256": "a3f4b2c9d8e1f7a2b5c8d3e6f9a0b1c2d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8"
}
```

### 7.3 Media Stream Available Event

When live streaming is activated:

```json
{
  "type": "MediaStreamStarted",
  "node_id": "identity_node_01",
  "stream_endpoint": "edge_controller:9090/node/identity_node_01",
  "stream_format": "h264",
  "timestamp_ms": 1710509422004
}
```

**Note:** The stream endpoint is at the edge controller, not the identity node. The edge controller relays the stream to the companion app over WiFi.

### 7.4 Data Flow for Live Streaming

```
1. Companion app requests stream via edge controller API
2. Edge controller sends StartStream command to identity node via mesh
3. Identity node encodes video, transmits via mesh radio to edge controller
4. Edge controller relays stream to companion app via WiFi
5. When stream ends, identity node sends MediaStreamEnded event
```

---

## 8. Node Variants

### Variant A — Standard (Single Camera)

**Description:** Single identification sensor in standard enclosure.

**Hardware:**
- Single MCU (STM32H7 class or equivalent)
- Single camera module (OV2640, IMX307, or similar)
- microSD slot
- BLE/Thread mesh radio
- Optional hardware switch footprint

**Use case:** Standard entry point identification, cost-effective deployment.

### Variant B — Multi-Camera Array

**Description:** 2–4 cameras at configured angles for enhanced coverage.

**Hardware:**
- Higher-performance MCU (STM32H7 or i.MX RT class)
- Multiple camera modules (2–4 × OV2640 or similar)
- Hardware FSYNC for synchronized capture
- PoE Ethernet for high-bandwidth transport to edge controller
- microSD slot

**Use case:** Wide entry points, full-coverage verification zones.

### Variant C — 360° Camera Array

**Description:** 4–8 cameras covering full hemisphere for complete 360° coverage.

**Hardware:**
- High-performance MCU or Linux SoM
- 4–8 camera modules in circular arrangement
- Hardware FSYNC for synchronized capture
- PoE Ethernet (mandatory for bandwidth)
- On-node stitching capability (optional)
- microSD slot

**Use case:** Comprehensive coverage zones, maximum evidence capture.

### Variant D — Biometric (Non-Visual)

**Description:** Non-visual identification via fingerprint, voice, or other biometric.

**Hardware:**
- Standard MCU
- Fingerprint scanner (Synaptics FS4500 or similar) OR
- Voice recognition module (Sensory TrulyHandsfree or similar)
- No camera
- BLE/Thread mesh radio

**Use case:** High-security entry points, alternative to visual identification.

---

## 9. Candidate Component Set (Test Variants — Not Locked)

### 9.1 MCU (Identity Processing)

| Variant | MCU | Key Features |
|---------|-----|--------------|
| A | STM32H755 | Dual Cortex-M7/M4, hardware crypto, TrustZone, USB, camera interface |
| B | NXP i.MX RT1060 | Cortex-M7, 600 MHz, camera interface, neural inference performance |
| C | NXP i.MX 8M Mini | Quad-core Cortex-A53 + Cortex-M4, Linux-capable, NPU |
| D | STM32H743 | Cortex-M7, 480 MHz, camera interface, lower cost |

**Important:** The identity MCU must NOT share the same die as the sensing mesh MCU. Physical isolation required.

### 9.2 Identification Sensor — All Are Test Variants

**Visual (Camera-Based):**

| Module | Resolution | Interface | Features | Notes |
|--------|------------|-----------|----------|-------|
| Arducam IMX219 | 8 MP | CSI-2 | Configurable lens | High resolution |
| Sony IMX307 | 2 MP | CSI-2 | Low light | Night performance |
| OmniVision OV5640 | 5 MP | MIPI/USB | Autofocus | Flexible |
| OmniVision OV2640 | 2 MP | DVP/SPI | Compact | Minimal cost |

**Multi-Camera Array:**

| Configuration | Cameras | FoV per camera | Notes |
|---------------|---------|----------------|-------|
| 2-camera | 2 × OV2640 | 90° | Entry point coverage |
| 4-camera | 4 × OV2640 | 60° | Wide entry coverage |
| 8-camera | 8 × OV2640 | 45° | Full 360° coverage |

**Depth / Structured Light:**

| Module | Type | Interface | Notes |
|--------|------|-----------|-------|
| Intel RealSense D415 | Stereo depth + RGB | USB | Research variant |
| Custom dot projector | IR structured light | Custom | Research only |

**Self-Contained Neural Inference:**

| Module | Resolution | Features | Notes |
|--------|------------|----------|-------|
| HiMax HM01B0 | QVGA | On-chip detection | Ultra-low power |
| Luxonis OAK-1 | 4K | On-module NPU | Neural inference |

**Non-Visual Biometric:**

| Module | Type | Interface | Notes |
|--------|------|-----------|-------|
| Synaptics FS4500 | Fingerprint | USB/SPI | Fixed entry points |
| Sensory TrulyHandsfree | Voice | UART | Audio biometric |

### 9.3 Optional Hardware Power Switch (User-Installed)

| Switch | Type | Features | Notes |
|--------|------|----------|-------|
| Alps SSSS811101 | SMD slider | 0.5 mm travel | Compact |
| Omron D2JW-01K13H | Snap-action | 100k cycles | Tactile feel |
| None | — | — | Software-only control via MOSFET |

### 9.4 Local Storage

| Component | Capacity | Interface | Notes |
|-----------|----------|-----------|-------|
| microSD card | Up to 2 TB | SPI | Primary, removable |
| QSPI NOR Flash | 8–128 MB | QSPI | Secondary, fixed |

### 9.5 Mesh Radio

| Radio | Protocol | Bandwidth | Notes |
|-------|----------|-----------|-------|
| nRF52840 | BLE 5.x | 2 Mbps | Primary choice |
| nRF5340 | BLE 5.x | 2 Mbps | Integrated with MCU |
| External Thread | Thread | 250 Kbps | Alternative protocol |
| W5500 | Ethernet | 100 Mbps | High-bandwidth (Variant C) |

---

## 10. Interface Summary

### 10.1 Standard Interface Header (20-pin)

| Pin | Signal | Direction | Description |
|---|---|---|---|
| 1 | VCC_3V3 | Power | 3.3 V regulated supply |
| 2 | V_SENSOR | Power | Sensor power (optionally gated) |
| 3 | GND | — | Ground |
| 4 | MESH_TX | Output | Mesh radio UART TX |
| 5 | MESH_RX | Input | Mesh radio UART RX |
| 6 | MESH_CS | Output | Mesh radio SPI CS |
| 7 | MESH_IRQ | Input | Mesh radio interrupt |
| 8 | I2C_SDA | Bidirectional | Sensor I2C data |
| 9 | I2C_SCL | Output | Sensor I2C clock |
| 10 | CAM_CLK | Output | Camera clock |
| 11 | CAM_DATA | Input | Camera data (CSI/DVP) |
| 12 | CAM_FSYNC | Output | Camera frame sync |
| 13 | SD_CLK | Output | SD card SPI clock |
| 14 | SD_MOSI | Output | SD card SPI MOSI |
| 15 | SD_MISO | Input | SD card SPI MISO |
| 16 | SD_CS | Output | SD card SPI CS |
| 17 | CAM_HW_SW_GPIO | Input | Camera hardware switch state |
| 18 | LED_STATUS | Output | Status LED |
| 19 | BOOT_SELECT | Input | Firmware boot mode |
| 20 | RESET | Input (active low) | MCU reset |

### 10.2 Additional Signals for Multi-Camera Variants

| Signal | Description |
|--------|-------------|
| CAM_MIPI_CLK+/- | MIPI CSI-2 clock (per camera) |
| CAM_MIPI_D0+/- | MIPI CSI-2 data lane (per camera) |
| CAM_FSYNC | Hardware frame sync to all cameras |
| ETH_TX+/- | Ethernet TX (PoE variants) |
| ETH_RX+/- | Ethernet RX (PoE variants) |

---

## 11. Power Architecture

### 11.1 Power Rails

| Rail | Voltage | Source | Usage |
|------|---------|--------|-------|
| V_MAIN | 5.0V | Input | Raw supply |
| V_MCU | 3.3V | LDO | MCU, digital peripherals |
| V_SENSOR | 3.3V/2.8V | Gated | Camera/sensor (optionally switched) |
| V_RADIO | 3.3V | LDO | Mesh radio |

### 11.2 Power Budget (Estimates)

| State | Power (mW) | Notes |
|-------|------------|-------|
| Sleep | 10–50 | MCU low-power, radio idle |
| Idle | 100–200 | MCU active, sensors off |
| Active (metadata-only) | 300–500 | Camera + inference |
| Active (recording) | 500–1000 | Camera + encode + SD write |
| Active (streaming) | 800–1500 | Camera + encode + mesh transmission |
| Peak | 2000 | Startup, all functions active |

---

## 12. QC Tests

### 12.1 All Units

| Test | Acceptance Criteria |
|------|---------------------|
| Power-on self-test | MCU boots, firmware CRC verified |
| Rail verification | All rails within ±5% of nominal |
| Sensor initialization | Camera responds to init sequence |
| Mesh radio ping | Node appears on network, responds within 200 ms |
| Firmware version | Matches current build hash |

### 12.2 Data Handling Mode Verification

| Mode | Verification |
|------|--------------|
| Metadata-only | Only classification tags in mesh output; no file on SD |
| Local storage | Recording file on SD within 500 ms of trigger |
| App streaming | Edge controller receives stream, relay to app within 2 seconds |
| Continuous recording | Recording file present with correct duration |

### 12.3 Optional Hardware Switch (Only If Installed)

| Test | Acceptance Criteria |
|------|---------------------|
| Switch DISABLED | V_SENSOR = 0 V ± 0.05 V; FAIL if > 0.1 V |
| Switch ENABLED | V_SENSOR > 3.0 V |
| Firmware state | Correct switch state reported via mesh |

### 12.4 Processing Tests

| Test | Target |
|------|--------|
| Identification accuracy | Known resident at 1m, 2m, 3m: > 95% true positive |
| Processing latency | < 2 seconds from activation to classification |

---

## 13. Integration with Edge Controller

### 13.1 Mesh Protocol Messages

The identity node sends and receives the following message types:

**Transmitted by Identity Node:**
- `IdentificationEvent` — Classification result
- `RecordingAvailable` — Local recording notification
- `MediaStreamStarted` — Live stream activation
- `MediaStreamEnded` — Stream termination
- `NodeHealth` — Battery, storage status

**Received by Identity Node:**
- `EnableIdentification` — Activate identification
- `DisableIdentification` — Deactivate
- `StartStreamToEdge` — Begin live stream
- `StopStreamToEdge` — End live stream
- `ConfigurationUpdate` — Parameter changes

### 13.2 Edge Controller Role

The edge controller:
- Receives all identity node data via mesh radio
- Stores recordings on its own SD card if configured
- Relays live streams to companion app via WiFi
- Generates integrity manifests for legal export
- Aggregates identification events across all identity nodes

---

## 14. Compliance Notes

### 14.1 Radio

- Mesh radio module must be FCC/CE/ISED certified.
- Antenna design must meet regulatory requirements.

### 14.2 Optical Safety

- If using IR illumination, ensure Class 1 eye-safe wavelengths.
- Visible light illumination must not exceed safe exposure limits.

### 14.3 Recording Laws

- Users are responsible for compliance with applicable recording consent laws in their jurisdiction.
- System imposes no jurisdictional restrictions.
- One-party vs. two-party consent is the deployer's responsibility.

### 14.4 Legal Evidence

- Integrity chain suitable for legal proceedings.
- Recordings include SHA-256 hash, firmware version, device ID, timestamp.
- Chain hash links sequential recordings for tamper evidence.

---

## 15. Directory Structure

```
identity_node/
├── variant_a_standard/
│   ├── identity_node_std.kicad_pro
│   ├── identity_node_std.kicad_sch
│   ├── identity_node_std.kicad_pcb
│   └── gerbers/
├── variant_b_multi_camera/
│   ├── identity_node_multi_cam.kicad_pro
│   ├── identity_node_multi_cam.kicad_sch
│   ├── identity_node_multi_cam.kicad_pcb
│   └── gerbers/
├── variant_c_360_camera/
│   ├── identity_node_360.kicad_pro
│   ├── identity_node_360.kicad_sch
│   ├── identity_node_360.kicad_pcb
│   ├── camera_calibration.json
│   └── gerbers/
├── variant_d_biometric/
│   ├── identity_node_biometric.kicad_pro
│   └── gerbers/
├── bom/
│   ├── identity_node_std_bom.csv
│   ├── identity_node_multi_cam_bom.csv
│   ├── identity_node_360_bom.csv
│   └── identity_node_biometric_bom.csv
└── README.md
```

---

## 16. Configuration Reference

```toml
# aegis-mesh.toml — Identity Node Configuration

[identity_node]
enabled = true

[identity_node.data]
# Data handling mode
mode = "metadata_only"               # "metadata_only" | "local_storage" | "app_streaming" | "full_recording"

# Storage settings
store_raw = false
storage_target = "sd_card"
retention_days = 90
continuous_recording = false
recording_trigger = "on_detection"
max_storage_mb = 0

# Streaming settings
enable_streaming = false
stream_quality = "high"
stream_codec = "h264"

# Integrity chain
include_integrity_chain = true

[identity_node.privacy]
# Hardware switch (only relevant if installed)
hardware_switch_installed = false

# Software controls
software_control_enabled = true
schedule_enabled = false
schedule_on = "08:00"
schedule_off = "22:00"

# Trigger-based activation
detection_triggered = true
trigger_zone = "front_entry"
trigger_events = ["UnknownPersonDetected", "MotionAfterHours"]
```

---

**End of Document**
