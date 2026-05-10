# Sensor Fusion — Multi-Modal Complementarity in Residential Sensing

**Project:** AEGIS-MESH
**Domain:** Multi-modal sensor fusion architecture for distributed residential perception
**Implementation:** Rust crates `aegis-perception`, `aegis-fusion`, `aegis-tracking`, `aegis-slam`, using OMNI-SENSE

---

## 1. Why Multi-Modal

No single sensor modality can answer all the questions a residential awareness system needs to answer. Each modality has characteristic strengths, characteristic failure modes, and characteristic information content. Multi-modal fusion is not a fallback for poor individual sensors — it is the design philosophy of the system.

The key insight is that each modality contributes a different *kind* of identification, not just a redundant measurement of the same thing. AEGIS-MESH is not "just a sensor system" — it is a **physics-based identification system** that classifies what is in the space using the physical properties of detection rather than imagery.

---

## 2. The Four Modalities and What They Identify

### 2.1 mmWave Radar

mmWave FMCW radar at 24, 60, or 77 GHz provides:

- **Presence detection:** Is something there? Threshold-based detection on range-Doppler map, implemented via `omni-sense-radar::cfar_detect_2d`.
- **Position:** Range and bearing from the sensor. Angular resolution depends on antenna aperture (typically 10–30° for ceiling-mounted antennas).
- **Velocity:** Direct radial velocity via Doppler shift. No frame-to-frame differentiation required. This makes radar the most reliable velocity sensor.
- **Activity classification via micro-Doppler:** A walking person produces a characteristic Doppler pattern — the body moves at roughly constant velocity, but the legs and arms produce periodic Doppler modulations (micro-Doppler). This pattern identifies "human walking" vs. "human stationary" vs. "pet." The micro-Doppler signature is implemented in `aegis-perception::radar_pipeline`.
- **Through-fabric and through-light-obstruction:** mmWave penetrates clothing, thin fabric, light vegetation. Objects partially obscured behind light curtains remain detectable.
- **Through-walls and through-furniture:** mmWave at 60 GHz penetrates drywall, interior walls, and fabric furniture. A person behind a couch or in an adjacent room may be detectable depending on wall material and distance.

**Failure modes:** Coarse angular resolution limits discrimination between closely-spaced objects. Static objects are invisible in motion-detection mode. Multi-path reflections off metal surfaces create ghost targets.

### 2.2 Solid-State LiDAR

Time-of-flight LiDAR with VCSEL emitters provides:

- **High-resolution geometry:** Sub-degree angular resolution, centimeter range resolution. Dense point clouds suitable for shape characterization.
- **Object sizing:** A 180 cm tall, 50 cm wide point cloud cluster is almost certainly a standing human. A 40 cm tall, 60 cm wide cluster at floor level is consistent with a dog.
- **Occupancy mapping:** The LiDAR continuously builds a geometric model of the room that the coverage solver and dynamic remap system use.
- **Static-scene observability:** Unlike radar, LiDAR sees stationary objects (though it won't report them as "detection events" — they're part of the background map).
- **3D point cloud generation:** Each scan produces a dense 3D point cloud that contributes to SLAM world model construction.

**Failure modes:** Fails in steam (kitchen, bathroom), smoke (cooking or fire), heavy dust. Scan-cycle latency (50–100 ms) means it misses fast transients. Specular surfaces (mirrors, glossy floors) cause reflections.

**OMNI-SENSE integration:** `omni-sense-lidar::cluster`, `omni-sense-lidar::centroid` produce `DetectionEvent` from point clouds. The atmospheric state in `omni-sense-atmospherics` degrades LiDAR effective range under adverse indoor conditions.

### 2.3 Acoustic (Full-Echo Profiling)

Microphone arrays with full-echo analysis provide:

