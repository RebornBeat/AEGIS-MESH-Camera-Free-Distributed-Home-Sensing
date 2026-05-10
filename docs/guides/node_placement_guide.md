# Node Placement Guide — AEGIS-MESH

**Project:** AEGIS-MESH
**Domain:** Geometry-driven distributed residential sensing deployment
**Implementation:** `crates/aegis-placement/`

---

## 1. Core Philosophy: Geometry-First, Not Grid-First

The most common mistake in sensor placement is treating the home as a uniform grid and dividing it evenly. This wastes hardware on redundant coverage of the same areas while leaving furniture-induced blind spots unaddressed.

AEGIS-MESH's placement approach is **occlusion-driven**: we first map the home's actual 3D geometry, identify where the coverage gaps are, and place nodes specifically to address those gaps with the minimum number of nodes.

### 1.1 The Geometry-First Principle

**Traditional approach (grid-based):**
- Divide floor plan into equal cells
- Place sensors at grid intersections
- Result: Overlapping coverage in open areas, gaps behind furniture, redundant nodes in hallways

**AEGIS-MESH approach (occlusion-driven):**
- Map 3D geometry including all obstructions
- Compute visibility cones from candidate positions
- Select minimum set of positions that cover all high-priority voxels
- Result: Optimal coverage with minimum hardware

### 1.2 The Single Gateway Principle

**Critical architectural constraint:** All sensor nodes connect to the edge controller via mesh radio. The edge controller is the sole node with WiFi, cellular, and external connectivity.

**Placement implication:** The edge controller should be centrally located for:
- Optimal mesh connectivity to all nodes
- Access to wired Ethernet (recommended)
- Proximity to WiFi router for companion app connectivity
- Secure location (not in public areas)

---

## 2. Node Architecture Overview

### 2.1 Node Classes

| Node Class | Primary Role | Typical Position | Mesh Radio |
|------------|--------------|------------------|------------|
| **Always-On Node** | Continuous presence sensing | Ceiling corners, choke points | BLE/Thread |
| **LiDAR Node** | High-resolution geometry, SLAM | Ceiling positions | PoE/BLE |
| **Identity Node** | Visual identification | Entry points | BLE/Thread |
| **Edge Controller** | Central compute, sole gateway | Central location | WiFi + Cellular + BLE |

### 2.2 Node Variants and Placement Implications

#### Always-On Node Variants

| Variant | Sensors | Placement Strategy |
|---------|---------|-------------------|
| A — Choke Point | PIR + 1D ToF | Doorways, hallways, stair entries |
| B — Volume Presence | mmWave + Acoustic | Ceiling corners, room centers |
| C — Enhanced Acoustic | mmWave + 6-element mic array | Areas requiring acoustic detail |
| D — Volume + Camera | All Variant B + optional camera | Areas needing visual capture |

#### LiDAR Node Variants

| Variant | Sensors | Placement Strategy |
|---------|---------|-------------------|
| A — Basic | LiDAR + Event camera | Standard ceiling positions |
| B — Full | LiDAR + Event + mmWave | Areas needing fog/smoke robustness |
| C — LiDAR + Camera | All Variant B + conventional camera | Areas needing visual SLAM |
| D — LiDAR + 360° Camera | All Variant B + 4-8 camera array | Center-ceiling for full room capture |

#### Identity Node Variants

| Variant | Configuration | Placement Strategy |
|---------|--------------|-------------------|
| A — Metadata-Only | Classification tags only | Entry points |
| B — Local Storage | SD card recording | Entry points with evidence needs |
| C — App Streaming | Live stream capability | Active monitoring areas |
| D — Multi-Camera Array | Multiple camera angles | High-traffic entry points |

### 2.3 Edge Controller — The Central Hub

**The edge controller is NOT a sensor node.** It is the central compute and communication hub for the entire mesh.

**Hardware variants:**

| Variant | Compute Platform | Use Case |
|---------|-----------------|----------|
| A — SBC | Raspberry Pi 4/5 class | Basic deployment, 2-4 nodes |
| B — ARM with NPU | Jetson Nano/Orin class | Full SLAM, 8+ nodes, multi-camera |
| C — x86 Mini PC | Intel/AMD based | Maximum flexibility, unlimited nodes |

