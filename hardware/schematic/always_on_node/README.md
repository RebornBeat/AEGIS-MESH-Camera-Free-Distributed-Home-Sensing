# Always-On Node — Hardware Schematic Reference Design

**Project:** AEGIS-MESH
**Node Class:** AlwaysOn — Choke Point / Volume Sensing Backbone
**Status:** Reference Design — All Variants
**License:** CERN-OHL-S v2

---

## 1. Connectivity Architecture — Mesh Radio Only

**The Always-On Node has NO WiFi and NO cellular connectivity.**

All Always-On Node data transmits via mesh radio (BLE/Thread/PoE Ethernet) to the edge controller exclusively. The edge controller is the sole external network gateway.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│              Always-On Node Connectivity Architecture                       │
│                                                                              │
│  Always-On Node                                                              │
│       │                                                                      │
│       │  Mesh Radio (BLE 5.x / Thread / PoE Ethernet)                       │
│       ▼                                                                      │
│  Edge Controller                                                             │
│       │                                                                      │
│       ├──► WiFi (802.11) ──► Companion App (local network)                  │
│       ├──► Cellular (LTE/5G) ──► Companion App (remote)                     │
│       └──► BLE 5.3 ──► Companion App (direct local fallback)                │
│                                                                              │
│  ⚠️  ALWAYS-ON NODE HAS NO WIFI / CELLULAR / DIRECT APP CONNECTION           │
│  ⚠️  ALL DATA ROUTES VIA MESH RADIO → EDGE CONTROLLER                        │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Why This Architecture?

| Constraint | WiFi on Sensor Node | BLE/Mesh on Sensor Node |
|------------|---------------------|-------------------------|
| **Power** | 300-1000+ mW active | 5-15 mW active |
| **Heat** | Significant thermal load | Minimal |
| **Form factor impact** | Requires larger antenna space | Minimal |
| **Installation** | Requires WiFi credentials per node | Auto-discovers edge controller |
| **Determinism** | Variable latency, contention | Consistent, schedulable |

**Conclusion:** WiFi on distributed sensor nodes creates power, thermal, and deployment complexity that outweighs any benefit. All external connectivity is concentrated at the edge controller where these constraints are manageable.

---

## 2. Purpose

The Always-On Node is the continuously-powered sensing backbone of AEGIS-MESH. It runs at all times on standby power and provides:
- Presence detection via mmWave radar (primary sensing modality)
- Binary crossing detection via PIR (choke-point nodes)
- Acoustic event detection via microphone array (material classification, direction-of-arrival)
- Environmental sensing (temperature, humidity, air quality — atmospheric compensation)
- Optional camera module for nodes where visual capture is desired
- Mesh radio communication to edge controller

**Data flow for all variants:**
```
Sensors on Node → MCU Processing → Mesh Radio → Edge Controller → WiFi/Cellular → Companion App
```

The camera-free variants (A, B, C) of this node class enforce camera-free sensing at the compile-time firmware dependency level — the `omni-sense-drivers-camera` crate is not linked for these variants. Variant D adds an optional camera module.

---

## 3. Design Philosophy

### 3.1 No Artificial Limits

The architecture supports any sensor configuration, any storage capacity, any data transmission mode. Users configure what is collected, what is stored locally, and what is transmitted to the edge controller and companion app.

**Storage hierarchy:**
- On-node SD card (up to 2 TB) — raw sensor data, recordings, SLAM contributions
- On-node QSPI Flash — metadata, logs, model weights
- Edge controller storage — aggregated from all nodes

### 3.2 Modularity

mmWave module, PIR, microphone array, environmental sensor, and optional camera are on independent sub-circuits. Each can be populated or omitted. Single PCB design supports multiple variants through configuration.

### 3.3 Acoustic Processing

Beamforming, material classification, and event detection run on-MCU. Mesh transmits classification metadata by default (low bandwidth). Raw audio storage and streaming are user-configured:

| Data Mode | Bandwidth | Description |
|-----------|-----------|-------------|
| Metadata only | ~1-10 Kbps | Classification tags, DOA, event type |
| Local SD recording | N/A (local) | Raw audio written to SD, indexed by edge controller |
| App streaming | ~64-256 Kbps | Compressed audio stream via edge controller to companion app |

### 3.4 Privacy and Data Handling

**Camera-free variants (A, B, C):**
- Produce no imagery by architecture
- Firmware binary has no `omni-sense-drivers-camera` dependency
- Cannot be configured to produce imagery