- **Direction-of-arrival:** MUSIC beamforming (`omni-sense-physics::MusicBeamformer`) locates the source direction of transient sounds.
- **Material identification:** The key differentiator. When an object breaks, falls, or is struck, it produces a characteristic acoustic signature. Glass breaking has a sharp broadband impulse followed by high-frequency ringing. Ceramic cracking has a different decay envelope. Wood cracking has different spectral content. Full-echo analysis (`omni-sense-physics::full_echo_profile`) extracts these features and the material classifier (`omni-sense-acoustic::classify_material`) identifies the material class.
- **Event classification:** Footsteps, door operations, water flow, appliance sounds — each has a recognizable acoustic pattern. The event classifier identifies these without needing to record or transmit audio.
- **Through-wall propagation:** Sound passes through walls and doors, providing awareness of events in adjacent spaces that optical sensors cannot see.
- **Full echo analysis:** Unlike simple sound level detection, full echo analysis examines the entire reflected waveform. This reveals material density and physical properties — glass break vs. plastic drop, wood vs. ceramic — providing identification capability that no imaging sensor can match.

**Failure modes:** Ambient noise (HVAC, traffic, appliances) degrades detection of quiet events. Highly reverberant spaces (tile, glass, bare concrete) make direction estimation difficult.

**Privacy note:** OMNI-SENSE's acoustic pipeline processes on-device. The `AcousticDriver::poll_frame()` output goes into `omni-sense-acoustic::full_echo_pipeline` on the node's MCU. Only the classification result (`MaterialMatch`, `EventType`, `DirectionOfArrival`) is transmitted over the mesh — never raw audio waveforms by default. Raw audio storage and streaming are user-configurable.

### 2.4 PIR (Passive Infrared)

Binary motion detection via infrared radiation change:

- **Low-cost, low-power choke-point sensing:** A PIR at a doorway reliably detects any warm-body crossing. Cost: under $5 per unit.
- **Choke-point traffic counting:** Multiple PIR sensors at a doorway can detect direction of crossing.

**Failure modes:** Binary output — no position, no classification. Fails on stationary warm bodies. No through-obstruction capability.

**Role:** Exclusively for choke-point nodes. The PIR triggers a `BinaryEvent` which activates higher-power sensors for refinement.

### 2.5 Neuromorphic Event Sensor (Optional)

Asynchronous pixel-level change detection:

- **Microsecond latency:** Detects fast motion edges before the LiDAR scan cycle completes.
- **Sparse output:** Only moving objects generate events. Background is silent.
- **Fast-event trigger:** Triggers LiDAR wakeup for high-resolution scan, saving power during quiet periods.
- **No frame latency:** Unlike conventional cameras, event sensors have no exposure time or frame rate constraint. Each pixel operates independently and fires when brightness changes.

**Integration with LiDAR:** The event sensor acts as a fast trigger for LiDAR activation. When rapid motion is detected (event rate exceeds threshold), the LiDAR wakes and performs a high-resolution scan. This provides the geometric detail of LiDAR with the temporal responsiveness of an event sensor.

---

## 3. What Multi-Modal Fusion Identifies That Cameras Don't

A common misconception is that camera-free sensing is a limitation — that you sacrifice identification capability by removing cameras. The reality is more nuanced.

**What cameras provide:** Visual identification of specific individuals (face recognition), reading of text (license plates), and fine-grained appearance details (what someone is wearing).

**What physics-based multi-modal sensing provides:** Object class, activity class, and material class derived from the physical properties of detection. This is *different from and complementary to* camera identification.

Consider a track detected at the front door:
- Radar micro-Doppler → "Human gait pattern, 1.5 m/s approach velocity"
- LiDAR geometry → "180 cm height, 50 cm width, upright posture"
- Acoustic → "Footstep pattern consistent with hard-sole shoes, direction of arrival matching track bearing"
- Combined classification → **HumanApproaching**, confidence 0.94

Now consider an anomaly:
- Same radar profile, same LiDAR geometry
- Acoustic → "Metallic impact, high-energy, short rise time, frequency consistent with metal blade"
- Combined classification → **HumanApproaching with ObjectClassification: MetallicEdgedObject**

The multi-modal system has identified not just that a person is approaching, but that they are carrying something with an acoustic signature consistent with a metal blade. This is not face recognition — it is physics-based threat characterization.

The visual identification layer (camera, opt-in) adds: "This person is John Doe." The multi-modal sensing layer adds: "This person is an approaching human of typical adult size carrying a metal object approaching at 1.5 m/s." Both are identification — at different levels of specificity and via different physical mechanisms.

---

## 4. The Fusion Architecture

Implemented in `aegis-fusion` using OMNI-SENSE fusion primitives.

### 4.1 Per-Node Multi-Modal Fusion (aegis-fusion::multimodal_fusion)

