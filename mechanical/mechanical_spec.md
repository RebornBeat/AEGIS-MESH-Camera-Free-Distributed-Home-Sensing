# Mechanical Specification — AEGIS-MESH Enclosures

**Version:** 0.1 | **Status:** Reference Design

---

## 1. Overview

This document specifies the mechanical enclosures for AEGIS-MESH node deployment. Each node class has a distinct enclosure optimized for its physical position and sensor aperture requirements.

---

## 2. Enclosure Types

### 2.1 Ceiling-Corner Mount (Always-On Variants B–D, LiDAR Variants A–D)

**Purpose:** Primary position for volume sensing nodes. Ceiling corner placement maximizes coverage while minimizing shadow cones from adjacent walls.

**Geometry:**
- Two flat faces at 90° (matching room corner)
- Main PCB face angled 45° downward from horizontal
- Sensor aperture windows: IR-transparent polymer for PIR; RF-transparent polymer/foam for radar; acoustic port for microphone

**Mounting:**
- 4× M4 wall anchors, standard drywall or masonry
- Cable exit: rear face or conduit-compatible cutout

**Weatherproofing:** IP54 standard (indoor). IP65 variant available for covered outdoor areas.

**Dimensions (reference):**
- Standard (Always-On B, LiDAR A/B): 90 mm × 90 mm × 60 mm
- LiDAR node (larger module): 120 mm × 120 mm × 80 mm
- LiDAR Variant D (360° camera): 150 mm diameter × 80 mm height (circular ceiling mount)

### 2.2 Wall Low-Angle Mount (Sub-Room, Ground-Level Coverage)

**Purpose:** Secondary position for low-angle occlusion coverage. Mounted 20–50 cm above floor level.

**Geometry:**
- Flat back-plate for wall mounting
- PCB angled 0–30° upward from horizontal
- Small profile for unobtrusive installation

**Mounting:**
- 2× M4 wall anchors
- Optional conduit entry (bottom face)

**Dimensions (reference):**
- 80 mm × 60 mm × 40 mm

### 2.3 Choke-Point Mount (Always-On Variant A)

**Purpose:** Doorway or hallway crossing detection. Mounted at frame or wall surface at ~1 m height.

**Geometry:**
- Slim profile, facing across the opening
- PIR window facing travel direction
- Optional retroreflector mount on opposite wall for ToF

**Mounting:**
- 2× M4 or adhesive mounting
- Low-profile: 60 mm × 40 mm × 25 mm

### 2.4 Identity Node Enclosure

**Purpose:** Entry point or verification zone. Suitable for visible or discreet installations.

**Geometry:**
- Camera window: optical-quality clear polycarbonate or glass
- Multiple camera window positions for multi-camera variants
- Optional hardware power switch: externally accessible slider on side face

**Mounting:**
- 4× M4 anchors (standard indoor)
- Flush-mount variant for wall-box installation

**Dimensions (reference):**
- Single-camera standard: 100 mm × 80 mm × 50 mm
- Multi-camera (4 cameras): 160 mm × 120 mm × 60 mm
- 360° entry variant: 120 mm diameter × 60 mm height

---

## 3. Sensor Aperture Materials

| Sensor | Aperture Material | Notes |
|---|---|---|
| mmWave Radar (60 GHz) | ABS plastic, HDPE, or thin polycarbonate | Avoid metal. RF passes through most non-metallic plastics. |
| PIR | IR-transparent polyethylene (LDPE) | Standard PIR dome material. |
| LiDAR (905 nm / 1550 nm) | Optical-quality polycarbonate or glass | Anti-reflection coating improves range. |
| Event Camera | Optical-quality polycarbonate or glass | Same as LiDAR. |
| Conventional Camera | Optical-quality polycarbonate | Anti-reflection coating strongly recommended. |
| Microphone | Acoustic mesh (gauze or PTFE membrane) | Waterproof PTFE for IP65+ variants. |
| Environmental | Ventilation slots with dust filter | Prevents sensor contamination while allowing air exchange. |

---

## 4. PoE and USB-C Cable Management

All ceiling-mount nodes include:
- Cable entry sleeve: 10 mm diameter, accepts standard Ethernet or USB-C cable
- Optional conduit adapter: ½" EMT conduit fitting (threaded)
- Strain relief: internal cable clamp prevents connector stress

For PoE nodes: Ethernet enters through the cable sleeve; the PoE splitter is inside the enclosure if needed, or the node PCB directly accepts PoE on the Ethernet connector.

---

## 5. CAD Files

All enclosure designs are available in STEP format:
- `mechanical/cad/ceiling_corner_mount_std.step`
- `mechanical/cad/ceiling_corner_mount_lidar.step`
- `mechanical/cad/ceiling_center_360_camera.step`
- `mechanical/cad/wall_low_angle_mount.step`
- `mechanical/cad/choke_point_mount.step`
- `mechanical/cad/identity_node_std.step`
- `mechanical/cad/identity_node_multi_cam.step`

STL exports for FDM 3D printing:
- `mechanical/stl/*.stl`

---

## 6. Production Notes

- All enclosures are designed for FDM 3D printing (PLA, PETG, ABS) for research/prototyping.
- Production enclosures: injection-molded ABS or PC-ABS, powder-coated aluminum for premium variants.
- Minimum wall thickness: 2 mm for FDM, 1.2 mm for injection-molded.
- Thread inserts (M3 heat-set) recommended for all screw holes in FDM parts.
