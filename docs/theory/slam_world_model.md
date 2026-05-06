# SLAM World Model — Dense 3D Reconstruction in AEGIS-MESH

**Applies to:** LiDAR Node Variants C and D
**Implementation:** `crates/aegis-slam/`

---

## 1. Purpose

AEGIS-MESH operates in two parallel world model modes. The sparse probabilistic model (PentaTrack-based) handles all real-time alerting at low latency. The dense SLAM world model builds a geometrically complete 3D reconstruction of monitored spaces for review, forensic analysis, and legal documentation.

Both modes run simultaneously when hardware permits. Mode A (sparse) always runs. Mode B (SLAM) requires LiDAR Variant C or D and sufficient edge controller compute.

---

## 2. SLAM Architecture

### 2.1 Input Sources

**LiDAR point clouds (all camera-capable LiDAR variants):** The solid-state LiDAR provides 3D point cloud at each scan cycle. Each frame is registered against the existing map via ICP (Iterative Closest Point) or LOAM (LiDAR Odometry and Mapping).

**Visual odometry from camera (Variant C):** The conventional camera provides keyframe images. Features are tracked frame-to-frame (ORB or FAST features). Visual odometry provides relative pose between keyframes, used for refinement between LiDAR scans.

**360° visual coverage (Variant D):** Eight (or more) cameras covering full horizontal field of view. No blind spots. Provides complete visual context for every voxel in the map. Each camera contributes independently to visual odometry; the combination provides high-accuracy pose tracking even during rapid rotation.

### 2.2 SLAM Backend

The edge controller runs the SLAM backend. For a Raspberry Pi CM4 class machine:
- **LiDAR SLAM:** LIO-SAM (LiDAR-Inertial Odometry via Smoothing and Mapping) or FAST-LIO2
- **Dense reconstruction:** TSDF (Truncated Signed Distance Function) fusion using Open3D
- **Loop closure:** Scan context descriptors for LiDAR; DBoW2 for visual
- **Object recognition:** YOLOv8-nano (sufficient for residential object classes on CM4-class hardware)

For higher compute variants (NXP i.MX 8M Plus, Coral Edge TPU add-on):
- RTAB-Map (RGB-D SLAM with loop closure)
- Dense neural depth estimation from stereo camera pairs (Variant D)
- Real-time semantic segmentation

### 2.3 Map Representation

The 3D map is stored as:
- **Voxel grid:** Occupancy probability per voxel (OctoMap format). Default resolution: 5 cm.
- **Mesh:** Marching-cubes surface reconstruction for visualization (PLY/OBJ format).
- **Point cloud archive:** Raw LiDAR scans with timestamps for forensic replay.
- **Keyframe database:** Camera images with associated poses for loop closure.

Map persistence: stored on edge controller SD card. Loaded at boot for incremental update (map reuse across sessions).

---

## 3. SLAM Output Integration

The SLAM map integrates with the sparse PentaTrack world model:

**Static background:** Known static structures (walls, furniture at rest) are removed from the PentaTrack detection field. Detections that project onto static voxels are classified as clutter and suppressed.

**Dynamic foreground:** Moving voxels (entities) are segmented from the static map. Their centroids are fed into `aegis-tracking::PentaTrackBridge` as detection candidates.

**Object labels from SLAM:** Object class labels from YOLO-based recognition are attached to tracked entities, improving PentaTrack's drift profile selection.

---

## 4. 360° Camera SLAM Specifics

LiDAR Variant D's 360° camera array enables room-scale visual SLAM without dead zones.

**Stitch-free SLAM:** Rather than stitching to equirectangular first, the SLAM pipeline processes each camera's perspective image independently. Multi-camera feature matching handles camera-boundary features without the resampling artifacts of equirectangular stitching.

**Complete loop closure:** With 360° coverage, the SLAM system always has visual features available regardless of the node's orientation relative to the room. Loop closure is reliable even during complex room traversal patterns.

**Forensic quality 3D reconstruction:** Textured 3D mesh from all 360° camera images represents the room at the time of recording. Users can review the mesh in the companion app, scrub through time to see how the scene changes, and export for legal use.

---

## 5. Civilian Transfer

