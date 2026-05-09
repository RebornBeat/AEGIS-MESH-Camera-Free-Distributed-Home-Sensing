# Hardware Configuration Reference — AEGIS-MESH

**Project:** AEGIS-MESH
**Domain:** Hardware reference, interface definitions, connectivity architecture, power management
**Status:** Research / Reference Design
**Version:** 0.3

---

## 1. Philosophy: No Imposed Constraints

This document specifies the **reference hardware interfaces**, **connectivity architecture**, and **power management** for AEGIS-MESH node designs. It defines how sensors connect, how power is managed, how nodes communicate, and how the edge controller serves as the sole external gateway.

**It does not define:**
- Sensor specifications (resolution, range, accuracy are determined by selected modules)
- Storage limits (nodes support any capacity compatible with the MCU interface)
- Processing limits (clock speeds, memory, compute are configuration choices)
- Data handling restrictions (all storage and transmission is user-configured)

**Core Principles:**
- **No Artificial Limits:** Interfaces support standard protocols at maximum speeds
- **Connectivity Agnostic:** Multiple mesh transports supported; external connectivity at edge controller only
- **Storage Agnostic:** microSD, internal flash, NAS, or companion app — user chooses
- **User-Controlled Data Handling:** No system-imposed restrictions on storage, streaming, or retention

---

## 2. Critical Architectural Constraint: Edge Controller as Sole External Gateway

**Only the Edge Controller connects to external networks (WiFi, Cellular).** This is not a configuration choice — it is an architectural constraint enforced at both hardware and firmware levels.

**All sensing nodes communicate exclusively via Mesh Network to the edge controller.** They have no WiFi, no cellular, and no direct connection to the companion app.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      AEGIS-MESH Connectivity Model                          │
│                                                                              │
│  Always-On Nodes ──┐                                                         │
│  LiDAR Nodes───────┤── Mesh Network (WiFi/Thread/PoE) ──► Edge Controller   │
│  Identity Nodes────┘                                      │                  │
│                                                           ├──► WiFi ──► App │
│                                                           ├──► Cellular ──► App │
│                                                           └──► BLE direct ──► App │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Why This Architecture?

| Constraint | WiFi on Sensor Node | Mesh Radio (BLE/Thread/PoE) |
|------------|---------------------|----------------------------|
| **Installation** | Requires WiFi credentials per node | Single PoE/thread drop per area |
| **Security** | Each node is attack surface | Single gateway to protect |
| **Cost** | WiFi module per node | Mesh radio shared |
| **Power** | 300-500 mW per node | 50-150 mW per node |
| **Maintenance** | Update credentials across fleet | Centralized credential management |

**Conclusion:** Centralizing external connectivity at the edge controller reduces attack surface, simplifies deployment, and enables professional-grade security management.

---

## 3. Transport Layer Hierarchy

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         BANDWIDTH HIERARCHY                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│  Thread / BLE Mesh (250 Kbps - 1 Mbps)                                       │
│  → Control plane (configuration, commands, health)                          │
│  → Metadata (detections, classification results)                            │
│  → Low-bandwidth sensor data                                                 │
│  → Always-on for choke-point nodes                                           │
├─────────────────────────────────────────────────────────────────────────────┤
│  WiFi Mesh (50-300+ Mbps)                                                    │
│  → High-bandwidth sensor data (LiDAR point clouds)                           │
│  → Camera streams from LiDAR and Identity nodes                               │
│  → SLAM data transfer                                                        │
│  → Primary mesh transport for volume nodes                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│  PoE Ethernet (100 Mbps - 1 Gbps)                                            │
│  → Maximum bandwidth for dense deployments                                   │
│  → Simultaneous power and data                                               │
│  → Preferred for LiDAR nodes with 360° camera arrays                         │
│  → Wired reliability for fixed installations                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│  WiFi 802.11ac/ax — EDGE CONTROLLER ONLY                                     │
│  → Primary companion app connection                                          │
│  → 360° video streaming to app                                               │
│  → SLAM map download                                                         │
│  → Recording library access                                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│  Cellular LTE/5G — EDGE CONTROLLER ONLY                                      │
│  → Remote access when away from home network                                 │
│  → Alert delivery to mobile app                                              │
│  → Evidence relay for security monitoring                                    │
│  → Optional; user-configured                                                 │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 4. Node Classes and Roles

