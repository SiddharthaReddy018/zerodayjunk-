# Integration Guide
## MOSIP Face Liveness Detection + PAD
### Version 0.3.0-M3

---

## 1. Overview

This guide explains how to integrate the face liveness detection system into the MOSIP Registration Client. The system plugs into the biometric capture pipeline at the **service layer** that creates the biometric record.

---

## 2. Prerequisites

- **Java 11+** runtime
- **Maven 3.9+** for building
- MOSIP Registration Client with SBI/MDS device support

### Build

```bash
mvn clean package -DskipTests
# Output: target/face-liveness-pad-0.3.0-M3.jar
```

---

## 3. Quick Start

### 3.1 Minimal Integration (5 lines)

```java
import liveness.service.LivenessService;
import liveness.service.LivenessService.*;
import liveness.device.MdsHttpDeviceAdapter;
import liveness.policy.LivenessConfig;
import liveness.engine.*;
import liveness.challenge.ChallengeGenerator;
import liveness.flow.RetryRateLimiter;
import liveness.crypto.EcdsaSignerStub;

// 1. Create config (use defaults or load from server)
LivenessConfig config = LivenessConfig.defaults();

// 2. Create components
MiniFasNetScorer padScorer = new MiniFasNetScorer(modelPath);  // null for stub
SameFaceChecker sameFaceChecker = new SameFaceChecker(0.6);
ChallengeGenerator challengeGen = new ChallengeGenerator(
        ChallengeGenerator.DEFAULT_POOL, Collections.emptyList());
RetryRateLimiter limiter = new RetryRateLimiter(config.maxRetries());
EcdsaSignerStub signer = new EcdsaSignerStub();

// 3. Create service
LivenessService service = new LivenessService(
        config, padScorer, sameFaceChecker, challengeGen, limiter, signer);

// 4. Create device adapter
MdsHttpDeviceAdapter device = new MdsHttpDeviceAdapter("http://127.0.0.1:4501");

// 5. Run liveness check (MUST be called from a background thread)
LivenessResult result = service.performLiveness(
        subjectId, device, machineUserId,
        WorkflowType.RESIDENT_REGISTRATION, callback);

if (result.isSuccess()) {
    Attestation attestation = result.attestation();
    // Proceed with biometric capture
} else {
    // Handle failure: result.verdict(), result.failureReason()
}
```

### 3.2 With UI Callbacks

```java
LivenessCallback callback = new LivenessCallback() {
    @Override
    public void onStateChanged(String state, String message) {
        Platform.runLater(() -> statusLabel.setText(message));
    }

    @Override
    public void onPreviewFrame(byte[] jpegBytes) {
        Platform.runLater(() -> updatePreviewImage(jpegBytes));
    }

    @Override
    public void onChallengePrompt(String action, long timeoutMs) {
        Platform.runLater(() -> showChallengePrompt(action, timeoutMs));
    }

    @Override
    public void onProgress(int framesProcessed, int framesNeeded) {
        Platform.runLater(() -> progressBar.setProgress(
                (double) framesProcessed / framesNeeded));
    }

    @Override
    public void onResult(LivenessService.LivenessResult result) {
        Platform.runLater(() -> showResult(result));
    }

    @Override
    public void onError(String errorCode, String message) {
        Platform.runLater(() -> showError(errorCode, message));
    }
};

// Run on background thread — NEVER on UI thread
executorService.submit(() ->
    service.performLiveness(subjectId, device, machineUserId,
            WorkflowType.RESIDENT_REGISTRATION, callback));
```

---

## 4. Workflow Types

| Workflow | When to use | Active challenges? |
|----------|------------|-------------------|
| `RESIDENT_REGISTRATION` | Enrolling a new resident | Yes (if uncertain band) |
| `OPERATOR_AUTH` | Operator login | Yes (if uncertain band) |
| `SUPERVISOR_AUTH` | Supervisor login + exemptions | Yes (if uncertain band) |
| `INFANT` | Child registration | No (passive only) |

---

## 5. Configuration

### 5.1 Default Configuration

```java
LivenessConfig config = LivenessConfig.defaults();
// attackThreshold=30, passThreshold=70, maxRetries=3,
// activeEnabled=true, minChallenges=2, challengeTimeoutMs=5000
```

