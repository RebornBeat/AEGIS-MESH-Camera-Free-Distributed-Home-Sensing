# AEGIS-MESH Node Test Jig — Universal Test Station (UTS)

**Project:** AEGIS-MESH
**Purpose:** Automated validation for all AEGIS-MESH node variants
**Status:** Reference Design
**License:** CERN-OHL-S v2

---

## 1. Purpose

The Universal Test Station (UTS) validates all AEGIS-MESH node variants before deployment. Testing confirms: electrical correctness, functional sensing, data handling mode verification, optional hardware switch testing (if installed), IP67 compliance, and mesh protocol integration.

**Critical architectural distinction:** WiFi and cellular connectivity tests apply **only to the edge controller**. All sensor nodes (Always-On, LiDAR, Identity) are tested for mesh radio connectivity only — they have no WiFi or cellular interfaces. The edge controller is the sole external network gateway.

---

## 2. Architecture: Carrier Board + Test Shields

### 2.1 Carrier Board (Universal)

The carrier board provides all test infrastructure used across node classes.

| Component | Specification | Purpose |
|---|---|---|
| Test Controller MCU | STM32H7 or RP2040 | Real-time signal generation, timing measurement |
| Host SBC | Raspberry Pi 5 or x86 mini PC | Runs test suite, logs results, serves results dashboard |
| Programmable Power Supply | 0–5V, 0–3A, programmable | Node power profiling, stress testing |
| Current Sense × 2 | INA219 (main rail + V_CAM/V_SENSOR rail) | Power measurement, isolation verification |
| Communications | USB, UART, SWD, I2C, SPI | Firmware flashing, debug access, sensor I/O |
| RF Shield | Aluminum can with RF-absorbing lining | Isolates DUT during radar tests |
| Mesh Golden Node | Reference nRF52840 or Thread device | Mesh connectivity verification baseline |
| WiFi Test AP | Dedicated 802.11ac/ax access point | Edge controller WiFi testing (edge controller only) |
| Cellular Signal Simulator | RF shielded enclosure with programmable LTE/5G simulator | Edge controller cellular testing without live SIM |
| LED Array Board | Programmable RGB LED matrix | Event camera testing, optical sensor stimulus |
| Audio Test System | Reference speaker + calibrated microphone | Acoustic DOA and classification testing |
| Calibration Target Board | Checkerboard + ArUco markers | Camera calibration, LiDAR registration |

### 2.2 Test Shields Per Node Class

Each node class has a dedicated test shield with specific test fixtures:

| Shield | Key Equipment | Primary Tests |
|---|---|---|
| AlwaysOn Shield (Var A–C) | Radar reflector, PIR LED trigger, acoustic speaker, environmental chamber port | Presence detection, acoustic DOA, environmental response |
| AlwaysOn Shield (Var D) | All above + camera calibration target | Camera data handling mode verification |
| LiDAR Shield (Var A/B) | LiDAR target array, event LED strobe, fog generator port | Depth accuracy, event triggering, atmospheric robustness |
| LiDAR Shield (Var C) | All above + camera chart | Camera + LiDAR fusion, SLAM validation |
| LiDAR Shield (Var D) | All above + 360° camera rig, stitching analyzer, sync timing analyzer | 360° array synchronization, stitching quality, SLAM |
| Identity Shield | Camera lightbox, optional hardware switch actuator, target charts | Identification accuracy, data handling modes, privacy switch |
| Edge Controller Shield | WiFi test AP, cellular signal simulator, mesh golden node, API test client, thermal load fixture | WiFi, cellular, API, mesh hub, thermal |

---

## 3. Test Sequences — All Node Types

### 3.1 All Nodes — Mandatory Tests (Every Unit)

These tests apply to every AEGIS-MESH node regardless of class:

| # | Test Name | Procedure | Pass Criteria |
|---|---|---|---|
| 1 | Power-on self-test | Apply power, observe boot sequence | MCU boots, all rail voltages within ±5% |
| 2 | Rail voltage verification | Measure VCC_3V3, VCC_1V8, V_CAM (if applicable) | All within ±5% of nominal |
| 3 | Sensor enumeration | Query I2C/SPI bus for all expected sensors | All sensors respond with correct IDs |
| 4 | Mesh radio ping | Power on golden node, observe DUT's mesh advertisement | DUT appears on mesh network within 500 ms |
| 5 | Firmware version check | Read firmware version via SWD or mesh query | Version matches current production build hash |
| 6 | Timestamp synchronization | Send time sync from test controller, verify DUT clock alignment | DUT clock within ±1 ms of test controller after sync |
| 7 | Message signing verification | Send signed test message, verify DUT accepts | DUT processes valid signed message |
| 8 | Message rejection (invalid signature) | Send message with invalid HMAC signature | DUT rejects message, no processing occurs |
| 9 | Battery level read | Query battery percentage via mesh protocol | Reading within expected range for battery state |
| 10 | Temperature monitoring | Read on-board temperature sensor | Reading within ±5°C of ambient |

