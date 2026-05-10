# Privacy Threat Model — AEGIS-MESH

**Project:** AEGIS-MESH
**Domain:** Privacy threats, architectural mitigations, and honest residual risk
**Version:** 1.0
**Status:** Final

---

## 1. Purpose

This document identifies the privacy threats AEGIS-MESH is designed against and the architectural mitigations for each. It honestly identifies residual threats that the architecture cannot fully address, and provides the user with the knowledge needed to make informed deployment decisions.

AEGIS-MESH takes a **hardware-first, user-configuration-second** approach to privacy. The architecture provides strong baseline privacy guarantees through structural design choices (camera-free sensing mesh, hardware kill switches). Users may then configure additional capabilities (cameras, streaming, remote access) according to their needs and jurisdictional requirements.

---

## 2. Threat Category Overview

| Threat Category | Architecture Mitigation Level | Residual Risk Level |
|----------------|------------------------------|---------------------|
| State-actor surveillance | Strong (local-first, optional encryption) | Medium (physical access, legal compulsion) |
| Non-state-actor surveillance | Strong (no cloud, signed protocol) | Low-Medium (physical compromise, supply chain) |
| Intimate-partner abuse facilitation | Medium-High (kill switches, audit logs) | Medium (shared access, physical control) |
| Aggregate behavioral profiling | Strong (local-only, no vendor) | Medium (local data exists) |
| Bystander privacy | High (camera-free sensing mesh) | Low-Medium (presence detectable, identity layer opt-in) |
| Remote surveillance | High (edge controller = sole gateway) | Low-Medium (if remote access explicitly enabled) |
| Data breach | High (local storage, encryption optional) | Medium (if encryption not used, physical theft) |

---

## 3. Architectural Foundation

### 3.1 The Edge Controller as Sole External Gateway

**Critical architectural principle:**

