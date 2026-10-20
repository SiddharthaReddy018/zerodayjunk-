# Android Build, Deploy and Test Guide (Physical Device)

Written after cloning and reading `zeroday` (main, commit 04b3a16) and `zerodayjunk-`.
I could NOT run Gradle, Flutter or Maven here (no Android SDK, no Maven Central access),
so build steps are verified by reading the code, not by running them. Items marked
[UNTESTED] need a first run on your machine.

---------------------------------------------------------------------
## 0. What the repo actually contains (read this first)

| Piece                          | State                                                         |
|--------------------------------|---------------------------------------------------------------|
| `android/liveness-android`     | Gradle library (AGP 8.2.2), compiles shared `src/main/java` + glue |
| `android/settings.gradle`      | No `pluginManagement`/repositories: will not resolve (fix below) |
| Gradle wrapper (`gradlew`)     | NOT in repo. Install Gradle 8.2+ or run `gradle wrapper`      |
| `flutter_embedding_release`    | placeholder version `1.0.0-REPLACE_WITH_ENGINE_HASH` (fix below) |
| `flutter/liveness_ui`          | Dart package: screens, widgets, client. NO `main.dart`, NO `android/` host app |
| ONNX model                     | NOT in repo (`*.onnx` is git-ignored). README claim that it is shipped is wrong |
| Face detector                  | Only the `FaceDetector` interface. No ML Kit/MediaPipe implementation |
| Native camera view for Flutter | Not provided (JPEG preview channel exists, see 5)             |
| Mock SBI Android app           | Not in repo                                                   |
| Real SBI app                   | Needs a device-vendor/MOSIP SBI app implementing the Android SBI draft |

So "build and run on a phone" = build the library, CREATE a host Flutter app, supply a
face detector and the model, and have an SBI app installed. Steps below do that.

## 1. Prerequisites

| Tool        | Version                                  | Check                    |
|-------------|------------------------------------------|--------------------------|
| JDK         | 17 (Gradle/AGP 8.2); sources stay Java 11 | `java -version`          |
| Gradle      | 8.2+ (no wrapper in repo)                | `gradle -v`              |
| Android SDK | platform 34, build-tools 34, platform-tools | `sdkmanager --list_installed` |
| Flutter     | stable, Dart >= 3.0 (pubspec constraint) | `flutter doctor -v`      |
| Maven       | 3.8+ and JDK 11+ for the core tests      | `mvn -v`                 |

```
export ANDROID_HOME=$HOME/Android/Sdk
export PATH=$PATH:$ANDROID_HOME/platform-tools
flutter doctor --android-licenses
```

Phone: Developer options > USB debugging, connect, accept RSA prompt.

```
adb devices                                  # must say "device"
adb shell getprop ro.build.version.sdk       # must be >= 28 (minSdk)
adb shell getprop ro.product.cpu.abi         # arm64-v8a expected
```

## 2. Core tests first (JVM)

```
git clone https://github.com/SiddharthaReddy018/zeroday.git && cd zeroday
mvn -q test                                  # expect the count printed by tools/test_report_counts.py (see TEST_REPORT.md generated block)
mvn -q -Dtest='liveness.android.*Test' test  # Android core only
```

`ModelQualificationTest` needs the real model file; without it expect it to skip/fail
(README-M4 excluded it when ONNX was stubbed). Record your actual count.

## 3. Build the Android library

3a. Replace `android/settings.gradle` with `android_fixes/settings.gradle` (attached).

3b. Fix the Flutter embedding coordinate in `android/liveness-android/build.gradle`:

```
# engine hash for your Flutter install
cat $(dirname $(dirname $(which flutter)))/bin/internal/engine.version
```
```
compileOnly 'io.flutter:flutter_embedding_release:1.0.0-<that hash>'
```

3c. Put the model where the library expects it (git-ignored, so copy by hand):

```
mkdir -p android/liveness-android/src/main/assets
cp /path/to/MiniFASNetV2-SE-int8.onnx android/liveness-android/src/main/assets/
```

3d. Build:

```
cd android
gradle wrapper --gradle-version 8.5          # once, creates ./gradlew
./gradlew :liveness-android:assembleDebug    # AAR: liveness-android/build/outputs/aar/
./gradlew :liveness-android:testDebugUnitTest
```

| Symptom                                 | Fix                                                    |
|-----------------------------------------|--------------------------------------------------------|
| `Plugin com.android.library not found`  | settings.gradle not replaced (3a)                      |
| cannot resolve `flutter_embedding_release` | 3b version and the flutter.io repo in settings      |
| `Unsupported class file major version`  | Gradle on wrong JDK; JAVA_HOME -> 17                   |
| `SDK location not found`                | `android/local.properties`: `sdk.dir=/path/to/Sdk`     |
| `UnsatisfiedLinkError` onnxruntime      | phone ABI missing from the AAR; check abiFilters       |

## 4. Create the host Flutter app (repo has none)

```
flutter create --platforms=android --org org.example liveness_host
cd liveness_host
```

pubspec.yaml:

```
dependencies:
  flutter: { sdk: flutter }
  liveness_ui: { path: ../zeroday/flutter/liveness_ui }
```

`android/settings.gradle` (host): add

```
include ':liveness-android'
project(':liveness-android').projectDir = file('../../zeroday/android/liveness-android')
```

`android/app/build.gradle`: `minSdkVersion 28`, and in dependencies:

```
implementation project(':liveness-android')
implementation 'com.google.mlkit:face-detection:16.1.6'   // for the detector in 4b
```

Make sure the host also resolves the flutter.io and google repos (same repositories as 3a).

`lib/main.dart`:

```dart
import 'package:flutter/material.dart';
import 'package:liveness_ui/liveness_client.dart';
import 'package:liveness_ui/screens/resident_registration_screen.dart';

void main() => runApp(MaterialApp(home: Builder(builder: (c) => Scaffold(
  body: Center(child: ElevatedButton(
    child: const Text('Start resident liveness'),
    onPressed: () => Navigator.push(c, MaterialPageRoute(
      builder: (_) => ResidentRegistrationScreen(
        subjectId: 'subject-1', machineUserId: 'machine-1',
        client: LivenessClient(), locale: 'en',
        onSuccess: (a) => debugPrint('ATTESTATION $a'),
        onClose: () => Navigator.pop(c)))),
  ))))));
```

AndroidManifest (host app): `<uses-permission android:name="android.permission.CAMERA"/>`.
The library manifest already declares the `<queries>` for `io.sbi.device` and the root
packages; manifest merger carries it into the app. [UNTESTED]

### 4a. MainActivity (Java). Must extend FlutterFragmentActivity

`AndroidSbiLauncher` needs a `ComponentActivity`; plain `FlutterActivity` is not one.

