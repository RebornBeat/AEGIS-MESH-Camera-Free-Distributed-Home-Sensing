# Occlusion-Driven Placement — Geometry-First Sensor Deployment

**Project:** AEGIS-MESH
**Domain:** Sensor placement optimization for residential coverage
**Implementation:** `crates/aegis-placement/`
**Status:** Reference Architecture

---

## 1. The Problem with Grid Placement

A uniform grid of sensors sounds comprehensive but is geometrically naive. Consider a living room with a large sectional sofa:

- A ceiling node directly above the sofa has its LiDAR view blocked for anyone sitting behind the sofa back.
- A second ceiling node on the opposite side of the room sees the space behind the sofa but has its view of other areas blocked by the sofa's height.
- A grid placed without accounting for the sofa's location misses both issues and the blind spots remain undetected.

Occlusion-driven placement starts from the actual 3D geometry of the space and computes placements that address specific occlusions with minimum node count.

---

## 2. Architecture Context

### 2.1 The Sensing Mesh

AEGIS-MESH uses a distributed sensing architecture where multiple node classes cooperate to build a unified world model:

| Node Class | Primary Sensing | Coverage Type | Typical Placement |
|------------|-----------------|---------------|-------------------|
| AlwaysOn A | PIR + 1D ToF | Binary crossing detection | Choke points (doorways, hallways) |
| AlwaysOn B | mmWave + Acoustic + Environmental | Volume presence | Ceiling/wall mount |
| AlwaysOn C | Enhanced acoustic array | Volume + material classification | Ceiling mount |
| AlwaysOn D | mmWave + Acoustic + Camera | Volume + visual capture | Ceiling/wall mount |
| LiDAR A | Solid-state LiDAR + Event camera | 3D geometry, fast transient | Ceiling corner |
| LiDAR B | LiDAR + Event + mmWave | 3D geometry + fog immunity | Ceiling corner |
| LiDAR C | LiDAR + Event + Camera | 3D geometry + SLAM | Ceiling position |
| LiDAR D | LiDAR + Event + 360° Camera Array | Full 360° geometry + SLAM | Ceiling center |
| Identity Node | Configurable visual/biometric | Identification | Entry points |

### 2.2 The Edge Controller Role

The edge controller serves as:
- **Compute hub:** Runs the placement optimizer, fusion algorithms, and SLAM backend
- **Data aggregator:** Receives all sensor data via mesh radio from all nodes
- **Sole external gateway:** Only device with WiFi and optional cellular connectivity
- **API server:** Serves companion app and external integrations

All placement optimization runs on the edge controller during configuration. The edge controller maintains the voxel map, coverage analysis, and placement recommendations.

### 2.3 Data Flow for Placement

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    PLACEMENT OPTIMIZATION DATA FLOW                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  [ Initial Setup Phase ]                                                    │
│                                                                              │
│  Mobile LiDAR Scan / Manual Floor Plan                                      │
│       │                                                                      │
│       ▼                                                                      │
│  Voxel Map Construction (Edge Controller)                                   │
│       │                                                                      │
│       ▼                                                                      │
│  Coverage Solver (aegis-placement)                                          │
│       │                                                                      │
│       ├──► Node placement recommendations                                   │
│       ├──► Coverage heatmap                                                 │
│       ├──► Blind spot identification                                        │
│       └──► Interference analysis                                            │
│                                                                              │
│  [ Operational Phase ]                                                       │
│                                                                              │
│  All Nodes → Mesh Radio → Edge Controller                                   │
│       │                                                                      │
│       ▼                                                                      │
│  Dynamic Remapper → Voxel Map Updates                                       │
│       │                                                                      │
│       ▼                                                                      │
│  Coverage Re-evaluation → Placement Adjustment Suggestions                  │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. The Algorithm

### Step 1: Voxel Map Construction

**Implementation:** `aegis-placement::voxel_map`

The home's interior is represented as a 3D grid of voxels at configurable resolution (default: 25 cm × 25 cm × 25 cm). Each voxel is labeled:

| Label | Definition | Sensor Transparency | Examples |
|-------|------------|---------------------|----------|
| **Open** | Traversable space | Transparent to all sensors | Floor area, room volume |
| **Structure** | Permanent building elements | Opaque to all sensors | Walls, floors, ceilings, columns |
| **Obstruction** | Furniture and fixed objects | Blocks sensor beams | Sofas, tables, cabinets, appliances |
| **Semi-Transparent** | Partially penetrable | mmWave passes, LiDAR blocked | Curtains, fabric, thin partitions |
| **Water/Vapor** | Environmental obstruction | Degrades optical sensors | Fish tanks, steam zones, fog |

The voxel map is populated from multiple sources:

**Primary source — Mobile LiDAR scan:**
- User walks through home with a handheld LiDAR scanner (or temporary node)
- Edge controller receives point cloud data via mesh radio
- Point cloud is voxelized and classified
- Typical scan time: 5-15 minutes for a typical home