**Placement requirements:**
- Central location (minimize average distance to all nodes)
- Ethernet connection (PoE aggregator for LiDAR nodes)
- Proximity to WiFi router
- Physical security (locked room or cabinet)
- Power outlet access
- Optional cellular antenna placement

---

## 3. The Placement Process

### 3.1 Phase 1: One-Time Geometry Scan

Walk through the home with the AEGIS-MESH app running. The app uses whichever sensor nodes are already installed (or a mobile LiDAR scan if no nodes are installed yet) to build a 3D voxel map of the space.

**What the scan captures:**
- Wall positions and thickness
- Furniture height, width, and placement
- Doorways, openings, archways
- Ceiling height variations, dropped ceilings, beams
- Stairs, landings, split levels
- Windows, glass partitions
- Built-in cabinets, shelving

**Output:** `VoxelMap` file saved to disk

**When to rescan:**
- Major furniture rearrangement
- Renovation or structural changes
- Significant new obstructions

**Scan procedure:**
```bash
# Start scan from companion app
# Or via CLI:
aegis-cli scan-start --output home.voxelmap

# Walk through home at normal pace (30-60 seconds)
# Cover all rooms, open doors, scan behind furniture if accessible

# Complete scan
aegis-cli scan-complete
```

### 3.2 Phase 2: Coverage Solver

Run the placement solver with your constraints:

```bash
cargo run --bin aegis-cli -- placement-solve \
  --voxel-map home.voxelmap \
  --budget 12 \
  --min-coverage-depth 2 \
  --node-class-budget "always_on=8,lidar=3,identity=1" \
  --output placement_report.html
```

**Solver outputs:**

| Output | Description |
|--------|-------------|
| Node positions | [x, y, z] coordinates in home frame |
| Node class assignment | AlwaysOn, LidarNode, or IdentityNode |
| Node variant recommendation | Specific variant per position |
| Coverage heatmap | Voxel-level coverage depth visualization |
| Blind-spot report | Uncovered high-priority voxels |
| Interference warnings | Nodes requiring time-division multiplexing |
| Cost estimate | Hardware cost based on node counts |
| Edge controller placement | Recommended location for hub |

### 3.3 Phase 3: Edge Controller Placement

The solver also recommends edge controller placement:

**Optimization criteria:**
- Minimize average distance to all nodes (mesh latency)
- Maximize mesh radio signal quality to all nodes
- Proximity to Ethernet infrastructure (for PoE nodes)
- Proximity to WiFi router (for companion app connectivity)
- Physical security considerations

**Placement options:**
- **Central closet:** Ideal for single-floor homes
- **Mechanical room:** Good for infrastructure integration
- **Home office:** Convenient for physical access
- **Utility room:** Often central, may have noise/interference issues

**Configuration:**
```toml
[edge_controller]
placement = "central_closet"
ethernet_enabled = true
wifi_enabled = true
cellular_enabled = false
battery_backup_hours = 4

[edge_controller.network]
# PoE aggregator for LiDAR nodes
poe_ports = 4

# Mesh radio configuration
ble_enabled = true
thread_enabled = false
```

### 3.4 Phase 4: Node Installation

Install nodes at or near recommended positions:

1. **Mark positions** from solver output on floor/ceiling
2. **Verify clearance** for sensor fields of view
3. **Run cables** (power, PoE for LiDAR nodes)
4. **Mount nodes** using appropriate hardware
5. **Connect to mesh** via companion app discovery

### 3.5 Phase 5: Calibration Walk

After installation, perform walk-through calibration:

```bash
# Start calibration
curl -X POST http://aegis-mesh.local/api/calibration/start

# Walk defined path through home (30-60 seconds)
# Cover all rooms, pass by each node

# Complete calibration
curl -X POST http://aegis-mesh.local/api/calibration/complete

# Verify calibration
curl http://aegis-mesh.local/api/calibration/status
```

**Calibration establishes:**
- Node positions relative to home frame
- Inter-node distances
- Initial occlusion map
- Baseline coverage depth

### 3.6 Phase 6: Iterative Refinement

After calibration, the system reports:
- Actual vs. planned coverage
- Calibration confidence per node
- Remaining blind spots

**Refinement actions:**
- Adjust node positions slightly for better coverage
- Add nodes for uncovered blind spots
- Remove redundant nodes if coverage exceeds requirements

