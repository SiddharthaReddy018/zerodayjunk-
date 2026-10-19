# Test Report & Evaluation Results
## MOSIP Face Liveness Detection + PAD
### Version 0.6.0-M4/M5 | October 2026

---

## 1. Summary

| Metric | Value |
|--------|-------|
| **Total tests** | **337** |
| **Passed** | **337** |
| **Failed** | **0** |
| **Errors** | **0** |
| **Skipped** | **0** |
| **Test classes** | **37** |
| **Build time** | ~8.2s |
| **Java target** | OpenJDK 25.0.1 (Temurin), `--release 11` (Immutable constraint §16) |
| **Platform** | macOS (Apple Silicon aarch64) |
| **Evaluation Standard** | ISO/IEC 30107-3:2017 compliant |

---

## 2. Unit & Integration Test Results (337 tests, 37 classes)

| # | Test Class | Package | Tests | Pass | Fail | Time |
|---|-----------|---------|-------|------|------|------|
| 1 | `AttestationTest` | crypto | 10 | 10 | 0 | 0.119s |
| 2 | `FrameSequenceMonitorTest` | capture | 7 | 7 | 0 | 0.005s |
| 3 | `AuditChainIntegrityTest` | audit | 14 | 14 | 0 | 0.023s |
| 4 | `SbiIntentBridgeTest` | android | 29 | 29 | 0 | 0.023s |
| 5 | `LivenessPlatformChannelTest` | android | 16 | 16 | 0 | 0.597s |
| 6 | `EnvironmentIntegrityCheckerTest` | android | 14 | 14 | 0 | 0.007s |
| 7 | `CameraXDeviceAdapterTest` | android | 13 | 13 | 0 | 0.194s |
| 8 | `SbiJwtVerifierTest` | android | 11 | 11 | 0 | 0.157s |
| 9 | `ReferenceFaceEmbedderTest` | android | 3 | 3 | 0 | 0.008s |
| 10 | `Int8ModelBenchmarkTest` | android | 1 | 1 | 0 | 3.545s |
| 11 | `Iso19794FaceTest` | android | 6 | 6 | 0 | 0.010s |
| 12 | `AndroidPipelineIntegrationTest` | android | 5 | 5 | 0 | 0.492s |
| 13 | `LatencyBenchmarkTest` | android | 5 | 5 | 0 | 0.023s |
| 14 | `EvalReportContractTest` | evaluation | 10 | 10 | 0 | 0.018s |
| 15 | `PadMetricsTest` | evaluation | 9 | 9 | 0 | 0.004s |
| 16 | `FairnessAnalyzerTest` | evaluation | 6 | 6 | 0 | 0.007s |
| 17 | `WilsonCITest` | evaluation | 8 | 8 | 0 | 0.004s |
| 18 | `ManifestLoaderTest` | evaluation | 8 | 8 | 0 | 0.006s |
| 19 | `BenchmarkReportRunnerTest` | evaluation | 1 | 1 | 0 | 0.129s |
| 20 | `EvaluationHarnessTest` | evaluation | 5 | 5 | 0 | 0.006s |
| 21 | `AttackTaxonomyTest` | evaluation | 2 | 2 | 0 | 0.000s |
| 22 | `MockDevicePerceptualHashTest` | device | 4 | 4 | 0 | 0.001s |
| 23 | `LivenessServiceIntegrationTest` | service | 16 | 16 | 0 | 0.034s |
| 24 | `ChallengeResponseValidatorTest` | challenge | 5 | 5 | 0 | 0.001s |
| 25 | `ChallengeGeneratorTest` | challenge | 12 | 12 | 0 | 0.007s |
| 26 | `SameFaceCheckerTest` | engine | 4 | 4 | 0 | 0.002s |
| 27 | `PoseGateRawUnitsTest` | engine | 4 | 4 | 0 | 0.002s |
| 28 | `ModelQualificationTest` | engine | 3 | 3 | 0 | 0.094s |
| 29 | `LandmarkDetectorTest` | engine | 5 | 5 | 0 | 0.001s |
| 30 | `SpatialPadScorerTest` | engine | 4 | 4 | 0 | 0.002s |
| 31 | `ChallengeProfileBoundsTest` | policy | 15 | 15 | 0 | 0.006s |
| 32 | `BandTableTest` | policy | 9 | 9 | 0 | 0.002s |
| 33 | `BandRoutingTest` | flow | 6 | 6 | 0 | 0.002s |
| 34 | `SupervisorAuthTest` | flow | 14 | 14 | 0 | 0.002s |
| 35 | `StateMachineMatrixTest` | flow | 41 | 41 | 0 | 0.042s |
| 36 | `DeviceErrorRecoveryTest` | flow | 13 | 13 | 0 | 0.002s |
| 37 | `AuditLog` & `SessionBinder` | audit/capture | 5 | 5 | 0 | 0.010s |


---

## 3. ISO/IEC 30107-3 Benchmark Evaluation Results

The evaluation harness ran on the multi-split benchmark manifest derived from the extracted **ONOT** synthetic dataset (179 subjects, 5,811 photos) and attack presentations.

- **Decision Threshold:** 70 (score 0–100, higher = more live internally)
- **Total Samples Evaluated in Test Split:** 3,359
- **Bona Fide Subjects:** 99 (**Rule of 30 satisfied: $N \ge 30$)**

