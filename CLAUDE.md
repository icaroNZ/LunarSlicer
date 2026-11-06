# CLAUDE.md - LunarSlicer (Snapmaker 2.0 / A250T workspace)

## Purpose
LunarSlicer is Snapmaker's fork of Ultimaker CuraEngine: a headless C++ FDM slicer that turns STL meshes plus Cura-style
`.def.json` settings into Marlin G-code. Luban spawns it as a child process for every 3D-print slice
(`Luban/src/server/slicer/Slicer.ts:30-52`). It is machine-agnostic: no A150/A250/A350/A250T identifiers exist in
`src/` (machine geometry, start/end G-code, speeds arrive as JSON from luban-print-settings).

## READ FIRST: two lineages, and this checkout is NOT the shipped one
- `a250t/main` (this checkout) == `origin/maintain-4.8` @ `5c40f1ba` (11/04/2025): CuraEngine 4.8-era fork point
  `83c65a1a` (06/11/2020) + 5 Snapmaker commits (`support` command, release CI, arm64 CI). Plain CMake, C++11.
  No workspace code changes; only this file was added.
- `origin/main` (and `origin/fork-5.3`): CuraEngine 5.3.0 (conan, Arachne, spdlog), forked 16/03/2023 (`14357ff25`)
  + Snapmaker commits to 26/01/2024 (single-model multicolour slicing, painted surfaces, PolygonsVoronoi, M109 C/W).
  **This is what Luban 4.15 ships** in npm `@snapmaker/snapmaker-lunar` 1.4.9
  (`Luban/node_modules/@snapmaker/snapmaker-lunar/engine/MacOS/LunarSlicer`, x86_64; `strings` shows `5.3.0`,
  `wall_transition_angle`, `colorful_slicing_enable`).
- Current luban-print-settings definitions are `setting_version 21` (Cura 5.x). Verified 07/11/2026: the 4.8 build
  from this checkout fails on them (`Trying to retrieve setting with no value given: 'material_shrinkage_percentage'`,
  then `roofing_layer_count`, `adhesion_extruder_nr`; with those forced via `-s` it segfaults at "Starting support...").
- Verified 07/11/2026: the shipped 5.3 binary (Rosetta) slices `tests/integration/resources/cube.stl` with
  `-j luban-print-settings/resources/printing/active.def.json` (exit 0, all logs on stdout, emits `M109 S215 C5 W0`);
  the file header keeps the placeholder `;TIME:6666` (`src/gcodeExport.cpp:232`), the real time is only on stdout.
- For "what does the A250T receive from Luban" questions, read `origin/main` with `git show origin/main:<path>` /
  `git grep <pat> origin/main -- src include`; do not check it out over `a250t/main`.

## Tech stack (a250t/main)
C++11 (`CMakeLists.txt:47`), CMake >= 3.6, vendored `libs/clipper` + `libs/rapidjson`, Stb (git-cloned at configure
time by `cmake/FindStb.cmake`), OpenMP optional (`cmake/FindOpenMP.cmake`, Snapmaker-added), Arcus/protobuf optional
(`ENABLE_ARCUS` OFF by default -> no `connect` command), GoogleTest 1.8.0 (`BUILD_TESTS`, git-cloned by
`cmake/FindGMock.cmake`). `CURA_ENGINE_VERSION` defaults to `master` (banner `Luban_Engine version master`).

## Build and test on this Mac (verified 07/11/2026, Apple Silicon, Apple clang, CMake 4.4.4)
cmake is not installed system-wide. Use `brew install cmake` or a throwaway venv (`python3 -m venv <dir> &&
<dir>/bin/pip install cmake`). libomp is at `/opt/homebrew/opt/libomp` (23.1.3). Build OUTSIDE the repo.
```sh
export CMAKE_POLICY_VERSION_MINIMUM=3.5      # needed: CMake 4 rejects gmock 1.8's cmake_minimum_required
OMP=/opt/homebrew/opt/libomp; B=/tmp/ls-build  # or a scratchpad dir
cmake -DCMAKE_BUILD_TYPE=Release -DCMAKE_OSX_ARCHITECTURES=arm64 \
  -DOpenMP_libomp_LIBRARY=$OMP/lib/libomp.a -DOpenMP_C_INCLUDE_DIR=$OMP/include -DOpenMP_CXX_INCLUDE_DIR=$OMP/include \
  -S . -B $B
cmake --build $B --target LunarSlicer -- -j 10
$B/LunarSlicer version     # stderr: "Luban_Engine version master"
$B/LunarSlicer ptest       # prints 0..9; out-of-order = OpenMP active
# tests (separate build dir; with BUILD_TESTS=ON the engine target depends on gmock):
cmake -DCMAKE_BUILD_TYPE=Release -DBUILD_TESTS=ON -S . -B $B-tests
cmake --build $B-tests --target build_all_tests -- -j 10 && (cd $B-tests && ctest --output-on-failure)  # 19/19 pass, ~13 s
```
- Without the `OpenMP_*` hints OpenMP is silently skipped ("Could NOT find OpenMP"), the binary still works single-threaded.
- `-Ofast` deprecation and `-Wdeprecated-literal-operator` warnings are harmless noise.
- CI reference: `.github/workflows/build-on-create-release.yml` (on `release: created`: macOS x64/arm64 codesigned,
  Windows MinGW, Ubuntu x64/arm64; runs `version` + `ptest`; uploads `LunarSlicer-<os>-<arch>.zip`).