### 3.2 Edge Controller — Additional Tests (Edge Controller Only)

The edge controller is the **only** node with WiFi, cellular, and external API capability. These tests apply only to the edge controller.

#### 3.2.1 WiFi Tests

| # | Test Name | Procedure | Pass Criteria |
|---|---|---|---|
| 11 | WiFi AP association | Configure edge controller with test AP credentials | Edge controller associates within 30 seconds |
| 12 | WiFi RSSI measurement | Measure signal strength at test antenna | RSSI > -70 dBm at 1 m distance |
| 13 | WiFi band selection | Verify 5 GHz preference when available | Connects on 5 GHz when both bands available |
| 14 | API server reachability | HTTP GET to `/api/health` | Returns 200 OK within 1 second |
| 15 | API authentication | Request protected endpoint without token | Returns 401 Unauthorized |
| 16 | API authenticated access | Request with valid bearer token | Returns 200 OK with expected data |
| 17 | WebSocket connection | Open WebSocket to `/ws/events` | Connection established within 2 seconds |
| 18 | Media stream initiation | Request RTSP stream start | Stream available within 5 seconds |
| 19 | mDNS resolution | Query `aegis-mesh.local` from test client | Resolves to edge controller IP |
| 20 | WiFi roaming test (if applicable) | Move between test AP locations | Edge controller reassociates without dropping mesh connections |
| 21 | WiFi throughput test | Transfer 100 MB test file via HTTP | Sustained throughput > 50 Mbps |
| 22 | WiFi under load | Maintain mesh traffic while streaming video | No mesh packet loss > 1% |

#### 3.2.2 Cellular / SIM Tests

**Note:** Cellular tests are conducted with a signal simulator to avoid requiring live carrier connections during manufacturing test. Live SIM tests are optional for field validation.

| # | Test Name | Procedure | Pass Criteria |
|---|---|---|---|
| 23 | SIM card detection | Insert test SIM, query via AT command | `SIM_DET` GPIO indicates card present |
| 24 | SIM power rail | Measure `SIM_VCC` | Within spec (1.8V or 3.0V per module requirement) |
| 25 | SIM communication | Send AT command to cellular module | `AT` returns `OK` within 5 seconds |
| 26 | SIM type detection | Query SIM type via AT command | Returns `nano_sim` or `esim` as appropriate |
| 27 | Cellular module power cycle | Power cycle cellular module, verify startup | Module reports ready within 10 seconds |
| 28 | Signal simulator registration | Connect to LTE test simulator | Module reports registered to test network |
| 29 | Cellular data throughput | Transfer test data over simulated LTE | Throughput meets module spec minimum |
| 30 | Cellular alert transmission | Inject test alert, verify cellular transmission | Alert reaches test endpoint within 5 seconds |
| 31 | Cellular current draw | Measure current during active transmission | Within module specification |
| 32 | Cellular power save mode | Enable power save, measure current | Current drops to specified idle level |
| 33 | GPS/GNSS fix (if equipped) | Provide GPS simulator signal | Position fix obtained within 60 seconds |
| 34 | Cellular under thermal stress | Heat to 50°C, verify operation | No cellular disconnection or errors |

#### 3.2.3 Mesh Hub Tests (Edge Controller as Central Hub)

| # | Test Name | Procedure | Pass Criteria |
|---|---|---|---|
| 35 | Multi-node mesh coordination | Connect 6+ test nodes simultaneously | All nodes appear in edge controller's node registry |
| 36 | Mesh message aggregation | All nodes send simultaneously | Edge controller processes all without loss |
| 37 | Mesh timestamp broadcast | Verify edge controller broadcasts sync messages | All nodes synchronize within ±1 ms |
| 38 | Detection fusion | Multiple nodes report same event | Edge controller produces single fused detection |
| 39 | Zone assignment | Assign nodes to zones via API | Zone assignment reflected in node status |
| 40 | Policy mode switching | Change policy via API | All nodes receive updated policy within 1 second |

