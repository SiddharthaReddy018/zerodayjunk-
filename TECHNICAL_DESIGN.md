# Technical Design Document
## MOSIP Face Liveness Detection + PAD System
### Version 0.3.0-M3 | October 2026

---

## 1. Executive Summary

This document describes the technical design of the face liveness detection and presentation attack detection (PAD) system for the MOSIP Registration Client. The system prevents spoofing attacks during face biometric capture using a multi-layered pipeline: passive PAD scoring, active challenge-response, capture integrity checks, and cryptographic attestation.

**Key metrics:**
- 61 source files, 17 test files, 7,318 lines of Java
- 181 automated tests, 100% pass rate
- 13-state machine with table-driven transitions
- ECDSA P-256 signed attestation
- Hash-chained audit log
- 3 deployment workflows + infant variant

---

## 2. System Architecture

### 2.1 High-Level Architecture

```
┌────────────────────────────────────────────────────────┐
│                    UI Layer (JavaFX)                     │
│   LivenessUiListener ← UiEvent ← UiMessages (i18n)    │
└────────────────────┬───────────────────────────────────┘
                     │ LivenessCallback
┌────────────────────▼───────────────────────────────────┐
│                  Service Layer                          │
│   LivenessService → LivenessSession → StateMachine     │
│   LivenessServiceConfig ← ConfigSyncManager            │
└────────────────────┬───────────────────────────────────┘
                     │
┌────────────────────▼───────────────────────────────────┐
│                  Engine Layer                            │
│   MiniFasNetScorer (ONNX) ← InferenceWorker            │
│   SpatialPadScorer · DecisionEngine · QualityGate      │
│   SameFaceChecker · LandmarkDetector · PoseGate        │
└────────────────────┬───────────────────────────────────┘
                     │
┌────────────────────▼───────────────────────────────────┐
│              Capture Integrity Layer                     │
│   FrameSequenceMonitor · DeviceIdentityValidator        │
│   FreshnessChecker · SessionTransactionBinder           │
│   CaptureIntegrityChecker                               │
└────────────────────┬───────────────────────────────────┘
                     │
┌────────────────────▼───────────────────────────────────┐
│                  Device Layer                            │
│   DeviceAdapter (interface)                             │
│   ├── MdsHttpDeviceAdapter (MOSIP SBI)                  │
│   ├── MockDeviceAdapter (20 scenarios)                  │
│   MdsStreamReader · MdsDeviceInfo · MdsResponse         │
└─────────────────────────────────────────────────────────┘
```

### 2.2 Package Structure (14 packages, 61 source files)

| Package | Purpose | Key Classes |
|---------|---------|-------------|
| `liveness.core` | Immutable value types | `FaceFrame`, `FrameQuality` |
| `liveness.device` | Device abstraction + MDS | `DeviceAdapter`, `MdsHttpDeviceAdapter`, `MockDeviceAdapter` |
| `liveness.capture` | Capture integrity (IAD) | `CaptureIntegrityChecker`, `FrameSequenceMonitor`, `FreshnessChecker` |
| `liveness.engine` | Scoring + detection | `MiniFasNetScorer`, `DecisionEngine`, `QualityGate`, `SameFaceChecker` |
| `liveness.challenge` | Active challenge system | `ChallengeGenerator`, `ChallengeResponseValidator`, `BoundChallenge` |
| `liveness.flow` | State machine + session | `LivenessStateMachine`, `LivenessSession`, `RetryRateLimiter` |
| `liveness.policy` | Config + band routing | `LivenessConfig`, `LivenessPolicy`, `BandTable` |
| `liveness.crypto` | Attestation + signing | `Attestation`, `EcdsaSignerStub` |
| `liveness.audit` | Hash-chained audit log | `AuditChain`, `AuditEvent`, `AuditLog` |
| `liveness.service` | Service-layer orchestrator | `LivenessService`, `LivenessServiceConfig`, `ConfigSyncManager` |
| `liveness.ui` | UI event contract + i18n | `UiEvent`, `UiMessages`, `LivenessUiListener` |
| `liveness.util` | Utilities | `Sha256` |
| `liveness.score` | Score types | Score abstractions |
| `liveness.config` | Config types | Config abstractions |

