# LiDAR Node — Hardware Schematic Reference Design

**Project:** AEGIS-MESH
**Node Class:** LidarNode (Volume sensing, high-resolution geometry)
**Primary Role:** Full 3D tracking, geometry, and material classification
**Status:** Reference design, variant 1 of N.

---

## 1. Purpose

The LiDAR Node provides high-resolution geometric sensing for volume tracking. It activates on-demand (triggered by the Always-On Node's motion events) or continuously at low duty cycle for dynamic remap scanning.

**Capabilities:**
- High-resolution 3D point cloud for object geometry, occupancy mapping, and dynamic remap
- Event-based vision for microsecond-latency motion detection and LiDAR trigger
- mmWave radar (optional on this node variant; may be consolidated from Always-On Node coverage)
- Mesh radio communication

---

## 2. Design Philosophy

**Modularity:** The LiDAR module, event camera, and optional radar are on independent sub-circuits. Populate only what the research configuration requires.

**No Artificial Limits:** Resolution, range, and accuracy are determined by the sensor modules selected. The architecture imposes no limits on point cloud density, frame rate, or scan pattern.

**Processing-First:** Point cloud clustering and centroid extraction run at the LiDAR scan rate on-MCU (~10 Hz scanning, up to 50 Hz solid-state). The MCU must have sufficient compute for this without offloading to the edge controller.

---

## 3. Candidate Component Set (Test Variants — Not Locked)

### MCU

- **Variant A:** STM32H743 (Cortex-M7, 480 MHz, sufficient for LiDAR point cloud processing + event stream)
- **Variant B:** Raspberry Pi RP2040 + co-processor FPGA (PIO for LiDAR I/O; FPGA for event stream processing)
- **Variant C:** NXP i.MX RT1060 (Cortex-M7, 600 MHz, rich peripheral set)

### Solid-State LiDAR Module

- **Variant A:** Livox MID-360 (non-repetitive scan pattern, 40 m range, 200k pts/s, Ethernet) — preferred for ceiling mount
- **Variant B:** Robosense BPearl (hemispherical FoV, 30 m, 120k pts/s, Ethernet/RS-485)
- **Variant C:** Ouster OS0-128 (128-beam, 50 m, high density, Ethernet) — highest resolution
- **Variant D (short-range):** Terabee 3Dcam (RGBD ToF, 2 m range, USB) — suitable for sub-room or close-range nodes

**Selection criteria:** FoV coverage from ceiling mount, range at indoor distances (1–10 m most relevant), angular resolution for human-sized object discrimination, power consumption, interface requirements.

### Event Camera (Fast-Motion Trigger)

- **Variant A:** Prophesee EVK4 (IMX636 sensor, 1280×720, USB, mature SDK) — evaluation
- **Variant B:** iniVation DAVIS346 (346×260, USB, combined frame+events)
- **Variant C:** Prophesee Metavision EVK3 (640×480, USB, lower power)

The event camera wakes the LiDAR for high-resolution scan when microsecond-latency motion is detected. This reduces LiDAR duty cycle during quiet periods, extending lifetime and reducing power consumption.

### Optional mmWave Radar

- **Variant A:** TI IWR6843AOP (60 GHz, as Always-On Node — for dual-modality volume node)
- **Variant B:** Acconeer XR112 (lower power, if radar only needed for velocity)

### Power Supply

- **PoE 802.3af/at preferred** for ceiling-mounted LiDAR nodes (avoids battery constraints).
- LiDAR modules consume 5–15 W; battery operation is marginal for extended deployment.
- LiDAR Ethernet interface may need a PoE splitter for power + data over single cable.

---

## 4. Node Activation Strategy (Event-Triggered)

```
Event Camera (always on, low power: ~200 mW)
       │
       │ Fast-motion event detected (< 1 ms latency)
       ▼
LiDAR Controller (MCU)
       │ Wakes LiDAR from sleep state
       ▼
Solid-State LiDAR (active for 1–3 scan cycles: ~100–300 ms)
       │
       ▼
Point Cloud → DBSCAN Cluster → Centroid extraction → DetectionEvent
       │
       ▼
Mesh Radio → Edge Controller (< 10 ms transport latency)
       │
       ▼
LiDAR returns to sleep (if no continued motion detected)
```

This event-triggered activation reduces mean LiDAR power from 10 W continuous to under 2 W average during typical residential occupancy patterns (quiet ~90% of the time).

---

## 5. Interface Mapping

| Component | Interface | Notes |
|---|---|---|
| **Solid-State LiDAR** | Ethernet (UDP) or RS-485 | Driver: custom Rust async UDP listener |
| **Event Camera** | USB 2.0 or SPI parallel | DMA required for high event rates |
| **Optional mmWave** | SPI/UART | As Always-On Node |
| **Mesh Radio** | SPI/UART | As Always-On Node |
| **Debug** | SWD + UART | Standard ARM 10-pin |

---

## 6. Power Architecture

| Component | Typical Power | Peak Power |
|---|---|---|
| LiDAR module | 5–10 W active | 15 W |
| Event camera | 0.2–0.5 W | 1 W |
| MCU | 0.1–0.3 W | 0.5 W |
| Mesh radio | 0.05–0.2 W | 0.5 W |
| **Total (active)** | **~6–11 W** | **~17 W** |

PoE 802.3at (Class 4, 25.5 W max) is comfortable. PoE 802.3af (Class 3, 15.4 W max) may be marginal for highest-resolution variants.

---

## 7. Testing Strategy

1. **Detection range test:** Detect adult human at 2 m, 5 m, 8 m from ceiling mount. Measure point cloud density and centroid accuracy (target: < 5 cm error).
2. **Geometry accuracy test:** Known-size box (50×50×50 cm) at 3 m. Measure reported dimensions vs. true dimensions.
3. **Occlusion test:** Human behind sofa; verify low-angle node sees what ceiling node cannot.
4. **Event-trigger accuracy:** Rapid hand motion; verify LiDAR wakes within one event-camera batch (< 5 ms from motion onset).
5. **Steam/fog immunity:** LiDAR point cloud in steam environment. Verify graceful degradation reporting (not false detections).
6. **Power profile:** Idle (event camera only), event-triggered, continuous scan. Record W per state.
7. **Mesh latency:** LiDAR detection to edge controller alert. Target: < 50 ms.

---

## 8. Schematic Files

| File | Description |
|---|---|
| `lidar_node.kicad_pro` | KiCad project file |
| `lidar_node.kicad_sch` | Main schematic |
| `lidar_node.kicad_pcb` | PCB layout |
| `lidar_node.pdf` | PDF export for review |
