# Always-On Node — Hardware Schematic Reference Design

**Project:** AEGIS-MESH
**Node Class:** AlwaysOn — Choke Point / Volume Sensing Backbone
**Status:** Reference Design — All Variants. No component is locked.
**License:** CERN-OHL-S v2

---

## 1. Purpose

The Always-On Node is the continuously-powered sensing backbone of AEGIS-MESH. It runs at all times on standby power and provides:
- Presence detection via mmWave radar (primary sensing modality)
- Binary crossing detection via PIR (choke-point nodes)
- Acoustic event detection via microphone array (material classification, direction-of-arrival)
- Environmental sensing (temperature, humidity, air quality — atmospheric compensation)
- Optional camera module for nodes where visual capture is desired
- Mesh radio communication to edge controller

The camera-free variants (A, B, C) of this node class enforce camera-free sensing at the compile-time firmware dependency level — the `omni-sense-drivers-camera` crate is not linked for these variants. Variant D adds an optional camera module.

---

## 2. Design Philosophy

**No Artificial Limits:** The architecture supports any sensor configuration, any storage capacity, any data transmission mode. Users configure what is collected and what is transmitted.

**Modularity:** mmWave module, PIR, microphone array, environmental sensor, and optional camera are on independent sub-circuits. Each can be populated or omitted.

**Acoustic Processing:** Beamforming, material classification, and event detection run on-MCU. Mesh transmits classification metadata by default. Raw audio storage and streaming are user-configurable — users may enable raw audio recording to SD card or direct streaming to the companion app.

**Privacy:** Camera-free variants produce no imagery by architecture. Variant D with camera follows user-configured data handling policy. No system-imposed data restrictions.

---

## 3. Node Variants

### Variant A — Choke Point (Minimal)

**Purpose:** Binary detection at doorways, hallways, stairways. Lowest cost and power.

**Sensors:**
- PIR (binary motion trigger)
- 1D ToF (range confirmation, prevents false triggers)
- Environmental sensor (temperature, humidity)

**No radar, no camera, no microphone array.** This is the lowest-cost node class.

**Use case:** Deploy at every interior doorway; provides crossing events and direction inference at minimal hardware cost.

### Variant B — Volume Presence (Primary)

**Purpose:** Full presence detection and acoustic classification in open volumes.

**Sensors:**
- mmWave FMCW radar (presence, velocity, micro-Doppler)
- Microphone array (4-element tetrahedral — DOA, material classification, event detection)
- Environmental sensor (temperature, humidity, pressure, VOC)

**No camera.** Acoustic processing on-MCU. Raw audio recording user-configurable.

**Use case:** Living rooms, bedrooms, kitchens, open-plan spaces.

### Variant C — Volume Presence + Enhanced Acoustic

**Purpose:** As Variant B with higher-density microphone array for improved DOA precision and material classification.

**Sensors:**
- mmWave FMCW radar
- Microphone array (6-element for enhanced 3D DOA)
- Environmental sensor

**No camera.**

**Use case:** Large open spaces, high-value rooms requiring higher acoustic precision.

### Variant D — Volume Presence + Camera

**Purpose:** As Variant B with an optional camera module for users who want visual capture in addition to non-imaging sensing.

**Sensors:**
- mmWave FMCW radar
- Microphone array (4-element)
- Environmental sensor
- Configurable camera module

**Camera data handling:** Fully user-configured. Metadata-only, local SD recording, app streaming, or continuous recording. No system-imposed restrictions.

**Use case:** Users wanting combined geometry-based sensing AND visual documentation of their space.

---

## 4. Candidate Component Set (Test Variants — Not Locked)

### MCU

- **Variant A:** STM32U5 series (Cortex-M33, 160 MHz, ultra-low power, TrustZone, USB)
- **Variant B:** nRF5340 (Cortex-M33 + M33 app+net, native BLE 5.3, Zephyr RTOS)
- **Variant C:** ESP32-S3 (Xtensa LX7, Wi-Fi 4, BLE 5, ULP coprocessor)

### mmWave Radar Module (Variants B, C, D)

- **Variant A:** Texas Instruments IWR6843AOP (60 GHz, antenna-on-package, FMCW, integrated DSP/ARM)
- **Variant B:** Infineon BGT60ATR24C (60 GHz, SPI interface, compact package)
- **Variant C:** Acconeer XM122 (60 GHz, pulsed coherent radar, SPI, ultra-low power)
- **Variant D:** Acconeer XM125 (60 GHz, long-range variant)