#### 3.2.4 Recording Manager Tests

| # | Test Name | Procedure | Pass Criteria |
|---|---|---|---|
| 41 | Recording index | Inject `RecordingAvailable` event from mesh | Event appears in recordings list within 500 ms |
| 42 | Recording retrieval | Request recording via API | File transfer begins within 2 seconds |
| 43 | Recording deletion | Delete recording via API | File removed, index updated within 1 second |
| 44 | Legal export | Request export with integrity chain | ZIP file generated with correct SHA-256 in manifest |
| 45 | Integrity chain verification | Verify hash chain in export | Hash matches file content, chain is valid |
| 46 | Retention enforcement | Configure retention, wait for expiration | Files older than retention period are deleted |
| 47 | Storage quota enforcement | Configure max_storage_mb, fill beyond limit | Oldest recordings deleted to stay within quota |
| 48 | Continuous recording mode | Enable continuous recording | Files written continuously without gaps |

#### 3.2.5 SLAM Validation (Edge Controller with Linux SoM)

| # | Test Name | Procedure | Pass Criteria |
|---|---|---|---|
| 49 | SLAM initialization | Enable SLAM mode | SLAM pipeline starts within 10 seconds |
| 50 | Point cloud ingestion | Receive LiDAR point clouds from test node | Point clouds processed and integrated |
| 51 | Loop closure detection | Simulate revisiting known location | Loop closure detected, drift corrected |
| 52 | Dense map generation | After sufficient scans | 3D mesh generated with < 5 cm error |
| 53 | SLAM under load | Run SLAM + API + recording simultaneously | No frame drops, latency within spec |
| 54 | 360° stitching (from LiDAR Var D) | Receive 8 camera streams from LiDAR node | Stitched 360° output generated |
| 55 | 360° stitching quality | Analyze stitching seams | Seam misalignment < 3 pixels at 1080p |

---

### 3.3 Always-On Node Tests

#### 3.3.1 All Variants (A–D)

| # | Test Name | Procedure | Pass Criteria |
|---|---|---|---|
| 60 | Presence detection (mmWave) | Walk test at 1 m, 3 m, 5 m, 10 m | Detection at all distances within 500 ms |
| 61 | PIR response (Variant A) | Trigger PIR with IR LED pulse | Detection within 100 ms |
| 62 | ToF ranging (Variant A) | Measure distance to target | Within ±10 cm at 3 m |
| 63 | Acoustic DOA | Speaker at 0°, 45°, 90°, 135°, 180° | Direction error < ±15° |
| 64 | Material classification | Glass break, ceramic, wood, footstep sounds | Classification accuracy > 85% |
| 65 | Environmental response | Vary temperature, humidity | Sensors report within ±5% of reference |
| 66 | Raw audio recording (if configured) | Trigger recording, check SD card | Clip file present within 500 ms |
| 67 | Mesh latency | Ping via golden node | Round-trip < 100 ms |

#### 3.3.2 Variant D (Camera) — Additional Tests

| # | Test Name | Procedure | Pass Criteria |
|---|---|---|---|
| 68 | Camera initialization | Power on, verify camera driver loads | Camera ready within 2 seconds |
| 69 | Camera image capture | Trigger single frame capture | Valid JPEG/raw frame received |
| 70 | Data handling mode — Metadata-only | Configure metadata-only, capture | Only classification tag in mesh output, no SD file |
| 71 | Data handling mode — Local storage | Configure local storage, capture | Recording file on SD card within 500 ms |
| 72 | Data handling mode — App streaming | Configure streaming, start capture | Edge controller receives stream within 2 seconds |
| 73 | Data handling mode — Continuous | Configure continuous recording | Files written continuously, no gaps |
| 74 | Optional hardware switch (if installed) — DISABLED | Set switch to DISABLED position | V_CAM = 0 V ± 0.05 V. FAIL if > 0.1 V |
| 75 | Optional hardware switch (if installed) — ENABLED | Set switch to ENABLED position | V_CAM > 3.0 V, camera operational |
| 76 | Optional hardware switch — Firmware state | Verify firmware reports switch state | Correct state reported via mesh message |

---

### 3.4 LiDAR Node Tests

