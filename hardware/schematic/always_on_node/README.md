# Always-On Node — Hardware Schematic Reference Design

**Project:** AEGIS-MESH
**Node Class:** AlwaysOn (Choke Point / Volume sensing)
**Primary Role:** Choke-point and low-cost volume sensing; continuously-powered backbone
**Status:** Reference design, variant 1 of N. All component choices are test candidates; alternatives are expected and encouraged.

---

## 1. Purpose

The Always-On Node is the continuously-powered sensing backbone of AEGIS-MESH and the workhorse of the sensor mesh. It is designed for **high-value, low-cost deployment** at choke points (doorways, hallways, entryways) and in secondary volume zones where full LiDAR coverage is unnecessary. It provides:

- **Presence detection** via mmWave radar (primary sensing modality)
- **Binary crossing detection** via PIR (choke-point deployment)
- **Acoustic event detection** via microphone array (material classification, direction-of-arrival)
- **Environmental sensing** (temperature, humidity, air quality — atmospheric compensation)
- **Mesh radio communication** to edge controller

It is the "tripwire" of the mesh — providing high-confidence crossing events and basic motion classification at minimal hardware cost.

This node class does **not** include LiDAR, event camera, or identification sensor. Those live in LidarNode and IdentityNode respectively.

---

## 2. Design Philosophy

This is a **reference design**, not a constraint. The architecture supports substitution of components to match research needs or available supply.

**Core Principles:**
- **Modularity:** The mmWave module, PIR, and microphone are on independent sub-circuits, allowing each to be populated or omitted.
- **Configurability:** Strapping resistors and GPIO headers allow configuration changes without firmware recompile.
- **Debug Access:** Full UART/SWD access for development; test points on all critical signals.
- **Power Efficiency:** Designed for battery or PoE operation; deep-sleep capability via RTC wake-up from PIR or radar interrupt.
- **No Imaging Sensors:** This node contains no camera, no optical sensor capable of producing imagery. Architecture enforces this at the dependency level (no `omni-sense-drivers-camera` in firmware Cargo.toml for this binary).
- **Privacy-Preserving Processing:** Acoustic processing (FFT, beamforming, classification) runs entirely on-MCU. No raw audio leaves the node. Only classification results (event type, direction, confidence) transit the mesh protocol.

---

## 3. Functional Architecture

```
                 ┌───────────────────────────────────────┐
                 │          Always-On Node PCB           │
                 │                                       │
Power Input ────►│ Power Management ──────► 3V3 / 1V8   │
(5V USB /        │     │                                 │
 Battery)        │     ▼                                 │
                 │  ┌──────────────────────────────────┐ │
                 │  │ Main MCU (ARM Cortex-M4/M33)     │ │
┌────────────┐   │  │                                  │ │
│ Debug/Prog │◄──┼──│ [SWD, UART]                      │ │
│ Header     │   │  │                                  │ │
└────────────┘   │  │ [GPIO] ──────────► PIR Sensor    │ │
                 │  │                                  │ │
                 │  │ [SPI/UART] ──────► mmWave Radar  │ │
                 │  │                                  │ │
                 │  │ [I2S/PDM] ◄────── MEMS Mic Array │ │
                 │  │                                  │ │
                 │  │ [I2C] ────────────► Env Sensor   │ │
                 │  │                                  │ │
                 │  │ [SPI/UART] ──────► Mesh Radio    │ │
                 │  └──────────────────────────────────┘ │
                 └───────────────────────────────────────┘
```

---

## 4. Candidate Component Set (Test Variants — Not Locked)

The following are candidate components. Architecture explicitly supports testing multiple variants at each position.

### MCU (Primary Compute)

- **Variant A:** STM32U5 series (Cortex-M33, 160 MHz, ultra-low power, TrustZone, USB) — preferred for power-sensitive deployments
- **Variant B:** nRF5340 (Cortex-M33 + M33 app+net, native BLE 5.3, Zephyr RTOS) — preferred when BLE mesh is the radio
- **Variant C:** ESP32-S3 (Xtensa LX7, Wi-Fi 4, BLE 5, ULP coprocessor) — preferred when Wi-Fi backhaul is required

**Selection criteria:** Mesh radio integration, power consumption in deep-sleep presence mode, real-time sensor interrupt handling, sufficient SRAM for acoustic processing.

### mmWave Radar Module (Primary Sensing)

- **Variant A:** Texas Instruments IWR6843AOP (60 GHz, antenna-on-package, FMCW, integrated DSP/ARM) — recommended for initial testing
- **Variant B:** Infineon BGT60ATR24C (60 GHz, SPI interface, compact package)
- **Variant C:** Acconeer XM122 (60 GHz, pulsed coherent radar, SPI, ultra-low power)

**Selection criteria:** Micro-Doppler capability, angular resolution, current draw in continuous mode, form factor for enclosure.

### PIR Sensor (Choke-Point Variant Only)

- **Variant A:** Panasonic EKMB1303112K (SMD, low power, 8 m range)
- **Variant B:** Murata IRA-S210ST01 (SMD, wide angle)

PIR is specific to choke-point nodes. Volume nodes omit PIR; they use radar for full presence detection.

### Microphone Array

- **Variant A:** Knowles SPH0645LM4H × 4 (I2S PDM MEMS, 65 dBA SNR, omnidirectional)
- **Variant B:** ST MP23DB01HP × 4 (analog MEMS, 64 dBA SNR)
- **Variant C:** InvenSense ICS-40180 × 4 (analog MEMS, 65 dBA SNR)

