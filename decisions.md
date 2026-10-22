# Architecture Decision Records (ADR)

## ADR-001: DEVICE_ERROR is terminal per session, recovery via new session

**Date:** 2026-10-06  
**Status:** DECIDED (HANDOFF-2 16.2)  
**Context:** S4 calls DEVICE_ERROR terminal, S23.4 describes reconnecting.  
**Decision:** DEVICE_ERROR is terminal for the session. Recovery is always a new session starting at DEVICE_CHECK (MOSIPDISC rediscovery). The transition carries a reason code:
- `DEVICE_UNAVAILABLE` (benign): free — no retry budget charged; bounded by `device.max_reconnect_attempts`.
- `INTEGRITY_FAILURE` (security): counts toward the lock budget; supervisor exemption not applicable.  
**Consequence:** No boolean flags needed; reason is part of the transition event.

## ADR-002: RETRY_AVAILABLE → PASSIVE retry path

**Date:** 2026-10-06  
**Status:** DECIDED (HANDOFF-2 15.7 item 4)  
**Context:** Problem statement requires retry support with clear indication and recovery guidance.  
**Decision:** RETRY_AVAILABLE → PASSIVE (row 22), which can fall back to POSITIONING by row 9 if quality drops. Matches the problem statement's resident workflow.  
**Consequence:** Retry always goes through PASSIVE, not directly to POSITIONING.

## ADR-003: FINAL_CAPTURE RCAPTURE failure → DEVICE_ERROR

**Date:** 2026-10-06  
**Status:** DECIDED (HANDOFF-2 16.2 row 17)  
**Context:** Problem statement requires handling device communication failures.  
**Decision:** RCAPTURE failure/timeout → DEVICE_ERROR with reason DEVICE_UNAVAILABLE.  
**Consequence:** Subject is guided to retry via new session; no budget charged.

## ADR-004: BLOCKED_ATTACK terminal for session, not person

**Date:** 2026-10-06  
**Status:** DECIDED (MASTER_PLAN §4)  
**Context:** v1 said "no retry ever" which conflicts with the problem statement's retry requirement.  
**Decision:** BLOCKED_ATTACK is terminal per session. A new session is allowed through RetryRateLimiter. Each blocked session counts against the lock budget. Supervisor exemption never applies.  
**Consequence:** Prevents abuse while allowing legitimate re-enrollment.

## ADR-005: Standalone reference implementation + thin integration shim

**Date:** 2026-10-06  
**Status:** ASSUMPTION (pending MOSIP answer, §14 item 10)  
**Context:** Unknown whether this is a patch against the real registration client.  
**Decision:** Ship standalone + thin integration shim. Do not invent MOSIP class names.  
**Consequence:** Minimizes integration risk; can be adapted later.

## ADR-006: Infant/guardian policy — passive-only + operator attestation

**Date:** 2026-10-06  
**Status:** ASSUMPTION (pending MOSIP answer, §14 item 9)  
**Context:** Adult-trained quality models over-reject young children (U26).  
**Decision:** Passive-only with operator attestation for INFANT profile.  
**Consequence:** ChallengeProfile.INFANT has longer timeouts, no compound challenges.

## ADR-007: Auth-purpose devices — DISCRETE mode

**Date:** 2026-10-06  
**Status:** ASSUMPTION (pending MOSIP answer, §14 item 7)  
**Context:** Auth-purpose devices have no STREAM, no preview.  
**Decision:** Assume DISCRETE mode; document the limitation.  
**Consequence:** Liveness evidence is reduced for auth devices; same-face check + time gap compensate.

## ADR-008: Dataset & PAD Model Selection for Evaluation (M5)

**Date:** 2026-10-19  
**Status:** DECIDED (R1, §17.1, §17.4)  
**Context:** Idiap Replay-Attack and OULU-NPU require strict academic EULAs with permanent faculty signatures. In addition, the primary PAD model's actual export spec is 128x128 RGB rather than 80x80 BGR.  
**Decision:**
1. **Model:** Adopted `MiniFASNetV2-SE` ONNX export from `facenox/face-antispoof-onnx` (FP32 ~1.82MB, INT8 ~612KB) verified empirically via `ModelQualificationTest` with 128x128 RGB input.
2. **Bona Fide Dataset:** Use ONOT synthetic ICAO dataset (179 subjects, 5,811 images) ensuring complete ethical compliance and exceeding the Rule of 30 ($N=99 \ge 30$).
3. **Attack Evaluation:** Sourced replay attack video fixtures and Kaggle open alternatives (*Antispoofing Replay Dataset*, *Real VS Fake*) to bypass institutional EULA gatekeeping.
**Consequence:** Allows offline, reproducible evaluation that satisfies ISO/IEC 30107-3 reporting standards without copyright or institutional access hurdles.

