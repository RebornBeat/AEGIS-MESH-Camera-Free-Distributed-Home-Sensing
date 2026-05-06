# Research Ethics — AEGIS-MESH

**Version:** 1.0

---

## 1. Project Ethics Posture

AEGIS-MESH is a research platform for distributed residential sensing. The maintainers are committed to responsible development and transparent documentation of all capabilities, limitations, and potential misuse vectors.

---

## 2. Privacy-by-Architecture

The primary sensing mesh (Always-On Variants A–C, LiDAR Variants A–B) is camera-free at the compile-time dependency level. This is the strongest possible privacy commitment at the architecture level: the firmware binary physically cannot produce imagery because the camera driver crate is not linked.

Camera-capable variants (Always-On D, LiDAR C/D, Identity Node) add visual sensing as an explicit user choice, with all data handling fully user-configured. The system does not make data handling choices on behalf of the user.

---

## 3. Dual-Use Considerations

The sensing capabilities of AEGIS-MESH (presence detection, tracking, acoustic classification, visual identification) could be misused for:
- Surveillance of individuals without their knowledge or consent.
- Tracking individuals in shared spaces without disclosure.
- Covert monitoring in contexts where monitoring is prohibited.

**Mitigation posture:**
- Documentation explicitly covers recording law compliance responsibilities.
- The system is designed for owner-deployed residential sensing, not covert public surveillance.
- No cloud relay, no external telemetry, no third-party data access.
- The maintainers condemn use of this platform for non-consensual surveillance.

---

## 4. Research Data

If research data (detection events, recordings, SLAM maps) is collected using AEGIS-MESH and published or shared, the research team is responsible for:
- IRB/ethics board approval if human subjects research applies.
- Informed consent from all persons whose data is captured.
- Anonymization before publication where identity could be inferred.
- Compliance with data protection regulations applicable to the research context.

---

## 5. Disclosure Policy

If security vulnerabilities are discovered in AEGIS-MESH (particularly those that could enable unauthorized access to recordings or live feeds), the maintainers commit to:
- Acknowledging the report within 7 days.
- Issuing a fix or workaround within 30 days for critical vulnerabilities.
- Publishing a security advisory upon fix release.

Report security vulnerabilities via: [project security contact — to be defined by deployer organization].
