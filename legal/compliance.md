# Legal Compliance Reference — AEGIS-MESH

**Version:** 1.0 | **Status:** Research Platform Reference

---

## 1. Scope

This document covers compliance considerations for the AEGIS-MESH distributed residential sensing platform as a research system. AEGIS-MESH is not a certified commercial product and does not carry any regulatory certification from Anthropic or the maintainers. Deployers are responsible for all applicable regulatory compliance in their jurisdiction.

---

## 2. Radio Frequency (RF) Compliance

### 2.1 Mesh Radio

The mesh radio module (Wi-Fi, Thread, or BLE depending on variant) must be sourced from a supplier with applicable regional certification:

- **FCC (United States):** Part 15 certification required for all intentional radiators. The radio module must carry an FCC ID. Do not use an uncertified radio module in any installation.
- **CE (European Union):** RED (Radio Equipment Directive 2014/53/EU) compliance required. Radio module must carry CE marking with RED declaration.
- **ISED (Canada):** IC certification required.
- **Other jurisdictions:** Source appropriately certified modules per local telecommunications authority requirements.

The edge controller firmware, sensor firmware, and software stack do not themselves affect radio certification — the radio module's inherent certification covers its operation in the intended frequency bands.

### 2.2 mmWave Radar (60 GHz Band)

The 60 GHz band (57–66 GHz in most jurisdictions) is license-free in most markets under specific power and emission limits. The radar modules specified in this project operate within these limits, but deployers must verify compliance for their specific jurisdiction.

- **FCC:** 47 CFR Part 15, Subpart B and applicable unlicensed emission limits.
- **EU:** ETSI EN 305 550 (Short Range Devices using UWB technology) and harmonized standards under the RED.
- **Japan:** ARIB STD-T96 for 60 GHz applications.

Note: indoor-only restriction applies in some jurisdictions for 60 GHz devices above certain power levels.

### 2.3 LiDAR (905 nm / 1550 nm Optical)

LiDAR modules using Class 1 (eye-safe) optical power levels require no special licensing for indoor use. Verify that the LiDAR module selected for your installation is certified Class 1 per IEC 60825-1. Do not modify LiDAR power settings beyond manufacturer specifications.

---

## 3. Recording and Surveillance Laws

**Critical:** Recording laws vary significantly by jurisdiction. AEGIS-MESH can be configured to capture audio, video, and biometric data. Deployers are fully responsible for compliance.

### 3.1 General Considerations

- **Consent:** Many jurisdictions require one-party or two-party consent for audio recording. "Two-party consent" states/countries require consent from all parties being recorded.
- **Video surveillance:** Notification requirements (signage, etc.) may apply for locations accessible to the public or shared with tenants.
- **Data retention:** GDPR (EU), CCPA (California), and similar data protection regulations may impose retention limits, deletion rights, and breach notification requirements.
- **Cross-border:** If recordings are transmitted or stored outside the jurisdiction of capture, additional regulations may apply.

**The AEGIS-MESH system does not enforce jurisdiction-specific recording restrictions.** This is a research platform. All compliance is the deployer's responsibility.

### 3.2 Biometric Data Handling

When the Identity Node is configured to capture biometric data (facial recognition, fingerprint, voice patterns):

- **BIPA (Illinois Biometric Information Privacy Act):** Requires written consent, data retention policy, and prohibits sale of biometric data. One of the strictest biometric frameworks in the United States.
- **GDPR (EU):** Biometric data is "special category" data requiring explicit consent and a lawful basis for processing.
- **CCPA (California):** Biometric data included in definition of personal information; consumer rights apply.
- **Other state/national laws:** Many jurisdictions have enacted or are enacting biometric privacy legislation.

AEGIS-MESH produces cryptographically-chained integrity manifests for all recordings, which are appropriate for legal proceedings. The system does not transmit biometric data externally unless the user configures streaming or remote access.

### 3.3 On-Premises vs. Remote Storage

Data stored only on the edge controller or local network is generally subject to local jurisdiction law. Data transmitted over cellular to remote servers may invoke additional regulations. Users should consult qualified legal counsel before enabling remote access or cellular transmission of biometric data.

---

## 4. Product Safety

AEGIS-MESH is a research platform. It is not certified as a life-safety product:

- **Not UL-listed or CE-marked for safety functions.** Do not use as a primary intrusion detection or life-safety system.
- **Not a substitute for smoke/CO detectors** or monitored alarm systems with professional response capability.
- **Not a medical device.** Do not use for health monitoring or medical applications.

---

## 5. Battery Safety

All nodes using lithium battery cells must use cells certified to UL 1642 or IEC 62133. The firmware enforces software voltage cutoffs, but hardware protection circuits (overcharge, overdischarge, short circuit) are the primary safety mechanism.

---

## 6. PoE Safety

When using Power over Ethernet (PoE 802.3af/at), ensure:
- PoE injector or switch is UL/CE-listed.
- Ethernet cabling meets appropriate ratings for the installation environment.
- PoE-powered devices are not submerged unless the specific enclosure variant is rated for immersion.

---

## 7. Export Control

AEGIS-MESH is a general-purpose sensing research platform with no design intent for military or dual-use applications. The mmWave radar uses commercially available modules operating in unlicensed frequency bands. LiDAR uses commercially available solid-state modules. No components of this project are believed to trigger export control restrictions under EAR (US), ITAR (US), or EU Dual-Use Regulation.

**Deployers are responsible for verifying export control status for their specific configuration before cross-border shipment or deployment.** Consult with an export control attorney if uncertain.

---

## 8. OMNI-SENSE and PentaTrack Software Compliance

The software stack depends on OMNI-SENSE and PentaTrack, both MIT-licensed open-source libraries. All crates used are MIT or Apache 2.0 licensed. No GPL-licensed code is used in the production binaries; verify this with `cargo license --json` before commercial deployment.

---

*This document is informational only and does not constitute legal advice. Consult qualified legal counsel for jurisdiction-specific compliance.*