---

## 4. The Layered Placement Strategy

### 4.1 Layer 1: Ceiling-Corner Nodes (Primary)

**Position:** Near room corners, mounted to the ceiling. Offset 30–60 cm from the corner and angled diagonally across the room.

**Why diagonal:**
- A camera pointed straight down only sees directly below
- A node angled diagonally sees the near half at high resolution and the far half at lower resolution
- Two diagonally opposing ceiling nodes cover the entire room with overlapping views

**Sensor complement:**
- **Always-On Node Variant B or C:** mmWave radar + microphone array
- **LiDAR Node Variants A or B:** Solid-state LiDAR + event camera + optional mmWave

**Coverage contribution:** 70–80% of room volume

**Why ceiling corners:**
- Unobstructed view of room volume
- Above furniture height (resilient to rearrangement)
- Clear line-of-sight to most areas
- Optimal for mmWave radar (RF propagates through light furniture)

**Typical count:**
- Small room (< 3×3 m): 1 node
- Medium room (3×3 m to 5×5 m): 2 nodes
- Large room (5×5 m to 8×8 m): 2–3 nodes
- Open-plan (> 8×8 m): 3–4 nodes

**Placement solver weight:** Ceiling-corner candidate positions receive geometric preference in the optimization.

### 4.2 Layer 2: Low-Angle Nodes (Occlusion Breakers)

**Position:** 20–50 cm above floor level, mounted on walls or furniture edges, oriented horizontally.

**Why this height:**
- Furniture creates occlusion at floor-to-furniture-top height
- Ceiling nodes cannot see under couches, beds, tables
- Low-angle nodes see into these spaces
- Critical for pet detection, fallen object detection

**Sensor complement:**
- **Always-On Node Variant B:** mmWave radar (sees through fabric)
- Optional short-range ToF for geometric refinement

**Placement criterion:** Only where occlusion analysis identifies floor-level gaps

**Typical count:** 1–2 per large room with heavy furniture

**Coverage contribution:** Addresses specific blind spots; not a coverage multiplier

### 4.3 Layer 3: Choke-Point Nodes (High-Value, Low-Cost)

**Position:** Doorways, hallways, stair entries — any location all traffic must pass through.

**Why choke points:**
- Everything passes through doorways
- High-confidence crossing events
- Track handoff between rooms
- Minimal hardware cost for high tracking value

**Sensor complement:**
- **Always-On Node Variant A:** PIR + 1D ToF
- Optional: mmWave presence module

**Coverage contribution:** Not volumetric — provides crossing events

**Typical count:** 1 per doorway or hallway entry

**Placement considerations:**
- Mount on door frame or adjacent wall at ~1 m height
- Field of view should cover entire doorway width
- Avoid placing where door swing blocks view

### 4.4 Layer 4: 360° LiDAR Nodes (Optional — Maximum Coverage)

**Position:** Center-ceiling, providing complete 360° horizontal coverage.

**Why center-ceiling for 360°:**
- Corner placement creates obstruction on corner-facing side
- Center position has clear view in all directions
- Single node can cover entire room with no blind spots

**Sensor complement:**
- **LiDAR Node Variant D:** Solid-state LiDAR + 4–8 camera array + mmWave

**Coverage contribution:** Complete room coverage from single node

**Tradeoffs:**
- Higher cost than standard LiDAR nodes
- Higher bandwidth (requires PoE)
- Higher power consumption
- More complex installation

**When to use:**
- Large open-plan spaces
- Areas requiring complete visual coverage
- High-security zones
- Legal evidence capture requirements

### 4.5 Layer 5: Identity Nodes (Optional — Entry Points)

**Position:** Entry points — front door, back door, garage entry.

**Why entry points:**
- Natural identification checkpoint
- All visitors pass through
- Entry/exit events provide context for tracking

**Sensor complement:**
- **Identity Node (any variant):** Configurable camera/biometric sensor

**Coverage contribution:** Identification, not volumetric coverage

**Placement considerations:**
- Camera field of view should cover entry approach (1–3 m in front of door)
- Outdoor-rated enclosure for exterior entries
- Position to avoid direct sunlight on lens
- Physical security (vandal-resistant enclosure if public-facing)

