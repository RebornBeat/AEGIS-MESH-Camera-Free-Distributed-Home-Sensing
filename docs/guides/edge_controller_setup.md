# Edge Controller Setup Guide — AEGIS-MESH

**Project:** AEGIS-MESH
**Component:** Edge Controller (Primary Compute and Network Hub)
**Status:** Reference Guide

---

## 1. Overview

The Edge Controller is the central compute and network hub of the AEGIS-MESH system. It is analogous to the Belt Node in SENTINEL-WEAR — it is the **sole external gateway** for all network connectivity and the primary processing unit for sensor fusion, SLAM, recording management, and the companion app API.

### Critical Architectural Principle

**Only the Edge Controller has WiFi, Cellular, and direct BLE connection to external networks.**

**All other AEGIS-MESH nodes (Always-On, LiDAR, Identity) communicate exclusively via mesh radio (BLE 5.x / Thread / PoE Ethernet) to the edge controller.** They have no WiFi, no cellular, and no direct connection to the companion app or external networks.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                     AEGIS-MESH Connectivity Model                           │
│                                                                              │
│  Always-On Nodes ──┐                                                         │
│  LiDAR Nodes───────┤── Mesh Radio ──► Edge Controller ──► WiFi ──► App      │
│  Identity Nodes────┘  (BLE/Thread/PoE)       │                               │
│                                              ├──► Cellular ──► Remote App   │
│                                              └──► BLE direct ──► Local App │
│                                                                              │
│  ⚠️  EDGE CONTROLLER IS THE ONLY NODE WITH WiFi / Cellular / BLE (external) │
│  ⚠️  ALL OTHER NODES = MESH RADIO ONLY (to edge controller, nothing external)│
└─────────────────────────────────────────────────────────────────────────────┘
```

### Why This Architecture?

| Constraint | WiFi on Sensor Node | BLE on Sensor Node | PoE on Sensor Node |
|------------|---------------------|-------------------|---------------------|
| **Power** | 300-1000+ mW active | 5-15 mW active | Powered by Ethernet |
| **Heat** | Requires thermal mgmt | Minimal | Minimal |
| **Cost** | WiFi module + antenna | Integrated or low-cost | Ethernet PHY + connector |
| **Placement flexibility** | Requires WiFi coverage | Works anywhere | Requires Ethernet cable |
| **Determinism** | Variable latency | Consistent | Excellent |

**Conclusion:** Sensor nodes use BLE/Thread/PoE for reliable, deterministic communication to the edge controller. WiFi and cellular are concentrated at the edge controller where power and thermal constraints are relaxed.

---

## 2. Edge Controller Role

The Edge Controller performs all central functions:

| Function | Description |
|----------|-------------|
| **Mesh Hub** | Receives data from all sensor nodes via BLE/Thread/PoE |
| **Fusion Center** | Multi-node detection fusion using CI + JPDA algorithms |
| **PentaTrack Engine** | Predictive tracking for all detected entities |
| **SLAM Processor** | Dense 3D reconstruction (LiDAR + camera nodes) |
| **Recording Manager** | Aggregates recordings from all nodes, manages retention |
| **API Server** | REST/WebSocket server for companion app |
| **Media Server** | RTSP/WebSocket streaming for video/audio |
| **WiFi Gateway** | Primary connection to local network and companion app |
| **Cellular Gateway** | Optional remote access via LTE/5G |
| **BLE Gateway** | Direct companion app connection (fallback) |
| **Time Master** | Clock synchronization for all mesh nodes |

---

## 3. Hardware Variants

### 3.1 Variant A — SBC Class (Raspberry Pi 4/5)

**Compute Platform:**
- Raspberry Pi 4 (Cortex-A72, quad-core, 1.5-1.8 GHz) or Raspberry Pi 5 (Cortex-A76, quad-core, 2.4 GHz)
- RAM: 4-8 GB
- Storage: microSD or NVMe (via HAT or USB)
- Built-in: WiFi 802.11ac, Bluetooth 5.0, Gigabit Ethernet

**Expansion:**
- PoE HAT for power + data from PoE switch (aggregates multiple LiDAR nodes)
- USB cellular module (optional)
- External storage (USB SSD/NAS)

**Capability:**
- Sparse tracking (PentaTrack): ✅ Excellent
- Dense SLAM: ⚠️ Limited (CPU-based, slower)
- Camera streams: 2-4 simultaneous
- 360° processing: ⚠️ Challenging without GPU

**Suitable for:**
- Residential deployments
- Proof-of-concept
- Budget-conscious installations

**Power:**
- AC adapter (5V, 3A) or PoE (via HAT)
- No battery (always plugged in)

### 3.2 Variant B — ARM with NPU (Jetson Nano / Orin Class)

**Compute Platform:**
- NVIDIA Jetson Nano (Cortex-A57 + 128-core Maxwell GPU) or Jetson Orin Nano (Cortex-A78 + Ampere GPU + NPU)
- RAM: 4-8 GB (Nano), 8-16 GB (Orin)
- Storage: NVMe SSD
- Built-in: Gigabit Ethernet, optional WiFi module

**Expansion:**
- PoE injector or separate PoE switch
- Cellular module via USB or M.2
- NAS for extended storage

**Capability:**
- Sparse tracking: ✅ Excellent
- Dense SLAM: ✅ Excellent (GPU-accelerated)
- Camera streams: 8-12 simultaneous
- 360° processing: ✅ Good (GPU handles stitching)

**Suitable for:**
- Professional installations
- Multi-room deployments
- Dense SLAM requirements

**Power:**
- DC adapter (5V, 4A for Nano; higher for Orin)
- No battery (always plugged in)
- Active cooling (fan included in dev kits)

### 3.3 Variant C — x86 Mini PC

**Compute Platform:**
- Intel NUC, Intel Compute Element, or custom mini PC
- CPU: Intel Core i3/i5/i7 or AMD Ryzen
- RAM: 8-32 GB
- Storage: NVMe SSD (256 GB - 2 TB)
- Built-in: WiFi 6, Gigabit Ethernet, Bluetooth

**Expansion:**
- PoE switch for LiDAR nodes
- Cellular module via USB
- External GPU (optional, via Thunderbolt)
- NAS for extended storage

**Capability:**
- Sparse tracking: ✅ Excellent
- Dense SLAM: ✅ Excellent (CPU or optional GPU)
- Camera streams: 16+ simultaneous
- 360° processing: ✅ Excellent

**Suitable for:**
- Enterprise deployments
- Large-scale installations
- Maximum flexibility and expansion

**Power:**
- AC adapter (typically 65-120W)
- No battery (always plugged in)
- Active cooling (standard PC cooling)

### 3.4 Variant Selection Matrix

| Criterion | Variant A (SBC) | Variant B (NPU) | Variant C (x86) |
|-----------|-----------------|-----------------|------------------|
| **Cost** | $50-150 | $200-600 | $400-1500 |
| **Power consumption** | 5-10 W | 10-25 W | 15-65 W |
| **SLAM performance** | Limited | Excellent | Excellent |
| **Camera capacity** | 2-4 | 8-12 | 16+ |
| **360° processing** | Challenging | Good | Excellent |
| **Expansion flexibility** | Limited | Moderate | High |
| **Setup complexity** | Low | Moderate | Moderate |
| **Suitable deployment** | Residential | Professional | Enterprise |

---

## 4. External Connectivity Architecture

### 4.1 WiFi Interface

**Purpose:** Primary connection to local network and companion app

**Specifications:**
- Standard: 802.11 a/b/g/n/ac/ax (depending on platform)
- Preferred band: **5 GHz** (avoids interference with BLE at 2.4 GHz)
- Role: API server, media streaming, SLAM data transfer

**Configuration:**

```toml
[connectivity.wifi]
enabled = true
ssid = "YourHomeNetwork"
password = ""                     # Set at runtime for security
prefer_5ghz = true                # Strongly recommended
country_code = "US"               # Set for regulatory compliance
```

### 4.2 Cellular / SIM Interface

**Purpose:** Remote access when WiFi unavailable, emergency alerts, cellular-primary deployments

**Hardware:**
- nano-SIM card slot (4FF) or eSIM footprint
- Cellular module via UART (Cat 1/4) or USB 3.x (5G)

**Supported Cellular Modules:**

| Module | Technology | Speed | Interface | Use Case |
|--------|------------|-------|-----------|----------|
| Quectel EC21 | LTE Cat 1 | 5-10 Mbps | UART | Alerts, metadata only |
| Quectel EC25 | LTE Cat 4 | 50-150 Mbps | UART/USB | Compressed video clips |
| Quectel RM502Q-AE | 5G Sub-6GHz | 100+ Mbps | USB 3.x | Live video, SLAM data |
| Quectel EG912Y-GL | LTE Cat 12 | 300+ Mbps | USB | Multi-carrier, eSIM |
| Sierra Wireless RV55 | LTE/5G | Variable | USB | Industrial grade |

**Configuration:**

```toml
[connectivity.cellular]
enabled = false
sim_type = "nano_sim"            # "nano_sim" | "esim"
apn = "your.carrier.apn"
fallback_to_wifi = true          # Prefer WiFi when available
always_on_cellular = false       # Keep cellular active even with WiFi
alert_via_cellular = true        # Send critical alerts via cellular
stream_via_cellular = false      # Stream video over cellular (data cost consideration)
emergency_contact = ""           # Phone/endpoint for emergency alerts
```

**Power Impact of Cellular:**

| Module | Active Current | Notes |
|--------|----------------|-------|
| EC21 (LTE Cat 1) | 150-300 mA | Lowest power cellular |
| EC25 (LTE Cat 4) | 400-600 mA | Moderate power |
| RM502Q-AE (5G) | 800-1200 mA | Highest power, highest bandwidth |

**Cellular is present ONLY on the edge controller. No sensor node has cellular capability.**

### 4.3 Bluetooth Interface

**Purpose:** Direct companion app connection when WiFi unavailable

**Capabilities:**
- Alerts, metadata, configuration
- Not suitable for video streaming or SLAM data transfer
- Fallback mode for local access

**Configuration:**

```toml
[connectivity.bluetooth]
enabled = true
advertising_name = "aegis-mesh-edge"
pairing_mode = "secure"          # "open" | "secure"
```

### 4.4 Ethernet Interface

**Purpose:**
- Primary mesh transport for LiDAR nodes (PoE)
- Wired connection to local network
- Higher reliability than WiFi

**For LiDAR Nodes:** PoE (Power over Ethernet) provides both power and data transport:
- 802.3af (15.4 W) for standard LiDAR nodes
- 802.3at (25.5 W) for LiDAR Variant D (360° camera array)

**Configuration:**

```toml
[connectivity.ethernet]
enabled = true
use_poe_aggregator = true        # Use PoE switch for LiDAR nodes
static_ip = ""                   # Leave empty for DHCP
```

### 4.5 USB Interface

**Purpose:**
- PC companion app connection
- Firmware updates via DFU
- External storage
- Optional cellular module connection
- Debug access

---

## 5. Mesh Network Architecture

### 5.1 Mesh Radio Options

| Transport | Use Case | Nodes Supported |
|-----------|----------|------------------|
| BLE 5.x | Low-power always-on nodes | 10-20 nodes typical |
| Thread | Low-power nodes with better range | 10-50 nodes |
| PoE Ethernet | High-bandwidth LiDAR nodes | Unlimited (via PoE switch) |

### 5.2 Bandwidth Hierarchy

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         BANDWIDTH HIERARCHY                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│  BLE 5.x / Thread (500 Kbps - 2 Mbps practical)                             │
│  → Control plane (configuration, commands, health)                          │
│  → Metadata (detections, classification results)                            │
│  → Low-bandwidth sensor data                                                 │
│  → Always-on layer communication                                            │
├─────────────────────────────────────────────────────────────────────────────┤
│  PoE Ethernet (100 Mbps - 1 Gbps)                                            │
│  → LiDAR point cloud streams                                                 │
│  → High-resolution camera streams                                            │
│  → SLAM data transfer                                                        │
│  → Dense world model synchronization                                         │
├─────────────────────────────────────────────────────────────────────────────┤
│  WiFi (50-300+ Mbps) — EDGE CONTROLLER ONLY                                 │
│  → High-bandwidth continuous (360° live streaming)                           │
│  → Multiple simultaneous camera streams                                      │
│  → SLAM world model download                                                 │
│  → Primary path to companion app                                             │
├─────────────────────────────────────────────────────────────────────────────┤
│  Cellular (Variable by module) — EDGE CONTROLLER ONLY                       │
│  → LTE Cat 1: 5-10 Mbps — alerts, metadata                                  │
│  → LTE Cat 4: 50-150 Mbps — compressed video                                │
│  → 5G: 100-1000+ Mbps — live streaming, SLAM relay                           │
│  → Remote access when WiFi unavailable                                       │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 5.3 RF Coexistence

**Frequency bands in use:**

| Radio | Frequency Band | Conflict Risk |
|-------|---------------|---------------|
| BLE | 2.4 GHz ISM | WiFi 2.4 GHz |
| WiFi | 2.4 GHz and 5 GHz | BLE (2.4 GHz only) |
| Cellular LTE | 700 MHz - 2.6 GHz | Possible adjacent to 2.4 GHz |
| Cellular 5G Sub-6 | 3.5 GHz | Minimal |

**Coexistence strategies:**

1. **WiFi prefers 5 GHz band** — Eliminates BLE/WiFi interference
2. **Antenna separation** — Cellular/WiFi antennas physically separated
3. **Coexistence firmware** — When 2.4 GHz WiFi required, coordinate timing with BLE

```toml
[rf_coexistence]
wifi_prefer_5ghz = true          # Strongly recommended
ble_wifi_timeshare = false       # Set true only if 2.4 GHz WiFi required
```

---

## 6. Power Architecture

### 6.1 Power Sources

| Source | Variants | Notes |
|--------|----------|-------|
| AC adapter | All | Primary power source |
| PoE (via HAT) | Variant A (Raspberry Pi) | Power + data from Ethernet |
| UPS integration | All | Optional for power failure resilience |

### 6.2 Power Domains

| Domain | Voltage | Usage |
|--------|---------|-------|
| `V_MAIN` | 5V / 12V | Raw input |
| `V_MCU` | 3.3V | MCU, logic |
| `V_RADIO` | 3.3V | WiFi, BLE |
| `V_CELLULAR` | 3.3V / 4.2V | Cellular module |

### 6.3 Power Budgets

| Variant | Idle | Active | Peak |
|---------|------|--------|------|
| A (SBC) | 2-3 W | 5-8 W | 10 W |
| B (NPU) | 5-8 W | 15-25 W | 35 W |
| C (x86) | 10-15 W | 30-50 W | 65 W |

**Add cellular module power:**
- LTE Cat 1: +0.5-1.5 W
- LTE Cat 4: +2-4 W
- 5G: +4-10 W

### 6.4 Thermal Management

**Edge controller thermal requirements:**

| Variant | Thermal Strategy | Monitoring |
|---------|------------------|------------|
| A (SBC) | Passive heatsink + optional fan | CPU temperature |
| B (NPU) | Fan required | GPU + CPU temperature |
| C (x86) | Standard PC cooling | Multiple sensors |

**Thermal throttling:**

```toml
[thermal]
monitoring_enabled = true
warning_threshold_c = 70
throttle_threshold_c = 80
shutdown_threshold_c = 90
throttle_action = "reduce_slam_rate"
```

---

## 7. Storage Architecture

### 7.1 Storage Hierarchy

| Storage Type | Capacity | Interface | Use Case |
|--------------|----------|-----------|----------|
| microSD card | 32 GB - 2 TB | SPI/SDIO | Boot, recordings, SLAM maps |
| NVMe SSD | 128 GB - 2 TB | PCIe | Production storage, SLAM |
| USB storage | Variable | USB 3.x | Backup, archive, NAS |
| Network storage | Unlimited | Ethernet | Extended storage, backup |

### 7.2 Storage Configuration

```toml
[data]
# What to store locally
store_raw_video = true
store_raw_audio = false
store_raw_sensor_data = false
store_slam_maps = true

