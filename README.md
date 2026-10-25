# Desktop semantics lifecycle measurements

Raw recordings, logs, source, and measurement files are retained locally. File paths below refer to those local records.

The 1000-node full-label-churn workload used more process CPU after the fix. Its median cost increased by 1.42 ms per completed frame. The small reproduction showed no measurable CPU difference. Memory differences were smaller than the observed run-to-run ranges in both workloads, so no memory improvement or regression is established.

Baseline is `2541e508405f5d9728286d6ed3307163a5406e08`. Patch is `c733f35bb1ec5e6a1be5e8c2d56cb82f02d0d580`, including the preceding regression-test commit `6d82820aae746e240a5ec87517ca110f1c1add6f`. Both apps use locally built release engines. The archived binary hashes and framework UUIDs are in `performance/results-external60-01/manifest.json`. Those local framework identities identify the tested engines.

## Results

Each cell is the median of five runs, followed by the minimum and maximum. CPU percent uses one core as 100 percent. Memory cells describe the median of each run's 250 ms samples, not maximum RSS or live tree storage.

### Small reproduction

| Metric | Baseline | Patched | Median difference |
|---|---:|---:|---:|
| CPU ms / completed frame | 2.410 [1.703, 2.520] | 2.422 [2.299, 2.544] | 0.012 |
| One-core CPU percent | 14.456 [10.216, 15.116] | 14.529 [13.793, 15.261] | 0.073 |
| Completed frames / second | 59.991 [59.988, 59.994] | 59.991 [59.988, 60.040] | -0.000 |
| RSS MiB | 122.328 [122.063, 122.531] | 122.422 [119.750, 122.500] | 0.094 |
| Physical footprint MiB | 59.251 [59.189, 59.454] | 59.017 [56.345, 59.251] | -0.234 |

### Synthetic 1000-node stress

| Metric | Baseline | Patched | Median difference |
|---|---:|---:|---:|
| CPU ms / completed frame | 8.202 [8.102, 8.271] | 9.623 [9.506, 9.729] | 1.421 |
| One-core CPU percent | 49.205 [48.609, 49.619] | 57.730 [57.034, 58.366] | 8.525 |
| Completed frames / second | 59.995 [59.990, 60.021] | 59.992 [59.991, 59.997] | -0.003 |
| RSS MiB | 134.453 [128.734, 135.781] | 132.984 [132.781, 139.547] | -1.469 |
| Physical footprint MiB | 67.595 [61.845, 68.845] | 66.345 [65.767, 72.517] | -1.250 |

## Work and errors

All 20 counted launches passed the frozen acceptance checks. There were zero AXTree errors, zero other matched diagnostic errors, zero missing or duplicate result records, and zero nonzero exits. Every counted launch held framework semantics on while native demand stayed false with no observed transitions.

Each run completed 1200 or 1201 frames in 20.001 to 20.010 seconds, with 81 memory samples. The reproduction retained nine attached Dart nodes at the start and seven at the end, with 13 banner toggles. The dense workload retained 1004 attached Dart nodes, including exactly 1000 visible, unmerged labeled nodes. It applied 1200 or 1201 build batches and 1,200,000 or 1,201,000 label assignments. Raw producer counters and frame timestamps are in each `result.json`.

## Limiter and native work

Both workloads were paced by the 60 Hz display in this experiment. Completed output stayed near 60 frames per second, and the separate sampled call graphs show the main thread waiting for events and display-linked callbacks between frames. The dense patched profile also shows the added `AccessibilityBridge::CommitUpdates` and `AXTree::Unserialize` work on the main thread. The small patched profile contains those native calls as well.

The four `performance/profiles/*/sample.txt` files come from separate eight-second sampling runs. No profiler or build ran during a counted interval. Sampling establishes that the native consumer ran while OS demand was off. It does not count native commits or establish their exact frequency. Dart node traversal proves the producer tree, not native ingestion. Native correctness is corroborated separately by the native tests and AX observations.

The automatic runner leaves its `limiter` and verdict fields pending because it does not read external profiles. This report combines that raw output with the separate profiles. No profiled timing enters the tables. FrameTiming build and raster values exclude platform-thread AX work. The headline metric is total process CPU divided by completed frames.

## Protocol and host

The host is a Mac14,5 with 12 logical CPUs and 32 GiB RAM, running macOS 27.0.1 and Xcode 27.0. Flutter and Dart versions are captured in `flutter-version.json`. Each app used an 800 by 600 content area, centered on external display 5 at 1920 by 1080 and 60 Hz. The native runner fixes that host-specific geometry.

