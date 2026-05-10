# Firmware Architecture — AEGIS-MESH

**Version:** 0.2 | **Status:** Research Reference Design | **Target:** `no_std` (ARM Cortex-M / RISC-V) / Linux (Edge Controller)

---

## 1. Purpose

This document specifies the firmware architecture for AEGIS-MESH sensing nodes and the edge controller. Three sensor node classes (AlwaysOn, LiDAR, Identity) each have a dedicated firmware binary compiled for the node's MCU target. The edge controller runs a full Rust software stack as a native Linux process.

**Firmware responsibilities:**
- Drive all sensors on the node (mmWave, acoustic, LiDAR, event camera, cameras, environmental).
- Perform on-node processing to reduce mesh bandwidth.
- Manage local storage (SD card, internal flash) per user data handling configuration.
- Communicate with the edge controller via the mesh radio (BLE, Thread, UWB, or PoE Ethernet).
- For Always-On Variant D and all camera-capable nodes: manage camera capture per user data handling policy.

**Scope:** Sensing-and-information only. No effectors. No actuation. Data handling is user-configured — firmware enforces the user's configuration without imposing additional restrictions.

---

## 2. Critical Architectural Constraint: Edge Controller as Sole External Gateway

### 2.1 The Connectivity Principle

**Only the Edge Controller has WiFi, Cellular/SIM, and direct companion app connectivity.**

**All sensor nodes (AlwaysOn, LiDAR, Identity) communicate exclusively via mesh radio to the edge controller.** They have no WiFi, no cellular, and no direct connection to external networks or the companion app.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      AEGIS-MESH Connectivity Model                          │
│                                                                              │
│  AlwaysOn Nodes ──┐                                                          │
│  LiDAR Nodes──────┤── Mesh Radio (BLE/Thread/UWB/PoE) ──► Edge Controller   │
│  Identity Nodes───┘                                          │              │
│                                                               ├──► WiFi ──► │
│                                                               │    Companion App
│                                                               │              │
│                                                               ├──► Cellular ──►
│                                                               │    Remote App
│                                                               │              │
│                                                               └──► BLE direct ──►
│                                                                    Local App │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 2.2 Why This Architecture

| Constraint | WiFi on Sensor Node | BLE on Sensor Node | UWB on Sensor Node |
|------------|---------------------|-------------------|-------------------|
| **Power** | 300-1000+ mW active | 5-15 mW active | 50-150 mW active |
| **Heat** | Significant thermal load | Minimal | Moderate |
| **Antenna space** | Requires clearance, diversity | Small PCB antenna | Moderate clearance |
| **Form factor impact** | Breaks ceiling/wall mount form factor | Compatible | Compatible |
| **Determinism** | Variable latency, contention | Consistent, schedulable | Excellent timing |
| **Regulatory** | Additional certification | Standard BLE certification | UWB-specific rules |

**Conclusion:** WiFi and Cellular on sensor nodes would destroy the low-power, always-on nature of the sensing mesh. All external connectivity is concentrated at the edge controller where power and thermal budgets are not constrained.

### 2.3 Firmware Implication

**Sensor node firmware:**
- NO WiFi driver
- NO Cellular driver
- NO external network stack
- Mesh radio driver ONLY (BLE 5.x, Thread, UWB, or PoE Ethernet)
- All data flows: sensors → MCU → mesh radio → edge controller

**Edge controller software:**
- WiFi driver (built into Linux SoM or external module)
- Cellular driver (LTE/5G module via USB or UART)
- HTTP/WebSocket server for companion app
- Mesh hub for all sensor nodes
- Fusion and SLAM processing

---

## 3. Target Platforms

### 3.1 Sensor Node MCUs

**Primary Targets:**
- ARM Cortex-M33 (STM32U5, nRF5340) — ultra-low power, TrustZone
- ARM Cortex-M7 (STM32H7, NXP i.MX RT1060) — higher performance for LiDAR nodes
- RISC-V (ESP32-S3) — for research variants

**Resource Profile (Minimum):**
- Flash: 512 KB (1 MB preferred)
- RAM: 256 KB (512 KB for LiDAR nodes)
- Storage: microSD card (for recording-capable nodes)

### 3.2 Edge Controller Platforms

| Variant | Platform | Compute | RAM | Storage |
|---------|----------|---------|-----|---------|
| A — SBC | Raspberry Pi 4/5 | ARM Cortex-A72/A76 | 4-8 GB | SD/NVMe |
| B — ARM-NPU | Jetson Nano/Orin | ARM + NVIDIA GPU/NPU | 4-16 GB | NVMe |
| C — x86 | Mini PC (Intel/AMD) | x86_64 | 8-32 GB | NVMe SSD |

**Edge controller runs full Linux. Software is standard `std` Rust, not `no_std`.**

---

## 4. Workspace Structure