# Storage target
storage_target = "nvme"          # "sd_card" | "nvme" | "usb" | "network_path"
network_path = "/mnt/nas"

# Retention
retention_days = 30              # 0 = keep forever
max_storage_mb = 0               # 0 = unlimited

# Recording behavior
continuous_recording = false
recording_trigger = "on_detection"
recording_grace_period_s = 5     # Keep 5 seconds before detection event
include_integrity_chain = true
```

### 7.3 Recording Sources

The edge controller aggregates recordings from:

| Source | Recording Type | Network Path |
|--------|---------------|--------------|
| Identity nodes | Video, audio | Mesh → edge → SD/NVMe |
| LiDAR nodes | Point clouds, video | PoE → edge → SD/NVMe |
| Always-On nodes | Audio, radar data | BLE → edge → SD/NVMe |

---

## 8. Software Installation

### 8.1 Operating System

**Variant A (Raspberry Pi):**
- Raspberry Pi OS Lite (64-bit) recommended
- Ubuntu Server 22.04 LTS alternative

**Variant B (Jetson):**
- NVIDIA JetPack SDK (Ubuntu-based)
- Includes CUDA, TensorRT, GPU drivers

**Variant C (x86):**
- Ubuntu Server 22.04 LTS
- Fedora Server alternative

### 8.2 Software Stack

```bash
# Install Rust
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
source ~/.cargo/env