#### 3.4.1 All Variants (A–D)

| # | Test Name | Procedure | Pass Criteria |
|---|---|---|---|
| 80 | LiDAR depth accuracy | Measure calibrated target at 1.0 m | Distance within ±2 cm |
| 81 | LiDAR depth accuracy (extended) | Measure at 2 m, 3 m, 5 m | Within ±5 cm at 5 m |
| 82 | Event sensor response | Trigger with < 1 ms LED strobe | Event detected within 1 ms of strobe |
| 83 | Event sensor timing | Measure event timestamp accuracy | Within ±100 µs of reference clock |
| 84 | Mesh/PoE latency to golden node | Ping via mesh/PoE | Round-trip < 50 ms |
| 85 | Point cloud frame rate | Measure scan rate | Meets configured rate (default 10 Hz) |
| 86 | Point cloud density | Count points per frame | Meets minimum points per scan |

#### 3.4.2 Variants A/B (No Camera)

| # | Test Name | Procedure | Pass Criteria |
|---|---|---|---|
| 87 | Steam/fog immunity | Generate steam, measure detection | Presence detection maintained |
| 88 | Radar + LiDAR fusion | Simultaneous radar and LiDAR detection | Fused detection produced |
| 89 | Radar through-obstacle detection | Place drywall between DUT and target | Detection maintained |

#### 3.4.3 Variant C (+ Camera) — Additional Tests

| # | Test Name | Procedure | Pass Criteria |
|---|---|---|---|
| 90 | Camera initialization | Verify camera driver loads | Camera ready within 3 seconds |
| 91 | Camera + LiDAR timestamp sync | Simultaneous capture | Timestamps aligned within ±5 ms |
| 92 | Data handling mode verification | Test all four modes | Correct behavior per mode |
| 93 | Optional hardware switch (if installed) | Test DISABLED and ENABLED positions | V_CAM = 0V when DISABLED, > 3.0V when ENABLED |
| 94 | SLAM readiness | Verify SLAM prerequisites met | Point cloud + camera stream available |

#### 3.4.4 Variant D (+ 360° Camera Array) — Additional Tests

| # | Test Name | Procedure | Pass Criteria |
|---|---|---|---|
| 95 | All cameras initialization | Power on, query all camera modules | All N cameras report ready within 5 seconds |
| 96 | FSYNC synchronization | Measure timing across all cameras | All cameras trigger within ±1 ms of FSYNC pulse |
| 97 | Individual camera streams | Capture from each camera separately | Each stream valid and synchronized |
| 98 | Stitching quality (via edge controller) | Analyze stitched 360° output | Seam misalignment < 3 px at 1080p |
| 99 | 360° recording on edge SD | Trigger recording, verify edge storage | File present with correct duration |
| 100 | 360° stream to companion app | Request stream via edge controller API | Stream accessible in test client within 5 seconds |
| 101 | 360° coverage verification | Analyze camera FoV overlap | No blind spots in 360° coverage |
| 102 | 360° calibration data | Verify calibration JSON loaded | Calibration data valid, timestamps current |
| 103 | SLAM with 360° input | Run SLAM with 360° camera input | Dense map generated with loop closure |

---

### 3.5 Identity Node Tests

| # | Test Name | Procedure | Pass Criteria |
|---|---|---|---|
| 110 | Power-on self-test | Apply power | MCU boots, all sensors initialize |
| 111 | Identification sensor initialization | Query sensor status | Sensor reports ready |
| 112 | Mesh radio ping | Ping via golden node | Round-trip < 100 ms |
| 113 | Firmware version verification | Read version | Matches production build |
| 114 | Kill switch state read (if installed) | Query `CAM_HW_SW_GPIO` | Correct state reported |

#### 3.5.1 Data Handling Mode Verification

| # | Test Name | Mode | Procedure | Pass Criteria |
|---|---|---|---|---|
| 115 | Metadata-only | A | Capture identification event | Only classification tag in mesh output; no SD file |
| 116 | Local storage | B | Capture with local storage enabled | Recording file on SD within 500 ms |
| 117 | App streaming | C | Start app stream via edge controller | Edge controller receives stream within 2 seconds |
| 118 | Full recording | D | Enable continuous recording | Files written continuously with integrity chain |

#### 3.5.2 Optional Hardware Switch Tests (Only If Installed)

