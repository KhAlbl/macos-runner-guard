# macOS Runner Guard Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build and verify a project-neutral toolkit that renders repository-scoped macOS runner guards with bounded automatic cleanup, protected controller generations, deterministic evidence bundles, and explicit trust limits.

**Architecture:** Implement four independently reviewable subsystems in order. Pure contract/render/policy logic comes first; the cleanup runtime consumes that contract; the protected controller lifecycle activates immutable cleanup generations; deterministic archives and adoption documentation package only reviewed, non-secret bytes. Live runner adoption remains a separate project-specific ceremony.

**Tech Stack:** Python 3.9+ standard library for installed hooks/controller; Python 3.9–3.14 for the toolkit; PyYAML 6.0.3 only for host-independent workflow parsing; POSIX/macOS descriptor-relative filesystem APIs; `/bin/bash`; JSON Schema 2020-12; pytest and unittest; GitHub Actions on hosted Ubuntu and macOS.

**Spec:** [Approved architecture](../../ARCHITECTURE.md)

## Global Constraints

- Keep the implementation repository-neutral: no private project name, private username, live runner ID, registration material, or private host path may enter source, examples, fixtures, archives, or logs.
- Do not create, register, start, stop, relabel, remove, or modify a live runner while executing these plans.
- Do not create macOS accounts, root services, GitHub Apps, JIT controllers, runner groups, LaunchDaemons, tokens, sudoers rules, signing identities, releases, tags, packages, or attestations.
- Installed job hooks and the cleanup/activation controller must run with `/usr/bin/python3 -I -S` and import only the Python standard library.
- Cleanup must use descriptor-relative, no-follow operations and fail closed when the capability probe cannot prove the required primitives. There is no path-recursive fallback.
- Every filesystem rejection test snapshots its fixture and proves all non-target bytes and metadata remain unchanged.
- A SHA-256 value is an integrity identifier, not a signature or authentication claim.
- Every implementation task follows red → observed failure → smallest passing change → focused regression → commit. Do not combine tasks to save commits.
- Never weaken a rejection, budget, workflow guard, or transition invariant to make a test pass.
- The active `main` ruleset requires pull requests and blocks deletion and non-fast-forward updates with no bypass actors. Do not push directly to `main`.

## Frozen Cross-Plan Decisions

| Decision | Frozen value |
|---|---|
| Initial toolkit version | `0.1.0` |
| Package import root | `runner_guard` |
| Contract schema ID | `urn:macos-runner-guard:schema:instance:v1` |
| Contract version value | `macos-runner-guard.instance.v1` |
| Generation manifest version | `macos-runner-guard.generation.v1` |
| Artifact manifest version | `macos-runner-guard.manifest.v1` |
| JSON format | UTF-8, sorted keys, separators `,` and `:`, no ASCII escaping, one final LF |
| Runtime Python | macOS CPython 3.9+ after capability proof |
| Toolkit test matrix | CPython 3.9–3.14 on hosted Ubuntu; 3.11 and 3.14 on hosted macOS |
| Workflow parser | `PyYAML==6.0.3`, `yaml.safe_load`, installed only in toolkit/validation environments |
| Validation toolchain | `pytest==8.4.2` and `setuptools==82.0.1`, the latest verified releases supporting Python 3.9; lock generation with `pip-tools==7.6.1` |
| Numeric run ID | `^[1-9][0-9]{0,19}$` |
| Numeric run attempt | `^[1-9][0-9]{0,9}$` |
| GitHub job token | `^[A-Za-z0-9_.-]{1,64}$` |
| Leg token | member of `allowed_legs` and `^[A-Za-z0-9_.-]{1,32}$` |
| Supported architecture values | `ARM64` and `X64` |
| Example cleanup limits | 200,000 entries; depth 64; 53,687,091,200 bytes; 120 seconds |
| Cleanup/free-space defaults | none; all safety limits and `minimum_free_bytes` are required contract values |
| ZIP timestamp | DOS epoch `1980-01-01T00:00:00` |
| ZIP profile | ZIP32, STORE, UNIX creator, regular files only, no extras/comments/descriptors/directories |
| Archive limits | 256 total members including both control files; 8,388,608 bytes/member; 67,108,864 total uncompressed bytes; 83,886,080 total archive bytes |
| Artifact inventories | generated in fresh staging and inside archives; never committed at repository root |
| Runtime availability after adoption | online routinely; stop only for maintenance, quarantine, security incident, special isolation, or reboot qualification |

