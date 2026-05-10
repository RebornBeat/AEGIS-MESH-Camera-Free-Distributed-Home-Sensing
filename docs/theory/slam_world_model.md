# SLAM World Model — Dense 3D Reconstruction in AEGIS-MESH

**Project:** AEGIS-MESH
**Domain:** Dense 3D world reconstruction for stationary sensor mesh
**Applies to:** LiDAR Node Variants C and D, Edge Controller Variants B and C
**Implementation:** `crates/aegis-slam/`

---

## 1. Purpose

AEGIS-MESH operates in two parallel world model modes that run simultaneously when hardware supports it:

**Mode A — Sparse Probabilistic World Model:**
- Technology: OMNI-SENSE → PentaTrack → Zone-anchored tracks
- What it produces: Per-zone tracked entities with position, velocity, trajectory, classification, and anomaly flags
- Visualization: Floor-plan overlay with detection blobs, velocity vectors, zone alerts
- Use cases: All real-time alerting, occupancy monitoring, presence detection
- Power: Runs on all hardware configurations; always active
- Latency: < 20 ms update rate

**Mode B — Dense SLAM World Model:**
- Technology: LiDAR point clouds + camera visual odometry → SLAM → 3D mesh + object recognition
- What it produces: Full 3D geometry of monitored spaces, objects classified and tracked in 3D context
- Visualization: 3D walkthrough of the captured environment with entity trajectories and event annotations
- Use cases: Forensic review, legal evidence preparation, architecture audit, precise occupancy mapping, insurance documentation
- Power: Requires edge controller with Linux and sufficient compute (Raspberry Pi CM4 class minimum)
- Latency: 100-500 ms update rate (not time-critical)

**Both modes run simultaneously when hardware permits.** Mode A always runs. Mode B activates based on hardware configuration and user preference. The two modes benefit each other: SLAM object classifications improve PentaTrack drift profiles, while PentaTrack predictions annotate the dense map with motion intent.

---

## 2. The Stationary vs Mobile SLAM Distinction

### 2.1 AEGIS-MESH SLAM Characteristics (Stationary Sensors)

Unlike wearable SLAM systems (SENTINEL-WEAR), AEGIS-MESH SLAM operates with **stationary sensors mounted in fixed positions**:

| Characteristic | AEGIS-MESH (Stationary) | SENTINEL-WEAR (Mobile) |
|----------------|-------------------------|------------------------|
| Sensor positions | Fixed in space | Move with wearer |
| World frame | Fixed (room coordinates) | Body-centric |
| Loop closure | Return to previously mapped rooms | Return to previously visited locations |
| Motion blur | None (sensors stationary) | Pendant sway during walking |
| Map persistence | Persistent across sessions | Body-centric, segmented by location |
| Drift accumulation | Very low (fixed sensors) | Higher (mobile sensors) |
| Calibration | One-time per installation | Per-wearer calibration |

### 2.2 Advantages of Stationary SLAM

**Stability:** Sensors don't move, so:
- No motion blur from sensor motion
- No body-induced vibration
- Consistent illumination conditions per node
- Deterministic sensor-to-sensor geometry

**Coverage overlap:** Multiple nodes observe the same space from different angles:
- Redundant measurements improve map quality
- Occlusions from one node's perspective are visible from another
- Cross-node feature matching provides strong constraints

**Persistent map:** The map is tied to the building, not a person:
- Map persists across sessions
- Multiple family members use the same map
- Historical comparison (what changed over time)

**Multi-node fusion:** Every LiDAR node contributes to a single unified map:
- Point clouds from multiple nodes are fused in a common coordinate frame
- Coverage density increases with more nodes
- Robustness to individual node failure

---

## 3. SLAM Architecture Overview

### 3.1 Input Sources

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         AEGIS-MESH SLAM INPUT SOURCES                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  LiDAR Point Clouds (Variants A–D)                                          │
│  ├── Primary geometry source                                                │
│  ├── Solid-state LiDAR at ceiling-corner positions                         │
│  ├── Scan rate: 10-50 Hz                                                    │
│  ├── Point density: 100k-200k points/sec per node                          │
│  └── Range: 20-40 m typical residential                                    │
│                                                                              │
│  Event Camera (Variants A–D)                                                 │
│  ├── Fast transient detection                                               │
│  ├── Triggers high-resolution LiDAR scan                                    │
│  ├── Microsecond latency for motion events                                  │
│  └── Provides temporal triggering, not primary geometry                     │
│                                                                              │
│  Conventional Camera (Variant C)                                             │
│  ├── Single camera per node                                                 │
│  ├── Visual odometry for pose refinement                                    │
│  ├── Texture for 3D mesh                                                    │
│  └── Object recognition (YOLO-based)                                        │
│                                                                              │
│  360° Camera Array (Variant D)                                               │
│  ├── 4-8 cameras per node                                                   │
│  ├── Complete visual coverage (no blind spots)                              │
│  ├── Full texture for 3D mesh                                               │
│  ├── Maximum visual odometry accuracy                                       │
│  └── Stitched 360° panorama for companion app                               │
│                                                                              │
│  Environmental Sensors (All Variants)                                        │
│  ├── Atmospheric compensation for LiDAR range                               │
│  ├── Temperature, humidity, pressure, VOC                                   │
│  └── Not used for SLAM directly, but affects range accuracy                 │
│                                                                              │
│  Edge Controller IMU (Optional)                                              │
│  ├── Provides building motion reference (earthquake, vibration)             │
│  └── Rarely needed for residential applications                             │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 3.2 SLAM Backend — Edge Controller

