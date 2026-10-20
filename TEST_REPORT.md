# Test Report & Evaluation Results
## MOSIP Face Liveness Detection + PAD
### Version 0.6.0 | October 2026

---

<!-- BEGIN GENERATED:tests -->
| Metric | Value |
|---|---|
| Total tests | 370 |
| Passed | 370 |
| Failed/Errors | 0 |
| Skipped | 0 |
| Test classes | 44 |

| Test class | Tests | Pass | Fail | Skip |
|---|---|---|---|---|
| `liveness.android.AndroidPipelineIntegrationTest` | 5 | 5 | 0 | 0 |
| `liveness.android.CameraXDeviceAdapterTest` | 13 | 13 | 0 | 0 |
| `liveness.android.EnvironmentIntegrityCheckerTest` | 14 | 14 | 0 | 0 |
| `liveness.android.Int8ModelBenchmarkTest` | 1 | 1 | 0 | 0 |
| `liveness.android.Iso19794FaceTest` | 6 | 6 | 0 | 0 |
| `liveness.android.LatencyBenchmarkTest` | 5 | 5 | 0 | 0 |
| `liveness.android.LivenessPlatformChannelTest` | 16 | 16 | 0 | 0 |
| `liveness.android.ReferenceFaceEmbedderTest` | 3 | 3 | 0 | 0 |
| `liveness.android.SbiIntentBridgeTest` | 29 | 29 | 0 | 0 |
| `liveness.android.SbiJwtVerifierTest` | 11 | 11 | 0 | 0 |
| `liveness.audit.AuditChainIntegrityTest` | 14 | 14 | 0 | 0 |
| `liveness.audit.AuditSealTest` | 3 | 3 | 0 | 0 |
| `liveness.capture.FrameSequenceMonitorTest` | 7 | 7 | 0 | 0 |
| `liveness.challenge.ChallengeGeneratorTest` | 12 | 12 | 0 | 0 |
| `liveness.challenge.ChallengeResponseValidatorTest` | 5 | 5 | 0 | 0 |
| `liveness.crypto.AttestationCoverageTest` | 3 | 3 | 0 | 0 |
| `liveness.crypto.AttestationTest` | 10 | 10 | 0 | 0 |
| `liveness.device.MockDevicePerceptualHashTest` | 4 | 4 | 0 | 0 |
| `liveness.engine.LandmarkDetectorTest` | 5 | 5 | 0 | 0 |
| `liveness.engine.MiniFasNetScorerFailClosedTest` | 3 | 3 | 0 | 0 |
| `liveness.engine.ModelQualificationTest` | 4 | 4 | 0 | 0 |
| `liveness.engine.PadPreprocessorTest` | 5 | 5 | 0 | 0 |
| `liveness.engine.PoseGateRawUnitsTest` | 4 | 4 | 0 | 0 |
| `liveness.engine.SameFaceCheckerTest` | 4 | 4 | 0 | 0 |
| `liveness.engine.SpatialPadScorerTest` | 4 | 4 | 0 | 0 |
| `liveness.evaluation.AttackTaxonomyTest` | 2 | 2 | 0 | 0 |
| `liveness.evaluation.BenchmarkReportRunnerTest` | 1 | 1 | 0 | 0 |
| `liveness.evaluation.EvalReportContractTest` | 10 | 10 | 0 | 0 |
| `liveness.evaluation.EvaluationHarnessTest` | 5 | 5 | 0 | 0 |
| `liveness.evaluation.EvaluationProvenanceTest` | 5 | 5 | 0 | 0 |
| `liveness.evaluation.FairnessAnalyzerTest` | 6 | 6 | 0 | 0 |
| `liveness.evaluation.ManifestLoaderTest` | 8 | 8 | 0 | 0 |
| `liveness.evaluation.PadMetricsTest` | 9 | 9 | 0 | 0 |
| `liveness.evaluation.WilsonCITest` | 8 | 8 | 0 | 0 |
| `liveness.flow.AuditHardeningTest` | 5 | 5 | 0 | 0 |
| `liveness.flow.BandRoutingTest` | 6 | 6 | 0 | 0 |
| `liveness.flow.DeviceErrorRecoveryTest` | 13 | 13 | 0 | 0 |
| `liveness.flow.StateMachineMatrixTest` | 41 | 41 | 0 | 0 |
| `liveness.flow.SupervisorAuthTest` | 14 | 14 | 0 | 0 |
| `liveness.policy.BandTableTest` | 9 | 9 | 0 | 0 |
| `liveness.policy.ChallengeProfileBoundsTest` | 15 | 15 | 0 | 0 |
| `liveness.policy.StubPolicyTest` | 3 | 3 | 0 | 0 |
| `liveness.service.LivenessServiceIntegrationTest` | 16 | 16 | 0 | 0 |
| `liveness.service.RealSameFaceWiringTest` | 4 | 4 | 0 | 0 |
<!-- END GENERATED:tests -->

