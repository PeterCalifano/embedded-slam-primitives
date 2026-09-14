# embedded-slam-primitives

C99 data structures supporting bounded-memory feature tracking. **Initial design and executable kernel, not flight-qualified or MISRA-compliant software.** The intended GitHub repository is `PeterCalifano/embedded-slam-primitives`, **private**. Remote publication was not performed during preparation.

Start with [the design](docs/DESIGN.md), [handoff](docs/HANDOFF.md) and [execution evidence](verification/REPORT.md). The design was the first local Git commit. Source baselines and file hashes are recorded in [the provenance manifest](docs/source-manifest.json).

## Implemented scope

The initial kernel provides double-precision 2D observations, frame-indexed bounded tracks, manual/capacity termination, a caller-owned fixed-capacity track pool, monotonic nonzero IDs with exhaustion checks, stale-handle rejection within a pool lifetime, and bounded copy-out queries. Functions return explicit status codes. No dynamic allocation is performed, including initialization. There is no C++ runtime, Eigen, OpenCV, CUDA, ROS, Python wrapper or network-fetched build dependency.

This is **not an optical-flow implementation**: feature detection, KLT and geometric verification are external. Standalone feature sets, covisibility, LiDAR augmentation and backend labeling are **designed but not yet implemented**. See the design's migration matrix and milestones.

## Build and test

Requirements: a C99 compiler; CMake 3.20 or newer; Ninja for the supplied scripts. GCC and Clang are needed for the full host matrix. Header-only use does not require CMake.

```sh
cmake -S . -B build -G Ninja -DCMAKE_BUILD_TYPE=Debug
cmake --build build
ctest --test-dir build --output-on-failure

# Compiler/dialect/build-mode matrix, sanitizers, installed consumers and guards:
./scripts/verify.sh
./scripts/run_compiler_analysis.sh
./scripts/fuzz.sh 50000
```

`CMakePresets.json` also supplies `gcc-debug`, `gcc-release`, `clang-sanitize` and `profile` configure presets. Build and run CTest in the selected preset's binary directory. Project diagnostics are applied to project-owned targets, not exported as consumer warning or sanitizer requirements.

## Integration choices

The **compiled reference** uses `src/esp.c` and `esp::core`. The **optional header-only mode** uses `esp::header_only`, which defines `ESP_HEADER_ONLY=1`; functions then have `static inline` linkage. Both modes use the same implementation and are tested. Header-only means a small header tree, not a generated single-file amalgamation. No translation unit may define both `ESP_HEADER_ONLY` and `ESP_IMPLEMENTATION`.

```cmake
find_package(embedded_slam_primitives CONFIG REQUIRED)
target_link_libraries(my_c_application PRIVATE esp::core)
# Alternative, not an additional requirement:
# target_link_libraries(my_c_application PRIVATE esp::header_only)
```

Without CMake, compile `src/esp.c` and the application with `-std=c99 -Iinclude`, or define `ESP_HEADER_ONLY` before including `<embedded_slam_primitives/esp.h>`. Preserve consistent configuration across translation units. Use explicit target toolchain settings; do not enable fast-math.

[examples/feature_tracking.c](examples/feature_tracking.c) demonstrates the lifecycle with statically allocated typed arrays and checked return codes. [examples/consumer](examples/consumer) verifies both installed integration modes.

## Memory, identity and error contracts

The caller owns the pool descriptor, slot array and observation slab. Storage must remain valid, disjoint and unmodified except through the API. No large automatic buffers are hidden inside the kernel. Default validation ceilings are 512 slots and 128 observations per track; applications supply their actual capacities, so the ceiling itself allocates no memory.

Handles contain a slot and monotonic ID, not a raw track pointer. Releasing and reallocating a slot invalidates its previous handle. Handles are **not globally unique**: ownership must not cross pools, and pool reinitialization requires quiescence and invalidation of all old references. `UINT32_MAX` can be allocated once; subsequent allocations reject ID exhaustion instead of wrapping.

Coordinates must be finite and frame IDs nonnegative and strictly increasing within each track. The append that fills a track succeeds and marks it terminated; later appends fail. Expected input/capacity failures do not partially mutate state. A too-small terminated-ID output buffer returns the required count without partially filling the output. Multi-feature batches are not transactional.

The API cannot establish the validity of arbitrary non-null pointers, repair uninitialized objects, provide memory isolation, or detect every form of external metadata corruption. Different pools can be used independently; shared-pool synchronization belongs to the application.

## Verification and profiling

| Command | Purpose | Preparation status |
|---|---|---|
| `scripts/verify.sh` | Strict C builds, contract/model tests, ASan/UBSan, install/consumer checks | Executed successfully |
| `scripts/run_compiler_analysis.sh` | GCC analyzer and Clang static analyzer | Executed successfully |
| `scripts/fuzz.sh 50000` | C libFuzzer operation sequences with ASan/UBSan | 50,000 runs completed |
| `scripts/coverage.sh` | GCC line/branch coverage of compiled kernel | Executed; gaps documented |
| `scripts/run_valgrind.sh` | Unsanitized host Memcheck | Script provided; tool unavailable here |
| `scripts/profile.sh` | Unsanitized Callgrind/Cachegrind and size/environment reports | Script provided; Valgrind unavailable here |
| `scripts/run_cppcheck.sh` | Additional C bug-finding analysis | Script provided; tool unavailable here |
| `scripts/run_misra.sh /absolute/approved-addon.json` | Explicitly configured partial MISRA tool pass | Not executed; not a compliance gate by itself |

The proposed policy baseline is MISRA C:2025, subject to project approval and licensed normative review. A partial analyzer pass is not MISRA compliance. [docs/misra](docs/misra) includes a preliminary enforcement plan, deviation template and open gaps. Do not claim certification, target WCET, full branch coverage or radiation tolerance from these host checks.

GitHub Actions configuration is included but has not run remotely. It has read-only repository permissions, no public documentation deployment and no source publication step. The runner's packages are not a qualified, version-frozen toolchain.

## Publish privately

Publication requires an authenticated GitHub CLI and local Git repository. The supplied script refuses existing remotes or an existing destination repository, creates the **private** repository, verifies privacy **before pushing**, and never modifies either upstream repository. Read [the handoff](docs/HANDOFF.md) before running it.

```sh
./scripts/publish_private.sh
```

The upstream MIT copyright notice is retained. Private repository visibility and software licensing are separate. AI-assisted implementation and design require independent engineering review before operational use.