---

## 5. Node Variant Placement Guide

### 5.1 Always-On Node Placement

#### Variant A — Choke Point

**Optimal positions:**
- Door frames (mount at ~1 m height)
- Hallway intersections
- Stair landings
- Room transitions

**Placement checklist:**
- [ ] Field of view covers entire doorway/hallway width
- [ ] Not blocked by door swing
- [ ] PIR window faces traffic direction
- [ ] Mesh connectivity verified (signal strength > -70 dBm to edge controller or nearest hub)

**Avoid:**
- Positions where doors block view when open
- Locations with direct sunlight on PIR
- Areas with strong HVAC airflow

#### Variant B — Volume Presence

**Optimal positions:**
- Ceiling corners (primary)
- Ceiling center (secondary)
- Wall mounts at 2 m height (alternative)

**Placement checklist:**
- [ ] Clear line of sight to most of room volume
- [ ] Radar antenna not blocked by ceiling fixtures
- [ ] Microphone array not near noise sources (HVAC, appliances)
- [ ] Mesh connectivity verified

**Radar antenna orientation:**
- Angle radar to cover primary room volume
- Avoid pointing at walls at close range (< 0.5 m)
- Minimize reflections from metal objects

#### Variant C — Enhanced Acoustic

**Optimal positions:**
- Ceiling center for acoustic coverage
- Away from HVAC vents
- Away from noisy appliances

**Placement checklist:**
- [ ] All Variant B requirements
- [ ] No HVAC vent within 1 m
- [ ] No appliance noise within 2 m
- [ ] Circular microphone array has clear acoustic path

#### Variant D — Volume + Camera

**Optimal positions:**
- Same as Variant B
- Additional consideration for camera field of view

**Placement checklist:**
- [ ] All Variant B requirements
- [ ] Camera field of view covers intended area
- [ ] Camera not pointed at bright windows
- [ ] Privacy considerations: avoid pointing at bedrooms, bathrooms

**Camera placement guidelines:**
- Angle 15–30° downward from horizontal
- Field of view overlap with adjacent nodes for tracking continuity
- Minimum 2 m distance from typical subject position

### 5.2 LiDAR Node Placement

#### Variant A — Basic LiDAR

**Optimal positions:**
- Ceiling corners
- Ceiling center for smaller rooms

**Placement checklist:**
- [ ] Clear 360° horizontal field of view (LiDAR typically hemispherical)
- [ ] No vertical obstructions (hanging lights, ceiling fans)
- [ ] Mesh connectivity verified (BLE or PoE)
- [ ] Event camera field of view clear

**LiDAR-specific considerations:**
- Minimum 0.5 m clearance to walls for near-field accuracy
- Avoid specular surfaces in primary field of view
- Consider thermal expansion of ceiling for long-term stability

#### Variant B — Full LiDAR + Radar

**Optimal positions:**
- Same as Variant A
- Additional mmWave radar for fog/smoke environments

**Placement checklist:**
- [ ] All Variant A requirements
- [ ] Radar and LiDAR fields of view aligned
- [ ] Radar not interfered by nearby metal objects

**Use case:** Kitchens, bathrooms, areas with steam or smoke potential

#### Variant C — LiDAR + Camera

**Optimal positions:**
- Ceiling positions with good visual coverage
- Overlap with other nodes for visual odometry

**Placement checklist:**
- [ ] All Variant B requirements
- [ ] Camera covers area of interest
- [ ] Camera not facing bright windows
- [ ] PoE connection for bandwidth (recommended)

**SLAM considerations:**
- Camera provides visual odometry for SLAM
- Position to see distinctive visual features
- Overlap with other LiDAR nodes improves loop closure

#### Variant D — LiDAR + 360° Camera Array

**Optimal positions:**
- Ceiling center (not corners)
- No obstructions in any direction

**Placement checklist:**
- [ ] All Variant C requirements
- [ ] 360° horizontal clearance
- [ ] PoE 802.3at (25.5 W) available
- [ ] Edge controller can handle bandwidth

**360° camera specifics:**
- Center-ceiling placement only (corners obstruct half the view)
- Minimum 2 m ceiling height recommended
- All N cameras must have clear view
- FSYNC synchronization verified