```
firmware/
├── Cargo.toml
├── build.rs
├── memory.x
├── firmware.md
├── README.md
└── src/
    ├── lib.rs
    ├── bin/
    │   ├── always_on_node.rs           # Variants A–C (no camera)
    │   ├── always_on_node_cam.rs       # Variant D (with camera)
    │   ├── lidar_node.rs               # Variants A–B (no camera)
    │   ├── lidar_node_cam.rs           # Variant C (with camera)
    │   ├── lidar_node_360.rs           # Variant D (360° camera array)
    │   └── identity_node.rs            # All identity variants
    ├── drivers/
    │   ├── mod.rs
    │   ├── mmwave.rs                   # mmWave radar driver
    │   ├── lidar.rs                    # Solid-state LiDAR
    │   ├── event_sensor.rs             # Event camera (microsecond trigger)
    │   ├── acoustic.rs                 # Microphone array
    │   ├── pir.rs                      # PIR binary detection
    │   ├── tof.rs                      # 1D ToF for choke-point nodes
    │   ├── environmental.rs            # Temperature, humidity, VOC
    │   ├── camera.rs                   # Single camera (MIPI/DVP)
    │   ├── camera_360.rs               # 360° camera array
    │   ├── storage.rs                  # SD card, integrity chain
    │   └── power.rs                    # Power management, battery
    ├── logic/
    │   ├── mod.rs
    │   ├── presence_detection.rs       # Radar-based presence
    │   ├── material_classifier.rs      # Acoustic material ID
    │   ├── recording_manager.rs        # SD recording lifecycle
    │   └── calibration.rs              # On-node calibration
    └── mesh_radio/
        ├── mod.rs
        ├── ble.rs                      # BLE 5.x protocol
        ├── thread.rs                   # Thread/Matter protocol
        ├── uwb.rs                      # UWB for precision timing
        ├── poe_ethernet.rs             # PoE mesh for LiDAR nodes
        └── protocol.rs                 # Message serialization
```

---

## 5. Node Binary Descriptions

### 5.1 `always_on_node.rs` (Variants A–C)

**Camera-free. No camera driver linked.**

Handles:
- PIR binary detection (Variant A)
- mmWave radar presence detection
- Microphone array acoustic processing
- Environmental sensors
- Mesh radio communication

**Camera-free at compile-time:** The `omni-sense-drivers-camera` crate is not a dependency. Users cannot "enable" cameras via configuration on these variants — it is architecturally impossible.

### 5.2 `always_on_node_cam.rs` (Variant D)

**All of Variants A–C plus:**
- Camera driver (MIPI CSI-2 or DVP)
- Data handling per `aegis-mesh.toml`
- Optional hardware power switch GPIO

**Data flow:**
```
Camera → MCU (encode) → Mesh Radio → Edge Controller → WiFi → Companion App
```

The camera data does NOT go directly to the companion app. It routes through the edge controller.

### 5.3 `lidar_node.rs` (Variants A–B)

**No conventional camera.**

Handles:
- Solid-state LiDAR point cloud capture
- Event camera trigger (microsecond latency)
- Optional mmWave radar fusion
- Mesh radio (BLE/Thread or PoE Ethernet)

**Event camera role:** Provides microsecond-latency wake signal for LiDAR activation. The event sensor is NOT a conventional camera — it produces sparse event data, not imagery.

### 5.4 `lidar_node_cam.rs` (Variant C)

**All of Variants A–B plus:**
- Conventional camera for SLAM visual odometry
- Camera data handling per configuration
- Video stream via mesh radio to edge controller

### 5.5 `lidar_node_360.rs` (Variant D)

**All of Variants A–B plus:**
- 4–8 camera array (360° coverage)
- FSYNC hardware synchronization
- Stitching data pipeline

**Data flow for 360°:**
```
8 Cameras → Vision Processor on Node
                   │
                   ├── Option A: Stitch on-node → Single stream to edge
                   └── Option B: Raw streams to edge → Stitch at edge controller
                                                   │
                                                   ▼
                                           WiFi → Companion App
```

### 5.6 `identity_node.rs`

**Configurable identification sensor.**

Handles:
- Visual or biometric capture
- On-device inference (classification)
- Data handling per user configuration:
  - Metadata-only: classification tags only
  - Local storage: raw frames to SD
  - App streaming: video to edge controller → WiFi → app
  - Full recording: all data stored
- Optional hardware power switch (physical camera disable)

---

## 6. Mesh Radio Architecture

### 6.1 Transport Options

| Transport | Nodes Supported | Bandwidth | Latency | Use Case |
|-----------|-----------------|-----------|---------|----------|
| BLE 5.x | All nodes | 1-2 Mbps | 5-30 ms | Standard mesh |
| Thread | All nodes | 250 Kbps | 10-50 ms | Low-power mesh |
| UWB | Optional | 4-6 Mbps | < 1 ms | Precision timing, high-bandwidth |
| PoE Ethernet | LiDAR nodes | 100 Mbps+ | < 1 ms | High-bandwidth, ceiling-mounted |

### 6.2 Mesh Protocol

All sensor nodes communicate ONLY with the edge controller. There is no node-to-node direct communication — all traffic routes through the edge controller.

**Message types:**