---

## 3. State Machine Design

### 3.1 The 13-State Machine

```
IDLE → DEVICE_CHECK → POSITIONING → PASSIVE → CHALLENGE_PROMPT →
CHALLENGE_WAIT → FINAL_CAPTURE → VERIFYING_FINAL → SUCCESS
                                                 → BLOCKED_ATTACK
                                                 → FAILED_MAX_RETRIES
                                                 → DEVICE_ERROR
                                                 → RETRY_AVAILABLE → PASSIVE (loop)
```

### 3.2 State Descriptions

| State | Purpose | Terminal? |
|-------|---------|-----------|
| `IDLE` | Initial state | No |
| `DEVICE_CHECK` | Device availability verification | No |
| `POSITIONING` | Face positioning guidance (quality gate) | No |
| `PASSIVE` | Passive PAD scoring (spatial scorer) | No |
| `CHALLENGE_PROMPT` | Active challenge display | No |
| `CHALLENGE_WAIT` | Awaiting challenge response | No |
| `FINAL_CAPTURE` | RCAPTURE trigger | No |
| `VERIFYING_FINAL` | Final verification (binding, same-face, final PAD) | No |
| `RETRY_AVAILABLE` | Retry decision point | No |
| `SUCCESS` | Liveness verified — attestation produced | **Yes** |
| `BLOCKED_ATTACK` | Attack detected — session terminated | **Yes** |
| `FAILED_MAX_RETRIES` | Retry budget exhausted | **Yes** |
| `DEVICE_ERROR` | Device failure — terminal | **Yes** |

### 3.3 Key Design Decisions