The edge controller is the compute hub for all SLAM processing. Node capacity depends on edge controller variant:

**Edge Controller Variant A — SBC (Raspberry Pi 4/5 Class):**

| Capability | Performance |
|------------|-------------|
| LiDAR-only SLAM | 10-15 Hz |
| LiDAR + single camera | 5-10 Hz |
| LiDAR + 360° camera | 2-5 Hz (strained) |
| Dense mesh reconstruction | 0.5-2 Hz |
| Object recognition | 2-5 FPS |
| Loop closure | Background (minutes) |

**Recommended for:** LiDAR-only or single-camera deployments

**Edge Controller Variant B — ARM with NPU (Jetson Nano/Orin Class):**

| Capability | Performance |
|------------|-------------|
| LiDAR-only SLAM | 20-30 Hz |
| LiDAR + single camera | 15-25 Hz |
| LiDAR + 360° camera | 10-20 Hz |
| Dense mesh reconstruction | 5-10 Hz |
| Object recognition | 15-30 FPS |
| Loop closure | Real-time |

**Recommended for:** Full 360° camera deployments, multi-node SLAM

**Edge Controller Variant C — x86 Mini PC:**

| Capability | Performance |
|------------|-------------|
| LiDAR-only SLAM | 30-50 Hz |
| LiDAR + single camera | 25-40 Hz |
| LiDAR + 360° camera | 20-35 Hz |
| Dense mesh reconstruction | 10-20 Hz |
| Object recognition | 30-60 FPS |
| Loop closure | Real-time |

**Recommended for:** Production deployments, multi-building installations

### 3.3 SLAM Algorithm Selection

The edge controller runs the SLAM backend. Algorithm selection depends on available sensors:

**LiDAR-Only SLAM (All Variants):**

| Algorithm | Use Case | Performance |
|-----------|----------|-------------|
| FAST-LIO2 | Real-time, direct LiDAR odometry | 20-50 Hz on Pi 4, 100+ Hz on x86 |
| LIO-SAM | LiDAR-inertial with smoothing | 10-30 Hz, better loop closure |
| A-LOAM | Lightweight odometry and mapping | 15-40 Hz, good for low-power |

**LiDAR + Visual SLAM (Variants C, D):**

| Algorithm | Use Case | Performance |
|-----------|----------|-------------|
| LVI-SAM | LiDAR-Visual-Inertial | 10-25 Hz, excellent accuracy |
| RTAB-Map | RGB-D SLAM with loop closure | 5-15 Hz, comprehensive features |
| ORB-SLAM3 | Visual odometry + LiDAR fusion | 15-30 Hz visual, 30 Hz LiDAR |

**Recommended Configuration by Node Variant:**

| Node Variant | Recommended SLAM Algorithm |
|--------------|---------------------------|
| Variant A (LiDAR only) | FAST-LIO2 |
| Variant B (LiDAR + radar) | FAST-LIO2 (radar for fog/steam) |
| Variant C (LiDAR + camera) | LVI-SAM or LIO-SAM with visual refinement |
| Variant D (LiDAR + 360° camera) | LVI-SAM with 360° visual features |

---

## 4. Multi-Node Fusion Architecture

### 4.1 The Challenge: Multiple LiDAR Nodes Covering the Same Space

AEGIS-MESH deployments typically have multiple LiDAR nodes, each observing overlapping portions of the home:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    MULTI-NODE COVERAGE EXAMPLE                              │
│                                                                              │
│   Living Room (5m × 6m)                                                      │
│   ┌─────────────────────────────────────────────────────────────────────┐    │
│   │  ○ LiDAR Node 1 (northeast corner)                                 │    │
│   │      └── Coverage: 70% of room                                     │    │
│   │                                                                     │    │
│   │                           ○ LiDAR Node 2 (southwest corner)        │    │
│   │                                └── Coverage: 70% of room            │    │
│   │                                                                     │    │
│   │         Overlap region: 40% of room observed by both nodes         │    │
│   │                                                                     │    │
│   └─────────────────────────────────────────────────────────────────────┘    │
│                                                                              │
│   Overlap provides:                                                          │
│   ├── Redundant measurements → improved map quality                         │
│   ├── Cross-node feature matching → strong geometric constraints            │
│   ├── Occlusion handling → blind spots from one node visible from other     │
│   └── Robustness → single node failure doesn't lose entire map              │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 4.2 Multi-Node Point Cloud Fusion