# Clone repository
git clone https://github.com/ungatedminds/aegis-mesh
cd aegis-mesh

# Build edge controller binary
cargo build --release --bin aegis-edge-controller

# Install as systemd service
sudo cp target/release/aegis-edge-controller /usr/local/bin/
sudo cp deployment/aegis-edge-controller.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable aegis-edge-controller
sudo systemctl start aegis-edge-controller
```

### 8.3 Configuration File

Create `/etc/aegis-mesh/config.toml`:

```toml
# Edge Controller Configuration

[deployment]
building_floor_count = 1
floor_height_m = 2.7
total_area_m2 = 150.0

[policy]
default_mode = "PrivacyFirst"
auto_away_after_minutes = 30
alert_cooldown_s = 60.0

[mesh_network]
protocol = "ble"                 # "ble" | "thread" | "hybrid"
edge_controller_address = "0.0.0.0:7171"
heartbeat_interval_ms = 100
message_signing = true

[data]
store_raw_video = true
store_raw_audio = false
store_raw_sensor_data = false
storage_target = "nvme"
retention_days = 30
max_storage_mb = 0
include_integrity_chain = true

[connectivity.wifi]
enabled = true
ssid = "YourHomeNetwork"
prefer_5ghz = true

[connectivity.cellular]
enabled = false
sim_type = "nano_sim"
apn = ""
fallback_to_wifi = true
alert_via_cellular = true

