# Sensing vs. Identification — The Core Architectural Separation

**Project:** AEGIS-MESH
**Domain:** Privacy architecture, sensor physics, and policy-driven perception
**Previous Title:** `camera_free_sensing.md` — renamed to better reflect the full architecture

---

## 1. Purpose

This document establishes the rigorous architectural separation between **Sensing** (continuous state observation) and **Identification** (discrete entity recognition) in AEGIS-MESH. It characterizes the failure modes of imaging-centric systems, the physics advantages of non-imaging modalities, and the architecture of the integrated but isolated Identity Layer.

**Core thesis:** Residential awareness does not require a continuous video stream. AEGIS-MESH achieves superior sensing capability (through walls, in darkness, through smoke) without the fragility and liability of cameras as primary sensors. Cameras are treated as **identification tools** — invoked when configured, isolated for security, and driven by flexible user policy.

**Critical clarification:** AEGIS-MESH does not prohibit cameras. It separates *when* and *why* cameras are used. The sensing mesh (Always-On and LiDAR nodes, Variants A and B) is camera-free by architecture. Camera capability exists as a configurable layer on specific variants (Always-On Variant D, LiDAR Variants C and D, Identity Nodes) — entirely under user control.

---

## 2. The Core Thesis: Sensing vs. Identification

### 2.1 The Fundamental Error

Traditional smart-home design conflates **Sensing** with **Identification**:

| Function | Nature | Requirements |
|----------|--------|--------------|
| **Sensing** | Continuous state observation | High temporal resolution, environmental robustness, low latency |
| **Identification** | Discrete entity recognition | High spatial resolution (imagery), episodic query |

These are fundamentally different operations with different physics, different data requirements, and different privacy implications.

### 2.2 Architectural Separation

AEGIS-MESH separates these functions physically and logically:

**Layer 1: Always-On Sensing Layer**

Non-imaging sensors running continuously to track geometry, motion, and behavior:
- **Function:** "Where is something? What is it doing? How fast is it moving?"
- **Sensors:** mmWave radar, solid-state LiDAR, acoustic arrays, PIR, ToF
- **Output:** Geometric metadata (position, velocity, classification)
- **Privacy property:** No imagery. No personally identifiable visual data.
- **Data handling:** User-configured — metadata-only by default, raw storage optional

**Layer 2: Identity Layer (Integrated but Isolated)**

Visual/biometric sensors for discrete identification events:
- **Function:** "Who is this specific individual?"
- **Sensors:** Conventional cameras, depth sensors, biometric modules
- **Output:** Classification tags, raw imagery (if configured)
- **Activation:** Policy-driven — manual, triggered, or continuous per user configuration
- **Privacy property:** Physically isolated, optional hardware kill switch

### 2.3 The Architectural Commitment

The sensing layer firmware (Always-On Variants A–C, LiDAR Variants A–B) has no dependency on camera drivers. These nodes **cannot** produce imagery — the privacy property is structural and enforced at the compile-time dependency level. Users who want camera capability must explicitly add camera-capable variants to their deployment.

---

## 3. The Failure Modes of Camera-Based Sensing

Camera-based systems fail as primary residential sensors for both technical and privacy reasons. These failures necessitate a non-imaging core, even if cameras are later added for identification.

### 3.1 Technical Failure Modes

**1. Fragility of Illumination**

Cameras require photons. In darkness, they fail unless supplemented by active illumination (IR or visible). This creates dependencies:
- Night-time coverage requires IR illuminators — additional power, additional points of failure
- Power outage affects both camera and illumination simultaneously
- Active illumination can be detected and avoided by intruders

**Contrast:** Non-imaging sensors (Radar, Acoustic) are immune to lighting conditions, providing continuous coverage regardless of day/night cycles or power outages affecting lighting.

| Lighting Condition | Camera (no IR) | Camera (with IR) | Radar | LiDAR | Acoustic |
|-------------------|----------------|------------------|-------|-------|----------|
| Full daylight | ✅ Works | ✅ Works | ✅ Works | ✅ Works | ✅ Works |
| Indoor lighting | ✅ Works | ✅ Works | ✅ Works | ✅ Works | ✅ Works |
| Darkness | ❌ Fails | ✅ Works | ✅ Works | ✅ Works | ✅ Works |
| Power outage (no lighting) | ❌ Fails | ⚠️ Requires backup power | ✅ Works | ✅ Works | ✅ Works |

**2. The Occlusion Problem**

Cameras are strictly Line-of-Sight (LoS). A camera in a living room cannot see:
- Behind a couch
- Under a table
- Through drywall
- Around a corner
- Through furniture