```rust
/// Mesh protocol message from sensor node to edge controller
#[derive(Debug, Clone, Serialize, Deserialize)]
#[serde(tag = "type")]
pub enum MeshMessage {
    /// Sensor detection event
    Detection {
        node_id: NodeId,
        timestamp_us: u64,
        detection: DetectionEvent,
    },
    
    /// Identification result from identity node
    Identification {
        node_id: NodeId,
        timestamp_us: u64,
        classification: String,
        confidence: f32,
        /// Path to recording if local storage configured
        recording_path: Option<String>,
        /// True if live stream available
        live_stream_available: bool,
    },
    
    /// Recording available for retrieval
    RecordingAvailable {
        node_id: NodeId,
        recording_path: String,
        start_timestamp_us: u64,
        duration_us: u32,
        size_bytes: u64,
        format: String,
        sha256: String,
    },
    
    /// Node health heartbeat
    Heartbeat {
        node_id: NodeId,
        timestamp_us: u64,
        battery_percent: Option<u8>,
        sensor_status: Vec<SensorStatus>,
    },
    
    /// Calibration data
    Calibration {
        node_id: NodeId,
        position: Option<[f32; 3]>,
        orientation: Option<[f32; 4]>,
    },
    
    /// 360° panorama frame available
    PanoramaFrame {
        node_id: NodeId,
        timestamp_us: u64,
        width: u32,
        height: u32,
        format: String,
        frame_path: Option<String>,
    },
}

/// Detection event from any sensor modality
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct DetectionEvent {
    pub class: DetectionClass,
    pub position: Option<[f32; 3]>,
    pub velocity: Option<[f32; 3]>,
    pub confidence: f32,
    pub sensor_modalities: Vec<String>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum DetectionClass {
    Human,
    Pet,
    Vehicle,
    Anomaly,
    Unknown,
}
```

### 6.3 Time Synchronization

**Edge controller as time master:**
- Broadcasts time sync messages at each heartbeat interval
- All nodes estimate clock offset using PTP-style four-timestamp exchange
- Per-node `omni-sense-time::ClockOffsetEstimator` tracks drift

**UWB upgrade:** For nodes with UWB hardware, sub-nanosecond time sync is achievable, essential for multi-node SLAM accuracy.

---

## 7. Driver Layer Specifications

### 7.1 `camera.rs` (Single Camera)

```rust
/// Camera driver for identity and LiDAR-cam nodes
pub struct CameraDriver {
    interface: CameraInterface,
    config: CameraConfig,
    encoder: VideoEncoder,
}

pub struct CameraConfig {
    /// Resolution: width × height
    pub resolution: (u16, u16),
    /// Frame rate in fps
    pub frame_rate: u8,
    /// Data handling mode
    pub data_mode: CameraDataMode,
    /// Hardware power switch GPIO (if installed)
    pub hw_switch_pin: Option<GpioPin>,
    /// SD card storage path prefix
    pub storage_path: Option<String>,
}

#[derive(Debug, Clone, Copy)]
pub enum CameraDataMode {
    /// On-device inference only, transmit classification tags
    MetadataOnly,
    /// Write raw frames to SD card
    LocalStorage,
    /// Stream encoded video to edge controller
    AppStreaming,
    /// Full continuous recording
    FullRecording,
}

impl CameraDriver {
    /// Initialize camera, checking hardware switch if present
    pub fn init(&mut self) -> Result<(), CameraError> {
        // Check hardware power switch if pin configured
        if let Some(ref pin) = self.config.hw_switch_pin {
            if pin.is_low() {
                // Hardware switch is DISABLED
                return Err(CameraError::HardwareSwitchDisabled);
            }
        }
        
        // Initialize camera sensor
        self.interface.init()?;
        
        Ok(())
    }
    
    /// Capture frame according to data handling mode
    pub fn capture(&mut self) -> Result<CameraFrame, CameraError> {
        let frame = self.interface.capture_frame()?;
        
        match self.config.data_mode {
            CameraDataMode::MetadataOnly => {
                // Run inference on device
                let classification = self.run_inference(&frame)?;
                // Frame is discarded, only classification returned
                Ok(CameraFrame::Metadata(classification))
            }
            CameraDataMode::LocalStorage => {
                // Write to SD card
                let path = self.write_to_storage(&frame)?;
                Ok(CameraFrame::Stored(path))
            }
            CameraDataMode::AppStreaming | CameraDataMode::FullRecording => {
                // Encode and queue for mesh transmission
                let encoded = self.encoder.encode(&frame)?;
                Ok(CameraFrame::Stream(encoded))
            }
        }
    }
}
```

### 7.2 `camera_360.rs` (360° Camera Array)