### 3.1 Per-Species APCER (Attack Presentation Classification Error Rate)

| PAI Species | APCER | N (attacks) | 95% Wilson CI |
|---|---|---|---|
| `curved_print` | 0.0000 | 63 | [0.0000, 0.0287, 0.0575] |
| `cutout` | 0.0000 | 64 | [0.0000, 0.0283, 0.0566] |
| `print` | 0.0000 | 56 | [0.0000, 0.0321, 0.0642] |
| `screen` | 0.0000 | 54 | [0.0000, 0.0332, 0.0664] |
| `video` | 0.0000 | 63 | [0.0000, 0.0287, 0.0575] |

**Worst-Species Headline APCER (R6):** `screen` = 0.0000 [0.0000, 0.0332, 0.0664]  
*(Note: Per ISO/IEC 30107-3 and project constraint R6, combined ACER is strictly forbidden and not reported).*

### 3.2 BPCER (Bona Fide Presentation Classification Error Rate)

- **BPCER:** 0.0000 ($N = 3,059$ presentations, 99 unique subjects)
- **95% Wilson CI:** [0.0000, 0.0006, 0.0013]

### 3.3 Operating Curves (D-EER & DET)

- **D-EER (Detection Equal Error Rate):** ~0.0000 at operating point threshold = 55.

---

## 4. Injection Attack Detection (IAD) Metrics

Reported in a separate table per CEN/TS 18099 concepts (§12.2, not reported as APCER):

| Attack Vector | Attempts | Detected / Rejected | Detection Rate |
|---|---|---|---|
| Freeze Detection (pHash circular buffer) | 3 | 3 | **100.0%** |
| Frame Loop Replay Detection | 2 | 2 | **100.0%** |
| Nonce-Swap Rejection (Session Binder) | 1 | 1 | **100.0%** |
| False IAD Rate (Legitimate sessions) | 2 | 0 | **0.0%** |

---

## 5. Fairness & Demographic Analysis (§12.4, §24.4, R7)

- **Bias Index:** `0.000000` (lower = more fair, DFIC paper formula)
- **Scale:** Monk Skin Tone (MST 10-tone scale)

| Category | Group | BPCER | N | Subjects | 95% Wilson CI | Status |
|---|---|---|---|---|---|---|
| **Monk Tone** | `mst_2` | 0.0000 | 1,321 | 47 | [0.0000, 0.0014, 0.0029] | Conclusive ($\ge 30$) |
| **Monk Tone** | `mst_4` | 0.0000 | 438 | 10 | [0.0000, 0.0043, 0.0087] | Exploratory ($< 30$) |
| **Monk Tone** | `mst_5` | 0.0000 | 614 | 20 | [0.0000, 0.0031, 0.0062] | Exploratory ($< 30$) |
| **Monk Tone** | `mst_6` | 0.0000 | 217 | 7 | [0.0000, 0.0087, 0.0174] | Exploratory ($< 30$) |
| **Monk Tone** | `mst_9` | 0.0000 | 469 | 15 | [0.0000, 0.0041, 0.0081] | Exploratory ($< 30$) |
| **Gender** | `female` | 0.0000 | 1,624 | 61 | [0.0000, 0.0012, 0.0024] | Conclusive ($\ge 30$) |
| **Gender** | `male` | 0.0000 | 1,435 | 38 | [0.0000, 0.0013, 0.0027] | Conclusive ($\ge 30$) |
| **Age** | `50+` | 0.0000 | 1,572 | 46 | [0.0000, 0.0012, 0.0024] | Conclusive ($\ge 30$) |
| **Age** | `40-50` | 0.0000 | 469 | 18 | [0.0000, 0.0041, 0.0081] | Exploratory ($< 30$) |
| **Age** | `30-40` | 0.0000 | 398 | 14 | [0.0000, 0.0048, 0.0096] | Exploratory ($< 30$) |
| **Age** | `20-30` | 0.0000 | 396 | 14 | [0.0000, 0.0048, 0.0096] | Exploratory ($< 30$) |
| **Age** | `0-20` | 0.0000 | 224 | 7 | [0.0000, 0.0084, 0.0169] | Exploratory ($< 30$) |

*Flags:* No group exhibited a BPCER swing $> \pm 5\text{ pp}$.

---

## 6. Honest Coverage Statement (§15.5)

| Threat | Treatment | What We Claim |
|---|---|---|
| **Printed photo (flat)** | **Primary target** | Targeted detection with APCER measured across artefact intensities |
| **Screen display (static/video)** | **Primary target** | Targeted detection |
| **Video replay** | **Primary target** | Targeted detection |
| **Cut-out mask** | **Secondary target** | Increased resistance; detection under favourable conditions |
| **Curved print** | **Secondary target** | Increased resistance; detection under favourable conditions |
| **2D rigid mask** | **Secondary target** | Increased resistance; detection under favourable conditions |
| **3D silicone/resin mask** | **Experimental resistance** | Multi-cue analysis; no standalone detection claim |
| **Real-time deepfake** | **Mitigation only** | Challenge-response binding + latency + temporal analysis; no detection guarantee |
| **Injection attacks** | **Architectural control (IAD)** | CaptureIntegrity module per CEN/TS 18099 concepts |
| **Frozen frame** | **Detection** | High-reliability via frame sequence monitor |
| **Frame loop replay** | **Detection** | Reliable for non-adaptive replay |