```java
package org.example.liveness_host;

import android.os.Bundle;
import android.widget.FrameLayout;
import androidx.annotation.NonNull;
import androidx.camera.view.PreviewView;
import io.flutter.embedding.android.FlutterFragmentActivity;
import io.flutter.embedding.engine.FlutterEngine;
import java.io.*;
import java.nio.file.Path;
import java.util.ArrayList;
import java.util.function.Supplier;
import liveness.android.*;
import liveness.android.glue.*;
import liveness.challenge.ChallengeGenerator;
import liveness.crypto.EcdsaSignerStub;
import liveness.engine.MiniFasNetScorer;
import liveness.engine.SameFaceChecker;
import liveness.flow.RetryRateLimiter;
import liveness.policy.LivenessConfig;
import liveness.service.LivenessService;
import liveness.device.DeviceAdapter;

public class MainActivity extends FlutterFragmentActivity {
    private final CameraXController cameraController = new CameraXController();

    @Override
    public void configureFlutterEngine(@NonNull FlutterEngine engine) {
        super.configureFlutterEngine(engine);

        // 1. Model: MiniFasNetScorer takes a file Path, so copy the asset out of the APK.
        Path model = copyAsset("MiniFASNetV2-SE-int8.onnx");
        MiniFasNetScorer scorer = new MiniFasNetScorer(model);
        if (scorer.isStubMode()) {   // CRITICAL, see section 7: stub returns 80 = PASS
            throw new IllegalStateException("PAD model failed to load; refusing to start");
        }

        // 2. Engine
        LivenessService service = new LivenessService(
                LivenessConfig.defaults(), scorer, new SameFaceChecker(0.6),
                new ChallengeGenerator(), new RetryRateLimiter(3), new EcdsaSignerStub());

        // 3. Devices (see README-M4 section 4)
        AndroidSbiLauncher sbiLauncher = new AndroidSbiLauncher(this);
        CameraXDeviceAdapter camera = new CameraXDeviceAdapter("front", new MlKitFaceDetector());
        CrossStreamBinder binder = new CrossStreamBinder(
                new SameFaceChecker(0.15), new ReferenceFaceEmbedder(), 8000);
        SbiJwtVerifier verifier = SbiJwtVerifier.disabledForDevelopment();   // DEV ONLY, see 6
        SbiIntentDeviceAdapter sbi = new SbiIntentDeviceAdapter(sbiLauncher, binder,
                camera::latestFrame, new IsoFaceRgbDecoder(new AndroidRgbDecoder(), null),
                verifier, "Developer", "Registration", 20000);
        Supplier<DeviceAdapter> factory = () -> new AndroidDeviceAdapter(
                new EnvironmentIntegrityChecker(new AndroidEnvProbe(this)), camera, sbi);
        LivenessFlutterPlugin.configure(service, factory);
        engine.getPlugins().add(new LivenessFlutterPlugin());

        // 4. CameraX needs a PreviewView; a 1x1 view is enough because Flutter shows the JPEG stream
        PreviewView pv = new PreviewView(this);
        addContentView(pv, new FrameLayout.LayoutParams(1, 1));
        requestCameraThen(() -> cameraController.bind(this, this, pv, camera));
    }

    private Path copyAsset(String name) { /* copy getAssets().open(name) -> new File(getFilesDir(), name); return its toPath() */ return null; }
    private void requestCameraThen(Runnable r) { /* ActivityResultContracts.RequestPermission for CAMERA, then r.run() */ }
}
```

Constructor names above come from README-M4 section 4 and the tests. The two helper
bodies are routine and left as comments. `ReferenceFaceEmbedder` threshold 0.15 follows
README-M4 section 7. [UNTESTED]. Check `ReferenceFaceEmbedder` and `IsoFaceRgbDecoder`
constructors before compiling.

### 4b. Face detector (missing piece, needed for the quality gate)

Implement `liveness.android.FaceDetector.detect(byte[] rgb, int w, int h)` returning an
`Observation(faceCount, bboxX, bboxY, bboxW, bboxH, yawDeg, pitchDeg, rollDeg, landmarks)`.
ML Kit gives bbox plus `headEulerAngleY/X/Z`. It does NOT give the 6-point eye and mouth
landmark layout the blink (EAR) and smile (MAR) challenges need, so with ML Kit alone
set `landmarks=null`: the passive path and head turn work, blink/smile cannot pass.
For blink/smile use MediaPipe Face Landmarker and map to the layout `LandmarkDetector`
expects (check `src/main/java/liveness/engine/LandmarkDetector.java`). I can write this
adapter next.

## 5. Run on the phone

```
cd liveness_host
flutter devices
flutter run -d <id> --release          # release for any latency numbers
flutter build apk --release && adb install -r build/app/outputs/flutter-apk/app-release.apk
adb shell pm grant org.example.liveness_host android.permission.CAMERA
```

Preview: the Flutter screen shows `cameraPreview` if you pass a native widget (a
PlatformView is not provided), otherwise JPEG frames from `mosip.liveness/preview`.
Enable them from Dart before starting: `client.updateConfig({'previewEnabled': true, 'previewMaxFps': 10})`.

## 6. Test CameraX frame flow

The only log tag in the repo is `LivenessBench` (ScorerBenchmark). The analyzer logs
nothing, so to count frames add a temporary `Log.d("LivenessFrames", ...)` in
`glue/CameraXAnalyzer.analyze()`, then:

```
adb logcat -c && adb logcat -v time LivenessFrames:D *:S | tee frames.log
awk '{print substr($2,1,8)}' frames.log | sort | uniq -c | head
```