```rust
/// 360° camera array driver for LiDAR Variant D
pub struct Camera360Driver {
    cameras: Vec<CameraInstance>,
    fsync_pin: GpioPin,
    calibration: StitchingCalibration,
    config: Camera360Config,
}

pub struct Camera360Config {
    /// Number of cameras in array
    pub camera_count: u8,
    /// Resolution per camera
    pub resolution: (u16, u16),
    /// Hardware FSYNC pin for simultaneous capture
    pub fsync_pin: GpioPin,
    /// Stitching mode
    pub stitch_mode: StitchMode,
    /// Data handling mode
    pub data_mode: CameraDataMode,
}

#[derive(Debug, Clone, Copy)]
pub enum StitchMode {
    /// Stitch on-node, transmit single panorama
    OnNode,
    /// Transmit individual streams, stitch at edge controller
    AtEdge,
    /// Progressive: low-res baseline + high-res keyframes
    Progressive,
}

impl Camera360Driver {
    /// Initialize all cameras and load calibration
    pub fn init(&mut self) -> Result<(), CameraError> {
        // Load calibration from flash
        self.calibration = self.load_calibration_from_flash()?;
        
        // Initialize each camera
        for cam in &mut self.cameras {
            cam.init()?;
        }
        
        Ok(())
    }
    
    /// Capture synchronized frame from all cameras
    pub fn capture_sync(&mut self) -> Result<Capture360, CameraError> {
        // Assert FSYNC to trigger all cameras simultaneously
        self.fsync_pin.set_high();
        // Hold for sync duration
        delay_us(100);
        self.fsync_pin.set_low();
        
        // Collect frames from all cameras
        let frames: Vec<CameraFrame> = self.cameras
            .iter_mut()
            .map(|c| c.capture())
            .collect::<Result<Vec<_>, _>>()?;
        
        match self.config.stitch_mode {
            StitchMode::OnNode => {
                // Stitch locally
                let panorama = self.stitch(&frames)?;
                Ok(Capture360::Stitched(panorama))
            }
            StitchMode::AtEdge => {
                // Return individual frames for edge controller stitching
                Ok(Capture360::Individual(frames))
            }
            StitchMode::Progressive => {
                // Generate low-res baseline + queue high-res keyframes
                let baseline = self.generate_baseline(&frames)?;
                Ok(Capture360::Progressive {
                    baseline,
                    keyframes: frames,
                })
            }
        }
    }
    
    /// Stitch individual frames into panorama
    fn stitch(&self, frames: &[CameraFrame]) -> Result<Panorama, StitchError> {
        // Use calibration data for transformation parameters
        // Output equirectangular or cylindrical panorama
        // Implementation uses loaded calibration transforms
        todo!()
    }
}
```

### 7.3 `storage.rs`

```rust
/// SD card storage manager with integrity chain
pub struct StorageDriver {
    sd_card: SdCard,
    config: StorageConfig,
    current_recording: Option<RecordingHandle>,
}

pub struct StorageConfig {
    /// Mount point for SD card
    pub mount_path: String,
    /// Maximum storage in MB (0 = unlimited)
    pub max_storage_mb: u64,
    /// Retention period in days (0 = forever)
    pub retention_days: u32,
    /// Generate SHA-256 integrity chain
    pub include_integrity_chain: bool,
}

pub struct RecordingHandle {
    pub path: String,
    pub start_time_us: u64,
    pub frames_written: u64,
    pub hasher: Sha256,
}

impl StorageDriver {
    /// Open new recording session
    pub fn open_recording(&mut self, prefix: &str) -> Result<RecordingHandle, StorageError> {
        let timestamp = self.get_timestamp_us();
        let filename = format!("{}_{}.mp4", prefix, timestamp);
        let path = format!("{}/{}", self.config.mount_path, filename);
        
        Ok(RecordingHandle {
            path: path.clone(),
            start_time_us: timestamp,
            frames_written: 0,
            hasher: Sha256::new(),
        })
    }
    
    /// Write frame to recording
    pub fn write_frame(&mut self, handle: &mut RecordingHandle, frame: &[u8]) -> Result<(), StorageError> {
        self.sd_card.write(&handle.path, frame)?;
        
        // Update hash for integrity chain
        if self.config.include_integrity_chain {
            handle.hasher.update(frame);
        }
        
        handle.frames_written += 1;
        Ok(())
    }
    
    /// Close recording and generate integrity manifest
    pub fn close_recording(&mut self, handle: RecordingHandle) -> Result<IntegrityManifest, StorageError> {
        let final_hash = handle.hasher.finalize();
        
        let manifest = IntegrityManifest {
            recording_path: handle.path.clone(),
            start_time_us: handle.start_time_us,
            duration_us: self.get_timestamp_us() - handle.start_time_us,
            frames: handle.frames_written,
            sha256: hex::encode(final_hash),
            firmware_version: FIRMWARE_VERSION,
            hardware_id: self.get_hardware_id(),
            generated_at_us: self.get_timestamp_us(),
        };
        
        // Write manifest alongside recording
        let manifest_path = format!("{}.manifest.json", handle.path);
        let manifest_json = serde_json::to_string(&manifest)?;
        self.sd_card.write(&manifest_path, manifest_json.as_bytes())?;
        
        Ok(manifest)
    }
    
    /// Enforce retention policy
    pub fn enforce_retention(&mut self) -> Result<u64, StorageError> {
        let now = self.get_timestamp_us();
        let retention_us = self.config.retention_days as u64 * 24 * 60 * 60 * 1_000_000;
        
        let mut deleted_bytes = 0u64;
        
        for recording in self.list_recordings()? {
            let age_us = now - recording.start_time_us;
            if age_us > retention_us {
                self.delete_recording(&recording.path)?;
                deleted_bytes += recording.size_bytes;
            }
        }
        
        Ok(deleted_bytes)
    }
}

#[derive(Debug, Serialize, Deserialize)]
pub struct IntegrityManifest {
    pub recording_path: String,
    pub start_time_us: u64,
    pub duration_us: u64,
    pub frames: u64,
    pub sha256: String,
    pub firmware_version: String,
    pub hardware_id: String,
    pub generated_at_us: u64,
}
```

