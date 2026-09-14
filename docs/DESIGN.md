# embedded-slam-primitives: C-only design baseline

**Status:** proposed architecture and initial executable kernel, not a qualified or MISRA-compliant product.  
**Date:** 2026-09-08. **Intended repository:** `PeterCalifano/embedded-slam-primitives`, PRIVATE.  
**Delivery boundary:** the design is committed before implementation. The remote repository was not created from this environment. See `docs/HANDOFF.md` and `verification/REPORT.md` for the actual delivery and execution status.

## 1. Purpose and scope

Provide deterministic, bounded-memory C data structures supporting a feature-tracking application. This is a selective redesign of `slam-primitives`, not a mechanical C++ translation and not a new optical-flow implementation. The upstream library represents observations, feature sets, temporal tracks, track bundles and sliding-window covisibility. It does not implement KLT, feature detection, image pyramids, RANSAC or a SLAM optimizer [S1-S7]. Those algorithms remain external producers/consumers.

The first implemented kernel comprises typed observations, bounded tracks, a fixed-capacity track pool, lifecycle transitions, stale-handle checking, bounded queries and explicit errors. Standalone feature sets, covisibility, LiDAR metadata and backend labeling have designs below but are not claimed implemented. Production readiness requires independent review, requirements closure, full selected-rule analysis and target evidence.

## 2. Inspected source baseline

| Repository | Inspected branch | Commit |
|---|---|---|
| `PeterCalifano/slam-primitives` | `develop` | `d4e3cca4896c1b299625fecdb5f359e4b6e73704` |
| `PeterCalifano/cpp_cuda_template_project` | `main` | `12041acb19433dfe1b98b34203f4721b5a568797` |

`docs/source-manifest.json` records inspected file blob hashes. This is an independently initialized repository derived from the reviewed template conventions, not a GitHub-generated template copy or a complete historical clone. Original repositories are not modified. Retain the upstream MIT notice. Repository visibility and software licensing are separate decisions; the retained MIT license does not make the repository public.

### Functional migration map

| Upstream component | C replacement | Change in contract |
|---|---|---|
| `SFeatureLocation2D` | `esp_point2` with `double u, v` | Preserve pixel-coordinate precision; reject non-finite input |
| `FrameID`, `SetID` | `int32_t`, `uint32_t` aliases | Negative frame IDs and zero set ID rejected; no counter wrap |
| `CFeatureSet` | Planned typed bounded point buffer | One count, no inheritance, no exception or ambiguous Boolean result |
| `CFeatureTrack` | `esp_track`, `esp_observation` | One count for paired point/frame observations; increasing frame IDs; explicit state |
| `CFeatureSetBundle` | `esp_pool`, typed slot and observation arrays | No vector/hash map; stable slot handles plus monotonically allocated IDs |
| `CCircularBuffer` | Planned typed frame ring | Checked access; explicit eviction result; no untyped generic byte arena |
| `CCovisibilityGraph` | Planned frame ring with sorted ID arrays | Fixed visibility capacity, deterministic intersection, no reverse index initially |
| `SLidarEnhancedData` | Planned optional sidecar | Explicit validity and units; absent from minimum track payload |
| `SLabelingEnabled` | Planned bounded label sidecar | Separate, reviewed payload; no generic compile-time policy machinery |
| logger, bindings, ROS overlay | Excluded | Application controls telemetry and transport |

### Source-inspection risks, not upstream patches

The bundle's `std::vector`, `std::unordered_map`, returned vectors and exceptions are unsuitable for this project's no-heap/error-code contract [S4]. Reserving slots is not equivalent to removing all allocations. `next_id_++` does not explicitly guard exhaustion. The C design must not reuse an ID after overflow.

The track inherits a public base `addKeypoint()` while maintaining a second length for frame IDs [S2,S3]. Mixing those entry points can break the point/frame invariant. The C representation stores each pair as one observation with one authoritative count.

