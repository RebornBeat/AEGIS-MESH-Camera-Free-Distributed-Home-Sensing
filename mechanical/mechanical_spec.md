# Mechanical Specification — AEGIS-MESH Enclosures

**Version:** 0.2 | **Status:** Research Reference Design
**License:** CERN-OHL-S v2

---

## 1. Overview

This document specifies the mechanical enclosures for AEGIS-MESH node deployment. Each node class has a distinct enclosure optimized for its physical position, sensor aperture requirements, thermal management, and environmental protection.

**Core Design Principles:**
- **Geometry-Driven:** Enclosure shape follows sensor coverage geometry, not arbitrary form factors
- **Modularity:** All enclosures support multiple configuration options via optional population
- **Environmental Appropriateness:** Indoor, covered outdoor, and harsh environment variants available
- **Thermal Awareness:** Enclosures designed for heat dissipation appropriate to node compute load
- **User Serviceability:** Access panels, removable covers, and field-replaceable components where applicable

---

## 2. Node Classes and Enclosure Strategy

### 2.1 Node Class Overview

| Node Class | Primary Position | Enclosure Type | Key Constraints |
|------------|------------------|----------------|-----------------|
| Choke Point Node | Doorways, hallways | Slim wall mount | Low profile, minimal visibility |
| Volume Node (Always-On) | Ceiling corners, open walls | Ceiling-corner mount | Wide coverage, unobtrusive |
| LiDAR Node (A-C) | Ceiling corners, center | Ceiling-corner mount | Larger aperture, thermal mgmt |
| LiDAR Node (D - 360°) | Room center, ceiling | Ceiling-center mount | Circular geometry, 360° aperture |
| Identity Node | Entry points, verification zones | Wall/flush mount | Camera window, optional switch |
| Edge Controller | Utility room, closet, shelf | Rack/shelf mount | Thermal, cable management |

### 2.2 Enclosure Design Philosophy

**For ceiling-mounted nodes:**
- Maximize coverage angle while minimizing shadow cones
- Angle PCB 30-45° from horizontal to optimize sensor field of view
- Provide cable entry that allows concealed wiring
- Consider thermal rising: heat-generating components positioned toward enclosure top

**For wall-mounted nodes:**
- Optimize for the specific sensing angle (upward for floor coverage, horizontal for presence)
- Minimize protrusion into living space
- Provide clear mounting reference for installers

**For edge controller:**
- Accept larger form factor (not jewelry-scale)
- Prioritize thermal management and cable organization
- Support multiple mounting options (wall, shelf, rack)

---

## 3. Enclosure Types — Detailed Specifications

### 3.1 Ceiling-Corner Mount — Standard

**Applies to:** Always-On Variants B, C, D (volume sensing); LiDAR Variants A, B, C

**Purpose:** Primary position for volume sensing nodes. Ceiling corner placement maximizes coverage while minimizing shadow cones from adjacent walls.

#### Geometry

```
                    Ceiling Plane
    ┌────────────────────────────────────────┐
    │                                        │
    │   ┌──────────────────────────────────┐ │
    │   │                                  │ │
    │   │     Sensor PCB                   │ │
    │   │     (angled 45° from horizontal) │ │
    │   │                                  │ │
    │   │   ┌─────────────────────┐       │ │
    │   │   │                     │       │ │
    │   │   │  Sensor Apertures   │       │ │  Front Face (90° to wall)
    │   │   │  - Radar window     │       │ │
    │   │   │  - Acoustic ports   │       │ │
    │   │   │  - Optional camera  │       │ │
    │   │   └─────────────────────┘       │ │
    │   │                                  │ │
    │   └──────────────────────────────────┘ │
    │                                        │
    └────────────────────────────────────────┘
                    Wall Corner
```

**Two flat faces at 90°** (matching room corner)

**PCB angle:** 30-45° downward from horizontal (configurable per installation)

**Dimensions (reference):**

| Node Type | Width × Depth × Height | Volume | Weight Target |
|-----------|------------------------|--------|---------------|
| Always-On B (standard) | 90 mm × 90 mm × 60 mm | ~485 cm³ | 120-180 g |
| Always-On C (enhanced acoustic) | 95 mm × 95 mm × 65 mm | ~585 cm³ | 150-220 g |
| Always-On D (+ camera) | 100 mm × 100 mm × 70 mm | ~700 cm³ | 180-250 g |
| LiDAR A/B | 120 mm × 120 mm × 80 mm | ~1150 cm³ | 300-450 g |
| LiDAR C (+ camera) | 130 mm × 130 mm × 85 mm | ~1440 cm³ | 400-600 g |

#### Sensor Apertures