Checks (all by reading `CameraXController`): analysis is 640x480 YUV_420_888 with
`STRATEGY_KEEP_ONLY_LATEST`, front camera only. Verify on device:

| Check            | How                                                   | Pass                           |
|------------------|-------------------------------------------------------|--------------------------------|
| Frame rate       | log above                                             | steady 15-30 fps, no gaps > 200 ms |
| Rotation         | rotate phone mid-session                              | face stays upright to detector |
| Lifecycle        | background 10 s, return                               | camera rebinds, no crash       |
| Permission       | `adb shell pm revoke <app> android.permission.CAMERA` | frame timeout -> DEVICE_ERROR, no crash |
| Model latency    | `ScorerBenchmark.run(ctx, "MiniFASNetV2-SE-int8.onnx", 30, 300)`, read `LivenessBench` | p95 << 200 ms (JVM test showed 9.45 ms; measure on the phone) |
| Environment      | run on rooted phone / emulator                        | `SESSION_COMPLETE{verdict:DEVICE_ERROR}`, no SBI launch |

Record the phone model with every latency number you publish.

## 7. SBI intent testing (corrected: the brief's action names are NOT what the code uses)

The code follows the MOSIP Android SBI draft (`SbiIntentBridge`):

| Step       | Intent action          | Request                                  |
|------------|------------------------|------------------------------------------|
| Discovery  | `io.sbi.device`        | byte[] extra `input` = `{"type":"Face"}`; reply `callbackId` = appId |
| Info       | `<appId>.Info`         | empty input                              |
| rCapture   | `<appId>.rCapture`     | byte[] extra `input` = JSON (env, purpose, specVersion 0.9.5, timeout, captureTime, transactionId, bio[]) |
| Response   | extra `response`       | bytes, or a `content://` URI read once   |

Rules enforced by the adapter: `transactionId` 4..50 chars `[A-Za-z0-9-]`; the engine
sends `"T" + 40 hex of SHA-256(boundId)`; result image is bound to the live stream by
nonce + time gap + same-face; JWT in `data` must verify (RS256, x5c).

7a. Is an SBI app installed and visible? (the discovery intent)

```
adb shell cmd package query-activities -a io.sbi.device
```

Empty = no SBI app answers, and the app will report `SBI_NOT_INSTALLED`.

7b. adb cannot send `byte[]` extras, so you cannot drive the real `input` contract from
the shell. Use one of: (i) the JVM tests, which already fake the launcher
(`SbiIntentBridgeTest`, `AndroidPipelineIntegrationTest`); (ii) a small caller or mock
SBI app (not in repo; I can write a mock SBI responder that returns discovery/Info/
rCapture responses so the full on-device flow runs without vendor hardware);
(iii) a real vendor SBI app installed on the phone.

7c. Real SBI run: install the SBI app, switch `SbiJwtVerifier` to production anchors
(Task 4 guide), run a resident session, expect
`STATE_CHANGE ... SESSION_COMPLETE {verdict:PASS, attestation.deviceTier:T1}`.
Watch: `adb logcat -v time | grep -iE "sbi|liveness"`.

Negative tests: unknown specVersion, bad transactionId, user Back mid-capture,
camera permission revoked, rooted device. Each must end in a defined error, not a crash.

## 8. Findings from the review (diagnosis, no fixes mixed in)

1. `MiniFasNetScorer` silently falls back to STUB mode when the model path is null,
   missing or fails to load. Stub score is 80 and the default pass threshold is 70,
   so with no model every frame scores as bona fide. Nothing in the service checks
   `isStubMode()`. Attestations record `modelVersion` = `STUB-v0.0.0` in that case.
2. `android/settings.gradle` has no repositories; build cannot resolve AGP or deps.
3. No Gradle wrapper; no `main.dart`/host app; no model; no face detector impl.
4. README-M4 section 7 states the model is shipped and the latency verified; the repo
   holds no `.onnx`. The latency table came from a local file that is not committed.
5. `EcdsaSignerStub` makes an ephemeral key per process; see the Task 4 guide.

Remediation for 1: check `scorer.isStubMode()` at startup (done in the MainActivity
sketch) and have the Registration Client reject attestations with a STUB model version.
