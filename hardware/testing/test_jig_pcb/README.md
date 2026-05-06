# AEGIS-MESH Node Test Jig — Universal Test Station (UTS)

**Project:** AEGIS-MESH
**Component:** Production & QA Test Platform
**Status:** Reference Design
**Target Users:** Manufacturing, QA Engineers, Research Validation

---

## 1. Purpose

This document specifies the hardware and software architecture for the AEGIS-MESH **Universal Test Station (UTS)**. The UTS is a modular test jig designed to validate all AEGIS-MESH node variants — **AlwaysOn**, **LiDAR**, and **Identity** — post-assembly and during incoming quality control.

The station provides automated, repeatable verification of electrical, functional, and compliance parameters. It enforces the **Hardware Kill Switch** verification required for all Identity Nodes, ensuring the privacy-by-design architecture is physically validated before deployment.

---

## 2. Design Philosophy: Modular Validation

AEGIS-MESH supports multiple node types with different sensor suites. To avoid proliferating single-purpose jigs, the UTS uses a **Carrier Board + Test Shield** architecture:

- **Carrier Board (Universal):** Contains the test controller, power supplies, core measurement instrumentation, and communication interfaces. Remains constant.
- **Test Shield (Node-Specific):** A mechanical and electrical adapter PCB that the node plugs into. Routes signals to the Carrier Board and provides node-specific test fixtures (radar absorbers, optical targets, kill-switch actuators).

This allows the same core test infrastructure to validate any AEGIS-MESH node by swapping the Test Shield.

---

## 3. Hardware Architecture

### 3.1 Carrier Board Specifications

| Component | Specification | Purpose |
|---|---|---|
| **Test Controller** | Raspberry Pi 5 (or x86 SBC) | Runs test suite, logs data, serves UI |
| **MCU Co-Processor** | STM32H7 or RP2040 | Real-time signal generation & capture |
| **Programmable Power** | 0–5V, 0–3A source/measure | Node power profiling (sleep/peak current) |
| **Measurement** | INA219 current sense + ADC | Precision voltage/current measurement |
| **Comms Interface** | USB, UART, SWD, I2C | Flashing firmware, debug, API testing |
| **RF Shielding** | Integrated aluminum RF can | Isolates DUT from ambient noise during radar tests |
| **Fixture Interface** | High-density pogo pin array | Connection to Test Shield |

### 3.2 Node-Specific Test Shields

**AlwaysOn Node Shield:**
- Radar: Reflective target on motorized linear slide (variable distance 0.5–3 m).
- PIR: IR LED source at calibrated flux level.
- Acoustic: Reference microphone + speaker for DOA loopback test.

**LiDAR Node Shield:**
- LiDAR: Blackbody target + retroreflector at calibrated distances (1 m, 5 m optical-path equivalent).
- Event Sensor: High-speed LED strobe (< 100 μs pulse width).

**Identity Node Shield:**
- Camera/Biometric: ISO 12233 resolving power chart + uniform lightbox.
- Kill Switch Actuator: **Solenoid or servo to physically toggle the switch.**
- GPIO Monitor: High-speed logic analyzer probes to verify electrical disconnect of `V_SENSOR`.
- Current Probe: Dedicated INA219 on `V_SENSOR` rail (independent from main supply measurement).

---

## 4. Test Capabilities

### 4.1 Electrical Validation (All Nodes)

| Test | Method | Pass Criteria |
|---|---|---|
| Power-On | Apply VCC; measure current | No short (< 500 mA spike) |
| Rail Verification | ADC measure key rails | Within ±5% of nominal |
| Flash Programming | SWD write + CRC | CRC matches build hash |
| Short Circuit | Current limit trip test | None triggered |

### 4.2 Functional Validation — AlwaysOn Node

| Test | Method | Pass Criteria |
|---|---|---|
| Radar Noise Floor | Record noise in shielded can | Within datasheet spec |
| Radar Target Detection | Motorized slide at 1.5 m | Detection event within 500 ms |
| PIR Trigger | IR LED stimulus at 5 mW/sr | Event within 200 ms |
| DOA Accuracy | Reference speaker at 0°, 90° | Error < ±15° |
| Mesh Radio PER | Golden node at 1 m | PER < 0.1% |

### 4.3 Functional Validation — LiDAR Node

| Test | Method | Pass Criteria |
|---|---|---|
| Depth Accuracy | Calibrated target at 1.0 m | Reading within ±2 cm |
| Event Sensor Response | LED strobe < 100 μs | Event burst within 1 ms |
| Mesh Latency | LiDAR detection to golden node | < 50 ms |