The covisibility reverse index appends a logical ring position even when the per-frame feature was already present. Logical positions also change when the oldest frame is evicted [S5,S6]. These are source-level migration hazards, not runtime-tested claims about downstream symptoms. The initial C graph deliberately omits that index; any future index must use stable frame identity or physical slot plus generation, and deduplicate updates.

## 3. Language and packaging decisions

**ADR-001: ISO C99 as the minimum language.** All production sources and test drivers are C, with C extensions disabled. C11 and C17 builds are verification configurations, not additional required language features. C99 provides fixed-width types and `static inline` without requiring C++ or modern target compiler support. Actual mission compiler availability remains an integration gate.

**ADR-002: compiled reference build plus optional header-only mode.** The default is an ordinary static C library. `ESP_HEADER_ONLY` selects internal-linkage `static inline` definitions from the same implementation file. `src/esp.c` uses `ESP_IMPLEMENTATION` for external definitions. The modes are mutually exclusive. No second algorithm implementation, global registry or mutable file-scope state is introduced.

Header-only is a packaging choice, not a safety assurance. The compiled build provides a stable analysis boundary, easier symbol/code-size inspection and simpler per-function profiling. Header-only can duplicate code across translation units and must receive separate multi-translation-unit and optimized-build tests. Function-address identity across translation units is not an API guarantee. Do not mix linkage modes for one logical library integration. Both modes require one consistent project configuration.

**ADR-003: preserve `double` initially.** The upstream stores doubles [S7]. A float/fixed-point profile would alter precision, layout and validation assumptions. It is deferred until the target's FPU, accuracy budget and profiling data justify it. Do not silently change precision merely to reduce memory.

## 4. Memory ownership and representation

The caller owns every context and typed storage array. Initialization and runtime operations make no heap calls. No VLA, recursion, untyped arena, pointer-punning, callback-based allocation, blocking I/O or runtime dependency discovery is permitted in the kernel. Do not put a large pool on a small task stack: declare application-owned static storage or allocate a typed region before library use according to the system's policy.

An observation is a point/frame pair. A track descriptor borrows a contiguous observation array and stores capacity, count and lifecycle state. A pool borrows an array of slots and a flat observation slab partitioned into equal-capacity track segments. The descriptor does not own these arrays and must not outlive them. Caller must not modify descriptor fields, stored observations or slots behind the API. Sharing arrays between live contexts is forbidden.

Initialization checks non-null required pointers, nonzero capacities, configured upper bounds, `size_t` multiplication overflow and slab length **before modifying any object**. These checks cannot determine whether a non-null pointer actually references a live allocation of the claimed size. Allocation extent, alignment, lifetime, non-overlap and initialized descriptor values are caller preconditions. C cannot portably validate arbitrary forged, dangling or uninitialized pointers.

For S slots and L observations per track, persistent storage is:

`sizeof(esp_pool) + S*sizeof(esp_pool_slot) + S*L*sizeof(esp_observation)`.

No unbounded scratch storage is needed by the initial kernel. Query outputs are caller-supplied arrays. The `benchmarks/bench_pool.c` report prints actual `sizeof` values. As an illustrative ABI assumption only, a 24-byte double observation gives 98,304 bytes for 128 x 32 observations, before descriptors. The default hard upper limits (512 tracks, 128 observations per track) are limits, not automatically allocated arrays or a recommended embedded configuration. Choose capacities from target RAM and workload requirements.

## 5. Identity, lifetime and bounded execution

**ADR-004: public IDs plus validated handles.** Preserve a monotonically increasing nonzero 32-bit set ID, but use `esp_handle {slot, id}` for hot-path lookup. Lookup checks the slot bound, occupancy and exact ID. A stale handle after release/reallocation therefore fails instead of referring to a new track. Allocation scans slots in increasing index order. It is O(S), deterministic and easy to bound. Handle lookup and release are O(1); compatibility lookup by ID is an explicit O(S) scan, not an advertised O(1) hash lookup.

