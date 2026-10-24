Composite latency bench (YuNet + ML Kit contours), 2026-10-24
Device: Nothing A001 (MT6878, Android 15), io.oxalpha.liveness.bench, release build.
Status: benchmark/candidate only. Not wired into LivenessAndroidBootstrap. No accuracy validation.

RESULTS (mean ms)
1-person crop (upscaled, face ~2/3 height), 3 runs, faces=1, pose ok, 42 landmark values:
  yunet 34.2-43.9 (noisy; run 1 stall, p95 120)
  mlkit 29.2-29.7
  composite 52.9-53.2 mean, p95 59.0-59.8, p99 60.4-61.9
  composite is ~10 ms below yunet+mlkit sum. Cause not established.
2-person frame: YuNet counts 2, composite returns early (~34 ms), ML Kit skipped,
  pose unavailable, landmarks null. By design in CompositeFaceDetector.detect.
  ML Kit alone counts 1 on that frame (largest face; a different person than YuNet's largest).
YuNet-only reference (thermal 1-2): 47.1-48.1 before 0002, 32.6-33.5 after; ORT ~88% of call.

CAVEATS
thermal_status was 3 for composite runs (YuNet-only runs: 1-2). Numbers are likely pessimistic.
Airplane/DND state not recorded in JSON.
Budget: composite ~53 + PAD ~16 = ~69 ms, sum of separate runs, not end to end.
Does not fit inline at 30 fps (33.3 ms): needs worker thread or every Nth frame.

NOT VERIFIED
End-to-end latency, camera/ImageProxy copy, SFace, signing, accuracy, ML Kit time vs face size,
cool-phone (thermal 0) numbers, why composite < sum, NaN entries in merged landmark array,
ML Kit FAST-mode pose quality, 1280x720 camera path.
Runs: MOSIP/3 = 3 runs on 2-person frame; MOSIP/4 = 3 runs on 1-person crop (folder 4 presumed 1-person, JSON contents not re-checked). Filenames carry epoch ms, ordered by run.