**Bandwidth requirements:**
- 4 cameras at 720p: ~4-8 Mbps
- 8 cameras at 720p: ~8-15 Mbps
- Requires PoE connection (BLE insufficient)

**Stitching configuration:**
- Performed on edge controller (recommended)
- Or on-node with dedicated vision processor

### 5.3 Identity Node Placement

#### All Variants

**Optimal positions:**
- Entry points: front door, back door, garage
- Verification zones: reception areas, security checkpoints
- High-traffic areas: main hallway, living room entry

**Placement checklist:**
- [ ] Camera/biometric sensor covers entry approach
- [ ] Field of view: 1–3 m in front of entry
- [ ] Not facing direct sunlight
- [ ] Outdoor-rated enclosure if exterior
- [ ] Mesh connectivity verified
- [ ] Physical security: tamper-resistant mount

**Outdoor placement:**
- Weatherproof enclosure (IP66 or better)
- Heated camera window for condensation/frost
- UV-resistant housing
- Secure mounting (vandal-resistant)
- Covered position preferred (porch, awning)

**Privacy considerations:**
- Position to capture entries only
- Avoid capturing public areas
- Consider legal requirements for signage
- Check local recording consent laws

### 5.4 Edge Controller Placement

**The edge controller is NOT placed by the coverage solver.** It requires manual placement based on infrastructure considerations.

**Optimal position criteria:**

| Criterion | Target | Reason |
|-----------|--------|--------|
| Central location | Minimize average distance to nodes | Mesh latency |
| Ethernet access | Cat 6+ wiring | PoE for LiDAR nodes, backhaul |
| WiFi router proximity | Same room or adjacent | Companion app connectivity |
| Power access | Dedicated circuit preferred | Reliability |
| Physical security | Locked room/cabinet | Tamper protection |
| Environmental | 10–35°C, dry | Hardware longevity |
| Radio clear | Away from metal, electronics | Mesh signal quality |

**Placement options:**

**Option A: Central Closet**
- Pros: Central, secure, existing infrastructure
- Cons: May need ventilation, limited space

**Option B: Utility/Mechanical Room**
- Pros: Power, infrastructure, central
- Cons: Noise, vibration, may need cabinet

**Option C: Home Office**
- Pros: Accessible, network infrastructure likely present
- Cons: May not be central, user activity in space

**Option D: Network Cabinet (Recommended for larger deployments)**
- Pros: Professional installation, ventilation, security
- Cons: Requires dedicated cabinet installation

**PoE aggregator placement:**
- For LiDAR nodes with PoE
- Place near edge controller
- Plan cable runs to LiDAR node positions
- Consider PoE switch vs. dedicated PoE injector per node

---

## 6. Placement Rules of Thumb

### 6.1 Ceiling Height Considerations

**Standard ceiling (2.4–2.7 m):**
- Ceiling-corner nodes work well
- Low-angle nodes may be needed for under-furniture coverage

**High ceiling (> 3 m):**
- Creates larger shadow cones beneath ceiling nodes
- Increase low-angle node count
- Consider lowering ceiling node position (suspend mount)

**Cathedral/sloped ceiling:**
- Place nodes at highest point of slope
- May need additional nodes for lower ceiling areas

### 6.2 Interference Avoidance

**mmWave radar interference:**
- Nodes within 2 m may interfere
- Use time-division multiplexing (configured automatically)
- Physical separation preferred when possible

**Acoustic interference:**
- Avoid placing near HVAC vents
- Avoid placing near appliances (refrigerators, dishwashers)
- Minimum 2 m from noise sources

**RF interference:**
- Keep mesh radio nodes away from WiFi routers (if on 2.4 GHz)
- Keep away from microwave ovens
- Metal objects near radar antennas degrade performance

### 6.3 Room-Specific Guidance

**Living Room:**
- 2 ceiling-corner Always-On nodes (or LiDAR nodes)
- 1 low-angle node if heavy furniture
- Total: 2–3 nodes

**Kitchen:**
- 1 ceiling-corner node (LiDAR Variant B for steam robustness)
- Keep away from stove, sink
- Total: 1 node

**Bedroom:**
- 1–2 ceiling-corner Always-On nodes
- Privacy: avoid cameras unless user-configured
- Total: 1–2 nodes

