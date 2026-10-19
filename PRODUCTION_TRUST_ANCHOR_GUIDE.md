# Production Deployment and Trust Anchor Guide

Based on the cloned repos (zeroday main 04b3a16, zerodayjunk-). Nothing below was run
against a MOSIP environment; code behavior was read from source.

---------------------------------------------------------------------
# PART A. DIAGNOSIS (what the code does today)

A1. Two different signatures exist. Do not mix them up.
- SBI device JWT: RS256 with `x5c` header, produced by the vendor's SBI/MDS device.
  Verified by `SbiJwtVerifier` (liveness.android).
- Liveness attestation: ECDSA P-256 over a canonical string
  `subjectId|sessionId|verdict|imageSha256|transcriptSha256|deviceTier|captureTimestamp|createdAtEpochMs`,
  signed by `EcdsaSignerStub`.

A2. `SbiJwtVerifier` (read in full):
- `fromPem(String)` parses every certificate in the PEM, throws
  `VerificationException("NO_CERTIFICATE_IN_PEM" | "BAD_CERTIFICATE: ...")` (checked
  exception). The constructor throws `IllegalArgumentException` on an empty anchor list.
  So an empty or missing trust store is fail-closed.
- `verify(jwt)` accepts only `alg=RS256`, takes `x5c[0]` as leaf, checks leaf validity
  dates, then requires the leaf to BE an anchor or be signed directly by one anchor,
  then checks the SHA256withRSA signature. Returns the payload JSON.
- Chains are NOT followed. Intermediates must be added as anchors. Revocation (CRL/OCSP)
  is not checked. Basic constraints, key usage and path length are not checked. The
  clock is `Instant::now` via `fromPem` (no skew setting, no token-age check).
- `disabledForDevelopment()` accepts any token.

A3. Where the verifier is used: `SbiIntentBridge` and `SbiIntentDeviceAdapter` only,
i.e. the ANDROID intent path. The desktop path, `MdsHttpDeviceAdapter` (HTTP to
127.0.0.1:4501, the path the integration guide shows for the Registration Client),
never calls it: when `bioValue` is absent it base64url-decodes the JWT payload and reads
`bioValue` without any signature or certificate check.
Consequence: injecting production CAs with `fromPem()` protects the Android path only.
On Desktop there is currently no trust-anchor verification at all.

