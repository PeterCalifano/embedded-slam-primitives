# Assurance gaps

No licensed MISRA rule inventory has been applied. No complete MISRA run or compliance summary exists. Independent design/code review, target qualification, MC/DC applicability, compiler/tool qualification and application hazard analysis remain open. Valgrind, Cppcheck and target execution are recorded separately in verification/REPORT.md; configured CI jobs are not completed checks.

Cross-pool/session handle discrimination and transactional multi-feature frame updates are not part of the initial kernel. The caller must honor the ownership, lifetime and failure contracts in DESIGN.md. Covisibility, feature sets and optional metadata are design-only. These gaps must not be represented as implemented capabilities.