AEGIS-MESH uses a **Hybrid Node Strategy**. Not every node needs every sensor.

| Node Class | Primary Role | Typical Sensors | Form Factor | Mesh Transport |
|---|---|---|---|---|
| **Always-On Node (Variant A)** | Choke-point binary detection | PIR + 1D ToF | Small, wall-mount | BLE/Thread |
| **Always-On Node (Variant B)** | Volume presence | mmWave + Acoustic | Ceiling/wall mount | WiFi/Thread |
| **Always-On Node (Variant C)** | Volume + enhanced acoustic | mmWave + 6-mic array | Ceiling mount | WiFi |
| **Always-On Node (Variant D)** | Volume + camera | mmWave + Acoustic + Camera | Ceiling mount | WiFi |
| **LiDAR Node (Variant A)** | Basic geometry | LiDAR + Event camera | Ceiling mount | PoE/WiFi |
| **LiDAR Node (Variant B)** | Geometry + fog immunity | LiDAR + Event + mmWave | Ceiling mount | PoE/WiFi |
| **LiDAR Node (Variant C)** | Geometry + visual SLAM | LiDAR + Event + Camera | Ceiling mount | PoE |
| **LiDAR Node (Variant D)** | 360° geometry + visual | LiDAR + Event + 360° Camera Array | Ceiling center | PoE |
| **Identity Node** | Configurable identification | Camera / Biometric | Entry point | WiFi/Thread |
| **Edge Controller** | Central hub + gateway | Compute + Storage + WiFi + Cellular | Server closet / shelf | External only |

---

## 5. Core Node Architecture

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                         AEGIS-MESH Node Architecture                         │
├──────────────────────────────────────────────────────────────────────────────┤
│  MCU Core (Configurable Per Node Class)                                       │
│  - ARM Cortex-M4F / M7 / M33 for sensor nodes                                │
│  - Linux SoM (Raspberry Pi CM4 / NXP i.MX 8M) for edge controller            │
│  - ESP32-S3 for cost-optimized variants                                      │
├──────────────────────────────────────────────────────────────────────────────┤
│  Sensing Interfaces (Populated Per Node Class)                                │
│  - mmWave Radar (SPI/UART)                                                   │
│  - Solid-State LiDAR (Ethernet/SPI)                                          │
│  - Event Camera (USB/MIPI)                                                   │
│  - Acoustic Array (I2S/PDM)                                                  │
│  - PIR (GPIO)                                                                │
│  - ToF (I2C/SPI)                                                             │
│  - Environmental (I2C)                                                       │
│  - Camera (MIPI CSI-2) — Identity and LiDAR C/D variants                     │
│  - 360° Camera Array (multiple MIPI + HW FSYNC) — LiDAR Variant D            │
├──────────────────────────────────────────────────────────────────────────────┤
│  Storage Interface (Per Configuration)                                        │
│  - microSD card (SPI/SDIO, up to 2 TB)                                       │
│  - QSPI NOR Flash (metadata, logs, model weights)                            │
│  - NVMe SSD (edge controller high-capacity variant)                          │
├──────────────────────────────────────────────────────────────────────────────┤
│  Mesh Network Interface — ALL SENSOR NODES                                    │
│  - WiFi 802.11ac/ax (high-bandwidth nodes)                                   │
│  - Thread / Matter (low-power nodes)                                         │
│  - BLE 5.x (configuration mode, choke-point nodes)                           │
│  - PoE Ethernet (LiDAR nodes, high reliability)                              │
│  - NO direct WiFi to companion app                                            │
│  - NO cellular interface                                                      │
├──────────────────────────────────────────────────────────────────────────────┤
│  Edge Controller Additional — External Network Interfaces                     │
│  - WiFi 802.11ac/ax (built into SoM) — companion app primary                 │
│  - Cellular LTE/5G module (nano-SIM or eSIM) — remote access                 │
│  - BLE 5.x (direct app connection fallback)                                  │
│  - USB-C (PC companion + firmware update)                                    │
│  - Ethernet (wired companion app + NAS)                                      │
├──────────────────────────────────────────────────────────────────────────────┤
│  Power Management                                                            │
│  - PoE 802.3af/at (15.4-25.5 W) — preferred for fixed nodes                  │
│  - USB-C (5V) — development and portable configurations                      │
│  - Battery backup (optional for edge controller)                             │
├──────────────────────────────────────────────────────────────────────────────┤
│  User Interface (Per Node)                                                    │
│  - Status LEDs                                                               │
│  - Hardware Kill Switch (Identity Node — mandatory)                          │
│  - Optional hardware switch (camera-capable nodes — user choice)             │
├──────────────────────────────────────────────────────────────────────────────┤
│  Debug Interface                                                              │
│  - SWD (Serial Wire Debug)                                                   │
│  - UART Console (115200 baud)                                                │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## 6. Standard Interface Header (20-pin, 2.54 mm Pitch)

