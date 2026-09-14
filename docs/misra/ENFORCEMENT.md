# Guideline enforcement plan: initial structure

Status: incomplete draft. No MISRA compliance claim is made.

Proposed baseline: MISRA C:2025, C99 language profile. Confirm with the licensed standard and owner before treating this as an approved compliance plan. An analyzer configured for MISRA C:2012 is a partial historical baseline, not evidence of complete C:2025 coverage.

Production scope: `include/embedded_slam_primitives/**` and `src/esp.c`, separately in external-definition and header-only configurations. Hosted tests, fuzzers, benchmarks and publication/build scripts are support tools and require their own review; do not silently count their exclusions as production coverage.

The accompanying CSV contains local engineering policies, not a licensed rule list or full MISRA matrix. Replace the mapping-pending entries with the exact selected guideline inventory, categories, tool diagnostic mappings and manual checks. Record tool version, compiler model, predefined macros, include paths, selected configurations and whole-program limitations for every run. Include the application integration, not just this library.

Required artifacts before a claim: approved guideline enforcement plan; justified recategorization where applicable; approved deviation records; review evidence for non-automatable requirements; guideline compliance summary; evidence linking every finding to disposition. An empty finding log is not proof that the analyzer was configured correctly.

Candidate review topics: API macro/linkage selection; public caller-owned descriptors; non-finite coordinate validation; multiple early returns; enum/Boolean essential-type treatment; `size_t` arithmetic; pointer/array lifetime and extent preconditions. These are review topics, not pre-approved deviations or verified rule violations.

Run `scripts/run_cppcheck.sh` for general checks. Run `scripts/run_misra.sh /absolute/path/to/approved-addon.json` only with an approved installed analyzer configuration. The latter intentionally fails when configuration or Cppcheck is absent. Never commit proprietary rule text or suppress all findings to make CI green. A tool's rule-coverage claims must be checked against the chosen edition.