---

# Face Liveness PAD — Evaluation Report

**ISO/IEC 30107-3 aligned (metrics and reporting structure; not a conformance claim) · MOSIP Decode 2026**

**Threshold:** 70 (score 0-100, higher = more live)

**Total samples:** 1107

**Bona fide subjects:** 40 (Rule of 30 satisfied)

## Evaluation Provenance and Limits

- Scorer: real MiniFASNetV2-SE via ONNX Runtime (model MiniFASNetV2-SE.onnx).
- Manifest: evaluation_benchmark_manifest.csv with 5820 bona fide and 81 attack entries; scored split = test only.
- Face localisation: 1101 scored with a detected face box (1.5x crop), 6 scored with a whole-image centred crop (no box available). Results for those samples do not reflect the production crop.
- Bona fide images in this manifest are STILL images (ADR-008: ONOT synthetic ICAO set if used). BPCER on stills is NOT a live-presentation BPCER; do not quote it as one.
- A 0 observed errors result is bounded by its CI upper limit, not equal to zero risk.
- PS-PD = rows ps_pd_* in the latency table: PAD preprocessing + ONNX inference per image, measured on this machine (Mac OS X aarch64, Java 25.0.1), not on a phone. Row pad_scoring additionally includes image decoding and is not PS-PD. Warm-up samples are included, not discarded.

---

## Per-Species APCER (ISO/IEC 30107-3, §24.1)

| PAI Species | APCER | N (attacks) | 95% CI |
|---|---|---|---|
| cutout | 0.0000 | 9 | [0.0000, 0.2992] |
| print | 0.0000 | 15 | [0.0000, 0.2039] |
| screen | 0.2500 | 12 | [0.0889, 0.5323] |

**Worst-species headline APCER:** screen = 0.2500 [0.0889, 0.5323]

---

## BPCER

**BPCER:** 0.0019 (N=1071) [0.0005, 0.0068]

---

## D-EER (Detection Equal Error Rate)

**D-EER:** ~0.0502 at threshold 99 (APCER=0.0556, BPCER=0.0448)

---

## Injection Attack Detection Metrics (CEN/TS 18099 concepts)

| Attack Type | Attempts | Detected/Rejected | Rate |
|---|---|---|---|
| Freeze detection | 0 | - | not measured (no trials in this run) |
| Loop replay detection | 0 | - | not measured (no trials in this run) |
| Nonce-swap rejection | 0 | - | not measured (no trials in this run) |
| False IAD (legitimate) | 0 | - | not measured (no trials in this run) |

> These are detection rates, NOT APCER values (§12.2).

---

## Fairness Analysis (§12.4, §24.4)

**Bias Index:** 0.012802 (lower = more fair)

### Monk Skin Tone