All AEGIS-MESH sensor nodes expose a common interface standard for firmware compatibility across hardware variants.

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
| 17 | CAM_HW_SW_GPIO | Input | Camera hardware switch state (if installed) |
| 18 | LED_STATUS | Output | Status LED drive |
| 19 | BOOT_SELECT | Input | Firmware boot mode select |
| 20 | RESET | Input (active low) | MCU reset |

### Edge Controller Additional Interfaces

| Interface | Description |
|---|---|
| WiFi antenna | U.FL or integrated antenna |
| Cellular SIM slot | nano-SIM 4FF or eSIM footprint |
| Cellular antenna | U.FL for external antenna |
| USB-C | Admin + firmware update + charging |
| Ethernet (optional) | High-speed access or PoE aggregator |

---

## 7. LiDAR Node Additional Interface (Differential)

| Pin | Signal | Direction | Description |
|---|---|---|---|
| A | LIDAR_ETH_TX+ | Differential | LiDAR Ethernet TX+ |
| B | LIDAR_ETH_TX- | Differential | LiDAR Ethernet TX- |
| C | LIDAR_ETH_RX+ | Differential | LiDAR Ethernet RX+ |
| D | LIDAR_ETH_RX- | Differential | LiDAR Ethernet RX- |
| E | EVENT_CAM_USB_D+ | Differential | Event camera USB D+ |
| F | EVENT_CAM_USB_D- | Differential | Event camera USB D- |
| G–J | CAM_MIPI_CLK+/-, D0+/- | Differential | Camera MIPI (100 Ω diff, 2 mil match) |

### 360° Camera Array Interface (LiDAR Variant D)

| Signal | Description |
|---|---|
| `CAM_n_MIPI_CLK+/-` | MIPI clock per camera n (n = 0 to N-1) |
| `CAM_n_MIPI_D0+/-` | MIPI data per camera n |
| `CAM_FSYNC` | Hardware sync broadcast to all cameras |
| `CAM_PWDN_n` | Power-down control per camera (optional) |

---

## 8. Edge Controller Hardware Architecture

### 8.1 Role in the Mesh

The Edge Controller is AEGIS-MESH's equivalent to SENTINEL-WEAR's Belt Node — the central compute hub and **sole external gateway**.