**Step 1: Per-Node Local Odometry**

Each LiDAR node maintains its own local coordinate frame and odometry estimate:

```
Node 1: Frame N1, pose estimate P1(t)
Node 2: Frame N2, pose estimate P2(t)
...
Node N: Frame NN, pose estimate PN(t)
```

**Step 2: Global Frame Registration**

All node frames are registered to a common global frame (edge controller coordinate system):

```
Global Frame G (edge controller origin)
├── Transform T_G_N1: Node 1 to global
├── Transform T_G_N2: Node 2 to global
└── Transform T_G_NN: Node N to global
```

**Registration methods:**

| Method | Requirement | Accuracy | Notes |
|--------|-------------|----------|-------|
| Manual calibration | User input | ±10 cm | Initial placement |
| Walk-through calibration | User walks with tracked phone | ±5 cm | Easy, accurate |
| Automatic calibration | Overlap region matching | ±2 cm | Best, requires overlap |
| Continuous refinement | ICP between node maps | ±1 cm | Ongoing improvement |

**Step 3: Point Cloud Fusion**

Point clouds from all nodes are transformed to global frame and merged:

```rust
// Pseudocode for multi-node point cloud fusion
fn fuse_point_clouds(nodes: &[LiDARNode], global_frame: &GlobalFrame) -> PointCloud {
    let mut global_cloud = PointCloud::new();
    
    for node in nodes {
        // Get node's local point cloud
        let local_cloud = node.get_latest_scan();
        
        // Transform to global frame
        let transform = global_frame.get_transform_to(&node.frame);
        let global_cloud_node = local_cloud.transform(&transform);
        
        // Merge into global cloud
        global_cloud.merge(global_cloud_node);
    }
    
    // Voxel grid filter to reduce redundancy in overlap regions
    global_cloud.voxel_grid_filter(voxel_size: 0.05);
    
    global_cloud
}
```

**Step 4: Uncertainty-Aware Fusion**

In overlap regions, multiple nodes observe the same surface. Rather than naive averaging, AEGIS-MESH uses uncertainty-weighted fusion:

```
For each voxel in overlap region:
    observation_1 from Node 1 with variance σ1²
    observation_2 from Node 2 with variance σ2²
    
    Fused value = (obs1/σ1² + obs2/σ2²) / (1/σ1² + 1/σ2²)
    Fused variance = 1 / (1/σ1² + 1/σ2²)
```

This produces a denser, more accurate map than any single node could achieve.

### 4.3 Cross-Node Feature Matching

For nodes with cameras (Variants C, D), visual features provide additional geometric constraints:

**Feature extraction:** Each camera extracts ORB or FAST features.

**Cross-node matching:** Features from overlapping regions are matched across nodes.

**Geometric constraint:** If Node 1's camera and Node 2's camera both see the same visual feature (e.g., a corner of a painting), that feature's position must be consistent in both nodes' maps.

**Optimization:** Bundle adjustment across all nodes, with LiDAR geometry as prior, visual features as constraints.

---

## 5. Dense 3D Reconstruction

### 5.1 Voxel Grid Representation (OctoMap)

The primary map representation is an occupancy voxel grid:

| Parameter | Default Value | Description |
|-----------|---------------|-------------|
| Resolution | 5 cm | Voxel size |
| Probabilistic | Yes | Each voxel stores occupancy probability |
| Octree depth | 16 | Max levels in OctoMap |
| Ray tracing | Yes | Free space marked as unoccupied |
| Persistence | SD card / SSD | Map stored between sessions |

**Occupancy update equation:**

```
P(occupied | measurement) = P(measurement | occupied) × P(occupied) / P(measurement)

Simplified log-odds update:
L(occupied) = L(previous) + L(measurement) - L(prior)
```

### 5.2 TSDF Fusion for Dense Mesh

For visualization and forensic purposes, a Truncated Signed Distance Function (TSDF) mesh is built:

```
For each LiDAR point:
    Cast ray from sensor to point
    For voxels along ray (up to truncation distance):
        TSDF value = signed distance to surface
        Weight = distance weighting
    
Marching cubes → surface mesh → OBJ/PLY export
```

**TSDF parameters:**

| Parameter | Default | Description |
|-----------|---------|-------------|
| Voxel size | 2 cm | Finer than occupancy grid |
| Truncation distance | 10 cm | Distance to surface to update |
| Weighting | Distance-based | Closer measurements weighted higher |

### 5.3 Texture Mapping

For nodes with cameras (Variants C, D), the mesh is textured using camera imagery:

```
For each triangle in mesh:
    Determine which cameras observe this triangle
    Select highest-resolution camera with best viewing angle
    Project triangle to camera image
    Sample texture from image
    Apply to triangle
```

**Result:** A fully textured 3D mesh of the home, suitable for:
- Forensic documentation
- Insurance claims
- Architectural review
- Virtual walkthrough

---

## 6. Loop Closure and Drift Correction

### 6.1 The Loop Closure Problem

