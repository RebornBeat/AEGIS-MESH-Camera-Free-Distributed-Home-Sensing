# Future Research Areas — AEGIS-MESH

## Purpose

This document maps the research domains adjacent to AEGIS-MESH's published distributed-sensing framework. AEGIS-MESH is the most regulatorily clean of the four projects in this family — camera-free residential sensing has no significant export-control or weapons-policy issue — so the future-research scope is broader than for the other projects.

MIT-licensed, survey-paper depth, no operational guidance.

---

## Tier 1 — Distributed Camera-Free Sensing (In Scope)

This is what AEGIS-MESH's repository develops. The mesh of mmWave radar, PIR, acoustic, solid-state LiDAR, and event-camera sensors with PentaTrack as the predictive substrate is the published research contribution. Open questions include:

**Sensor-modality complementarity at residential scale.** Each modality has characteristic strengths (mmWave through walls, LiDAR for geometry, event cameras for fast motion, acoustic for material classification) and characteristic failure modes. The fusion architecture's open research question is which modalities contribute information in which residential conditions.

**Privacy-preserving fusion.** The published differential-privacy and federated-learning literatures provide formal frameworks for limiting information leakage from aggregated data. AEGIS-MESH's research question is how to apply these frameworks to a residential mesh where the threat model includes both compromised individual nodes and compromised network observers.

**Adaptive remap.** Periodic LiDAR rescans detect furniture changes; the research question is how often to remap, what triggers a remap, and how to integrate the new map with historical track data.

**Cross-modal calibration.** Each node's sensors must be calibrated to a common spatial frame. The research question is how to calibrate automatically without user intervention, how to detect calibration drift, and how to recover from sensor failures gracefully.

**Multi-occupant tracking.** When multiple people are present, the tracker must distinguish them without imagery. PentaTrack's object-type awareness with per-track drift profiles is the natural framework; the open research question is how distinguishable are individual gait, motion-pattern, and acoustic signatures across occupants.

This tier is in scope. Civilian transfer is broad: hospital privacy-preserving patient monitoring, retail occupancy analytics without imagery, smart-building HVAC and lighting control, accessibility (haptic spatial awareness for visually impaired residents).

---

## Tier 2 — Activity Recognition and Anomaly Detection

The published activity-recognition literature addresses recognition of daily activities (cooking, sleeping, watching TV, exercising) from sensor data. AEGIS-MESH's research questions:

**Camera-free activity recognition.** How well can the mesh recognize activities without imagery? Published research on radar-based activity recognition, acoustic-based activity recognition, and motion-pattern-based activity recognition each have positive results; the fusion question is how the modalities combine.

**Anomaly detection.** Falls, intrusions, fires, water leaks, medical events. The published anomaly-detection literature provides statistical, machine-learning, and rule-based frameworks. The AEGIS-MESH-specific question is how to balance false-positive and false-negative rates in a residential context where false positives are annoying but false negatives can be dangerous.

**Behavioral baselines.** Long-term patterns of occupancy, activity, and movement establish baselines against which anomalies become detectable. The published research includes both general-purpose baseline-learning methods and population-specific methods (elderly, post-surgical, family with infants).

**Privacy-preserving anomaly detection.** Anomaly detection that does not require persistent storage of behavioral data. Differential-privacy and on-device-only approaches are published; the AEGIS-MESH-specific question is the system-architecture for on-device-only anomaly detection across a multi-node mesh.

This tier is in scope.

---

## Tier 3 — Health Monitoring (Research-Only, Non-Clinical)

The published research on contactless health monitoring addresses:

**Vital signs from radar.** Heart rate and breathing rate from mmWave radar are well-published with published accuracy characterization. The AEGIS-MESH-specific question is whether existing residential-grade mmWave nodes can perform vital-sign monitoring with useful accuracy.

**Sleep monitoring.** Sleep stage classification from radar and acoustic data. Published research is positive at research-grade accuracy; clinical-grade accuracy is regulated and outside research scope.

**Gait analysis.** Walking speed, gait variability, and stride characteristics from radar and LiDAR. Published research relates these to health status (frailty, post-stroke recovery, neurodegenerative disease).

**Fall-risk prediction.** Long-term gait-based prediction of fall risk. Published research is positive at population-level accuracy.

**Medication-adherence inference.** Activity-pattern-based inference of medication-related routines. Published research is preliminary.

The AEGIS-MESH posture is *strictly research-only and non-clinical*: any clinical extension requires regulatory clearance (FDA, EMA, etc.) that the repository does not provide. The published literature is the resource for researchers pursuing clinical extension.

This tier is in scope as research only. The medical-device disclaimer applies in full to anything in this tier.

---

## Tier 4 — Identity Layer Research (Opt-In Only)

The published research on contactless identification includes:

**Gait biometrics.** Published research demonstrates that gait is identifying at population scale. The published accuracy depends on sensor modality (LiDAR, radar, accelerometer), population size, and behavioral consistency.

**Acoustic biometrics.** Voice biometrics for speaker identification, on-device, are commercially mature. Footstep-based identification is published research at lower maturity.

**Behavioral biometrics.** Movement patterns, activity patterns, and habit patterns are individually identifying at population scale per published research.

The AEGIS-MESH posture: identity-capable layers are *opt-in only*, are physically isolated from the always-on mesh, and have hardware kill switches. The published research is the resource for understanding what is possible; the repository implements only the architectural separation.

This tier is in scope as architecture research only. Implementation of identity-capable nodes is opt-in and outside the always-on mesh.

---

## Tier 5 — Integration with Smart-Home Systems

Published research on smart-home integration addresses:

**HVAC and lighting control** based on occupancy and activity. Energy-efficiency benefits are well-published.

**Appliance integration** for kitchen-fire prevention, water-leak detection, and similar. Published research is mature.

**Voice assistants** with privacy-preserving local processing. Published research and commercial products both exist.

**Emergency-services integration.** AEGIS-MESH does not implement automatic emergency calling, by deliberate design. The research question of whether and how automatic calling should be implemented is published in the human-computer-interaction and emergency-response literature; the AEGIS-MESH posture is that automatic calling has serious failure modes (false-positive 911 calls have caused harm) and is opt-in only.

This tier is in scope as architecture research, with the integration interfaces defined and the integrations themselves left as user-configurable.

---

## Tier 6 — Privacy-Preserving Machine Learning at the Mesh

The published differential-privacy, federated-learning, and homomorphic-encryption literatures provide frameworks for ML on sensitive data. AEGIS-MESH's research questions:

**Federated training across meshes.** Multiple residential meshes contribute to a shared anomaly-detection model without sharing raw data. Published federated-learning research provides the framework; the AEGIS-MESH-specific question is the threat model and the privacy budget appropriate for residential data.

**Differentially-private aggregation.** Aggregate statistics across meshes (for research) without leaking individual residential data. Published differential-privacy research provides the framework.

**On-device learning.** Adaptation of generic anomaly detectors to a specific residential mesh's baseline, on-device. Published research is positive; the AEGIS-MESH-specific question is the resource budget on edge controllers.

**Adversarial robustness.** Robustness of mesh-based detection to adversaries who manipulate sensors or environment. Published adversarial-ML research provides the framework.

This tier is in scope.

---

## Tier 7 — Mesh Resilience and Cybersecurity

Residential meshes are network-attached devices. The published smart-home cybersecurity literature addresses:

**Compromised-node containment.** What happens when one mesh node is compromised. Architectural patterns (zero-trust, microsegmentation) are published; the AEGIS-MESH-specific question is how compromise affects the privacy posture of the larger mesh.

**Authentication and key management.** Per-node, per-user, and per-controller authentication. Published research is mature.

**Update integrity.** Signed firmware updates, reproducible builds, and update-rollback policies. Published research is mature.

**Network-attack resistance.** Resistance to denial-of-service against the mesh, replay attacks against mesh messages, and poisoning of the anomaly-detection baseline. Published research is mature.

**Side-channel attacks.** Attacks that infer mesh state from observable network traffic, power consumption, or RF emission. Published research is at varying maturity.

This tier is in scope.

---

## Tier 8 — Outdoor and Extended-Boundary Sensing

The published research on residential perimeter and outdoor sensing addresses:

**Driveway and approach monitoring.** Vehicle and pedestrian detection at the property boundary. Published modalities include radar, LiDAR, and ground-vibration sensors.

**Wildlife and pet monitoring.** Distinguishing pets and wildlife from human activity. Published research is mature for some species.

**Weather and environmental sensing.** Temperature, humidity, air quality, and similar. Commercially mature; integration with the mesh is straightforward.

**Vehicle classification.** Distinguishing residents' vehicles from visitors and unknown vehicles. Published research at varying maturity.

This tier is in scope.

---

## Tier 9 — Long-Horizon and Speculative

Programmable-matter sensors that reconfigure based on context. Bio-inspired distributed sensing modeled on animal sensory systems. Quantum sensing for ultra-low-power passive monitoring. Each is published in the academic literature at varying maturity.

The AEGIS-MESH-specific question for any of these is whether the mesh architecture's assumptions still hold when the underlying sensor technology changes. The published mesh layer is parameterized enough that researchers can substitute new sensor models as technology matures.

This tier is documented for completeness.

---

## Summary

AEGIS-MESH develops Tiers 1, 2, 3 (research-only, non-clinical), 4 (architecturally), 5, 6, 7, and 8. Tier 9 is documented as long-horizon. All MIT-licensed.