| Sensor | Aperture Position | Material | Size |
|--------|-------------------|----------|------|
| mmWave radar | Front face, centered | RF-transparent polycarbonate (≤ 1 mm) | 30 × 30 mm |
| Acoustic ports (Var B) | Front face, lower | PTFE acoustic mesh | 4 × 5 mm diameter |
| Acoustic ports (Var C) | Front + sides | PTFE acoustic mesh | 6 × 5 mm diameter |
| PIR (if present) | Front face | IR-transparent LDPE dome | 15 mm diameter |
| Camera (Var D) | Front face | Optical polycarbonate, AR coating | 20 × 20 mm |
| Environmental | Bottom face | Ventilation slots | 10 × 3 mm slots × 4 |

#### Mounting

**Standard drywall installation:**
- 4× M4 wall anchors (drywall or masonry)
- Integrated mounting tabs on rear face
- Mounting holes: 10 mm from each corner

**Conduit installation:**
- Optional conduit cutout: ½" EMT fitting (threaded)
- Conduit adapter plate available as separate accessory

**Cable exit:**
- Rear face: 10 mm cable sleeve (standard Ethernet or power cable)
- Alternative: bottom edge exit for conduit runs

#### Thermal Management

**Heat sources (Always-On nodes):**
- MCU: 50-200 mW continuous
- mmWave radar: 100-500 mW active
- Camera (Variant D): 200-500 mW active

**Thermal design:**
- Passive convection through vent slots
- Enclosure material: ABS (thermal conductivity 0.2 W/m·K)
- Maximum surface temperature: 45°C at 25°C ambient
- No active cooling required for standard variants

#### Weatherproofing

| Rating | Indoor | Covered Outdoor | Exposed Outdoor |
|--------|--------|-----------------|-----------------|
| IP54 | ✅ Standard | ✅ Standard | ❌ Not suitable |
| IP65 | Optional upgrade | ✅ Recommended | ⚠️ Limited suitability |
| IP66 | N/A | Optional | ✅ Required |

**IP65/66 variants:**
- Gasketed enclosure halves
- Watertight cable gland
- Sealed sensor apertures (radar window sealed, acoustic mesh is waterproof PTFE)

---

### 3.2 Ceiling-Corner Mount — LiDAR Variants (Extended)

**Applies to:** LiDAR Variants A, B, C

**Additional considerations beyond standard ceiling-corner mount:**

#### Larger Enclosure Requirements

LiDAR modules (Livox MID-360 class) require:
- Larger internal volume: 120-130 mm × 120-130 mm × 80-90 mm
- Mounting points for LiDAR module (typically 4× M3)
- Clear aperture for LiDAR emission (no obstruction in 90° cone)
- Event camera aperture alongside LiDAR

#### Sensor Apertures (LiDAR Node)

| Sensor | Aperture Position | Material | Size |
|--------|-------------------|----------|------|
| Solid-state LiDAR | Front face, centered | Optical glass, AR coating | 50 × 50 mm |
| Event camera | Front face, side | Optical polycarbonate | 15 × 15 mm |
| mmWave radar (Var B) | Front face, lower | RF-transparent polymer | 30 × 30 mm |
| Camera (Var C) | Front face, opposite event cam | Optical polycarbonate | 25 × 25 mm |

#### Thermal Management (LiDAR Nodes)

**Heat sources:**
- LiDAR module: 5-10 W active, 15 W peak
- Event camera: 0.2-0.5 W
- MCU: 0.1-0.5 W
- Total active: 5-11 W

**Thermal design for LiDAR variants:**
- Thermal pad between LiDAR module and enclosure wall
- Ventilation slots on top and sides (if IP54)
- For IP65+: sealed enclosure with aluminum heat spreader to exterior
- Maximum surface temperature: 50°C at 25°C ambient

**Thermal modeling required before production for:**
- LiDAR Variant C (+ camera) with sustained operation
- Enclosed installations with limited airflow

---

### 3.3 Ceiling-Center Mount — 360° LiDAR Node (Variant D)

**Applies to:** LiDAR Variant D (360° camera array)

**Purpose:** Room-center ceiling position for complete 360° coverage without occlusion from wall corners.

#### Geometry

**Circular dome enclosure:**
- Base diameter: 150 mm
- Height: 80-100 mm
- Form: Dome rising from flat ceiling plate

```
        Ceiling Plane
    ┌────────────────────────────────────────┐
    │                                        │
    │    ┌───────────────────────────────┐   │
    │    │                               │   │
    │    │   ┌───────────────────────┐   │   │
    │    │   │                       │   │   │
    │    │   │    Central LiDAR      │   │   │
    │    │   │    (Livox MID-360)     │   │   │
    │    │   │                       │   │   │
    │    │   └───────────────────────┘   │   │
    │    │                               │   │
    │    │   Camera Array (8 cameras)    │   │
    │    │   arranged in circle          │   │
    │    │   ↓ ↓ ↓ ↓ ↓ ↓ ↓ ↓            │   │
    │    │                               │   │
    │    └───────────────────────────────┘   │
    │                                        │
    └────────────────────────────────────────┘
              360° Horizontal Coverage
```

#### Camera Aperture Ring

**8 cameras at 45° intervals:**
- Each camera requires optical window
- Windows arranged in ring around dome perimeter
- Overlapping coverage between adjacent cameras (each needs ≥ 55° FoV)