| # | Test Name | Procedure | Pass Criteria |
|---|---|---|---|
| 119 | DISABLED position voltage | Set switch to DISABLED | V_SENSOR = 0 V ± 0.05 V. FAIL if > 0.1 V |
| 120 | ENABLED position voltage | Set switch to ENABLED | V_SENSOR > 3.0 V |
| 121 | DISABLED blocks capture | Attempt capture with switch DISABLED | No capture occurs, firmware reports switch engaged |
| 122 | ENABLED permits capture | Attempt capture with switch ENABLED | Capture succeeds |
| 123 | Firmware state reporting | Query mesh for switch state | Correct physical state reported |

#### 3.5.3 Identification Accuracy Tests

| # | Test Name | Procedure | Pass Criteria |
|---|---|---|---|
| 130 | Known resident recognition | Present enrolled face/person | Correct classification, confidence > 0.90 |
| 131 | Unknown person detection | Present unenrolled face/person | Classification: Unknown, confidence noted |
| 132 | Multiple subjects | Present 2+ subjects | All subjects detected and classified |
| 133 | Processing latency | Measure time from capture to classification | < 2000 ms |
| 134 | False positive rate | 100 negative samples | < 5% false positive |
| 135 | False negative rate | 100 positive samples | < 5% false negative |

---

## 4. IP67 Test (Applicable Nodes)

Applies to: Always-On all variants, LiDAR all variants, Identity all variants.

**Procedure:**
1. Submerge node in 1 m water depth
2. Duration: 30 minutes
3. Remove, dry exterior
4. Power on within 60 seconds

**Post-submersion tests:**
- Powers on successfully
- All sensors initialize
- Mesh radio connects
- Rail voltages within ±2%
- Camera function (if equipped)

**Pass criteria:** All post-submersion tests pass. No corrosion or water ingress visible on disassembly.

---

## 5. Thermal Testing

### 5.1 All Nodes — Basic Thermal

| # | Test Name | Procedure | Pass Criteria |
|---|---|---|---|
| 140 | Ambient operation | Run at 25°C for 1 hour | All sensors functional |
| 141 | High temperature | Operate at 50°C for 1 hour | All sensors functional, no thermal shutdown |
| 142 | Low temperature | Operate at -10°C for 1 hour | All sensors functional |
| 143 | Rapid thermal cycle | Cycle -10°C to 50°C × 5 | No failures, no calibration drift |

### 5.2 Edge Controller — Extended Thermal

| # | Test Name | Procedure | Pass Criteria |
|---|---|---|---|
| 144 | Sustained compute load | Run SLAM + recording at 25°C | Enclosure temp < 45°C after 2 hours |
| 145 | Thermal throttling | Heat to 50°C ambient | Throttling activates, no shutdown |
| 146 | Cellular thermal | Transmit on cellular at max power | No cellular disconnection |
| 147 | WiFi thermal | Stream continuously | No WiFi disconnection |

### 5.3 LiDAR Node — Extended Thermal

| # | Test Name | Procedure | Pass Criteria |
|---|---|---|---|
| 148 | LiDAR sustained operation | Run LiDAR continuously | Enclosure temp < 50°C |
| 149 | 360° camera thermal | All cameras active continuously | Enclosure temp < 50°C |

---

## 6. Software Test Runner

### 6.1 Architecture

The test runner is a Rust CLI application running on the host SBC.

```
test_runner/
├── src/
│   ├── main.rs
│   ├── tests/
│   │   ├── mod.rs
│   │   ├── edge_controller.rs
│   │   ├── always_on.rs
│   │   ├── lidar.rs
│   │   └── identity.rs
│   ├── hardware/
│   │   ├── mod.rs
│   │   ├── power.rs
│   │   ├── mesh.rs
│   │   ├── wifi.rs
│   │   ├── cellular.rs
│   │   └── api.rs
│   └── reporting/
│       ├── mod.rs
│       ├── json.rs
│       └── dashboard.rs
└── Cargo.toml
```

### 6.2 Command Line Interface

```bash
# Run all tests on a specific DUT
aegis-test-runner --dut edge_controller --serial AEG-EDGE-00042

# Run specific test category
aegis-test-runner --dut always_on_b --tests presence,acoustic

# Run with custom configuration
aegis-test-runner --config test_config.toml --dut lidar_d

# Generate report only (no execution)
aegis-test-runner --report-only --results test_results.json

# Run production test sequence
aegis-test-runner --production --dut identity_a --serial AEG-ID-00123
```