[companion_app]
enabled = true
api_port = 8080
media_port = 9090
enable_remote_access = false
remote_access_token = ""

[thermal]
monitoring_enabled = true
warning_threshold_c = 70
throttle_threshold_c = 80
```

---

## 9. Network Setup

### 9.1 WiFi Configuration

```bash
# Raspberry Pi OS: Use raspi-config
sudo raspi-config
# Select: System Options → Wireless LAN

# Ubuntu: Netplan configuration
sudo nano /etc/netplan/50-cloud-init.yaml
```

```yaml
network:
  version: 2
  wifis:
    wlan0:
      dhcp4: true
      access-points:
        "YourHomeNetwork":
          password: "your-password"
```

```bash
sudo netplan apply
```

### 9.2 Static IP (Optional)

```yaml
network:
  version: 2
  ethernets:
    eth0:
      addresses:
        - 192.168.1.100/24
      routes:
        - to: default
          via: 192.168.1.1
      nameservers:
        addresses: [8.8.8.8, 8.8.4.4]
```

### 9.3 Cellular Configuration

For USB cellular modules:

```bash
# Install PPP daemon
sudo apt install ppp

# Configure connection
sudo nano /etc/ppp/peers/cellular
```

```
/dev/ttyUSB0
115200
noauth
defaultroute
usepeerdns
persist
nodetach
user ""
password ""
connect "/usr/sbin/chat -v -f /etc/chatscripts/cellular"
```

```bash
# Test connection
sudo pon cellular
```

---

## 10. Node Discovery and Pairing

### 10.1 Automatic Discovery

When sensor nodes boot, they broadcast on the mesh network. The edge controller automatically discovers them:

```bash
# Check discovered nodes
curl http://localhost:8080/api/nodes
```

### 10.2 Manual Node Registration

```bash
# Register a node manually
curl -X POST http://localhost:8080/api/nodes \
  -H "Content-Type: application/json" \
  -d '{
    "id": "living_room_ceiling",
    "class": "LidarNode",
    "position_m": [4.5, 3.2, 2.8]
  }'