**Window arrangement:**
```
    ┌─────────────────────────────────────┐
    │                                     │
    │       [Cam 0°]      [Cam 45°]       │
    │                                     │
    │  [Cam 315°]              [Cam 90°]  │
    │                                     │
    │       LiDAR Central Aperture        │
    │                                     │
    │  [Cam 270°]             [Cam 135°]  │
    │                                     │
    │       [Cam 225°]    [Cam 180°]      │
    │                                     │
    └─────────────────────────────────────┘
```

#### Dimensions (Reference)

| Parameter | Value |
|-----------|-------|
| Base diameter | 150 mm |
| Dome height | 80-100 mm |
| Camera window size | 15 × 15 mm each (× 8) |
| LiDAR aperture diameter | 50 mm |
| Internal volume | ~1.4 liters |

#### Mounting

**Ceiling attachment:**
- Central ceiling plate: 160 mm diameter
- 4× M6 anchors into ceiling joist or drywall anchor rated for 2 kg
- Cable exit through central hole or conduit

**Service access:**
- Dome cover removable (twist-lock or screws)
- Access to all cameras for cleaning
- LiDAR module accessible for replacement

#### Thermal Management (360° LiDAR Node)

**Heat sources:**
- LiDAR: 5-10 W
- 8 × cameras: 1-2 W
- Event camera: 0.2-0.5 W
- MCU/SBC (if local processing): 1-5 W
- **Total: 7-18 W**

**Thermal design:**
- **Passive cooling insufficient for sustained operation**
- Aluminum dome exterior (acts as heat sink)
- Internal heat spreader (aluminum plate under LiDAR)
- Optional: small silent fan (if IP54 indoor)
- For IP65: external fins on dome exterior

**Thermal budget:**
- Maximum sustained operation: 2-4 hours at full power
- Typical operation: sparse mode (low power) with periodic high-power bursts
- Firmware throttles camera frame rate if enclosure exceeds 50°C

---

### 3.4 Wall Low-Angle Mount — Sub-Room Coverage

**Applies to:** All node types for under-furniture and ground-level coverage

**Purpose:** Secondary position for low-angle occlusion coverage. Mounted 20–50 cm above floor level.

#### Geometry

**Low-profile rectangular enclosure:**
- Flat back-plate for wall mounting
- PCB angled 0–30° upward from horizontal
- Sensor apertures on angled face

```
            Wall Surface
    ┌────────────────────────────────────────┐
    │                                        │
    │   ┌────────────────────────────────┐   │
    │   │                                │   │
    │   │   PCB (angled 0-30° upward)    │   │
    │   │   ┌────────────────────────┐   │   │
    │   │   │  Sensor Apertures      │   │   │
    │   │   │  - Radar               │   │   │
    │   │   │  - ToF                 │   │   │
    │   │   │  - Optional camera     │   │   │
    │   │   └────────────────────────┘   │   │
    │   │                                │   │
    │   └────────────────────────────────┘   │
    │                                        │
    └────────────────────────────────────────┘
              Floor Level
```

#### Dimensions (Reference)

| Parameter | Value |
|-----------|-------|
| Width | 80 mm |
| Height | 60 mm |
| Depth | 40 mm |
| Internal volume | ~190 cm³ |
| Weight target | 80-120 g |

#### Mounting

- 2× M4 wall anchors
- Optional: adhesive mounting strip (for tile/glass)
- Optional conduit entry (bottom face)

#### Sensor Apertures

| Sensor | Aperture Position | Material |
|--------|-------------------|----------|
| mmWave radar | Angled face, centered | RF-transparent polymer |
| ToF/LiDAR | Angled face | Optical polycarbonate |
| Camera (optional) | Angled face | Optical polycarbonate |

#### Use Cases

- Under-sofa coverage in living rooms
- Behind-furniture detection
- Pet tracking (ground-level presence)
- Fall detection supplement (ankle-height coverage)

---

### 3.5 Choke-Point Mount — Always-On Variant A

**Applies to:** Always-On Variant A (minimal choke-point node)

**Purpose:** Doorway or hallway crossing detection. Mounted at frame or wall surface at ~1 m height.

#### Geometry

**Slim profile enclosure:**
- Faces across the doorway opening
- PIR window centered on travel direction
- Optional ToF emitter/receiver for ranging

```
         Doorway Opening
    ┌────────────────────────────────────────┐
    │                                        │
    │  ┌────────┐              ┌────────┐   │
    │  │  Node  │  ◄──────────►│ Retro- │   │
    │  │  Unit  │   ToF Beam   │ reflec-│   │
    │  │        │              │ tor    │   │
    │  └────────┘              └────────┘   │
    │    Wall                    Wall       │
    │                                        │
    └────────────────────────────────────────┘
              Floor Level
```

#### Dimensions (Reference)

