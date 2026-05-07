# Hardware — AEGIS-MESH

This directory contains the hardware reference designs for AEGIS-MESH's sensor nodes:

- Always-on nodes (mmWave radar + PIR + MCU + microphone)
- LiDAR nodes (solid-state LiDAR + event sensor)
- Optional identity-layer nodes (physically isolated, opt-in, hardware kill switch required)

All designs are sensing-only and conform to the residential-electronics regulatory regimes documented in `legal/compliance.md` (FCC Part 15, CE RED, equivalent national regimes; UL or equivalent electrical safety; standard residential RF emission limits).

The architecture's privacy-by-design commitments (camera-free always-on layer, hardware kill switches on identity-capable nodes, local-only processing by default) are realized at the hardware level rather than relying solely on firmware or software for enforcement. PRs that would weaken these architectural commitments will be closed regardless of technical merit.

See `legal/compliance.md` and `legal/research_ethics.md` for the project's full posture.