```

### 10.3 Node Authentication

All nodes use HMAC-SHA256 message signing. Keys are exchanged during initial pairing:

1. Node enters pairing mode (hardware button or firmware command)
2. Edge controller generates unique key for node
3. Key transmitted securely to node via mesh
4. Node stores key in secure flash
5. All subsequent messages signed with node's key

---

## 11. Calibration

### 11.1 Walk-Through Calibration

```bash
# Start calibration mode
curl -X POST http://localhost:8080/api/calibration/start

# Walk through home at normal pace (30-60 seconds)
# System trilaterates node positions

# Check calibration status
curl http://localhost:8080/api/calibration/status

# When confidence > 0.85, complete calibration
curl -X POST http://localhost:8080/api/calibration/complete
```

### 11.2 Manual Calibration

For precise installations:

```bash
# Manually set node position
curl -X POST http://localhost:8080/api/nodes/living_room_ceiling/position \
  -H "Content-Type: application/json" \
  -d '{"position_m": [4.5, 3.2, 2.8]}'
```

---

## 12. Companion App Connection

### 12.1 Local Network Access

The companion app discovers the edge controller via mDNS:

- Hostname: `aegis-mesh.local`
- API port: 8080
- Media port: 9090

**Connection URL:** `http://aegis-mesh.local:8080`