**Bathroom:**
- 1 ceiling node (LiDAR Variant B for steam)
- Privacy: no cameras recommended
- IP-rated enclosure if steam is significant
- Total: 1 node

**Hallway:**
- Choke-point node at entry/exit
- Spacing: 1 node per 3–4 m of hallway length
- Total: depends on hallway length

**Garage:**
- 1–2 ceiling nodes for vehicle detection
- Choke-point node at garage door
- Identity node at garage entry if desired
- Total: 2–3 nodes

**Entry/Mudroom:**
- Choke-point node at doorway
- Identity node for entry identification
- Total: 1–2 nodes

**Stairs:**
- Choke-point node at top and bottom
- Low-angle node at mid-point if long run
- Total: 2–3 nodes

---

## 7. Dynamic Remap: When Furniture Moves

### 7.1 Automatic Detection

After initial placement and calibration, AEGIS-MESH continuously compares current sensor readings against the baseline voxel map.

**Detection process:**
1. LiDAR nodes perform low-duty-cycle background scans
2. System compares current occupancy to baseline
3. Persistent changes trigger remap analysis

**Change detection threshold:**
```toml
[dynamic_remap]
scan_interval_seconds = 30
persistence_scan_count = 5        # Require 5 consecutive scans
voxel_change_threshold = 0.15     # 15% of voxels changed in region
notification_delay_minutes = 2.5
auto_update_voxel_map = true
```

### 7.2 Remap Events

When significant change detected:

1. **VoxelMap update:** Affected voxels relabeled
2. **Coverage recomputation:** Solver re-evaluates coverage
3. **User notification:** Dashboard shows affected areas
4. **Placement suggestions:** If new blind spots created

**Event types:**
- `FurnitureMoved`: Large object repositioned
- `ObstructionAdded`: New blocking object
- `ObstructionRemoved`: Previously blocked area now visible
- `CoverageDegraded`: Blind spot created
- `CoverageImproved`: Previous blind spot now covered

### 7.3 Manual Remap Trigger

```bash
# Trigger immediate remap
curl -X POST http://aegis-mesh.local/api/calibration/remap

# Or via companion app
# Settings → System → Calibration → Force Remap
```

### 7.4 Companion App Integration

When remap event occurs, companion app displays:
- Affected zones highlighted
- Before/after coverage comparison
- Recommended placement adjustments
- Option to dismiss if change is temporary

---

## 8. Multi-Floor Deployment

### 8.1 Floor Separation

Each floor operates as a semi-independent mesh:
- Nodes on same floor communicate via mesh radio
- Edge controller coordinates across floors
- Stairway nodes provide floor-to-floor track handoff

### 8.2 Floor-Specific Edge Controllers

For large multi-floor homes:
- One edge controller per floor (recommended)
- Cross-floor coordination via network
- Alternatively, single edge controller with sufficient mesh range

### 8.3 Stairway Placement

**Critical for track continuity:**
- Choke-point node at top of stairs
- Choke-point node at bottom of stairs
- LiDAR node on stairway wall (for fall detection)

**Configuration:**
```toml
[[nodes]]
id = "stairs_top"
class = "AlwaysOnNode"
variant = "A"
position_m = [0.5, 3.0, 3.0]
floor = 1
role = "choke_point"

[[nodes]]
id = "stairs_bottom"
class = "AlwaysOnNode"
variant = "A"
position_m = [0.5, 3.0, 0.0]
floor = 0
role = "choke_point"

[[nodes]]
id = "stairs_mid"
class = "LidarNode"
variant = "A"
position_m = [1.0, 3.0, 1.5]
floor = 0.5
role = "fall_detection"
```

---

## 9. Outdoor and Covered Areas

### 9.1 Covered Porch/Entry

**Sensor nodes:**
- Weatherproof enclosure (IP66)
- Identity node for entry identification
- Always-On node for approach detection

**Placement:**
- Under roof overhang
- Camera/biometric sensor facing entry path
- Avoid direct weather exposure

### 9.2 Driveway/Garage

**Sensor nodes:**
- LiDAR node for vehicle detection
- mmWave radar for motion
- Choke-point node at garage entry

**Placement:**
- Garage ceiling (interior)
- Exterior node under overhang

### 9.3 Outdoor Considerations

**Temperature range:** -20°C to +50°C (select modules accordingly)