## Public Module Boundaries

The implementation may add internal helpers, but these interfaces are the contract between child plans:

| Owner | Exact cross-plan interface |
|---|---|
| `contract.py` | `load_contract(path) -> InstanceContract`; `load_contract_bytes(data) -> InstanceContract`; `validate_contract_set(contracts) -> None`; `canonical_contract_bytes(contract) -> bytes`; `validate_job_identity_fields(contract, run_id, run_attempt, github_job, leg) -> tuple[str, str, str, str]` |
| `policy.py` | `load_workflow_bytes(data) -> Mapping[str, object]`; `check_workflow(document, contract) -> tuple[PolicyViolation, ...]` |
| `render.py` | `render_instance(contract, components, template_root, policy_source) -> tuple[RenderedFile, ...]`; `write_rendered_tree(output_dir, files) -> None` |
| `cleanup.py` | `CleanupPhase` owns `before`, `after`, and `workflow`; `run_cleanup(phase, contract, environment) -> CleanupReceipt` |
| `generation.py` | `stage_generation(source, controller_root) -> GenerationBinding`; `verify_generation(root, expected_sha256) -> GenerationBinding` |
| `activation.py` / `transition.py` | `read_active_binding(controller_root) -> GenerationBinding`; `activate_generation(controller_root, binding) -> None`; `recover_transition(controller_root, action) -> RecoveryReport` |
| `archive.py` | `write_canonical_zip(members, destination, policy) -> VerifiedArchive`; `verify_canonical_zip(archive_path, policy) -> VerifiedArchive`; `extract_verified_zip(verified, destination, policy) -> ExtractionReport`, where `VerifiedArchive` owns immutable bounded archive bytes and extraction never reopens the source path |

Types referenced across plans live in the module that owns the behavior; child plans must import rather than redefine them. `contract.py` owns `InstanceContract`, `InstanceLayout`, and `CleanupLimits`; `policy.py` owns `PolicyViolation`; `render.py` owns `ComponentPayload` and `RenderedFile`; `cleanup.py` owns `CleanupPhase` and cleanup receipts; `generation.py` owns generation bindings; `manifest.py` owns payload and manifest records; `archive.py` owns `ArtifactPolicy`, `VerifiedArchive`, and `ExtractionReport`.

The renderer returns exactly these 15 payload members and modes. The ten component inputs are executable `0555` files; the remaining five members are derived from explicit contract, template, and policy-source bytes. Artifact staging, not rendering, later adds `MANIFEST.json` and `SHA256SUMS`.

| Rendered path | Exact mode |
|---|---:|
| `instance.json` | `0444` |
| `activate.py` | `0555` |
| `cleanup_controller.py` | `0555` |
| `job-started.sh` | `0555` |
| `job-completed.sh` | `0555` |
| `install-instance.sh` | `0555` |
| `recover-instance.sh` | `0555` |
| `rollback-instance.sh` | `0555` |
| `remove-instance.sh` | `0555` |
| `restore-instance.sh` | `0555` |
| `residue-audit.py` | `0555` |
| `workflow-guard.yml` | `0444` |
| `workflow-policy.py` | `0555` |
| `ADOPTION_CHECKLIST.md` | `0444` |
| `AGENT_PROMPT.md` | `0444` |

## Plan Set and Dependency Order

1. [Contract, renderer, and workflow policy](2026-08-30-contract-render-policy.md)
2. [Cleanup runtime, hooks, and residue](2026-08-30-cleanup-runtime.md)
3. [Host verification and controller lifecycle](2026-08-30-controller-lifecycle.md)
4. [Deterministic artifacts and adoption](2026-08-30-artifacts-and-adoption.md)