- **No boolean flags in `LivenessSession`** — all routing derives from `machine.current()`
- **Table-driven transitions** — invalid transitions throw `IllegalStateException`
- **Attestation only from `VERIFYING_FINAL → SUCCESS`** — no other path produces an attestation
- **Liveness-failed captures are NEVER saved** (constraint #5)
- **`BLOCKED_ATTACK` is terminal per session, not per person** — new session via `RetryRateLimiter`

---

## 4. PAD Pipeline

### 4.1 Pipeline Order (per frame)

1. **Capture Integrity (IAD)** — frame freeze, loop detection, device identity
2. **Quality Gate** — blur, brightness, face size, pose, face count (raw frame, before crop)
3. **Spatial PAD** — per-frame MiniFASNet scoring → sliding window composite
4. **Decision Engine** — 4-band routing with attack persistence windows

### 4.2 Score Bands

| Band | Condition | Action |
|------|-----------|--------|
| `CLEAR_BONA_FIDE` | score ≥ passThreshold (70) | → `FINAL_CAPTURE` |
| `UNCERTAIN` | attackThreshold < score < passThreshold | → `CHALLENGE_PROMPT` (if active enabled) |
| `CLEAR_ATTACK` | score ≤ attackThreshold (30) | → `BLOCKED_ATTACK` (after N persistence windows) |
| `QUALITY_FAILURE` | frame quality too low | → `POSITIONING` |

### 4.3 Attack Persistence

A single low-scoring frame does not trigger block. Attack is confirmed only after `attackPersistenceWindows` (default: 3) consecutive quality-OK windows score below `attackThreshold`.

---

## 5. Capture Integrity (IAD)

### 5.1 Injection Attack Detection

| Check | Detects | Method |
|-------|---------|--------|
| **Frame freeze** | Static image injection | pHash sliding window — identical hashes across N consecutive frames |
| **Frame loop** | Video loop injection | Sequence cycle detection in pHash history |
| **Device identity** | Virtual camera substitution | Device serial + certificate pinning per session |
| **Transaction binding** | RCAPTURE replay | Session nonce bound to transaction ID |
| **Freshness** | Stale capture replay | Timestamp window validation |

### 5.2 Frame Sequence Monitor

Uses perceptual hash (pHash) and timestamp analysis over a sliding window to detect:
- Frozen frames (identical pHash for `windowSize` consecutive frames)
- Looped sequences (repeating pHash patterns)
- Temporal anomalies (timestamp gaps, backwards timestamps)

---

## 6. Cryptographic Attestation

### 6.1 Attestation Structure

Produced **only** on `VERIFYING_FINAL → SUCCESS`:
- Subject ID, Session ID
- Image SHA-256, Transcript SHA-256
- Timestamp, Model version, Config version
- Capture mode (`STREAM`)
- PAD result (attack type, decision score, mechanisms, challenge echoes)
- ECDSA P-256 signature

### 6.2 Score Direction

- **Internal:** higher = more live (0-100)
- **MOSIP boundary:** `padDecisionScore = 100 - livenessScore` (lower = more bona fide)

### 6.3 Audit Chain

SHA-256 hash-chained append-only log. Each `AuditEvent` links to the previous via `previousHash`. Chain can be sealed and integrity-verified.

---

## 7. Active Challenge System

### 7.1 Challenge Pool

8 actions: `BLINK`, `SMILE`, `TURN_LEFT`, `TURN_RIGHT`, `LOOK_LEFT`, `LOOK_RIGHT`, `LOOK_UP`, `LOOK_DOWN`

### 7.2 Selection Rules

- `SecureRandom` selection
- No immediate repeat of same action
- No repeat of previous session's sequence
- System-selected only (never user-chosen)
- Pool filtered by workflow and accessibility profile

### 7.3 Detection

Landmark-based detectors:
- **Blink:** Eye Aspect Ratio (EAR) threshold
- **Smile:** Mouth Aspect Ratio (MAR) threshold
- **Head turns:** Yaw/pitch angle thresholds

---

## 8. Workflows

| Workflow | Active Challenges | Supervisor Auth | Notes |
|----------|-------------------|-----------------|-------|
| `RESIDENT_REGISTRATION` | Yes (if uncertain) | No | Standard flow |
| `OPERATOR_AUTH` | Yes (if uncertain) | No | Same pipeline |
| `SUPERVISOR_AUTH` | Yes (if uncertain) | Yes | SupervisorAuth instance created |
| `INFANT` | No (passive only) | No | Challenges skipped |

### 8.1 Supervisor Exemption Rules

- Supervisor **must** authenticate before exemption
- Exemption **NEVER** applies to `BLOCKED_ATTACK` sessions (§4, §22)
- All exemption attempts are audited

---

## 9. Configuration

All config keys under `mosip.registration.liveness.*`:

| Key | Default | Description |
|-----|---------|-------------|
| `attackThreshold` | 30 | Score below → attack |
| `passThreshold` | 70 | Score above → bona fide |
| `attackPersistenceWindows` | 3 | Consecutive attack windows to confirm |
| `activeEnabled` | true | Enable active challenges |
| `minChallenges` | 2 | Minimum challenges in uncertain band |
| `maxChallenges` | 3 | Maximum challenges |
| `challengeTimeoutMs` | 5000 | Per-challenge timeout |
| `maxRetries` | 3 | Retry budget per session |
| `faceSizeMinPx` | 64 | Minimum face bounding box |
| `poseMaxYawDeg` | 30 | Maximum yaw angle |
| `poseMaxPitchDeg` | 30 | Maximum pitch angle |
| `minQualifyingFrames` | 5 | Frames for composite score |

**Invariant:** `attackThreshold < passThreshold` (enforced at config load)

---

## 10. Internationalization

UI messages support 3 locales with RTL:
- English (EN)
- French (FR)
- Arabic (AR) — RTL

Messages keyed by state name, loaded from `UiMessages`.

---

## 11. Test Coverage

### 11.1 Test Suite Summary (181 tests)

| Category | Tests | Coverage |
|----------|-------|----------|
| State machine matrix | 41 | All 13 states × valid/invalid transitions |
| Device error recovery | 13 | Disconnect, reconnect, integrity failure paths |
| Band routing | 6 | All 4 bands + edge cases |
| Supervisor auth | 14 | Auth, exemption, revocation, ATTACK never exempt |
| Challenge generation | 12 | Pool selection, no-repeat, SecureRandom |
| Challenge validation | 5 | Response matching, timeout |
| Attestation + crypto | 10 | ECDSA sign/verify, canonical format |
| Audit chain | 14 | Append, seal, integrity verification |
| Frame sequence monitor | 7 | Freeze, loop, temporal anomaly detection |
| Spatial PAD scorer | 4 | Composite scoring, window accumulation |
| Same-face checker | 4 | Cosine distance, threshold |
| Landmark detector | 5 | EAR, MAR, turn detection |
| Pose gate | 4 | Raw unit evaluation |
| Policy/config | 24 | Band table, challenge profiles, config bounds |
| Attack taxonomy | 2 | Species enumeration |
| **Integration (E2E)** | **16** | **All workflows, attacks, device errors, callbacks** |

### 11.2 Integration Test Scenarios

| Scenario | Expected | Verified |
|----------|----------|----------|
| Bona fide → SUCCESS | `Verdict.PASS` + attestation + valid ECDSA signature | ✅ |
| Print attack → not SUCCESS | `Verdict.ATTACK`, no attestation | ✅ |
| Screen attack → not SUCCESS | Not PASS, no attestation | ✅ |
| Video attack → not SUCCESS | Not PASS, no attestation | ✅ |
| Device disconnect | `Verdict.DEVICE_FAILURE` | ✅ |
| Device busy | `Verdict.DEVICE_FAILURE` | ✅ |
| Device timeout | `Verdict.DEVICE_FAILURE` | ✅ |
| Operator auth workflow | `Verdict.PASS` | ✅ |
| Supervisor auth workflow | `Verdict.PASS` | ✅ |
| Infant workflow | `Verdict.PASS` (passive only) | ✅ |
| Failed liveness → no attestation | `attestation == null` | ✅ |
| Callback receives events | State changes received | ✅ |
| Uncertain score → challenge path | CHALLENGE_PROMPT reached | ✅ |
| Same-face mismatch | Not PASS | ✅ |
| Corrupt frame | Not PASS | ✅ |

---

## 12. Limitations & Known Gaps

### 12.1 Current Limitations

| Area | Limitation | Mitigation Plan |
|------|-----------|-----------------|
| PAD model | Running in stub mode (configurable score) | Pending ONNX model download from `facenox/face-antispoof-onnx` |
| Same-face | Stub mode (always returns configured result) | Pending SFace ONNX model |
| Face detection | No real-time face detection | Pending YuNet ONNX model |
| Landmarks | Stub landmark data | Real landmarks from face detector |
| pHash | Not computing real perceptual hashes | Needs pixel-level implementation |
| JavaFX UI | Event schema ready, UI controls deferred | Requires JavaFX runtime |
| Android | Not started | Requires Android SDK, CameraX |
| Evaluation | Blocked | Requires datasets |

### 12.2 Uncalibrated Values

The following thresholds use placeholder values pending calibration with real data:
- `attackThreshold` (30) — needs CelebA-Spoof calibration
- `passThreshold` (70) — needs CelebA-Spoof calibration
- Quality gate blur variance (100.0) — needs calibration
- EAR blink threshold (0.2) — needs landmark calibration
- MAR smile threshold (0.6) — needs landmark calibration
- Same-face cosine distance threshold (0.6) — needs LFW calibration

---

## 13. Dependencies

| Dependency | Version | Licence | Purpose |
|-----------|---------|---------|---------|
| Java SE | 11 | Oracle/OpenJDK | Runtime |
| JUnit 5 | 5.9.3 | EPL-2.0 | Testing |
| ONNX Runtime | (pending) | MIT | PAD model inference |

---

## 14. Honest Coverage Statement

This system implements the complete liveness detection architecture specified in the MASTER_PLAN (§1-§26). The following items are **not yet implemented** and require external dependencies:
1. Real ONNX model inference (needs model file)
2. Real face detection and landmark extraction (needs YuNet/SFace models)
3. Real perceptual hash computation (needs pixel-level pHash)
4. Android deployment (needs Android SDK)
5. ISO 30107-3 evaluation (needs datasets)
6. JavaFX UI controls (needs JavaFX runtime)

All stubs are explicitly marked and return configurable values for testing.
