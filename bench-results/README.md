# Phone benchmark results (bench-app)

Device: Nothing A001 (CMF Phone 2 Pro), MT6878, 8 cores, Android 15, not rooted, release build, ORT 1.16.3 CPU, BASIC_OPT, default intra-op threads.

Models (SHA-256):
- MiniFASNetV2-SE.onnx (FP32): af2381b88f38769222ed93379e12444e2a50814575de1c46170de570c55a42b6
- MiniFASNetV2-SE-int8.onnx (dynamic INT8): fde20585635cae62ed1d41796f76b6f8bc4b92cd91ec1cf0f1bc6485d2d587a9

Files are named `<job>_<model>_<epoch-millis>.json`. Model .onnx files are never committed.

## 2026-10-23-run2-plugged-in
13 runs on the bench-app build with targetSdk 34 (4 INT8 lockstep, 4 FP32 lockstep, 5 FP32 paced). Every JSON reports `charging: true`, so these were taken while plugged in. Thermal status 1, battery temp 37-38 C. Each run is about 4-6 s.

Caveats for every file here: fixed-box detector (real detector cost NOT included), synthetic noise frames (latency only, not accuracy), SFace/signing/camera copy/UI not measured, one phone, no sustained-thermal claim. In paced runs the `per_frame_serial` and `max_sustainable_fps_by_mean` fields only echo the 30 fps producer; use PROCESS_TOTAL and lockstep figures.

## 2026-10-23-run3-unplugged
12 runs on the same build (4 INT8 lockstep, 4 FP32 lockstep, 4 FP32 paced), interleaved. Every JSON reports `charging: false`, thermal status 0 (1 at the end of the last paced run), battery temp 35-37 C, battery 71%. Each run is about 4-6 s. Airplane mode and Do Not Disturb are not recorded in the JSON; the owner reported them on.

## DEV_AGREEMENT_REPORT-2026-10-23.md
Output of `DevAgreementRunnerTest` (desktop JVM, DEV split only, FP32 vs INT8 at threshold 70). Not a phone result. Both model hashes are listed inside the report and match the hashes above.