### PIR Sensor (Variant A only)

- **Variant A:** Panasonic EKMB1303112K (SMD, low power, 8 m range)
- **Variant B:** Murata IRA-S210ST01 (SMD, wide angle, 5 m range)

### Microphone Array (Variants B, C, D)

**4-element tetrahedral:**
- Knowles SPH0645LM4H × 4 (I2S PDM, 65 dBA SNR)
- ST MP23DB01HP × 4 (analog MEMS, 64 dBA SNR)

**6-element circular (Variant C):**
- Knowles SPH0645LM4H × 6 (I2S PDM)

### Environmental Sensor

- **Variant A:** Bosch BME688 (temperature, humidity, pressure, VOC — single chip)
- **Variant B:** Sensirion SEN55 (PM2.5, PM10, temperature, humidity, VOC, NOx)

### Camera Module (Variant D only — user-configured)

- **Variant A:** OmniVision OV2640 (2 MP, DVP/SPI, tiny package)
- **Variant B:** OmniVision OV5640 (5 MP, autofocus, MIPI/DVP)
- **Variant C:** Arducam IMX219 (8 MP, CSI-2, configurable optics)
- **Variant D:** Sony IMX307 (2 MP, excellent low-light sensitivity)
- **Variant E:** HiMax HM01B0 (QVGA, ultra-low power, on-chip first-stage detection)

### Local Storage (Optional — All Variants)

- **Primary:** microSD card slot (SPI, up to 2 TB) — raw audio, radar IQ data, video (Variant D)
- **Secondary:** On-board QSPI NOR Flash (8–128 MB) — metadata, logs, compressed clips

Storage activation and retention configured in `aegis-mesh.toml`. Default: no local storage, metadata-only transmission.

### Mesh Radio

- **Variant A (Wi-Fi/Thread):** Murata Type 2AE (ESP32-based, Wi-Fi + BLE)
- **Variant B (BLE only):** Nordic nRF52840 module (BLE 5.3 mesh, ultra-low power)
- **Variant C (PoE Ethernet):** Direct Ethernet via W5500 (ceiling-mounted nodes with cable run)

### Power Supply

- **Input:** USB-C PD (5–20 V) OR PoE 802.3af/at (for hardwired nodes)
- **Battery backup:** LiPo 400–600 mAh (brief power interruption continuity)

---

## 5. Block Diagram

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
                 │  │ [SPI/UART] ──── Mesh Radio        │ │
                 │  │ [SPI] ──────── SD Card (optional) │ │
                 │  │ [MIPI/DVP] ──── Camera (Var D)    │ │
                 │  └──────────────────────────────────┘ │
                 └───────────────────────────────────────┘
```

---

## 6. Power Budgets (Estimates — All Variants TBD by Test)

| Variant | Sleep (mW) | Active (mW) | Peak (mW) |
|---|---|---|---|
| A — Choke Point (PIR only) | 10–50 | 100–300 | 500 |
| B — Volume Presence | 50–150 | 500–1000 | 1500 |
| C — Volume + Enhanced Acoustic | 60–180 | 600–1100 | 1600 |
| D — Volume + Camera | 60–200 | 600–1500 | 2500 |

*All values are estimates pending test measurement. Use for power supply sizing only.*

---

## 7. Testing Strategy

1. **Presence detection range test:** Walk test at 1 m, 3 m, 5 m, 10 m in direct LOS and through drywall.
2. **Acoustic DOA test:** Sound source at 0°, 45°, 90°, 135°, 180° at 2 m. Target: < ±15° error.
3. **Material classification test:** Glass break, ceramic drop, wood knock, footstep at 3 m.
4. **Power consumption test:** All states measured per variant.
5. **Mesh latency test:** End-to-end from PIR trigger to edge controller alert. Target: < 100 ms.
6. **Environmental immunity:** Steam, smoke, dust. Radar and acoustic continue operating.
7. **Raw audio recording test (if configured):** Verify audio clip written to SD card within 500 ms of event.
8. **Camera test (Variant D):** Image quality check; data handling mode verification per configured mode.

---

## 8. Directory Structure

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
└── README.md
```