Even with stationary sensors, drift accumulates:
- LiDAR odometry is not perfect
- ICP alignment has residual error
- Over time, the map "drifts" from true geometry

**Loop closure:** When the system observes a previously mapped area, it can correct accumulated drift.

### 6.2 Loop Closure Mechanisms

**LiDAR-based loop closure:**

| Method | Mechanism | When Used |
|--------|-----------|-----------|
| Scan context | Global descriptor matching | Every 10-50 scans |
| M2DP | Multi-view projection descriptor | Every 50-100 scans |
| Iris-based | Feature histogram | Camera-equipped nodes only |

**Visual loop closure (Variants C, D):**

| Method | Mechanism | Performance |
|--------|-----------|-------------|
| DBoW2 | Bag of visual words | Real-time |
| FAB-MAP | Appearance-based | Offline batch |
| VLAD | Vector of locally aggregated descriptors | Real-time |

**Hybrid LiDAR-visual loop closure:**

For 360° camera variants, the most robust loop closure combines both modalities:

```
1. Visual feature matching detects potential loop closure
2. LiDAR geometry validates the match
3. Pose graph optimization corrects accumulated drift
4. Map is updated with corrected geometry
```

### 6.3 Pose Graph Optimization

When loop closure is detected, a pose graph optimization corrects the entire trajectory:

```
Nodes in pose graph:
├── Keyframe poses (from odometry)
├── Loop closure constraints (from detection)
└── Inter-node constraints (from multi-node fusion)

Optimization:
├── g2o or GTSAM backend
├── Minimize reprojection error
└── Produce globally consistent map
```

### 6.4 Drift Accumulation in AEGIS-MESH

Because sensors are stationary, drift is very low compared to mobile SLAM:

| Factor | Drift Impact | Mitigation |
|--------|--------------|------------|
| Odometry error | < 0.1% per hour | Loop closure eliminates |
| Thermal expansion | < 1 cm per °C | Not significant for residential |
| Furniture movement | Creates temporary "objects" | Dynamic remap handles |
| Sensor noise | < 1 cm per measurement | Averaging reduces |

**Expected drift:** < 5 cm per day without loop closure. Loop closure reduces to < 1 cm globally.

---

## 7. Object Recognition and Semantic Labeling

### 7.1 Integration with PentaTrack

The SLAM pipeline recognizes objects in the environment and attaches labels to the 3D map. These labels improve PentaTrack tracking:

**Without semantic labels:**
```
PentaTrack sees: "Moving object, human-sized, 1.8 m tall"
Classification: "Unknown person"
```

**With semantic labels:**
```
SLAM detects: "Couch, 2.0 m × 0.9 m, at position (3.5, 2.1, 0.0)"
PentaTrack sees: "Moving object near couch"
Classification: "Person sitting on couch" or "Person walking behind couch"
```

### 7.2 Object Recognition Pipeline

**For camera-equipped nodes (Variants C, D):**

```
Camera frame → YOLOv8 inference → Bounding boxes
           ↓
LiDAR point cloud → Cluster detection → 3D regions
           ↓
Association: Match bounding boxes to point cloud clusters
           ↓
Label: Attach object class to cluster
           ↓
Map: Add labeled object to 3D map
           ↓
PentaTrack: Use label for context-aware tracking
```

**Supported object classes (residential context):**

| Category | Classes |
|----------|---------|
| Furniture | Couch, chair, table, bed, desk, cabinet, shelf |
| Appliances | Refrigerator, stove, TV, microwave, washer |
| People | Person (multiple, distinguishable by position) |
| Pets | Dog, cat |
| Objects | Bag, box, luggage, plant |
| Hazards | Fire, smoke, water on floor |

### 7.3 On-Edge Controller Object Recognition

Object recognition runs on the edge controller, not on nodes:

| Edge Controller Class | Recognition Performance |
|----------------------|------------------------|
| SBC (Pi 4/5) | YOLOv8-nano: 2-5 FPS |
| ARM + NPU (Jetson) | YOLOv8-small: 15-30 FPS |
| x86 Mini PC | YOLOv8-medium: 30-60 FPS |

**Configuration:**

```toml
[slam.object_recognition]
enabled = true
model = "yolov8-nano"            # "yolov8-nano" | "yolov8-small" | "yolov8-medium"
confidence_threshold = 0.6
classes = ["person", "couch", "chair", "table", "bed", "dog", "cat"]
frame_skip = 5                    # Process every 5th frame
```

---

## 8. Dynamic Object Handling

### 8.1 The Dynamic Object Problem

SLAM assumes the world is static. In homes, this assumption is violated by:
- People walking
- Pets moving
- Furniture being moved
- Doors opening/closing

**Without dynamic handling:** Moving objects create "ghost" geometry in the map.

**AEGIS-MESH solution:** Integration with PentaTrack identifies and removes dynamic objects from the SLAM map.

### 8.2 Dynamic-Static Separation