The SLAM architecture in AEGIS-MESH is structurally identical to that used in:
- **Robotics:** Autonomous mobile robots use LiDAR SLAM for navigation
- **AR/VR:** Visual SLAM for environment mapping in mixed reality
- **BIM (Building Information Modeling):** 3D capture of building geometry for architecture and construction
- **Insurance documentation:** 3D capture of property state for insurance claims
- **Forensic documentation:** Crime scene or incident documentation in 3D
```

---

### New File: `sentinel-wear/docs/theory/slam_world_model.md`

```markdown
# SLAM World Model — Dense 3D Reconstruction in SENTINEL-WEAR

**Applies to:** Belt Node Variant B (Linux SoM), with pendant 360° cameras and/or eyewear Variant C
**Implementation:** `crates/sentinel-slam/`

---

## 1. Overview

SENTINEL-WEAR's world model operates in two simultaneous modes:

**Mode A — Sparse:** PentaTrack-based body-centric probabilistic tracking. Always active. Low power. Fast (< 20 ms update).

**Mode B — Dense SLAM:** Full 3D reconstruction of the wearer's environment as they move. Requires Linux SoM belt node and 360° pendant cameras or eyewear Variant C visual array.

---

## 2. SLAM Inputs from Wearable Nodes

**360° Curved Pendant (Variant B):**
- 8 camera streams at 45° intervals = no blind spots at chest level
- IMU provides pendant pose for sensor-to-world transform
- This is the primary SLAM anchor: chest level, omnidirectional coverage, stable relative to torso

**Eyewear Node (Variant C — full array):**
- Forward + side cameras = 180°+ visual coverage from head level
- Head IMU provides head pose (separate from torso)
- Contributes forward visual odometry, loop closure on revisited areas
- Wide baseline between forward and side cameras enables depth from stereo

**Anklet ToF:**
- Ground plane height and obstacle detection
- Constrains vertical drift in SLAM (floor-level reference)

**Belt IMU:**
- Primary motion reference
- Provides velocity estimate for map prediction between sensor updates

---

## 3. Mobile SLAM Characteristics

Unlike room-mounted SLAM (AEGIS-MESH), SENTINEL-WEAR SLAM is **mobile** — the sensors move with the wearer. This creates unique challenges:

**Motion blur:** When the wearer walks, the pendant swings. Camera frames during high-motion moments are rejected by the SLAM system (too blurry for reliable feature extraction). Only frames with acceptable motion score are used as keyframes.

**Revisitation:** The wearer returns to the same locations (doorways, kitchen, living room). Each revisit provides loop closure opportunity that corrects accumulated drift.

**Privacy boundary:** The SLAM map is stored locally on the belt node SD card. It is not transmitted anywhere unless the user configures cloud backup or legal export.

**Map segmentation:** The SLAM system segments the map by location cluster (home, office, transit, outdoor). Users can selectively delete specific location segments.

---

## 4. Output: Body-Centric SLAM Map

The dense world model updates as the wearer moves:

**Immediate update (< 100 ms):** New LiDAR/ToF readings integrate into the occupancy grid around the wearer's current position.

**Keyframe update (every 0.5–2 s):** Camera keyframe processed for feature matching, map extension, and loop closure.

**Global consistency (background, minutes):** Loop closure optimization runs in the background. When the wearer revisits a known area, the map is adjusted for global consistency.

**Companion app view:** Users see the body-centric 3D map update in real-time. As they walk through their environment, the 3D model builds around them. On the companion app: **World View → 3D Map → Current Environment**.

---

## 5. Use Cases

**Security review:** "What did my environment look like at 3pm yesterday?" — scrub the 3D map timeline to replay the state of the environment at any recorded moment.

**Evidence documentation:** Encountered an incident? Export the 3D map segment with timestamps and integrity manifest. The map includes 360° video texture for every area the wearer passed through.

**Navigation memory:** For users with memory or cognitive challenges, the SLAM map provides a complete visual record of everywhere visited, with timestamps and event annotations.

**Research:** The body-frame SLAM dataset (position + sensor data from a mobile human) is a research contribution. Phase 5 of the roadmap includes a public open-data release of anonymized body-frame SLAM trajectories.