**Variant D with camera:**
- Camera follows user-configured data handling policy
- Metadata-only, local storage, app streaming, or continuous recording
- Optional hardware power switch (user-installed) for guaranteed physical disable
- No system-imposed data restrictions

**Jurisdictional compliance:** Recording consent laws, surveillance regulations, and data protection frameworks are the deployer's responsibility. The system provides the mechanisms; the user configures compliance.

---

## 4. Node Variants

### 4.1 Variant A — Choke Point (Minimal)

**Purpose:** Binary detection at doorways, hallways, stairways. Lowest cost and power.

**Sensors:**
- PIR (binary motion trigger)
- 1D ToF (range confirmation, prevents false triggers from thermal drafts)
- Environmental sensor (temperature, humidity)

**No radar, no camera, no microphone array.** This is the lowest-cost node class.

**Use case:** Deploy at every interior doorway; provides crossing events and direction inference at minimal hardware cost.

**Data output:**
- Binary crossing event → Mesh → Edge controller
- Crossing direction (inferred from sequential trigger at adjacent nodes)
- Environmental readings for atmospheric compensation

### 4.2 Variant B — Volume Presence (Primary)

**Purpose:** Full presence detection and acoustic classification in open volumes.

**Sensors:**
- mmWave FMCW radar (presence, velocity, micro-Doppler)
- Microphone array (4-element tetrahedral — DOA, material classification, event detection)
- Environmental sensor (temperature, humidity, pressure, VOC)

**No camera.** Acoustic processing on-MCU. Raw audio recording user-configurable.

**Use case:** Living rooms, bedrooms, kitchens, open-plan spaces.

**Data output:**
- Detection events (position, velocity, classification) → Mesh → Edge controller
- Acoustic events (classification, DOA) → Mesh → Edge controller
- Environmental readings → Mesh → Edge controller
- Optional: Raw audio clips → SD card, indexed by edge controller

### 4.3 Variant C — Volume Presence + Enhanced Acoustic

**Purpose:** As Variant B with higher-density microphone array for improved DOA precision and material classification.

**Sensors:**
- mmWave FMCW radar
- Microphone array (6-element for enhanced 3D DOA)
- Environmental sensor

**No camera.**

**Use case:** Large open spaces, high-value rooms requiring higher acoustic precision (living rooms with complex geometry, home theaters, open-plan kitchens).

**Data output:** Same as Variant B, with improved DOA accuracy (target: < ±10° vs < ±15°).

### 4.4 Variant D — Volume Presence + Camera

**Purpose:** As Variant B with an optional camera module for users who want visual capture in addition to non-imaging sensing.

**Sensors:**
- mmWave FMCW radar
- Microphone array (4-element)
- Environmental sensor
- Configurable camera module

**Camera data handling:** Fully user-configured:
- **Metadata-only:** On-device inference, classification tags only
- **Local storage:** Raw video/images to SD card
- **App streaming:** Video stream via edge controller to companion app
- **Continuous recording:** Full-time capture with configurable retention

**Data output:**
- All sensor data from Variant B
- Plus camera data per configured mode:
  - Metadata: Classification tags → Mesh → Edge controller
  - Local storage: Recording files on SD, indexed by edge controller API
  - Streaming: Video stream → Mesh → Edge controller → WiFi → Companion app

**Use case:** Users wanting combined geometry-based sensing AND visual documentation of their space.

---

## 5. Hardware Architecture

### 5.1 Core Node Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                      Always-On Node PCB                          │
├──────────────────────────────────────────────────────────────────┤
│  MCU Core                                                        │
│  - ARM Cortex-M4F / M33 (STM32U5, nRF5340, ESP32-S3)            │
│  - FPU required for sensor fusion                                │
│  - DSP extensions preferred for radar and audio processing       │
├──────────────────────────────────────────────────────────────────┤
│  Sensing Interfaces (Populated per Variant)                      │
│  - mmWave Radar (SPI/UART) — Variants B, C, D                   │
│  - PIR (GPIO interrupt) — Variant A                             │
│  - 1D ToF (I2C/SPI) — Variant A                                 │
│  - Acoustic / Microphone Array (I2S/PDM) — Variants B, C, D     │
│  - Environmental (I2C) — All variants                           │
│  - Camera (MIPI CSI-2 / DVP) — Variant D only                   │
├──────────────────────────────────────────────────────────────────┤
│  Storage Interface (Optional Per Configuration)                 │
│  - microSD card (SPI, up to 2 TB)                               │
│  - QSPI NOR Flash (8-128 MB)                                    │
├──────────────────────────────────────────────────────────────────┤
│  Mesh Radio — MANDATORY                                          │
│  - BLE 5.x (integrated or module)                               │
│  - Thread (optional, same hardware)                             │
│  - NO WiFi, NO Cellular                                          │
├──────────────────────────────────────────────────────────────────┤
│  Power Management                                                │
│  - LDO or buck regulator from main supply                       │
│  - Battery backup (optional)                                    │
│  - Power rail for sensors (optionally gated for camera)         │
├──────────────────────────────────────────────────────────────────┤
│  Debug Interface                                                 │
│  - SWD (Serial Wire Debug)                                      │
│  - UART Console                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### 5.2 Block Diagram

