# Adaptive Remap — AEGIS-MESH

---

## Overview

Adaptive remap is the continuous process by which AEGIS-MESH keeps its understanding of the home synchronized with the home's actual layout. It answers the question: "What has changed since we last calibrated?"

---

## How It Works

### Baseline Establishment

After the initial calibration walk, the system establishes a **baseline voxel map**: a 3D grid of the home's interior where each voxel is labeled as Open, Structure (wall/floor/ceiling), or Obstruction (furniture, fixed objects).

The baseline is stored on disk and loaded at each boot. It is the reference state against which all future observations are compared.

### Continuous Comparison

During normal operation, LiDAR nodes scan their fields of view at a low background rate (configurable; default: 1 scan per 30 seconds per node). Each scan produces current occupancy data.

The `DynamicRemapper` component compares:
- **Current scan:** What does the sensor see right now?
- **Baseline:** What should the sensor see, given the known room geometry?

Discrepancies fall into two categories:
1. **Transient:** A person or pet is in the scan path. This is expected and filtered by requiring changes to persist for multiple scans.
2. **Persistent:** Something has changed that doesn't move. This is new furniture, moved furniture, or a structural change.

### Persistence Filtering

A change must appear in the same voxels across N consecutive scans before it is classified as persistent. Default N = 5 (meaning the change must persist for at least 5 × 30s = 2.5 minutes before triggering a remap event). This is configurable.

### Remap Events

When a persistent change is detected:

1. **VoxelMap update:** The affected voxels are relabeled (Open → Obstruction or vice versa).

2. **Coverage recomputation:** The coverage solver re-runs against the updated voxel map to identify any new blind spots or coverage changes.

3. **User notification:** If new blind spots appear, the dashboard highlights them and the API emits a `BlindSpotEvent`. If coverage improves (furniture removed, opening cleared), the dashboard updates accordingly.

4. **Placement suggestions:** If a new blind spot cannot be covered by the existing nodes, the placement solver suggests where an additional node would eliminate it.

---

## Configuration

```toml
[dynamic_remap]
scan_interval_seconds = 30          # How often each node rescans
persistence_scan_count = 5          # Consecutive scans required to confirm change
voxel_change_threshold = 0.15       # Fraction of voxels in a region that must change
notification_delay_minutes = 2.5    # Don't notify until change persists this long
auto_update_voxel_map = true        # Automatically update map; false = require user confirmation
```

---

## Manual Remap Trigger

```bash
# Trigger immediate remap (useful after intentional furniture rearrangement)
curl -X POST http://localhost:7171/api/calibration/remap
```

After a manual remap trigger, the system performs a full rescan immediately, updates the voxel map, and recomputes coverage in a single cycle.

---

## When Remap is Not Sufficient

Remap handles layout changes within the existing node deployment. It cannot add new coverage if all existing nodes have blind spots in a region. If a remap cycle consistently reports blind spots that cannot be resolved by the existing nodes, a full placement rescan (Phase 1 of the placement process) is recommended.