Each workload ran in order A,B,B,A,A,B,B,A,A,B. A launch warmed for five seconds, measured for twenty seconds, then drained FrameTiming callbacks for 1.5 seconds. CPU uses the delta from `clock_gettime(CLOCK_PROCESS_CPUTIME_ID)`, summed across all threads and divided by actual wall time. RSS uses `ProcessInfo.currentRss`. Physical footprint uses `proc_pid_rusage(RUSAGE_INFO_V0)`. End-of-run tree traversal and JSON output occur after the CPU and memory interval closes.

Native demand transitions are recorded continuously while preserving the framework's callback. Trials fail on any observed transition, nonzero exit, diagnostic error, archive mismatch, missing result, display mismatch, fewer than 80 percent of predicted frames, or failed producer counts. Foreground activation must verify the owned visible window during warmup. The pilot and all counted runs preserved native demand off. No GUI queries occur during the measured interval. Ordinary unrelated applications remained open. Balanced ordering limits their drift but cannot remove host noise.

## Native release reproduction

The standalone Dart reproduction holds a `SemanticsHandle` before `runApp`, changes text each frame, and toggles a banner every 1.5 seconds. Run it with `--target=lib/original_repro.dart --dart-define=HOLD_SEMANTICS=true`. After the app has run for at least five seconds, query its window using a native AX client. The recorded runs used Cua Driver 0.37.0, `bring_to_front` by PID, then `get_window_state` for the exact window.

On the baseline, the explicit AX query started 8.56 seconds after launch. The process exited with SIGSEGV and a fault at 0x48 in `CreateRemoveReparentedNodesUpdate`, followed by `CommitUpdates` and `updateSemantics`. On the patch, the explicit query exposed the Frame text and Dismiss action. The app continued beyond 22 seconds with no AXTree errors and then exited normally on Cmd+Q. This is one correctness run per variant and does not establish a failure rate.

The uploaded videos are assembled from exact-window screenshots at about two snapshots per second. They preserve the recorded gaps between capture requests, and do not show every rendered frame. The original 800 by 632 window PNGs and AX records are under `native/`. No desktop pixels or fabricated app states are included.

## Native tests

The new macOS regression failed before the fix because the bridge was absent. The patched native run passed all 241 tests in the macOS target, with two existing disabled tests. The focused embedder accessibility run passed 15 tests, including six shared-runner and separate-runner cases for ordering, null callbacks, and legacy short argument structs. Complete logs are under `tests/`.

## Repeat the measurements

Copy `source/lib`, `source/pubspec.yaml`, `source/pubspec.lock`, and `source/macos/Runner/MainFlutterWindow.swift` into a generated macOS Flutter project. Change the fixed screen number before the experiment if the host uses another display. Keep that condition identical for both variants.

Build each engine source revision with the same `host_release_arm64` configuration. Build targets are `//flutter/shell/platform/darwin/macos:flutter_framework`, `//flutter/lib/snapshot:generate_snapshot_bins`, and `//flutter/tools/font_subset:_font-subset`. Build and archive both baseline apps before replacing that engine output with patched binaries.

For each app, pass `--local-engine=host_release_arm64 --local-engine-host=host_release_arm64 --local-engine-src-path=SDK/engine/src` before `build macos --release`. Supply `WORKLOAD=repro` or `WORKLOAD=dense`, `SOURCE_LABEL` with the engine source revision, `BUILD_LABEL` with the variant and workload, and `EXPECTED_REFRESH_HZ=60` as Dart defines. Archive the complete app, compute the manifest hashes, then run `dart source/run.dart --help` for the four-binary comparison command. Cua Driver is required for the runner's pre-measurement foreground activation.

## Limits

These measurements cover macOS arm64 release builds on one host while framework generation is on and OS demand is off. The dense fixture changes every label every frame and is synthetic stress. It is not a typical app estimate or a per-node cost model. RSS and physical footprint include allocator retention and are not live tree bytes. Windows execution, physical VoiceOver interaction, other platforms, and other activation methods are not covered.

Machine-specific home-directory prefixes are replaced with `$HOME` in this public copy. Numeric samples, counters, binary hashes, and outcomes are unchanged. Private crash-report metadata and application binaries are retained locally rather than included in this archive.