Each child plan produces an independently testable result. Start the next child only after the previous child is green on its exact head and has no unresolved actionable review finding.

## Task 1: Establish the implementation claim and isolated worktree

**Files:** None. Child plan 1 owns the first repository files, including CI, validation locks, and the pull-request template.

- [ ] **Step 1: Create an issue linked to this umbrella plan**

Record the architecture commit, this plan commit, the four child plans, explicit exclusions, and the rule that one child plan is active at a time. Do not put live host evidence in the issue.

- [ ] **Step 2: Create an isolated implementation worktree**

Run:

```bash
git fetch origin
git worktree add ../macos-runner-guard-implementation -b codex/core-implementation-v1 origin/main
cd ../macos-runner-guard-implementation
git status --short --branch
```

Expected: a clean worktree on `codex/core-implementation-v1`; `main` is unchanged.

- [ ] **Step 3: Verify the live contribution boundary before editing**

Read ruleset ID `21842288` through the GitHub API. Require active enforcement on `refs/heads/main`, no bypass actors, and exact `pull_request`, `deletion`, and `non_fast_forward` rules. Record the live ruleset response in the implementation issue without copying account credentials or local host evidence. Stop if the ruleset is absent or weaker.

- [ ] **Step 4: Hand off file creation to child plan 1**

Do not create CI, validation requirements, tests, or templates in this umbrella task. Begin Task 1 of the contract/render/policy child plan, which owns those bytes and their red-first verification.

## Task 2: Execute child plan 1 — contract, renderer, policy

- [ ] **Step 1: Follow every checkbox in the child plan in order**

Use [the contract/render/policy plan](2026-08-30-contract-render-policy.md). Do not begin cleanup code while any child-plan test or review is unresolved.

- [ ] **Step 2: Record the subsystem acceptance evidence**

Required endpoint:

```text
CONTRACT_RENDER_POLICY_ACCEPTED
```

Evidence includes exact head/tree, contract/schema hashes, example render tree hashes, policy mutation totals, supported Python results, and zero unresolved actionable findings.

## Task 3: Execute child plan 2 — cleanup runtime

- [ ] **Step 1: Follow every checkbox in the child plan in order**

Use [the cleanup runtime plan](2026-08-30-cleanup-runtime.md). Run destructive-path tests only inside disposable fixture roots created by the test suite.

- [ ] **Step 2: Record the subsystem acceptance evidence**

Required endpoint:

```text
CLEANUP_RUNTIME_ACCEPTED
```

Evidence includes exact head/tree, capability matrix, rejection mutation count, interruption results, receipt shapes, and cross-instance non-mutation proof.

## Task 4: Execute child plan 3 — protected lifecycle

- [ ] **Step 1: Follow every checkbox in the child plan in order**

Use [the controller lifecycle plan](2026-08-30-controller-lifecycle.md). Use disposable macOS fixtures; never target `/Library/Application Support/macos-runner-guard` during core implementation.

- [ ] **Step 2: Record the subsystem acceptance evidence**

Required endpoint:

```text
CONTROLLER_LIFECYCLE_ACCEPTED
```

Evidence includes every injected interruption point, old-or-new generation proof, rollback and removal results, Bash syntax checks, and unchanged external-fixture proof.

## Task 5: Execute child plan 4 — artifacts and adoption

- [ ] **Step 1: Follow every checkbox in the child plan in order**

Use [the deterministic artifact plan](2026-08-30-artifacts-and-adoption.md). Generated archives are test artifacts only; do not create a GitHub release or package publication.

- [ ] **Step 2: Record the subsystem acceptance evidence**

Required endpoint:

```text
ARTIFACTS_AND_ADOPTION_ACCEPTED
```

Evidence includes two clean-build archive hashes, manifest coverage, adversarial ZIP rejection totals, secret-scan result, and documentation review.

## Task 6: Run the integrated verification gate

**Files:**
- Create: `scripts/validate.py`
- Create: `tests/test_end_to_end.py`
- Create: `tests/test_public_inventory.py`