The final representable ID is allowed once. The pool then sets an exhaustion flag; later allocations fail with `ESP_ERR_ID_EXHAUSTED`, without incrementing or wrapping the counter. A failed allocation consumes neither a slot nor an ID. Releasing a track does not make its ID reusable during that pool lifetime.

Handles are scoped to one owning pool and one initialized lifetime. This initial design does not detect accidental use with a different pool whose slot/ID happen to match. Reinitialization invalidates every external reference and requires application quiescence; it is not an in-flight recovery operation. Cross-pool references, persisted sessions or reset-surviving IDs require a caller-managed pool/session identity before integration. Exhaustion is handled by mission policy, not by silently resetting IDs.

| Operation | Upper bound |
|---|---|
| Track append/get/terminate | O(1) |
| Track lookup by frame | O(L) |
| Pool initialize | O(S), descriptor initialization only |
| Pool allocate/find by ID | O(S) |
| Pool append/get/release by handle | O(1) |
| Collect terminated handles | O(S), two bounded passes |
| Planned frame lookup/intersection | O(W) lookup, O(Fa+Fb) intersection |

These are source-level operation bounds, not measured worst-case execution times. Compiler output, caches, memory placement, interrupts and scheduling remain system concerns.

## 6. Track API contract

A valid track has `0 <= count <= capacity` and exactly `count` paired observations. Capacity is at least one. States are ACTIVE, TERMINATED_MANUAL and TERMINATED_FULL. There is no separate mutable base-class point count.

`esp_track_append()` validates the descriptor, state, nonnegative frame ID, finite coordinates and strict increase of frame ID. Gaps are allowed. Duplicate/out-of-order observations return `ESP_ERR_FRAME_ORDER`; replacing an observation is not implicit. If capacity was reached by this append, the operation returns `ESP_OK` and changes state to TERMINATED_FULL. Subsequent appends return `ESP_ERR_TERMINATED`. There is no rolling overwrite of a temporal track.

Manual termination is idempotent and preserves a pre-existing full termination reason. Track get/find copy one observation to caller output. Invalid queries leave that output unchanged. Finite negative or out-of-image coordinates are permitted: image bounds and tracker-specific acceptance gates belong to the producer. NaN and infinity are rejected. Fast-math/finite-math compiler modes are not allowed because they can invalidate non-finite checks.

Pool APIs expose copied observations and metadata rather than mutable track pointers. Metadata contains ID, count, capacity and state. Scalar mutating operations validate all recoverable errors before committing changes. Slot release invalidates the handle but does not zero the slab. Inaccessible historical values are not a secure-erasure guarantee.

`esp_pool_collect_terminated()` always reports the required number after valid argument checks. When output capacity is too small, it returns `ESP_ERR_CAPACITY`, updates the required-count output, and leaves the handles array unchanged. A null handles pointer is permitted only for a zero-capacity sizing query. Output arguments must not overlap each other or the pool/storage. Iteration order is increasing slot index, not unordered-map or numeric-ID order.

A multi-feature application update is **not** automatically transactional: earlier successful appends remain after a later failure. Applications needing all-or-nothing frame updates must prevalidate and stage bounded input, or use a future explicitly transactional batch API. This distinction must not be hidden by the wrapper.

## 7. Feature-tracker integration boundary

An external detector creates a point and requests a pool handle. An external tracker computes the next observation, applies image bounds/quality/geometry gates, and calls the pool append API. A lost feature is manually terminated. Reaching length capacity terminates automatically; the application may start a successor track and preserve association in a separate label map.

A backend consumes terminated tracks, acknowledges completion and only then releases their slots. It must not retain raw observations after the application reuses storage. Backpressure when the pool is full is explicit: skip creation, prioritize candidates externally or raise a monitored fault. The library never silently evicts an active/undelivered track. The sample application demonstrates this lifecycle without embedding an optical-flow algorithm.

No ROS, GTSAM, Eigen, OpenCV, CUDA or transport dependency belongs in this kernel. An eventual Hera/RTEMS/LEON adapter would be a separate integration profile after confirming actual ABI, compiler, RAM, stack, FPU and upload constraints. This baseline does not assert mission compatibility.