```
                 ┌───────────────────────────────────────┐
                 │          Always-On Node PCB           │
                 │                                       │
Power Input ────►│ Power Management ──────► 3V3 / 1V8   │
                 │     │                                 │
                 │     ▼                                 │
                 │  ┌──────────────────────────────────┐ │
                 │  │ Main MCU (ARM Cortex-M33)        │ │
                 │  │                                  │ │
                 │  │ [SWD, UART] ─── Debug/Prog       │ │
                 │  │ [GPIO] ──────── PIR (Var A)       │ │
                 │  │ [SPI/UART] ──── mmWave (Var B/C/D)│ │
                 │  │ [I2S/PDM] ◄──── Mic Array(B/C/D) │ │
                 │  │ [I2C] ──────── Env Sensor         │ │
                 │  │ [SPI/UART] ──── Mesh Radio ──────┼►► To Edge Controller
                 │  │ [SPI] ──────── SD Card (optional) │ │
                 │  │ [MIPI/DVP] ──── Camera (Var D)    │ │
                 │  └──────────────────────────────────┘ │
                 └───────────────────────────────────────┘
                          Mesh Radio
                          (to Edge Controller only)
```

---

## 6. Candidate Component Set (Test Variants — Not Locked)

### 6.1 MCU Options

| MCU | Core | Clock | Flash | RAM | Radio | Notes |
|-----|------|-------|-------|-----|-------|-------|
| STM32U5 | Cortex-M33 | 160 MHz | 2 MB | 786 KB | None | Ultra-low power, TrustZone |
| nRF5340 | Cortex-M33 × 2 | 128 MHz | 1 MB | 512 KB | BLE 5.3 | Native mesh capability |
| ESP32-S3 | Xtensa LX7 × 2 | 240 MHz | 8 MB (ext) | 512 KB | WiFi/BLE | Note: WiFi not used for external connectivity |

**Note on ESP32-S3:** If selected, its WiFi interface is used ONLY as mesh transport to edge controller, NOT for external network connectivity. The edge controller remains the sole external gateway.

### 6.2 mmWave Radar Module (Variants B, C, D)

| Module | Frequency | Interface | Range | Features | Notes |
|--------|-----------|-----------|-------|----------|-------|
| TI IWR6843AOP | 60 GHz | SPI/UART | 15 m | Integrated antenna, DSP | Reference choice |
| Infineon BGT60ATR24C | 60 GHz | SPI | 10 m | Compact, MMIC | Low cost option |
| Acconeer XM122 | 60 GHz | SPI | 8 m | Pulsed coherent, ultra-low power | Battery-optimized |
| Acconeer XM125 | 60 GHz | SPI | 15 m | Long-range variant | Larger spaces |

### 6.3 PIR Sensor (Variant A)

| Sensor | Range | Power | Package |
|--------|-------|-------|---------|
| Panasonic EKMB1303112K | 8 m | 6 µA | SMD |
| Murata IRA-S210ST01 | 5 m | 15 µA | SMD, wide angle |

### 6.4 1D ToF Sensor (Variant A)

| Sensor | Range | Interface | Accuracy |
|--------|-------|-----------|----------|
| ST VL53L0X | 2 m | I2C | ±3% |
| ST VL53L1X | 4 m | I2C | ±3% |

### 6.5 Microphone Array (Variants B, C, D)

**4-element tetrahedral (Variant B, D):**
| Microphone | Interface | SNR | Notes |
|------------|-----------|-----|-------|
| Knowles SPH0645LM4H | I2S PDM | 65 dBA | Integrated PDM output |
| ST MP23DB01HP | Analog | 64 dBA | Requires external ADC |

**6-element circular (Variant C):**
- Knowles SPH0645LM4H × 6 (I2S PDM, circular arrangement)
- Enhanced 3D DOA precision

