# Test Jig PCB — AEGIS-MESH Production Testing

**Purpose:** Automated manufacturing validation for all AEGIS-MESH node types.

---

## Overview

The test jig is a PCB that contacts the node under test (NUT) via pogo pins. It verifies:
- Power-on self-test
- Sensor initialization (each sensor responds to initialization sequence)
- Mesh radio connectivity (radio appears on network, responds to ping)
- Kill-switch function (Identity Node only): power to identity sensor cuts and restores with switch actuation
- Flash programming verification (firmware CRC matches published build hash)

The test jig connects to a host PC via USB. The host runs the test suite from `hardware/testing/test_suite.py` (or equivalent Rust CLI). Results are logged to `hardware/testing/results/` with timestamp, node serial number, and pass/fail per test.

---

## Design Requirements

- Pogo pin landing pattern matches each node variant's test pads (separate jig variants per node class)
- Electrically isolated power supply for each power domain under test (prevents bus contamination)
- GPIO inputs for reading kill-switch state under test
- USB isolation between host PC and DUT (prevents ground loop faults)
- Status LEDs: green (pass), red (fail), amber (in progress)

---

## Kill Switch Test Procedure

This test is mandatory for every Identity Node unit:

1. Jig connects to Identity Node.
2. Host commands: `SET_KILL_SWITCH_POSITION disabled`.
3. Jig measures current on identity sensor power rail. Expected: 0 mA ± 2 mA.
4. If current > 2 mA: FAIL — unit does not isolate sensor. REJECT.
5. Host commands: `SET_KILL_SWITCH_POSITION enabled`.
6. Jig measures current on identity sensor power rail. Expected: > 10 mA (sensor powered).
7. Firmware boot: verify `KILL_SWITCH_STATE = ENABLED` reported via mesh protocol.
8. If any step fails: REJECT unit. Log failure reason and unit serial number.

All Identity Node units that fail the kill-switch test are rejected. No exceptions.