```
Point cloud from LiDAR scan
        │
        ▼
PentaTrack tracks → Identify moving entities
        │
        ▼
Remove points belonging to moving entities from SLAM input
        │
        ▼
SLAM processes static points only
        │
        ▼
Map represents static structure
```

**Implementation:**

```rust
// Pseudocode for dynamic-static separation
fn process_scan(node: &LiDARNode, tracker: &PentaTracker) -> StaticPointCloud {
    // Get raw point cloud
    let raw_cloud = node.get_scan();
    
    // Get currently tracked entities
    let tracks = tracker.get_active_tracks();
    
    // For each point, check if it belongs to a moving entity
    let static_points: Vec<Point> = raw_cloud.points()
        .filter(|point| {
            // Check if point is near any tracked entity
            !tracks.iter().any(|track| {
                let entity_volume = track.get_bounding_volume();
                entity_volume.contains(point.position)
            })
        })
        .collect();
    
    StaticPointCloud::from_points(static_points)
}
```

### 8.3 Furniture Movement Detection

When furniture is moved, the SLAM map must be updated:

**Detection:** PentaTrack detects that a "static" object (couch) has moved.

**Remap trigger:** The dynamic remap system is triggered (see `docs/guides/adaptive_remap.md`).

**Map update:** The old furniture position is removed, new position is added.

**Loop closure:** The updated geometry may trigger re-optimization.

---

## 9. Map Persistence and Management

### 9.1 Storage Architecture

The SLAM map is stored on the edge controller:

| Storage Location | Contents | Size Estimate |
|------------------|----------|---------------|
| SD card / SSD | OctoMap, TSDF mesh, keyframes | 100 MB - 5 GB |
| Internal flash | Metadata, calibration | < 10 MB |
| Network storage (optional) | Backup, archive | Unlimited |

**File structure:**

```
/var/lib/aegis-mesh/slam/
├── octomap/
│   └── global_map.bt                 # Binary OctoMap file
├── mesh/
│   ├── global_mesh.ply               # Surface mesh
│   └── textured_mesh.obj             # Textured mesh (if cameras)
├── keyframes/
│   ├── kf_001234.jpg                 # Keyframe images
│   ├── kf_001234.pose                # Associated poses
│   └── ...
├── descriptors/
│   └── scan_context.bin              # Loop closure descriptors
├── objects/
│   └── detected_objects.json         # Recognized objects
└── metadata/
    ├── calibration.json              # Node calibration
    └── session_info.json             # Current session metadata
```

### 9.2 Incremental Map Updates

The map is updated incrementally, not rebuilt from scratch:

```
Boot:
├── Load existing map from disk
├── Load calibration data
└── Continue adding new observations

Runtime:
├── Each LiDAR scan → incremental update
├── Each camera keyframe → add to database
└── Loop closure → pose graph optimization

Shutdown:
├── Save updated map
├── Save optimized poses
└── Compact database
```

### 9.3 Map Versioning and History

For forensic and legal purposes, AEGIS-MESH can maintain map history:

```toml
[slam.history]
enabled = true
snapshot_interval_hours = 24        # Save daily snapshots
max_snapshots = 30                  # Keep 30 days
changes_only = true                 # Only save changed regions
```

**Snapshot format:**

```
/snapshots/
├── 2024-01-01_00-00/
│   ├── octomap.bt
│   ├── mesh.ply
│   └── metadata.json
├── 2024-01-02_00-00/
│   └── ...
```

---

## 10. SLAM Output Integration with PentaTrack

### 10.1 Sparse-Dense Fusion

The dense SLAM map integrates with the sparse PentaTrack world model:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    SPARSE-DENSE WORLD MODEL FUSION                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  DENSE SLAM (Mode B)                                                         │
│  ├── Full 3D geometry                                                        │
│  ├── Static structure (walls, furniture)                                    │
│  ├── Object labels (couch, table, door)                                    │
│  └── Texture (from cameras)                                                 │
│                                                                              │
│          │                                                                    │
│          ▼                                                                    │
│                                                                              │
│  SPARSE PENTATRACK (Mode A)                                                  │
│  ├── Real-time entity tracking                                              │
│  ├── Position, velocity, trajectory                                         │
│  ├── Classification (human, pet, object)                                    │
│  └── Anomaly detection                                                      │
│                                                                              │
│  FUSION OUTPUT                                                               │
│  ├── Entity positions anchored to dense geometry                            │
│  ├── Object context for classification (person near couch)                  │
│  ├── Static background removed from detection field                         │
│  └── Occlusion-aware tracking                                               │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 10.2 Static Background Subtraction

Detections that project onto static voxels (walls, furniture) are suppressed or reclassified:

**Without static subtraction:**
```
LiDAR point cluster on couch
→ PentaTrack: "Detection at (3.5, 2.1)"
→ Alert: "Motion in living room"
→ False positive (person sat down, now stationary)
```

**With static subtraction:**
```
LiDAR point cluster on couch
→ SLAM: "Couch at (3.5, 2.1)"
→ PentaTrack: "Detection matches static object"
→ Classification: "Person seated" or "Static, ignore"
→ No false alert
```