## 8. Planned covisibility module

Retain W frame entries, each holding at most F distinct set IDs in sorted ascending order. Bind all arrays from caller-owned typed storage. Reject negative/duplicate/out-of-order frame IDs. `push_frame()` returns whether eviction occurred and which frame ID was evicted; track data are never released as a side effect.

For `add_visibility()`, validate IDs and capacity before mutation. The simplest bounded baseline requires sorted unique input, performs a two-pointer preflight union count, then merges into caller-provided F-element scratch storage and commits. A future convenience normalizer must document sorting cost and scratch requirements. Do not partially insert and then report capacity failure.

Intersection returns sorted unique IDs through a caller buffer with the same required-count/no-partial-output convention as pool queries. Missing-frame and empty-intersection outcomes are distinguishable. Scanning W frames is acceptable until profiling proves otherwise. A reverse index is deliberately absent initially: maintaining index correctness during ring eviction is extra mutable state with no demonstrated first-release need.

Visibility describes retained history. Releasing a pool track must not automatically delete historical visibility. If an application wants pruning, provide an explicit, documented prune operation; this intentionally separates lifecycle from the upstream convenience cleanup behavior. Randomized model tests must include more than W successive evictions, duplicate links, slot reuse, capacity failures and historical-ID retention.

## 9. Optional metadata and feature sets

A standalone feature set will use a typed point buffer with explicit capacity/count and will not be an inheritance substrate for tracks. LiDAR payloads preserve range [m], azimuth [rad] and elevation [rad], with explicit validity. Plausibility bounds and sensor reference frames must be specified before adding them. Backend landmark labels use bounded optional sidecars so the minimum tracking configuration pays no per-track payload cost. No `void *` metadata or arbitrary generic container is planned.

## 10. Safety-oriented development profile

This is a safety-oriented engineering baseline, not a safety classification, certification or claim of MISRA compliance. Proposed guideline baseline: **MISRA C:2025**, subject to the owner's licensed standard and selected analyzer. Perforce's published 2026.2 enforcement documentation identifies a C:2025 rule set and explicitly distinguishes enforceable and non-statically-enforceable rules [T1]. The project must freeze the exact edition/amendments, language implementation and enforcement coverage instead of assuming that an older Cppcheck MISRA add-on covers that edition.

The initial `docs/misra` artifacts define local policies, analysis boundaries, a gap register and a deviation template. They are not a complete rule-by-rule compliance matrix. Required completion: licensed rule inventory, guideline enforcement plan, any recategorization plan, approved deviations and a guideline compliance summary. Manual review remains necessary. Never commit proprietary rule text, analyzer licenses or credentials.

Project policy forbids runtime heap, recursion, VLAs, unchecked indexing, counter wrap, type-punned storage, hidden global state, process termination, ignored status values and public function-like utility macros. Use explicit types, prototypes, `const`, braces, small functions and status returns. Do not assert that a strict compiler build or clean sanitizer run establishes MISRA compliance.

`docs/misra/ENFORCEMENT.md` separates production headers/implementation from hosted tests, benchmarks and build tools. Header-only definitions and preprocessor configurations are in analysis scope. Tool-specific suppressions require a narrow location, rationale, reviewer and expiry; broad silent suppressions are not acceptable.

## 11. Verification architecture

Run GCC and Clang, Debug and optimized configurations, compiled and header-only modes. Include C99, C11 and C17 compilation checks, multi-TU header-only linking, installed-package consumers and no-C++ dependency checks. Unit tests must use checks that remain active under `NDEBUG`.

ASan detects classes of memory misuse; UBSan detects selected undefined behavior [T2,T3]. They are host diagnostics, not target runtime requirements. Use combined ASan+UBSan in one build, separate unsanitized builds for Valgrind, and separate MSan only with a fully supported instrumentation environment [T4]. TSan is deferred because the kernel has no concurrent API contract. Independent pools may be used concurrently only with disjoint storage; access to the same pool requires external synchronization. Do not infer ISR safety from absence of locks.