Within a single node, radar + LiDAR + acoustic observations are fused using `omni-sense-fusion::TrackLevelFuser`. The output is one `DetectionEvent` per target per node, not one per modality.

This is important: the mesh network sees track-level data, not raw detections. Bandwidth is proportional to the number of tracked objects, not the sensor data rate.

**Data flow within node:**

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    SINGLE NODE FUSION PIPELINE                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  [ Radar Pipeline ]                                                          │
│  ├── Range-Doppler processing                                                │
│  ├── CFAR detection                                                          │
│  ├── Micro-Doppler classification                                            │
│  └── Output: RadarDetection { range, bearing, velocity, activity_class }     │
│                                                                              │
│  [ LiDAR Pipeline ]                                                          │
│  ├── Point cloud clustering                                                  │
│  ├── Centroid extraction                                                     │
│  ├── Size estimation                                                         │
│  └── Output: LiDARDetection { position, size_class, confidence }              │
│                                                                              │
│  [ Acoustic Pipeline ]                                                        │
│  ├── Beamforming (DOA estimation)                                            │
│  ├── Full-echo analysis                                                      │
│  ├── Material classification                                                 │
│  └── Output: AcousticDetection { direction, material_class, event_type }     │
│                                                                              │
│  [ Per-Node Fusion ]                                                         │
│  ├── Track-level fusion (omni-sense-fusion::TrackLevelFuser)                 │
│  ├── Association of modalities to same target                                │
│  └── Output: DetectionEvent { fused_track, class, confidence }               │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 4.2 Mesh-Level Fusion (aegis-fusion::mesh_fusion)

Multiple nodes' track outputs are fused using JPDA association (`omni-sense-fusion::JpdaTracker`) and Covariance Intersection (`omni-sense-fusion::covariance_intersection`). CI fusion is correct even when cross-node correlations are unknown.

**Multi-node observation of same target:**

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    MESH-LEVEL FUSION                                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  Node A (Living Room North) ──┐                                              │
│  Node B (Living Room South) ──┼── Observe same target                        │
│  Node C (Living Room East) ───┘                                              │
│                                                                              │
│  Each node produces: DetectionEvent with position estimate and covariance    │
│                                                                              │
│  [ Mesh Fusion ]                                                             │
│  ├── JPDA association: match detections to same track                        │
│  ├── Covariance Intersection: fuse position estimates                        │
│  └── Output: FusedTrack { global_position, velocity, class, confidence }     │
│                                                                              │
│  Benefits of multi-node observation:                                         │
│  ├── Reduced position uncertainty (multiple angles)                          │
│  ├── Occlusion resilience (one node blocked, others see)                     │
│  └── Classification confidence boost (multiple modalities agree)             │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 4.3 Classification Fusion (aegis-fusion::classification)

The `ResidentialObjectClass` is assigned by combining:

| Modality | Contribution to Classification |
|----------|--------------------------------|
| Radar micro-Doppler | Activity class (walking, stationary, running, falling) |
| LiDAR geometry | Size class (adult, child, pet, object) |
| Acoustic full-echo | Material class (biological, metallic, glass, ceramic) |
| Combined | `ResidentialObjectClass` (Human, Pet, FallenPerson, UnknownObject, etc.) |

**Classification hierarchy:**

```
ResidentialObjectClass
├── Human
│   ├── HumanWalking
│   ├── HumanRunning
│   ├── HumanStationary
│   ├── HumanFallen
│   └── HumanUnknown
├── Pet
│   ├── Dog
│   ├── Cat
│   └── PetUnknown
├── Vehicle
│   ├── CarInDriveway
│   ├── Motorcycle
│   └── Bicycle
├── Object
│   ├── Furniture
│   ├── Appliance
│   ├── Door
│   └── UnknownObject
└── Unclassified
```

This is the output that makes AEGIS-MESH fundamentally different from "motion sensors."

### 4.4 PentaTrack Integration (aegis-tracking::pentatrack_bridge)

`FusedTrack` objects from the mesh-level fusion are fed to PentaTrack as `DetectionEvent` streams. PentaTrack maintains the predictive-center field for each track, enabling:

