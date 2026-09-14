# Initial verification report

**Date:** 2026-09-08. **Scope:** initial C track/pool kernel, not a full upstream port or safety qualification.  
**Code/scaffold commit:** `d1151808ae29a61a12407596f99f48e5ebb97b0e`.  
**Design-first commit:** `f5dd186f6d16adff5a31111b6527353ad05fd398`.  
This report and logs are added in a later evidence-only commit. Kernel source hashes are in `source-sha256.json`. The original source baselines are in `docs/source-manifest.json`.

## Executive result

The local main matrix passed **44 CTest executions** across seven host configurations and two installed-consumer tests. These are repeated executions of a small test suite, **not 44 different requirements or independent test cases**. The separate coverage configuration passed six additional CTest executions. A Clang libFuzzer run completed **50,000 runs** with AddressSanitizer and UndefinedBehaviorSanitizer enabled and no reported failure. GCC and Clang static-analyzer commands completed with no diagnostics.

Valgrind, Cppcheck and a licensed/full-coverage MISRA analysis were **not run**. GitHub Actions has **not run remotely**. No target hardware was tested. No performance improvement, WCET, complete rule compliance or safety certification is claimed.

## Actual execution matrix

| Check | Actual result | Evidence |
|---|---|---|
| GCC C99 Debug and Release | Six CTest tests per configuration passed | `logs/host-matrix-final.log` |
| Clang C99 Debug and Release | Six CTest tests per configuration passed | Same |
| GCC C11 and C17 Release | Six CTest tests per configuration passed | Same |
| Clang C99 Debug + ASan/UBSan | Six CTest tests passed; no sanitizer report | Same |
| Installed compiled and header-only consumers | Two CTest tests passed | Same |
| Header-only multi-translation-unit linkage | Included in each six-test configuration; passed | Same |
| No heap-allocation symbols in unsanitized archive | Guard passed for that archive | `scripts/verify.sh`, matrix log |
| Fast-math rejection | Negative compile failed with expected diagnostic | Same |
| GCC `-fanalyzer` | External implementation: no diagnostic | `logs/compiler-analysis.log` |
| Clang static analyzer | External implementation and header-only contract/model TUs: no diagnostic | Same |
| ASan/UBSan/libFuzzer | 50,000 runs completed | `logs/fuzz-50000.log` |
| Coverage configuration | Six CTest tests passed; measured coverage below | `logs/coverage.log` |
| Native benchmark functional execution | Completed; checksum 82323200 | `logs/benchmark.log` |
| Publication script | Shell syntax and mocked privacy-refusal guard passed; no remote test | `logs/publication-guard.log` |

The six-test suite comprises compiled/header-only contract tests, compiled/header-only model tests, header multi-TU linkage and the lifecycle example. Each model-test executable performs 20,000 deterministic operations from the same reference-model seed; repeated configurations are not different random campaigns. Fuzzer seed was **3165653816**. Its run began with an empty corpus; the coverage-guided campaign is not an exhaustive adversarial test. `scripts/fuzz.sh` preserves a local corpus for subsequent runs, so exact results also depend on starting corpus and tool version.

The host compilers were GCC 14.2.0 and Clang 17.0.0; CMake 3.31.6. Full versions and host architecture are in `logs/environment.txt`. Source code and host toolchain do not constitute an independently qualified verification environment.

## Coverage and remaining paths

GCC gcov measured the **compiled kernel implementation** (`include/embedded_slam_primitives/detail/esp_impl.h`) under the Debug coverage test suite:

- Executable lines: **97.71% of 218**, i.e. 213 covered and five uncovered.
- Branch outcomes taken at least once: **90.22% of 184**, i.e. 166 covered and 18 not taken.
- All branch sites were reached, reported by gcov as “Branches executed: 100%”. **This is not 100% branch-outcome coverage.**

These numbers do not describe the entire repository, header-only instantiations, fuzz campaign, MC/DC or target coverage. Exact annotated output is retained in `logs/esp_impl.h.gcov.txt`.

Uncovered executable returns at the code baseline are line 178 (observation-count multiplication overflow), line 237 (no free slot despite a non-full count), line 247 (occupied handle but zero active count), line 296 (invalid track metadata through get-info) and line 345 (invalid track metadata while collecting terminated tracks). The first path is unreachable under the default caps on this 64-bit host; the latter four concern externally inconsistent descriptors. They must receive configuration-aware justification, appropriate fault-injection tests or redesign during assurance closure, not blanket suppression. Additional individual short-circuit outcomes remain uncovered.

## Memory and profiling baseline

The deterministic workload exercises 128 slots, 32 observations per slot and 100 complete allocate/append/query/release cycles. On this host ABI:

| Item | Measured value |
|---|---:|
| `sizeof(esp_observation)` | 24 bytes |
| `sizeof(esp_pool_slot)` | 32 bytes |
| `sizeof(esp_pool)` | 32 bytes |
| Caller-owned persistent storage for this workload | 102,432 bytes |
| Unsanitized GCC RelWithDebInfo kernel archive object text | 3,390 bytes |
| Kernel archive object data/bss | 0 / 0 bytes |

The last row does not mean the application uses no RAM: the caller owns the slot and observation arrays. The archive text value is one host object, not whole-application flash usage. Size depends on ABI, compiler, configuration and linkage. Compiler-generated function stack-use entries are included in `logs/host-stack-usage.txt`; they are **not a worst-case call-chain stack bound**.

**No Valgrind profile or timing result exists in this evidence package.** The native workload was run for correctness and size only. `scripts/profile.sh` supplies a separate unsanitized Callgrind/Cachegrind run with compiler/system/version, size, checksum and profiler reports. `scripts/run_valgrind.sh` supplies Memcheck separately from sanitizer instrumentation. Optimization decisions must await actual traces and target measurements.

## Not executed or not established

| Gate | Status / reason |
|---|---|
| Valgrind Memcheck, Callgrind, Cachegrind | Tool unavailable locally; reproducible scripts and CI jobs supplied |
| Cppcheck | Tool unavailable locally; script/CI job supplied |
| MISRA C:2025 full applicable-rule/directive assessment | Normative inventory, tool mapping, approved deviations and independent review remain open |
| MemorySanitizer | CMake mode available, but suitable fully instrumented environment not validated or run |
| Target cross-compilation, FPU/runtime checks, WCET and stack closure | Target and assurance requirements not specified |
| Covisibility, standalone feature-set, labeling and LiDAR tests | Components not implemented in this kernel |
| Complete source-to-C differential validation | Independent pool model exists; no full upstream implementation comparison |
| GitHub repository creation / CI / branch protection | No remote write capability used; all remain unperformed |

## Reproduction

```sh
./scripts/verify.sh
./scripts/run_compiler_analysis.sh
./scripts/fuzz.sh 50000
./scripts/coverage.sh

# On a host with the additional tools installed:
./scripts/run_cppcheck.sh
./scripts/run_valgrind.sh
./scripts/profile.sh
# Only after explicitly approving and supplying the licensed tool configuration:
./scripts/run_misra.sh /absolute/path/to/approved-addon.json
```

Read the complete ownership/precondition and error contracts in `docs/DESIGN.md`. All source was AI-assisted and requires independent review. Passing host tests is evidence about those executions, not proof of absence of defects or readiness for safety-critical deployment.