- `origin/main` builds with conan 1.x (`conanfile.py`, `conandata.yml` "5.3.0") - NOT attempted on this Mac.
- Skills: `lunarslicer-build-test`, `lunarslicer-run-slice`, `lunarslicer-inspect-main-lineage` (in `../.claude/skills/`).

## Directory map
| Path | Purpose |
|---|---|
| `src/main.cpp`, `src/Application.cpp` | entry; command dispatch `connect`(ARCUS only)/`slice`/`support`/`ptest`/`version`/`help` (`:224-252`) |
| `src/communication/CommandLine.cpp` | CLI parser `-v -m -p -j -e -l -o -g -s --next` (`:100-300`), JSON definition loader with `inherits` + `machine_extruder_trains` (`:315-420`), `CURA_ENGINE_SEARCH_PATH` (`:346`) |
| `src/Slice.cpp`, `src/Scene.cpp` | per-command flow: `slice` -> `Scene::processMeshGroup` (`:66`), `support` -> `processMeshGroupSupport` (`:128`) |
| `src/FffSupportGenerator.*`, `src/SupportDataStorage.h` | Snapmaker: builds a support STL by extruding flagged faces down to Z=0 |
| `src/MeshGroup.cpp` | STL loading (binary STL attribute word -> `MeshFace::support_flag`, `:206-207`) and Snapmaker `saveMeshSTL` (`:392`) |
| `src/FffPolygonGenerator.cpp`, `FffGcodeWriter.cpp`, `LayerPlan.cpp` | upstream slicing pipeline (areas -> layer plans -> G-code) |
| `src/gcodeExport.cpp` | every G-code line and header comment (`;FLAVOR`, `;TIME`, `;Filament used`, `;LAYER`, `;TYPE:`) |
| `src/timeEstimate.h` | Snapmaker-tuned print-time model (`:71-76`: 120 mm/s XY, 1000 mm/s^2, jerk 10/0.3) |
| `src/settings/`, `src/infill/`, `src/pathPlanning/`, `src/utils/` | upstream settings store, infill patterns, combing, geometry |
| `libs/`, `cmake/` | vendored clipper/rapidjson; Find modules (OpenMP, Stb, GMock, Polyclipping, Arcus) |
| `tests/` | upstream GoogleTest suites + `tests/integration/resources/cube.stl` |
| `docs/`, `meta/`, `docker/`, `Jenkinsfile`, `CPackConfig.cmake`, `Cura.proto` | upstream leftovers, unused by Snapmaker |

## CLI contract (what Luban relies on)
- Luban: `LunarSlicer slice -v -p -o <out.gcode> -j <configDir>/printing/active_final.def.json -l <model.stl>
  -j <model.def.json> [-l <support.stl> -j <configDir>/printing/support.def.json]` with env
  `CURA_ENGINE_SEARCH_PATH=<configDir>/printing` (`Luban/src/server/slicer/Slicer.ts:30-52,288-306`).
- Settings bind to the last object given: global, `-e<n>` extruder, `-l` mesh, `-g` group, `--next` new group.
  `-s key=value` must be two separate argv entries (`-s`, `key=value`).
- Support meshes from Luban are normal meshes with `support_mesh: true` (`luban-print-settings/resources/printing/support.def.json`).
- Output streams: on 4.8 ALL logging goes to stderr (`src/utils/logoutput.cpp`; `-p` prints
  `Progress:<stage>:<i>:<n> \t<pct>%`). On 5.3 spdlog logs to stdout (`git show origin/main:src/Application.cpp` `:33`), and
  Luban parses stdout for `Processing insets for layer`, `GcodeWriter processing layer`, `Print time (s):`,
  `;TIME:`, `;Filament used:` (`Slicer.ts:173-243`, time x1.07). A 4.8 binary dropped into Luban gives no progress.
- Exit codes: 0 ok, 1 bad CLI/model/output, 2 missing setting/JSON error, 139 crash.

## Snapmaker additions on a250t/main (git diff 83c65a1a HEAD: 22 files, +1320/-123)
- `support` command: `Application.cpp:234` calls `slice()`; `Slice::compute` branches on `cmd` (`Slice.cpp:24-43`).
  With `support_generate_simple=true` it writes a binary STL to `-o` (`Scene.cpp:151-158`); otherwise it slices normally.
  This setting exists in no current definition; Luban's live support button uses the separate legacy binary
  `LubanEngine` (`modelsupport` command, npm `snapmaker-luban-engine` 0.9.1, `Luban/src/server/slicer/slice.js:67`).