- **Trajectory prediction:** Where is this track heading? PentaTrack projects future positions based on velocity and historical motion patterns.
- **Anomaly detection:** Does the drift pattern indicate a fall? Sudden changes in velocity or trajectory that don't match expected motion models trigger anomaly flags.
- **Alert anticipation:** The person is predicted to reach the restricted zone in 8 seconds — alert now. Preemptive alerts before threshold crossing.
- **Object-type-aware prediction:** Different object types have different motion characteristics. A pet moves differently from a human. PentaTrack maintains per-object-type drift profiles for more accurate prediction.

**PentaTrack output:**

```
PentaTrackOutput {
    track_id: TrackId,
    position: Position3D,
    velocity: Velocity3D,
    prediction_centers: Vec<PredictionCenter>,  // Predicted future positions
    drift_profile: DriftProfile,                 // Motion model for this object type
    anomaly_flags: Vec<AnomalyFlag>,
    confidence: f32,
}
```

---

## 5. OMNI-SENSE Integration Architecture

All sensor pipelines in AEGIS-MESH use OMNI-SENSE as the sensor abstraction layer. The integration produces a unified `DetectionEvent` contract between hardware drivers and the fusion engine.

### 5.1 OMNI-SENSE Crate Mapping

| Modality | OMNI-SENSE Crate | AEGIS-MESH Consumer |
|----------|------------------|---------------------|
| mmWave Radar (FMCW) | `omni-sense-radar`, `omni-sense-drivers-mmwave` | `aegis-perception::radar_pipeline` |
| Solid-State LiDAR | `omni-sense-lidar`, `omni-sense-drivers-lidar` | `aegis-perception::lidar_pipeline` |
| Event Camera | `omni-sense-event`, `omni-sense-drivers-event` | `aegis-perception::event_pipeline` |
| Microphone Array | `omni-sense-acoustic`, `omni-sense-drivers-acoustic` | `aegis-perception::acoustic_pipeline` |
| PIR | `omni-sense-drivers-pir` | `aegis-perception::pir_pipeline` |
| ToF (1D) | `omni-sense-drivers-tof` | `aegis-perception::tof_pipeline` |
| Environmental | `omni-sense-drivers-env` | `aegis-perception::env_pipeline` |
| Camera (Variants C, D, Identity) | `omni-sense-drivers-camera` (opt-in feature) | `aegis-perception::camera_pipeline` |

### 5.2 Fusion Primitive Usage

| Fusion Stage | OMNI-SENSE Primitive | Purpose |
|--------------|----------------------|---------|
| Per-node multi-modal | `omni-sense-fusion::TrackLevelFuser` | Fuse radar + LiDAR + acoustic within node |
| Mesh-level association | `omni-sense-fusion::JpdaTracker` | Associate tracks across nodes |
| Mesh-level position fusion | `omni-sense-fusion::covariance_intersection` | Fuse position estimates without correlation knowledge |
| Atmospheric compensation | `omni-sense-atmospherics` | Adjust sensor weights for conditions |
| Time synchronization | `omni-sense-time::ClockOffsetEstimator` | Align timestamps across nodes |

### 5.3 DetectionEvent Contract

The unified output of all sensor pipelines:

```rust
pub struct DetectionEvent {
    pub timestamp: Timestamp,
    pub position: Position3D,
    pub velocity: Option<Velocity3D>,
    pub detection_class: DetectionClass,
    pub confidence: f32,
    pub node_id: NodeId,
    pub modality: ModalitySet,
    pub raw_data_path: Option<String>,  // If raw data stored locally
    pub attributes: HashMap<String, AttributeValue>,
}

pub enum DetectionClass {
    Human { activity: ActivityClass },
    Pet { species: PetSpecies },
    Vehicle { vehicle_type: VehicleType },
    Object { object_type: ObjectType },
    Unknown,
}

pub struct ActivityClass {
    pub activity: Activity,
    pub confidence: f32,
}

pub enum Activity {
    Walking,
    Running,
    Stationary,
    Sitting,
    Falling,
    Fallen,
    Unknown,
}
```

---

## 6. Atmospheric Compensation (Indoor)

Even indoors, atmospheric conditions affect sensor performance:

- **Steam** (kitchen, bathroom): Degrades LiDAR significantly. OMNI-SENSE `AtmosphericProfile::LightFog` approximates this condition.
- **Cooking smoke:** Similar to steam, primarily affects optical sensors.
- **Dust** (cleaning, renovation): Degrades LiDAR range and accuracy.
- **High humidity:** Slight effect on acoustic propagation (small correction to speed of sound).
- **Temperature variations:** Affects speed of sound, LiDAR calibration.

The fusion pipeline monitors atmospheric state estimates (from environmental sensors where present) and adjusts sensor weights accordingly:

**Fusion weight adjustment example:**

```
Normal conditions:
├── LiDAR weight: 0.4
├── Radar weight: 0.3
└── Acoustic weight: 0.3

Steam-filled bathroom:
├── LiDAR weight: 0.1  (significantly degraded)
├── Radar weight: 0.5  (unaffected)
└── Acoustic weight: 0.4  (slightly affected)

Dusty construction area:
├── LiDAR weight: 0.2  (range reduced)
├── Radar weight: 0.5  (unaffected)
└── Acoustic weight: 0.3  (unaffected)
```

In a steam-filled bathroom, acoustic + radar carry the fusion; LiDAR is down-weighted or excluded from the fused output.

---

## 7. Edge Controller as Fusion Hub

### 7.1 Role of the Edge Controller

The edge controller is the central compute node for AEGIS-MESH, analogous to the belt node in SENTINEL-WEAR. It serves as:

- **Fusion hub:** Receives detection events from all nodes, performs mesh-level fusion
- **PentaTrack host:** Runs predictive tracking for all tracks
- **SLAM processor:** Builds dense 3D world model from LiDAR + camera data
- **API server:** Serves companion app, provides real-time and historical data
- **Recording manager:** Aggregates recordings from all nodes, manages retention
- **Policy engine:** Enforces user-configured operational policies

### 7.2 Data Flow to Edge Controller

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                 AEGIS-MESH MESH NETWORK DATA FLOW                           │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  [ Sensor Nodes ]                                                           │
│  ├── Always-On Nodes (Variants A-D)                                         │
│  │   ├── Per-node fusion → DetectionEvent                                   │
│  │   └── Mesh radio (BLE/Thread/PoE) ──────────────────────────────────┐    │
│  │                                                                       │    │
│  ├── LiDAR Nodes (Variants A-D)                                         │    │
│  │   ├── Per-node fusion → DetectionEvent                               │    │
│  │   ├── Point cloud data (optional, for SLAM)                          │    │
│  │   ├── 360° camera streams (Variant D)                                 │    │
│  │   └── Mesh radio (PoE preferred for high bandwidth) ─────────────────┼──┐ │
│  │                                                                       │  │ │
│  └── Identity Nodes                                                       │  │ │
│      ├── Classification results                                          │  │ │
│      ├── Optional video streams                                          │  │ │
│      └── Mesh radio ─────────────────────────────────────────────────────┼──┼─┤
│                                                                          │  │ │
│  [ Edge Controller ] ◄──────────────────────────────────────────────────────┘ │
│  │                                                                          │
│  ├── [ Mesh-Level Fusion ]                                                 │
│  │   └── JPDA + Covariance Intersection → FusedTrack                       │
│  │                                                                          │
│  ├── [ PentaTrack ]                                                        │
│  │   └── Predictive tracking, anomaly detection                            │
│  │                                                                          │
│  ├── [ SLAM Processor ]                                                    │
│  │   ├── LiDAR point cloud processing                                     │
│  │   ├── 360° camera stitching (Variant D nodes)                           │
│  │   └── Dense 3D world model construction                                │
│  │                                                                          │
│  ├── [ API Server ]                                                        │
│  │   ├── REST endpoints for companion app                                  │
│  │   ├── WebSocket for real-time events                                   │
│  │   └── Media streaming (video, audio, point clouds)                     │
│  │                                                                          │
│  └── [ Recording Manager ]                                                 │
│      ├── Aggregate recordings from all nodes                               │
│      ├── Retention enforcement                                             │
│      └── Legal export with integrity chain                                 │
│                                                                              │
│  [ External ]                                                               │
│  ├── WiFi → Companion App (local)                                          │
│  └── Cellular → Companion App (remote, if configured)                       │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 8. SLAM Integration for Camera-Equipped Nodes

### 8.1 Which Nodes Contribute to SLAM