| Function | Edge Controller | Belt Node (SW Equivalent) |
|----------|-----------------|--------------------------|
| Compute hub | ✅ Yes | ✅ Yes |
| Fusion center | ✅ Yes | ✅ Yes |
| SLAM processor | ✅ Yes | ✅ Yes |
| API server | ✅ Yes | ✅ Yes |
| Recording manager | ✅ Yes | ✅ Yes |
| WiFi gateway | ✅ Yes | ✅ Yes |
| Cellular gateway | ✅ Yes | ✅ Yes |
| **Power source** | AC power (unlimited) | Battery (limited) |
| **Form factor** | Shelf/rack mounted | Wearable |
| **Thermal constraints** | Relaxed (active cooling possible) | Tight (skin contact) |

### 8.2 Hardware Variants

#### Variant A — SBC Class (Raspberry Pi 4/5)

| Component | Specification |
|-----------|---------------|
| Compute | ARM Cortex-A72/A76, quad-core |
| RAM | 4-8 GB |
| Storage | SD card or NVMe |
| WiFi | Built-in 802.11ac |
| Cellular | Optional USB module |
| Ethernet | 1 Gbps built-in |
| Power | USB-C 5V/3A |
| Cost | $50-100 |
| Capability | Sparse tracking, API, 2-4 camera streams, basic SLAM |

#### Variant B — ARM with NPU (Jetson Nano/Orin Class)

| Component | Specification |
|-----------|---------------|
| Compute | ARM + NVIDIA GPU + NPU |
| RAM | 4-16 GB |
| Storage | NVMe SSD |
| WiFi | Built-in or external |
| Cellular | Optional via M.2 or USB |
| Power | 10-15W active |
| Cost | $200-500 |
| Capability | Full SLAM, 8+ camera streams, 360° processing, neural inference |

#### Variant C — x86 Mini PC

| Component | Specification |
|-----------|---------------|
| Compute | Intel/AMD x86 |
| RAM | 8-32 GB |
| Storage | NVMe SSD |
| WiFi | External card |
| Cellular | Optional via USB |
| Power | 15-30W |
| Cost | $300-800 |
| Capability | Maximum flexibility, Docker containers, full SLAM, unlimited streams |

### 8.3 Power Architecture

**Edge controller variants are AC-powered:**

| Variant | Power Source | Typical Draw | Cooling |
|---------|--------------|--------------|---------|
| SBC (Var A) | USB-C 5V/3A | 5-10W | Passive |
| NPU (Var B) | 12-19V DC | 10-20W | Passive + heatsink |
| x86 (Var C) | 19V DC | 15-35W | Active fan |

**Battery backup (optional):**
- UPS module for power-loss protection
- Graceful shutdown on power failure
- Recording flush to storage before shutdown

### 8.4 Edge Controller Interface Summary

| Interface | Purpose | Notes |
|-----------|---------|-------|
| WiFi 802.11ac/ax | Companion app primary | Built into most SoMs |
| Cellular SIM | Remote access | Nano-SIM slot on PCB |
| Cellular antenna | RF connection | U.FL to external antenna |
| USB-C | Admin + firmware update | Direct PC connection |
| Ethernet 1Gbps | High-speed access | NAS, PC companion app |
| PoE pass-through | Power LiDAR nodes | Optional PoE injector |

---

## 9. Cellular / SIM Integration

### 9.1 Edge Controller SIM Interface

The edge controller PCB includes:
- nano-SIM card slot (4FF form factor) OR eSIM footprint
- Cellular module interface (UART for Cat 1/4, USB 3.x for 5G)
- RF antenna connector (U.FL) for external cellular antenna
- GNSS capability (if cellular module includes)

### 9.2 Supported Cellular Modules