### 10.3 Object Context for Classification

SLAM-recognized objects provide context that improves PentaTrack classification:

| Context | Improved Classification |
|---------|------------------------|
| "Near couch" | Likely seated person |
| "Near door" | Likely person entering/exiting |
| "Near window" | Could be person or reflection |
| "In kitchen" | Likely resident doing activities |
| "Near pet bed" | Likely pet, not person |

**Implementation in PentaTrack:**

```rust
// Classification with object context
fn classify_with_context(
    detection: &Detection,
    slam_map: &SLAMMap
) -> Classification {
    let nearby_objects = slam_map.get_objects_near(detection.position, radius: 2.0);
    
    match nearby_objects.as_slice() {
        [Object::Couch, ..] => Classification::PersonSeated,
        [Object::Door, ..] => Classification::PersonEntering,
        [Object::PetBed, ..] => Classification::Pet,
        [] => classify_from_motion(detection),
    }
}
```

### 10.4 Occlusion-Aware Tracking

The dense map tells PentaTrack which areas are occluded from which nodes:

**Without occlusion awareness:**
```
Node 1 loses track as person walks behind couch
→ Track fragmented into two tracks
→ Confusion about continuity
```

**With occlusion awareness:**
```
SLAM: "Couch creates occlusion zone for Node 1"
PentaTrack: "Track lost behind occlusion, predict emergence on other side"
→ Continuous track maintained
```

---

## 11. 360° Camera SLAM Specifics

### 11.1 LiDAR Variant D — Full Room Coverage

LiDAR Variant D with 360° camera array enables room-scale visual SLAM without blind spots:

| Configuration | Camera Count | Angular Spacing | FoV per Camera |
|---------------|--------------|-----------------|-----------------|
| Minimal 360° | 4 | 90° | ≥ 100° |
| Standard 360° | 6 | 60° | ≥ 70° |
| Dense 360° | 8 | 45° | ≥ 55° |
| Ultra 360° | 12 | 30° | ≥ 40° |

### 11.2 Stitch-Free SLAM Processing

Rather than stitching cameras to equirectangular first, the SLAM pipeline processes each camera independently:

**Advantages:**
- No resampling artifacts at seams
- Individual camera calibration used directly
- No data loss from stitching compression
- Multi-camera feature matching handles boundary features naturally

**Implementation:**

```
Camera 0 → Feature extraction → Features_F0
Camera 1 → Feature extraction → Features_F1
...
Camera 7 → Feature extraction → Features_F7

Multi-camera feature matching:
├── Features_F0 matched to Features_F1 at overlap boundary
├── Features_F1 matched to Features_F2
└── ...

Global feature database:
├── All features in unified 3D coordinates
└── Camera-to-camera constraints enforced
```

### 11.3 Complete Loop Closure

With 360° coverage, visual loop closure is always available regardless of node orientation:

**Traditional camera SLAM:**
```
Camera faces north
├── Features visible in north direction
├── No features visible south
└── Loop closure depends on camera facing same direction again
```

**360° camera SLAM:**
```
All directions visible simultaneously
├── Features in all directions at all times
├── Loop closure works regardless of orientation
└── Robust to viewpoint changes
```

### 11.4 Forensic Quality 3D Reconstruction

The textured 3D mesh from 360° cameras provides forensic-quality documentation:

**Characteristics:**
- Complete visual coverage (no blind spots)
- High resolution (depends on camera selection)
- Texture from all angles
- Timestamp for legal chain of custody

**Export formats:**
- PLY/OBJ (geometry only)
- GLTF/GLB (geometry + texture)
- FBX (compatible with forensic software)
- Custom AEGIS format (includes metadata, integrity chain)

---

## 12. Calibration and Setup

### 12.1 Initial Node Calibration

For SLAM to work correctly, each node's position and orientation must be known:

**Manual calibration:**
- User enters node positions in configuration file
- Orientation determined from IMU
- Accuracy: ±10 cm position, ±5° orientation

**Walk-through calibration:**
- User walks through home with tracked phone
- System observes user from multiple nodes
- Trilateration determines node positions
- Accuracy: ±5 cm position, ±2° orientation

**Automatic calibration (multi-node overlap):**
- Nodes observe overlapping regions
- ICP alignment determines relative positions
- Accuracy: ±2 cm position, ±1° orientation

### 12.2 Camera Calibration (Variants C, D)

Camera calibration is essential for accurate visual SLAM:

**Intrinsics:** Each camera's lens distortion and focal length

**Extrinsics:** Position and orientation relative to LiDAR

**Multi-camera extrinsics (Variant D):** Relative positions of all cameras in the 360° array

**Calibration process:**

```
1. Print calibration checkerboard
2. Place at multiple positions in the room
3. System captures images from all cameras
4. OpenCV calibration determines intrinsics
5. LiDAR-camera alignment determines extrinsics
6. Calibration stored in node flash and edge controller
```