### 7.4 `mmwave.rs` (mmWave Radar)

```rust
/// mmWave radar driver for presence detection
pub struct MmWaveDriver {
    spi: SpiDevice,
    config: RadarConfig,
}

pub struct RadarConfig {
    /// Operating frequency band
    pub frequency_band: RadarBand,
    /// Maximum detection range in meters
    pub max_range_m: f32,
    /// Range resolution in meters
    pub range_resolution_m: f32,
    /// Maximum velocity in m/s
    pub max_velocity_ms: f32,
    /// Velocity resolution in m/s
    pub velocity_resolution_ms: f32,
    /// Update rate in Hz
    pub update_rate_hz: u16,
}

#[derive(Debug, Clone, Copy)]
pub enum RadarBand {
    Ghz24,
    Ghz60,
    Ghz77,
}

impl MmWaveDriver {
    /// Configure radar profile
    pub fn configure(&mut self, profile: RadarProfile) -> Result<(), RadarError> {
        match profile {
            RadarProfile::StandardPresence => {
                // Standard presence detection profile
                self.write_profile(StandardPresenceConfig::default())?;
            }
            RadarProfile::ExtendedDoppler => {
                // Extended Doppler range for fast object detection
                self.write_profile(ExtendedDopplerConfig::default())?;
            }
            RadarProfile::LongRange => {
                // Maximize range, trade resolution
                self.write_profile(LongRangeConfig::default())?;
            }
        }
        Ok(())
    }
    
    /// Read detection frame
    pub fn read_detections(&mut self) -> Result<Vec<Detection>, RadarError> {
        let raw = self.read_frame()?;
        let detections = self.process_frame(&raw)?;
        Ok(detections)
    }
}

#[derive(Debug, Clone, Copy)]
pub enum RadarProfile {
    StandardPresence,
    ExtendedDoppler,
    LongRange,
}
```

### 7.5 `acoustic.rs`

```rust
/// Acoustic sensing driver for material classification
pub struct AcousticDriver {
    i2s: I2sDevice,
    microphone_count: u8,
    beamformer: Beamformer,
    classifier: MaterialClassifier,
}

impl AcousticDriver {
    /// Read audio frame and classify events
    pub fn process_frame(&mut self) -> Result<Option<AcousticEvent>, AcousticError> {
        let samples = self.i2s.read_frame()?;
        
        // Beamforming for DOA
        let doa = self.beamformer.compute_doa(&samples)?;
        
        // Event detection
        if let Some(event) = self.detect_event(&samples) {
            // Material classification
            let material = self.classifier.classify(&samples)?;
            
            return Ok(Some(AcousticEvent {
                event_type: event,
                direction_of_arrival: doa,
                material_hint: material,
                timestamp_us: self.get_timestamp_us(),
            }));
        }
        
        Ok(None)
    }
}

#[derive(Debug, Clone)]
pub struct AcousticEvent {
    pub event_type: AcousticEventType,
    pub direction_of_arrival: (f32, f32), // azimuth, elevation
    pub material_hint: Option<MaterialClass>,
    pub timestamp_us: u64,
}

#[derive(Debug, Clone)]
pub enum AcousticEventType {
    GlassBreak,
    Footstep,
    DoorOpen,
    DoorClose,
    Impact,
    Water,
    Unknown,
}

#[derive(Debug, Clone)]
pub enum MaterialClass {
    Glass,
    Ceramic,
    Wood,
    Metal,
    Plastic,
    Biological,
    Unknown,
}
```

### 7.6 `event_sensor.rs`

```rust
/// Event camera driver for microsecond-latency detection
pub struct EventSensorDriver {
    interface: EventInterface,
    event_buffer: EventBuffer,
}

impl EventSensorDriver {
    /// Poll for events (non-blocking)
    pub fn poll_events(&mut self) -> Result<Option<Vec<Event>>, EventError> {
        let events = self.interface.read_events()?;
        
        if events.is_empty() {
            return Ok(None);
        }
        
        // Analyze event pattern for fast motion
        if self.detect_fast_motion(&events) {
            return Ok(Some(events));
        }
        
        Ok(None)
    }
    
    /// Detect fast motion from event burst
    fn detect_fast_motion(&self, events: &[Event]) -> bool {
        // High event density in short time indicates fast motion
        let event_rate = events.len() as f32 / self.config.window_ms as f32;
        event_rate > self.config.fast_motion_threshold
    }
}
```