| Module | Technology | Speed | Interface | Power (Active) | Use Case |
|--------|------------|-------|-----------|----------------|----------|
| Quectel EC21 | LTE Cat 1 | 5-10 Mbps | UART | 150-300 mA | Alerts, metadata only |
| Quectel EC25 | LTE Cat 4 | 50-150 Mbps | UART/USB | 400-600 mA | Compressed video clips |
| Quectel RM502Q-AE | 5G Sub-6GHz | 100+ Mbps | USB 3.x | 800-1200 mA | Live 360° streaming |
| Quectel EG912Y-GL | LTE Cat 12 | 300+ Mbps | USB | 500-800 mA | Multi-carrier, eSIM |
| Sierra Wireless RV55 | LTE/5G | Variable | USB | Variable | Industrial grade |

### 9.3 SIM Interface Signals

| Signal | Description |
|--------|-------------|
| `SIM_CLK` | SIM card clock |
| `SIM_DATA` | SIM card data |
| `SIM_RST` | SIM card reset |
| `SIM_VCC` | SIM card power (1.8V or 3.0V selectable) |
| `SIM_DET` | SIM card present detect |

### 9.4 Configuration

```toml
[connectivity.cellular]
enabled = false
sim_type = "nano_sim"            # "nano_sim" | "esim"
apn = "carrier.apn"
fallback_to_wifi = true          # Prefer WiFi when available
always_on_cellular = false       # Keep cellular active even with WiFi
alert_via_cellular = true        # Send critical alerts via cellular
stream_via_cellular = false      # Stream video over cellular (data cost)
```

---

## 10. Power Sources and Management

### 10.1 Power Sources

AEGIS-MESH is source-agnostic:

| Source | Typical Use Case |
|---|---|
| **PoE 802.3af/at** | LiDAR nodes, Identity nodes, Edge controller |
| **USB-C (5V)** | Development, always-on nodes |
| **DC Power (12-24V)** | Edge controller, outdoor nodes |
| **Solar + Battery** | Outdoor perimeter nodes |

### 10.2 Power Domains

| Domain | Voltage | Source | Usage |
|---|---|---|---|
| `V_MAIN` | 5-12V | PoE, USB-C, DC | Raw input |
| `V_MCU` | 3.3V | Regulated from V_MAIN | MCU and sensors |
| `V_RADIO` | 3.3V | Regulated from V_MAIN | Mesh radio |
| `V_SENSOR` | 5V or 12V | Regulated from V_MAIN | LiDAR, mmWave, cameras |
| `V_CAM` | 3.3V or 2.8V | Camera power rail | Camera modules |

### 10.3 Power Mode Summary

| Node Class | Sleep (mW) | Active (mW) | Peak (mW) |
|---|---|---|---|
| Always-On (Variant A) | 10-50 | 100-300 | 500 |
| Always-On (Variant B-D) | 50-150 | 500-1000 | 1500 |
| LiDAR Node (Variant A/B) | 200-500 | 2000-5000 | 8000 |
| LiDAR Node (Variant C) | 300-600 | 3000-8000 | 12000 |
| LiDAR Node (Variant D, 360°) | 400-1000 | 6000-14000 | 25000 |
| Identity Node | 50-100 | 500-2000 | 3000 |
| Edge Controller (SBC) | N/A | 5000-10000 | 15000 |
| Edge Controller (NPU) | N/A | 10000-20000 | 30000 |

---

## 11. Mesh Network Architecture

### 11.1 Physical Layer Options

| Physical Layer | Bandwidth | Latency | Power | Use Case |
|----------------|-----------|---------|-------|----------|
| **WiFi Mesh** | 50-300+ Mbps | 5-20 ms | 300-500 mW | Volume nodes, high bandwidth |
| **Thread / Matter** | 250 Kbps-1 Mbps | 10-50 ms | 50-150 mW | Choke-point nodes, low power |
| **BLE 5.x** | 1-2 Mbps | 10-30 ms | 50-100 mW | Configuration, fallback |
| **PoE Ethernet** | 100-1000 Mbps | < 5 ms | N/A (wired power) | LiDAR nodes, maximum reliability |

### 11.2 Mesh Protocol Features