**Secondary source — Manual floor plan:**
- User draws floor plan in companion app
- Furniture placement specified manually
- Voxel map generated from 2D plan with assumed heights
- Less accurate but faster setup

**Tertiary source — CAD import:**
- Architectural plans imported directly
- Most accurate but requires source files
- Suitable for new construction

### Step 2: Visibility Cone Computation

**Implementation:** `aegis-placement::occlusion_analyzer`

For each candidate sensor position, we compute the **visibility cone**: the set of voxels that the sensor can observe from that position.

**Ray-march algorithm:**

```
for each candidate_position in candidate_grid:
    for each direction in sensor_field_of_regard:
        ray = Ray(origin=candidate_position, direction=direction)
        current_voxel = candidate_position
        
        while current_voxel in voxel_map:
            if voxel_map[current_voxel] in [Structure, Obstruction]:
                # Ray blocked - stop
                break
            else:
                # Voxel is observable
                visibility_map[candidate_position].add(current_voxel)
                current_voxel = step_along_ray(ray, current_voxel, step_size)
```

**Computational optimizations:**

| Optimization | Description | Speedup |
|--------------|-------------|---------|
| Spatial hashing | Obstruction voxels stored in hash map for O(1) lookup | 10-50× |
| Early exit | Stop ray after hitting structure within first few voxels | 2-5× |
| Adaptive step size | Larger steps for open space, smaller near obstructions | 1.5-3× |
| GPU acceleration | Ray marching on GPU for large homes | 10-100× |

**Typical compute time:**
- Raspberry Pi 4: < 1 second per candidate position
- x86 desktop: < 100 ms per candidate position
- Full home (100-200 candidates): 1-5 minutes on Raspberry Pi

### Step 3: Coverage Optimization

**Implementation:** `aegis-placement::coverage_solver`

#### Inputs

| Input | Type | Description |
|-------|------|-------------|
| Voxel map | 3D array | Home geometry with voxel classifications |
| Node budget | Integer | Maximum number of nodes to deploy |
| Coverage depth | Integer | Minimum N nodes per voxel (default: 2) |
| Node class budget | Map | Maximum count per node class |
| Candidate positions | List | Possible mounting locations |
| Voxel priorities | Weights | Importance of each voxel |

#### Voxel Priority Assignment

Each voxel receives a priority weight based on multiple factors:

```rust
fn calculate_voxel_priority(voxel: Voxel, context: &PlacementContext) -> f32 {
    let mut priority = 1.0;
    
    // Vertical position weighting
    match voxel.z {
        z if z < 0.3 => priority *= 2.0,      // Floor level - highest (objects of interest)
        z if z < 1.0 => priority *= 1.5,      // Below furniture height
        z if z < 1.5 => priority *= 1.2,      // Human height range
        z if z > 2.0 => priority *= 0.3,      // Near ceiling - lower priority
        _ => priority *= 1.0,
    }
    
    // Room type weighting
    match context.get_room_type(voxel) {
        RoomType::Entryway => priority *= 3.0,
        RoomType::Stairwell => priority *= 2.5,
        RoomType::Hallway => priority *= 2.0,
        RoomType::LivingRoom => priority *= 1.5,
        RoomType::Bedroom => priority *= 1.2,
        RoomType::Bathroom => priority *= 1.0,
        RoomType::Kitchen => priority *= 1.0,
        RoomType::Garage => priority *= 1.5,
        RoomType::Utility => priority *= 0.8,
        RoomType::Storage => priority *= 0.5,
    }
    
    // Proximity to entry points
    let entry_distance = context.distance_to_nearest_entry(voxel);
    if entry_distance < 3.0 {
        priority *= 2.0 / (1.0 + entry_distance);
    }
    
    // Choke-point detection
    if context.is_choke_point(voxel) {
        priority *= 5.0;
    }
    
    // Hazard zones
    if context.is_hazard_zone(voxel) {
        priority *= 2.0;
    }
    
    priority
}
```

#### Greedy Weighted Set Cover Algorithm

```
function coverage_solver(voxel_map, budget, node_class_budget, candidates):
    placed_nodes = []
    covered_voxels = {}
    remaining_budget = budget
    
    while remaining_budget > 0:
        best_candidate = None
        best_marginal_coverage = 0
        
        for candidate in candidates:
            # Skip if node class budget exhausted
            if node_class_budget[candidate.class] <= 0:
                continue
            
            # Calculate marginal coverage
            new_voxels = visibility_map[candidate] - covered_voxels
            weighted_coverage = sum(priority[v] for v in new_voxels)
            
            # Penalize redundancy
            redundancy_penalty = count_nodes_observing_same_voxels(
                candidate, placed_nodes
            )
            marginal_coverage = weighted_coverage / (1.0 + redundancy_penalty)
            
            # Apply interference constraint
            if has_radar_interference(candidate, placed_nodes):
                if not time_division_multiplexing_configured:
                    marginal_coverage *= 0.5  # Heavy penalty
            
            if marginal_coverage > best_marginal_coverage:
                best_marginal_coverage = marginal_coverage
                best_candidate = candidate
        
        if best_candidate is None:
            break  # No improvement possible
        
        # Place the node
        placed_nodes.append(best_candidate)
        covered_voxels += visibility_map[best_candidate]
        remaining_budget -= 1
        node_class_budget[best_candidate.class] -= 1
        
        # Remove from candidates to avoid duplicate placement
        candidates.remove(best_candidate)
    
    return placed_nodes
```