---

## 8. Mesh Protocol Implementation

### 8.1 BLE Mesh Protocol

```rust
/// BLE mesh protocol handler
pub struct BleMeshProtocol {
    ble: BleDevice,
    edge_address: BleAddress,
    connection_interval_ms: u16,
    message_buffer: Vec<u8>,
}

impl BleMeshProtocol {
    /// Connect to edge controller
    pub fn connect(&mut self) -> Result<(), MeshError> {
        self.ble.connect(&self.edge_address)?;
        self.ble.set_connection_interval(self.connection_interval_ms)?;
        Ok(())
    }
    
    /// Send message to edge controller
    pub fn send(&mut self, message: &MeshMessage) -> Result<(), MeshError> {
        let encoded = message.encode();
        
        // Sign message
        let signed = self.sign_message(&encoded)?;
        
        // Transmit over BLE
        self.ble.write(&signed)?;
        
        Ok(())
    }
    
    /// Sign message with HMAC-SHA256
    fn sign_message(&self, payload: &[u8]) -> Result<Vec<u8>, MeshError> {
        let key = self.get_signing_key();
        let mut hmac = HmacSha256::new_from_slice(&key)?;
        hmac.update(payload);
        
        let mut signed = payload.to_vec();
        let signature = hmac.finalize().into_bytes();
        signed.extend_from_slice(&signature);
        
        Ok(signed)
    }
}
```

### 8.2 PoE Ethernet Mesh (LiDAR Nodes)

```rust
/// PoE Ethernet mesh for LiDAR nodes
pub struct PoeMeshProtocol {
    ethernet: EthernetDevice,
    edge_address: SocketAddr,
}

impl PoeMeshProtocol {
    /// Send high-bandwidth data (LiDAR point clouds, camera streams)
    pub fn send_stream(&mut self, stream: &[u8]) -> Result<(), MeshError> {
        // Split into chunks for UDP transmission
        for chunk in stream.chunks(MAX_CHUNK_SIZE) {
            self.ethernet.send_to(chunk, &self.edge_address)?;
        }
        
        Ok(())
    }
    
    /// Send metadata (detections, events)
    pub fn send_message(&mut self, message: &MeshMessage) -> Result<(), MeshError> {
        let encoded = message.encode();
        let signed = self.sign_message(&encoded)?;
        self.ethernet.send_to(&signed, &self.edge_address)?;
        Ok(())
    }
}
```

---

## 9. Data Handling Architecture

### 9.1 Configuration Schema

```toml
# aegis-mesh.toml

[data_handling]
# Raw data storage options
store_raw_video = false              # Enable raw video storage
store_raw_audio = false              # Enable raw audio storage
store_raw_sensor_data = false        # Enable raw radar/LiDAR data storage
storage_target = "internal"          # "sd_card" | "internal" | "network_path"
retention_days = 30                  # 0 = keep forever
max_storage_mb = 0                   # 0 = unlimited

# Recording behavior
recording_trigger = "on_detection"   # "always" | "on_detection" | "on_alert" | "manual"
continuous_recording = false

# Companion app integration
enable_app_streaming = false         # Stream to companion app
stream_quality = "medium"            # "low" | "medium" | "high" | "raw"

# Integrity chain
include_integrity_chain = true      # SHA-256 hash for legal evidence

# Identity node specific
[identity_node.data]
mode = "metadata_only"              # "metadata_only" | "local_storage" | "app_streaming" | "full_recording"
store_raw = false
```

### 9.2 Data Handling Modes

**Mode A — Metadata-Only (Default):**
- Inference runs on MCU
- Only classification tags transmitted over mesh
- No raw data stored or transmitted
- Lowest bandwidth, highest privacy

**Mode B — Local Storage:**
- Raw data written to SD card
- Recording metadata transmitted to edge controller
- User retrieves via companion app (edge controller reads from node SD)
- Suitable for: evidence capture, privacy-sensitive deployments

**Mode C — App Streaming:**
- Raw or encoded data transmitted to edge controller over mesh
- Edge controller relays to companion app via WiFi
- Live viewing in companion app
- Requires sufficient mesh bandwidth (UWB or PoE for video)

**Mode D — Full Recording:**
- All data stored locally
- Optional streaming to app
- Complete audit trail
- Highest storage requirement

### 9.3 Hardware Power Switch (Optional)

**If user installs a hardware power switch on the camera rail:**

```rust
/// Check hardware power switch at boot
fn check_hw_switch(&self) -> Result<(), SensorError> {
    if let Some(ref pin) = self.config.hw_switch_pin {
        if pin.is_low() {
            // Hardware switch is in DISABLED position
            // Camera is physically unpowered
            return Err(SensorError::HardwareSwitchDisabled);
        }
    }
    
    // Switch is ENABLED or not installed
    Ok(())
}
```

**The hardware switch overrides all software configuration.** If DISABLED, the camera cannot be initialized regardless of any software settings.

---

## 10. Power Management

### 10.1 Power States