Test capacity 1, hard maximum, empty/full transitions, missing IDs, stale handles, invalid slots, duplicate/out-of-order frames, zero/negative frame boundaries, NaN/infinity, output buffer shortage, identity exhaustion, initialization overflow and reuse. State-machine/property tests compare each operation with a small independent reference model. Fuzz API operation sequences over **valid allocated objects**, not arbitrary forged pointers. Coverage must report production branches and justified gaps; no numerical coverage target is a certification claim.

`verification/REPORT.md` distinguishes executed checks from prepared-but-unavailable tools. CI is a configuration until an actual remote run exists. A licensed analyzer run is a release gate, not a fabricated green checkbox.

## 12. Profiling and optimization plan

Memcheck is used for memory/initialization diagnostics [T5]. Cachegrind and Callgrind provide instruction/cache simulation and call-cost evidence [T6,T7]. Their results are host profiling evidence, not hardware WCET or flight-CPU cache behavior. Run them on a non-sanitized optimized binary with symbols. Debug Memcheck and optimized profiling are separate jobs.

The initial deterministic pool benchmark exercises allocation, append, termination, query and release with checked return values and an observable checksum. It prints configuration and layout sizes. The profiling script records tool/compiler/system information and writes versioned output files. A baseline measured in this environment must never be invented when Valgrind is absent.

Before optimizing, sweep S/L, occupancy, churn and query frequency. Record text/rodata/data/bss sizes, compiler `.su` stack estimates, instruction counts and, on the real target, cycles and measured latency distribution. Candidate changes include a free-slot stack, fixed-capacity ID map, AoS/SoA layout or optional precision profiles. Every change needs regression/equivalence evidence and a memory/timing tradeoff. Keep host-native tuning, fast-math, FMA assumptions, auto-vectorization-specific APIs and LTO out of the default qualification profile. Target-specific optimization is opt-in and separately verified.

## 13. Template reduction

Retain the template's CMake-package concept, namespaced consumer targets, out-of-source build guard, tests, install/consumer verification and MIT attribution. Rewrite CMake to enable **only C**, use target-scoped flags, and avoid automatic downloads or global compiler mutation [S8,S9].

Remove CUDA/OptiX/TensorRT, Eigen/MKL, TBB/OpenMP, OpenGL, Python/MATLAB wrappers, gtwrap, ROS/colcon, Catch2, GPU/ROS workflows, allocator/profiler dependencies, complex version discovery, machine-specific editor state and GPU devcontainer scripts. New C-only tests use a tiny local harness. No public documentation deployment, release publication or telemetry is enabled for the private repository.

## 14. Requirements and implementation gates

| ID | Requirement | Initial evidence / next gate |
|---|---|---|
| R-C-01 | C99-only production kernel; no third-party runtime | Compiler matrix, exported consumer, source and symbol review |
| R-M-01 | No heap in initialization or updates | Source policy, undefined-symbol inspection, future full program audit |
| R-M-02 | Bounded caller-owned storage | Capacity/overflow tests, caller contract review |
| R-I-01 | No ID reuse within pool lifetime | Exhaustion and stale-handle tests |
| R-T-01 | Paired, ordered observations | Track and model tests |
| R-E-01 | Explicit failures; documented non-mutation | Error/output-boundary tests |
| R-D-01 | Bounded operations and deterministic ordering | Complexity review and slot-order tests |
| R-Q-01 | Both linkage configurations verified | Unit and multi-TU builds |
| R-Q-02 | Sanitizer and static-analysis evidence | Host runs; analyzer coverage gaps remain explicit |
| R-P-01 | Optimization based on evidence | Reproducible benchmark; actual Valgrind and target runs pending |
| R-G-01 | Correct bounded covisibility | Design only, implementation and model tests required |
| R-S-01 | Safety claims supported by complete evidence | Not satisfied; owner/system assurance work remains |

