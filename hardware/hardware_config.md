# Hardware Configuration Reference — AEGIS-MESH

---

## Node Interface Standard

All AEGIS-MESH nodes expose a common interface standard for firmware compatibility across hardware variants.

### Standard Interface Header (20-pin, 2.54 mm pitch)

| Pin | Signal | Direction | Description |
|---|---|---|---|
| 1 | VCC_3V3 | Power | 3.3 V regulated supply to node |
| 2 | VCC_SENSOR | Power | Sensor power rail (switchable on Identity Node) |
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
| 14 | I2C_SDA | Bidirectional | Environmental sensor I2C data |
| 15 | I2C_SCL | Output | Environmental sensor I2C clock |
| 16 | PIR_INT | Input | PIR interrupt (choke-point nodes) |
| 17 | KILL_SW_GPIO | Input | Kill switch state (Identity Node: HIGH = enabled) |
| 18 | LED_STATUS | Output | Status LED drive |
| 19 | BOOT_SELECT | Input | Firmware boot mode select |
| 20 | RESET | Input (active low) | MCU reset |

### LiDAR Node Additional Interface

| Pin | Signal | Direction | Description |
|---|---|---|---|
| A | LIDAR_ETH_TX+ | Differential | LiDAR Ethernet TX+ |
| B | LIDAR_ETH_TX- | Differential | LiDAR Ethernet TX- |
| C | LIDAR_ETH_RX+ | Differential | LiDAR Ethernet RX+ |
| D | LIDAR_ETH_RX- | Differential | LiDAR Ethernet RX- |
| E | EVENT_CAM_USB_D+ | Differential | Event camera USB D+ |
| F | EVENT_CAM_USB_D- | Differential | Event camera USB D- |

---

## Power Budget Summary (Estimates — All Variants TBD by Test)

| Node Class | Sleep (mW) | Active (mW) | Peak (mW) |
|---|---|---|---|
| Always-On (radar + mic) | 50–150 | 500–1000 | 1500 |
| Choke-Point (PIR only) | 10–50 | 100–300 | 500 |
| LiDAR Node (event trigger) | 200–500 | 2000–5000 | 8000 |
| LiDAR Node (continuous) | — | 8000–15000 | 15000 |
| Identity Node (dormant) | 50–100 | 500–2000 | 3000 |

*All values are estimates pending test measurement with selected hardware variants. Use these only for power supply sizing exercises.*

---

## Mesh Protocol Compatibility

All nodes must run firmware from `firmware/src/bin/<node_name>.rs` compiled for the correct MCU target. The mesh protocol (documented in `docs/protocol/protocol_spec.md`) enforces:
- Message signing (HMAC-SHA256, key established at pairing)
- Sequence numbers (replay protection)
- Heartbeat messages at 10 Hz (node health monitoring)

Nodes that do not respond to heartbeat within 3 missed periods are flagged as `Offline` by the edge controller.