- **Message signing:** HMAC-SHA256 on all messages
- **Sequence numbers:** Replay protection
- **Heartbeat:** 100 ms intervals; 3 missed = node flagged Offline
- **Time synchronization:** Edge controller as time master; sub-ms sync across nodes

### 11.3 Mesh Bandwidth Budget

| Node Type | Typical Bandwidth | Recommended Transport |
|-----------|-------------------|----------------------|
| Always-On A (PIR) | < 10 Kbps | BLE/Thread |
| Always-On B-D | 50-200 Kbps | WiFi/Thread |
| LiDAR A/B (no camera) | 1-5 Mbps | PoE/WiFi |
| LiDAR C (camera) | 5-20 Mbps | PoE |
| LiDAR D (360° camera) | 10-50 Mbps | PoE required |
| Identity Node | 1-10 Mbps | WiFi/PoE |

---

## 12. 360° LiDAR Node Architecture

### 12.1 Camera Coverage Configurations

| Config | Cameras | Angular Spacing | FoV Needed | PoE Bandwidth (VGA H.264) |
|--------|---------|-----------------|------------|---------------------------|
| Minimal | 4 | 90° | ≥ 100° | 4-8 Mbps |
| Standard | 6 | 60° | ≥ 70° | 6-12 Mbps |
| Dense | 8 | 45° | ≥ 55° | 8-16 Mbps |
| Ultra | 12 | 30° | ≥ 40° | 12-24 Mbps |

### 12.2 Wired Internal Camera Bus

**Recommended architecture:** All cameras connect via internal MIPI bus to vision processor, then single encoded stream to edge controller via PoE.

```
4-8 Cameras → MIPI CSI-2 (internal) → Vision Processor on Node
                                              │
                                              ├── Stitch on-node → Single 360° stream
                                              └── Encode (H.264/H.265)
                                                      │
                                                      ▼
                                                 PoE to Edge Controller
                                                      │
                                                      ▼
                                              Edge Controller → WiFi → App
```

### 12.3 FSYNC Hardware Synchronization

| Requirement | Specification |
|-------------|---------------|
| FSYNC source | Vision processor GPIO |
| Distribution | Single trace to all camera FSYNC pins |
| Trace matching | < 10 ns skew between any two cameras |
| Trigger | All cameras capture simultaneously |

### 12.4 Power Requirements

| Component | Power |
|-----------|-------|
| 4-8 cameras | 1-2 W |
| Vision processor | 1-2 W |
| LiDAR | 5-10 W |
| Event camera | 0.2-0.5 W |
| mmWave radar (optional) | 0.5-1 W |
| **Total active** | **8-15 W** |

**Requires PoE 802.3at (25.5 W) for Variant D.**

---

## 13. Identity Node — Privacy Controls

### 13.1 Hardware Kill Switch (Mandatory)

**All Identity Nodes require a hardware kill switch.** This is not optional or software-configurable.

**Implementation:**
- Physical switch on `V_CAM` rail
- PCB includes MOSFET + switch footprint
- When DISABLED: camera physically unpowered
- Firmware reads `CAM_HW_SW_GPIO` at boot; no override possible

### 13.2 Data Handling Configuration

```toml
[identity_node.data]
mode = "metadata_only"          # "metadata_only" | "local_storage" | "app_streaming" | "full_recording"
store_raw = false
storage_target = "sd_card"
retention_days = 90
enable_streaming = false
stream_quality = "high"
```

### 13.3 Privacy Controls Summary

| Control | Mandatory/Optional | Implementation |
|---------|-------------------|----------------|
| Hardware kill switch | **Mandatory** | PCB-mounted physical switch |
| Software disable | Optional | Firmware configuration |
| Activity scheduling | Optional | Time-based camera disable |
| Detection-triggered | Optional | Camera activates on event |
| Continuous recording | Optional | Full capture per config |

---

## 14. Storage Architecture

### 14.1 Node-Level Storage