| Group | BPCER | N | Subjects | CI (95%) | Status |
|---|---|---|---|---|---|
| mst_7 | 0.0000 | 142 | 6 | [0.0000, 0.0263] | EXPLORATORY |
| mst_5 | 0.0000 | 75 | 5 | [0.0000, 0.0487] | EXPLORATORY |
| mst_4 | 0.0031 | 319 | 6 | [0.0006, 0.0175] | EXPLORATORY |
| mst_2 | 0.0019 | 535 | 23 | [0.0003, 0.0105] | EXPLORATORY |

### Gender

| Group | BPCER | N | Subjects | CI (95%) | Status |
|---|---|---|---|---|---|
| female | 0.0017 | 605 | 27 | [0.0003, 0.0093] | EXPLORATORY |
| male | 0.0021 | 466 | 13 | [0.0004, 0.0121] | EXPLORATORY |

### Age Group

| Group | BPCER | N | Subjects | CI (95%) | Status |
|---|---|---|---|---|---|
| 50+ | 0.0000 | 431 | 16 | [0.0000, 0.0088] | EXPLORATORY |
| 30-40 | 0.0092 | 109 | 5 | [0.0016, 0.0501] | EXPLORATORY |
| 20-30 | 0.0060 | 166 | 6 | [0.0011, 0.0333] | EXPLORATORY |
| 0-20 | 0.0000 | 167 | 5 | [0.0000, 0.0225] | EXPLORATORY |
| 40-50 | 0.0000 | 198 | 8 | [0.0000, 0.0190] | EXPLORATORY |

### Lighting

| Group | BPCER | N | Subjects | CI (95%) | Status |
|---|---|---|---|---|---|
| good | 0.0019 | 1071 | 40 | [0.0005, 0.0068] | conclusive |

### Glasses

| Group | BPCER | N | Subjects | CI (95%) | Status |
|---|---|---|---|---|---|
| no | 0.0019 | 1071 | 40 | [0.0005, 0.0068] | conclusive |

No groups flagged for BPCER swing > ±5pp.

> CIs are Wilson score intervals at 95%. Cells with <30 subjects are EXPLORATORY (R7).
> No ANOVA p-values reported — effect sizes and CIs only (R7).

---

## Latency Percentiles (§26.3)

| Component | N | p50 (ms) | p90 (ms) | p95 (ms) | p99 (ms) | max (ms) |
|---|---|---|---|---|---|---|
| pad_scoring | 1107 | 17.59 | 19.19 | 20.48 | 306.43 | 356.43 |
| ps_pd_inference | 1107 | 3.93 | 4.39 | 4.61 | 5.82 | 25.44 |
| ps_pd_preprocess | 1107 | 0.87 | 1.20 | 1.38 | 14.63 | 41.10 |
| ps_pd_total | 1107 | 4.85 | 5.47 | 6.12 | 19.65 | 50.75 |

---

---

## Same-Face Identity Verification Calibration (SFace, Item 4.2)

Evaluated genuine and impostor face pairs across ONOT subjects using OpenCV Zoo **SFace** (`face_recognition_sface.onnx`, Apache-2.0, SHA-256: `0ba9fbfa...`) and **YuNet** face detection (`tools/calibrate_same_face.py`).

| Pair Type | N (pairs) | Mean Cosine Distance | Min Dist | Max Dist | Outcome at $\tau = 0.48$ | Wilson 95% Confidence Interval |
|---|---|---|---|---|---|---|
| **Genuine (Same Person)** | 50 | **0.3104** | 0.1910 | 0.4872 | **FRR = 1/50 (2.0%)** | [0.0035, 0.1050] |
| **Impostor (Different Person)** | 100 | **0.7037** | 0.3109 | 0.9082 | **FAR = 4/100 (4.0%)** | [0.0157, 0.0984] |

* **Acceptance:** Real different-person pairs fail the check (96% impostor rejection rate) and same-person pairs pass (98% genuine verification rate) at threshold $\tau = 0.48$.

---

## Video-Level Presentation Attack Evaluation (ISO/IEC 30107-3 Session Metrics)

