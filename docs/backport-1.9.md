# 1.9 GDPS backport (in progress)

Target: the Windows x86 and Android ARMv7 clients distributed at
https://19gdps.com/download (u9.0.4). This is a native 1.9 client port.

## SDK references

- SDK: https://github.com/qimiko/geode, branch `gdps`, commit
  `d860fcecdfe47d1a42171cd5e0c857c1451fd3f8` (Geode 4.9.0).
- Bindings: https://github.com/geode-sdk/bindings, commit
  `7f6c2a75742856de88dad354e576dcff8a28e881`, directory `bindings/1.920`.
- Set `GEODE_SDK` and `GEODE_BINDINGS_REPO_PATH` to these local checkouts.
- Windows requires an x86 compiler target (`-A Win32` with Visual Studio).
- Android requires `armeabi-v7a`; Android64 and iOS are not release targets.

The SDK still contains comments advertising GD 2.2074 for upstream build
scripts. Do not use those comments to select bindings or upstream loader
binaries. The SDK's actual default game version is 1.920.

## Work remaining before release

The manifest and build configuration alone do not constitute a working port.
Do not distribute this branch until source conversion and validation finish.

- Replace GJBaseGameLayer hooks with 1.9 PlayLayer and LevelEditorLayer hooks.
  In 1.9 these are separate CCLayer subclasses.
- Convert input recording/playback to pushButton/releaseButton and frame
  advancement in PlayLayer::update; preserve 1.9 frame-dependent physics.
- Map checkpoint and player snapshots to actual 1.920 fields and signatures.
- Port utility hooks, trajectory, rendering, and UI to the older game API.
- Adapt Geode 5 UI/event calls to the Geode 4.9 SDK and replace the upstream
  node-ids dependency with verified 1.9 node lookup.
- Replace upstream CI SDK downloads with pinned GDPS SDK builds.
- Compile and package for Windows x86 and Android ARMv7; exercise recording,
  playback, restart, practice checkpoints, dual mode, frame stepping, macro
  load/save/edit, rendering, and utility settings in the target clients.