| Storage Type | Capacity | Interface | Use Case |
|--------------|----------|-----------|----------|
| microSD | Up to 2 TB | SPI/SDIO | Raw video, point clouds, audio |
| QSPI NOR Flash | 8-128 MB | QSPI | Metadata, logs, model weights |
| Internal MCU Flash | 512 KB - 2 MB | Internal | Firmware, critical config |

### 14.2 Edge Controller Storage

| Storage Type | Capacity | Interface | Use Case |
|--------------|----------|-----------|----------|
| microSD | Up to 2 TB | SDIO | Aggregated recordings, SLAM maps |
| NVMe SSD | 128 GB - 4 TB | PCIe | High-capacity deployments |
| USB Storage | Variable | USB 3.x | Backup, archive |
| NAS | Variable | Ethernet | Network-attached storage |

### 14.3 Configuration

```toml
[data]
store_raw_video = false
store_raw_audio = false
store_raw_sensor_data = false
storage_target = "edge_controller"   # "edge_controller" | "node_local" | "nas"
retention_days = 30
continuous_recording = false
recording_trigger = "on_detection"   # "always" | "on_detection" | "on_alert" | "manual"
max_storage_mb = 0                   # 0 = unlimited
include_integrity_chain = true
```

---

## 15. RF Coexistence

### 15.1 Frequency Bands

| Radio | Frequency | Conflict Risk |
|-------|-----------|---------------|
| WiFi | 2.4 GHz and 5 GHz | BLE at 2.4 GHz |
| BLE | 2.4 GHz ISM | WiFi at 2.4 GHz |
| Thread (802.15.4) | 2.4 GHz ISM | WiFi and BLE at 2.4 GHz |
| mmWave radar | 60 GHz | None (separate band) |
| Cellular LTE | 700 MHz - 2.6 GHz | Possible adjacent to 2.4 GHz |
| Cellular 5G | 3.5 GHz and 24-47 GHz | None at residential scale |

### 15.2 Coexistence Strategies

**Strategy 1: WiFi Prefers 5 GHz**
- Most residential routers support 5 GHz
- BLE/Thread operate at 2.4 GHz
- No frequency overlap

**Strategy 2: Antenna Placement (Edge Controller)**
- WiFi antenna on exterior of enclosure
- BLE/Thread antenna for mesh (if applicable)
- Cellular antenna on exterior

**Strategy 3: Time Division (If 2.4 GHz WiFi Required)**
- Coordinate WiFi and BLE/Thread timing
- BLE connection events during WiFi idle periods

### 15.3 Configuration

```toml
[rf_coexistence]
wifi_prefer_5ghz = true
```

---

## 16. Thermal Management

### 16.1 Sensor Nodes

Most sensor nodes are thermally benign (< 1W active):
- Always-On nodes: Passive cooling sufficient
- LiDAR nodes: May require thermal pad under LiDAR module
- 360° LiDAR node: Requires thermal vias and potential heatsink

### 16.2 Edge Controller

| Variant | Typical Power | Cooling Strategy |
|---------|---------------|------------------|
| SBC (Var A) | 5-10W | Passive + enclosure vents |
| NPU (Var B) | 10-20W | Heatsink + vents or small fan |
| x86 (Var C) | 15-35W | Active fan required |

**Thermal monitoring:**
```toml
[thermal]
monitoring_enabled = true
warning_threshold_c = 60
throttle_threshold_c = 70
shutdown_threshold_c = 85
```

---

## 17. Debug & Configuration Interface

### 17.1 Standard Debug Header