### 6.3 Test Configuration File

```toml
# test_config.toml

[runner]
log_level = "info"
timeout_seconds = 300
retry_count = 2

[hardware]
power_supply_device = "/dev/ttyUSB0"
golden_node_address = "192.168.1.200"
wifi_test_ap_ssid = "AEGIS_TEST_AP"
wifi_test_ap_password = "test_password"

[reporting]
output_format = "json"              # "json" | "html" | "csv"
output_path = "./test_results/"
dashboard_enabled = true
dashboard_port = 8080

[tests.all_nodes]
mandatory = true
timeout_ms = 5000

[tests.edge_controller]
wifi_tests = true
cellular_tests = true
slam_tests = true
recording_tests = true

[tests.cellular]
use_simulator = true                # Use RF simulator instead of live SIM
simulator_port = "/dev/ttyUSB1"

[tests.always_on]
presence_distances_m = [1.0, 3.0, 5.0, 10.0]
acoustic_doa_angles_deg = [0, 45, 90, 135, 180]

[tests.lidar]
depth_targets_m = [1.0, 2.0, 3.0, 5.0]
stitching_quality_threshold_px = 3

[tests.identity]
known_subject_count = 5
unknown_subject_count = 3
confidence_threshold = 0.90

[pass_criteria]
all_mandatory_must_pass = true
allow_optional_failures = false
minimum_overall_score = 0.95
```

---

## 7. Test Result Format

### 7.1 JSON Output

```json
{
  "serial": "AEG-EDGE-00042",
  "unit_type": "EdgeController_CM4",
  "firmware_rev": "v1.2.0",
  "hardware_rev": "2.1",
  "test_timestamp_utc": "2024-03-15T14:30:22.004Z",
  "test_duration_s": 245.6,
  "tests": [
    {
      "name": "Power_Rails",
      "status": "PASS",
      "value": {
        "vcc_3v3": 3.31,
        "vcc_1v8": 1.82,
        "v_cam": 3.30
      },
      "duration_ms": 120
    },
    {
      "name": "WiFi_AP_Connect",
      "status": "PASS",
      "latency_ms": 4200,
      "ssid": "AEGIS_TEST_AP",
      "band": "5GHz",
      "rssi_dbm": -52,
      "duration_ms": 5230
    },
    {
      "name": "API_Server_Reachable",
      "status": "PASS",
      "endpoints_tested": 15,
      "failed_endpoints": 0,
      "duration_ms": 850
    },
    {
      "name": "Cellular_AT_Response",
      "status": "PASS",
      "module": "Quectel EC25",
      "response_time_ms": 3200,
      "duration_ms": 3500
    },
    {
      "name": "SIM_Card_Detected",
      "status": "PASS",
      "sim_type": "nano_sim",
      "sim_vcc": 1.82,
      "duration_ms": 150
    },
    {
      "name": "Cellular_Simulator_Registration",
      "status": "PASS",
      "registration_time_ms": 8500,
      "duration_ms": 9000
    },
    {
      "name": "Mesh_Golden_Node",
      "status": "PASS",
      "latency_ms": 12,
      "packet_loss_pct": 0.0,
      "duration_ms": 5000
    },
    {
      "name": "Recording_Manager",
      "status": "PASS",
      "export_integrity": true,
      "sha256_verified": true,
      "duration_ms": 12000
    },
    {
      "name": "SLAM_Initialization",
      "status": "PASS",
      "init_time_ms": 8500,
      "duration_ms": 9000
    },
    {
      "name": "360_Stitching_Quality",
      "status": "PASS",
      "seam_error_px": 2.1,
      "cameras_synced": 8,
      "fsync_jitter_us": 0.8,
      "duration_ms": 15000
    },
    {
      "name": "Thermal_Sustained",
      "status": "PASS",
      "max_temp_c": 44.2,
      "duration_ms": 7200000
    }
  ],
  "failures": [],
  "warnings": [],
  "final_result": "PASS",
  "overall_score": 1.0
}
```

### 7.2 Identity Node Result Example