#### Constraints

| Constraint | Description | Enforcement |
|------------|-------------|-------------|
| Node class budget | Maximum nodes of each type | Hard constraint |
| Coverage depth | Minimum N nodes per voxel | Soft constraint (warn if unmet) |
| Radar interference | mmWave nodes > 2 m apart or TDM configured | Soft constraint (penalty) |
| Power availability | Node near power outlet or PoE | Hard constraint (user-configurable) |
| Aesthetic preference | Avoid visible node placements | Soft constraint (user weight) |

### Step 4: Placement Report Generation

**Implementation:** `aegis-placement::report_generator`

The solver generates an HTML report with:

```html
<!DOCTYPE html>
<html>
<head>
    <title>AEGIS-MESH Placement Report</title>
    <script src="three.min.js"></script>
    <script src="orbitcontrols.min.js"></script>
</head>
<body>
    <div id="viewer"></div>
    <div id="report">
        <h1>Placement Summary</h1>
        <table>
            <tr><th>Node Count</th><td>12</td></tr>
            <tr><th>Coverage Depth</th><td>2.3 (average)</td></tr>
            <tr><th>Blind Spots</th><td>3 voxels</td></tr>
            <tr><th>Interference Warnings</th><td>1 (kitchen + dining room)</td></tr>
        </table>
        
        <h2>Node Placements</h2>
        <table>
            <tr>
                <th>ID</th>
                <th>Class</th>
                <th>Position</th>
                <th>Coverage</th>
                <th>Marginal Voxels</th>
            </tr>
            <!-- Node entries -->
        </table>
        
        <h2>Coverage Heatmap</h2>
        <div id="heatmap"></div>
        
        <h2>Blind Spots</h2>
        <ul>
            <!-- Blind spot entries -->
        </ul>
        
        <h2>Interference Warnings</h2>
        <ul>
            <!-- Interference entries -->
        </ul>
        
        <h2>Cost Estimate</h2>
        <table>
            <tr><th>Node Class</th><th>Count</th><th>Est. Cost</th></tr>
            <!-- Cost breakdown -->
        </table>
    </div>
    
    <script>
        // Three.js 3D viewer initialization
        // Displays voxel map with placed nodes
        // Color-coded coverage depth
        // Interactive camera controls
    </script>
</body>
</html>
```

---

## 4. The Layered Strategy Output

The coverage solver produces placements that naturally fall into layered categories. The algorithm discovers optimal positions through geometry optimization rather than hard-coded rules.

### 4.1 Why Ceiling-Corner Positions Emerge

**Geometric advantage:**
- Unobstructed line-of-sight to most room volume
- No furniture between sensor and floor area
- Diagonal angle maximizes coverage spread

**Coverage analysis:**
- Single ceiling-corner node covers ~70-80% of typical room floor area
- Two opposing corners achieve >95% coverage
- Three corners provide depth-2 coverage for most voxels

**Sensor performance:**
- LiDAR: Full range utilization (no nearby obstructions)
- mmWave: Wide angular coverage without multipath from furniture
- Acoustic: Room-mode interaction from corner position

### 4.2 Why Low-Angle Positions Emerge

**Geometric insight:**
- Ceiling nodes cannot see under furniture
- Ray-march from ceiling hits furniture top, not floor beneath
- Floor-level node angled upward sees under furniture

**Priority triggers:**
- Pet ownership (pets hide under beds/couches)
- Young children (play on floor, under tables)
- Security concern (intruder hiding)
- Fall detection (elderly residents)

**Solver behavior:**
- Low-angle positions appear when under-furniture voxels have high priority weight
- Not required for every room; only where relevant

### 4.3 Why Choke-Point Positions Appear

**Geometric insight:**
- Doorways and hallways are geometrically constrained passages
- Every traversal must pass through
- Single node at choke point captures all traffic

**Priority multiplier:**
- Choke-point voxels receive 5× priority boost
- Any candidate observing a choke point contributes massively to weighted score
- AlwaysOn nodes (lowest cost) are sufficient for choke points

**Coverage efficiency:**
- Choke-point node costs ~$20
- Volume node costs ~$100-200
- Choke-point provides more "crossings detected per dollar" than volume coverage

---

## 5. Node Class Placement Strategy

### 5.1 AlwaysOn Node Placement

**Variant A — Choke Point Nodes:**

| Criterion | Value |
|-----------|-------|
| Position | Doorways, hallway entries, stair entries/landings |
| Height | 1.0-1.5 m (typical switch height) or ceiling |
| Coverage | Binary crossing detection only |
| Spacing | One per choke point |
| Priority | Highest (every home should have these at all entries) |

