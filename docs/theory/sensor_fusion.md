# Sensor Fusion — Multi-Modal Complementarity in Camera-Free Home Sensing

**Project:** AEGIS-MESH
**Domain:** Multi-modal sensor fusion architecture
**Implementation:** Rust crates `aegis-perception`, `aegis-fusion`, using OMNI-SENSE

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

**Failure modes:** Coarse angular resolution limits discrimination between closely-spaced objects. Static objects are invisible in motion-detection mode. Multi-path reflections off metal surfaces create ghost targets.

### 2.2 Solid-State LiDAR

Time-of-flight LiDAR with VCSEL emitters provides:

- **High-resolution geometry:** Sub-degree angular resolution, centimeter range resolution. Dense point clouds suitable for shape characterization.
- **Object sizing:** A 180 cm tall, 50 cm wide point cloud cluster is almost certainly a standing human. A 40 cm tall, 60 cm wide cluster at floor level is consistent with a dog.
- **Occupancy mapping:** The LiDAR continuously builds a geometric model of the room that the coverage solver and dynamic remap system use.
- **Static-scene observability:** Unlike radar, LiDAR sees stationary objects (though it won't report them as "detection events" — they're part of the background map).

**Failure modes:** Fails in steam (kitchen, bathroom), smoke (cooking or fire), heavy dust. Scan-cycle latency (50–100 ms) means it misses fast transients. Specular surfaces (mirrors, glossy floors) cause reflections.

**OMNI-SENSE integration:** `omni-sense-lidar::cluster`, `omni-sense-lidar::centroid` produce `DetectionEvent` from point clouds. The atmospheric state in `omni-sense-atmospherics` degrades LiDAR effective range under adverse indoor conditions.

### 2.3 Acoustic (Full-Echo Profiling)

Microphone arrays with full-echo analysis provide:

- **Direction-of-arrival:** MUSIC beamforming (`omni-sense-physics::MusicBeamformer`) locates the source direction of transient sounds.
- **Material identification:** The key differentiator. When an object breaks, falls, or is struck, it produces a characteristic acoustic signature. Glass breaking has a sharp broadband impulse followed by high-frequency ringing. Ceramic cracking has a different decay envelope. Wood cracking has different spectral content. Full-echo analysis (`omni-sense-physics::full_echo_profile`) extracts these features and the material classifier (`omni-sense-acoustic::classify_material`) identifies the material class.
- **Event classification:** Footsteps, door operations, water flow, appliance sounds — each has a recognizable acoustic pattern. The event classifier identifies these without needing to record or transmit audio.
- **Through-wall propagation:** Sound passes through walls and doors, providing awareness of events in adjacent spaces that optical sensors cannot see.

**Failure modes:** Ambient noise (HVAC, traffic, appliances) degrades detection of quiet events. Highly reverberant spaces (tile, glass, bare concrete) make direction estimation difficult.

**Privacy note:** OMNI-SENSE's acoustic pipeline processes on-device. The `AcousticDriver::poll_frame()` output goes into `omni-sense-acoustic::full_echo_pipeline` on the node's MCU. Only the classification result (`MaterialMatch`, `EventType`, `DirectionOfArrival`) is transmitted over the mesh — never raw audio waveforms.

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

### 4.2 Mesh-Level Fusion (aegis-fusion::mesh_fusion)

Multiple nodes' track outputs are fused using JPDA association (`omni-sense-fusion::JpdaTracker`) and Covariance Intersection (`omni-sense-fusion::covariance_intersection`). CI fusion is correct even when cross-node correlations are unknown.

### 4.3 Classification Fusion (aegis-fusion::classification)

The `ResidentialObjectClass` is assigned by combining:
- Radar micro-Doppler → activity class (walking, stationary, running)
- LiDAR geometry → size class (adult, child, pet, object)
- Acoustic → material class (biological, metallic, glass, ceramic)
- Combined → `ResidentialObjectClass` (Human, Pet, FallenPerson, UnknownObject, etc.)

This is the output that makes AEGIS-MESH fundamentally different from "motion sensors."

### 4.4 PentaTrack Integration (aegis-tracking::pentatrack_bridge)

`FusedTrack` objects from the mesh-level fusion are fed to PentaTrack as `DetectionEvent` streams. PentaTrack maintains the predictive-center field for each track, enabling:
- Trajectory prediction (where is this track heading?)
- Anomaly detection (does the drift pattern indicate a fall?)
- Alert anticipation (the person is predicted to reach the restricted zone in 8 seconds — alert now)

---

## 5. Atmospheric Compensation (Indoor)

Even indoors, atmospheric conditions affect sensor performance:

- **Steam** (kitchen, bathroom): Degrades LiDAR significantly. OMNI-SENSE `AtmosphericProfile::LightFog` approximates this.
- **Cooking smoke:** Similar to steam, primarily affects optical sensors.
- **Dust** (cleaning, renovation): Degrades LiDAR.
- **High humidity:** Slight effect on acoustic propagation (small correction to speed of sound).

The fusion pipeline monitors atmospheric state estimates (from environmental sensors where present) and adjusts sensor weights accordingly. In a steam-filled bathroom, acoustic + radar carry the fusion; LiDAR is down-weighted or excluded.

---

## 6. Modality Complementarity Summary

| Dimension | mmWave Radar | LiDAR | Acoustic | PIR |
|---|---|---|---|---|
| Through-obscuration | Yes | No | Yes | No |
| Spatial resolution | Low | High | Very low | None |
| Velocity readout | Direct (Doppler) | Inferred | Direction-only | None |
| Static observability | No (motion mode) | Yes | No | No |
| Material discrimination | Limited (micro-Doppler activity) | No | Yes (full-echo) | No |
| Power consumption | Moderate | Higher | Low | Very low |
| Cost | Moderate | Higher | Low | Very low |
| Illumination dependence | None | None | None | Thermal |