**10-pin ARM Cortex Debug Connector (0.05" pitch):**

| Pin | Signal | Description |
|---|---|---|
| 1 | VTref | Target voltage reference |
| 2-4 | GND | Ground |
| 5 | SWDIO | Serial Wire Debug I/O |
| 6 | SWCLK | Serial Wire Clock |
| 7 | SWO | Serial Wire Output (optional) |
| 8 | NC | No Connect |
| 9 | RST | MCU Reset |
| 10 | NC | No Connect |

### 17.2 UART Console

| Signal | Description |
|--------|-------------|
| UART_TX | Debug console transmit (115200 baud) |
| UART_RX | Debug console receive |

### 17.3 Production Debug Disable

After production, debug can be permanently disabled via fuse/OTP to prevent firmware extraction.

---

## 18. Camera Privacy Controls

### 18.1 Identity Node (Mandatory)

- Hardware kill switch required
- No firmware override possible
- Switch physically cuts camera power

### 18.2 Camera-Capable Nodes (Optional)

For Always-On Variant D, LiDAR Variants C/D:

| Control | Implementation |
|---------|----------------|
| Hardware switch | Optional, user-installed |
| Software disable | Firmware configuration |
| Activity scheduling | Time-based disable |
| Detection-triggered | Camera activates on event |

```toml
[privacy.camera]
hardware_switch_installed = false
software_control_enabled = true
default_state = "disabled"
schedule = []
trigger_on_detection = false
```

---

## 19. Test Pad Standard

### 19.1 Sensor Node Test Pads

| Pad | Signal | Description |
|---|---|---|
| 1 | VCC_BATT | Battery/input positive |
| 2 | GND | Ground |
| 3 | CHRG_IN+ | Charging input positive |
| 4 | CHRG_IN- | Charging input negative |
| 5 | MESH_RF_TEST | Mesh radio test point |
| 6 | MCU_BOOT | Boot mode select |
| 7 | MCU_RESET | MCU reset |
| 8 | CAM_HW_SW_TEST | Camera switch state |
| 9 | LED_TEST | Status LED test |
| 10 | SWD_CLK | SWD clock |
| 11 | SWD_DATA | SWD data |
| 12 | UART_TX | Debug UART TX |

### 19.2 Edge Controller Additional Test Pads

| Pad | Signal | Description |
|---|---|---|
| 13 | CELLULAR_PWR | Cellular module power rail |
| 14 | SIM_VCC | SIM card power |
| 15 | WIFI_RST | WiFi module reset |
| 16 | CELLULAR_STATUS | Cellular status LED |
| 17 | USB_D+ | USB data plus |
| 18 | USB_D- | USB data minus |

---

## 20. Data Handling Configuration

### 20.1 Complete Configuration Reference

```toml
# aegis-mesh.toml

[data]
store_raw_video = false
store_raw_audio = false
store_raw_sensor_data = false
storage_target = "edge_controller"   # "edge_controller" | "node_local" | "nas"
retention_days = 30
continuous_recording = false
recording_trigger = "on_detection"
max_storage_mb = 0
include_integrity_chain = true

[connectivity]
wifi_enabled = true
wifi_prefer_5ghz = true

[connectivity.cellular]
enabled = false
sim_type = "nano_sim"
apn = ""
fallback_to_wifi = true
always_on_cellular = false
alert_via_cellular = true
stream_via_cellular = false

[companion_app]
enabled = true
api_port = 8080
media_port = 9090
enable_remote_access = false
remote_access_token = ""

[privacy.camera]
hardware_switch_installed = false
software_control_enabled = true
default_state = "disabled"

[thermal]
monitoring_enabled = true
warning_threshold_c = 60
throttle_threshold_c = 70
shutdown_threshold_c = 85
```

---

## 21. Summary

The AEGIS-MESH hardware configuration is designed to be:

- **Flexible:** Supports any sensor, any power source, any MCU family
- **Standardized:** Common 20-pin interface for mesh, power, and debug
- **Privacy-Respecting:** Identity nodes require hardware kill switches
- **Connectivity-Centralized:** Edge controller as sole external gateway
- **Research-Oriented:** No imposed limits on resolution, storage, or compute

This document defines connectivity, power infrastructure, and architecture constraints. Sensor specifications and capabilities are configuration choices, not constraints.