### 5.2 Custom Configuration

```java
LivenessConfig config = LivenessConfig.builder()
    .attackThreshold(25)
    .passThreshold(75)
    .maxRetries(5)
    .activeEnabled(true)
    .minChallenges(2)
    .maxChallenges(3)
    .challengeTimeoutMs(8000)     // 8s for accessibility
    .faceSizeMinPx(80)
    .poseMaxYawDeg(25)
    .poseMaxPitchDeg(25)
    .minQualifyingFrames(5)
    .build();
```

### 5.3 Server-Synced Configuration

```java
ConfigSyncManager syncManager = new ConfigSyncManager(
    "https://config-server/api/liveness",
    serviceConfig,
    300_000L  // sync every 5 minutes
);
syncManager.startSync();
```

---

## 6. Device Adapters

### 6.1 MDS/SBI Device (Production)

```java
// Discover device
MdsHttpDeviceAdapter device = new MdsHttpDeviceAdapter("http://127.0.0.1:4501");
// Device will be opened/closed by LivenessService
```

### 6.2 Mock Device (Testing)

```java
MockDeviceAdapter device = new MockDeviceAdapter(MockDeviceAdapter.Scenario.BONA_FIDE);
// Available scenarios: BONA_FIDE, PRINT, SCREEN, VIDEO, CUTOUT,
// CURVED_PRINT, MASK_2D, MASK_3D, DEEPFAKE, FRAME_FREEZE,
// FRAME_LOOP, VIRTUAL_CAMERA, DISCONNECT, CORRUPT_FRAME,
// LOW_FPS, BUSY, TIMEOUT, LOW_LIGHT, MULTIPLE_FACES, NO_FACE
```

---

## 7. Result Handling

### 7.1 Success Path

```java
if (result.isSuccess()) {
    Attestation att = result.attestation();
    String imageSha256 = att.imageSha256();
    byte[] signature = att.signature();
    // Include attestation in biometric record
}
```

### 7.2 Failure Handling

```java
switch (result.verdict()) {
    case PASS:         // Success (handled above)
    case ATTACK:       // Presentation attack detected — do NOT save capture
    case MAX_RETRIES:  // Retry budget exhausted — prompt operator
    case DEVICE_FAILURE: // Device error — check hardware
    case PENDING:      // Should not occur in result
}
```

### 7.3 Critical Rule

> **Liveness-failed captures are NEVER saved.** The `LivenessResult` enforces this:
> `attestation` is `null` for any non-PASS verdict.

---

## 8. Threading Model

```
UI Thread                    Background Thread
    │                              │
    │  submit(performLiveness)  →  │
    │                              ├── device.open()
    │  ← onStateChanged()         ├── frame loop
    │  ← onPreviewFrame()         │   ├── captureFrame()
    │  ← onChallengePrompt()      │   ├── processFrame()
    │  ← onProgress()             │   └── (repeat)
    │                              ├── RCAPTURE
    │  ← onResult()               ├── verification
    │                              └── attestation
```

- **NEVER** call `performLiveness()` on the UI thread
- All `LivenessCallback` methods are called from the background thread
- Use `Platform.runLater()` (JavaFX) or `runOnUiThread()` (Android) in callbacks

---

## 9. Internationalization

```java
// Get localized message for a state
String msg = UiMessages.get("POSITIONING");  // "Position your face in the oval"

// Set locale
UiMessages.setLocale("fr");  // French
UiMessages.setLocale("ar");  // Arabic (RTL)
```

Supported locales: `en` (default), `fr`, `ar`

---

## 10. Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| `DEVICE_FAILURE` on startup | MDS not running or wrong port | Check MDS service, verify port |
| `DEVICE_FAILURE` during capture | Camera disconnected | Reconnect camera, retry |
| `ATTACK` on real face | PAD threshold too aggressive | Lower `attackThreshold` (needs calibration) |
| All captures fail quality | Poor lighting or distance | Check `faceSizeMinPx`, improve lighting |
| Challenge timeout | Subject too slow | Increase `challengeTimeoutMs`, use ACCESSIBILITY profile |
