# Delivery and private-repository handoff

## What was and was not done

Prepared on 2026-09-08. The connected GitHub service was used to inspect both specified repositories and record exact source baselines. The template's reusable layout, CMake packaging, testing and profiling intentions were retained selectively, while the C++/CUDA/middleware/wrapper build was replaced with a small C-only build. This is a **fresh Git repository derived from inspected conventions**, not a complete template clone or GitHub template-generated repository. The source manifest records that distinction.

The design was committed first (`f5dd186`, full hash available in the bundle). The second commit adds the executable kernel and verification scaffolding. A final evidence commit records actual local execution and this handoff. Commits identify the assistant rather than impersonating the owner.

**No remote repository was created and no source was pushed.** The connected GitHub actions exposed reads but no repository-creation or code-write operation. The local environment also lacked an authenticated GitHub CLI. Neither upstream repository was changed.

## Deliverables

`embedded-slam-primitives.zip` is the tracked source snapshot with documentation, C code, tests, scripts, CI configuration and verification evidence. Build directories, tool installations, private credentials and unrelated files are excluded. The ZIP does not contain Git history.

`embedded-slam-primitives.bundle` contains the complete local Git history, including the design-first commit. Cloning this local bundle normally creates an `origin` remote pointing to the bundle file; remove that local-file remote before using the publication script.

```sh
git clone embedded-slam-primitives.bundle embedded-slam-primitives
cd embedded-slam-primitives
# The bundle is a LOCAL FILE. This removes only its local-file remote reference.
git remote -v
git remote remove origin
./scripts/verify.sh
```

The verification script may require installation of local compiler/build tools. It does not fetch source dependencies. The artifact archive is suitable for inspection without Git, but use the bundle to preserve the design-first history.

## Publish the requested private repository

Install GitHub CLI and authenticate as `PeterCalifano` using its supported authentication workflow. Do not paste tokens into this repository. Then run:

```sh
./scripts/publish_private.sh
```

The script fixes the destination to `PeterCalifano/embedded-slam-primitives`, checks the authenticated login, rejects existing remotes and an existing destination, creates it with `--private`, queries privacy before any push, and pushes the current commit to `main` without forcing. It checks privacy again after the push. It does not inherit upstream branch protection, permissions, secrets or release settings. It configures Git to use the authenticated GitHub CLI credential helper; it does not embed credentials in the remote URL.

If creation succeeds but a later network or authentication step fails, inspect the destination and local remotes manually. The script deliberately refuses to overwrite/reuse an existing repository on a retry. **The script has syntax/ordering checks but has not been exercised against live GitHub here.** Its guarded create/push sequence follows the official CLI model, not a claim of a completed remote operation.

After publication, review collaborator access and Actions settings, and establish branch protection/rulesets requiring review and the intended CI checks. Do not enable public Pages or public artifact deployment for this private project. GitHub CI configuration exists but actual service execution is still a separate gate.

## Engineering acceptance before expanding scope

Review the design's memory, aliasing, lifetime, stale-handle, monotonic-frame and failure-atomicity contracts. Confirm target CPU/compiler, C dialect, floating-point support, maximum tracks/history, memory budget, scheduling ownership and applicable assurance standard. The default validation ceilings are not mission requirements.

Approve the API as a selective C redesign. Then implement the standalone feature set and optional bounded covisibility layer against independent reference models, explicitly testing duplicate insertion and ring rollover. LiDAR/label sidecars follow only when their measurement/ownership contracts are agreed. KLT remains outside the library.

Close the full selected-MISRA-edition inventory, analyzer coverage and formal deviations. Execute Valgrind/C++-free static analysis gates on a host that has the tools, then perform target-specific stack, memory, timing, integration and compiler-runtime verification. Host dynamic tests and analyzer output do not establish flight readiness.

## Primary operational reference

GitHub CLI repository creation: https://cli.github.com/manual/gh_repo_create (checked 2026-09-08).