- `-o` no longer opens the file in the parser; it is stored as setting `output_path` (`CommandLine.cpp:239`) and opened
  per mesh group in `Scene::processMeshGroup` (`Scene.cpp:71`); `finalize()` moved there (`:125`).
- Observation (unverified impact): `saveFacesSTLBinary` / `saveFacesSTLAscii` iterate `i += 3` (`MeshGroup.cpp:312,364`)
  while the binary header stores `faces.size()`, so only every third face is written under a full-count header.
- `getpath.cpp` off-by-one buffer fix; `timeEstimate.h` defaults lowered for SM2 (affects estimate only).

## Snapmaker additions on origin/main (5.3, shipped) - read with git show
- Painted/colour STL: `MeshGroup.cpp:203-213` reads byte 49 of the STL attribute word; `>= MESH_PAINTING_COLOR (2)`
  marks painted faces (`include/utils/Constant.h:6-7`); `colorful_slicing_enable` splits meshes
  (`FffPolygonGenerator.cpp:199-242`, `slicer.cpp:744,783`, `MultiMaterialSegmentation.cpp`).
- `M109 S<t> C<range> W<time>` from `material_print_temp_wait_range/_time` (`gcodeExport.cpp:1259-1269`,
  `FffGcodeWriter.cpp:574-592`, `LayerPlan.cpp:1749-1795`); the SM2 controller honours C/W
  (`Snapmaker2-Controller/Marlin/src/gcode/temperature/M104_M109.cpp:140-146`, `ENABLE_CUSTOM_M109_PARAM`).

## Conventions
- Namespace `cura`; upstream style (Allman braces, `clang-format` file at root, no leading dot), CamelCase classes, snake_case members.
- Logging (4.8): `logError/logWarning/logAlways/log/logDebug/logProgress` (`src/utils/logoutput.h`); `log` needs `-v`,
  `logDebug` needs `-vv`. 5.3 uses `spdlog::info/warn/error`.
- Settings: `settings.get<T>("key")`; a missing key calls `exit(2)` - every key the engine reads must exist in the
  definitions. Errors are fatal `exit(n)`, no exceptions in the CLI path.
- Snapmaker headers use `LUNARSLICER_SRC_*` / `LUBANENGINE_SRC_LUBAN_*` include guards; commit messages `Feature:`,
  `Fix:`, `Refactor:`, `Build:`, `Improvement:`.

## How to extend
- New engine setting: read it with `settings.get<T>()` where needed, AND add it to
  `luban-print-settings/resources/printing/fdmprinter.def.json` (or `snapmaker_modify_0.def.json`) with a default,
  otherwise every slice exits 2. Expose it in Luban's UI via `Luban/src/app` definitions (ask `sm-luban`).
- New G-code / parameter: emit from `src/gcodeExport.cpp` (one writer method per command, flavour switch on
  `EGCodeFlavor`); confirm the controller parses it (ask `sm-controller`).
- New CLI command: add a branch in `Application::run` (`:224-252`) and, if it slices, a `cmd` branch in `Slice::compute`.
- Test a rebuilt engine in Luban (unverified recipe): back up and replace
  `Luban/node_modules/@snapmaker/snapmaker-lunar/engine/MacOS/LunarSlicer`; it must be a 5.3-lineage build.

## A250T notes
Nothing T-specific exists or is needed here. A250 vs A250T differences (TMC linear modules, leads) live in firmware;
the slicer only sees `machine_width/depth/height` and start/end G-code from `luban-print-settings/resources/printing/a250_single|a250_dual/machine.def.json`
(machine_name "Snapmaker A250"). `timeEstimate.h` limits are generic SM2 guesses, not A250T-measured.

## Cross-repo interfaces (full map: `../.claude/knowledge/DEPENDENCIES.md`)
- Luban (`sm-luban`): spawns the binary, writes `active_final.def.json`, parses progress, prepends header
  `;Header Start ... ;Slicer:LunarSlicer` (`Luban/src/server/slicer/post-processor.ts:159`); STL attribute bytes written by
  `Luban/src/shared/lib/STL/STLExporter.js:116,132`.
- luban-print-settings (`sm-luban-print-settings`): all `.def.json` inputs (`resources/printing/`).
- npm `@snapmaker/snapmaker-lunar` (external; binaries for LunarSlicer, LunarMP (`sm-lunarmp`), closed LunarTPP).
- Snapmaker2-Controller (`sm-controller`) and touchscreen (`sm-screen`) consume the G-code (Marlin flavour, M109 C/W).
- Not related: OrcaSlicer/snapmaker-profiles (Orca has its own slicer), SACP, Cura plugin (Cura has its own CuraEngine).

## Do NOT
- Push, change remotes, or retarget `a250t/main` to `origin/main` without the owner deciding.
- Build inside the repo tree except in `build/` (ignored by `.gitignore`); prefer an out-of-tree dir. Never commit build outputs or G-code.
- Overwrite the binary in `Luban/node_modules` without a backup; never replace it with a 4.8 build.
- Flash anything: this repo produces no firmware.
- Treat `docs/`, `README.md`, `Changelog.md` as Snapmaker docs (they are upstream Cura).
