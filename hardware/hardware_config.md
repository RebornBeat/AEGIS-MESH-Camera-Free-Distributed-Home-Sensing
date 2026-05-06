# Hardware Configuration Reference — AEGIS-MESH

**Project:** AEGIS-MESH
**Domain:** Hardware reference, interface definitions, pin mappings, power architecture
**Status:** Research / Reference Design

---

## 1. Philosophy: No Imposed Constraints

This document specifies the **reference hardware interfaces** for AEGIS-MESH node designs. It defines how sensors connect, how power is managed, and how nodes communicate.

**It does not define:**
- Sensor specifications (resolution, range, accuracy are determined by selected modules).
- Storage limits (nodes support any capacity compatible with the MCU interface).
- Processing limits (clock speeds, memory, compute are configuration choices).

**Core Principle:** The architecture is sensor-agnostic and compute-agnostic. The reference designs provide connectivity and power infrastructure; sensors and processors are configuration choices that evolve with research.

---

## 2. Node Classes and Roles

AEGIS-MESH uses a **Hybrid Node Strategy**. Not every node needs every sensor.

| Node Class | Primary Role | Typical Sensors | Form Factor |
|---|---|---|---|
| **Always-On Node** | Choke-point detection, low-power presence | PIR, 1D ToF, mmWave (optional) | Small, battery or PoE |
| **Volume Node** | Full 3D perception, geometry, material analysis | Solid-state LiDAR, mmWave Radar, Acoustic Array | Larger, PoE or wired |
| **Identity Node** | Episodic identification (opt-in) | Configurable Camera / Biometric | Isolated, kill switch mandatory |

---

## 3. Standard Interface Header (20-pin, 2.54 mm Pitch)

All AEGIS-MESH nodes expose a common interface standard for firmware compatibility across hardware variants.

| Pin | Signal | Direction | Description |
|---|---|---|---|
| 1 | VCC_3V3 | Power | 3.3 V regulated supply to node |
| 2 | VCC_SENSOR | Power | Sensor power rail (switchable on Identity Node via Kill Switch) |
| 3 | GND | — | Ground |
| 4 | MESH_TX | Output | Mesh radio UART TX |
| 5 | MESH_RX | Input | Mesh radio UART RX |
| 6 | MESH_CS | Output | Mesh radio SPI CS (if SPI variant) |
| 7 | RADAR_MOSI | Output | mmWave radar SPI MOSI |
| 8 | RADAR_MISO | Input | mmWave radar SPI MISO |
| 9 | RADAR_SCK | Output | mmWave radar SPI clock |
| 10 | RADAR_CS | Output | mmWave radar SPI CS |
| 11 | I2S_BCLK | Output | Microphone array bit clock |
| 12 | I2S_WS | Output | Microphone array word select |
| 13 | I2S_DATA | Input | Microphone array data (daisy-chained) |
| 14 | I2C_SDA | Bidirectional | Environmental + haptic I2C data |
| 15 | I2C_SCL | Output | Environmental + haptic I2C clock |
| 16 | PIR_INT | Input | PIR interrupt (choke-point nodes only) |
| 17 | KILL_SW_GPIO | Input | Kill switch state (Identity Node: HIGH = enabled) |
| 18 | LED_STATUS | Output | Status LED drive |
| 19 | BOOT_SELECT | Input | Firmware boot mode select |
| 20 | RESET | Input (active low) | MCU reset |

---

## 4. LiDAR Node Additional Interface (Differential)

| Pin | Signal | Direction | Description |
|---|---|---|---|
| A | LIDAR_ETH_TX+ | Differential | LiDAR Ethernet TX+ |
| B | LIDAR_ETH_TX- | Differential | LiDAR Ethernet TX- |
| C | LIDAR_ETH_RX+ | Differential | LiDAR Ethernet RX+ |
| D | LIDAR_ETH_RX- | Differential | LiDAR Ethernet RX- |
| E | EVENT_CAM_USB_D+ | Differential | Event camera USB D+ |
| F | EVENT_CAM_USB_D- | Differential | Event camera USB D- |

---

## 5. Power Sources and Management

### 5.1 Power Sources

AEGIS-MESH is source-agnostic:

| Source | Typical Use Case |
|---|---|
| **PoE 802.3af/at** | Volume Nodes, Identity Nodes (continuous high power) |
| **USB-C (5V)** | Development, always-on nodes |
| **Li-Ion / Li-Po Battery** | Portable or choke-point nodes |
| **Solar + Battery** | Outdoor / perimeter nodes |

### 5.2 Power Domains

| Domain | Voltage | Source | Usage |
|---|---|---|---|
| `V_MAIN` | 5.0V nominal | USB-C, PoE, Battery | Raw input |
| `V_MCU` | 3.3V | Regulated from V_MAIN | MCU and low-power sensors |
| `V_RADIO` | 3.3V | Regulated from V_MAIN | Mesh radio module |
| `V_SENSOR` | Configurable | Regulated from V_MAIN | High-power sensors (LiDAR, Radar) |

### 5.3 Power Mode Summary

| Node Class | Deep Sleep (mW) | Active (mW) | Peak (mW) |
|---|---|---|---|
| Always-On (radar + mic) | 50–150 | 500–1000 | 1500 |
| Choke-Point (PIR only) | 10–50 | 100–300 | 500 |
| LiDAR Node (event trigger) | 200–500 | 2000–5000 | 8000 |
| LiDAR Node (continuous) | — | 8000–15000 | 15000 |
| Identity Node (dormant) | 50–100 | 500–2000 | 3000 |