Only the Edge Controller connects to external networks (WiFi, Cellular). All sensor nodes communicate exclusively via mesh radio to the edge controller.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    AEGIS-MESH Privacy Architecture                          │
│                                                                              │
│  Always-On Nodes ──┐                                                         │
│  LiDAR Nodes───────┼── Mesh Radio ──► Edge Controller ──► WiFi ──►App       │
│  Identity Nodes────┘  (BLE/Thread/PoE)       │                              │
│                                              ├──► Cellular ──►App          │
│                                              └──► BLE direct ──►App        │
│                                                                              │
│  ⚠️  EDGE CONTROLLER IS THE ONLY NODE WITH WiFi / Cellular / BLE (external) │
│  ⚠️  ALL OTHER NODES = MESH RADIO ONLY (to edge controller, nothing else)   │
└─────────────────────────────────────────────────────────────────────────────┘
```

**Privacy implications:**

1. **No distributed attack surface:** An attacker cannot directly target individual sensor nodes from external networks. All external traffic must pass through the edge controller.

2. **Centralized access control:** All authentication, authorization, and audit logging occurs at one point.

3. **Configurable egress:** External connectivity is user-configured. Default blocks external traffic.

### 3.2 Privacy-by-Architecture vs Privacy-by-Configuration

AEGIS-MESH provides two layers of privacy control:

**Layer 1 — Privacy-by-Architecture (Structural Guarantees):**

These privacy properties are built into the system design and cannot be circumvented by configuration:

| Property | Implementation |
|----------|----------------|
| Sensing mesh is camera-free | AlwaysOn and LiDAR nodes (variants A-B) do not link `omni-sense-drivers-camera` |
| Identity nodes have hardware kill switches | Physical switch cuts power rail; no firmware override |
| Edge controller is sole external gateway | All other nodes have no WiFi/cellular hardware |
| Local-first by default | No cloud service; all data stored locally |
| Mesh protocol is signed | HMAC-SHA256 on all messages |

**Layer 2 — Privacy-by-Configuration (User Choices):**

These privacy properties depend on user configuration choices:

| Property | Configuration Option |
|----------|----------------------|
| Remote access enabled/disabled | `companion_app.enable_remote_access` |
| Video recording enabled/disabled | `data.store_raw_video` |
| Streaming to app enabled/disabled | `companion_app.enabled` |
| Cellular enabled/disabled | `connectivity.cellular.enabled` |
| Camera on identity nodes enabled/disabled | Hardware kill switch position |

---

## 4. Threat: State-Actor Surveillance

### 4.1 Threat Description

Government agencies with legal authority compel disclosure of data from smart-home systems via:
- Subpoena or warrant
- National security letters (US)
- Informal requests to service providers
- Direct physical seizure of equipment
- Mandatory data retention requirements

### 4.2 AEGIS-MESH Mitigations

**Local-First Architecture:**

- The edge controller maintains all data locally. No cloud back-end holds any user data.
- There is no vendor-controlled infrastructure to subpoena.
- Law enforcement must physically access the edge controller to obtain data.

**No Persistent Imagery by Default:**

- The always-on sensing mesh (AlwaysOn nodes variants A-C, LiDAR nodes variants A-B) captures no frames at all — the camera driver is not linked.
- Identity nodes (if deployed) capture frames only when the hardware kill switch is in the ENABLED position and the user has configured recording.

**Encrypted Local Storage (Optional):**

Users may configure encryption of local storage:

```toml
[data.encryption]
enabled = true
method = "aes-256-gcm"
key_source = "user_passphrase"  # User-controlled; system does not hold key
key_derivation = "argon2id"
```

**If encryption is enabled:**
- Data on SD card and internal flash is encrypted
- User holds the passphrase; edge controller derives key at runtime
- Without passphrase, physical seizure yields only encrypted data

**Cellular Connectivity (Optional):**

- If cellular is enabled, the edge controller has a SIM module and can be reached remotely
- All cellular connections require authentication (bearer token)
- Cellular module communicates only with user-configured endpoints (companion app, emergency contact)
- No telemetry or vendor connections over cellular

**Mitigation Summary:**

| Attack Vector | Mitigation |
|---------------|------------|
| Cloud provider subpoena | N/A — no cloud provider |
| Vendor data request | N/A — open-source, no vendor back-end |
| Physical seizure of edge controller | Encrypted storage (if configured); user holds key |
| Cellular intercept | TLS optional; bearer token required; no vendor endpoints |
| Mesh node compromise | No external connectivity; sensing data only |

### 4.3 Residual Risk

**A government with physical access to the home can:**
- Seize the edge controller
- Obtain unencrypted data (if encryption not configured)
- Obtain encrypted data and compel the passphrase from the user
- Install surveillance equipment independently of AEGIS-MESH

**A government with legal authority can:**
- Compel the user to provide passphrase, tokens, or access
- Order the user to enable remote access
- Require installation of monitoring software

**A government with covert access capability can:**
- Physically modify the edge controller hardware
- Install firmware that exfiltrates data
- Replace nodes with compromised units

**Honest assessment:** Local-only architecture raises the cost and visibility of surveillance but cannot prevent determined state-actor access. Encryption protects against casual or opportunistic seizure but not against compelled disclosure.

---

## 5. Threat: Non-State-Actor Surveillance

### 5.1 Threat Description

Cybercriminals, stalkers, or other malicious actors attempt to:
- Compromise the system remotely to observe residents
- Exfiltrate data for blackmail, stalking, or identity theft
- Use the system as a surveillance platform
- Hold data for ransom

### 5.2 AEGIS-MESH Mitigations

**No Cloud Attack Surface:**

- No cloud back-end to attack at scale
- No cloud credentials to steal
- No cloud API to exploit
- Each deployment is an independent, air-gapped (by default) instance

**Mesh Network Security:**

- All mesh protocol messages are HMAC-SHA256 signed
- Replay protection via sequence numbers and timestamps
- Message forgery detected
- Compromised node cannot inject false detections without detection (audit log verification)

**No Imagery to Exfiltrate from Sensing Mesh:**

- AlwaysOn and LiDAR nodes do not have camera drivers linked
- A compromised sensing node can only exfiltrate: track-level metadata (geometric positions, classification labels)
- No faces, no identifying imagery, no audio recordings (by default)

**Identity Node Protection:**

- Hardware kill switch: physically cuts power to identification sensor
- If kill switch is in DISABLED position, no imagery can be produced regardless of software compromise
- User must physically enable the camera for any imagery to exist

**Reproducible Builds:**

- Firmware is reproducibly buildable from published source
- Users can verify their nodes run published code by comparing build hashes
- Detects firmware modification attacks

**Audit Logging:**

- All system access is logged locally
- Audit log is append-only
- Logs are locally inspectable
- Detects unauthorized access patterns

**Mitigation Summary:**

| Attack Vector | Mitigation |
|---------------|------------|
| Cloud provider breach | N/A — no cloud provider |
| Remote exploitation of nodes | No external connectivity on sensing nodes |
| Remote exploitation of edge controller | Bearer token required; TLS optional; no default external access |
| Mesh protocol forgery | HMAC signatures; replay protection |
| Firmware modification | Reproducible builds; hash verification |
| Data exfiltration from sensing nodes | No imagery; only track metadata |
| Data exfiltration from identity nodes | Hardware kill switch; optional encryption |
| Ransomware | Local-only; no vendor backdoor; encrypted backups possible |

### 5.3 Residual Risk

**A compromised identity node (if kill switch ENABLED):**
- Could produce and exfiltrate imagery if remote access is configured
- Limited by local storage and transmission bandwidth
- Audit log would show activation; user could detect compromise

**An attacker with physical access:**
- Could modify firmware
- Could install hardware implants
- Could replace nodes with compromised units

**Supply chain compromise:**
- Malicious component inserted during manufacturing
- Backdoor in firmware at build time
- Mitigation: reproducible builds detect build-time modification; physical inspection detects hardware implants

**Companion app compromise:**
- If companion app is compromised, it could exfiltrate data received from edge controller
- Mitigation: use official app builds; verify signatures; review source code

**Honest assessment:** AEGIS-MESH raises the cost and complexity of surveillance significantly compared to cloud-based smart-home systems. Determined attackers with physical access or supply chain access remain a residual risk.

---

## 6. Threat: Intimate-Partner Abuse Facilitation

### 6.1 Threat Description

An abusive party uses smart-home systems to:
- Monitor a partner's activities, movements, and visitors
- Track when partner is home alone or away
- Identify visitors to the home
- Maintain surveillance after separation

### 6.2 AEGIS-MESH Mitigations

**Hardware Kill Switch on Identity Nodes:**

- Physical switch cuts power to identification sensor
- Cannot be overridden remotely by any software configuration
- Provides unambiguous visible indicator: camera is physically disabled when switch is OFF
- Abusive party with administrative access cannot remotely enable cameras if switch is DISABLED

**Append-Only Audit Log:**

- Every identity-layer activation is logged with timestamp
- Any user of the system can read the audit log
- Provides evidence of surveillance activity
- Abusive party cannot delete log entries (append-only architecture)

**Local-Only by Default:**

- No remote viewing capability by default
- Remote access requires explicit configuration and bearer token
- User can verify whether remote access is enabled

**Camera-Free Always-On Mesh:**

- The most intrusive monitoring vector (continuous live video) is architecturally absent from the sensing mesh
- Presence and movement are tracked, but without identifying imagery
- No way to determine who a visitor is without enabling the identity layer

**Multiple User Accounts:**

- AEGIS-MESH supports multiple user accounts with different permission levels
- Each account has its own credentials
- Audit log shows which account performed which actions
- Enables accountability in multi-user households

### 6.3 Residual Risk

**Shared administrative access:**

If an abusive party has administrative access (initially shared, coerced, or obtained through deception):
- Can configure remote viewing (if the user enables remote access capability)
- Can view live data and recordings
- Can change settings to increase monitoring

**Limitation:** The architecture cannot resolve consent disputes between household members. A system with legitimate shared access between partners cannot prevent one partner from monitoring the other.

**Physical control:**

If an abusive party has physical control of the home:
- Can physically enable identity node kill switches
- Can relocate or tamper with hardware
- Can install additional surveillance equipment independent of AEGIS-MESH

**Coerced access:**

An abusive party can coerce the user into:
- Providing credentials
- Enabling remote access
- Disabling security features

**Honest assessment:** The hardware kill switch provides a meaningful protection — an abusive party with only software access cannot enable cameras remotely. However, shared legitimate access and physical control cannot be solved by technology. The architecture raises the bar but cannot eliminate this threat.

### 6.4 Guidance for Users in Abusive Situations

**AEGIS-MESH documentation includes explicit guidance:**

1. **Hardware kill switch position:** Verify the physical position of all identity node kill switches. If switches are in DISABLED position, cameras cannot be enabled remotely.

2. **Audit log review:** Periodically review the audit log for unexpected identity-layer activations. Unexplained activations may indicate unauthorized surveillance.

3. **Remote access verification:** Check whether remote access is enabled. If enabled, verify who has access tokens.

4. **Physical security of edge controller:** The edge controller contains all data. Physical control of this device equals control of the system.

5. **Independent consultation:** Users in abusive relationship contexts should consult domestic-violence organizations' technology-safety resources. AEGIS-MESH is a tool, not a substitute for expert guidance.

**Resources:**
- National Network to End Domestic Violence (NNEDV) — Tech Safety
- National Domestic Violence Hotline
- Local domestic violence advocacy organizations

---

## 7. Threat: Aggregate Behavioral Profiling

### 7.1 Threat Description

Over time, sensor data reveals detailed patterns about residents' daily lives:
- Wake and sleep times
- When home is vacant
- Visitor patterns
- Room occupancy patterns
- Activity patterns (cooking, TV, etc.)

This data could be:
- Commercially valuable (targeted advertising, insurance risk assessment)
- Used for social engineering attacks
- Used for stalking or harassment
- Subpoenaed in legal proceedings

### 7.2 AEGIS-MESH Mitigations

**No Cloud Back-End:**

- Behavioral profiles exist only on the local edge controller
- No vendor holds data
- No commercial incentive to monetize data
- No third-party data sharing

**Open-Source Architecture:**

- No hidden data collection
- No telemetry (unless explicitly configured by user)
- Source code auditable for data handling

**Privacy-First Mode:**

- Default mode: tracks are anonymous geometric objects
- No identity information attached to tracks
- Presence detectable; identity not

**User-Controlled Retention:**

- Retention period user-configured
- Data can be deleted at any time
- No indefinite retention without user choice

**Metadata Minimalism:**

- Sensor nodes perform local processing
- Only classification results and track metadata transmitted
- Raw sensor data (radar IQ, audio waveforms) not transmitted by default

### 7.3 Residual Risk

**Local profiles exist:**

- Anyone with access to the edge controller can analyze behavioral patterns
- Long-term data reveals detailed life patterns
- This is inherent to any persistent sensing system

**Lower resolution than camera-based:**

- AEGIS-MESH profiles are less revealing than camera-based profiles
- No faces, no visual appearances
- But still: presence patterns, occupancy patterns, room usage

**Legal discovery:**

- Behavioral profiles could be subpoenaed in legal proceedings
- Divorce, custody, criminal cases could request edge controller data
- User responsible for understanding local legal requirements

**Honest assessment:** Local-only architecture prevents large-scale behavioral data harvesting but does not prevent local analysis. The data is inherently revealing over long time periods.

---

## 8. Threat: Bystander Privacy

### 8.1 Threat Description

Visitors to the home are monitored without their knowledge or consent:
- Presence detected and logged
- Movement patterns recorded
- Potentially photographed (if identity layer active)

### 8.2 AEGIS-MESH Mitigations

**Camera-Free Sensing Mesh:**

- AlwaysOn and LiDAR nodes do not capture identifying imagery
- Visitors are tracked as anonymous geometric objects
- No visual record of visitor appearance

**Acoustic Processing On-Device:**

- Audio is processed locally for event classification
- No audio recording by default
- No audio transmission or storage by default

**Identity Layer is Opt-In:**

- Identity nodes require hardware kill switch in ENABLED position
- User must have configured identity layer for any visitor identification
- Default: identity layer is dormant

**Activity-Based Triggering:**

- Identity layer can be configured to activate only on specific triggers
- User can configure: activate on unknown person, activate on motion in restricted zone, etc.
- Or: never activate (Privacy-First mode)

### 8.3 Residual Risk

**Presence is detectable:**

- A moving object of human size arriving and departing is logged
- Duration of visit is recorded
- This is inherent to any presence-sensing system

**Identity layer activation:**

- In Security-First mode, identity layer may activate on visitor arrival
- Visitor may be photographed if identity layer triggers
- This is the primary bystander-privacy risk

### 8.4 Guidance

**For users:**

1. **Inform visitors:** When Security-First mode is active, inform visitors that the home has an active security system with identification capability.

2. **Privacy-First mode:** For gatherings or social events, consider switching to Privacy-First mode (identity layer dormant).

3. **Review logs:** After visitors depart, review audit log to see what was captured.

4. **Deletion policy:** Configure retention and deletion of visitor-related data.

---

## 9. Threat: Remote Surveillance

### 9.1 Threat Description

Attacker gains remote access to the system to:
- View live feeds
- Access recordings
- Monitor presence in real-time
- Use the system as a surveillance platform

### 9.2 AEGIS-MESH Mitigations

**Edge Controller is Sole Gateway:**

- All external access must go through the edge controller
- Single point to secure, monitor, and audit
- No distributed attack surface across sensor nodes

**No Default Remote Access:**

- Remote access is disabled by default
- User must explicitly enable and configure
- Requires bearer token for authentication

**Optional TLS:**

- TLS recommended for remote access
- Protects against man-in-the-middle attacks
- User provides certificate

**Authentication Required:**

- All API endpoints require bearer token
- Token set by user; not stored in plaintext on device
- Token can be rotated

**Audit Logging:**

- All API access logged with source IP, timestamp, action
- Enables detection of unauthorized access patterns

**Cellular as Controlled Path:**

- If cellular enabled, edge controller connects only to user-configured endpoints
- No vendor telemetry endpoints
- No automatic data transmission

### 9.3 Residual Risk

**Compromised credentials:**

- If bearer token is stolen or coerced, attacker has full access
- Mitigation: use strong token; rotate regularly; protect token as password

**Vulnerable edge controller:**

- If edge controller OS or software has vulnerabilities, could be exploited
- Mitigation: keep software updated; use firewall; minimize exposed services

**Compromised companion app:**

- If app is compromised, could exfiltrate received data
- Mitigation: use official app; verify signatures

**User-enabled remote access:**

- If user enables remote access, attack surface increases
- This is a tradeoff: convenience vs security
- User must assess risk and configure accordingly

---

## 10. Threat: Data Breach

### 10.1 Threat Description

Data stored on the edge controller or nodes is accessed by unauthorized parties:
- Physical theft of SD card
- Physical theft of entire edge controller
- Unauthorized access by household members
- Access by service personnel

### 10.2 AEGIS-MESH Mitigations

**Optional Encrypted Storage:**

```toml
[data.encryption]
enabled = true
method = "aes-256-gcm"
key_source = "user_passphrase"
```

- When enabled, all data on SD card and internal flash is encrypted
- User passphrase required at boot to derive decryption key
- Without passphrase, physical theft yields only encrypted data

**Tamper-Evident Audit Log:**

- Audit log is append-only
- Hash chain enables detection of tampering
- Previous entries cannot be modified without detection

**SD Card Removability:**

- SD cards can be removed for physical access to data
- Encrypted storage protects against SD card theft
- User can choose to store SD card in secure location when not in use

### 10.3 Residual Risk

**Physical access = data access (if not encrypted):**

- If encryption not enabled, physical access to edge controller or SD card yields all data
- Mitigation: enable encryption; secure edge controller physically

**Passphrase compromise:**

- If passphrase is obtained by attacker, encryption provides no protection
- Mitigation: use strong passphrase; do not share

---

## 11. The Identification Layer — Detailed Threat Analysis

### 11.1 Architecture

The identification layer consists of:
- Identity nodes with configurable visual/biometric sensors
- Hardware kill switch on each identity node
- Optional local storage
- Optional streaming to companion app
- Optional remote access

### 11.2 Kill Switch as Primary Control

**The hardware kill switch is the unconditional privacy control:**

| Kill Switch Position | System Behavior |
|---------------------|-----------------|
| DISABLED (switch OFF) | Camera physically unpowered. No imagery possible. No firmware override. No remote override. All configuration options ignored. |
| ENABLED (switch ON) | Camera powered. Behavior determined by user configuration. |

**This is the architectural guarantee.** No amount of software compromise, remote access, or administrative privilege can produce imagery when the kill switch is DISABLED.

### 11.3 Configuration-Based Controls

When kill switch is ENABLED, user configuration determines behavior:

**Metadata-Only Mode:**
- On-device inference
- Only classification results transmitted
- No imagery stored or transmitted

**Local Storage Mode:**
- Imagery stored on local SD card
- No network transmission of imagery
- User accesses recordings via companion app on local network

**App Streaming Mode:**
- Imagery streamed to companion app on demand
- Streaming requires user-initiated connection
- No persistent cloud storage

**Continuous Recording Mode:**
- All imagery stored locally
- Retention period user-configured
- Legal export with integrity chain

### 11.4 Threat Analysis for Identity Layer

| Threat | Kill Switch DISABLED | Kill Switch ENABLED, Metadata-Only | Kill Switch ENABLED, Recording |
|--------|---------------------|-----------------------------------|-------------------------------|
| Remote surveillance | Impossible | Impossible | Requires credentials + remote access enabled |
| Local surveillance by abuser | Impossible | Minimal (only classification tags) | Possible with local access |
| Data breach (physical) | N/A | N/A | Protected by encryption (if enabled) |
| Visitor photography | Impossible | Impossible | Depends on trigger policy |
| Legal subpoena | No data to provide | Metadata only | All recordings (if compelled) |

---

## 12. Cellular Connectivity — Privacy Implications

### 12.1 Architecture

Cellular connectivity (LTE/5G) is present ONLY on the edge controller, never on sensor nodes.

### 12.2 Privacy Implications

**Benefits:**
- Enables remote access when away from home WiFi
- Enables alert delivery via cellular (not dependent on home internet)
- User-controlled endpoint (no vendor back-end)

**Risks:**
- Creates additional attack surface (cellular module)
- Data transmitted over cellular network
- Carrier may have visibility into connection patterns

**Mitigations:**
- TLS encryption for all cellular traffic
- Bearer token authentication
- No automatic data transmission
- User-configured endpoints only
- Audit logging of cellular connections

**Configuration:**

```toml
[connectivity.cellular]
enabled = false                   # Opt-in only
sim_type = "nano_sim"
apn = "your.carrier.apn"
fallback_to_wifi = true          # Prefer WiFi when available
always_on_cellular = false       # Cellular off by default
alert_via_cellular = true        # Send critical alerts via cellular
stream_via_cellular = false      # Do not stream video over cellular (data cost)
emergency_contact = ""
```

### 12.3 Carrier Visibility

**What the carrier can see:**
- Connection times and durations
- Data volumes
- Destination endpoints (IP addresses)

**What the carrier cannot see:**
- Payload content (encrypted with TLS)
- Specific alerts or detections
- Imagery content

**Residual risk:** Traffic analysis could reveal presence patterns (data spikes when alerts occur). This is inherent to any cellular communication.

---

## 13. The 360° Camera and SLAM — Privacy Implications

### 13.1 Architecture

LiDAR Node Variant D includes:
- Solid-state LiDAR
- 4-8 camera array covering 360° horizontal field of view
- Event camera for fast transient detection
- Optional mmWave radar

Data flow:
- Cameras capture 360° panoramic imagery
- LiDAR captures 3D geometry
- Edge controller runs SLAM for 3D reconstruction
- All data stored on edge controller SD card
- Streamed to companion app on user request

### 13.2 Privacy Implications

**Comprehensive visual coverage:**

A single LiDAR Variant D node captures 360° imagery of an entire room. This is the most surveillance-capable component of AEGIS-MESH.

**Mitigations:**

1. **Hardware kill switch:** Identity nodes have hardware kill switches. LiDAR Variant D nodes should also have a hardware kill switch for the camera array.

2. **User-configured activation:** Cameras can be configured to activate only on triggers, not continuously.

3. **Local storage only:** No cloud transmission of 360° video by default.

4. **Companion app streaming only:** Live 360° view requires user to request it through the companion app.

5. **Encryption:** SD card encryption protects against physical theft.

### 13.3 Deployment Guidance

For users deploying LiDAR Variant D nodes:

1. **Install hardware kill switch:** Ensure camera array has a hardware power switch accessible on the enclosure.

2. **Configure trigger-based activation:** Do not configure continuous recording unless necessary.

3. **Inform household members:** Ensure all residents understand that 360° coverage means comprehensive visual monitoring when cameras are active.

4. **Review retention policy:** Configure appropriate retention and deletion for 360° recordings.

5. **Secure edge controller:** The edge controller stores all 360° recordings. Physical security of the edge controller is critical.

---

## 14. User-Controlled Data Handling

### 14.1 Philosophy

AEGIS-MESH imposes no system-level restrictions on data handling. Users configure:
- What data is stored
- Where data is stored
- How long data is retained
- Whether data is streamed
- Whether data is accessible remotely

### 14.2 Configuration Reference

```toml
[data]
# Storage options
store_raw_video = false           # Store video from identity/LiDAR nodes
store_raw_audio = false           # Store audio from acoustic nodes
store_raw_sensor_data = false     # Store raw radar/LiDAR data
storage_target = "sd_card"        # "sd_card" | "internal" | "network_path"
retention_days = 30               # 0 = keep forever
continuous_recording = false
recording_trigger = "on_detection"  # "always" | "on_detection" | "on_alert" | "manual"
max_storage_mb = 0                # 0 = unlimited
include_integrity_chain = true    # Add hash chain for legal evidence