| Parameter | Value |
|-----------|-------|
| Width | 60 mm |
| Height | 40 mm |
| Depth | 25 mm |
| Internal volume | ~60 cm³ |
| Weight target | 40-60 g |

#### Mounting Options

**Standard:**
- 2× M4 wall anchors
- Or: adhesive mounting strip

**Door frame:**
- Thin profile variant: 50 mm × 30 mm × 15 mm
- Adhesive mount to door frame edge

**Optional retroreflector:**
- Mounted on opposite wall/frame
- 30 mm × 30 mm reflective patch
- Enables ToF ranging across doorway

#### Sensor Apertures

| Sensor | Position | Size |
|--------|----------|------|
| PIR | Front face, centered | 10 mm diameter dome |
| ToF emitter (optional) | Front face, side | 5 mm diameter |
| ToF receiver (optional) | Front face, adjacent | 5 mm diameter |

#### Weatherproofing

- IP54 indoor standard
- IP65 variant for covered exterior doorways

---

### 3.6 Identity Node Enclosure — Entry Point / Verification Zone

**Applies to:** All Identity Node variants

**Purpose:** Entry point identification, visitor verification, access control sensing

#### Geometry

**Wall-mount rectangular enclosure:**
- Camera window: optical-quality clear polycarbonate or glass
- Multiple camera window positions for multi-camera variants
- Optional hardware power switch: externally accessible slider on side face

```
         Front View                    Side View
    ┌─────────────────────┐      ┌─────────────────┐
    │                     │      │                 │
    │   ┌─────────────┐   │      │  ┌───────────┐  │
    │   │   Camera    │   │      │  │  Camera   │  │
    │   │   Window    │   │      │  │  Window   │  │
    │   └─────────────┘   │      │  └───────────┘  │
    │                     │      │                 │
    │   ┌─────────────┐   │      │  [HW Switch]    │ ← Optional
    │   │ Optional    │   │      │                 │
    │   │ Additional  │   │      │  PCB            │
    │   │ Camera      │   │      │                 │
    │   └─────────────┘   │      │                 │
    │                     │      └─────────────────┘
    └─────────────────────┘
```

#### Dimensions (Reference)

| Variant | Dimensions | Weight Target |
|---------|------------|---------------|
| Single-camera standard | 100 mm × 80 mm × 50 mm | 150-200 g |
| Dual-camera | 140 mm × 90 mm × 55 mm | 200-280 g |
| Multi-camera (4 cameras) | 160 mm × 120 mm × 60 mm | 300-400 g |
| 360° entry variant | 120 mm diameter × 60 mm height | 250-350 g |

#### Camera Window Configurations

**Single-camera (Variant A-C):**
- Centered optical window: 25 × 25 mm minimum
- Material: Optical polycarbonate, AR coating recommended
- For IR night vision: IR-transparent visible-opaque coating available

**Multi-camera (Variant D):**
- 4 windows: each 20 × 20 mm
- Arranged at 0°, 90°, 180°, 270° horizontal
- Covering entry approach from all angles

**360° entry variant:**
- Circular window ring: 8 cameras at 45° intervals
- Each window: 15 × 15 mm
- Continuous 360° coverage of entry zone

#### Optional Hardware Power Switch

**Location:** Side face, near bottom edge

**Types supported:**
- SMD slider switch (Alps SSSS811101)
- Snap-action push switch (Omron D2JW-01K13H)
- Rotary switch (user-customizable)

**Operation:**
- DISABLED position: physically cuts power to camera rail
- ENABLED position: camera powered normally
- Firmware reads switch state at boot

**Enclosure integration:**
- Cutout in enclosure side wall
- Gasket around switch shaft (IP65 variants)
- Switch clearly labeled: "CAMERA ON/OFF"

#### Mounting

**Standard indoor:**
- 4× M4 anchors
- Wall box compatibility: designed to fit standard US/UK/EU electrical boxes

**Flush-mount:**
- Recessed enclosure body
- Only front face visible
- Requires wall cutout

**Outdoor covered:**
- IP65 variant with sealed cable gland
- Heated window option (to prevent fog/frost)
- UV-stabilized enclosure material

#### Privacy Indicator

**Visual status:**
- LED indicator above camera window
- Red: camera disabled (switch OFF or software disabled)
- Green: camera active
- Amber: camera enabled but not currently capturing
- Off: system unpowered

---

### 3.7 Edge Controller Enclosure

**Applies to:** Edge Controller Variants A, B, C

**Purpose:** Central compute, storage, and external connectivity hub. Contains the only WiFi and cellular radios in the entire AEGIS-MESH system.

**Critical distinction:** The edge controller is NOT jewelry-scale. It is a fixed installation unit, typically in a utility room, closet, or central shelf location.

#### Geometry

**Rack-mountable rectangular enclosure:**
- Standard 19" rack width for variant C (x86 high-performance)
- Smaller desktop/shelf form factor for variants A, B

