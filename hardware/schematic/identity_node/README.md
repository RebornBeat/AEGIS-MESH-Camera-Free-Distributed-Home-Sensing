# Identity Node — Hardware Schematic Reference Design

**Node Class:** IdentityNode (Opt-in identification — isolated from sensing mesh)
**Status:** Reference design, variant 1 of N. Physical kill switch is mandatory and non-negotiable.

---

## Purpose

The Identity Node is the episodic identification layer. It activates only when the policy engine triggers it (Security-First mode with configured trigger conditions). It captures data sufficient to classify who or what is present, processes the classification locally, and transmits only the classification result (never raw imagery or biometric data) over the mesh.

This node is **physically isolated** from the sensing mesh:
- Separate hardware from Always-On and LiDAR nodes
- Separate power rail controlled by hardware kill switch
- Separate MCU that does not share bus with sensing mesh MCU
- Kill-switch GPIO must read HIGH for any sensor activity to occur

---

## Kill Switch: The Primary Trust Anchor

**This is the most important element of this schematic.**

The kill switch is a physical slider or toggle that cuts:
1. Power rail to all imaging/biometric sensors on this node
2. Data lines (SPI/I2C/USB) from sensors to the MCU

The MCU continues to receive power (to respond to activation commands from the edge controller) but the sensors are physically disconnected until the kill switch is physically enabled.

**Firmware reads kill-switch GPIO at boot and at every activation cycle.** If the GPIO reads KILL_SWITCH_DISABLED, the firmware returns `SensorError::KillSwitchEngaged` and halts any sensor activity. The only activity permitted without a valid kill-switch reading is: responding to health queries and emitting `KillSwitchEngaged` status messages.

---

## Candidate Component Set (Test Variants — Not Locked)

### MCU (Identity Processing)
- **Variant A:** STM32H755 (dual Cortex-M7/M4, hardware crypto, TrustZone, USB)
- **Variant B:** NXP i.MX RT1060 (Cortex-M7, 600 MHz, camera interface)

The identity MCU must NOT be the same die as the sensing mesh MCU. Physical isolation is required.

### Identification Sensor — Configurable (All Are Test Variants)
All of the following are under research consideration. No one type is mandated.

**Visual (Camera-Based):**
- **Variant A:** Arducam IMX219 (8 MP, CSI, configurable — wide or narrow angle lens)
- **Variant B:** Sony IMX307 (2 MP, low light, MIPI CSI)
- **Variant C:** OmniVision OV5640 (5 MP, autofocus, USB/MIPI)

**Near-IR / Structured Light:**
- **Variant D:** Intel RealSense D415 (stereo depth + RGB, USB)
- **Variant E:** Apple dot-projector style structured light (IR emitter + IR camera — research only)

**Face Recognition Module (All-in-One):**
- **Variant F:** HiMax HM01B0 (QVGA, ultra-low power, first-stage detection on-chip)
- **Variant G:** Luxonis OAK-1 (neural inference on-module, no raw video off-module)

**Non-Visual Biometric:**
- **Variant H:** Fingerprint scanner (Synaptics FS4500) — only viable for fixed entry points
- **Variant I:** Voice recognition pre-processor (Sensory TrulyHandsfree) — audio only variant

Selection criteria: identification accuracy at deployment distance (1–3 m typical), processing capability for on-device inference, power consumption during active identification cycle, physical size for enclosure.

### Kill Switch Component
- **Variant A:** Alps SSSS811101 (SMD slider, 0.5 mm travel, clear tactile)
- **Variant B:** Omron D2JW-01K13H (subminiature snap-action, rated 100k cycles)

The kill switch must be physically accessible to the user. Enclosure design must expose the switch for user operation without requiring tools.

### Mesh Radio
- Same mesh radio as Always-On Node (consistent protocol layer across all node types)
- The identity node uses the same mesh radio but transmits on the isolated identity channel (logical isolation enforced by protocol; physical isolation by separate antenna if available)

---

## Schematic Block Diagram

```
 ┌──────────────────────────────────────────────────────────────┐
 │                    Identity Node PCB                         │
 │                                                              │
 │                    KILL SWITCH (Physical Slider)             │
 │                           │                                  │
 │      ┌────────────────────┤────────────────────┐             │
 │      │  SENSING POWER     │   MESH POWER        │             │
 │      │  (controlled)      │   (always on)       │             │
 │      ▼                    │        ▼             │             │
 │  ┌─────────┐              │   ┌─────────┐        │             │
 │  │ Identity│              │   │  Mesh   │        │             │
 │  │ Sensor  │──Kill───────►│   │  Radio  │        │             │
 │  │ (camera │ Switch GPIO  │   │         │        │             │
 │  │ etc.)   │              │   └────┬────┘        │             │
 │  └────┬────┘              │        │              │             │
 │       │                  │    ┌────▼────┐         │             │
 │       │ CSI/USB          │    │ Identity│         │             │
 │       ▼                  │    │  MCU    │         │             │
 │  ┌─────────┐             └───►│         │         │             │
 │  │ On-Board│                  │ GPIO:   │         │             │
 │  │Inference│◄─────────────────│ kill_sw │         │             │
 │  │ Engine  │                  │ ─────── │         │             │
 │  └─────────┘                  │ reads   │         │             │
 │                               │ HIGH =  │         │             │
 │                               │ active  │         │             │
 │                               └─────────┘         │             │
 │                                                              │
 └──────────────────────────────────────────────────────────────┘
```

---

## On-Device Processing Requirement

Raw sensor data **must not leave this node.** The identity MCU processes all raw data locally. The output is exclusively classification labels and confidence scores.

For camera variants: the image capture, inference, and result extraction all occur on-MCU. Images are never written to flash, transmitted over any interface, or buffered in RAM after inference completes. Only the classification result (a small JSON struct: `{"class": "KnownResident", "confidence": 0.94}`) transmits over the mesh radio.

This is enforced architecturally by the firmware binary: there is no serial protocol or interface that accepts raw frame data from an external host.

---

## QC Required Tests

1. **Kill-switch continuity test:** With kill switch in DISABLED position, confirm zero current to identity sensor using bench multimeter. This test is mandatory for every unit before shipping.
2. **Kill-switch firmware test:** Boot with kill switch DISABLED; verify firmware emits `KillSwitchEngaged` status and no sensor activation occurs.
3. **Identification accuracy test:** Known resident at 1 m, 2 m, 3 m. Unknown person at same distances. Measure true-positive rate for known, false-positive rate for unknown.
4. **Processing latency test:** Time from activation command received to classification result transmitted. Target: < 2 seconds for camera-based identification.
5. **No-egress test:** Use packet sniffer on mesh radio during identification cycle. Verify zero raw imagery or biometric data in any transmitted packet.
