# Occlusion-Driven Placement — Geometry-First Sensor Deployment

**Project:** AEGIS-MESH
**Domain:** Sensor placement optimization for residential coverage
**Implementation:** `aegis-placement` crate

---

## 1. The Problem with Grid Placement

A uniform grid of sensors sounds comprehensive but is geometrically naive. Consider a living room with a large sectional sofa:

- A ceiling node directly above the sofa has its LiDAR view blocked for anyone sitting behind the sofa back.
- A second ceiling node on the opposite side of the room sees the space behind the sofa but has its view of other areas blocked by the sofa's height.
- A grid placed without accounting for the sofa's location misses both issues and the blind spots remain undetected.

Occlusion-driven placement starts from the actual 3D geometry of the space and computes placements that address specific occlusions with minimum node count.

---

## 2. The Algorithm

### Step 1: Voxel Map Construction (aegis-placement::voxel_map)

The home's interior is represented as a 3D grid of voxels at configurable resolution (default: 25 cm × 25 cm × 25 cm). Each voxel is labeled:
- **Open:** Traversable space. Relevant for both objects-of-interest and sensor beams.
- **Structure:** Walls, floors, ceilings. Opaque to sensor beams.
- **Obstruction:** Furniture, fixed appliances. Blocks sensor beams.

The voxel map is populated from the geometry scan described in `node_placement_guide.md`.

### Step 2: Visibility Cone Computation (aegis-placement::occlusion_analyzer)

For each candidate sensor position, we compute the **visibility cone**: the set of voxels that the sensor can observe from that position.

This is a ray-march computation: from the candidate position, cast rays in all directions within the sensor's field of regard. For each ray, mark voxels as observable until the ray hits a Structure or Obstruction voxel. Stop the ray there.

The result is an `OcclusionMap`: a per-candidate-position set of observable and unobservable voxels.

Computational optimization: spatial hashing of obstruction voxels for fast ray-traversal. Early exit on rays that hit structure within a few voxels. Typical compute time: < 1 second per candidate position on a Raspberry Pi-class device.

### Step 3: Coverage Optimization (aegis-placement::coverage_solver)

**Inputs:**
- Voxel map (the home geometry)
- Node budget (maximum number of nodes to deploy)
- Coverage depth requirement (minimum N nodes per important voxel; default N=2 for redundancy)
- Node class budget (how many AlwaysOn vs. LidarNode vs. IdentityNode)
- Candidate positions (a grid of possible mounting locations, refined by user input about mounting feasibility)

**Algorithm:** Greedy weighted set cover.

1. Assign each voxel a priority weight based on: likelihood of containing objects of interest (floor level higher than near-ceiling), proximity to entries (front of house higher than center of back room), room type.

2. Iteratively select the candidate position that provides the greatest marginal coverage (weighted by voxel priority, penalized for already-covered voxels).

3. Apply constraints: node class must match zone requirements (choke-point positions get AlwaysOn nodes, open-space positions get LidarNode), interference avoidance (no two radar nodes within 2 m without time-division multiplexing configured).

4. Stop when budget is exhausted or all high-priority voxels are covered.

**Output:** Ordered list of node placements, each with: position, recommended class, estimated coverage contribution, and marginal voxels added by this node.

### Step 4: Placement Report

A human-readable HTML report with:
- 3D interactive view of the home with proposed node positions
- Coverage heatmap (color-coded by depth: how many nodes observe each voxel)
- Blind-spot list (voxels not covered within budget)
- Interference warning (nodes close enough to require time-division multiplexing)
- Cost estimate (based on node class counts)

---

## 3. The Layered Strategy Output

The coverage solver produces placements that naturally fall into the layered categories described in `node_placement_guide.md`. The algorithm does not know about "ceiling corner nodes" as a concept — it discovers them because they are geometrically optimal for the objective function.

Why ceiling-corner positions emerge: they provide wide, unobstructed fields of view for LiDAR and radar (no furniture in the path from the ceiling). The diagonal angle from a corner covers the entire room floor area without any object at floor level being in the direct path between the sensor and the opposite corner.

Why low-angle positions emerge: the solver identifies that under-furniture spaces are not covered by ceiling nodes (ray-march to those voxels from the ceiling hits the furniture top). A candidate position at floor level, angled upward, can see under furniture. These positions appear in the solution when the under-furniture voxels are weighted as important (e.g., a family with pets or young children).

Why choke-point positions appear: the solver marks choke-point voxels (doorway crossing regions) as extremely high priority. Any candidate position observing a doorway crossing region contributes enormously to the weighted score. AlwaysOn nodes (PIR + 1D ToF) appear at doorways because they cover these voxels cheaply.

---

## 4. Dynamic Remap and Coverage Maintenance

The initial placement is computed against the baseline geometry. After furniture changes, the dynamic remap system updates the voxel map and re-evaluates coverage using the existing node positions (not the full solver — a faster "feasibility check" that determines whether existing nodes can cover the updated geometry).

If the feasibility check fails (existing nodes cannot cover the updated geometry within the required depth), the solver runs a partial re-optimization to suggest placement changes.