**Placement algorithm behavior:**
- Choke-point candidate positions pre-defined from doorway geometry
- Solver assigns AlwaysOn-A to these positions first
- Guarantees crossing detection regardless of volume coverage

**Variant B — Volume Presence Nodes:**

| Criterion | Value |
|-----------|-------|
| Position | Ceiling corners or wall-high (2.4-2.7 m) |
| Coverage | 360° horizontal, 90° vertical |
| Spacing | 2 per room for depth-2 coverage |
| Priority | High (primary sensing backbone) |

**Placement algorithm behavior:**
- Ceiling-corner candidates generated automatically
- Solver places at positions maximizing marginal coverage
- Prefers positions avoiding radar interference

**Variant C — Enhanced Acoustic Nodes:**

| Criterion | Value |
|-----------|-------|
| Position | Ceiling center or room center |
| Coverage | Omnidirectional acoustic |
| Use case | Rooms where acoustic material classification valuable |
| Examples | Kitchen (glass break), entry (door classification) |

**Variant D — Volume + Camera Nodes:**

| Criterion | Value |
|-----------|-------|
| Position | Ceiling or high-wall |
| Coverage | Volume + visual capture |
| Use case | Areas requiring visual monitoring |
| Priority | User-configured |

**Camera data routing:**
- All camera data transmits via mesh radio (BLE/PoE) to edge controller
- Edge controller processes, stores, or streams to companion app
- Camera-free variants A/B/C have no camera dependency at firmware level

### 5.2 LiDAR Node Placement

**Variant A — Basic Event-Triggered:**

| Criterion | Value |
|-----------|-------|
| Position | Ceiling corner (primary), ceiling center (secondary) |
| Height | 2.4-3.0 m |
| Coverage | 3D geometry, event-triggered fast transient |
| Spacing | 1 per room for geometry, 2 for depth |

**Placement optimization:**
- Maximize geometric coverage (voxel count)
- Minimize overlap redundancy
- Consider event camera field of regard

**Variant B — LiDAR + Radar:**

| Criterion | Value |
|-----------|-------|
| Position | Ceiling corner or center |
| Coverage | 3D geometry + fog/steam immunity |
| Use case | Kitchens, bathrooms, areas with atmospheric variability |

**Fusion advantage:**
- LiDAR fails in steam/fog
- mmWave continues operating
- Placement algorithm considers environmental zones

**Variant C — LiDAR + Camera:**

| Criterion | Value |
|-----------|-------|
| Position | Ceiling position with good room coverage |
| Coverage | 3D geometry + visual SLAM odometry |
| Camera orientation | Into room (not toward wall) |
| SLAM requirement | Camera FoV should overlap with another LiDAR node for loop closure |

**SLAM placement consideration:**
- Multiple LiDAR-C nodes should have overlapping coverage for loop closure
- Edge controller runs SLAM backend, requires point clouds + keyframes from all LiDAR nodes
- Placement algorithm includes SLAM connectivity as a constraint

**Variant D — LiDAR + 360° Camera Array:**

| Criterion | Value |
|-----------|-------|
| Position | Ceiling center (preferred) |
| Coverage | Full 360° visual + LiDAR geometry |
| Form factor | Circular ceiling mount |
| Bandwidth | Requires PoE 802.3at (25.5 W) for power + data |

**Placement optimization:**
- Variant D provides room-scale 360° coverage
- Single node sufficient for complete room geometry + visual
- Placement at ceiling center maximizes horizontal coverage
- Not suitable for ceiling corners (obstruction on corner sides)

**Why center placement for 360°:**
```
Ceiling Corner Placement:
┌─────────────────────┐
│ [Wall]    [Node]    │
│      ╲      │      │
│        ╲    │      │
│          ╲  │      │
│            ╲│      │
└─────────────────────┘
- One side blocked by wall
- 270° effective coverage

Ceiling Center Placement:
┌─────────────────────┐
│                     │
│       [Node]        │
│                     │
│                     │
│                     │
└─────────────────────┘
- Full 360° coverage
- No obstructions
```

### 5.3 Identity Node Placement

| Criterion | Value |
|-----------|-------|
| Position | Entry points (front door, back door, garage) |
| Height | 1.5-1.8 m (face-level) or ceiling |
| Coverage | Entry approach (1-3 m from door) |
| Privacy | Hardware power switch (optional) |

**Placement considerations:**
- Identity node camera should cover entry approach zone
- May be triggered by sensing layer (AlwaysOn node at same choke point)
- Placement algorithm treats identity nodes as separate layer (post-coverage optimization)

---

## 6. Dynamic Remap and Coverage Maintenance

### 6.1 Initial Calibration

**Walk-through auto-calibration:**

1. User enables Calibration Mode in companion app
2. User walks defined path through home (30-60 seconds)
3. Multiple nodes detect user simultaneously
4. Edge controller trilaterates exact node positions
5. Occlusion map generated automatically
6. Dynamic remap baseline established