- [ ] **Step 1: Write a failing end-to-end test**

The test renders two disjoint sample contracts, runs before/after cleanup only within their fixture roots, stages and activates two controller generations, builds both artifact classes twice, and proves project A cannot address project B.

Run:

```bash
python -m unittest tests.test_end_to_end -v
```

Expected: FAIL because the integrated validation driver does not exist.

- [ ] **Step 2: Add the deterministic validation driver**

`scripts/validate.py` invokes module entry points directly, emits a bounded JSON summary without paths containing user data, and returns nonzero on any skipped required gate. It accepts only an output directory that does not yet exist.

- [ ] **Step 3: Add exact public inventory tests**

`tests/test_public_inventory.py` pins package modules, CLI entry points, schemas, templates, example contracts, and documentation files. It rejects an undocumented extra executable or missing planned component.

- [ ] **Step 4: Run full validation**

Run:

```bash
python -m compileall src tests scripts -q
python -m unittest discover -s tests -v
python -m pytest -q
python scripts/validate.py --output "$(mktemp -d)/validation"
git diff --check
```

Expected: every required gate passes; no test is silently skipped except an explicitly platform-gated macOS integration test on Linux.

- [ ] **Step 5: Commit the integration gate**

```bash
git add scripts/validate.py tests/test_end_to_end.py tests/test_public_inventory.py
git commit -m "test: add integrated runner guard verification"
```

## Task 7: Complete independent review without publishing

- [ ] **Step 1: Run focused independent reviews**

Require separate review of:

1. descriptor-relative cleanup and archive extraction;
2. controller activation/recovery/rollback atomicity;
3. workflow and supply-chain policy;
4. public documentation and trust claims.

Every review is bound to the exact head/tree. Actionable findings require a new commit, full affected validation, and a fresh review of the new exact head.

- [ ] **Step 2: Verify the contribution ruleset and PR state**

Confirm the active `main` ruleset still has no bypass actors and includes `pull_request`, `deletion`, and `non_fast_forward`. Once stable CI check names exist, propose a separate governance change to require those checks and at least one approving review. Do not make that strengthening change within an implementation-fix commit.

- [ ] **Step 3: Produce the final handoff**

Return exact base/head/tree, module inventory, dependency lock hash, test totals, mutation totals, archive hashes, manifest hashes, platform matrix, review/thread state, limitations, and a release recommendation.

Required recommendation before separate release authorization:

```text
CORE IMPLEMENTATION MAY MERGE AFTER EXACT-HEAD CI AND REVIEW.
DO NOT PUBLISH AN EXECUTABLE RELEASE OR ADOPT INTO A LIVE PROJECT YET.
```

## Spec Coverage Matrix

| Architecture section | Plan owner |
|---|---|
| §§1–5 goals, non-goals, trust profiles | Umbrella constraints; plans 1 and 4 documentation |
| §§6.1–6.3 source, contract, rendering | Plan 1 |
| §6.4 protected instance/generations | Plan 3 |
| §7 cleanup contract | Plan 2 |
| §8 workflow guard | Plan 1, integrated by plan 2 |
| §9 install/upgrade/rollback/removal | Plan 3; live adoption deferred |
| §10 routine operations | Plans 2 and 4 documentation |
| §11 deterministic artifacts | Plan 4 |
| §§12.1–12.2 unit, rejection, positive tests | All child plans and Task 6 |
| §12.3 real-project qualification | Explicit follow-up, not core implementation |
| §13 adoption-agent contract | Plan 4 |
| §14 delivery | Umbrella sequencing and final handoff |
| §15 external references | Plan 4 documentation refresh |
| §16 deferred decisions | Global exclusions |

## Execution Handoff

The implementation sequence is fixed. Start with Task 1 and child plan 1 in a fresh implementation session. The preferred execution mode is subagent-driven development with one fresh reviewer per completed task; inline execution is acceptable when agent slots are unavailable. Never run two child plans concurrently because they share interface files and security invariants.