Gate D0: review this design, language/edition, limits and lifecycle. Gate D1: complete foundational tests and independent code review. Gate D2: implement feature sets and covisibility against the specified contracts. Gate D3: port application traces and differential tests, recording intentional semantic differences. Gate D4: run complete static/MISRA analysis and resolve findings. Gate D5: target memory/stack/WCET integration and safety-case review. The initial kernel is not a substitute for these gates.

## 15. Open integration decisions

Target CPU/ABI and actual compiler; maximum observations/features/frame window; RAM and stack budgets; allowed diagnostic/telemetry transport; session/reset identity; thread/ISR ownership; retention policy for backend-delivery backpressure; numeric accuracy budget; safety standard and assurance level; exact MISRA edition/tool coverage; requirement for MC/DC and tool qualification. These are recorded rather than guessed. They do not block a bounded host-side design baseline.

## 16. Sources

Source links identify inspected baselines, not an assertion of complete upstream review.

- [S1] [Upstream README](https://github.com/PeterCalifano/slam-primitives/blob/d4e3cca4896c1b299625fecdb5f359e4b6e73704/README.md).
- [S2] [CFeatureTrack](https://github.com/PeterCalifano/slam-primitives/blob/d4e3cca4896c1b299625fecdb5f359e4b6e73704/src/slam_primitives/feature_sets/CFeatureTrack.h).
- [S3] [CFeatureSet](https://github.com/PeterCalifano/slam-primitives/blob/d4e3cca4896c1b299625fecdb5f359e4b6e73704/src/slam_primitives/feature_sets/CFeatureSet.h).
- [S4] [CFeatureSetBundle](https://github.com/PeterCalifano/slam-primitives/blob/d4e3cca4896c1b299625fecdb5f359e4b6e73704/src/slam_primitives/bundle/CFeatureSetBundle.h).
- [S5] [CCovisibilityGraph](https://github.com/PeterCalifano/slam-primitives/blob/d4e3cca4896c1b299625fecdb5f359e4b6e73704/src/slam_primitives/covisibility/CCovisibilityGraph.h).
- [S6] [CCircularBuffer](https://github.com/PeterCalifano/slam-primitives/blob/d4e3cca4896c1b299625fecdb5f359e4b6e73704/src/slam_primitives/containers/CCircularBuffer.h).
- [S7] [SFeatureLocation2D](https://github.com/PeterCalifano/slam-primitives/blob/d4e3cca4896c1b299625fecdb5f359e4b6e73704/src/slam_primitives/types/SFeatureLocation2D.h).
- [S8] [Template CMakeLists](https://github.com/PeterCalifano/cpp_cuda_template_project/blob/12041acb19433dfe1b98b34203f4721b5a568797/CMakeLists.txt), inspected lines 1-230.
- [S9] [Template compiler flags](https://github.com/PeterCalifano/cpp_cuda_template_project/blob/12041acb19433dfe1b98b34203f4721b5a568797/cmake/HandleCompilerFlags.cmake).
- [T1] [Perforce QAC MISRA C:2025 enforcement documentation](https://help.perforce.com/qac/enforcement/doc/MISRA_MC25CM.html). MISRA's own main site could not be fetched in this session; the licensed standard must be consulted for normative interpretation.
- [T2] [Clang ASan](https://clang.llvm.org/docs/AddressSanitizer.html).
- [T3] [Clang UBSan](https://clang.llvm.org/docs/UndefinedBehaviorSanitizer.html).
- [T4] [Clang MSan](https://clang.llvm.org/docs/MemorySanitizer.html).
- [T5] [Valgrind Memcheck](https://valgrind.org/docs/manual/mc-manual.html).
- [T6] [Valgrind Cachegrind](https://valgrind.org/docs/manual/cg-manual.html).
- [T7] [Valgrind Callgrind](https://valgrind.org/docs/manual/cl-manual.html).
- [T8] [Cppcheck manual](https://cppcheck.sourceforge.io/manual.html).
- [T9] [GitHub CLI private repository creation](https://cli.github.com/manual/gh_repo_create).