**During walk-through:**
- Anklet IMUs provide accurate walking trajectory
- Multiple nodes provide distance estimates
- Edge controller fuses data to determine node positions relative to floor plan

### 6.2 Continuous Monitoring

**Baseline comparison:**

The edge controller continuously compares current sensor readings against the baseline voxel map:

| Comparison | Trigger | Response |
|------------|---------|----------|
| LiDAR scan discrepancy | Persistent difference over 5+ scans | Flag for remap evaluation |
| Radar detection in unexpected location | Potential obstruction moved | Update occupancy pattern |
| New acoustic reflection pattern | Furniture moved or added | Request user confirmation |

**Persistence filtering:**

A change must appear in the same voxels across N consecutive scans before triggering remap evaluation:
- Default N = 5 scans
- At 30-second scan interval: 2.5 minutes minimum
- Configurable based on environment stability

### 6.3 Remap Evaluation

**Feasibility check:**

When a persistent change is detected, the edge controller runs a feasibility check:

```
function evaluate_remap(change_event):
    affected_voxels = change_event.voxels
    existing_nodes = get_placed_nodes()
    
    # Calculate if existing nodes can still cover affected voxels
    for node in existing_nodes:
        node_visibility = recompute_visibility(node, change_event)
    
    uncovered_voxels = affected_voxels - union(node_visibility)
    
    if len(uncovered_voxels) > threshold:
        return RemapResult(
            status: FEASIBILITY_FAILED,
            uncovered_voxels: uncovered_voxels,
            suggested_placements: run_partial_solver(uncovered_voxels)
        )
    else:
        return RemapResult(
            status: FEASIBILITY_PASSED,
            coverage_degradation: calculate_degradation()
        )
```

**Partial re-optimization:**

If feasibility fails, the solver runs partial re-optimization targeting only the uncovered voxels:

- Restrict candidate positions to those near uncovered region
- Budget for additional nodes or node repositioning
- Minimize total changes while achieving coverage

### 6.4 User Notification

When remap is needed:

```json
{
  "event": "RemapRequired",
  "severity": "info",
  "details": {
    "zone": "living_room",
    "description": "Large object detected near south wall",
    "estimated_coverage_loss": 12.5,
    "suggested_actions": [
      {
        "type": "add_node",
        "suggested_class": "AlwaysOnB",
        "suggested_position": [3.2, 5.8, 2.5]
      },
      {
        "type": "adjust_node",
        "node_id": "lidar_living_room_north",
        "adjustment": "rotate 15° southwest"
      }
    ]
  }
}
```

### 6.5 360° Camera Contribution to Remap

LiDAR Variant D nodes with 360° camera arrays contribute enhanced remap information:

**Visual change detection:**
- Camera images detect surface-level changes (new object on shelf)
- LiDAR geometry alone may miss thin objects
- Combined detection improves remap accuracy

**Cross-referencing:**
- LiDAR geometry change + camera visual change = confirmed furniture move
- LiDAR change only = potential transient object
- Camera change only = surface change without geometry impact

**Implementation:**

```rust
// In aegis-placement::dynamic_remapper

fn evaluate_change_with_visual(
    &mut self,
    geometry_change: GeometryChange,
    visual_change: Option<VisualChange>,
) -> RemapDecision {
    match (geometry_change, visual_change) {
        (Some(geo), Some(vis)) if geo.location == vis.location => {
            // Confirmed furniture/object move
            RemapDecision::ConfirmedChange {
                voxels: geo.voxels,
                confidence: 0.95,
                action: RemapAction::UpdateMap,
            }
        },
        (Some(geo), None) => {
            // Geometry change without visual confirmation
            // Might be transient (person standing)
            RemapDecision::PendingConfirmation {
                voxels: geo.voxels,
                confidence: 0.6,
                observation_count: 0,
                threshold: 5,
            }
        },
        (None, Some(vis)) => {
            // Visual change without geometry impact
            // Surface-level change (wall art, decorations)
            RemapDecision::SurfaceChange {
                location: vis.location,
                action: RemapAction::NoMapUpdate,
            }
        },
        _ => RemapDecision::NoChange,
    }
}
```

---

## 7. Multi-Room and SLAM Integration

### 7.1 Room Transition Zones

For multi-room homes, the placement algorithm must ensure coverage continuity across room transitions:

**Doorway coverage:**
- Each doorway should be observed from both sides
- Enables track handoff between rooms
- Prevents tracking loss during room transition

**Hallway coverage:**
- Long hallways require multiple nodes
- Spacing based on sensor range and hallway width
- Consider radar interference in narrow spaces

**Stairwell coverage:**
- Stairs require coverage at top and bottom landings
- Intermediate coverage for longer stair runs
- Fall detection priority for stair voxels

### 7.2 SLAM Loop Closure Considerations

**For LiDAR Variant C nodes with cameras:**

Visual SLAM requires loop closure for drift correction. Placement algorithm considers:

- Overlapping camera fields of view between adjacent LiDAR-C nodes
- Common visual features in overlap regions
- Minimum overlap percentage for reliable loop closure

**Placement constraint:**