### 12.3 SLAM-Specific Calibration

**Time synchronization:** All nodes must share a common clock:

```
Edge controller as time master
├── Broadcasts time via mesh protocol
├── Nodes sync via PTP-style exchange
└── Accuracy: < 1 ms
```

**Coordinate frame alignment:**

```
User defines origin (e.g., front door)
├── All nodes transformed to this frame
├── Map is in "home coordinates"
└── Easy for humans to understand
```

---

## 13. Performance Optimization

### 13.1 Computational Budget

SLAM is computationally intensive. The edge controller must balance:

| Function | CPU Budget | Priority |
|----------|------------|----------|
| PentaTrack (sparse) | 10-20% | Highest (real-time) |
| LiDAR odometry | 20-30% | High |
| Visual odometry | 10-20% | Medium |
| Dense reconstruction | 10-20% | Low (background) |
| Object recognition | 10-15% | Low (background) |
| Loop closure | 5-10% | Lowest (background) |

### 13.2 Adaptive Quality

For resource-constrained edge controllers, SLAM quality adapts:

| Available Resources | SLAM Quality |
|---------------------|--------------|
| Abundant (x86) | Full quality, real-time |
| Moderate (Jetson) | Full quality, occasional lag |
| Limited (Pi 4) | Reduced resolution, lower update rate |
| Very limited (Pi 3) | LiDAR-only, no visual |

**Configuration:**

```toml
[slam.adaptive]
enabled = true
target_cpu_percent = 70          # Don't exceed 70% CPU
target_ram_percent = 60          # Don't exceed 60% RAM

# Quality degradation hierarchy
[slam.adaptive.degradation]
# When resource constrained, reduce quality in this order:
1 = "reduce_mesh_update_rate"     # Slow mesh reconstruction
2 = "reduce_camera_fps"           # Fewer camera keyframes
3 = "increase_voxel_size"         # Coarser map
4 = "disable_object_recognition"  # No semantic labels
5 = "disable_visual_odometry"     # LiDAR-only
```

### 13.3 Power Considerations

SLAM is the most power-intensive function on the edge controller:

| Edge Controller | SLAM Mode | Power Draw |
|-----------------|-----------|------------|
| Pi 4 | LiDAR-only | 4-6 W |
| Pi 4 | Full SLAM | 6-8 W |
| Jetson Nano | Full SLAM | 8-12 W |
| x86 Mini PC | Full SLAM | 15-25 W |

**For always-on deployments:** Edge controllers are typically AC-powered, so power is not a constraint.

---

## 14. Companion App Integration

### 14.1 3D World Model Viewer

The companion app provides full 3D visualization of the SLAM map:

**Features:**
- Interactive 3D navigation (orbit, pan, zoom)
- Textured mesh display (from camera nodes)
- Entity trajectory overlay (from PentaTrack)
- Time scrubbing (replay past states)
- Object labels (from recognition)
- Export to standard 3D formats

**API endpoints:**

| Endpoint | Description |
|----------|-------------|
| `GET /api/world-model/3d` | Current 3D mesh |
| `GET /api/world-model/3d/textured` | Textured mesh with texture files |
| `GET /api/world-model/3d/timestamp/{ts}` | Map state at specific time |
| `GET /api/world-model/objects` | List of recognized objects |
| `GET /api/world-model/export` | Export mesh in selected format |

### 14.2 Forensic Export

For legal and insurance purposes, the SLAM map can be exported with full integrity chain:

**Export package contains:**
- 3D mesh (PLY/OBJ)
- Texture images (if available)
- Object annotations (JSON)
- Session metadata
- Cryptographic hash chain
- Timestamp range

**Example integrity manifest:**

```json
{
  "export_version": "1.0",
  "export_type": "slam_map",
  "timestamp_range": {
    "start": "2024-01-15T00:00:00Z",
    "end": "2024-01-22T00:00:00Z"
  },
  "nodes_contributing": ["lidar_node_01", "lidar_node_02"],
  "mesh_file": "world_mesh.ply",
  "mesh_sha256": "a3f4b2c9d8e1f7a2b5c8d3e6f9a0b1c2...",
  "texture_count": 1247,
  "object_count": 34,
  "firmware_versions": {
    "edge_controller": "v1.2.0",
    "lidar_node_01": "v1.1.0",
    "lidar_node_02": "v1.1.0"
  },
  "export_timestamp": "2024-01-22T12:00:00Z",
  "chain_hash": "e3f5a7b9c1d2e4f6a8b0c2d4e6f8a0b2..."
}
```

---

## 15. Civilian Transfer Applications

The SLAM architecture in AEGIS-MESH is structurally identical to that used in multiple civilian domains:

### 15.1 Robotics

**Application:** Autonomous mobile robots use LiDAR SLAM for navigation

**Transfer:**
- Same algorithms (FAST-LIO2, LIO-SAM)
- Same sensor modalities (LiDAR + camera + IMU)
- Same edge computing approach

