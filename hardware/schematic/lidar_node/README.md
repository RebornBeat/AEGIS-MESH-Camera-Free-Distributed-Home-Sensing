# LiDAR Node — Hardware Schematic Reference Design

**Node Class:** LidarNode (Volume sensing, high-resolution geometry)
**Status:** Reference design, variant 1 of N.

---

## Purpose

The LiDAR Node provides high-resolution geometric sensing for volume tracking. It activates on-demand (triggered by the Always-On Node's motion events) or continuously at low duty cycle for dynamic remap scanning.

Capabilities:
- High-resolution 3D point cloud for object geometry, occupancy mapping, and dynamic remap
- Event-based vision for microsecond-latency motion detection and LiDAR trigger
- mmWave radar (optional on this node variant; may be consolidated from Always-On Node coverage)
- Mesh radio communication

---

## Candidate Component Set (Test Variants — Not Locked)

### MCU
- **Variant A:** STM32H743 (Cortex-M7, 480 MHz, sufficient for LiDAR point cloud processing + event stream)
- **Variant B:** Raspberry Pi RP2040 + co-processor FPGA (PIO for LiDAR I/O; FPGA for event stream processing)
- **Variant C:** NXP i.MX RT1060 (Cortex-M7, 600 MHz, rich peripheral set)

Point cloud clustering and centroid extraction must run at the LiDAR scan rate (~10 Hz for scanning, up to 50 Hz for solid-state). MCU must have sufficient compute for this without offloading to the edge controller.

### Solid-State LiDAR Module
- **Variant A:** Livox MID-360 (non-repetitive scan pattern, 40 m range, 200k pts/s, Ethernet)
- **Variant B:** Robosense BPearl (hemispherical FoV, 30 m, 120k pts/s, Ethernet/RS-485)
- **Variant C:** Ouster OS0-128 (128-beam, 50 m, high density, Ethernet)
- **Variant D (short-range):** Terabee 3Dcam (RGBD ToF, 2 m range, USB, suitable for sub-room nodes)

Selection criteria: FoV coverage from ceiling mount, range at indoor distances (1–10 m most relevant), angular resolution for human-sized object discrimination, power consumption, interface requirements.

### Event Camera (Fast-Motion Trigger)
- **Variant A:** Prophesee EVK4 (IMX636 sensor, 1280×720, USB, mature SDK)
- **Variant B:** iniVation DAVIS346 (346×260, USB, combined frame+events)
- **Variant C:** Prophesee Metavision EVK3 (640×480, USB, lower power)

The event camera wakes the LiDAR for high-resolution scan when microsecond-latency motion is detected. This reduces LiDAR duty cycle during quiet periods, extending lifetime and reducing power.

### Power Supply
- **PoE 802.3af/at preferred** for ceiling-mounted LiDAR nodes (avoids battery constraints)
- LiDAR modules can consume 5–15 W; battery operation is marginal for extended deployment

---

## Node Activation Strategy

```
Event Camera (always on, low power)
       │
       │ Fast-motion event detected
       ▼
LiDAR Controller (MCU)
       │ Wakes LiDAR from sleep
       ▼
Solid-State LiDAR (active for 1–3 scan cycles)
       │
       ▼
Point Cloud → Cluster → Centroid extraction → DetectionEvent
       │
       ▼
Mesh Radio → Edge Controller
       │
       ▼
LiDAR returns to sleep (if no continued motion)
```

This event-triggered activation strategy reduces mean LiDAR power consumption from 10 W continuous to under 2 W average during typical residential occupancy patterns.

---

## Testing Strategy

1. **Detection range test:** Detect adult human at 2 m, 5 m, 8 m from ceiling mount. Measure point cloud density and centroid accuracy.
2. **Geometry accuracy test:** Known-size box (50×50×50 cm) at 3 m. Measure reported dimensions vs. true dimensions.
3. **Occlusion test:** Human behind sofa; verify low-angle node sees what ceiling node cannot.
4. **Event-trigger accuracy:** Rapid hand motion; verify LiDAR wakes within one event-camera batch (< 5 ms).
5. **Steam immunity test:** LiDAR point cloud in steam environment. Verify graceful degradation reporting (not false detections).
6. **Power profile:** Idle, event-triggered, continuous scan. Record W per state.
