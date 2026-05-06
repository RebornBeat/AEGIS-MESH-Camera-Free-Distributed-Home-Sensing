# Identity Node — Hardware Schematic Reference Design

**Project:** AEGIS-MESH
**Node Class:** IdentityNode (Opt-in identification — physically isolated from sensing mesh)
**Status:** Reference design, variant 1 of N. Physical kill switch is mandatory and non-negotiable.

---

## 1. Purpose

The Identity Node is the episodic identification layer. It activates only when:
1. The **Hardware Kill Switch** is in the **ENABLED** position (user-controlled physical switch).
2. A trigger event is received from the Policy Engine (e.g., "Unknown Object at Entry," "Security-First Mode Active").

It captures data sufficient to classify who or what is present, processes the classification locally, and transmits only the classification result (never raw imagery or biometric data) over the mesh.

This node is **physically isolated** from the sensing mesh:
- Separate hardware from Always-On and LiDAR nodes.
- Separate power rail controlled by hardware kill switch.
- Separate MCU that does not share bus with sensing mesh MCU.
- Kill-switch GPIO must read HIGH for any sensor activity to occur.

---

## 2. The Hardware Kill Switch — Primary Trust Anchor

**This is the most important element of this schematic. No part of the design compromises it.**

The kill switch is a physical slider or toggle that cuts:
1. **Power rail** to all imaging/biometric sensors on this node.
2. **Data lines** (SPI/I2C/USB) from sensors to the MCU (via analog switches or direct trace cuts).

The MCU continues to receive power (to respond to activation commands from the edge controller) but the sensors are physically disconnected until the kill switch is physically enabled.

**Firmware reads kill-switch GPIO at boot and at every activation cycle.** If the GPIO reads `KILL_SWITCH_DISABLED`, the firmware returns `SensorError::KillSwitchEngaged` and halts any sensor activity. The only activity permitted without a valid kill-switch reading: responding to health queries and emitting `KillSwitchEngaged` status messages.

### 2.1 Kill Switch Physical Implementation

The kill switch (`SW1`) is a **physical DPST (Double Pole Single Throw) slider switch** accessible on the enclosure exterior.

**Pole 1:** Breaks the `V_SENSOR` power rail (disconnects power to all identification sensors).
**Pole 2:** Breaks the data lines OR grounds the `KILL_SW_GPIO` signal to MCU.

**Positions:**
- **DISABLED (Default):** Power to identification sensor is physically cut. MCU reads `KILL_SW_GPIO` = LOW. Firmware halts.
- **ENABLED:** Power rail closed. MCU reads `KILL_SW_GPIO` = HIGH. Firmware boots identification subsystem. Node enters Standby, awaiting activation command.

### 2.2 User Verification

A user can verify the kill switch is active by:
1. **Visual Inspection:** The switch is physically in the DISABLE position.
2. **Electrical Test:** Zero voltage at identification sensor power input when DISABLED.
3. **Firmware Report:** Node reports `KillSwitchEngaged` status over mesh radio when booted with switch DISABLED.

---

## 3. Core Architecture

```
Power Supply (5V USB / Battery)
       │
       ├── [ Hardware Kill Switch SW1 (DPST) ] ─────────────┐
       │         │                                           │
       │    Pole 1: V_SENSOR rail cut               GPIO read by MCU
       │         │                                           │
       ▼         ▼                                           │
  ┌─────────┐  V_SENSOR ──────────────────────────────────  │
  │ MCU     │                                        │       │
  │ (SoC)   │◄────── KILL_SW_GPIO (reads LOW=disabled) ◄──── ┘
  └────┬────┘
       │
       │ (CSI / DVP / USB)
       ▼
┌─────────────┐     Processed locally; never transmitted raw
│ Identification  │
│ Sensor          │
│ (Configurable)  │
└─────────────┘
       │
       ▼
┌─────────────┐
│ Local       │  (Optional: logs, embeddings — user-configured)
│ Storage     │
└─────────────┘
       │
       ▼
┌─────────────┐
│ Mesh Radio  │  Transmits: classification metadata ONLY
│ Interface   │  Never transmits: raw frames, biometrics
└─────────────┘
```

---

## 4. Candidate Component Set (Test Variants — Not Locked)

### MCU (Identity Processing)

- **Variant A:** STM32H755 (dual Cortex-M7/M4, hardware crypto, TrustZone, USB, camera interface)
- **Variant B:** NXP i.MX RT1060 (Cortex-M7, 600 MHz, rich camera interface, excellent neural inference performance)

The identity MCU must NOT be the same die as the sensing mesh MCU. Physical isolation is required.

### Identification Sensor — Configurable (All Are Test Variants — No One Type Mandated)

**Visual (Camera-Based):**
- **Variant A:** Arducam IMX219 (8 MP, CSI, configurable wide or narrow angle lens)
- **Variant B:** Sony IMX307 (2 MP, low light, MIPI CSI)
- **Variant C:** OmniVision OV5640 (5 MP, autofocus, USB/MIPI)

**Near-IR / Structured Light:**
- **Variant D:** Intel RealSense D415 (stereo depth + RGB, USB)
- **Variant E:** Dot-projector style structured light (IR emitter + IR camera — research only)

