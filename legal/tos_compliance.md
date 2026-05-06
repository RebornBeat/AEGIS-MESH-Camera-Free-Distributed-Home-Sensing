# Terms of Service Compliance — AEGIS-MESH

**Version:** 1.0

---

## 1. Open-Source License Compliance

AEGIS-MESH uses the following license structure:
- **Code (firmware and software):** MIT License
- **Hardware designs (schematics, PCB layouts):** CERN-OHL-S v2
- **Documentation:** CC BY 4.0

### CERN-OHL-S v2 Hardware Obligations

If you manufacture hardware based on AEGIS-MESH PCB designs and distribute that hardware:
- You must make the complete source files (KiCad project, Gerbers, BOM) available to recipients.
- You must indicate that the design is derived from AEGIS-MESH and provide attribution.
- Modifications must also be released under CERN-OHL-S v2.

This obligation does not apply to hardware manufactured solely for your own use (internal research, personal deployment).

---

## 2. OMNI-SENSE Dependency Compliance

AEGIS-MESH depends on OMNI-SENSE (`omni-sense-*` crates), which are MIT-licensed. MIT license requires:
- Retain the MIT license notice in any distribution of the compiled binaries.
- No warranty is provided by the OMNI-SENSE contributors.

AEGIS-MESH also depends on PentaTrack, also MIT-licensed with the same requirements.

All Rust crate dependencies are MIT or Apache 2.0. Run `cargo license --json` to produce a machine-readable license inventory for compliance review.

---

## 3. Third-Party Component Compliance

Hardware components (radar modules, LiDAR modules, camera modules) are sourced from third-party manufacturers. Each component has its own terms:
- Evaluation module terms often restrict commercial use — verify before commercial deployment.
- Production modules (e.g., Livox MID-360) are typically licensed for commercial use subject to manufacturer terms.
- Review manufacturer terms for each component before commercial deployment.

---

## 4. Companion App and API

The AEGIS-MESH companion app communicates exclusively with the user's own edge controller. No data is transmitted to the maintainers or any third party. The API is self-hosted on user hardware.

---

## 5. Research Use

This is a research platform. Commercial use is not prohibited by the MIT license, but:
- Regulatory compliance (FCC, CE, recording laws, data protection) is the deployer's responsibility.
- No warranty is provided.
- The maintainers are not liable for any damages arising from deployment.