**Enclosure:** IP66 minimum, IP67 for exposed locations

**Power:** PoE preferred for outdoor nodes (no battery maintenance)

**Mesh connectivity:** Outdoor nodes may need directional antennas or relay nodes

---

## 10. SLAM Integration Considerations

### 10.1 LiDAR Node Placement for SLAM

For dense SLAM capability:
- LiDAR nodes need visual overlap for loop closure
- Place nodes where fields of view intersect
- Ensure sufficient visual features in scene

**Placement guidelines:**
- Minimum 2 LiDAR nodes with overlapping views for SLAM
- 360° LiDAR node (Variant D) provides single-node SLAM for that room
- Multi-room SLAM requires edge controller with Linux SoM

### 10.2 SLAM Map Persistence

**Storage:** Edge controller SD card or network storage

**Retention:** Configurable; SLAM maps can persist indefinitely

**Updates:** Incremental updates when furniture moves (dynamic remap)

**Configuration:**
```toml
[slam]
enabled = true
mode = "dense"                    # "sparse" | "dense"
storage_path = "/data/slam"
map_update_policy = "incremental"  # "incremental" | "replace"
loop_closure_threshold = 0.8
```

---

## 11. Calibration Procedures

### 11.1 Initial Calibration

**Automated walk-through:**
1. Start calibration from companion app or CLI
2. Walk defined path through home
3. System detects you via all nodes simultaneously
4. Trilateration establishes node positions
5. Occlusion map generated automatically

**Confidence targets:**
- Per-node confidence > 0.85 (excellent)
- Per-node confidence 0.70–0.85 (acceptable)
- Per-node confidence < 0.70 (recalibrate)

### 11.2 Calibration for 360° LiDAR Nodes

**Additional requirements:**
1. Place calibration checkerboard at 4+ positions
2. Each position visible to multiple cameras
3. Run camera calibration
4. Verify stitching quality in companion app

**Calibration command:**
```bash
aegis-cli calibrate-360 --node lidar_node_main_room \
  --output calibration.json \
  --positions 4
```

### 11.3 Identity Node Calibration

**Face enrollment:**
1. Position in front of identity node
2. Companion app prompts for face capture
3. Multiple angles captured
4. Embedding generated on edge controller

**Configuration:**
```toml
[identity_node.calibration]
auto_enroll_on_detection = false  # Manual enrollment preferred
min_enrollment_images = 5
embedding_model = "lightweight"    # "lightweight" | "full"
```

---

## 12. Power and Infrastructure

### 12.1 Power Sources

| Node Class | Recommended Power | Backup |
|------------|------------------|--------|
| Always-On Node | USB-C (5V) or PoE | Optional battery |
| LiDAR Node | PoE 802.3af/at | Battery not practical |
| Identity Node | USB-C or PoE | Optional battery |
| Edge Controller | AC power | 4+ hour UPS recommended |

### 12.2 PoE Infrastructure

**For LiDAR nodes:**
- PoE switch at edge controller location
- Cat 6 cabling to LiDAR node positions
- 802.3af for standard nodes, 802.3at for 360° variants

**PoE planning:**
- Count LiDAR nodes requiring PoE
- Size PoE switch accordingly (with 20% headroom)
- Plan cable routes before installation

### 12.3 Edge Controller Power

**Requirements:**
- Dedicated circuit preferred
- UPS for power loss protection
- Battery backup for cellular failover

---

## 13. Interference Analysis

### 13.1 mmWave Radar Interference

**Scenario:** Two Always-On nodes within 2 m

**Mitigation:**
- Automatic time-division multiplexing (firmware)
- Physical separation (preferred)
- Different frequency channels (if supported)

**Detection:**
- Placement solver flags interference
- Companion app shows warning
- Manual configuration available

### 13.2 Acoustic Interference

**Sources:**
- HVAC vents
- Appliances (refrigerator hum, dishwasher)
- Exterior noise (traffic, construction)

**Mitigation:**
- Place acoustic nodes 2+ m from noise sources
- Use directional microphone arrays
- Software noise cancellation

---

## 14. Legal and Compliance Considerations

### 14.1 Recording Laws

**User responsibility:** Recording consent laws vary by jurisdiction