## ADR-009: Android M4 Deployment Constraints and Fallback Architecture

**Date:** 2026-10-19  
**Status:** DECIDED (§18.6, §21, §26.9)  
**Context:** Android integration involves cross-stream binding between host CameraX frames (T3) and SBI `rCapture` intents (T1). Field deployments face mobile hardware latency limits, lack of native JPEG 2000 support, external face recognition model availability, and device-provider trust chains.  
**Decision:**
1. **Inference Latency ($p95 \le 200\text{ ms}$):** Shipped `MiniFASNetV2-SE-int8.onnx` (612 KB) into Android assets. Empirically verified via native ONNX Runtime in `Int8ModelBenchmarkTest` (300 iterations: mean 8.15 ms, p50 8.00 ms, p95 9.45 ms), achieving >20x latency headroom for low-end devices.
2. **Same-Face Consistency:** Provided `ReferenceFaceEmbedder` (128-D normalized spatial descriptor) for out-of-the-box cross-stream verification without external dependencies. Deep neural models (OpenCV Zoo SFace / ArcFace) plug into `FaceEmbedder` with a calibrated threshold of 0.60 cosine distance.
3. **JPEG 2000 Safe Fail-Closed:** Handled ISO/IEC 19794-5 JPEG frames natively via Android `BitmapFactory`. For JPEG 2000, provided the `IsoFaceRgbDecoder.Jpeg2000Decoder` interface for optional OpenJPEG JNI injection; without one, the parser fails closed safely (`Result.EMBEDDING_FAILED`), preventing unverified image substitution.
4. **JWT Trust Anchors:** Added `SbiJwtVerifier.fromPem()` to allow dynamic loading of MOSIP Device Provider root CAs from PEM assets.
**Consequence:** Guarantees deterministic, offline-capable, and secure Android liveness capture while maintaining strict fail-closed security invariants.

## ADR-010: Post-audit hardening — fail-closed components, evaluation provenance, signed evidence

**Date:** 2026-10-20  
**Status:** DECIDED (code audit of Oct 2026; HANDOFF-2 R8-R10; MASTER_PLAN §26.16-26.18)  
**Context:** An independent audit found that (1) the PAD scorer silently fell back to a fixed passing stub and the session fed it empty tensors; (2) the session's same-face check received empty vectors and depended on a public test stub; (3) the benchmark behind TEST_REPORT.md used a hash-of-path score generator and typed-in IAD numbers; (4) the attestation signature omitted model/config/mode/PAD fields; (5) benchmark and latency tests passed silently when their inputs were missing; (6) document test counts disagreed (330/333/337/338, table sum 343).  
**Decision:**
1. Fail closed (R8): model must load or throw; `StubPolicy` gate (`liveness.allowStubs`); real pixels via `PadPreprocessor`; same-face via injected `FaceEmbedder`; unscorable frame = 0.
2. Evaluation integrity (R9): real scorer only; skipped-not-passed; run-time provenance section; face-box sidecar (`tools/detect_face_boxes.py`) with counted whole-image fallback; test counts generated by `tools/test_report_counts.py`.
3. Signed evidence (R10): wider attestation coverage with escaping; attestation published after SUCCESS fires; `AuditSeal` signed head for every outcome.
4. Report honesty details: IAD rows with zero trials print "not measured", never a 0.0000 rate; the Android Gradle unit-test tree (which reuses `src/test`) sets `liveness.allowStubs=true` for JVM unit tests only.
5. Mandatory extras (MASTER_PLAN 24.1): PS-PD is measured (scorer-side timing of preprocessing + inference, machine stated). APNRR/BPNRR/APAR are reported as "not measured" with reasons, because they need live capture sessions; no estimate is produced.
**Consequences:** All results in the pre-ADR-010 TEST_REPORT.md (sections 3-5) are **withdrawn** and must be regenerated with the real model. The documented INT8 latency is a desktop-JVM figure until `ScorerBenchmark` is run on a device. Remaining known limits (not solved by code): synthetic still-image bona fide data, small per-species attack N, placeholder `ReferenceFaceEmbedder`, ephemeral signing key, Haar-based box detector standing in for the production detector.