### 6.6 Environmental Sensor (All Variants)

| Sensor | Measurements | Interface |
|--------|--------------|-----------|
| Bosch BME688 | T, H, P, VOC | I2C |
| Sensirion SEN55 | T, H, VOC, NOx, PM2.5, PM10 | I2C |

### 6.7 Camera Module (Variant D — User Configured)

| Module | Resolution | Interface | Low-Light | Notes |
|--------|------------|-----------|-----------|-------|
| OV2640 | 2 MP | DVP/SPI | Moderate | Cost-effective |
| OV5640 | 5 MP | MIPI/DVP | Good | Autofocus available |
| IMX219 | 8 MP | MIPI CSI-2 | Good | Configurable optics |
| IMX307 | 2 MP | MIPI CSI-2 | Excellent | Night-vision capable |
| HM01B0 | QVGA | SPI | Ultra-low power | On-chip detection |

### 6.8 Local Storage (All Variants — Optional)

| Type | Capacity | Interface | Use Case |
|------|----------|-----------|----------|
| microSD card | Up to 2 TB | SPI | Raw audio, radar IQ, video (Variant D) |
| QSPI NOR Flash | 8-128 MB | QSPI | Metadata, logs, compressed clips |

### 6.9 Mesh Radio

| Option | Protocol | Range | Power | Notes |
|--------|----------|-------|-------|-------|
| nRF52840 module | BLE 5.x / Thread | 50 m | Low | Primary recommendation |
| ESP32-based (Murata 2AE) | BLE/WiFi | 30 m | Moderate | WiFi not used for external connectivity |
| W5500 + Ethernet | PoE | Wired | Low | Ceiling nodes with cable run |

### 6.10 Power Supply

| Source | Voltage | Notes |
|--------|---------|-------|
| PoE 802.3af | 48V input, 5V @ node | Ceiling-mount standard |
| USB-C PD | 5-20 V input | Development, portable |
| Battery backup | 3.7V LiPo | Optional, brief power interruption continuity |

---

## 7. Interface Mapping

### 7.1 Standard Interfaces

| Interface | Signals | Direction | Description |
|-----------|---------|-----------|-------------|
| **SPI Bus 1** | MOSI, MISO, SCK, CS0 | Bidirectional | Radar module (primary) |
| **SPI Bus 2** | MOSI, MISO, SCK, CS1 | Bidirectional | SD card (if populated) |
| **I2C Bus 1** | SDA, SCL | Bidirectional | Environmental, ToF |
| **I2S/PDM** | BCLK, WS, DATA | Output/Input | Microphone array |
| **UART** | TX, RX | Bidirectional | Debug console |
| **GPIO** | Various | Bidirectional | PIR, interrupts, status LEDs |
| **SWD** | CLK, DIO | Debug | Firmware programming |

### 7.2 Variant D Additional Interfaces (Camera)

| Interface | Signals | Description |
|-----------|---------|-------------|
| MIPI CSI-2 | CLK+, CLK-, D0+, D0- | Camera data (100 Ω differential) |
| DVP | D0-D7, PCLK, VSYNC, HREF | Parallel camera (lower-res modules) |
| CAM_PWR_EN | GPIO | Camera power enable (MOSFET gate) |
| CAM_HW_SW (optional) | GPIO | Hardware switch state input |

---

## 8. Power Architecture

### 8.1 Power Domains

| Domain | Voltage | Usage |
|--------|---------|-------|
| V_MAIN | 5.0V | Input from PoE splitter or USB-C |
| V_MCU | 3.3V | MCU and digital peripherals |
| V_RADIO | 3.3V | Mesh radio |
| V_SENSOR | 3.3V | Radar, microphones, environmental |
| V_CAM (Var D) | 3.3V / 2.8V | Camera module |

### 8.2 Power Budgets (Estimates — TBD by Test)

| Variant | Sleep (mW) | Active (mW) | Peak (mW) | Typical Runtime (PoE) |
|---------|------------|-------------|------------|----------------------|
| A — Choke Point | 10-50 | 100-300 | 500 | Unlimited (PoE powered) |
| B — Volume Presence | 50-150 | 500-1000 | 1500 | Unlimited (PoE powered) |
| C — Volume + Enhanced Acoustic | 60-180 | 600-1100 | 1600 | Unlimited (PoE powered) |
| D — Volume + Camera | 60-200 | 600-1500 | 2500 | Unlimited (PoE powered) |

**Note:** All variants are designed for continuous PoE operation. Battery backup provides brief continuity during power interruption.