```
    Front View (Desktop/Shelf)
    ┌─────────────────────────────────────────┐
    │                                         │
    │   ┌───────────────────────────────────┐ │
    │   │  Status LEDs                      │ │
    │   │  ○ Power   ○ System   ○ Network   │ │
    │   └───────────────────────────────────┘ │
    │                                         │
    │   ┌───────────────────────────────────┐ │
    │   │                                   │ │
    │   │   Internal PCB / SBC              │ │
    │   │   - WiFi module                   │ │
    │   │   - Cellular module (optional)    │ │
    │   │   - Storage (SSD/microSD)         │ │
    │   │                                   │ │
    │   └───────────────────────────────────┘ │
    │                                         │
    │   ┌───────────────────────────────────┐ │
    │   │  Ventilation Slots (both sides)   │ │
    │   │  |||||||||||||||||||||||||||||||| │ │
    │   └───────────────────────────────────┘ │
    │                                         │
    └─────────────────────────────────────────┘

    Rear View
    ┌─────────────────────────────────────────┐
    │                                         │
    │   ┌───┐ ┌───┐ ┌─────────────────────┐  │
    │   │USB│ │Eth│ │ Cellular Antenna    │  │
    │   │   │ │   │ │ WiFi Antenna        │  │
    │   └───┘ └───┘ └─────────────────────┘  │
    │                                         │
    │   ┌───────────────────────────────────┐ │
    │   │  Power In (USB-C PD or AC)       │ │
    │   └───────────────────────────────────┘ │
    │                                         │
    └─────────────────────────────────────────┘
```

#### Dimensions (Reference)

| Variant | Form Factor | Dimensions | Weight Target |
|---------|-------------|------------|---------------|
| A — SBC (Raspberry Pi class) | Desktop/shelf | 150 mm × 100 mm × 40 mm | 300-500 g |
| B — ARM-NPU (Jetson class) | Desktop/shelf | 180 mm × 120 mm × 50 mm | 500-800 g |
| C — x86 Mini PC | Desktop or 1U rack | 200 mm × 150 mm × 45 mm | 800-1500 g |

#### Connectivity Interfaces (Enclosure Features)

| Interface | Position | Notes |
|-----------|----------|-------|
| USB-C (power + data) | Rear face | PD 5-20V supported |
| Ethernet (PoE option) | Rear face | 1 Gbps, PoE 802.3af/at |
| WiFi antenna | Rear or top | U.FL connector to external antenna |
| Cellular antenna | Rear face | U.FL connector to external antenna |
| Status LEDs | Front face | Power, System, Network, Storage |
| Reset button | Front face | Recessed to prevent accidental press |

#### Thermal Management

**Heat sources:**
- Variant A (SBC): 3-5 W sustained, 8 W peak
- Variant B (ARM-NPU): 5-10 W sustained, 15 W peak
- Variant C (x86): 10-25 W sustained, 40 W peak

**Thermal design:**

**Variant A (SBC):**
- Passive convection sufficient
- Ventilation slots on sides and top
- No fan required

**Variant B (ARM-NPU):**
- Passive with ventilation slots (normal load)
- Optional 40 mm silent fan (sustained high load)
- Aluminum heat spreader under SBC

**Variant C (x86):**
- Active cooling required
- 60-80 mm silent fan
- Large ventilation apertures
- Aluminum enclosure acts as heat sink
- Mounting position: allow 10 cm clearance on all sides

#### Mounting Options

**Desktop/shelf:**
- Rubber feet on bottom
- VESA mount compatible (100 mm pattern) for wall mounting behind monitors

**Wall mount:**
- Keyhole slots on rear face
- Cable management clips

**Rack mount (Variant C):**
- 1U rack ears included
- Standard 19" rack compatibility

#### Storage

- Internal: microSD card slot (rear access)
- Variant C: 2.5" SSD bay (internal, tool-free access)
- Optional: external USB drive support

#### Weatherproofing

- IP20 (indoor, not dust-sealed)
- IP40 variant available for dusty environments
- No outdoor-rated variant (indoor use only)

---

## 4. Sensor Aperture Materials — Detailed Specifications

### 4.1 Material Selection by Sensor Type

| Sensor | Wavelength/Frequency | Recommended Material | Thickness | Transmission |
|--------|---------------------|----------------------|-----------|---------------|
| mmWave radar (60 GHz) | 5 mm | HDPE, ABS, polycarbonate | ≤ 2 mm | > 95% |
| PIR (8-14 µm IR) | 8-14 µm | LDPE, IR-transparent polymer | 0.5-1 mm | > 80% |
| LiDAR (905 nm) | 905 nm | Optical polycarbonate, glass | 1-3 mm | > 92% |
| LiDAR (1550 nm) | 1550 nm | Optical glass (low water absorption) | 2-4 mm | > 95% |
| Event camera | Visible + NIR | Optical polycarbonate | 1-2 mm | > 90% |
| Conventional camera | Visible | Optical polycarbonate, glass | 1-3 mm | > 92% |
| Acoustic | N/A | PTFE membrane, acoustic mesh | 0.1-0.5 mm | N/A |
| Environmental | N/A | Ventilation slots + filter | N/A | N/A |