*All values are estimates pending hardware measurement. Use for power supply sizing only.*

---

## 6. Mesh Radio Interface

All nodes communicate over the AEGIS-MESH protocol. Physical layer is agnostic.

**Supported Physical Layers:**
- **Wi-Fi 6/6E** (Volume Nodes, high bandwidth)
- **Thread / Matter** (Low-power, Always-On Nodes)
- **BLE 5.x** (Configuration mode, fallback)

**Standard Radio Interface (SPI/UART):**

| Signal | Description |
|---|---|
| `RADIO_MOSI` | SPI Master Out Slave In |
| `RADIO_MISO` | SPI Master In Slave Out |
| `RADIO_CLK` | SPI Clock |
| `RADIO_CS` | Chip Select |
| `RADIO_IRQ` | Interrupt Request |
| `RADIO_RST` | Reset |

**Firmware Note:** Specific pin mappings are defined in MCU-specific firmware configuration, not hardcoded in hardware. This allows the same PCB to support different MCU families by re-routing in firmware.

---

## 7. Debug & Configuration Interface

All nodes expose a standard debug header.

**Standard 10-pin ARM Cortex Debug Connector (0.05" pitch):**

| Pin | Signal | Description |
|---|---|---|
| 1 | `VTref` | Target voltage reference |
| 2, 3, 4 | `GND` | Ground |
| 5 | `SWDIO` | Serial Wire Debug I/O |
| 6 | `SWCLK` | Serial Wire Clock |
| 7 | `SWO` | Serial Wire Output (optional printf) |
| 8 | `NC` | No Connect |
| 9 | `RST` | MCU Reset |
| 10 | `NC` | No Connect |

**UART Console:**

| Signal | Description |
|---|---|
| `UART_TX` | Debug console transmit (115200 baud) |
| `UART_RX` | Debug console receive |

---

## 8. Class-Specific Configurations

### 8.1 Always-On Node (Choke Point)

**Role:** Binary detection at doorways, hallways. Low cost, low power.

| Interface | Signal Names | Connection |
|---|---|---|
| **PIR** | `PIR_OUT` | GPIO, interrupt-capable |
| **ToF (1D)** | `TOF_SDA`, `TOF_SCL` | I2C |
| **mmWave (optional)** | `MMWAVE_TX`, `MMWAVE_RX` | UART |
| **Mesh Radio** | `RADIO_*` | SPI or UART |

**Power:** Deep sleep > 95% of time. PIR interrupt wakes MCU. ToF/mmWave polled on wake for confirmation.

---

### 8.2 Volume Node (Primary Perception)

**Role:** Full 3D tracking, geometry, material classification.

| Interface | Signal Names | Connection |
|---|---|---|
| **LiDAR** | `LIDAR_MOSI/MISO/CLK/CS` | SPI or Ethernet |
| **mmWave Radar** | `MMWAVE_TX/RX` | UART or SPI |
| **Acoustic Array** | `MIC_CLK`, `MIC_WS`, `MIC_DATA[0:3]` | I2S / PDM |
| **Environmental** | `ENV_SDA`, `ENV_SCL` | I2C |

**Processing Requirements:** High-performance MCU or SoM (STM32H7, ESP32-S3, Linux SoM). Hardware FPU for acoustic processing. Sufficient RAM for point cloud buffering.

---

### 8.3 Identity Node (Opt-In Identification)

**Role:** Face recognition, object classification. **Strictly opt-in. Kill switch mandatory.**

| Interface | Signal Names | Connection |
|---|---|---|
| **Camera** | `CAM_D[0:7]`, `CAM_PCLK`, `CAM_HSYNC`, `CAM_VSYNC` | Parallel CSI or MIPI CSI-2 |
| **Kill Switch** | `KILL_SWITCH_EN` | GPIO (read at boot; no override) |
| **Separate MCU** | Not shared with sensing mesh | Physical isolation required |

**Privacy:**
- Identity Node is on a **separate physical mesh** from the sensing mesh.
- Data flows to local edge controller only. No direct internet connection.
- No raw frames ever transmitted. Only classification metadata.

---

## 9. Mesh Protocol Compatibility

All nodes must run firmware from `firmware/src/bin/<node_name>.rs` compiled for the correct MCU target. The mesh protocol (documented in `docs/protocol/protocol_spec.md`) enforces:

- Message signing: HMAC-SHA256, key established at pairing.
- Sequence numbers: Replay protection.
- Heartbeat messages: 10 Hz node health monitoring.
- Nodes that miss 3 consecutive heartbeat periods are flagged `Offline` by the edge controller.

---

## 10. Expansion and Customization

All designs include:
- Unused GPIO broken out to test points.
- Standard expansion headers (I2C, SPI, UART).
- Firmware-configurable pin mappings where MCU permits.

This enables testing **any sensor variant** without PCB redesign. Connect via expansion headers and update firmware `board.h` configuration.

---

## 11. Summary

The hardware configuration is designed to be:
- **Flexible:** Supports any sensor, any power source, any MCU family.
- **Standardized:** Common 20-pin interface header for mesh radio, power, and debug.
- **Privacy-Respecting:** Identity Nodes require physical hardware kill switches — no exceptions.
- **Research-Oriented:** No imposed limits on resolution, storage, or compute.

This document defines connectivity and power infrastructure. Sensor specifications and capabilities are configuration choices, not constraints.
