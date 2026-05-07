# Privacy Threat Model — AEGIS-MESH

**Project:** AEGIS-MESH
**Domain:** Privacy threats, architectural mitigations, and honest residual risk

---

## 1. Threat Category Overview

This document identifies the privacy threats AEGIS-MESH is designed against and the architectural mitigations for each. It also honestly identifies residual threats that the architecture cannot fully address.

---

## 2. State-Actor Surveillance

**Threat:** Government agencies with legal authority compel disclosure of data from smart-home systems via subpoena, warrant, or informal request.

**AEGIS-MESH mitigation:**
- Local-only processing by default. The edge controller maintains all data locally. No cloud back-end holds any user data.
- No persistent imagery. The always-on sensing mesh captures no frames. The identity layer (opt-in) captures frames that are processed immediately and not stored.
- Encrypted local storage for any identity classification results. User holds the encryption key.

**Residual risk:** A government with physical access to the home can access the local device. A government can compel the user. Local data, however minimal, exists.

---

## 3. Non-State-Actor Surveillance

**Threat:** Cybercriminals or stalkers compromise the system to observe residents.

**AEGIS-MESH mitigation:**
- No cloud back-end to attack at scale.
- No imagery to exfiltrate from the always-on sensing mesh.
- Cryptographically signed mesh protocol — message replay and forgery are detected.
- Reproducible firmware builds — users can verify their nodes run the published code.
- The worst-case exfiltration from a compromised sensing-mesh node: track-level metadata (anonymous geometric positions of objects). No imagery.

**Residual risk:** A compromised identity node, if the kill switch is in the enabled position, could produce imagery locally. An attacker with physical access could modify firmware. Supply-chain compromise is possible.

---

## 4. Intimate-Partner Abuse Facilitation

**Threat:** An abusive party uses smart-home systems to monitor a partner's activities, movements, and visitors.

**AEGIS-MESH mitigation:**
- Hardware kill switches on all identity nodes. Physical switch that cannot be overridden remotely.
- Append-only audit log readable by any user of the system, showing every identity-layer activation.
- Local-only by default — no remote viewing without explicit user configuration.
- Camera-free always-on mesh — the most intrusive monitoring vector (live video) is architecturally absent.

**Residual risk:** An abusive party with administrative access (initially shared or coerced) can configure remote viewing if the user has enabled it. The architecture cannot resolve consent disputes between household members. Physical control of the home by an abusive party undermines all software safeguards.

**Additional guidance:** AEGIS-MESH documentation explicitly addresses this threat scenario and recommends that users in abusive relationship contexts consult domestic-violence organizations' technology-safety resources, not rely on this system for protection.

---

## 5. Aggregate Behavioral Profiling

**Threat:** Over time, sensor data reveals detailed patterns about residents' daily lives, which could be commercially valuable or used for targeting.

**AEGIS-MESH mitigation:**
- No cloud back-end. Behavioral profiles exist only on the local device.
- No vendor holds data — the open-source architecture has no vendor to provide data to third parties.
- Track-level data does not include personally identifying information by default (Privacy-First mode).

**Residual risk:** Local behavioral profiles exist. Anyone with access to the local device can analyze them. The profiles are less revealing than camera-based profiles (no faces, no appearances) but still informative.

---

## 6. Bystander Privacy

**Threat:** Visitors to the home are monitored without consent.

**AEGIS-MESH mitigation:**
- Camera-free always-on sensing mesh does not capture identifying imagery of visitors.
- Visitors are tracked as anonymous geometric objects.
- Audio is processed on-device; no audio is transmitted or stored.

**Residual risk:** Visitor presence is detectable (a moving object of human size arrives and departs). In Security-First mode, if the identity layer activates on visitor arrival, a visitor may be photographed. This is the primary residual bystander-privacy risk. Recommendation: inform visitors when Security-First mode is active.

---

## 7. Architectural Commitments

The mitigations above depend on AEGIS-MESH maintaining these commitments:

1. The always-on sensing mesh (AlwaysOn nodes, LidarNode nodes) does not depend on `omni-sense-drivers-camera`.
2. Identity nodes have hardware kill switches; firmware reads kill-switch GPIO at boot.
3. Raw sensor data does not leave nodes — only classification results and track metadata.
4. The audit log is append-only and locally readable.
5. Firmware is reproducibly buildable from the published source.
6. Pull requests that weaken any of these commitments will not be merged.