---

## 9. Test Pad Standard (12-Pad Interface)

Standard test interface for production validation:

| Pad | Signal | Description |
|-----|--------|-------------|
| 1 | VCC_3V3 | 3.3V regulated supply |
| 2 | GND | Ground |
| 3 | CHRG_IN+ | Charging input positive (if battery backup present) |
| 4 | CHRG_IN- | Charging input negative |
| 5 | MESH_RF_TEST | Mesh radio test point (antenna tap) |
| 6 | MCU_BOOT | Boot mode select (HIGH = bootloader) |
| 7 | MCU_RESET | MCU reset (active low) |
| 8 | CAM_HW_SW_TEST | Camera hardware switch state (Variant D only) |
| 9 | MIC_TEST | Microphone test point (if populated) |
| 10 | SWD_CLK | SWD clock |
| 11 | SWD_DATA | SWD data |
| 12 | UART_TX | Debug UART TX |

---

## 10. Testing Strategy

### 10.1 All Variants — Mandatory Tests

1. **Power-on self-test:** MCU boots, firmware CRC verified
2. **Rail verification:** All power rails within ±5% of nominal
3. **Sensor enumeration:** All I2C/SPI sensors respond
4. **Mesh connectivity:** Node appears on mesh network, responds to edge controller ping within 200 ms
5. **Firmware version:** Matches current build hash

### 10.2 Variant A — Choke Point Tests

1. PIR trigger test at 1 m, 3 m, 5 m, 8 m
2. ToF ranging accuracy: within ±5% at 0.5-4 m
3. False trigger immunity: thermal draft (hair dryer at 2 m, 50°C)
4. Crossing direction inference: sequential trigger at adjacent nodes

### 10.3 Variants B, C — Volume Presence Tests

1. Radar detection range: 1 m, 3 m, 5 m, 10 m, 15 m LOS and through drywall
2. Acoustic DOA: Sound source at 0°, 45°, 90°, 135°, 180° at 2 m. Target: < ±15° error (B) or < ±10° error (C)
3. Material classification: Glass break, ceramic drop, wood knock, footstep at 3 m
4. Environmental immunity: Steam, smoke, dust — radar and acoustic continue operating
5. Mesh latency to edge controller: < 100 ms

### 10.4 Variant D — Camera Tests

1. All Variant B tests
2. Camera initialization: Sensor responds within 500 ms
3. Data handling mode verification:
   - Metadata-only: Classification tag in mesh output, no SD file
   - Local storage: Recording file on SD within 500 ms of trigger
   - App streaming: Edge controller receives stream within 2 seconds
   - Continuous recording: Recording present with correct duration
4. Optional hardware switch test (if installed): V_CAM = 0V ± 0.05V when DISABLED

---

## 11. Data Handling Configuration

### 11.1 Storage Options

All variants support optional local storage:
- microSD card (up to 2 TB) — removable, user-accessible
- QSPI NOR Flash (8-128 MB) — non-removable

### 11.2 Configuration (`aegis-mesh.toml`)

```toml
[data_handling]
# Storage configuration
store_raw_sensor_data = false      # Enable raw radar/LiDAR data storage
store_raw_audio = false            # Enable raw audio storage
storage_target = "sd_card"         # "sd_card" | "internal_flash" | "edge_controller"
retention_days = 30                # 0 = keep forever
max_storage_mb = 0                 # 0 = unlimited

# Acoustic configuration
[acoustic]
transmit_metadata = true           # Classification results to edge controller
record_on_event = false            # Record audio clip on acoustic event
stream_to_app = false              # Live audio stream to companion app

# Camera configuration (Variant D only)
[identity_node.data]
mode = "metadata_only"             # "metadata_only" | "local_storage" | "app_streaming" | "full_recording"
store_raw = false
recording_trigger = "on_detection"
```

### 11.3 Data Flow by Configuration

**Metadata-only (default):**
```
Sensors → MCU Processing → Classification → Mesh → Edge Controller → WiFi → App
Bandwidth: ~1-10 Kbps per node
```

**Local storage:**
```
Sensors → MCU Processing → Classification → Mesh → Edge Controller (index)
                 └──→ SD Card (raw data)
Edge Controller API serves SD index; user retrieves clips on demand
```

**App streaming:**
```
Sensors → MCU Processing → Compression → Mesh → Edge Controller → WiFi → App
Bandwidth: ~64-256 Kbps (audio), higher for camera
```

---

## 12. Mesh Radio Interface