```rust
fn evaluate_slam_connectivity(node: &CandidateNode, existing: &[PlacedNode]) -> f32 {
    let mut slam_score = 0.0;
    
    for existing_node in existing {
        if existing_node.class == NodeClass::LidarC || existing_node.class == NodeClass::LidarD {
            // Calculate camera field of view overlap
            let overlap = calculate_camera_overlap(node, existing_node);
            
            if overlap > 0.15 {  // 15% minimum overlap for loop closure
                slam_score += overlap * 2.0;
            } else if overlap > 0.05 {
                slam_score += overlap * 0.5;  // Marginal benefit
            }
        }
    }
    
    slam_score
}
```

### 7.3 Variant D — 360° SLAM Coverage

LiDAR Variant D (360° camera array) simplifies SLAM placement:

**Single-node room coverage:**
- One Variant D node provides complete room geometry + 360° visual
- No overlap requirement within room
- Reduces total node count for SLAM-capable deployments

**Inter-room connectivity:**
- Variant D nodes in adjacent rooms still need visual overlap at doorways
- Camera orientation at doorway edges provides natural overlap
- Placement algorithm accounts for this when positioning Variant D nodes

**Edge controller SLAM backend:**

The edge controller (Linux SoM variant) runs:
- LIO-SAM or FAST-LIO2 for LiDAR SLAM
- ORB-SLAM3 or DBoW2 for visual SLAM
- Multi-node SLAM with loop closure across rooms

---

## 8. Radar Interference Analysis

### 8.1 Interference Zones

mmWave radar nodes operating at the same frequency can interfere when placed too close:

| Separation | Interference Risk | Recommended Action |
|------------|------------------|-------------------|
| > 5 m | Negligible | No mitigation needed |
| 2-5 m | Low | Monitor; TDM if issues observed |
| < 2 m | Moderate-High | Time-division multiplexing required |
| < 1 m | High | Avoid this placement; relocate |

### 8.2 Time-Division Multiplexing Configuration

If nodes must be placed within 2 m (e.g., small bathroom with both LiDAR-B and AlwaysOn-B):

```toml
[radar.interference]
mitigation = "tdm"              # "none" | "tdm" | "frequency_division"

[radar.interference.tdm]
schedule = [
    { node_id = "always_on_bathroom", time_slot_ms = "0-50" },
    { node_id = "lidar_bathroom", time_slot_ms = "50-100" },
]
cycle_period_ms = 100
sync_method = "edge_controller_broadcast"
```

### 8.3 Placement Solver Integration

The placement solver applies interference penalties:

```rust
fn calculate_interference_penalty(
    candidate: &CandidateNode,
    existing: &[PlacedNode],
) -> f32 {
    let mut penalty = 1.0;
    
    for node in existing {
        if node.has_mmwave_radar() && candidate.has_mmwave_radar() {
            let distance = (node.position - candidate.position).norm();
            
            match distance {
                d if d < 1.0 => penalty *= 0.1,   // Severe penalty
                d if d < 2.0 => penalty *= 0.5,   // Moderate penalty
                d if d < 5.0 => penalty *= 0.9,   // Minor penalty
                _ => {},                          // No penalty
            }
        }
    }
    
    penalty
}
```

---

## 9. Coverage Depth Analysis

### 9.1 Depth-1 Coverage

**Definition:** Each voxel observed by at least one node.

**Use cases:**
- Basic presence detection
- Low-budget deployments
- Non-critical monitoring

**Limitations:**
- Single point of failure per voxel
- No redundancy during node maintenance
- Reduced tracking confidence

### 9.2 Depth-2 Coverage (Recommended Default)

**Definition:** Each voxel observed by at least two nodes.

**Advantages:**
- Redundancy against single-node failure
- Improved tracking confidence from multiple perspectives
- Better occlusion handling (if one node blocked, another sees)

**Cost impact:**
- Approximately 2× node count vs depth-1
- Trade-off: fewer rooms with depth-2 vs more rooms with depth-1

### 9.3 Depth-3+ Coverage

**Definition:** Each voxel observed by three or more nodes.

**Use cases:**
- High-security areas (entries, valuables storage)
- Critical zones (baby rooms, elderly resident areas)
- Maximum confidence tracking

**Implementation:**
- Selective application (not whole-home depth-3)
- Zone-specific depth configuration

**Configuration:**

```toml
[coverage]
default_depth = 2
zones = [
    { name = "front_entry", depth = 3 },
    { name = "master_bedroom", depth = 3 },
    { name = "living_room", depth = 2 },
    { name = "garage", depth = 2 },
    { name = "utility", depth = 1 },
]
```

---

## 10. Environmental Considerations

### 10.1 Atmospheric Zones

**Steam-prone areas:**
- Kitchens, bathrooms, laundry rooms
- LiDAR degraded in steam
- Placement favors mmWave + acoustic

**Dust-prone areas:**
- Garages, workshops
- Optical sensors affected
- Placement favors radar-dominant coverage

**Configuration:**