### 12.2 Remote Access (Cellular)

When cellular is enabled:

1. Edge controller obtains cellular IP
2. Register with dynamic DNS service (optional)
3. Configure companion app with remote endpoint
4. Use remote access token for authentication

```toml
[companion_app]
enable_remote_access = true
remote_access_token = "your-secure-token"
```

### 12.3 API Authentication

All API requests require bearer token:

```bash
curl http://aegis-mesh.local:8080/api/nodes \
  -H "Authorization: Bearer your-token"
```

---

## 13. 360° Camera Integration (LiDAR Variant D)

### 13.1 PoE Network for LiDAR Nodes

For LiDAR Variant D with 360° camera array:

1. Deploy PoE switch (802.3at capable)
2. Connect LiDAR nodes via Ethernet
3. Edge controller connects to PoE switch

```toml
[connectivity.ethernet]
use_poe_aggregator = true
```

### 13.2 Stitching and SLAM Processing

The edge controller handles:
- 360° video stitching from LiDAR Variant D
- SLAM processing from LiDAR + camera data
- 3D world model generation

**Compute requirements for 360° stitching:**
- GPU recommended (Jetson or x86 + GPU)
- Real-time stitching requires ~500 MB GPU memory per node
- SLAM processing requires ~2 GB RAM per node

---

## 14. Testing and Validation

### 14.1 Health Check

```bash
# System health
curl http://localhost:8080/api/health

# Node health
curl http://localhost:8080/api/nodes/health

# Coverage status
curl http://localhost:8080/api/coverage/status
```

### 14.2 Connectivity Test

```bash
# Test WiFi
ping google.com

# Test cellular (if enabled)
ping -I ppp0 google.com

# Test mesh network
curl http://localhost:8080/api/nodes
# All nodes should show "Online" status
```

### 14.3 Recording Test

```bash
# Trigger test recording
curl -X POST http://localhost:8080/api/nodes/identity_entry/record \
  -d '{"duration_s": 5}'

# Check recording
curl http://localhost:8080/api/recordings
```

---

## 15. Troubleshooting

### 15.1 Node Not Discovered

1. Check node power and LED status
2. Verify mesh radio is functional
3. Check edge controller logs: `journalctl -u aegis-edge-controller -f`
4. Verify node is in pairing mode (first boot)

### 15.2 WiFi Connectivity Issues

1. Verify 5 GHz preference is set
2. Check WiFi signal strength
3. Verify country code is set correctly
4. Check for 2.4 GHz interference from other devices

### 15.3 Cellular Not Connecting

1. Verify SIM is inserted correctly
2. Check APN configuration
3. Verify antenna is connected
4. Check cellular module logs: `dmesg | grep ttyUSB`

### 15.4 SLAM Performance Issues

1. Check GPU utilization
2. Reduce SLAM update rate
3. Reduce camera resolution
4. Enable throttling in thermal config

### 15.5 Storage Full