To cover a single room with cameras requires multiple overlapping angles — drastically increasing cost, complexity, and installation difficulty.

**Contrast:** mmWave radar penetrates drywall, fabric, and furniture. Acoustic sensing penetrates walls and doors. A non-imaging mesh "sees" through the clutter that blinds a camera, ensuring the sensing layer never loses track of an entity due to furniture placement.

**Coverage comparison for a 5×5 m room:**

| Approach | Units Required | Coverage Quality | Cost |
|----------|-----------------|------------------|------|
| Camera grid (full visual) | 4-6 cameras | 85-95% (blind spots under furniture) | High |
| Single ceiling LiDAR + radar | 1 node | 95-99% (penetrates under furniture) | Moderate |
| Radar + acoustic mesh | 2-3 nodes | 99%+ (through-wall awareness) | Low-Moderate |

**3. Atmospheric Degradation**

Cameras fail in environmental conditions common to residential hazards:

| Condition | Camera | Radar | LiDAR | Acoustic | Relevance |
|-----------|--------|-------|-------|----------|-----------|
| Steam (kitchen/bathroom) | ❌ Blinded | ✅ Works | ⚠️ Degraded | ✅ Works | Fall detection in bathroom |
| Smoke (cooking/fire) | ❌ Blinded | ✅ Works | ❌ Blinded | ✅ Works | Fire event detection |
| Dust (renovation) | ⚠️ Degraded | ✅ Works | ⚠️ Degraded | ✅ Works | Construction sites |
| Fog/mist | ❌ Blinded | ✅ Works | ⚠️ Degraded | ✅ Works | Covered outdoor areas |

**Critical insight:** During fire events — precisely when sensing is most critical — cameras are blinded by smoke. Radar and acoustic maintain functionality.

**4. Data Density and Latency**

Video streams are data-heavy. Processing 1080p/4K video for simple binary questions ("Is anyone home?") is computationally wasteful:

| Query | Camera Approach | Non-Imaging Approach |
|-------|-----------------|----------------------|
| "Is anyone home?" | Process 1080p @ 30fps | Process radar presence detection |
| Bandwidth | 3-6 Mbps per camera | < 50 Kbps total |
| Compute | GPU/CPU inference | MCU-level processing |
| Latency | 100-300 ms (frame + inference) | < 20 ms |

Non-imaging sensors emit sparse metadata (coordinates, velocity vectors, event triggers), enabling millisecond-latency reactions on modest hardware.

### 3.2 Privacy Failure Modes

**1. The Surveillance Asset**

Any camera-based system operating 24/7 creates a permanent, high-value surveillance asset:

| Risk | Camera System | AEGIS-MESH Sensing Layer |
|------|---------------|--------------------------|
| Data breach target | High (imagery) | Low (geometric metadata) |
| Subpoena target | High | Low (no visual evidence) |
| Intimate partner abuse tool | High | Low (no imagery for blackmail/surveillance) |
| Stalking tool | High | Low (cannot identify specific individuals) |

AEGIS-MESH's sensing layer avoids this by defaulting to geometric abstraction (point clouds, radar detections, acoustic events) rather than visual recording.

**2. The "Always-On" Dilemma**

To function as a sensor, a camera must be watching. To protect privacy, it must be off. This paradox is solved by AEGIS-MESH's split architecture:

| Layer | State | Privacy Property |
|-------|-------|------------------|
| Sensing Layer | Always-on | No imagery possible (structural) |
| Identity Layer | User-configured | Camera active only when configured |
| Kill switch | User-controlled | Hardware-enforced camera disable |

The "ears and reflexes" (sensing layer) are always on. The "eyes" (identity layer) are controlled by policy.

**3. Bystander Privacy**

Visitors to the home are monitored by the sensing layer as anonymous geometric objects. The sensing layer does not capture identifying imagery. In Security-First mode, if the identity layer activates on visitor arrival, the visitor may be photographed — this is the primary residual bystander-privacy risk. Recommendation: inform visitors when Security-First mode is active.

---

## 4. The Physics of Non-Imaging Sensing

AEGIS-MESH leverages the specific physical properties of light (LiDAR), radio waves (Radar), and sound (Acoustics) to create a superior sensing mesh.

### 4.1 LiDAR: Precision Geometry without Imagery

Solid-state LiDAR emits laser pulses and measures Time-of-Flight (ToF):

| Capability | Value | Privacy Relevance |
|------------|-------|-------------------|
| Geometric precision | Sub-centimeter | High spatial accuracy |
| 3D point cloud | Dense surface geometry | No texture (no face recognition) |
| Detection range | 20-80 m (sensor dependent) | Full-room coverage from single position |
| Through-obscuration | None (like camera) | Requires complementary radar |