### 4.2 Aperture Design Guidelines

**RF-transparent materials (mmWave radar):**
- Avoid metal in aperture zone (creates RF shadow)
- Use non-metallic screws or plastic clips within 20 mm of aperture
- No conductive coatings on aperture material

**Optical apertures (LiDAR, cameras):**
- Anti-reflection coating recommended for LiDAR (increases effective range by 10-15%)
- IR-cut coating for cameras (if color accuracy needed in daylight)
- Hydrophobic coating for outdoor variants (water shedding)

**Acoustic apertures:**
- PTFE membrane: waterproof, acoustic-transparent
- Acoustic mesh: IP54 rating, not fully waterproof
- Port geometry affects frequency response; follow microphone manufacturer guidance

### 4.3 Environmental Protection for Apertures

| IP Rating | Radar | Optical | Acoustic |
|-----------|-------|---------|----------|
| IP54 | Standard polymer | Standard polymer | Acoustic mesh |
| IP65 | Sealed polymer | Sealed glass | PTFE membrane |
| IP66 | Sealed polymer + gasket | Sealed glass + gasket | Welded PTFE |
| IP67 | Sealed polymer + gasket | Sealed glass + gasket | Welded PTFE |

---

## 5. Hypoallergenic Materials for Skin Contact

**Applicable to:** All nodes with potential skin contact (wearable testing, handling during installation)

### 5.1 Compliant Materials

| Material | Application | Notes |
|----------|-------------|-------|
| Grade 5 titanium (Ti-6Al-4V) | Premium enclosures, mounting brackets | Highest biocompatibility, premium cost |
| 316L surgical stainless steel | Standard metal enclosures | Industry standard, widely accepted |
| Medical-grade silicone (ISO 10993) | Gaskets, sealing, flexible components | For IP65+ variants |
| Hard-coat anodized aluminum | Aluminum enclosures | Coating must be Type III, > 25 µm |
| Polycarbonate | Plastic enclosures | Hypoallergenic by default |
| PETG | 3D-printed prototypes | Generally hypoallergenic |
| ABS | Production enclosures | Generally hypoallergenic |

### 5.2 Non-Compliant Materials (Do Not Use for Skin Contact)

- 6061 / 7075 aluminum (uncoated or poorly anodized)
- Standard (non-surgical) stainless steel
- Brass
- Zinc die-cast
- Nickel-plated components

---

## 6. Thermal Management — Design Guidelines

### 6.1 Thermal Design by Node Type

| Node Type | Power (Active) | Cooling Strategy | Notes |
|-----------|----------------|------------------|-------|
| Always-On A (choke) | 100-500 mW | Passive | No thermal design needed |
| Always-On B (volume) | 500-1500 mW | Passive vents | Minor thermal considerations |
| Always-On D (+ camera) | 600-2500 mW | Passive vents | Increased thermal budget |
| LiDAR A/B | 2-8 W | Passive + heat spreader | Thermal pad to enclosure |
| LiDAR C (+ camera) | 3-12 W | Passive + heat spreader | May require fan for sustained use |
| LiDAR D (360°) | 6-25 W | Active (fan) + heat spreader | Fan required |
| Identity Node | 0.5-3 W | Passive | Adequate for most cases |
| Edge Controller A | 3-5 W | Passive | Ventilation slots |
| Edge Controller B | 5-15 W | Passive + optional fan | Fan for sustained NPU use |
| Edge Controller C | 10-40 W | Active (fan) | Fan mandatory |

### 6.2 Thermal Interface Materials

| Application | Material | Thermal Conductivity | Notes |
|-------------|----------|---------------------|-------|
| LiDAR to enclosure | Silicone thermal pad | 2-6 W/m·K | 0.5-1 mm thickness |
| SBC to heat spreader | Graphite sheet | 10-25 W/m·K | Very thin (0.1-0.3 mm) |
| Enclosure interior to exterior | Aluminum heat spreader plate | 200+ W/m·K | For high-power nodes |
| Fan-to-vent duct | None needed | N/A | Ensure clear airflow path |

### 6.3 Thermal Monitoring

**Nodes with thermal sensors:**
- All LiDAR variants
- Edge Controller variants
- Always-On Variant D (optional)

**Configuration:**
```toml
[thermal]
monitoring_enabled = true
sensor_type = "ntc_thermistor"       # "ntc_thermistor" | "on_die"
warning_threshold_c = 45
throttle_threshold_c = 50
shutdown_threshold_c = 60
```

---

## 7. Weatherproofing and IP Ratings

### 7.1 IP Rating Definitions