```bash
# Check storage usage
df -h

# List recordings by size
curl http://localhost:8080/api/recordings?sort=size

# Delete old recordings
curl -X DELETE http://localhost:8080/api/recordings/before/2024-01-01
```

---

## 16. Security Configuration

### 16.1 Firewall

```bash
# Allow only necessary ports
sudo ufw allow 8080/tcp   # API
sudo ufw allow 9090/tcp   # Media
sudo ufw allow 22/tcp     # SSH (for administration)
sudo ufw enable
```

### 16.2 TLS Configuration (Recommended for Remote Access)

```bash
# Install certbot
sudo apt install certbot

# Generate certificate (if you have a domain)
sudo certbot certonly --standalone -d your-domain.com

# Configure TLS
cp /etc/letsencrypt/live/your-domain.com/fullchain.pem /etc/aegis-mesh/tls/
cp /etc/letsencrypt/live/your-domain.com/privkey.pem /etc/aegis-mesh/tls/
```

```toml
[companion_app]
tls_enabled = true
tls_cert_path = "/etc/aegis-mesh/tls/fullchain.pem"
tls_key_path = "/etc/aegis-mesh/tls/privkey.pem"
```

### 16.3 Audit Logging

All configuration changes and data access are logged:

```bash
# View audit log
curl http://localhost:8080/api/audit

# Audit log location
tail -f /var/log/aegis-mesh/audit.log
```

---

## 17. Maintenance

### 17.1 Firmware Updates

```bash
# Check for updates
curl http://localhost:8080/api/system/updates

# Apply updates
curl -X POST http://localhost:8080/api/system/updates/apply
```

### 17.2 Backup

```bash
# Export configuration
curl http://localhost:8080/api/config/export > aegis-mesh-config-backup.toml

# Backup recordings
rsync -av /var/lib/aegis-mesh/recordings/ /mnt/backup/recordings/
```

### 17.3 Log Rotation

```bash
# Configure logrotate
sudo nano /etc/logrotate.d/aegis-mesh
```

```
/var/log/aegis-mesh/*.log {
    daily
    rotate 30
    compress
    missingok
    notifempty
}
```

---

## 18. Production Checklist

Before deploying to production:

- [ ] WiFi configured with 5 GHz preference
- [ ] Cellular configured (if remote access needed)
- [ ] All nodes discovered and calibrated
- [ ] Coverage analysis shows no blind spots
- [ ] Storage configured with appropriate retention
- [ ] TLS enabled for remote access
- [ ] Firewall configured
- [ ] Firmware updated to latest version
- [ ] Backup strategy implemented
- [ ] Test recording and playback verified
- [ ] Test alert delivery verified

---

## 19. Example Deployment Configurations

### 19.1 Residential Deployment

```toml
# Single-family home, ~150 m²
# Variant A edge controller (Raspberry Pi)
# 6 sensor nodes

[deployment]
building_floor_count = 1
floor_height_m = 2.7
total_area_m2 = 150.0

[policy]
default_mode = "PrivacyFirst"

[mesh_network]
protocol = "ble"

[data]
storage_target = "sd_card"
retention_days = 14

[connectivity.cellular]
enabled = false
```

### 19.2 Small Business Deployment

```toml
# Office space, ~300 m²
# Variant B edge controller (Jetson)
# 12 sensor nodes + 2 LiDAR nodes

[deployment]
building_floor_count = 1
floor_height_m = 3.0
total_area_m2 = 300.0

[policy]
default_mode = "SecurityFirst"

[mesh_network]
protocol = "hybrid"              # BLE + PoE for LiDAR

[data]
storage_target = "nvme"
retention_days = 90

[connectivity.cellular]
enabled = true
alert_via_cellular = true
```

### 19.3 Enterprise Deployment

```toml
# Multi-floor, ~1500 m²
# Variant C edge controller (x86 mini PC)
# 30+ sensor nodes + 4 LiDAR nodes

[deployment]
building_floor_count = 3
floor_height_m = 3.0
total_area_m2 = 1500.0

[policy]
default_mode = "SecurityFirst"

[mesh_network]
protocol = "thread"              # Better range for large deployment

[data]
storage_target = "network_path"
network_path = "/mnt/nas/aegis-mesh"
retention_days = 180

[connectivity.cellular]
enabled = true
stream_via_cellular = false
```

---

**End of Edge Controller Setup Guide**