### 12.1 Physical Layer

| Parameter | Value |
|-----------|-------|
| Protocol | BLE 5.x / Thread |
| Interface | SPI or integrated MCU radio |
| Range | 30-50 m indoor (typical) |
| Data rate | 1-2 Mbps (BLE 5.x) |
| Latency target | < 100 ms to edge controller |

### 12.2 Message Types (Node → Edge Controller)

```rust
/// Detection event from radar/acoustic
pub struct DetectionEvent {
    pub timestamp_ms: u64,
    pub node_id: String,
    pub position_m: [f32; 3],
    pub velocity_ms: [f32; 3],
    pub classification: String,
    pub confidence: f32,
}

/// Acoustic event
pub struct AcousticEvent {
    pub timestamp_ms: u64,
    pub node_id: String,
    pub event_type: String,         // "glass_break", "footstep", etc.
    pub direction_deg: f32,
    pub material_class: String,
    pub confidence: f32,
    pub audio_clip_path: Option<String>,  // If recorded
}

/// Environmental reading
pub struct EnvironmentalReading {
    pub timestamp_ms: u64,
    pub node_id: String,
    pub temperature_c: f32,
    pub humidity_pct: f32,
    pub pressure_hpa: f32,
    pub voc_index: Option<u16>,
}

/// Camera event (Variant D)
pub struct CameraEvent {
    pub timestamp_ms: u64,
    pub node_id: String,
    pub classification: String,
    pub confidence: f32,
    pub recording_path: Option<String>,  // If stored locally
    pub stream_available: bool,          // If streaming is active
}
```

---

## 13. Installation Considerations

### 13.1 Mounting

| Mount Type | Use Case | Notes |
|------------|----------|-------|
| Ceiling mount | Volume sensing (B, C, D) | Requires cable run |
| Wall mount | Choke point (A) | Doorway/hallway position |
| Corner mount | Volume sensing | Maximizes coverage angle |

### 13.2 Power Delivery

| Method | Applicable Variants | Notes |
|--------|-------------------|-------|
| PoE (802.3af) | All variants | Preferred for permanent install |
| USB-C | All variants | Development, temporary deployment |
| Battery backup | All variants (optional) | Brief power continuity |

### 13.3 Environmental Rating

| Rating | Variants | Notes |
|--------|----------|-------|
| IP54 | All variants (indoor) | Dust protection, splash resistant |
| IP65 | Custom variants | Outdoor covered areas |

---

## 14. Production Notes

### 14.1 Assembly Variants

Single PCB design supports all four variants:
- Populate only required components per variant
- Camera footprint unpopulated for A, B, C
- Microphone array footprint: 4-element standard, 6-element for C
- SD card slot: optional population

### 14.2 Quality Control

1. Visual inspection: Solder joint quality, component placement
2. Electrical test: Rail voltages, sensor enumeration
3. Functional test: As specified in Section 10
4. Mesh connectivity test: Golden node test fixture
5. Data handling mode verification: Per configured mode

---

## 15. Summary Table

| Aspect | Variant A | Variant B | Variant C | Variant D |
|--------|-----------|-----------|-----------|-----------|
| **Purpose** | Choke point | Volume presence | Volume + enhanced acoustic | Volume + camera |
| **Radar** | None | mmWave | mmWave | mmWave |
| **Acoustic** | None | 4-mic array | 6-mic array | 4-mic array |
| **Camera** | None | None | None | Optional |
| **Power** | Lowest | Moderate | Moderate | Higher |
| **Cost** | Lowest | Low | Moderate | Moderate-High |
| **Mesh only** | ✅ | ✅ | ✅ | ✅ |
| **WiFi/Cellular** | ❌ | ❌ | ❌ | ❌ |
| **Data to edge** | Metadata only | Metadata (default) | Metadata (default) | Configurable |

---

## 16. Directory Structure

```
always_on_node/
├── variant_a_chokepoint/
│   ├── always_on_chokepoint.kicad_pro
│   ├── always_on_chokepoint.kicad_sch
│   └── gerbers/
├── variant_b_volume/
│   ├── always_on_volume.kicad_pro
│   └── gerbers/
├── variant_c_volume_enhanced_acoustic/
│   └── gerbers/
├── variant_d_volume_camera/
│   └── gerbers/
├── bom/
│   ├── variant_a_bom.csv
│   ├── variant_b_bom.csv
│   ├── variant_c_bom.csv
│   └── variant_d_bom.csv
└── README.md
```

---

**End of Document**
