# Export Control Posture — AEGIS-MESH

**Version:** 1.0

---

## 1. Summary

AEGIS-MESH is a general-purpose residential sensing research platform. The maintainers do not believe any component of AEGIS-MESH as designed and published constitutes a controlled item under the Export Administration Regulations (EAR), ITAR (International Traffic in Arms Regulations), or the EU Dual-Use Regulation.

**This assessment is the maintainers' good-faith opinion, not legal advice.** Deployers should conduct their own export control review, particularly if:
- Deploying outside their country of residence.
- Shipping hardware components across national borders.
- The deployment context has any military, intelligence, or defense application.

---

## 2. Component-Level Analysis

### 2.1 mmWave Radar (60 GHz)

60 GHz radar modules used in this project are commercially available consumer and industrial components. They operate in unlicensed frequency bands. Similar components are used in automotive radar, gesture recognition, and building automation globally.

**EAR classification (US):** Likely EAR99 (no license required for most destinations). Verify with the specific module's manufacturer for their ECCN classification.

**EU Dual-Use:** Likely not controlled. Radar systems are controlled only under specific performance parameters for military applications. Consumer-grade 60 GHz FMCW radar does not meet those thresholds.

### 2.2 Solid-State LiDAR

Commercial solid-state LiDAR modules (Livox, Ouster, Robosense) are widely exported globally for automotive and robotics applications. No export license is expected to be required for the modules specified in this project.

### 2.3 Software (OMNI-SENSE, PentaTrack, AEGIS-MESH)

**EAR Technology:** Publicly available open-source software is generally exempt from EAR license requirements under the publicly available exclusion (§ 734.3(b)(3)).

**Encryption:** If the companion app or edge controller API uses TLS encryption (recommended for remote access), the encryption software is subject to EAR § 740.13(e) License Exception ENC for publicly available source code. No license is required for publicly available open-source encryption.

---

## 3. Recommended Precautions

- Do not deploy AEGIS-MESH in contexts involving military, intelligence, or weapons-adjacent applications.
- Do not ship evaluation hardware to countries subject to comprehensive US or EU trade embargoes without legal review.
- If uncertain, consult a licensed export control attorney before cross-border shipment.

---

## 4. No Government or Military Application Claim

AEGIS-MESH is designed for civilian residential sensing research. The maintainers make no claim of suitability for government or military applications and do not solicit such use.
