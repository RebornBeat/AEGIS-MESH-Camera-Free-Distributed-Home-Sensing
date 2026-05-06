# Identity Node — Hardware Schematic Reference Design

**Project:** AEGIS-MESH
**Node Class:** IdentityNode — Configurable Visual / Biometric Identification
**Status:** Reference Design — All Variants. No component is locked.
**License:** CERN-OHL-S v2

---

## 1. Purpose

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

## 2. Core Architecture

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
│ — see Section 4)    │
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
│ Mesh Radio          │   Transmits per configured data handling mode:
│                     │   - Metadata only (classification tags)
│                     │   - Recording availability events
│                     │   - Live stream relay
└─────────────────────┘
```

---

## 3. Data Handling — Fully User Configurable

All data handling is configured in `aegis-mesh.toml`. The architecture supports any combination.

### Mode A — Metadata-Only (Default)

- On-device inference produces classification tags.
- Only `IdentificationEvent` metadata transmitted over mesh.
- Example: `{"class": "KnownResident", "confidence": 0.94}`
- No raw frames stored or transmitted.
- Minimum bandwidth and storage footprint.

### Mode B — Local Storage

- Raw video, images, or clips written to on-node SD card or internal flash.
- Available for review via AEGIS-MESH companion app or direct SD card access.
- Suitable for security investigations, legal evidence, system calibration, incident review.
- Storage capacity and retention period user-configured.
- `RecordingAvailable` mesh event sent to edge controller on completion.

### Mode C — App Streaming

- Raw or compressed video/audio streamed to the AEGIS-MESH companion app over Wi-Fi.
- Users view live footage and review recordings from mobile or desktop.
- Simultaneous local storage available if configured.

### Mode D — Continuous Recording

- All sensor data captured continuously or on-trigger.
- Recordings carry embedded integrity metadata (timestamp, node ID, cryptographic hash chain) for legal use.
- Users export for legal proceedings, insurance claims, or personal records.

### Configuration (`aegis-mesh.toml`)

```toml
[identity_node.data]
mode = "metadata_only"               # "metadata_only" | "local_storage" | "app_streaming" | "full_recording"
store_raw = false                    # Enable raw frame/video storage
storage_target = "sd_card"          # "sd_card" | "internal_flash"
retention_days = 90                  # 0 = keep forever
continuous_recording = false
recording_trigger = "on_detection"
enable_streaming = false
stream_quality = "high"
include_integrity_chain = true
```

---

## 4. Privacy Architecture

The privacy model is user-configured, not system-imposed.

**User-controlled options:**

**Hardware Power Switch (optional — user-installed):**
- A physical slider or toggle that cuts the `V_SENSOR` power rail.
- When DISABLED: sensor is physically unpowered; no capture occurs.
- When ENABLED: all software data handling modes are available.
- **This is an option the user can add — it is not architecturally required.**
- Some users want guaranteed physical camera disable without software interaction; this enables that.
- Other users prefer software-only control; no switch is needed.

**Software Scheduling:**
- Camera inactive during configured hours in `aegis-mesh.toml`.

**Detection-Triggered Mode:**
- Sensing layer triggers Identity Node only when anomaly criteria are met.

**Always-On Mode:**
- Identity Node captures continuously per storage configuration.

**No Restrictions:**
- The system does not impose any jurisdiction-specific data limits.
- Users are responsible for compliance with applicable recording, privacy, and data protection laws in their jurisdiction.

---

## 5. Schematic Blocks

### 5.1 Power Supply

- **Input:** 5V DC (USB-C, screw terminal, or PoE splitter).
- **Main rail:** 3.3V LDO for MCU and peripherals.
- **Sensor rail (`V_SENSOR`):** Secondary power rail for identification sensor.
  - Can be gated by an optional user-installed hardware switch.
  - Can also be software-controlled via MCU GPIO → MOSFET for software-only privacy control.

### 5.2 Camera Interface

- **Connector:** 24-pin FPC (compatible with standard camera modules).
- **Modular:** Any compatible camera module can be connected.
- **Route:** MIPI CSI-2 traces with 100 Ω differential impedance.

### 5.3 Local Storage

- **Primary:** microSD card slot (SPI mode, up to 2 TB).
- **Secondary:** On-board QSPI NOR Flash (8–128 MB) for metadata, logs, model weights.
- **Purpose:** Raw video, image clips, acoustic recordings, integrity manifests.
- Removable SD card allows user to physically retrieve recordings without network access.
- Strongly recommended for legal evidence applications.

### 5.4 Mesh Communication

**Metadata-only (default):**
```json
{"id": "node_04", "type": "identity", "classification": "Known Resident", "confidence": 0.96}
```

**With recording available:**
```json
{
  "id": "node_04",
  "type": "identity",
  "classification": "Known Resident",
  "confidence": 0.96,
  "raw_media_path": "node_04/clip_20240315_143022.mp4",
  "clip_duration_s": 5.2,
  "live_stream_available": false
}
```

**App streaming mode:** Live video/image stream transmitted to companion app over Wi-Fi. Endpoint announced via `MediaStreamStarted` mesh event.

---

## 6. Candidate Component Set (Test Variants — Not Locked)

### MCU (Identity Processing)

- **Variant A:** STM32H755 (dual Cortex-M7/M4, hardware crypto, TrustZone, USB, camera interface)
- **Variant B:** NXP i.MX RT1060 (Cortex-M7, 600 MHz, camera interface, neural inference performance)
- **Variant C:** NXP i.MX 8M Mini (quad-core Cortex-A53 + Cortex-M4, Linux-capable, NPU)

The identity MCU must NOT share the same die as the sensing mesh MCU. Physical isolation required.

### Identification Sensor — All Are Test Variants

**Visual (Camera-Based):**
- **Variant A:** Arducam IMX219 (8 MP, CSI, configurable lens)
- **Variant B:** Sony IMX307 (2 MP, low light, MIPI CSI-2)
- **Variant C:** OmniVision OV5640 (5 MP, autofocus, USB/MIPI)
- **Variant D:** OmniVision OV2640 (2 MP, compact, DVP/SPI) — minimal cost option

**Multi-Camera Array (Advanced):**
- **Variant E:** 2–4 OV2640 modules at configured angles (coverage per entry geometry)
- **Variant F:** 360° camera array using 4–8 OV2640 or IMX219 modules for full entry point coverage

**Depth / Structured Light:**
- **Variant G:** Intel RealSense D415 (stereo depth + RGB, USB)
- **Variant H:** Dot-projector structured light + IR camera (research only)

**Self-Contained Neural Inference:**
- **Variant I:** HiMax HM01B0 (QVGA, ultra-low power, first-stage detection on-chip)
- **Variant J:** Luxonis OAK-1 (neural inference on-module)

**Non-Visual Biometric:**
- **Variant K:** Fingerprint scanner (Synaptics FS4500) — fixed entry points only
- **Variant L:** Voice recognition pre-processor (Sensory TrulyHandsfree) — audio identification

### Optional Hardware Power Switch (User-Installed)

- **Variant A:** Alps SSSS811101 (SMD slider, 0.5 mm travel)
- **Variant B:** Omron D2JW-01K13H (snap-action, 100k cycles)
- **Variant C:** None — software-only control via MCU GPIO → MOSFET

The switch is user-installed. The PCB provides the option via:
- Designated footprint on `V_SENSOR` rail
- Optional MOSFET for software-only equivalent without physical switch

### Local Storage

- **Primary:** microSD card slot (SPI) — up to 2 TB, removable
- **Secondary:** On-board QSPI NOR Flash (metadata, logs, model weights)

### Mesh Radio

- Same mesh radio as Always-On Node (consistent protocol across all node types).

---

## 7. QC Tests

### All Units

1. **Power-on self-test** — MCU boots, firmware CRC verified.
2. **Rail verification** — All power rails within spec.
3. **Identification sensor initialization** — Sensor responds to init sequence.
4. **Mesh radio ping** — Node appears on network, responds within 200 ms.

### Data Handling Mode Verification

5. Verify configured data handling mode operates correctly:
   - **Metadata-only:** Only classification tags transmitted over mesh. No file created on SD.
   - **Local storage:** Recording file written to SD card within 500 ms of trigger.
   - **App streaming:** Companion app receives live stream within 2 seconds of activation.
   - **Full recording:** Recording file present with correct duration after trigger event.

### Optional Hardware Switch Verification (Only If Switch Is Installed)

6. Switch DISABLED: Confirm zero current on `V_SENSOR` rail (0 mA ± 2 mA). If current > 2 mA: FAIL — unit does not isolate sensor power.
7. Switch ENABLED: Confirm `V_SENSOR` > 3.0 V (sensor powered).
8. Firmware switch state report: verify correct status reported via mesh radio.

### Processing Tests

9. **Identification accuracy** — Known resident at 1 m, 2 m, 3 m; unknown at same distances. Measure true-positive and false-positive rates.
10. **Processing latency** — Time from activation to first classification result. Target: < 2 seconds.

---

## 8. Compliance Notes

- **Radio:** Wi-Fi / BLE module must be FCC/CE certified.
- **Optical Safety:** If using IR illumination, ensure Class 1 eye-safe wavelengths.
- **Recording Laws:** Users are responsible for compliance with applicable recording consent laws in their jurisdiction. System imposes no jurisdictional restrictions.
- **Legal Evidence:** Integrity chain suitable for legal proceedings. Recordings include SHA-256 hash, firmware version, device ID, and timestamp.

---

## 9. Directory Structure

```
identity_node/
├── variant_a_standard/
│   ├── identity_node_std.kicad_pro
│   ├── identity_node_std.kicad_sch
│   ├── identity_node_std.kicad_pcb
│   └── gerbers/
├── variant_b_multi_camera/
│   ├── identity_node_multi_cam.kicad_pro
│   └── gerbers/
├── bom/
│   ├── identity_node_std_bom.csv
│   └── identity_node_multi_cam_bom.csv
└── README.md
```
