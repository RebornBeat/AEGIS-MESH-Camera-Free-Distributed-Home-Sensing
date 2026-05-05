# Always-On Node — Hardware Schematic Reference Design

**Node Class:** AlwaysOn (Choke Point / Volume sensing)
**Status:** Reference design, variant 1 of N. All component choices are test candidates; alternatives are expected and encouraged.

---

## Purpose

The Always-On Node is the continuously-powered sensing backbone of AEGIS-MESH. It runs at all times on standby power and handles:
- Presence detection via mmWave radar (primary sensing modality)
- Binary crossing detection via PIR (choke-point deployment)
- Acoustic event detection via microphone array (material classification, direction-of-arrival)
- Environmental sensing (temperature, humidity, air quality — atmospheric compensation)
- Mesh radio communication to edge controller

This node class does **not** include LiDAR, event camera, or identification sensor. Those live in LidarNode and IdentityNode respectively.

---

## Candidate Component Set (Test Variants — Not Locked)

The following are candidate components for the reference design. The architecture explicitly supports testing multiple variants at each position.

### MCU (Primary Compute)
- **Variant A:** STM32U5 series (Cortex-M33, 160 MHz, ultra-low power, TrustZone, USB)
- **Variant B:** nRF5340 (Cortex-M33 + M33 app+net, native BLE 5.3, Zephyr RTOS)
- **Variant C:** ESP32-S3 (Xtensa LX7, Wi-Fi 4, BLE 5, ULP coprocessor)

MCU selection criteria: mesh radio integration, power consumption in deep-sleep presence mode, real-time sensor interrupt handling, sufficient SRAM for acoustic processing.

### mmWave Radar Module (Primary Sensing)
- **Variant A:** Texas Instruments IWR6843AOP (60 GHz, antenna-on-package, FMCW, integrated DSP/ARM) — recommended for initial testing
- **Variant B:** Infineon BGT60ATR24C (60 GHz, SPI interface, compact package)
- **Variant C:** Acconeer XM122 (60 GHz, pulsed coherent radar, SPI, ultra-low power)

Selection criteria: micro-Doppler capability, angular resolution, current draw in continuous mode, form factor for enclosure.

### PIR Sensor (Choke-Point Variant Only)
- **Variant A:** Panasonic EKMB1303112K (SMD, low power, 8m range)
- **Variant B:** Murata IRA-S210ST01 (SMD, wide angle)

PIR is population-specific to choke-point nodes. Volume nodes omit PIR; they use radar for full presence detection.

### Microphone Array
- **Variant A:** Knowles SPH0645LM4H × 4 (I2S PDM MEMS, 65 dBA SNR, omnidirectional)
- **Variant B:** ST MP23DB01HP × 4 (analog MEMS, 64 dBA SNR)
- **Variant C:** InvenSense ICS-40180 × 4 (analog MEMS, 65 dBA SNR)

Array geometry: tetrahedral or planar (4-element recommended for 3D direction-of-arrival; 6-element for higher resolution in large-volume nodes).

### Environmental Sensor
- **Variant A:** Bosch BME688 (temperature, humidity, pressure, gas/VOC — single chip)
- **Variant B:** Sensirion SEN55 (PM2.5, PM10, temperature, humidity, VOC, NOx)

Selection criteria: required quantity set per deployment context. Indoor residential: BME688 is sufficient. Environments with air quality concerns: SEN55.

### Mesh Radio
- **Variant A (Wi-Fi/Thread):** Murata Type 2AE (ESP32-based, Wi-Fi + BLE, proven supply chain)
- **Variant B (BLE only):** Nordic nRF52840 module (BLE 5.3 mesh, ultra-low power, USB)
- **Variant C (PoE):** Direct Ethernet via W5500 (no wireless; ceiling-mounted nodes with cable run)

Selection criteria: latency requirements, power budget, installation environment. PoE preferred for fixed ceiling nodes; wireless for temporary or retrofit installations.

### Power Supply
- **Input:** USB-C PD (5–20 V) OR PoE 802.3af/at (for hardwired nodes)
- **Regulation:** LDO + buck/boost DC-DC (exact topology TBD per MCU VCC requirements)
- **Battery backup:** LiPo 400–600 mAh (maintains operation during brief power interruptions)

---

## Schematic Block Diagram

```
 ┌──────────────────────────────────────────────────────────────┐
 │                    Always-On Node PCB                        │
 │                                                              │
 │   Power Input ──► Power Management ──► 3V3 / 1V8 rails      │
 │                          │                                   │
 │         ┌────────────────┼────────────────┐                  │
 │         ▼                ▼                ▼                  │
 │      MCU (SoC)    mmWave Module    Mic Array (×4)            │
 │     [STM32U5]     [IWR6843AOP]   [SPH0645×4 I2S]            │
 │         │                │                │                  │
 │         ├────────────────┘                │                  │
 │         │         SPI/UART                │                  │
 │         │                                 │                  │
 │         ├──────────────────────────────────┘                 │
 │         │         I2S PDM                                    │
 │         │                                                    │
 │         ├──► PIR ──► GPIO INT     (choke-point only)         │
 │         │                                                    │
 │         ├──► Env Sensor (BME688) via I2C                     │
 │         │                                                    │
 │         └──► Mesh Radio Module via SPI/UART + GPIO           │
 │                                                              │
 └──────────────────────────────────────────────────────────────┘
```

---

## Key Design Constraints

**No imaging sensors:** This node contains no camera, no optical sensor capable of producing imagery. The architecture enforces this at the dependency level (no `omni-sense-drivers-camera` in firmware Cargo.toml for this binary).

**Privacy-preserving processing:** Acoustic processing (FFT, beamforming, classification) runs entirely on-MCU. No raw audio leaves the node. Only classification results (event type, direction, confidence) transit the mesh protocol.

**Kill-switch requirement:** This node class does not have a kill switch (none required — no identity-capable hardware). Identity nodes are the only nodes with kill switches.

**Environmental seal:** Target IP54 minimum (splash resistance). IP67 preferred for ceiling-mounted nodes in kitchens or bathrooms. Enclosure design in `mechanical/cad/`.

---

## Testing Strategy

For each hardware variant combination, the following tests are required before production:

1. **Presence detection range test:** Walk test at 1 m, 3 m, 5 m, 10 m in direct line-of-sight and through one wall (drywall). Record detection rate vs. range.
2. **Acoustic direction-of-arrival test:** Sound source at 0°, 45°, 90°, 135°, 180° at 2 m range. Record DOA error (degrees).
3. **Material classification test:** Glass break, ceramic drop, wooden knock, footstep (hard and soft sole) at 3 m. Record classification accuracy.
4. **Power consumption test:** Idle (all sensors polling), active (radar detection event), peak (acoustic classification burst). Record mA per state.
5. **Mesh latency test:** End-to-end latency from PIR trigger to edge controller alert. Target: < 100 ms.
6. **Environmental immunity:** Operate under steam (bathroom simulation), cooking smoke, and dust conditions. Verify radar and acoustic continue operating while confirming no false LiDAR pings (LiDAR not present on this node).

All test results are recorded in `hardware/testing/` for the tested variant combination.