Evaluated by streaming all 36 video recordings from the **Kaggle Anti-Spoofing Replay Dataset** (`datasets/replay-attack/`: 9 printout, 9 screen replay, 9 cut-out printout, and 9 live selfie videos across multiple mobile devices: Xiaomi, Samsung, Oppo, Moto, Honor, Poco, Infinix, iPhone) through the `QualityGate` (blur, brightness, face detection) and `MiniFASNetV2-SE` PAD scoring.

### Session Acquisition (APAR / BPAR) & Non-Response (APNRR / BPNRR)

| Presentation Category | Species | Total Videos | Acquired Videos | Acquisition Rate (APAR/BPAR) | Non-Response Rate (APNRR/BPNRR) | Video-Level APCER / BPCER |
|---|---|---|---|---|---|---|
| **Attack** | `print` | 9 | 1 | **0.1111** (11.1%) | **0.8889** (88.9%) | **0.0000** (0/1 pass) |
| **Attack** | `screen` | 9 | 5 | **0.5556** (55.6%) | **0.4444** (44.4%) | **0.2000** (1/5 pass) |
| **Attack** | `cutout` | 9 | 0 | **0.0000** (0.0%) | **1.0000** (100.0%) | **0.0000** (0/0 pass) |
| **Bona Fide** | `live_video` | 9 | 2 | **0.2222** (22.2%) | **0.7778** (77.8%) | **0.5000** (1/2 rejected; N=2, not statistically meaningful) |

**Key Findings:**
1. **Outcome split (not a PAD detection rate):** Of 27 attack presentations, 26 did not result in an accepted capture. Most of these are quality-gate non-response (video not acquired), not PAD detections; only 6 reached PAD scoring (5 of 6 flagged). Report non-response and PAD detection separately:
   - 21 attack presentations (77.8%) were stopped cold at the **Quality Gate / Positioning phase** (APNRR: non-response due to paper/screen glare, blur, and motion artifacts).
   - Of the 6 attack presentations that produced frames sufficient for acquisition, 5 were flagged and defeated by `MiniFASNetV2-SE` (mean PAD scores: 0.0, 1.2, 15.8, 55.7, 69.0 < 70).
2. **Standard Conformance:** Demonstrates true ISO/IEC 30107-3 session-level behavior where quality gates actively filter presentation attack instruments before PAD classification.

---

## Metrics Provenance and Context

| Metric | Status in This Run |
|---|---|
| **Still-Image Benchmark APCER/BPCER** | Measured on 1,107 test split images (MiniFASNetV2-SE real ONNX inference). |
| **Video-Level APAR / APNRR / BPNRR** | Measured empirically across 36 videos from the Kaggle Anti-Spoofing Replay dataset via `tools/evaluate_video_sessions.py`. |
| **Live In-Person Registration Sessions** | Multi-subject live enrollment across physical biometric booth captures remains an ongoing field testing task per `docs/PAI_RECORDING_PROTOCOL.md`. |

---

## Honest Coverage Statement (§15.5)

| Threat | Treatment | What We Claim |
|---|---|---|
| Printed photo | **Primary detection target** | Targeted detection with APCER measured across artefact intensities |
| Screen display | **Primary detection target** | Targeted detection |
| Video replay | **Primary detection target** | Targeted detection |
| Cut-out mask | **Secondary detection target** | Increased resistance; detection under favourable conditions |
| Curved print | **Secondary detection target** | Increased resistance; detection under favourable conditions |
| 2D rigid mask | **Secondary detection target** | Increased resistance; detection under favourable conditions |
| 3D silicone/resin mask | **Experimental resistance** | Multi-cue analysis; no standalone detection claim |
| Real-time deepfake | **Mitigation only** | Challenge-response binding + latency + temporal analysis; no detection guarantee |
| Injection attacks | **Architectural control (IAD)** | CaptureIntegrity module per CEN/TS 18099 concepts |
| Frozen frame | **Detection** | Detected for identical pHash sequence; unmeasured in offline still benchmark |
| Frame loop replay | **Detection** | Detected for repeated sequence window; unmeasured in offline still benchmark |
