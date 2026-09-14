# Project working rules

Read docs/DESIGN.md and verification/REPORT.md before editing. Update requirements and contracts before implementing new behavior. Production code and test drivers must remain C99, no C++, heap, VLA, recursion, exceptions, unbounded work or third-party runtime dependencies. Honor caller ownership, ID exhaustion, failure non-mutation and finite-coordinate rules. Keep one implementation for compiled and header-only builds.

Never infer safety/MISRA compliance from test success. Record actual executed checks and missing evidence. Do not commit proprietary standard text, licenses, credentials or local artifacts. Run scripts/verify.sh and record exact compiler/configuration evidence. Add boundary/model tests for every behavior change. Private repository is required; do not enable public pages/releases or push to either upstream repository.