A4. `EcdsaSignerStub` generates an ephemeral EC key pair per process (javadoc: "M2
replaces this with a key-store-backed signer"; M2 did not). `Attestation.verify` needs
the same signer instance. Nobody outside the process can verify an attestation; keys do
not survive a restart.

A5. `MiniFasNetScorer` falls back to a stub (score 80, pass threshold 70) when the model
is missing or fails to load, and nothing refuses to start. Attestations then carry
`modelVersion=STUB-v0.0.0`.

A6. Packaging: `pom.xml` is `io.oxalpha:face-liveness-pad:0.3.0-M3`, plain jar, no
shade/assembly plugin, onnxruntime is a normal dependency. `mvn package` therefore
makes a thin JAR (all `liveness.*` classes including evaluation and android packages,
no dependencies, no model). The integration guide says 0.6.0; the pom was never bumped.

A7. The Registration Client has no integration code in the repo. The docs only show the
generic `LivenessService` + `MdsHttpDeviceAdapter` snippet (INTEGRATION_GUIDE 3.1, 6.1).

---------------------------------------------------------------------
# PART B. REMEDIATION AND PROCEDURE

## B1. Get and vet the anchors

1. From the adopter's PRODUCTION MOSIP partner-management/key-manager admin, obtain the
   Device Provider root CA and every intermediate CA in PEM.
2. Verify each out-of-band:

```
openssl x509 -in root.pem -noout -subject -issuer -dates -fingerprint -sha256
openssl x509 -in root.pem -noout -text | grep -A1 "Basic Constraints"   # CA:TRUE
```

3. Concatenate into one bundle (root plus intermediates, because the verifier does not
   walk chains). Record SHA-256 fingerprints in the release notes. Never include the
   test PKI from `SbiJwtVerifierTest`.

## B2. Inject on Android (works today)

```java
String pem = readUtf8(getAssets().open("trust-anchors.pem"));   // or a file path
SbiJwtVerifier verifier;
try {
    verifier = SbiJwtVerifier.fromPem(pem);
} catch (SbiJwtVerifier.VerificationException e) {
    throw new IllegalStateException("trust anchors invalid: " + e.getMessage(), e);  // fail closed
}
SbiIntentDeviceAdapter sbi = new SbiIntentDeviceAdapter(launcher, binder, camera::latestFrame,
        decoder, verifier, "Staging" /* or "Production" */, "Registration", 20000);
```

Use `"Production"` env only with production anchors. Remove every call to
`disabledForDevelopment()` from release builds (add a release-build test that greps for it).

## B3. Desktop: wire the verifier in (required, not present today)

Minimal change to `MdsHttpDeviceAdapter` (sketch; adapt names):

```java
// constructor gets a verifier; reject construction without one in production
public MdsHttpDeviceAdapter(String baseUrl, SbiJwtVerifier verifier) { ... }

// in captureFinal(...), replace the manual payload decode:
String data = extractJsonValue(responseJson, "data");           // the biometric "data" JWT
String payloadJson = verifier.verify(data);                      // throws on any failure
String bioValue = extractJsonValue(payloadJson, "bioValue");
```

Also: send a real `transactionId` and `requestedScore`/`previousHash` per the MDS
spec (currently `requestedScore: 9`, `previousHash: ""`), and decode `bioValue` as
base64url ISO 19794-5 (Android path uses `Iso19794Face`; desktop uses plain Base64).
Add tests mirroring `SbiJwtVerifierTest`: unknown CA, expired leaf, tampered payload,
`alg=none`, empty trust store.

## B4. Startup guards (host application)

```java
if (scorer.isStubMode()) throw new IllegalStateException("PAD model not loaded");
if (verifier.isDisabled()) throw new IllegalStateException("SBI verification disabled");
```

The Registration Client should also reject attestations whose `modelVersion` starts
with `STUB`.

## B5. Attestation signing key

Replace `EcdsaSignerStub` with a key-store-backed signer (PKCS12/HSM on desktop,
Android Keystore on mobile; ECDSA P-256), export the public key (or a certificate) to
the verifying side, and add rotation. Until then attestations are self-signed with a
throwaway key and are not verifiable by the Registration Client.

## B6. Remaining verifier limits to disclose or fix

| Gap                             | Impact                                         | Fix                                   |
|---------------------------------|------------------------------------------------|---------------------------------------|
| no chain building                | adopters must flatten intermediates into anchors | use `CertPathValidator` with PKIX   |
| no revocation                    | revoked device certs still trusted             | CRL/OCSP or short-lived anchors       |
| no key usage / basic constraints | any anchor-signed leaf accepted                | check `keyUsage`, `basicConstraints`  |
| no token-age / skew              | old captured responses acceptable              | enforce `timestamp` freshness (`FreshnessChecker` exists) |

## B7. Package the standalone JAR

```
java -version                          # 11+
mvn clean verify                       # full test run (needs the model for ModelQualificationTest)
mvn clean package -DskipTests
ls -l target/*.jar                     # face-liveness-pad-0.3.0-M3.jar today: THIN jar
```

Make it standalone: add to `pom.xml` (not present today) and bump `<version>`:

```
<plugin>
  <groupId>org.apache.maven.plugins</groupId>
  <artifactId>maven-shade-plugin</artifactId>
  <version>3.5.1</version>
  <executions><execution><phase>package</phase><goals><goal>shade</goal></goals>
    <configuration>
      <createDependencyReducedPom>false</createDependencyReducedPom>
      <filters><filter><artifact>*:*</artifact><excludes>
        <exclude>META-INF/*.SF</exclude><exclude>META-INF/*.DSA</exclude><exclude>META-INF/*.RSA</exclude>
      </excludes></filter></filters>
    </configuration></execution></executions>
</plugin>
```

The shaded JAR has no `Main-Class` (the code is a library); "standalone" means no
extra dependency JARs, not runnable. Then:

```
jar tf target/face-liveness-pad-*.jar | grep -c onnxruntime     # natives bundled
jar tf target/face-liveness-pad-*.jar | grep -iE "test|\.onnx|\.pem"   # must print nothing
sha256sum target/face-liveness-pad-*.jar
```

Ship separately: `MiniFASNetV2-SE-int8.onnx` (git-ignored, 612 KB) with its SHA-256
(`ModelRegistry.registerModel(name, path, expectedSha256)` exists for this), and the
trust-anchor PEM. Smoke test on a clean JVM with a tiny main that loads the model and
asserts `!scorer.isStubMode()`.

## B8. Embedding in the MOSIP Registration Client

The repo gives no client-side integration beyond the generic snippet, so confirm module
names with the adopter's Registration Client build. General route:
1. publish the shaded JAR to the adopter's private Maven repository,
2. add it as a dependency of the client module that performs face capture [VERIFY module],
3. construct `LivenessService` once, call `performLiveness(...)` from a background thread
   (INTEGRATION_GUIDE section 8), using `new MdsHttpDeviceAdapter(url, verifier)` from B3,
4. load the trust PEM from the client's configuration directory, not from the JAR,
5. rebuild the client so its update/integrity manifest includes the JAR; do not hand-copy
   into an installed client [VERIFY with the client's updater].

## B9. Release checklist

- [ ] anchors vetted, fingerprints recorded; test PKI absent
- [ ] desktop path calls `SbiJwtVerifier.verify` (B3) and has negative tests
- [ ] startup guards for stub model and disabled verifier (B4)
- [ ] key-store-backed attestation signer and a verification path (B5)
- [ ] shaded JAR, version bumped, SHA-256 recorded, no test/model/pem inside
- [ ] model shipped with checksum; `isStubMode()==false` verified on target machines
- [ ] real (not simulated) benchmark run; see the protocol file about `BenchmarkReportRunnerTest`