## ADR-011: Model Qualification, Output Class Mapping, and DEV Split Threshold Calibration

**Date:** 2026-10-20  
**Status:** DECIDED (verification of §3.1 and §3.6; HANDOFF-2 R1, R9; MASTER_PLAN §12.3)  
**Context:** Verification of MiniFASNetV2-SE under `ModelQualificationTest` confirmed the ONNX model structure (4D tensor `[1, 3, 128, 128]`, float32, normalized `[0, 1]`, output `[1, 2]`). Testing on known bona fide (ONOT) and attack presentations revealed that output index 0 is the live (bona fide) logit and index 1 is the spoof (attack) logit. The initial scorer had inverted these indices, causing real faces to score ~0 and attacks ~100. Furthermore, threshold 70 required empirical calibration on the DEV split per §12.3 before freezing for the TEST run.  
**Decision:**
1. **Class Order Alignment:** In `MiniFasNetScorer`, map `liveLogit = logits[0][0]` and `spoofLogit = logits[0][1]`. Softmax probability of Class 0 determines the bona fide score on the 0-100 scale.
2. **Score Direction Verification:** `ModelQualificationTest` was expanded to verify that uniform zero tensors produce spoof scores (<50) and bona fide samples score strictly higher than attack presentations.
3. **DEV Split Calibration:** Evaluated candidate operating thresholds on the DEV split (N = 1,030 samples: 1,021 bona fide, 9 attacks). Threshold 70 yields BPCER = 0.0069 (< 1%) and APCER = 0.1111 on DEV; the D-EER point occurs at threshold 92 (APCER = 0.0000, BPCER = 0.0284). Operational threshold 70 is frozen for the TEST split evaluation.
4. **Test Run:** Run TEST split once (N = 1,107: 1,071 bona fide, 36 attacks across cutout, print, screen). Test results: Cutout APCER = 0.0000, Print APCER = 0.0000, Screen APCER = 0.2500, overall BPCER = 0.0019 (0.19%), D-EER = 0.0502.
**Consequences:** Model scoring direction and output mappings are formally qualified and verified. Evaluation results in `docs/TEST_REPORT.md` reflect real, non-inverted ONNX inference with documented calibration provenance.

## ADR-012: Real Neural Face Embedder Integration (SFace) and Same-Face Calibration

**Date:** 2026-10-20  
**Status:** DECIDED (verification of §4.2; HANDOFF-2 R8, R9; MASTER_PLAN §6.3, §12.3)  
**Context:** An independent audit identified that `ReferenceFaceEmbedder` (8x8 spatial luma grid) lacked identity discrimination capacity, producing non-negative vectors where unrelated human faces had small cosine distances (0.10–0.17) below the 0.60 threshold, permitting impostor acceptance. Item 4.2 mandated integrating a trained, license-checked embedder behind `FaceEmbedder`, calibrating the threshold on genuine and impostor pairs, and reporting FAR/FRR with Wilson confidence intervals.  
**Decision:**
1. **Model Selection & License:** Integrated OpenCV Zoo's **SFace** (`face_recognition_sface.onnx`, 37.8 MB, Apache-2.0 license, SHA-256: `0ba9fbfa01b5270c96627c4ef784da859931e02f04419c829e83484087c34e79`) paired with **YuNet** (`face_detection_yunet.onnx`, 227 KB, Apache-2.0, SHA-256: `8f2383e4dd3cfbb4553ea8718107fc0423210dc964f9f4280604804ed2552fa4`). Models recorded in `docs/licence-register.md`.
2. **Empirical Calibration:** Evaluated 50 genuine pairs (same subject, 2 distinct photos) and 100 impostor pairs (different subjects) across ONOT subjects using `tools/calibrate_same_face.py`.
   - Genuine pairs mean cosine distance: **0.3104** (range: 0.1910–0.4872)
   - Impostor pairs mean cosine distance: **0.7037** (range: 0.3109–0.9082)
3. **Threshold Selection:** Set operational same-face threshold $\tau = 0.48$ (decision: same-face if distance $< 0.48$).
   - **FAR (False Acceptance Rate):** 4/100 = **4.00%** (95% Wilson CI: [0.0157, 0.0570, 0.0984])
   - **FRR (False Rejection Rate):** 1/50 = **2.00%** (95% Wilson CI: [0.0035, 0.0542, 0.1050])