**System features:**
- Configurable recording modes
- Privacy zones (no-recording areas)
- Schedule-based recording

### 14.2 Camera Placement

**Avoid:**
- Bathrooms (illegal in most jurisdictions)
- Bedrooms (legal but privacy concern)
- Areas visible from public spaces
- Neighbor property

**Recommended:**
- Entry points
- Public-facing exterior (with signage)
- Common areas

### 14.3 Signage

**Recommended signs:**
- "Video Surveillance in Progress" at entry points
- "Audio Recording" if audio enabled
- Check local requirements

---

## 15. Configuration Reference

### 15.1 Deployment Configuration

```toml
# aegis-mesh.toml

[deployment]
building_floor_count = 2
floor_height_m = 2.7
total_area_m2 = 180.0
primary_edge_controller = "edge_controller_main"

[placement]
solver_budget_nodes = 15
min_coverage_depth = 2
coverage_priority = "floor_level"  # "floor_level" | "uniform"

# Node class budget
[placement.node_class_budget]
always_on = 10
lidar = 4
identity = 1

# Coverage priorities
[placement.priority_zones]
[[placement.priority_zones.zone]]
name = "front_entry"
weight = 2.0          # Double priority
reason = "Primary entry point"

[[placement.priority_zones.zone]]
name = "stairway"
weight = 1.5
reason = "Fall risk"

[[placement.priority_zones.zone]]
name = "bedroom_1"
weight = 0.5          # Reduced priority
reason = "Privacy zone, reduced coverage acceptable"

[placement.interference]
mmwave_min_separation_m = 2.0
acoustic_min_distance_from_hvac_m = 1.0
```

### 15.2 Edge Controller Configuration

```toml
[edge_controller]
id = "edge_controller_main"
position_m = [5.0, 4.0, 0.0]      # Central location
variant = "linux_som"             # For SLAM support

[edge_controller.connectivity]
wifi_enabled = true
wifi_prefer_5ghz = true
cellular_enabled = false
ethernet_enabled = true

[edge_controller.connectivity.cellular]
enabled = false
sim_type = "nano_sim"
apn = ""
fallback_to_wifi = true
always_on_cellular = false
alert_via_cellular = true

[edge_controller.power]
primary_source = "ac"
ups_backup_hours = 4
battery_capacity_wh = 100

[edge_controller.poe]
ports = 8
power_budget_w = 90              # 802.3at capable
```

---

## 16. Troubleshooting Placement Issues

### 16.1 Coverage Gaps

**Symptom:** Blind spot reported after installation

**Diagnosis:**
1. Check coverage heatmap in companion app
2. Identify missing voxels
3. Review occlusion from furniture

**Resolution:**
- Add low-angle node
- Adjust existing node angle
- Move furniture slightly

### 16.2 Mesh Connectivity Issues

**Symptom:** Node shows offline or intermittent

**Diagnosis:**
1. Check signal strength to nearest hub/edge controller
2. Identify obstacles (metal, walls, distance)

**Resolution:**
- Move node closer to hub
- Add mesh relay node
- Use wired connection (PoE)

### 16.3 Interference Detection

**Symptom:** Erratic detection, high false positives

**Diagnosis:**
1. Check interference warnings in dashboard
2. Review mmWave node placement relative to each other

**Resolution:**
- Increase separation between radar nodes
- Enable time-division multiplexing
- Relocate one node

---

## 17. Summary: The Placement Checklist

### Pre-Installation

- [ ] Complete geometry scan
- [ ] Run coverage solver
- [ ] Review placement report
- [ ] Identify edge controller location
- [ ] Plan cable runs (power, PoE)
- [ ] Check mesh connectivity paths

### Installation

- [ ] Mount nodes at recommended positions
- [ ] Install edge controller
- [ ] Run cabling
- [ ] Connect power
- [ ] Verify mesh connectivity per node

### Post-Installation

- [ ] Run calibration walk
- [ ] Verify calibration confidence
- [ ] Check coverage heatmap
- [ ] Review blind spots
- [ ] Configure recording modes
- [ ] Set up identity enrollment (if applicable)

### Ongoing

- [ ] Monitor dynamic remap notifications
- [ ] Review coverage after furniture changes
- [ ] Check node health monthly
- [ ] Review alert history for patterns

---

**End of Node Placement Guide**