**Array geometry:** Tetrahedral or planar (4-element recommended for 3D DOA; 6-element for higher resolution in large-volume nodes).

### Environmental Sensor

- **Variant A:** Bosch BME688 (temperature, humidity, pressure, gas/VOC — single chip)
- **Variant B:** Sensirion SEN55 (PM2.5, PM10, temperature, humidity, VOC, NOx)

**Selection criteria:** Required quantity set per deployment context. Indoor residential: BME688 sufficient. Air quality environments: SEN55.

### Mesh Radio

- **Variant A (Wi-Fi/Thread):** Murata Type 2AE (ESP32-based, Wi-Fi + BLE, proven supply chain)
- **Variant B (BLE only):** Nordic nRF52840 module (BLE 5.3 mesh, ultra-low power, USB)
- **Variant C (PoE):** Direct Ethernet via W5500 (no wireless; ceiling-mounted nodes with cable run)

**Selection criteria:** Latency requirements, power budget, installation environment. PoE preferred for fixed ceiling nodes; wireless for temporary or retrofit installations.

### Power Supply

- **Input:** USB-C PD (5–20 V) OR PoE 802.3af/at (for hardwired nodes)
- **Regulation:** LDO + buck/boost DC-DC (topology TBD per MCU VCC requirements)
- **Battery backup:** LiPo 400–600 mAh (maintains operation during brief power interruptions)

---

## 5. Interface Mapping (Reference)

| Component | Interface | MCU Pins (Example) | Notes |
|---|---|---|---|
| **mmWave Radar** | SPI | SCK, MISO, MOSI, CS, IRQ, RST | Or UART (TX/RX) |
| **PIR** | GPIO | GPIO_EXT0 | Interrupt-capable |
| **Microphone Array** | I2S / PDM | I2S_CK, I2S_WS, I2S_DATA | 2–6 channels |
| **Environmental** | I2C | I2C_SDA, I2C_SCL | BME688 or SEN55 |
| **Mesh Radio** | UART / SPI | UART_TX, UART_RX | Or integrated in MCU |
| **Debug** | SWD | SWDIO, SWCLK, SWO | Standard ARM 10-pin header |
| **Console** | UART | UART_TX, UART_RX | 115200 baud |
| **Power** | Power | USB-C 5V / PoE / VBAT | LDO to 3.3V |

---

## 6. Power Architecture

**Flexible power sources:**
- **Primary (permanent):** USB-C (5V) or PoE 802.3af/at.
- **Secondary (portable):** Li-Ion / Li-Po battery (3.7V) with on-board charging (e.g., MCP73831).
- **Regulation:** 3.3V LDO (e.g., AP2112) for digital logic.

**Power Modes:**

| Mode | Active Peripherals | MCU State | Wake Source |
|---|---|---|---|
| **Active** | All sensors + radio | Running | N/A |
| **Low-Power** | Radar (presence), PIR armed | Sleep | Radar IRQ, PIR GPIO |
| **Deep Sleep** | RTC + PIR only | Deep sleep | PIR interrupt |

---

## 7. Design Considerations

### 7.1 RF Isolation

- Keep mmWave antenna area clear of ground planes and copper traces on all layers.
- Follow module manufacturer keep-out recommendations strictly.
- Minimum 5 mm clearance to any copper fill.

### 7.2 Acoustic Isolation

- Place MEMS microphones away from vibration sources (voltage regulators, RF components).
- Use a sealed gasket between microphone port and enclosure wall.
- Avoid mechanical coupling paths from PCB to microphone housing.

### 7.3 PIR Placement

- Position PIR with clear field of view; no obstructions within 20°.
- Keep away from heat sources (voltage regulators, mmWave module) to prevent false triggers.
- Shield from solar/IR radiation if near windows.

---

## 8. Schematic Files

| File | Description |
|---|---|
| `always_on_node.kicad_pro` | KiCad project file |
| `always_on_node.kicad_sch` | Main schematic |
| `always_on_node.kicad_pcb` | PCB layout |
| `always_on_node.pdf` | PDF export for review |

---

## 9. Manufacturing Notes

- **Layer Count:** 4-layer recommended for RF integrity and ground plane quality.
- **Impedance Control:** Required for high-speed SPI and antenna traces.
- **Assembly:** Hand-solderable prototypes; refer to `../assembly/` for pick-and-place files.

---

## 10. Testing Strategy

For each hardware variant combination, the following tests are required before production:

1. **Presence detection range test:** Walk test at 1 m, 3 m, 5 m, 10 m in direct LOS and through one drywall. Record detection rate vs. range.
2. **Acoustic DOA test:** Sound source at 0°, 45°, 90°, 135°, 180° at 2 m range. Record direction-of-arrival error (degrees).
3. **Material classification test:** Glass break, ceramic drop, wooden knock, footstep (hard and soft sole) at 3 m. Record classification accuracy.
4. **Power consumption test:** Idle (all sensors polling), active (radar detection event), peak (acoustic classification burst). Record mA per state.
5. **Mesh latency test:** End-to-end latency from PIR trigger to edge controller alert. Target: < 100 ms.
6. **Environmental immunity:** Operate under steam, cooking smoke, and dust. Verify radar and acoustic continue operating with no false positives.

All results recorded in `hardware/testing/results/` with variant combination identified.