| State | MCU | Sensors | Radio | Wake Source |
|---|---|---|---|---|
| Active | Full power | All active | Connected | N/A |
| Idle | Low power | Duty cycled | Connected | Timer, interrupt |
| Sleep | Off (RAM retained) | Key sensors | Advertising | PIR, timer |
| Deep Sleep | Off | RTC only | Off | External interrupt |

### 10.2 Duty Cycling

```rust
/// Power management configuration
pub struct PowerConfig {
    /// Sleep after inactivity (seconds)
    pub sleep_after_s: u16,
    /// Sensor duty cycle in idle (percentage active)
    pub idle_duty_cycle_percent: u8,
    /// Reduce update rate in idle
    pub idle_update_rate_hz: u16,
}

impl PowerManager {
    /// Enter low-power mode
    pub fn enter_idle(&mut self) {
        // Reduce sensor update rate
        self.set_sensor_update_rate(self.config.idle_update_rate_hz);
        
        // Keep radio connected for quick wake
        // But increase connection interval for power savings
        self.mesh.set_connection_interval(100); // 100 ms
    }
    
    /// Enter sleep mode
    pub fn enter_sleep(&mut self) {
        // Power down most sensors
        for sensor in &mut self.sensors {
            sensor.power_down();
        }
        
        // Keep PIR active for wake trigger
        self.pir.enable();
        
        // Enter MCU low-power mode
        self.mcu.enter_sleep_mode();
    }
}
```

---

## 11. Build System

### 11.1 Cargo.toml

```toml
[workspace]
members = ["."]

[package]
name = "aegis-mesh-firmware"
version = "0.2.0"
edition = "2021"

[dependencies]
cortex-m = "0.7"
cortex-m-rt = "0.7"
embedded-hal = "1.0"
panic-halt = "0.2"
embedded-sdmmc = "0.7"
serde = { version = "1.0", default-features = false, features = ["derive"] }
postcard = "1.0"
heapless = "0.7"

# Crypto for message signing
hmac = "0.12"
sha2 = { version = "0.10", default-features = false }

[features]
default = []
camera = []
camera_360 = ["camera"]
hw_camera_switch = []

[[bin]]
name = "always_on_node"
path = "src/bin/always_on_node.rs"

[[bin]]
name = "always_on_node_cam"
path = "src/bin/always_on_node_cam.rs"
required-features = ["camera"]

[[bin]]
name = "lidar_node"
path = "src/bin/lidar_node.rs"

[[bin]]
name = "lidar_node_cam"
path = "src/bin/lidar_node_cam.rs"
required-features = ["camera"]

[[bin]]
name = "lidar_node_360"
path = "src/bin/lidar_node_360.rs"
required-features = ["camera_360"]

[[bin]]
name = "identity_node"
path = "src/bin/identity_node.rs"
required-features = ["camera"]

[profile.release]
opt-level = "z"
lto = true
codegen-units = 1
strip = true
```

### 11.2 Memory Layout

```text
/* memory.x */

MEMORY
{
  FLASH : ORIGIN = 0x08000000, LENGTH = 1M
  RAM : ORIGIN = 0x20000000, LENGTH = 256K
}

_stack_start = ORIGIN(RAM) + LENGTH(RAM);
```

---

## 12. Calibration

### 12.1 On-Node Calibration Data

```rust
/// Node calibration data stored in flash
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct NodeCalibration {
    /// Unique node identifier
    pub node_id: NodeId,
    /// Position in home coordinate frame [x, y, z]
    pub position: [f32; 3],
    /// Orientation quaternion [w, x, y, z]
    pub orientation: [f32; 4],
    /// Sensor-specific calibration
    pub sensor_calibrations: HashMap<String, SensorCalibration>,
    /// Calibration confidence (0.0-1.0)
    pub confidence: f32,
    /// Timestamp of calibration
    pub calibrated_at_us: u64,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum SensorCalibration {
    Radar {
        range_bias_m: f32,
        angle_bias_rad: f32,
    },
    Acoustic {
        doa_bias_rad: f32,
        sensitivity_correction: f32,
    },
    Camera {
        intrinsics: CameraIntrinsics,
        distortion: DistortionCoefficients,
    },
    Camera360 {
        camera_count: u8,
        per_camera_extrinsics: Vec<CameraExtrinsics>,
        stitch_calibration: StitchCalibration,
    },
}
```

### 12.2 Calibration Procedure

1. **Edge controller initiates:** Sends `CalibrationCommand` to all nodes
2. **User walks defined path:** Triggers multiple node detections
3. **Edge controller trilaterates:** Computes node positions from simultaneous detections
4. **Calibration data transmitted:** Sent to each node and stored in flash
5. **Confidence calculation:** Based on detection count and geometric spread

---

## 13. Companion App Communication

### 13.1 Edge Controller API Server

The edge controller runs an embedded HTTP/WebSocket server. Sensor nodes do NOT communicate directly with the companion app — all communication routes through the edge controller.