| Rating | Dust | Water | Typical Use |
|--------|------|-------|-------------|
| IP20 | Not protected | Not protected | Indoor, clean environments |
| IP40 | Protected > 1 mm | Not protected | Indoor, dusty environments |
| IP54 | Dust protected | Splash resistant | Covered outdoor, humid indoor |
| IP65 | Dust tight | Water jets | Exposed outdoor (light rain) |
| IP66 | Dust tight | Powerful water jets | Exposed outdoor (heavy rain) |
| IP67 | Dust tight | Immersion (1 m, 30 min) | Harsh outdoor, washdown environments |

### 7.2 Sealing Strategies

**IP54 (standard covered outdoor):**
- Overlapping enclosure halves
- Foam gasket at seams
- Cable glands for entries

**IP65 (exposed outdoor):**
- Tongue-and-groove enclosure halves
- Silicone gasket at all seams
- Sealed cable glands (PG9 or PG11)
- Sealed sensor apertures

**IP66/67 (harsh environment):**
- Welded or ultrasonically welded seams
- O-ring gaskets (silicone or EPDM)
- Waterproof connectors (M12 circular connectors)
- Sealed sensor windows (gasketed glass or polymer)

### 7.3 Condensation Prevention

**For outdoor variants with optical apertures:**
- Desiccant packet inside enclosure
- Breather vent with Gore-Tex membrane (allows air exchange, blocks water)
- Optional heating element (for cold climate deployment)

---

## 8. PoE and USB-C Cable Management

### 8.1 Cable Entry Options

**Standard cable sleeve (all ceiling/wall nodes):**
- 10 mm diameter
- Accepts Ethernet (Cat 5e/6), USB-C, or power cable
- Strain relief: internal cable clamp

**Conduit adapter (optional):**
- ½" EMT conduit fitting (threaded)
- Convertible to cable gland for outdoor variants

**PoE splitter integration:**
- Internal PoE splitter inside enclosure (if node PCB does not accept PoE directly)
- PoE input on Ethernet connector, split to 5V power output to PCB

### 8.2 Cable Management Best Practices

**Ceiling-mount nodes:**
- Cable enters through rear face into wall cavity
- If surface-mount: use conduit to hide cable run

**Wall-mount nodes:**
- Cable enters through bottom face (water drainage)
- Use wall conduit for professional installation

**Edge controller:**
- Cable management clips on rear face
- Velcro straps included
- Optional cable organizer accessory

---

## 9. Production Notes

### 9.1 Prototyping (FDM 3D Printing)

**Materials:**
- PLA: Indoor prototypes, non-load-bearing
- PETG: Functional prototypes, outdoor testing
- ABS: Higher temperature resistance
- ASA: UV-resistant outdoor prototypes

**Print settings:**
- Minimum wall thickness: 2 mm
- Infill: 20-40% (structural parts higher)
- Print orientation: Minimize layer lines across stress points

**Post-processing:**
- Sanding for smooth finish
- Acetone vapor smoothing (ABS only)
- Primer and paint for appearance prototypes

### 9.2 Production (Injection Molding)

**Materials:**
- ABS: Standard, low cost, easy to mold
- PC-ABS: Higher impact resistance, thermal stability
- Polycarbonate: Transparent variants, high strength
- Nylon: Impact resistance, abrasion resistance

**Minimum wall thickness:** 1.2 mm for injection-molded parts

**Thread inserts:**
- M3 heat-set brass inserts for all screw holes
- Ultrasonic welding for permanent seams

**Tooling:**
- Single-cavity mold for low volume
- Multi-cavity mold for production volume

### 9.3 Assembly

**Standard assembly sequence:**
1. PCB installed into bottom half
2. Cables routed through strain relief
3. Top half installed, screws tightened
4. Mounting hardware attached
5. Function test
6. Final inspection

**Service access:**
- All ceiling-mount nodes: removable front cover
- All wall-mount nodes: removable back-plate
- Edge controller: removable top cover

---

## 10. Mounting Hardware Specifications

### 10.1 Wall Anchors

| Wall Type | Anchor Type | Load Rating | Notes |
|-----------|-------------|-------------|-------|
| Drywall (standard) | Plastic expansion | 5 kg | For lightweight nodes |
| Drywall (heavy) | Toggle bolt | 15 kg | For LiDAR nodes |
| Masonry | Concrete screw | 20 kg | For outdoor variants |
| Metal stud | Toggle anchor | 10 kg | For commercial buildings |

### 10.2 Mounting Patterns

**4-hole rectangular (ceiling-corner, identity):**
- Hole spacing: variable by node size
- Hole diameter: 5 mm (M4 screw)
- Minimum edge distance: 8 mm

**2-hole horizontal (wall low-angle, choke-point):**
- Hole spacing: 50 mm standard
- Hole diameter: 5 mm

**Keyhole slots (edge controller):**
- Slot width: 5 mm
- Slot length: 15 mm
- Screw head diameter: 8-10 mm

### 10.3 Ceiling Attachment

**For joist mounting:**
- Use wood screws (M4 × 40 mm minimum)
- Pilot hole: 3 mm

**For drywall ceiling:**
- Use toggle bolts or ceiling anchors rated for 5 kg
- Do not rely on drywall alone for LiDAR nodes