[companion_app]
enabled = true
api_port = 8080
media_port = 9090
enable_remote_access = false
remote_access_token = ""          # Set to enable remote access

[connectivity.cellular]
enabled = false
sim_type = "nano_sim"
apn = ""
alert_via_cellular = true
stream_via_cellular = false

[data.encryption]
enabled = false
method = "aes-256-gcm"
key_source = "user_passphrase"
```

### 14.3 Jurisdictional Responsibility

Users are responsible for complying with recording consent laws in their jurisdiction:

| Jurisdiction Type | Requirement |
|------------------|-------------|
| One-party consent (US most states) | At least one party to conversation must consent to recording |
| Two-party consent (US some states) | All parties must consent to audio recording |
| GDPR (EU) | Data minimization, consent, right to erasure |
| CCTV regulations (UK, others) | Notification signs, purpose limitation, retention limits |

AEGIS-MESH provides tools (metadata-only mode, retention limits, deletion) but does not enforce jurisdictional requirements. Users must understand and comply with applicable law.

---

## 15. Architectural Commitments

The mitigations in this threat model depend on AEGIS-MESH maintaining these architectural commitments:

### 15.1 Hardware Commitments

1. **Sensing mesh is camera-free at the compile-time level.** AlwaysOn nodes (variants A-C) and LiDAR nodes (variants A-B) do not link `omni-sense-drivers-camera`. No configuration can enable cameras on these variants.

2. **Identity nodes have hardware kill switches.** Physical switch cuts power to identification sensor. Firmware reads GPIO at boot. No firmware override.

3. **Edge controller is sole external gateway.** Sensor nodes have no WiFi, no cellular, no direct external connectivity.

### 15.2 Software Commitments

4. **Mesh protocol messages are signed.** HMAC-SHA256 on all messages. Replay protection via sequence numbers.

5. **Audit log is append-only.** Previous entries cannot be modified. Locally readable by any user.

6. **Firmware is reproducibly buildable.** Users can verify nodes run published code by comparing hashes.

7. **No mandatory data restrictions.** Users configure all data handling. The system imposes no jurisdiction-specific limits.

### 15.3 Process Commitments

8. **Pull requests that weaken architectural commitments will not be merged.** This includes:
   - Adding camera drivers to sensing-mesh node firmware
   - Removing hardware kill switch requirements from identity nodes
   - Adding external connectivity to sensing nodes
   - Weakening mesh protocol security
   - Making audit log entries modifiable

---

## 16. Honest Assessment: What AEGIS-MESH Cannot Do

### 16.1 Cannot Prevent State-Actor Access

A government with physical access and legal authority can access any local system. AEGIS-MESH raises the cost and visibility of access but cannot prevent it.

### 16.2 Cannot Resolve Consent Disputes

In multi-user households, any user with legitimate access can monitor other household members. The architecture cannot determine who has consent.

### 16.3 Cannot Prevent Physical Compromise

An attacker with physical access can modify hardware, install implants, or replace nodes. Defense against physical attacks is the user's responsibility.

### 16.4 Cannot Eliminate Presence Detection

Any presence-sensing system inherently detects that someone is present. The best privacy mitigation is to limit identification, not presence detection.

### 16.5 Cannot Enforce Jurisdictional Compliance

Users must understand and comply with recording consent, data protection, and surveillance laws in their jurisdiction. AEGIS-MESH provides tools but not legal enforcement.

---

## 17. Summary

AEGIS-MESH provides **strong privacy architecture** through:

1. **Local-first design** — no cloud dependency, no vendor back-end
2. **Camera-free sensing mesh** — no imagery from always-on nodes
3. **Hardware kill switches** — unconditional physical control over cameras
4. **Edge controller as sole gateway** — centralized security and audit
5. **Signed mesh protocol** — tamper detection
6. **Append-only audit log** — accountability
7. **Reproducible firmware** — verification

AEGIS-MESH provides **user-configurable capability** with:

1. **No mandatory data restrictions** — users choose what to store, stream, and retain
2. **Optional encryption** — protects against physical theft
3. **Optional remote access** — for users who want it
4. **Optional cellular** — for users who need it
5. **360° camera variants** — for users who want comprehensive visual coverage

**Residual risks are honestly acknowledged:** state-actor access, intimate-partner abuse, aggregate behavioral profiling, bystander presence detection. The architecture raises the bar but cannot eliminate all risks.

**Users are responsible for:** jurisdictional compliance, physical security, access control, and informed deployment.

---

**End of Privacy Threat Model**