**REST Endpoints:**

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/nodes` | List all nodes with health status |
| GET | `/api/zones` | List all zones |
| GET | `/api/alerts` | Recent alerts |
| GET | `/api/recordings` | List recordings |
| GET | `/api/recordings/{id}` | Download recording |
| GET | `/api/recordings/{id}/manifest` | Integrity manifest |
| GET | `/api/nodes/{id}/stream` | Open live media stream |
| POST | `/api/config` | Update configuration |
| GET | `/api/world-model` | Current world model |

**WebSocket Streams:**

| Path | Description |
|------|-------------|
| `/ws/events` | All mesh events |
| `/ws/detections` | Detection events |
| `/ws/alerts` | Zone alerts |

**Media Streaming:**

| Protocol | Port | Description |
|----------|------|-------------|
| RTSP | 9090 | Per-node video stream |
| WebSocket H.264 | 9091 | Browser-compatible stream |
| WebSocket 360° | 9092 | 360° panorama stream |

### 13.2 Data Flow

```
Sensor Node
    │
    │ (mesh radio: BLE/Thread/UWB/PoE)
    ▼
Edge Controller
    ├── Storage (SD/NVMe)
    ├── SLAM Processing
    ├── Fusion Processing
    └── API Server
         │
         ├── WiFi → Companion App (local)
         └── Cellular → Companion App (remote)
```

---

## 14. Legal & Compliance

### 14.1 Scope Limitation

- **Sensing-only effectors:** No actuation path in any binary
- **Data handling:** User-configured. No system-imposed restrictions
- **Recording laws:** Deployer's responsibility to comply with jurisdiction-specific requirements

### 14.2 Radio Compliance

- BLE 5.x modules: FCC Part 15, CE RED, ISED certified
- UWB modules: FCC Part 15 Subpart F, CE RED
- User responsible for proper installation and antenna use

### 14.3 Electrical Safety

- All nodes powered via low-voltage DC (5V or less)
- PoE nodes: IEEE 802.3af/at compliant
- Battery protection: Hardware overcharge/overdischarge required
- Firmware provides secondary cutoff thresholds

### 14.4 Data Integrity

- SHA-256 hash chain for all recordings
- Firmware version embedded in integrity manifest
- Reproducible builds supported

---

## 15. Testing

### 15.1 Unit Tests

```bash
# Run unit tests on host
cargo test
```

### 15.2 Hardware-in-the-Loop Testing

Test jig provides:
- Programmatic power cycling
- Simulated sensor inputs
- Mesh golden node for connectivity verification
- RF shielded chamber for radar testing

### 15.3 Data Handling Mode Verification

| Mode | Verification Test |
|------|-------------------|
| Metadata-only | Mesh packet contains classification tags, no raw data |
| Local storage | Recording file on SD within 500 ms |
| App streaming | Edge controller receives stream within 2 seconds |
| Full recording | File present with correct duration and hash |

---

## 16. Debug Interface

### 16.1 SWD

Standard ARM Serial Wire Debug via 10-pin connector or Tag-Connect footprint.

### 16.2 UART Console

Debug output at 115200 baud. Controlled by compile-time feature:

```toml
[features]
debug_uart = []
```

### 16.3 Bootloader

- USB DFU for firmware updates (nodes with USB)
- ROM bootloader for OTA updates (nodes without USB)
- Secure boot optional via user-installed keys

---

## 17. Configuration File Reference

### 17.1 Complete Example

```toml
# aegis-mesh.toml

[deployment]
building_floor_count = 1
floor_height_m = 2.7
total_area_m2 = 120.0

[policy]
default_mode = "PrivacyFirst"
auto_away_after_minutes = 30
alert_cooldown_s = 60.0

[mesh_network]
protocol = "ble"
edge_controller_address = "192.168.1.100:8080"
heartbeat_interval_ms = 100
message_signing = true

[data_handling]
store_raw_video = false
store_raw_audio = false
store_raw_sensor_data = false
storage_target = "internal"
retention_days = 30
max_storage_mb = 0
include_integrity_chain = true

[companion_app]
enabled = true
api_port = 8080
media_port = 9090
enable_remote_access = false
remote_access_token = ""

[connectivity.cellular]
enabled = false
sim_type = "nano_sim"
apn = ""
fallback_to_wifi = true
always_on_cellular = false
alert_via_cellular = true
stream_via_cellular = false

[identity_node.data]
mode = "metadata_only"
store_raw = false
```

---

## 18. Summary

**AEGIS-MESH firmware implements:**

1. **Sensing-only architecture** — No actuators, no effectors
2. **Mesh-only connectivity for sensor nodes** — No WiFi/Cellular, only mesh radio to edge controller
3. **Edge controller as sole gateway** — All external connectivity concentrated at edge
4. **User-configured data handling** — Metadata-only to full recording, user choice
5. **Hardware privacy controls** — Optional power switches that override all software
6. **Integrity chain** — SHA-256 hashes for legal evidence
7. **Tiered node capabilities** — From minimal choke-point to full 360° camera arrays

**The architecture respects user autonomy:** Data handling, retention, and privacy controls are entirely user-configured without system-imposed restrictions.