---

## 11. CAD Files and Manufacturing Data

### 11.1 STEP Files

All enclosure designs available in STEP format:

```
mechanical/cad/
├── ceiling_corner_mount_std.step
├── ceiling_corner_mount_lidar.step
├── ceiling_center_360_camera.step
├── wall_low_angle_mount.step
├── choke_point_mount.step
├── identity_node_std.step
├── identity_node_multi_cam.step
├── identity_node_360_entry.step
├── edge_controller_desktop.step
├── edge_controller_rack_1u.step
└── accessories/
    ├── conduit_adapter.step
    ├── cable_gland_pg9.step
    └── mounting_brackets.step
```

### 11.2 STL Files (3D Printing)

```
mechanical/stl/
├── ceiling_corner_mount_std.stl
├── ceiling_corner_mount_lidar.stl
├── ceiling_center_360_camera.stl
├── wall_low_angle_mount.stl
├── choke_point_mount.stl
├── identity_node_std.stl
├── identity_node_multi_cam.stl
├── edge_controller_desktop.stl
└── test_fit_jigs/
    ├── sensor_aperture_test.stl
    └── pcb_fit_test.stl
```

### 11.3 Manufacturing Drawings

2D dimensioned drawings available in PDF and DXF format:

```
mechanical/drawings/
├── ceiling_corner_mount_std.pdf
├── ceiling_corner_mount_lidar.pdf
├── ceiling_center_360_camera.pdf
├── wall_low_angle_mount.pdf
├── choke_point_mount.pdf
├── identity_node_std.pdf
├── identity_node_multi_cam.pdf
├── edge_controller_desktop.pdf
└── edge_controller_rack_1u.pdf
```

---

## 12. Quality Control for Mechanical Components

### 12.1 Dimensional Inspection

| Feature | Tolerance | Measurement Method |
|---------|-----------|-------------------|
| External dimensions | ± 0.5 mm | Calipers |
| Mounting hole spacing | ± 0.2 mm | Calipers or gauge |
| Sensor aperture position | ± 0.3 mm | Optical comparator |
| PCB fit | H7/h6 clearance fit | Go/no-go gauge |
| Enclosure seam gaps | ≤ 0.3 mm | Feeler gauge |

### 12.2 Material Verification

- Material certificates for all metal components
- Color matching verification for plastic parts
- UV resistance testing for outdoor variants (accelerated weathering)

### 12.3 Sealing Verification (IP65+)

- Pressure decay test for sealed enclosures
- Water immersion test (IP67 only)
- Gasket compression verification

---

## 13. Installation Guidelines

### 13.1 Pre-Installation Checklist

- [ ] Verify wall type and select appropriate anchors
- [ ] Check for electrical wiring behind mounting surface
- [ ] Plan cable routing before mounting
- [ ] For ceiling nodes: verify joist locations if possible
- [ ] For outdoor nodes: verify weather protection meets environment

### 13.2 Installation Steps

**Ceiling-corner mount:**
1. Mark corner position and mounting hole locations
2. Drill pilot holes
3. Install anchors
4. Route cable through enclosure
5. Mount enclosure with screws
6. Connect cable and verify operation
7. Snap on cover (if removable)

**Wall-mount (all types):**
1. Mark position at desired height
2. Level and mark mounting holes
3. Drill pilot holes
4. Install anchors
5. Mount enclosure
6. Connect cables
7. Verify operation

**Edge controller:**
1. Select location with adequate ventilation
2. Mount (shelf, wall, or rack)
3. Connect power
4. Connect Ethernet
5. Connect external antennas (WiFi, cellular)
6. Power on and verify boot

### 13.3 Post-Installation Verification

- [ ] All sensors respond in companion app
- [ ] Mesh connectivity verified (edge controller sees all nodes)
- [ ] No audible rattles or loose components
- [ ] Cable management neat and secure
- [ ] Enclosure seals intact (outdoor nodes)

---

## 14. Maintenance and Service

### 14.1 Routine Maintenance

| Task | Frequency | Notes |
|------|-----------|-------|
| Visual inspection | Monthly | Check for damage, seal integrity |
| Aperture cleaning | Quarterly | Soft cloth, no solvents |
| Cable inspection | Annually | Check for wear, strain |
| Firmware update | As released | Via companion app |

### 14.2 Field Service

**Sensor replacement:**
- All nodes designed for PCB replacement without destroying enclosure
- Standard screws and mounting hardware

**Battery replacement (nodes with battery):**
- User-replaceable battery compartment
- Standard cell sizes (18650 or LiPo pouch)

**Storage media replacement:**
- microSD card accessible without enclosure disassembly
- Hot-swap supported on edge controller

---

## 15. Document Revision History

| Version | Date | Changes |
|---------|------|---------|
| 0.1 | Initial | Basic enclosure specifications |
| 0.2 | Current | Added edge controller, expanded all sections, thermal management, IP ratings, installation guidelines |

---

**End of Mechanical Specification**