**Privacy by resolution:** A LiDAR point cloud is geometrically rich but visually poor. It answers "Where is the object?" and "What shape is it?" without answering "Who is it?"

**Point cloud example (human silhouette):**
```
    * * *
  *       *
 *    *    *      ← Head/shoulders
 *  *   *  *
   *     *
    *   *        ← Torso
    *   *
   *     *
  *       *
 *         *      ← Legs
*           *
```

The system knows a human is present, their position, their velocity, and their pose. It does not know their identity.

### 4.2 mmWave Radar: Through-Obscuration Velocity

Frequency-Modulated Continuous Wave (FMCW) radar at 60 GHz provides:

| Capability | Value | Use Case |
|------------|-------|----------|
| Velocity detection | Direct Doppler measurement | Instant speed/direction |
| Through-material vision | Penetrates drywall, fabric, bedding | Behind-furniture detection |
| Micro-Doppler signatures | Activity classification from limb motion | Walking, sitting, falling detection |
| Atmospheric robustness | Works in steam, smoke, dust | Bathroom fall detection, fire events |

**Through-penetration examples:**

| Material | Radar Penetration | Camera Visibility |
|----------|-------------------|-------------------|
| Drywall (½") | ✅ Full | ❌ None |
| Fabric sofa | ✅ Full | ❌ None |
| Wooden door | ✅ Partial | ❌ None |
| Glass (standard) | ⚠️ Partial (reflection) | ✅ Full |
| Human body | ⚠️ Partial (absorption) | ❌ None |

### 4.3 Acoustic Sensing: Material and Event Intelligence

While LiDAR and radar excel at geometry, acoustic sensing excels at material and event classification:

**Full Echo Analysis:**

Instead of detecting "loud noise," the system analyzes frequency spectrum and decay profile:

| Event | Acoustic Signature | Classification |
|-------|-------------------|-----------------|
| Glass breaking | Sharp broadband impulse, high-frequency ring (3-8 kHz decay) | `GlassBreak` |
| Ceramic shattering | Mid-frequency impact, shorter decay | `CeramicBreak` |
| Wood cracking | Low-frequency crack, minimal ring | `WoodStress` |
| Footstep (hard floor) | Regular impact pattern, 1-2 Hz | `Footstep` |
| Water flow | Continuous broadband (faucet) vs. irregular (leak) | `WaterFlow` / `WaterLeak` |
| Door operation | Impact + metal mechanism | `DoorEvent` |

**Direction of Arrival (DOA):**

Microphone arrays with MUSIC beamforming localize sound sources:
- 4-element array: ±15° accuracy
- 6-element array: ±10° accuracy

**Privacy note:** Acoustic processing runs on-device. Only classification results (`MaterialMatch`, `EventType`, `DirectionOfArrival`) transmit over the mesh — never raw audio waveforms.

### 4.4 PIR: The Choke-Point Trigger

Passive infrared sensors detect rapid changes in heat signatures:

| Characteristic | Value | Use Case |
|----------------|-------|----------|
| Power consumption | < 1 mW | Ultra-low-power trigger |
| Detection type | Binary (motion/no motion) | Presence trigger |
| Range | 8-12 m | Doorway coverage |
| Cost | <$5 per unit | Economical choke-point deployment |

**Role:** Choke-point nodes use PIR to trigger higher-power sensors (LiDAR, radar) only when motion is detected, preserving energy and reducing false positives.

### 4.5 Environmental Sensing

| Sensor | Measurements | Purpose |
|--------|--------------|---------|
| BME688 | Temperature, humidity, pressure, VOC | Atmospheric compensation, air quality |
| SEN55 | PM2.5, PM10, temperature, humidity | HVAC integration, health monitoring |

Environmental data feeds the fusion engine for atmospheric compensation (LiDAR range degradation in steam/smoke) and provides context for smart-home integration.

---

## 5. The Identity Layer: Policy-Driven Integration

AEGIS-MESH acknowledges that visual identification is often a necessary component of a complete security system. The Identity Layer is a distinct, modular component managed by a **Policy Engine**.

### 5.1 Architectural Isolation

**Hardware isolation:**
- Identity nodes (cameras/biometrics) are physically distinct from sensing nodes
- Separate PCB, separate power domain, separate data bus
- Compromise of sensing mesh cannot leak visual data

**Network isolation:**
- Identity data handled on separate logical channel from sensing data
- Mesh protocol includes distinct message types for identity events
- Edge controller routes identity data separately from geometric tracks

**Hardware kill switch:**
- Every Identity node features a physical switch that cuts power to the sensor
- Hardware-enforced trust mechanism that overrides any software configuration
- Optional — users who prefer software-only control can omit the switch

### 5.2 Data Handling Modes (All User-Configured)

**Mode A — Metadata-Only (Default)**

- Inference runs on-device (MCU or edge controller)
- Only classification tags transit the mesh
- Example: `{"class": "KnownResident", "confidence": 0.94}`
- No raw frames stored or transmitted
- Lowest bandwidth and storage footprint

**Mode B — Local Storage**

- Raw video/images written to on-node SD card
- Available for review via AEGIS-MESH companion app
- Suitable for: security investigations, incident review, legal evidence
- Storage capacity and retention period user-configured

**Mode C — App Streaming**

- Raw or compressed video streamed to companion app over local network
- Live viewing capability
- Optional remote streaming over cellular (edge controller routes traffic)

**Mode D — Full Recording**

- Continuous or trigger-based recording
- All data captured with integrity chain (SHA-256 hash, timestamp, device ID)
- Suitable for legal proceedings, insurance claims

**Configuration:**
```toml
[identity_node.data]
mode = "metadata_only"          # "metadata_only" | "local_storage" | "app_streaming" | "full_recording"
store_raw = false
storage_target = "sd_card"
retention_days = 90
continuous_recording = false
recording_trigger = "on_detection"
enable_streaming = false
```

### 5.3 Operational Modes (Policy Engine)

**Privacy-First Mode (Default)**

- Sensing layer: Active, metadata-only
- Identity layer: Off or metadata-only (if enabled)
- Use case: Privacy-sensitive residential environments

**Security-First Mode**

- Sensing layer: Active
- Identity layer: Standby, triggered by policy events
- Trigger events: "Unknown Object in Zone", "Motion in Restricted Area", "Approach to Entry Point"
- Use case: 24/7 security monitoring, automated alerting

**Away Mode**

- Sensing layer: Maximum sensitivity
- Identity layer: Active on any detection (user-configured)
- Cellular alerts enabled (if configured)
- Use case: Absence monitoring

**Silent Mode**

- Sensing layer: Record-only, no alerts
- Identity layer: Follows configured recording policy
- Use case: Evidence gathering, minimal intrusion

### 5.4 24/7 Monitoring Implementation

In Security-First mode, the system functions as a fully automated monitoring station:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                   SECURITY-FIRST AUTOMATED WORKFLOW                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  [1. SENSING LAYER — Always On]                                              │
│  Radar/LiDAR detects motion in front yard                                    │
│  └── Establishes geometric track with velocity vector                        │
│                                                                              │
│  [2. POLICY ENGINE EVALUATION]                                               │
│  Track location = "Driveway"                                                 │
│  Track classification = "Human"                                              │
│  Policy rule: "Human in Driveway → Trigger Identity Node Front Entry"        │
│                                                                              │
│  [3. IDENTITY LAYER ACTIVATION]                                              │
│  Identity Node (front entry) activated                                       │
│  Camera captures frame(s)                                                    │
│  Local inference classifies subject                                          │
│                                                                              │
│  [4. RESULT]                                                                 │
│  Classification = "Delivery Driver"                                          │
│  Track label updated to "DeliveryDriver"                                     │
│  Action: "Turn on porch light" (integration with smart home)                 │
│  No alert sent (known visitor category)                                      │
│                                                                              │
│  [ALTERNATE RESULT]                                                          │
│  Classification = "Unknown Person"                                           │
│  Track label updated to "UnknownPerson"                                      │
│  Action: "Send alert to companion app"                                       │
│  Recording stored with integrity chain                                       │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 6. Comparative Summary: Imaging vs. Non-Imaging Architectures

| Feature | Camera-Only Architecture | AEGIS-MESH (Hybrid Architecture) |
|:--------|:-------------------------|:----------------------------------|
| **Primary Modality** | Visible Light / IR (Imagery) | LiDAR, Radar, Acoustic (Geometry) + Camera (ID) |
| **Data Output** | Video Frames (High Bandwidth) | Metadata Tracks + Identity Events (Efficient) |
| **Latency** | High (Frame rate + Processing) | Sensing: Low (ms) / ID: Moderate |
| **Operation in Darkness** | Requires Active Illumination | Sensing: Native / ID: Requires Light |
| **Through-Occlusion** | None (Line-of-Sight only) | Sensing: High (Radar/Acoustic) / ID: None |
| **Atmospheric Robustness** | Poor (Blinded by steam/smoke) | Sensing: High / ID: Poor |
| **Default Privacy** | Low (Visual recording) | Configurable (Privacy-First or Security-First) |
| **Identification** | Continuous (Inefficient) | Policy-Driven (Efficient) |
| **Liability Profile** | High (Surveillance Footage) | Segmented (Sensing data is generic; ID data is protected) |
| **Cost per Room** | High (Multiple cameras) | Moderate (1-2 sensing nodes + optional ID) |
| **Installation Complexity** | High (Wiring, angles, coverage) | Lower (Ceiling mount, wireless mesh) |

---

## 7. Node Variant Camera Capability Matrix

| Node Class | Variant | Camera Capability | Camera-Free by Architecture? |
|------------|---------|-------------------|------------------------------|
| Always-On | A | None | ✅ Yes (no camera footprint) |
| Always-On | B | None | ✅ Yes (no camera footprint) |
| Always-On | C | None | ✅ Yes (no camera footprint) |
| Always-On | D | User-configured camera module | ❌ No (camera footprint optional) |
| LiDAR | A | None (event camera only) | ✅ Yes (event camera ≠ visual capture) |
| LiDAR | B | None (event camera only) | ✅ Yes |
| LiDAR | C | User-configured camera | ❌ No |
| LiDAR | D | 4-8 camera 360° array | ❌ No |
| Identity | All | Primary sensor is camera/biometric | ❌ No (camera is the sensor) |

**Architectural commitment:** Variants with "Camera-Free by Architecture? = Yes" have firmware with no `omni-sense-drivers-camera` dependency. They **cannot** produce imagery even if a camera module is physically present — the driver is not compiled into the binary.

---

## 8. Legal and Ethical Posture

### 8.1 The "Not a Safety System" Disclaimer

AEGIS-MESH is an open-source research platform and hardware specification. It is **not** a UL-listed or CE-certified safety product.

**Capability vs. Certification:**
- The system is *capable* of 24/7 monitoring, fall detection, and intrusion alerting
- The maintainers do not certify it for life-safety applications
- No warranty of fitness for safety-critical use

**User Responsibility:**
- Deployment of automatic identification and monitoring features is the user's responsibility
- Users must ensure their configuration complies with local surveillance laws:
  - One-party vs. two-party consent for audio/video recording
  - CCTV regulations
  - Data retention requirements
  - Private property vs. public space distinctions

### 8.2 Liability Limitation

The Policy Engine provides technical capability for automation, but configuration is the user's choice. This separates the **tool** (AEGIS-MESH) from the **regulated activity** (24/7 video surveillance).

### 8.3 Privacy Compliance Advantages

Even in Security-First mode, the system offers advantages over traditional camera-based systems:

| Aspect | Traditional CCTV | AEGIS-MESH Security-First |
|--------|------------------|---------------------------|
| Recording trigger | Continuous 24/7 | Detection-triggered only |
| Empty room recording | Yes | No |
| Data minimization | Poor | Good (geometric sensing + event-triggered ID) |
| Local processing | Rare (usually cloud) | Default (edge controller) |
| User control over data | Limited | Full |

---

## 9. Civilian Transfer Applications

The sensing-layer architecture transfers directly to civilian domains:

| Domain | Application | Value Proposition |
|--------|-------------|-------------------|
| **Healthcare** | Hospital patient monitoring | Privacy-preserving fall detection without cameras |
| **Retail** | Occupancy analytics | Customer tracking without visual surveillance |
| **Smart building** | HVAC control | Presence-based automation |
| **Accessibility** | Spatial awareness for visually impaired | Haptic feedback from environmental geometry |
| **Elder care** | Aging-in-place monitoring | Privacy-respecting presence and fall detection |
| **Industrial** | Warehouse safety | Proximity alerts without video liability |
| **Research** | Human activity analysis | Privacy-compliant data collection |

---

## 10. Summary

AEGIS-MESH rejects the premise that residential awareness requires a continuous video stream. The architecture establishes a fundamental separation:

**Sensing Layer (Always-On, Non-Imaging):**
- Provides full geometric awareness through walls, in darkness, through smoke
- Structurally incapable of capturing imagery
- Low power, low latency, high reliability
- Privacy-by-architecture, not privacy-by-policy

**Identity Layer (Configurable, User-Controlled):**
- Provides visual identification when configured
- Activated by policy trigger or user request
- Physically isolated, hardware kill switch optional
- Data handling entirely user-configured

This separation allows AEGIS-MESH to scale from a privacy-respecting presence monitor to a 24/7 automated security system, adapting to the specific needs and legal constraints of the user's environment.

The architecture treats the camera not as a sensor, but as an **identification tool** — invoked when necessary, isolated for security, and driven by a flexible policy engine.