### 4.4 Compliance Validation — Identity Node (CRITICAL)

This test is **mandatory** for every Identity Node unit. Failure = reject and rework.

**Kill Switch Verification Sequence:**

1. Actuator sets SW1 to **DISABLED** position.
2. **Power Rail Probe:** Verify `V_SENSOR` rail at 0 V ± 0.05 V.
3. **Data Line Probe:** Verify MIPI data lines are high-impedance.
4. If `V_SENSOR` > 0.1 V: **FAIL — REJECT UNIT.**
5. Actuator sets SW1 to **ENABLED** position.
6. **Power Rail Probe:** Verify `V_SENSOR` > 3.0 V (sensor powered).
7. **Firmware State:** Verify `KILL_SWITCH_STATE = ENABLED` reported via mesh radio.
8. **Camera Response:** Verify identification sensor initializes (first-frame timeout < 5 seconds).

---

## 5. Software Stack

### 5.1 Test Runner

Written in **Rust** (`aegis-test-harness` crate). CLI or simple Web UI for operator control.

### 5.2 Test Sequence State Machine

```
Start → Scan QR/Serial
      → Detect Node Type
      → Electrical Test
        → Pass? → Flash Firmware
               → Functional Test
                 → Pass? → Compliance Test
                          → Pass? → Mark Passed → End
                          → Fail  → Fail: Log & Halt
               → Fail  → Fail: Log & Halt
        → Fail  → Fail: Log & Halt
```

### 5.3 Logging

All results logged to:
- Local SQLite database.
- Network database (if connected).

Each entry: Serial Number + Node Type + Timestamp + Test Results (per-test pass/fail + measured values) + Firmware Version.

**Example PASS report:**
```json
{
  "serial": "AEG-ALW-00123",
  "node_type": "AlwaysOn",
  "firmware_rev": "v1.2.0",
  "tests": [
    {"name": "Power_Rails", "status": "PASS", "value": "3.31V", "duration_ms": 120},
    {"name": "Radar_Noise_Floor", "status": "PASS", "value_db": -42},
    {"name": "PIR_Trigger", "status": "PASS", "latency_ms": 145},
    {"name": "Mesh_PER", "status": "PASS", "per_pct": 0.02}
  ],
  "final_result": "PASS"
}
```

**Example FAIL report (Identity Node kill switch):**
```json
{
  "serial": "AEG-IDN-00077",
  "node_type": "Identity",
  "tests": [
    {"name": "Kill_Switch_Isolation", "status": "FAIL", "v_sensor_mv": 3280, "expected_mv": 0}
  ],
  "final_result": "FAIL",
  "disposition": "REJECT — Kill switch does not isolate sensor power. Rework required."
}
```

---

## 6. Mechanical Design

- **Enclosure:** Light-tight box for optical sensor tests.
- **Fixture:** 3D-printed or machined bed with alignment pins for Test Shield.
- **E-Stop:** Emergency stop button cuts power to DUT immediately.
- **Indicator:** Green LED (PASS), Red LED (FAIL), Amber LED (IN PROGRESS).

---

## 7. Maintenance Schedule

| Item | Interval | Action |
|---|---|---|
| Distance targets | Monthly | Verify calibration with reference target |
| Pogo pins | Every 500 insertions | Inspect for wear, replace bent pins |
| RF absorbers | Quarterly | Verify attenuation spec maintained |
| Kill-switch actuator | Every 1000 cycles | Verify solenoid full travel |

---

## 8. Bill of Materials (Test Jig)

| Component | Description | Qty | Notes |
|---|---|---|---|
| Test Jig PCB | Main interface board | 1 | Manufactured per Gerbers |
| Test Controller SBC | Raspberry Pi 5 / STM32H7 | 1 | Runs test firmware |
| Pogo Pins | Spring-loaded contacts | ~30 | Match pitch to node pads |
| INA219 × 2 | Current sense amplifier | 2 | One main supply, one V_SENSOR |
| Kill Switch Actuator | Solenoid 5V | 1 | Identity Shield only |
| LED Strobe | High-speed LED driver | 1 | LiDAR/Event Shield |
| USB-C Cable | Host PC connection | 1 | |
| Fixture (Mechanical) | 3D printed or machined | 1 | Per node geometry |
| Power Supply | 5V DC, 3A bench supply | 1 | |
