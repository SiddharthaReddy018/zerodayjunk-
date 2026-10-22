# Audit Remediation -- Part 2 Items (Requiring External Hardware/Data)

This document tracks items from Part 2 of the codebase audit that cannot be completed purely in software within the offline desktop environment. These items require physical mobile devices, physical biometric hardware, production trust anchors, live subjects, or external institutional access.

---

## Part 2 Item Inventory

### [NOT MEASURED - Part 2 X1] Full Offline/Online Network & Coverage Certification
- **Status:** Open
- **Description:** Complete independent verification of build artifacts on network-isolated environments and external CI/CD pipeline certification.

### [NOT MEASURED - Part 2 X2] Model-Dependent End-to-End Field Qualification
- **Status:** Open
- **Description:** Final certification of ONNX model binaries on target hardware architectures (ARM64-v8a, armeabi-v7a) with NPU hardware acceleration delegates.

### [NOT MEASURED - Part 2 X4] Android / Flutter Production Build & Engine Hash Verification
- **Status:** Open
- **Description:** Full release build assembly with real Android SDK/NDK, Flutter toolchain, ProGuard/R8 shrinking rules, and MOSIP release signing keys.

### [NOT MEASURED - Part 2 X5] SFace Calibration on Real Cross-Stream Mobile Pairs
- **Status:** Open
- **Description:** Collecting and calibrating bona fide genuine and impostor cross-stream image pairs where host preview frames (CameraX) are compared directly with physical SBI device output (`rCapture`) across target demographics.

### [NOT MEASURED - Part 2 X6] Physical Phone Latency Benchmarks
- **Status:** Open
- **Description:** Real-world execution timing on physical target Android smartphones (e.g. low-end quad-core ARM devices) under thermal throttling and battery saver constraints.

### [NOT MEASURED - Part 2 X7] Clock Skew & NTP Verification Under Device Drift
- **Status:** Open
- **Description:** Rigorous testing of token expiry and freshness windows on mobile devices experiencing extreme network time drift or misconfigured system clocks.

### [NOT MEASURED - Part 2 X8] Android KeyStore Hardware Attestation on Physical Device
- **Status:** Open
- **Description:** Verifying `AndroidKeyStoreSigner.ensureKey()` and hardware-backed key attestation certificates on real devices equipped with StrongBox Keymaster or TEE (Trusted Execution Environment).

### [NOT MEASURED - Part 2 X9] Physical Penetration Testing on Rooted/Frida Devices
- **Status:** Open
- **Description:** Validating `EnvironmentIntegrityChecker` against active in-memory hooks (Frida, LSPosed, Magisk Zygisk modules) executing directly on compromised physical handsets.

### [NOT MEASURED - Part 2 X10] Production Face Localisation Model Replacement
- **Status:** Open
- **Description:** Upgrading face bounding box detector to production lightweight neural model (e.g. YuNet / SCRFD) on mobile device pipeline.

### [NOT MEASURED - Part 2 X11] Real SBI / MDS Hardware & JPEG 2000 Native Decoder Integration
- **Status:** Open
- **Description:** Integration with physical MOSIP L0/L1 biometric scanners via USB OTG / BLE and native OpenJPEG JNI libraries for 19794-5 JPEG 2000 decoding.

### [NOT MEASURED - Part 2 X12] Live Demographically Diverse Population Data Collection
- **Status:** Open
- **Description:** Large-scale field data collection of physical presentation attacks (3D latex/silicone masks, makeup, wax heads) and live subjects per ISO/IEC 19795-1 Rule of 30 across all Monk skin tones, ages, and lighting profiles.

### [NOT MEASURED - Part 2 X13] Real-Camera Frame Freeze & Loop Replay Cadence Tuning
- **Status:** Open
- **Description:** Fine-tuning `FrameSequenceMonitor` distance windows and cadence thresholds against real mobile CMOS camera noise profiles and auto-exposure/auto-focus hunting.

### [NOT MEASURED - Part 2 X14] Formal Legal & License Conformance Clearance
- **Status:** Open
- **Description:** Final institutional legal review of dataset and model licenses (ONOT synthetic ICAO, OpenCV Zoo Apache-2.0 SFace/YuNet, facenox MiniFASNetV2-SE).

### [NOT MEASURED - Part 2 X15] Independent Third-Party Laboratory Review
- **Status:** Open
- **Description:** Conformance and security evaluation conducted by an accredited independent testing laboratory (FIDO / iBeta / BSI) for ISO/IEC 30107-3 PAD compliance.
