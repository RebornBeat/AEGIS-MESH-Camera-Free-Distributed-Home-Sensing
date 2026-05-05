# Node Placement Guide — AEGIS-MESH

---

## Core Philosophy: Geometry-First, Not Grid-First

The most common mistake in sensor placement is treating the home as a uniform grid and dividing it evenly. This wastes hardware on redundant coverage of the same areas while leaving furniture-induced blind spots unaddressed.

AEGIS-MESH's placement approach is **occlusion-driven**: we first map the home's actual 3D geometry, identify where the coverage gaps are, and place nodes specifically to address those gaps with the minimum number of nodes.

---

## The Placement Process

### Phase 1: One-Time Geometry Scan

Walk through the home with the AEGIS-MESH app running. The app uses whichever sensor nodes are already installed (or a mobile LiDAR scan if no nodes are installed yet) to build a 3D voxel map of the space.

This scan captures: wall positions, furniture height and placement, doorways and openings, ceiling height variations, stairs.

The scan produces a `VoxelMap` saved to disk. It does not need to be redone unless the home's structure changes significantly.

### Phase 2: Coverage Solver

Run the placement solver with your constraints:

```bash
cargo run --bin aegis-cli -- placement-solve \
  --voxel-map home.voxelmap \
  --budget 12 \
  --min-coverage-depth 2 \
  --output placement_report.html
```

The solver outputs:
- Recommended node positions (as [x, y, z] in your home's coordinate system)
- Node class for each position (AlwaysOn for choke points, LidarNode for open spaces)
- Coverage visualization (HTML report with interactive 3D view)
- Blind-spot report (areas that cannot be covered within the node budget)

### Phase 3: Iterative Refinement

Install nodes at or near the recommended positions. Run a calibration walk. The system reports actual coverage vs. planned coverage. For any gaps, the solver suggests placement adjustments.

---

## The Layered Placement Strategy

### Layer 1: Ceiling-Corner Nodes (Primary — start here)

**Position:** Near room corners, mounted to the ceiling. Offset 30–60 cm from the corner and angled diagonally across the room.

**Why diagonal:** A camera pointed straight down only sees below it. A node angled diagonally across the room sees the near half of the room at high resolution and the far half at lower resolution. Two diagonally opposing ceiling nodes cover the entire room with overlapping views.

**Sensor complement:** LiDAR + mmWave Radar + Acoustic. These are your volume nodes.

**Coverage contribution:** Approximately 70–80% of room volume. Resilient to furniture rearrangement because furniture sits on the floor; ceiling views are largely unaffected.

**Typical count per room:** 2 for a room up to 5×5 m. 4 for larger open-plan spaces.

### Layer 2: Low-Angle Nodes (Occlusion Breakers)

**Position:** 20–50 cm above floor level, mounted on walls or furniture edges, oriented horizontally.

**Why this height:** Furniture (couches, beds, tables) creates occlusion at floor-to-furniture-top height. A sensor at this level can see into the space under furniture that a ceiling node cannot. This is how the system detects pets under couches, fallen objects, and low-profile motion.

**Sensor complement:** mmWave Radar (primary — sees through furniture fabric). Optional short-range ToF for geometric refinement.

**Placement criterion:** Only place low-angle nodes where the occlusion analysis identifies floor-level coverage gaps. Not in every room.

**Typical count:** 1–2 per large room with heavy furniture.

### Layer 3: Choke-Point Nodes (High-Value, Low-Cost)

**Position:** Doorways, hallways, stair entries — any location that all traffic must pass through.

**Why choke points:** Everything in the home passes through doorways. A cheap, low-complexity node at a doorway provides more tracking value per dollar than an expensive volume node in an already-covered open space. This is the insight from cellular network planning applied to home sensing.

**Sensor complement:** PIR + 1D ToF (minimum). Optional: mmWave presence module.

**Coverage contribution:** Not volumetric — choke-point nodes don't cover area. They provide high-confidence "crossing events" that the tracking system uses to hand off tracks between rooms.

**Typical count:** 1 per doorway or hallway entry.

---

## Placement Rules of Thumb

**Ceiling height:** Nodes above 3 m ceiling height create large shadow cones (the blind spot directly beneath). If your ceiling is higher than 3 m, compensate with low-angle nodes to cover the shadow region.

**Interference avoidance:** mmWave radar nodes within 2 m of each other may interfere. The mesh protocol uses time-division multiplexing to mitigate this, but physical separation is preferable. The placement solver accounts for this.

**Acoustic node placement:** Acoustic nodes work best away from HVAC vents and appliances that generate continuous background noise. The solver does not know about your HVAC layout — annotate these on your floor plan before running the solver.

**Identity node placement:** If using identity nodes, place them at entry points: front door, back door, garage entry. The node activates when the sensing layer detects a relevant track approaching the entry. The node's camera field-of-view should cover the entry approach (1–3 m in front of the door).

---

## Dynamic Remap: What Happens When Furniture Moves

After the initial placement and calibration, AEGIS-MESH continuously compares current sensor readings against the baseline voxel map. When a large couch moves across the room:

1. The system detects that a previously observable region is now occluded.
2. A `BlindSpotEvent` is generated and logged.
3. The dashboard shows the new blind spot in yellow.
4. If the blind spot persists for more than the configured duration, the system suggests a node placement adjustment.

You don't need to re-run the placement solver after moving furniture — the system tells you what changed and what to do about it.
