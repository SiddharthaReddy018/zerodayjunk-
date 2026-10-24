# Relative blur gate: final report (2026-10-25)

```
MOSIP Decode 2026 P04: relative blur gate, final report draft (2026-10-25)
Status: summary of work merged as of 2026-10-25 (code main e73911d, docs main 3f90d60 before this file). Constants are candidates, not fitted.

WHAT IS ON MAIN
Code repo (SiddharthaReddy018/zeroday) main = e73911d:
- FaceFrame.fullFrameNorm (step C).
- LivenessConfig relativeBlurEnabled (default false), Window 30, DropFactor 2.0, YShiftFactor 1.5 (step D).
- LivenessSession hook after the QualityGate fail block: gate fed only when enabled; a non-OK state after samples exist is a quality failure (QUALITY_DROPPED from PASSIVE, else stay in POSITIONING).
- Live gate bench (CameraRelativeGateBenchmark) and latency bench (FullNormLatencyBenchmark).
- RelativeBlurSessionTest: 7 tests, including 2 that read the audit trail for the blur reason and the steady control.
- Maven on the last branch: 639 tests, 0 failures, 0 errors, BUILD SUCCESS.
Docs repo (SiddharthaReddy018/zerodayjunk-) main = 3f90d60: live-relative-gate-flip and fullnorm-latency-thermal1 folders with READMEs and raw JSONs, index updated.

RESULTS (one phone: Nothing A001, MT6878, Android 15, release build)
- Flip run (user-confirmed Run 1): norm p50 0.0214 frames 0-149 vs 0.0067 frames 250-329, ratio 3.19 (above the 2.0 drop factor). Counts I 29, O 136, B 78, L 0, not_fed 87. First blur at index 203. Y p50 stayed 103-109 after the first block.
- Control (Run 2, flap OFF all run): I 29, O 301, B 0, L 0, not_fed 0, ratio 1.07.
- Four no-flip runs: B 0, L 0. Sharp-level run-to-run drift about 1.5x, close to the 2.0 drop factor.
- Latency (2 runs, 500 timed): fullFrameNorm p50 1.223 / 1.221 ms, p95 1.259 / 1.256. toRgb p50 3.444 / 3.455 ms. fullFrameNorm is about 35% of toRgb. No detector running, no ImageProxy copy. Thermal status 1, battery 37.0 C (above the 36.0 C rule), charging False per JSON.
- Session test: with the flag on, a 3x drop after a sharp baseline puts guide.quality.blur_relative in the audit trail (synthetic device).
- Sensitivity check (temporary local edit, reverted, tree verified clean): with the reason string changed, 1 of 7 RelativeBlurSessionTest tests failed (the blur-reason test). In that run the session ended in POSITIONING (verdict PENDING), one run only.

DECISIONS
- Baseline LOST: fail closed (user decision).
- Sessions with no valid full-frame sample: relative check skipped. Documented hole, never decided closed.
- Feature is additive, default off. Nothing in the bench or the real app turns it on.
- laplacianVariance and blurMinVariance 100.0 untouched.
- Constants 30, 2.0, 1.5 are candidates, not fitted.

NOT VERIFIED
- LivenessSession with the flag on, live on the phone (no bench job builds a session; Maven test only, synthetic device).
- PACED composite run; FROZEN under rot270.
- Other phones, lights, distances; hand shake or motion blur; face-region blur over a sharp background.
- Accuracy against attacks.
- Cost while the detector runs concurrently; ImageProxy copy cost.
- Flip time of the flip run (not recorded); flapon_1792871351791 (still on the phone, never pulled, excluded).
- Flap state, same light, unplugged for the gate runs: user-stated, not in the JSONs. Gate JSONs have no thermal or charging fields.
- Signing; UI mapping of the reason strings; :liveness-android module compile (Flutter missing).
- Hackathon judging criteria.

OPEN DESIGN HOLES
- No reset for LOST: a sustained light change stays LOST.
- INSUFFICIENT at session start blocks the first 30 fed frames (fail closed; pass-through is a one-line change).
- A session that starts blurred is not caught (the baseline is its own first window).
- Honest users who change light get rejected.
- Sessions with no samples bypass the check.
- Class comment says "face crop" but full_norm uses the whole-frame bbox path.
```