| Node Class | Camera? | SLAM Contribution |
|------------|---------|-------------------|
| Always-On Variant D | Optional camera | Limited — not primary SLAM source |
| LiDAR Variant C | Single camera | Visual odometry, loop closure refinement |
| LiDAR Variant D | 360° camera array | **Primary visual SLAM source** |
| Identity Node | Configurable camera | Visual capture, not SLAM primary |

### 8.2 SLAM Input Sources

**LiDAR point clouds (all LiDAR variants):**
- Each scan produces a dense 3D point cloud
- Registered against existing map via ICP or LOAM
- Provides geometric backbone for world model

**Visual odometry from camera (Variant C):**
- Camera provides keyframe images
- Features tracked frame-to-frame (ORB or FAST)
- Visual odometry provides relative pose between keyframes
- Used for refinement between LiDAR scans

**360° visual coverage (Variant D):**
- Eight (or more) cameras covering full horizontal FoV
- No blind spots at room level
- Complete visual context for every voxel
- Each camera contributes independently to visual odometry
- High-accuracy pose tracking even during rapid rotation

### 8.3 SLAM Backend

The edge controller runs the SLAM backend. Hardware requirements:

| Edge Controller Variant | SLAM Capability |
|------------------------|------------------|
| SBC (Raspberry Pi 4/5) | Basic SLAM, 2-4 camera streams |
| ARM with NPU (Jetson) | Full SLAM, 8+ camera streams, real-time |
| x86 Mini PC | Maximum SLAM, unlimited streams, GPU acceleration |

**SLAM algorithms:**

| Algorithm | Use Case | Computational Requirement |
|-----------|----------|---------------------------|
| ORB-SLAM3 | General purpose visual SLAM | Moderate |
| LIO-SAM | LiDAR-Inertial Odometry | Moderate-High |
| FAST-LIO2 | Fast LiDAR odometry | Moderate |
| RTAB-Map | RGB-D SLAM with loop closure | High |

### 8.4 Multi-Room SLAM

**Challenge:** LiDAR nodes are room-mounted. How to handle transitions between rooms?

**Approach:**

1. **Per-room SLAM maps:** Each room with a LiDAR node maintains a local SLAM map

2. **Doorway handoff:** When a track crosses a doorway (detected by choke-point nodes), the track is transferred from one room's map to another

3. **Common frame alignment:** All room maps are expressed in a common global frame established during initial calibration walk-through

4. **Loop closure across rooms:** When the same space is revisited (walk from living room to kitchen and back), loop closure corrects drift in all affected room maps

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    MULTI-ROOM SLAM ARCHITECTURE                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  [ Living Room LiDAR Node ]                                                 │
│  └── Local SLAM map (living room geometry)                                  │
│                                                                              │
│  [ Kitchen LiDAR Node ]                                                     │
│  └── Local SLAM map (kitchen geometry)                                       │
│                                                                              │
│  [ Choke-Point at Doorway ]                                                 │
│  └── Detects track crossing → triggers handoff                              │
│                                                                              │
│  [ Edge Controller ]                                                        │
│  ├── Maintains global frame                                                 │
│  ├── Aligns local maps to global frame                                      │
│  ├── Handles track handoff between rooms                                    │
│  └── Performs loop closure when space revisited                             │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 8.5 SLAM Output Integration

The SLAM map integrates with the sparse PentaTrack world model:

**Static background:** Known static structures (walls, furniture at rest) are removed from the PentaTrack detection field. Detections that project onto static voxels are classified as clutter and suppressed.

**Dynamic foreground:** Moving voxels (entities) are segmented from the static map. Their centroids are fed into `aegis-tracking::PentaTrackBridge` as detection candidates.

**Object labels from SLAM:** Object class labels from YOLO-based recognition are attached to tracked entities, improving PentaTrack's drift profile selection.

**World model persistence:** The SLAM map is persisted to disk and loaded at boot for incremental updates. Map reuse across sessions improves accuracy over time.

---

## 9. 360° Camera Contribution (LiDAR Variant D)

### 9.1 Room-Scale 360° Visual Coverage