**Self-Contained Neural Inference Modules:**
- **Variant F:** HiMax HM01B0 (QVGA, ultra-low power, first-stage detection on-chip)
- **Variant G:** Luxonis OAK-1 (neural inference on-module, no raw video off-module)

**Non-Visual Biometric:**
- **Variant H:** Fingerprint scanner (Synaptics FS4500) — only viable for fixed entry points
- **Variant I:** Voice recognition pre-processor (Sensory TrulyHandsfree) — audio-only identification variant

**Selection criteria:** Accuracy at 1–3 m deployment distance, on-device inference capability, power consumption during active identification cycle, physical size for enclosure.

### Kill Switch Component

- **Variant A:** Alps SSSS811101 (SMD slider, 0.5 mm travel, clear tactile feedback)
- **Variant B:** Omron D2JW-01K13H (subminiature snap-action, rated 100k cycles)

The kill switch must be physically accessible to the user. Enclosure design must expose the switch for operation without requiring tools.

### Mesh Radio

- Same mesh radio as Always-On Node (consistent protocol layer across all node types).
- Identity node transmits on the isolated identity channel (logical isolation enforced by protocol; physical isolation by separate antenna if available).

---

## 5. Schematic Blocks

### 5.1 Power Supply

- **Input:** 5V DC (USB-C or screw terminal).
- **Main rail:** 3.3V LDO for MCU and peripherals.
- **Switched rail:** Secondary power rail for identification sensor, gated by Kill Switch.

### 5.2 Camera Interface

- **Connector:** 24-pin FPC connector (compatible with standard camera modules).
- **Design:** Modular — any compatible camera module can be connected.
- **Route:** MIPI CSI-2 traces with 100 Ω differential impedance; length-match pairs within 5 mil.

### 5.3 Local Storage (Optional)

- microSD Card slot (SPI mode) or external QSPI Flash.
- **Control:** User configures if/what is stored. Default: no local storage.

### 5.4 Mesh Communication

- **Protocol:** Only `IdentificationEvent` metadata transmitted.
- **Example payload:** `{"id": "node_04", "type": "identity", "classification": "Known Resident", "confidence": 0.96}`
- **Never transmitted:** Raw frames, embeddings, intermediate processing data.

---

## 6. Privacy Architecture

- **Sensing Mesh:** Never receives raw frames. Only receives `IdentificationEvent` metadata.
- **Identity Node:** Processes frames locally. Never transmits raw video.
- **User Control:**
  - User must physically enable the node via kill switch.
  - User configures retention policies (default: no retention).
  - User approves model updates (cannot be OTA'd without consent).

---

## 7. On-Device Processing Requirement

Raw sensor data **must not leave this node.** The identity MCU processes all raw data locally. Output is exclusively classification labels and confidence scores.

For camera variants: image capture, inference, and result extraction all occur on-MCU. Images are never written to external flash, transmitted over any interface, or buffered in RAM after inference completes.

This is enforced architecturally by the firmware binary: there is no serial protocol or interface that accepts raw frame data from an external host.

---

## 8. QC Required Tests

1. **Kill-switch continuity test:** With switch DISABLED, confirm zero current to identity sensor (bench multimeter). Mandatory for every unit before shipping.
2. **Kill-switch firmware test:** Boot with switch DISABLED; verify firmware emits `KillSwitchEngaged` and no sensor activation occurs.
3. **Identification accuracy test:** Known resident at 1 m, 2 m, 3 m; unknown person at same distances. Measure true-positive rate for known, false-positive rate for unknown.
4. **Processing latency test:** Time from activation command received to classification result transmitted. Target: < 2 seconds for camera-based identification.
5. **No-egress test:** Packet sniffer on mesh radio during identification cycle. Verify zero raw imagery or biometric data in any transmitted packet.

---

## 9. Compliance Notes

- **Radio:** Wi-Fi / BLE module must be FCC/CE certified.
- **Optical Safety:** If using IR illumination, ensure Class 1 (eye-safe) wavelengths.
- **Privacy:** Hardware kill switch satisfies "privacy by design" principles for most jurisdictions. Consult legal counsel for specific deployment contexts.
- **Kill Switch:** The kill switch is not optional. Any Identity Node without a hardware kill switch does not comply with AEGIS-MESH privacy requirements.

---

## 10. Directory Contents

```
identity_node/
├── identity_node.kicad_pro
├── identity_node.kicad_sch
├── identity_node.kicad_pcb
├── identity_node.pdf
└── README.md
```

---

## 11. Design Checklist (For Implementation)

- [ ] Kill switch cut verified: zero voltage on `V_SENSOR` when SW1 = DISABLED.
- [ ] GPIO pull-up/down: MCU reads correct state at boot.
- [ ] Firmware halt: `if kill_switch == DISABLED { loop {} }`.
- [ ] FPC connector accessible and correct pinout orientation.
- [ ] Power budget: LDO can handle peak current during neural inference.
- [ ] Thermal vias: present under high-performance MCU.
- [ ] Mounting holes: provided for enclosure integration.
- [ ] Data line isolation: analog switches in series with MIPI data lines, controlled by SW1 Pole 2.