**Differences:** Mobile robots need loop closure during motion; AEGIS-MESH has stationary sensors.

### 15.2 Augmented and Virtual Reality

**Application:** Visual SLAM for environment mapping in mixed reality

**Transfer:**
- Visual odometry techniques
- Dense reconstruction for occlusion
- Real-time pose tracking

**Differences:** AR/VR needs sub-millisecond pose latency; AEGIS-MESH SLAM latency is not critical.

### 15.3 Building Information Modeling (BIM)

**Application:** 3D capture of building geometry for architecture and construction

**Transfer:**
- LiDAR scanning workflow
- Point cloud to mesh conversion
- Semantic labeling of objects

**Differences:** BIM scans are episodic (single capture); AEGIS-MESH is continuous.

### 15.4 Insurance Documentation

**Application:** 3D capture of property state for insurance claims

**Transfer:**
- Textured 3D mesh of interior
- Object inventory
- Timestamped evidence

**Differences:** Insurance needs episodic capture; AEGIS-MESH provides continuous documentation.

### 15.5 Forensic Documentation

**Application:** Crime scene or incident documentation in 3D

**Transfer:**
- High-fidelity geometry capture
- Textured mesh for visual evidence
- Cryptographic integrity chain

**Differences:** Forensic needs controlled capture process; AEGIS-MESH provides opportunistic capture.

---

## 16. Configuration Reference

### 16.1 Full SLAM Configuration

```toml
# aegis-mesh.toml

[slam]
# Master enable
enabled = true

# SLAM mode
mode = "hybrid"                    # "lidar_only" | "lidar_visual" | "hybrid"

# Map representation
[slam.map]
octomap_resolution_m = 0.05
tsdf_voxel_size_m = 0.02
ray_tracing_enabled = true

# LiDAR odometry
[slam.lidar_odometry]
algorithm = "fast_lio2"            # "fast_lio2" | "lio_sam" | "a_loam"
max_range_m = 40.0
min_range_m = 0.5
feature_extraction = true

# Visual odometry (Variants C, D)
[slam.visual_odometry]
enabled = true
algorithm = "orb_slam3"            # "orb_slam3" | "dso" | "svo"
feature_type = "orb"               # "orb" | "fast" | "sift"
keyframe_interval_frames = 10

# Loop closure
[slam.loop_closure]
enabled = true
lidar_method = "scan_context"      # "scan_context" | "m2dp"
visual_method = "dbow2"            # "dbow2" | "vlad"
min_interval_scans = 50

# Dense reconstruction
[slam.dense_reconstruction]
enabled = true
update_interval_ms = 100
background_priority = "low"

# Object recognition
[slam.object_recognition]
enabled = true
model = "yolov8-nano"
confidence_threshold = 0.6
frame_skip = 5

# Multi-node fusion
[slam.multi_node]
enabled = true
fusion_method = "uncertainty_weighted"
overlap_threshold = 0.3            # Minimum overlap for cross-node constraints

# Dynamic object handling
[slam.dynamic_handling]
enabled = true
remove_dynamic_entities = true
min_entity_velocity_ms = 0.5       # Objects moving > 0.5 m/s are dynamic

# Persistence
[slam.persistence]
auto_save_interval_minutes = 30
snapshot_enabled = true
snapshot_interval_hours = 24
max_snapshots = 30

# Performance
[slam.performance]
adaptive_quality = true
target_cpu_percent = 70
target_ram_percent = 60

# Export
[slam.export]
default_format = "ply"             # "ply" | "obj" | "gltf"
include_textures = true
include_objects = true
include_integrity_chain = true
```

### 16.2 Node-Specific SLAM Configuration

For LiDAR nodes participating in SLAM:

```toml
# Per-node configuration in aegis-mesh.toml

[[nodes]]
id = "lidar_living_room_north"
class = "LidarNode"
position = [2.0, 7.5, 2.8]

[nodes.lidar_slam]
# This node's contribution to global SLAM
enabled = true
contribute_to_global_map = true
local_odometry = true
send_point_clouds = true

# If camera-equipped
[nodes.lidar_slam.camera]
contribute_keyframes = true
contribute_textures = true
visual_odometry = true
```

---

## 17. Summary

**Key architectural decisions:**

1. **Two parallel world models:** Sparse (PentaTrack) for real-time; dense (SLAM) for documentation

2. **Stationary sensors advantage:** Low drift, persistent maps, multi-node overlap

3. **Multi-node fusion:** Overlapping coverage improves map quality and robustness

4. **360° camera integration:** Complete visual coverage without blind spots

5. **Semantic labeling:** Object recognition improves PentaTrack context awareness

6. **Dynamic-static separation:** Moving entities removed from SLAM; static structure only

7. **Loop closure:** LiDAR and visual methods combine for drift elimination

8. **Forensic capability:** Textured 3D mesh with integrity chain for legal use

**Civilian transfer:** Architecture applies directly to robotics, AR/VR, BIM, insurance, and forensics

---

**End of Document**