```toml
[environmental_zones]
[[environmental_zones.zones]]
name = "kitchen"
steam_risk = "high"
dust_risk = "low"
prefer_radar = true
prefer_lidar = false

[[environmental_zones.zones]]
name = "bathroom"
steam_risk = "high"
dust_risk = "none"
prefer_radar = true
prefer_lidar = false

[[environmental_zones.zones]]
name = "garage"
steam_risk = "none"
dust_risk = "high"
prefer_radar = true
prefer_lidar = false
```

### 10.2 Material Penetration

mmWave radar penetrates different materials at varying effectiveness:

| Material | Penetration | Impact on Placement |
|----------|-------------|---------------------|
| Drywall | Excellent | No constraint |
| Wood (thin) | Good | Minor consideration |
| Glass | Poor (reflection) | Avoid direct line-of-sight through glass |
| Metal | None | Creates shadow zones |
| Ceramic tile | Poor | Reduced effective range |

**Placement algorithm integration:**

```rust
fn calculate_material_penalty(
    ray: &Ray,
    voxel_map: &VoxelMap,
) -> f32 {
    let mut penetration_score = 1.0;
    
    for voxel in ray.traverse(voxel_map) {
        match voxel.material {
            Material::Drywall => {},  // No penalty
            Material::Glass => penetration_score *= 0.7,
            Material::Metal => return 0.0,  // Ray blocked
            Material::Ceramic => penetration_score *= 0.8,
            Material::WoodThin => penetration_score *= 0.95,
            _ => {},
        }
    }
    
    penetration_score
}
```

---

## 11. Edge Controller Integration

### 11.1 Edge Controller Role in Placement

The edge controller serves as the central hub for all placement operations:

| Function | Edge Controller Responsibility |
|----------|-------------------------------|
| Voxel map storage | Maintains persistent copy of home geometry |
| Solver execution | Runs coverage optimization |
| Report generation | Produces HTML placement reports |
| Dynamic remap | Monitors for geometry changes |
| SLAM backend | Processes multi-node SLAM data |
| Companion app API | Serves placement data to app |

### 11.2 Placement Optimization Workflow

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    PLACEMENT OPTIMIZATION WORKFLOW                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  1. GEOMETRY CAPTURE                                                        │
│     User performs walk-through scan with mobile LiDAR                       │
│     OR imports floor plan / CAD                                             │
│     Data: Point cloud or 2D plan                                           │
│              │                                                               │
│              ▼                                                               │
│  2. VOXEL MAP CONSTRUCTION                                                  │
│     Edge controller:                                                        │
│     - Voxelizes input data                                                  │
│     - Classifies voxels (Open/Structure/Obstruction)                        │
│     - Assigns priorities                                                    │
│     Data: VoxelMap struct                                                   │
│              │                                                               │
│              ▼                                                               │
│  3. CANDIDATE GENERATION                                                    │
│     Edge controller:                                                        │
│     - Generates candidate positions (grid + user refinement)                │
│     - Filters infeasible positions (no power, blocked, aesthetic)           │
│     Data: Vec<CandidatePosition>                                            │
│              │                                                               │
│              ▼                                                               │
│  4. VISIBILITY COMPUTATION                                                  │
│     Edge controller:                                                        │
│     - For each candidate, compute visible voxels                            │
│     - Store in OcclusionMap                                                 │
│     Data: HashMap<Position, HashSet<Voxel>>                                 │
│              │                                                               │
│              ▼                                                               │
│  5. COVERAGE OPTIMIZATION                                                   │
│     Edge controller:                                                        │
│     - Run greedy weighted set cover                                         │
│     - Apply constraints (budget, class, interference)                      │
│     - Iterate until budget exhausted or coverage achieved                   │
│     Data: Vec<PlacedNode>                                                   │
│              │                                                               │
│              ▼                                                               │
│  6. REPORT GENERATION                                                       │
│     Edge controller:                                                        │
│     - Generate HTML report                                                  │
│     - Create 3D visualization                                               │
│     - List blind spots and warnings                                         │
│     Data: HTML report + visualization assets                                │
│              │                                                               │
│              ▼                                                               │
│  7. USER REVIEW                                                             │
│     Companion app:                                                          │
│     - Display report to user                                                │
│     - Allow position adjustment                                             │
│     - Confirm placement                                                     │
│              │                                                               │
│              ▼                                                               │
│  8. DEPLOYMENT                                                              │
│     User installs nodes at recommended positions                            │
│     Edge controller calibrates actual positions                             │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 11.3 API Endpoints for Placement

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/placement/scan/start` | Begin geometry scan |
| GET | `/api/placement/scan/status` | Scan progress |
| POST | `/api/placement/scan/complete` | Finalize scan and build voxel map |
| GET | `/api/placement/voxel-map` | Download voxel map |
| POST | `/api/placement/solve` | Run coverage solver |
| GET | `/api/placement/report` | Get placement report (HTML) |
| GET | `/api/placement/report/3d` | Get 3D visualization data |
| POST | `/api/placement/candidates` | Submit manual candidate positions |
| GET | `/api/placement/remap/status` | Check for geometry changes |
| POST | `/api/placement/remap/apply` | Apply suggested changes |

---

## 12. Configuration Reference

### 12.1 Placement Configuration

```toml
# aegis-mesh.toml