```json
{
  "serial": "AEG-ID-00123",
  "unit_type": "IdentityNode_V1",
  "firmware_rev": "v1.0.0",
  "test_timestamp_utc": "2024-03-15T14:35:00.000Z",
  "tests": [
    {
      "name": "Power_Rails",
      "status": "PASS",
      "value": {"vcc_3v3": 3.29, "v_sensor": 3.30}
    },
    {
      "name": "Mesh_Radio_Ping",
      "status": "PASS",
      "latency_ms": 45
    },
    {
      "name": "Camera_Initialization",
      "status": "PASS"
    },
    {
      "name": "Data_Handling_Metadata_Only",
      "status": "PASS",
      "mesh_output_contains_tags": true,
      "sd_file_present": false
    },
    {
      "name": "Data_Handling_Local_Storage",
      "status": "PASS",
      "recording_file_ms": 380,
      "file_size_bytes": 2458624
    },
    {
      "name": "Data_Handling_App_Streaming",
      "status": "PASS",
      "edge_received_stream": true,
      "stream_latency_ms": 1850
    },
    {
      "name": "Hardware_Switch_DISABLED",
      "status": "PASS",
      "v_sensor_volts": 0.02
    },
    {
      "name": "Hardware_Switch_ENABLED",
      "status": "PASS",
      "v_sensor_volts": 3.31
    },
    {
      "name": "Known_Resident_Recognition",
      "status": "PASS",
      "classification": "KnownResident",
      "confidence": 0.94
    },
    {
      "name": "Unknown_Person_Detection",
      "status": "PASS",
      "classification": "Unknown",
      "confidence": 0.87
    },
    {
      "name": "IP67_Submersion",
      "status": "PASS",
      "post_submersion_boot": true,
      "all_sensors_functional": true
    }
  ],
  "final_result": "PASS"
}
```

### 7.3 LiDAR 360° Variant Result Example

```json
{
  "serial": "AEG-LD360-00007",
  "unit_type": "LiDARNode_360",
  "firmware_rev": "v1.0.0",
  "test_timestamp_utc": "2024-03-15T14:40:00.000Z",
  "tests": [
    {
      "name": "Power_Rails",
      "status": "PASS",
      "value": {"vcc_3v3": 3.30, "v_cams": 2.85, "v_isp": 1.82}
    },
    {
      "name": "Mesh_PoE_Connectivity",
      "status": "PASS",
      "latency_ms": 32
    },
    {
      "name": "LiDAR_Depth_Accuracy",
      "status": "PASS",
      "distance_m": 1.02,
      "error_cm": 2.1
    },
    {
      "name": "Event_Sensor_Response",
      "status": "PASS",
      "detection_latency_us": 820
    },
    {
      "name": "Camera_Array_Initialization",
      "status": "PASS",
      "cameras_found": 8,
      "expected": 8
    },
    {
      "name": "FSYNC_Synchronization",
      "status": "PASS",
      "max_jitter_us": 0.8,
      "all_cameras_synced": true
    },
    {
      "name": "360_Stitching_Quality",
      "status": "PASS",
      "seam_error_px": 2.1,
      "stitching_latency_ms": 85
    },
    {
      "name": "SLAM_Integration",
      "status": "PASS",
      "point_cloud_fused": true,
      "loop_closure_detected": true
    },
    {
      "name": "IP67_Submersion",
      "status": "PASS"
    }
  ],
  "final_result": "PASS"
}
```

---

## 8. Dashboard Interface

### 8.1 Web Dashboard

The test runner hosts a local web dashboard for real-time test monitoring:

```
http://localhost:8080/
├── Dashboard (test summary)
├── Tests (individual test status)
├── Results (historical results browser)
├── Configuration (test configuration)
└── Calibration (test equipment calibration)
```

### 8.2 API Endpoints

| Method | Path | Description |
|---|---|---|
| GET | `/api/tests` | List all test definitions |
| POST | `/api/run` | Start test run |
| GET | `/api/run/{id}` | Get test run status |
| GET | `/api/run/{id}/results` | Get test results |
| GET | `/api/results` | List all stored results |
| GET | `/api/results/{serial}` | Get results by serial |
| GET | `/api/equipment/status` | Test equipment status |
| POST | `/api/equipment/calibrate` | Start equipment calibration |

---

## 9. Calibration and Maintenance

### 9.1 Test Equipment Calibration Schedule

| Equipment | Calibration Interval | Calibration Procedure |
|---|---|---|
| Programmable power supply | Annual | NIST-traceable voltage/current calibration |
| Current sense modules | Annual | Precision resistor reference |
| WiFi test AP | Quarterly | Signal level verification |
| Cellular signal simulator | Annual | Manufacturer calibration |
| Acoustic reference speaker | Quarterly | SPL calibration at 1 m |
| LiDAR target array | Annual | Distance verification with laser distance meter |
| Camera calibration chart | Annual | Check for fading, damage |
| Mesh golden node | Quarterly | Timestamp sync verification |