LiDAR Variant D's 360° camera array at ceiling position provides complete visual coverage without blind spots:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    360° CAMERA ARRAY (LiDAR VARIANT D)                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  Camera Configuration:                                                       │
│  ├── 8 cameras at 45° intervals                                             │
│  ├── Each camera: ≥ 55° FoV                                                 │
│  └── Complete 360° horizontal coverage                                       │
│                                                                              │
│  Ceiling mount position:                                                    │
│  ├── Center of room preferred                                               │
│  ├── 1-2 m from ceiling                                                     │
│  └── Clear line of sight to all room areas                                  │
│                                                                              │
│  SLAM contribution:                                                          │
│  ├── Full visual odometry from 360° coverage                                │
│  ├── No dead zones regardless of motion direction                           │
│  ├── Dense texture mapping on 3D geometry                                   │
│  └── Complete visual context for forensic review                            │
│                                                                              │
│  Recording:                                                                  │
│  ├── Stitched 360° equirectangular video                                    │
│  ├── Individual camera streams preserved                                    │
│  └── Full resolution on SD card or edge controller storage                  │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 9.2 Camera Data Routing

All camera data from LiDAR Variant D routes through the edge controller:

```
8 Cameras → Internal MIPI bus → Vision Processor on Node
                                    │
                                    ├── Stitch to 360° panorama
                                    ├── Encode (H.264/H.265)
                                    └── Transmit via PoE to Edge Controller
                                            │
                                            ├── SLAM processing
                                            ├── 360° recording storage
                                            └── WiFi relay to companion app
```

**Bandwidth considerations:**

| Stream Type | Bandwidth | PoE Capability | Notes |
|-------------|-----------|----------------|-------|
| 8 × VGA H.264 | 2-4 Mbps | ✅ Handled | Standard quality |
| Stitched 360° 2K H.264 | 3-4 Mbps | ✅ Handled | Good quality panorama |
| Stitched 360° 4K H.264 | 8-15 Mbps | ✅ Handled (PoE 100 Mbps+) | High quality panorama |
| Raw 8 × 1080p | 50-100 Mbps | ✅ Handled (PoE 100 Mbps+) | Maximum quality |

PoE Ethernet (100 Mbps+) easily handles all 360° camera configurations.

---

## 10. Modality Complementarity Summary

| Dimension | mmWave Radar | LiDAR | Acoustic | PIR |
|---|---|---|---|---|
| Through-obscuration | Yes | No | Yes | No |
| Through-walls/furniture | Yes | No | Yes | No |
| Spatial resolution | Low | High | Very low | None |
| Velocity readout | Direct (Doppler) | Inferred | Direction-only | None |
| Static observability | No (motion mode) | Yes | No | No |
| Material discrimination | Limited (micro-Doppler) | No | Yes (full-echo) | No |
| Power consumption | Moderate | Higher | Low | Very low |
| Cost | Moderate | Higher | Low | Very low |
| Illumination dependence | None | None | None | Thermal |

---

## 11. LiDAR vs. Full Echo Acoustic Complementarity

### 11.1 The Fundamental Difference

**LiDAR provides high-resolution surface geometry but fails in adverse atmospheric conditions.**

**Full Echo Acoustic analyzes the entire reflected sound waveform, penetrates smoke/fog/steam, and reveals material properties.**

### 11.2 Complementary Failure Modes

| Condition | LiDAR Performance | Acoustic Performance | Fusion Strategy |
|-----------|-------------------|---------------------|------------------|
| Clear air | Excellent | Good | Both active |
| Steam (bathroom) | Severely degraded | Good | Acoustic primary |
| Smoke (cooking, fire) | Severely degraded | Moderate | Acoustic primary |
| Dust (construction) | Moderate degradation | Good | Acoustic enhanced |
| Heavy rain (outdoor) | Moderate degradation | Moderate | Both active |
| Mirrors/glossy surfaces | Specular reflection artifacts | Good | Acoustic validates |

### 11.3 The Combined Output

**Together: LiDAR for shape, Acoustic for material classification and robustness.**

Detection example in steamy bathroom:

```
Radar: "Human-shaped moving object at 2.3 m, velocity 0.8 m/s"
LiDAR: [Degraded — high noise, low confidence]
Acoustic: "Footsteps on tile, material: ceramic, direction: bearing 45°"
Fused: HumanWalking at 2.3 m, moving toward bathtub, high confidence (radar + acoustic agree)
```

The LiDAR provides precise position when conditions permit. The acoustic provides material and event context in all conditions. The radar provides through-obstruction presence and velocity. Together, they provide robust residential awareness that no single modality can achieve.