[placement]
# Voxel map parameters
voxel_resolution_m = 0.25
min_ceiling_height_m = 2.4
max_ceiling_height_m = 4.0

# Candidate generation
candidate_grid_spacing_m = 0.5
candidate_heights_m = [1.5, 2.0, 2.5, 2.8]  # Mount height options
prefer_ceiling_corner = true

# Coverage optimization
default_coverage_depth = 2
max_nodes = 50
solver_budget_iterations = 10000
solver_timeout_seconds = 60

# Node class budgets
[placement.budget]
always_on_a = { max = 20, min = 1 }   # Choke points
always_on_b = { max = 30, min = 1 }   # Volume presence
always_on_c = { max = 10, min = 0 }   # Enhanced acoustic
always_on_d = { max = 10, min = 0 }   # Volume + camera
lidar_a = { max = 20, min = 0 }
lidar_b = { max = 15, min = 0 }
lidar_c = { max = 10, min = 0 }
lidar_d = { max = 5, min = 0 }        # 360° nodes
identity = { max = 5, min = 0 }

# Voxel priorities
[placement.priorities]
floor_level_multiplier = 2.0
entry_proximity_m = 5.0
entry_proximity_multiplier = 2.0
choke_point_multiplier = 5.0
hazard_zone_multiplier = 2.0

# Environmental zones
[[placement.environmental_zones]]
name = "kitchen"
steam_risk = "high"
prefer_radar = true

[[placement.environmental_zones]]
name = "bathroom"
steam_risk = "high"
prefer_radar = true

[[placement.environmental_zones]]
name = "garage"
dust_risk = "high"
prefer_radar = true

# Zone-specific coverage
[[placement.zones]]
name = "front_entry"
depth = 3
description = "Primary entry point - highest security priority"

[[placement.zones]]
name = "master_bedroom"
depth = 3
description = "Primary sleeping area"

[[placement.zones]]
name = "living_room"
depth = 2
description = "Main living space"

[[placement.zones]]
name = "kitchen"
depth = 2
description = "Kitchen with steam considerations"

[[placement.zones]]
name = "utility"
depth = 1
description = "Utility room - lower priority"

# Interference mitigation
[placement.interference]
min_radar_separation_m = 2.0
interference_penalty = 0.5
tdm_required_below_m = 1.5

# Dynamic remap
[placement.dynamic_remap]
enabled = true
scan_interval_seconds = 30
persistence_scan_count = 5
auto_update_voxel_map = true
notification_on_change = true

# SLAM integration
[placement.slam]
enable_loop_closure_check = true
min_visual_overlap_fraction = 0.15
prefer_variant_d_for_large_rooms = true
```

---

## 13. Testing and Validation

### 13.1 Coverage Validation

After deployment, validate coverage:

```bash
# Via edge controller API
curl -X POST http://aegis-mesh.local/api/placement/validate

# Response
{
  "overall_coverage": 0.94,
  "depth_1_voxels": 1.0,
  "depth_2_voxels": 0.87,
  "depth_3_voxels": 0.45,
  "blind_spots": [
    {
      "voxel": [3.2, 5.1, 0.15],
      "volume_m3": 0.0156,
      "reason": "Under sectional sofa"
    }
  ],
  "recommendations": [
    {
      "action": "add_low_angle_node",
      "suggested_position": [3.5, 5.0, 0.3],
      "reason": "Under-sofa coverage"
    }
  ]
}
```

### 13.2 Node Position Verification

During calibration walk:

| Metric | Target | Validation Method |
|--------|--------|-------------------|
| Position error | < 20 cm | Compare actual vs calculated position |
| Orientation error | < 10° | IMU heading comparison |
| Coverage match | > 95% | Actual vs predicted visibility map |

### 13.3 Interference Testing

| Test | Method | Acceptance |
|------|--------|------------|
| Radar-to-radar | Power spectral analysis | No spurious peaks |
| Detection accuracy | Known target walk-through | < 5% false negative |
| Tracking continuity | Multi-room walk | No track loss at doorways |

---

## 14. Summary

Occlusion-driven placement in AEGIS-MESH follows a geometry-first approach:

1. **Voxel map construction** captures true 3D geometry
2. **Visibility computation** determines what each candidate position can observe
3. **Coverage optimization** selects positions for maximum weighted coverage
4. **Dynamic remap** maintains coverage as furniture changes
5. **SLAM integration** ensures multi-node coherence for 3D world model
6. **Edge controller** serves as compute hub for all placement operations

The algorithm discovers optimal positions rather than applying hard-coded rules, adapting to each home's unique geometry and furniture layout. All node classes integrate into a unified coverage model, with the edge controller serving as the sole gateway for data aggregation, SLAM processing, and companion app communication.

---

**End of Document**
