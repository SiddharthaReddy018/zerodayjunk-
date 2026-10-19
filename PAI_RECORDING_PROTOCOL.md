# PAI Recording Protocol ("Record Yourselves")

Produces the team-recorded set for `datasets/manifests/evaluation_benchmark_manifest.csv`.
Verified against the repo: `ManifestLoader` schema, species names and split values.
`02_build_manifest.py` output was loaded by the repo's real ManifestLoader (7/7 rows OK).

---------------------------------------------------------------------
## 1. Rules

1. Only team members who sign a consent line appear. No third-party photos.
2. Count PRESENTATIONS, not frames. One clip/photo = one presentation.
3. Same capture rig for attacks and bona fide.
4. Record everything, including bad attempts. No cherry-picking.
5. Originals are read-only after import. Never commit images (repo ignores `datasets/`).
6. Splits are per SUBJECT (subject-disjoint). Team data defaults to `test` only; freeze
   thresholds on dev data first, run the test split once (HANDOFF-2 M5 gate).

## 2. Sample size reality check

Wilson 95 percent upper bound with ZERO failures: n=30 -> 11.3 percent,
n=60 -> 5.9 percent, n=100 -> 3.6 percent.

| Group                    | Minimum presentations | Better |
|--------------------------|-----------------------|--------|
| each attack species      | 30                    | 60     |
| bona fide, total         | 30                    | 100+   |

Fairness (HANDOFF-2 R7): a skin-tone / lighting / glasses cell needs >= 30 SUBJECTS to be
reported as more than "exploratory". A team of 4 cannot reach this. Report team
fairness numbers as exploratory, and keep ONOT (99 test subjects) as the fairness
evidence. The script prints subjects per cell so this is explicit.

## 3. Species (these are the exact manifest values)

| Folder name   | Meaning                                         | Token `sub-` examples       |
|---------------|-------------------------------------------------|-----------------------------|
| `print`       | flat 2D print                                   | matte, glossy               |
| `curved_print`| print bent/wrapped on a cylinder                | bottle, hand                |
| `screen`      | STILL photo shown on a screen                   | phone, tablet, laptop       |
| `video`       | VIDEO replay shown on a screen                  | phone, tablet, laptop       |
| `cutout`      | print with eye (or eye+mouth) holes, live eyes  | eyes, eyes_mouth            |

Note: matte vs glossy and phone/tablet/laptop are recorded in the extra `pai_subtype`
column, NOT as separate species. Keeping species canonical means the harness reports
one APCER per species and the worst-species headline is computed over the 5 above.
If you want a per-subtype table, filter on `pai_subtype` yourself.

Presenting (vary across the 30+ per species): hand-held vs stick vs clamp; hand visible
vs hidden; distance 40 and 60 cm; slight shake vs still; A4 and A5; inkjet vs laser;
screen brightness 50 and 100 percent; bezel visible vs cropped; angle 0 and 20 degrees;
cutout holes tight vs loose; glasses on/off behind the cutout; blink in one clip and a
head turn in another.
Replay source video: record with a DIFFERENT camera than the capture rig, 10 s, frontal,
natural blink, slight motion.
Deepfake replays (if any): species stays `screen` or `video`; set content_origin deepfake
by hand in the CSV (script writes `real`).

## 4. Rig

Capture device = the one used in the demo; note its model in `dev-`.
Tripod, eye level, 40-60 cm, 720p or 1080p, 30 fps, auto-exposure ON, no beauty filter.
Clip length 5-8 s. Face or PAI fills the oval.

Lighting tokens (`lit-`), rotate through ALL for every species and for bona fide:
`office` (ceiling light), `window` (side daylight), `dim` (one lamp), `backlit`
(window behind), `warm` (warm lamp only). The script does NOT guess lighting: a
missing `lit-` token leaves the column blank and prints a warning.

Bona fide (`bona_fide/genuine/`): same five lightings; glasses on/off (`gl-yes`/`gl-no`);
cap on/off; slight tilt; 40 and 60 cm.

## 5. Folder and file names

```
recordings/
  subjects.csv
  S01/
    attack/
      print/         001__lit-office__dev-pixel7__sub-matte__pai-hpLaser__var-a4hand.jpg
      screen/        002__lit-dim__dev-pixel7__sub-phone__pai-iphone12__var-b100.jpg
      video/         003__lit-window__dev-pixel7__sub-tablet__pai-ipad9__var-angle20.mp4
      curved_print/  004__lit-office__dev-pixel7__sub-bottle__var-a4.mp4
      cutout/        005__lit-office__dev-pixel7__sub-eyes__gl-no__var-tight.mp4
    bona_fide/
      genuine/       001__lit-office__dev-pixel7__gl-yes.mp4
  _frames/           (created by the script)
```

Tokens: `lit` lighting, `dev` capture device, `sub` PAI subtype, `pai` PAI device/printer,
`gl` glasses yes/no, `var` variant (no spaces, no commas), `sess` session id.

## 6. subjects.csv

```
subject_id,mst_scale,consent_signed,split,gender,age_group
S01,4,yes,test,F,20-30
S02,8,yes,test,M,20-30
```

`split` optional (default `test`). `gender`, `age_group` optional; `age_group` must be one
of 0-20, 20-30, 30-40, 40-50, 50+ if given. `mst_scale` 1-10 labeled by two people
independently against the MST card in daylight; differences > 1 get discussed. The
script never guesses MST. Subjects missing from this file, or without consent=yes,
are skipped.

## 7. Run

```
pip install opencv-python numpy
python 02_build_manifest.py --root recordings --subjects recordings/subjects.csv \
    --out manifest_out --fps 2 --max-frames 20
# optional: --mst-mode raw   (1..10 instead of MST_1-3 / MST_4-6 / MST_7-10 bands)
```

Output: `manifest_out/evaluation_benchmark_manifest.csv`, frames in `recordings/_frames/`.
Paths in the CSV are relative to `recordings/`.

CRITICAL (found while reviewing the repo): the existing ISO benchmark numbers are NOT
measured. `BenchmarkReportRunnerTest` scores each sample with a fake scorer built from
`path.hashCode()` and hard-coded "realistic" distributions, asserts >1000 entries, and
fills the IAD table with hard-coded attempt counts. So `docs/TEST_REPORT.md` section 3-4
(APCER 0.0000, BPCER 0.0000, D-EER ~0, IAD 100 percent) is a simulation of the harness
plumbing, not model performance on images. No real `EvaluationHarness.Scorer` exists
in the repo, and no `.onnx` model is committed.
Before claiming empirical results you need a Scorer that, per manifest row, loads the
image, runs face detection, QualityGate and crop, preprocesses to 128x128 RGB [0,1],
and calls `MiniFasNetScorer.score` with the real INT8 model loaded (assert
`isStubMode() == false`). Your recorded set is only meaningful once that exists.
Until then, label the report "harness validation on synthetic scores".

## 8. Session checklist

- [ ] battery > 50 percent, storage > 5 GB, lens clean
- [ ] distance marked on the table
- [ ] subject says "S01, print, window, 001" at clip start
- [ ] one species at a time; rotate lighting after every 6
- [ ] import, run the script, read every warning
- [ ] back up `recordings/` privately; push only the CSV