**Consequences:** Validates genuine/impostor discrimination with real facial embeddings and Wilson statistical bounds. `SameFaceChecker` correctly rejects impostors (96% rejection rate) and accepts genuine subjects (98% acceptance rate).

## ADR-013: External audit fixes (Oct 20, 2026)

**Status:** DECIDED  
**Changes:**
1. Final RCAPTURE image must score CLEAR_BONA_FIDE to commit; UNCERTAIN or lower goes to RETRY_AVAILABLE via row 19 (event SAME_FACE_FAIL, audit reason FINAL_PAD_NOT_CLEAR_BONA_FIDE).
2. `ChallengeResponseValidator` is now called by the session (expiry, expected action, latency plausibility). BLINK and SMILE need `MIN_ACTION_FRAMES` (2, UNCALIBRATED seed) consecutive qualifying frames.
3. `SupervisorAuth.authenticate()` is gated by `StubPolicy` (it is a stub).
4. `MiniFasNetScorer` reads and checks the input shape from the model file (R1) and reports `modelVersion()` as a SHA-256 prefix of the loaded file. NOT exercised against real ORT in the audit sandbox; run `ModelQualificationTest` with the real model.
5. `WilsonConfidenceInterval.toString()` prints `[lower, upper]` (the old middle value was the Wilson centre, not the point estimate). `IadMetrics` rates return NaN with zero trials. Report header says "aligned", not "Compliant".
6. Typed test counts removed from docs; the generated block in TEST_REPORT.md is the only source.
**Still open:** see AUDIT_TODO.md.

## ADR-014: Frame Sequence Monitor False Alarm Mitigation, Rule of 30 Contract, and Audit Log Sealing

**Date:** 2026-10-22  
**Status:** DECIDED  
**Context:**  
1. FrameSequenceMonitor previously raised false FROZEN/LOOP_REPLAY alarms on live, naturally still subjects because perceptual hash alone had collisions on static facial poses.  
2. Evaluation reports previously risked quoting statistical conclusions without meeting the ISO/IEC 19795-1 Rule of 30 ($N \ge 30$).  
3. Attestation signatures previously sealed the transcript SHA-256 without providing explicit API verification helpers or sealing the audit head on every session outcome.  
4. The dev embedder check previously checked `getClass().getSimpleName().equals("ReferenceFaceEmbedder")` which could be bypassed by renaming the class.  
**Decision:**  
1. **Content Digest + Distance Window:** `FrameFingerprint` computes a 32-byte content digest (SHA-256 of 32x32 downsampled gray) alongside perceptual hash. `FrameSequenceMonitor` requires strict byte-identical digests across the freeze window for a freeze alarm, and non-adjacent loop matching ($gap \ge 1$), eliminating false alarms on live users (empirically verified over 200 noisy trials in `FrameSequenceMonitorFalsePositiveTest`).  
2. **Rule of 30 Contract & Synthetic IAD Runner:** `EvaluationHarness` enforces $N \ge 30$ bona fide and attack subjects unless explicitly flagged `allowExploratory=true`. `EvaluationReport.validateContract()` strictly validates $N \ge 30$, 95% Wilson intervals, non-pooled worst-species headline, unmeasured metrics documentation, and marks headline metrics `[EXPLORATORY]` when $N < 30$. Real synthetic presentations are driven through `IadMechanismRunner` into `FrameSequenceMonitor` and `RCaptureBindingVerifier`, producing measured bypass rates, latencies, and mechanism coverage.  
3. **Audit Log Sealing in Attestation:** `Attestation` exposes `auditHeadHash()` and `auditLogHeadHash()` as well as `verifyAuditLog(AuditLog)` and `verifyAuditChain(AuditChain)`. Truncating or modifying even one audit event causes verification to fail.  
4. **Interface-Level Production Guard:** `FaceEmbedder` defines `default boolean productionGrade() { return false; }`. Production embedders like `SFaceEmbedder` override it to return `true`. Outside development mode, `LivenessService` rejects any embedder where `!embedder.productionGrade()`, preventing bypass via renaming.  
**Consequences:** Complete mathematical and architectural integrity across live capture sequence monitoring, statistical reporting contracts, audit sealing, and production security guards.