---

## 12. Event-Based Triggering

### 12.1 Event Camera as LiDAR Trigger

Event cameras provide microsecond-latency motion detection. They serve as the wake trigger for LiDAR:

```
Event Camera (always on, ~200 mW)
       │  Fast motion detected (< 1 ms)
       ▼
LiDAR Controller (MCU) — wakes LiDAR
       │
       ▼
LiDAR active (100–300 ms)
       │
       ▼
Point Cloud → Cluster → Centroid → DetectionEvent
       │
       ▼
Mesh → Edge Controller (< 50 ms transport)
       │
       ▼
Edge Controller → WiFi → Companion App
```

### 12.2 Power Efficiency

During quiet periods:
- Event camera: active (~200 mW)
- LiDAR: dormant (< 10 mW standby)
- Radar: low-duty-cycle presence detection

When motion detected:
- Event camera triggers LiDAR
- LiDAR wakes, performs high-resolution scan
- Point cloud processed, detection produced
- LiDAR returns to dormant state

This architecture provides LiDAR-class precision with event-camera-class latency and lower average power consumption.

---

## 13. Privacy Architecture in the Fusion Layer

### 13.1 Sensing Mesh Privacy

The always-on sensing mesh (Always-On nodes, LiDAR Variants A and B) has **no camera dependency at the firmware level**. The firmware binaries for these nodes do not link `omni-sense-drivers-camera`. This is a compile-time architectural constraint:

- No configuration change can enable cameras on these nodes
- Privacy is structural, not policy-based
- Users who want cameras on these node types must use different variants (D for Always-On, C or D for LiDAR)

### 13.2 Identity Layer Separation

The identity layer (camera-equipped nodes) is architecturally separate from the sensing mesh:

| Layer | Nodes | Camera Capability | Privacy Control |
|-------|-------|-------------------|-----------------|
| Always-On Sensing | A, B, C | None | Architecture-enforced |
| Always-On + Camera | D | Optional, user-configured | Software + optional hardware switch |
| LiDAR Sensing | A, B | None | Architecture-enforced |
| LiDAR + Camera | C, D | Present, user-configured | Software + optional hardware switch |
| Identity | All variants | Configurable | Software + optional hardware switch |

### 13.3 Data Flow Separation

The sensing mesh data (radar, LiDAR, acoustic) flows through one logical channel. The identity layer data (camera, biometrics) flows through a separate logical channel. The fusion engine processes both channels but maintains their separation:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    FUSION LAYER DATA SEPARATION                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  [ Sensing Mesh Channel ]                                                   │
│  ├── Radar detections → Fusion → FusedTrack                                 │
│  ├── LiDAR detections → Fusion → FusedTrack                                 │
│  ├── Acoustic events → Classification → EventLog                            │
│  └── Output: Geometric awareness, activity classification                  │
│                                                                              │
│  [ Identity Layer Channel ] (opt-in only)                                    │
│  ├── Camera frames → Local inference → Classification                       │
│  ├── Classification tags → Mesh → Edge Controller                          │
│  ├── Optional raw media → Local storage / streaming                        │
│  └── Output: Visual identification, evidence capture                        │
│                                                                              │
│  [ Fusion at Edge Controller ]                                              │
│  ├── Sensing data: always processed                                         │
│  ├── Identity data: processed only when identity layer enabled             │
│  └── Combined output: Enriched track with optional identity label          │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 14. Summary

Multi-modal sensor fusion is the architectural foundation of AEGIS-MESH. Each sensor modality contributes a unique identification capability derived from physics:

- **Radar:** Velocity and through-obstruction presence
- **LiDAR:** High-resolution geometry and object sizing
- **Acoustic:** Material identification and event classification
- **PIR:** Low-cost choke-point detection
- **Event camera:** Microsecond-latency motion triggering

The fusion architecture produces unified `DetectionEvent` outputs at the per-node level, `FusedTrack` outputs at the mesh level, and predictive tracking via PentaTrack. The SLAM subsystem integrates LiDAR and camera data for dense 3D world modeling when camera-equipped nodes are deployed.

The separation between sensing and identification layers ensures that privacy-by-design is structural in the sensing mesh while enabling full user control over the identity layer.

---

**End of Document**