### 9.2 Self-Test Sequence

The test jig runs a self-test on power-up:

| # | Self-Test | Criteria |
|---|---|---|
| 1 | Power supply connectivity | All voltage rails reachable |
| 2 | Communication with test MCU | UART/SWD responds |
| 3 | Host SBC boot | Linux booted, network reachable |
| 4 | Mesh golden node health | Node responds, time sync valid |
| 5 | WiFi AP availability | AP responding |
| 6 | Cellular simulator ready | Simulator module responding |
| 7 | Storage available | Sufficient disk space for results |

---

## 10. Production Workflow

### 10.1 Standard Production Test Sequence

```
1. Scan DUT serial number barcode
   └── Load test configuration for this unit type

2. Place DUT on test shield
   └── Verify DUT detected via ID resistor

3. Connect test cables
   ├── Power cable
   ├── Mesh antenna cable (or RF shield close)
   └── Debug cable (SWD/UART)

4. Start automated test sequence
   ├── Mandatory tests (all nodes)
   ├── Variant-specific tests
   ├── Data handling mode tests
   └── Optional hardware switch tests (if applicable)

5. Review results
   ├── All tests PASS → Label "TESTED OK"
   ├── Any FAIL → Route to debug station
   └── Any optional test FAIL → Review, may still ship

6. Store results
   ├── Local database
   ├── Network backup
   └── Barcode linked to results

7. Print test label
   └── Serial, test date, firmware version

8. Package unit
```

### 10.2 Debug Station Workflow

For units that fail production test:

```
1. Scan failed unit barcode
   └── Load failure details

2. Connect to debug interface
   └── SWD/UART console

3. Run targeted debug tests
   ├── Single test repetition with verbose logging
   ├── Signal inspection
   └── Manual override

4. Document failure
   ├── Failure mode
   ├── Root cause
   └── Corrective action

5. Resolution
   ├── Rework and retest
   ├── Scrap and log
   └── Escalate to engineering
```

---

## 11. Safety

### 11.1 Electrical Safety

- Test jig grounded to earth
- ESD mat and wrist strap required
- Power supply current-limited to prevent damage
- No high voltage present

### 11.2 RF Safety

- RF shield closed during radar tests
- Cellular simulator operates at low power
- WiFi AP output limited to 20 dBm

### 11.3 Battery Safety

- Battery charge monitored
- Thermal monitoring during charge
- Fire-resistant test enclosure for battery tests

### 11.4 Cellular Test Safety

- Cellular simulator operates in shielded enclosure
- No live carrier connection during production test
- Live SIM testing optional, requires RF shielded room

---

## 12. Directory Structure

```
test_jig_pcb/
├── carrier_board/
│   ├── carrier_board.kicad_pro
│   ├── carrier_board.kicad_sch
│   ├── carrier_board.kicad_pcb
│   └── gerbers/
├── shields/
│   ├── edge_controller_shield/
│   │   ├── edge_controller_shield.kicad_pro
│   │   └── gerbers/
│   ├── always_on_shield/
│   │   ├── always_on_shield.kicad_pro
│   │   └── gerbers/
│   ├── lidar_shield/
│   │   ├── lidar_shield.kicad_pro
│   │   └── gerbers/
│   ├── lidar_shield_360/
│   │   ├── lidar_shield_360.kicad_pro
│   │   ├── stitching_calibration.json
│   │   └── gerbers/
│   └── identity_shield/
│       ├── identity_shield.kicad_pro
│       └── gerbers/
├── firmware/
│   └── test_runner_firmware/
├── software/
│   └── aegis-test-runner/
│       ├── src/
│       ├── Cargo.toml
│       └── README.md
├── calibration/
│   ├── calibration_records/
│   └── calibration_procedures.md
├── production/
│   ├── sop/
│   └── debug_procedures.md
├── bom/
│   └── test_jig_bom.csv
└── README.md
```

---

## 13. Compliance

- Test jig designed to IEC 61010-1 (electrical safety)
- RF shielded testing per FCC Part 15.209
- ESD protection per IEC 61340-5-1
- Calibration records maintained per ISO 9001

---

**End of Document**
