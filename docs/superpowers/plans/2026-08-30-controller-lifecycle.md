# Protected Controller Lifecycle Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Implement pure host-collision verification and a transactional, root-protectable controller-generation lifecycle that always exposes either the fully verified old generation or the fully verified new generation.

**Architecture:** A redacted inventory is evaluated without mutation. Generation bytes are staged in immutable digest-named directories, recursively synced, and activated through one same-directory `ACTIVE.json` replacement plus parent-directory `fsync`. A transition journal makes interruption explicit; recovery never silently falls back. All tests use disposable fixtures, never a live runner root.

**Tech Stack:** Python 3.9+ standard library; descriptor-relative POSIX filesystem operations; canonical JSON; `/bin/bash` wrapper templates; unittest/pytest; macOS filesystem integration tests.

**Spec:** [Approved architecture §§5–6.4 and 9](../../ARCHITECTURE.md)

## Global Constraints

- Prerequisites: the contract/render/policy and cleanup-runtime plans are accepted on the same branch.
- Use generation manifest version `macos-runner-guard.generation.v1` and canonical JSON defined by the umbrella plan.
- Never target `/Library/Application Support/macos-runner-guard` in tests. Every writable target is beneath a new fail-if-exists temporary fixture.
- Never enumerate keychains, agents, credentials, another user's files, or private runner metadata.
- Never start, stop, register, remove, signal, or reconfigure a live service or runner.
- No automatic fallback is permitted when `ACTIVE.json` or its generation is invalid.
- Every transition is exact-instance and accepts no caller-supplied broad target.
- `.env` and `.path` are modeled and verified but remain untouched during ordinary generation updates.
- Run development and unit-test commands with the child-plan-1 `.venv/bin/python`. Use `/usr/bin/python3 -I -S` only for rendered standalone bootstrap subprocess tests.
- Each task ends in a focused commit; do not squash task commits while the subsystem is under review.

## Frozen Layout and Interfaces

```text
controller/
├── bootstrap/
│   ├── activate.py
│   ├── job-started.sh
│   └── job-completed.sh
├── generations/
│   └── <64-lowercase-hex-manifest-sha>/
│       ├── cleanup_controller.py
│       ├── instance.json
│       └── GENERATION_MANIFEST.json
├── transition-journal.json
└── ACTIVE.json
```

| Type | Exact fields |
|---|---|
| `RunnerRecord` | `runner_id: int`, `name: str`, `repository: str`, `scope: Literal["repository"]`, `account_name: str`, `labels: tuple[str, ...]`, `status: Literal["online", "offline"]`, `busy: bool`, `runner_root: PurePosixPath`, `work_root: PurePosixPath`, `service_identity: str` |
| `WorkerRecord` | `pid: int`, `repository: str`, `account_name: str`, `runner_root: PurePosixPath` |
| `QueuedJobRecord` | `run_id: int`, `repository: str`, `labels: tuple[str, ...]`, `status: Literal["queued", "waiting", "in_progress"]` |
| `ServiceRecord` | `identity: str`, `account_name: str`, `runner_root: PurePosixPath`, `status: Literal["started", "stopped"]` |
| `FilesystemBinding` | `path: PurePosixPath`, `kind: Literal["directory", "regular"]`, `uid: int`, `gid: int`, `mode: int`, `device: int`, `inode: int`, `link_count: int`, `has_allow_acl: bool`, `immutable: bool`, `sha256: Optional[str]` |
| `HostInventory` | `complete: bool`, `runners`, `workers`, `queued_jobs`, `services`, and `filesystem_bindings` as tuples of the exact records above |
| `HostVerificationReport` | `accepted: bool`, `finding_codes: tuple[str, ...]`, `bound_runner_id: Optional[int]`, `runner_uid: Optional[int]`, `root_uid: int`, `wheel_gid: int` |
| `GenerationBinding` | `generation_name: str`, `manifest_sha256: str`, `directory_device: int`, `directory_inode: int` |
| `OpenedGeneration` | `binding: GenerationBinding`, `directory_fd: int`, `controller_fd: int`, `controller_bytes: bytes`, `instance_fd: int`, `instance_bytes: bytes` |
| `RecoveryReport` | `state: RecoveryState`, `active: Optional[GenerationBinding]`, `old: Optional[GenerationBinding]`, `new: Optional[GenerationBinding]`, `allowed_actions: tuple[RecoveryAction, ...]`, `finding_codes: tuple[str, ...]`, `mutated: bool` |
| `RemovalReport` | `runner_id: int`, `registration_absent: bool`, `local_instance_removed: bool`, `backup_manifest_sha256: Optional[str]`, `finding_codes: tuple[str, ...]` |

| Module | Exact interface |
|---|---|
| `inventory.py` | `load_redacted_inventory(path) -> HostInventory`; `verify_host_binding(contract, inventory, accounts) -> HostVerificationReport` where `accounts` resolves account name to UID and group name to GID |
| `generation.py` | `build_generation_manifest(source_root) -> bytes`; `verify_generation(root, expected_sha256) -> GenerationBinding`; `stage_generation(source, controller_root) -> GenerationBinding` |
| `activation.py` | `read_active_binding(controller_root) -> GenerationBinding`; `open_active_generation(controller_root) -> ContextManager[OpenedGeneration]`; `execute_opened_generation(opened, phase, environment) -> CleanupReceipt` |
| `transition.py` | `activate_generation(controller_root, binding) -> None`; `rollback_generation(controller_root, previous) -> None`; `recover_transition(controller_root, action) -> RecoveryReport`; `remove_local_instance(contract, expected_runner_id, inventory) -> RemovalReport`; `build_lifecycle_components(source_root) -> tuple[ComponentPayload, ...]` |

`InstanceContract` stores `account_name`, not a numeric UID. The host verifier resolves the declared runner UID through `pwd.getpwnam(account_name)` and the expected root/wheel IDs through an injected account adapter. Tests use a fake adapter; portable contract bytes never contain host-specific numeric identities. The cleanup runtime, not this operator-side verifier, separately proves that its current process UID equals the resolved runner UID.

## Task 1: Model a redacted host inventory and collision report

**Files:**
- Create: `src/runner_guard/inventory.py`
- Create: `scripts/inventory-runners.py`
- Create: `scripts/verify-instance.py`
- Create: `tests/test_host_verification.py`
- Create: `tests/test_inventory_cli.py`
- Modify: `src/runner_guard/errors.py`

**Interfaces:**
- Consumes: `InstanceContract`, owner-exported GitHub runner/job JSON, bounded local filesystem/service/process probes, and an injected account/group resolver.
- Produces: `HostInventory`, `HostVerificationReport`, `load_redacted_inventory(path)`, `verify_host_binding(contract, inventory, accounts)`, and the credential-free `inventory-runners.py` / `verify-instance.py` CLIs.

- [ ] **Step 1: Write failing parse and collision tests**

Cover exact schema keys, unknown-key rejection, duplicate runner IDs, duplicate routing labels, service/name-prefix overlap, root overlap, active workers, queued unique-label jobs, and permission-denied inventory evidence. Add both trust profiles: any account reuse fails for `dedicated-account`; acknowledged reuse passes for `shared-account-trusted-only` only when roots, labels, prefixes, and services are disjoint.

The positive host fixture must prove the complete protected ancestor chain is `root:wheel`, mode `0711`, directory, non-symlink, and free of allow ACLs; the runner/work/temp/cache roots are real `0700` directories owned by the resolved runner UID; and `.env`/`.path` are root-owned regular single-link files with the exact rendered modes/hashes and the macOS immutable flag. Add a subprocess proof that the runner UID can traverse an exact `0711` path but cannot list the protected parent, while it can read/write only its declared `0700` mutable children.

Representative test:

```python
def test_dedicated_profile_rejects_live_account_reuse(self):
    report = verify_host_binding(
        dedicated_contract(account_name="projectrunner"),
        inventory_with_runner(account_name="projectrunner", label="other-ci"),
    )
    self.assertEqual(report.codes, ("dedicated_account_already_in_use",))
```

Run:

```bash
.venv/bin/python -m unittest tests.test_host_verification -v
```

Expected: FAIL with `ModuleNotFoundError: runner_guard.inventory`.

- [ ] **Step 2: Implement strict redacted parsing**

Use the exact dataclasses above with explicit scalar fields only. Reject absolute credential paths, environment maps, arbitrary command lines, and unbounded free-form records. Store service identities, root paths, labels, status, and ownership metadata needed for collision proof; never store token-bearing files or values. Resolve `account_name`, `root`, and `wheel` only through the injected account adapter; reject a missing account/group or filesystem ownership that differs from the resolved identities.

- [ ] **Step 3: Implement deterministic collision evaluation**

Return sorted finding codes and bounded public identifiers. Require exactly one repository-scoped record whose repository, runner-name prefix, unique routing label, roots, account, and service identity match the contract. Prove no organization-scoped or foreign repository runner can route on the unique label. An unavailable or incomplete inventory, missing filesystem binding, or unproven ACL/immutable flag produces `inventory_unproven`, never a warning-only pass.

- [ ] **Step 4: Add credential-free inventory and verification CLIs**

`scripts/inventory-runners.py --contract CONTRACT --github-runners-json OWNER_EXPORT --queued-jobs-json OWNER_EXPORT --output NEW_FILE` validates owner-exported GitHub API JSON, then inspects only the contract-declared local roots, the exact service identity, bounded `ps` records, filesystem metadata, ACLs, flags, and free space. It never invokes `gh`, accepts a token, reads registration files, reads environment values, or follows a symlink. Command execution goes through an injected adapter so Linux tests use fixtures and macOS integration tests use disposable paths only. It writes canonical redacted `HostInventory` JSON to a new file.

`scripts/verify-instance.py --contract CONTRACT --inventory INVENTORY` parses both files, invokes `verify_host_binding()`, emits only bounded finding codes and public runner IDs, and returns nonzero unless `accepted` is true. Tests reject an overwritten output, incomplete owner export, repository mismatch, foreign label, unavailable ACL/flag probe, and accidental credential-bearing field.

- [ ] **Step 5: Run focused tests and commit**

```bash
.venv/bin/python -m unittest tests.test_host_verification tests.test_inventory_cli -v
git diff --check
git add src/runner_guard/inventory.py src/runner_guard/errors.py scripts/inventory-runners.py scripts/verify-instance.py tests/test_host_verification.py tests/test_inventory_cli.py
git commit -m "feat: add redacted runner collision verification"
```

Expected: all inventory tests pass.

## Task 2: Define the immutable generation manifest

**Files:**
- Create: `src/runner_guard/generation.py`
- Create: `tests/test_generation.py`

**Interfaces:**
- Consumes: an exact source directory containing `cleanup_controller.py` mode `0555` and canonical `instance.json` mode `0444`.
- Produces: canonical `GENERATION_MANIFEST.json` bytes and `verify_generation(root, expected_sha256) -> GenerationBinding`.

- [ ] **Step 1: Write failing manifest tests**

Require exactly two payload members, `cleanup_controller.py` mode `0555` and `instance.json` mode `0444`. The manifest records schema version, each path, mode, byte count, and SHA-256; it excludes itself. Reject extra members, wrong modes, path aliases, symlinks, hard links, non-regular files, noncanonical JSON, and member drift.

```python
def test_generation_manifest_binds_every_payload_byte(self):
    manifest = build_generation_manifest(self.source)
    self.assertEqual(
        json.loads(manifest)["schema_version"],
        "macos-runner-guard.generation.v1",
    )
    self.assertEqual(
        [item["path"] for item in json.loads(manifest)["members"]],
        ["cleanup_controller.py", "instance.json"],
    )
```

Run:

```bash
.venv/bin/python -m unittest tests.test_generation -v
```

Expected: FAIL because manifest construction is absent.

- [ ] **Step 2: Implement canonical manifest creation and verification**

The generation name is the lowercase SHA-256 of the exact `GENERATION_MANIFEST.json` bytes. `verify_generation()` opens the directory without following symlinks, verifies all members and no extras, and returns device/inode identity.

- [ ] **Step 3: Add mutation coverage**

Mutate each field and each member independently. Rejection must not change any fixture byte or metadata.

- [ ] **Step 4: Run and commit**

```bash
.venv/bin/python -m unittest tests.test_generation -v
git add src/runner_guard/generation.py tests/test_generation.py
git commit -m "feat: bind immutable controller generations"
```

## Task 3: Stage and recursively sync a complete generation

**Files:**
- Modify: `src/runner_guard/generation.py`
- Modify: `tests/test_generation.py`

**Interfaces:**
- Consumes: verified generation source bytes, a validated controller root, and an injected `GenerationFilesystem` adapter.
- Produces: `stage_generation(source, controller_root) -> GenerationBinding`, with a recursively synced digest-named generation or a verified reusable identical generation.

- [ ] **Step 1: Add a failing staged-generation test**

Instrument file open/write/`fsync`/rename calls. Require a fresh staging directory named `.staging-<manifest-sha>-<nonce>`, mode `0700` while writing; sync every file, then every directory bottom-up; set final file and directory modes; rename to the digest name only after verification and sync.

- [ ] **Step 2: Implement `stage_generation()` with injected filesystem operations**

Use a small `GenerationFilesystem` protocol so tests can fail after each boundary. If the final digest directory already exists, verify it exactly and reuse it; a drifted existing directory fails closed.

- [ ] **Step 3: Prove partial staging is never active**

For every injected failure, assert `ACTIVE.json` is unchanged and only a clearly named staging entry plus journal evidence may remain.

- [ ] **Step 4: Run and commit**

```bash
.venv/bin/python -m unittest tests.test_generation.GenerationStagingTests -v
git add src/runner_guard/generation.py tests/test_generation.py
git commit -m "feat: stage and sync controller generations"
```

## Task 4: Bind the stable bootstrap to one active generation

**Files:**
- Create: `src/runner_guard/activation.py`
- Create: `tests/test_activation.py`
- Modify: `src/runner_guard/bundle_runtime.py`
- Verify/modify: `templates/activate.py`
- Verify: `templates/job-started.sh`
- Verify: `templates/job-completed.sh`

**Interfaces:**
- Consumes: `ACTIVE.json`, a verified generation manifest, descriptor-opened `cleanup_controller.py` and `instance.json` bytes, `CleanupPhase`, and the exact five-field environment mapping.
- Produces: `read_active_binding(controller_root)`, `open_active_generation(controller_root) -> ContextManager[OpenedGeneration]`, and `execute_opened_generation(opened, phase, environment) -> CleanupReceipt`, which calls the exact generated entry point `run_generation_entry(phase, contract_bytes, environment)`.

- [ ] **Step 1: Write failing active-binding tests**

Require `ACTIVE.json` to contain only `schema_version`, `generation_name`, and `manifest_sha256`. The name and manifest digest must agree; the opened generation directory's device/inode is revalidated before invocation. Reject symlinked ancestors, mutable/wrong-owner bootstrap files, invalid generation names, manifest drift, an unlisted member, and a replaced directory between verification and execution.

Add two race regressions: replace the pathname after the controller descriptor opens, and replace it after hash verification but before execution. Both tests must prove the already-opened verified bytes execute or the call fails; bytes from the replacement path must never execute.

- [ ] **Step 2: Implement descriptor-bound `read_active_binding()`**

Open the controller and generations roots once, open the generation relative to those descriptors with `O_DIRECTORY|O_NOFOLLOW`, verify the manifest, then open both `cleanup_controller.py` and `instance.json` relative to the retained generation descriptor. Read bounded bytes from those descriptors; compare each SHA-256/size/mode to the verified manifest; require `instance.json` to equal `canonical_contract_bytes(load_contract_bytes(instance_bytes))`; compile the exact controller bytes with filename `<active-cleanup-controller>`; execute them in an isolated namespace; require an exact callable `run_generation_entry(phase, contract_bytes, environment)`; and call it with the verified `instance_bytes`. Retain all three descriptors until it returns. Never import or reopen either generation member by pathname.

- [ ] **Step 3: Add minimal stable bootstrap templates**

Child plan 2's `bundle_runtime.py` deterministically creates the standalone, standard-library-only `activate.py`. This task integrates its frozen active-generation lookup without changing the standalone/no-site boundary: output contains no import of `runner_guard`, no site-package dependency, no source-root path, and no dynamic `sys.path` edit. The already reviewed shell wrappers remain exactly:

```bash
#!/bin/bash
set -euo pipefail
umask 077
exec /usr/bin/python3 -I -S "<absolute-root-owned-activate.py>" --phase before
```

The completed hook uses `--phase after`. `activate.py` uses only standard-library imports and the rendered absolute controller root; it does not read a caller-supplied controller path. The generated hook passes no contract or instance path: the standalone activation program reads `instance.json` only from the verified active generation.

- [ ] **Step 4: Verify syntax, isolation, and commit**

```bash
/bin/bash -n templates/job-started.sh
/bin/bash -n templates/job-completed.sh
.venv/bin/python -m unittest tests.test_activation -v
activation_test_dir="$(mktemp -d)"
.venv/bin/python -m runner_guard.bundle_runtime --output "$activation_test_dir/activate.py"
/usr/bin/python3 -I -S "$activation_test_dir/activate.py" --self-test
git add src/runner_guard/activation.py src/runner_guard/bundle_runtime.py templates/activate.py templates/job-started.sh templates/job-completed.sh tests/test_activation.py
git commit -m "feat: add descriptor-bound controller bootstrap"
```

## Task 5: Activate through one atomic manifest switch

**Files:**
- Create: `src/runner_guard/transition.py`
- Create: `tests/test_transition.py`

**Interfaces:**
- Consumes: verified old/new `GenerationBinding` values, the exact active binding, and an injected durable filesystem transition adapter.
- Produces: `activate_generation(controller_root, binding) -> None` and a canonical journal state in `prepared`, `active_replaced`, or `committed`.

- [ ] **Step 1: Write failing activation-order tests**

Pin this order: verify old active binding → verify new generation → write/sync journal → write/sync temporary active file → `os.replace()` within controller root → `fsync()` controller directory → update/sync journal outcome → remove journal → final directory `fsync()`.

- [ ] **Step 2: Implement the transition state machine**

The journal contains schema version, operation ID, exact old/new bindings, and state from the closed set `prepared`, `active_replaced`, `committed`. No timestamps or host-dependent fields enter binding bytes.

- [ ] **Step 3: Reject concurrent or ambiguous transitions**

An existing journal blocks a new activation. An active binding not equal to the journal's old or new binding is `transition_state_ambiguous`.

- [ ] **Step 4: Run and commit**

```bash
.venv/bin/python -m unittest tests.test_transition.ActivationOrderingTests -v
git add src/runner_guard/transition.py tests/test_transition.py
git commit -m "feat: activate generations transactionally"
```

## Task 6: Recover every interrupted transition fail closed

**Files:**
- Modify: `src/runner_guard/transition.py`
- Modify: `tests/test_transition.py`
- Create: `templates/recover-instance.sh`

**Interfaces:**
- Consumes: the exact transition journal, active binding, verified old/new generations, and an explicit `RecoveryAction`.
- Produces: `recover_transition(controller_root, action) -> RecoveryReport`; no recovery mutation occurs unless the action is explicit and still matches the current report state.

- [ ] **Step 1: Add interruption tests at every durable boundary**

Inject termination:

1. after staging creation;
2. after each member write;
3. before recursive sync;
4. after recursive sync;
5. after journal sync;
6. before active-file replace;
7. after replace but before controller-directory `fsync`;
8. after controller-directory `fsync` but before journal cleanup.

Each restart must expose a verified old or verified new binding—never mixed bytes. A questionable `fsync` boundary reports `operator_decision_required` and preserves evidence.

Pin this recovery table in tests before implementation:

| Observed state | `RecoveryState` | Allowed explicit actions |
|---|---|---|
| No journal, active valid, no staging | `clean` | `inspect_only` |
| No journal, active valid, one verified orphan staging entry | `orphan_staging` | `inspect_only`, `discard_orphan_staging` |
| `prepared`, active equals verified old, new verified | `prepared_old_active` | `inspect_only`, `discard_prepared`, `activate_new` |
| `active_replaced` or `committed`, active equals verified new | `new_active_journal_present` | `inspect_only`, `finalize_new` |
| Replace may have occurred but directory sync is unproven | `fsync_ambiguous` | `inspect_only`, `reactivate_old`, `reactivate_new` |
| Active equals neither binding, a manifest is invalid, or multiple journals/staging entries exist | `blocked_ambiguous` | `inspect_only` |

Every action other than `inspect_only` requires the exact current report state and exact old/new manifest identities as preconditions. If the state changed, mutation is refused.

- [ ] **Step 2: Implement `recover_transition()`**

`recover_transition(controller_root, action)` reads only exact journal and active paths, verifies both generation manifests, and returns a `RecoveryReport`. Its default CLI action is `inspect_only`. It may delete an orphan staging directory only for the explicit `discard_orphan_staging` action after the same descriptor-relative proof used by cleanup. It never activates either generation automatically, and `blocked_ambiguous` has no mutating action.

- [ ] **Step 3: Render a no-argument recovery wrapper**

The generated wrapper is bound to one static instance path, requires root, requires the exact service stopped, and invokes the reviewed transition CLI. Core tests render and syntax-check it but do not execute it against the host.

- [ ] **Step 4: Run and commit**

```bash
.venv/bin/python -m unittest tests.test_transition.InterruptionRecoveryTests -v
/bin/bash -n templates/recover-instance.sh
git add src/runner_guard/transition.py templates/recover-instance.sh tests/test_transition.py
git commit -m "feat: recover interrupted controller transitions"
```

## Task 7: Implement explicit rollback and exact-instance removal

**Files:**
- Modify: `src/runner_guard/transition.py`
- Create: `tests/test_removal.py`
- Create: `templates/rollback-instance.sh`
- Create: `templates/remove-instance.sh`
- Create: `templates/restore-instance.sh`

**Interfaces:**
- Consumes: exact current/previous generation bindings for rollback, or `InstanceContract`, exact runner ID, and fresh complete `HostInventory` for local removal.
- Produces: `rollback_generation(controller_root, previous)`, `remove_local_instance(contract, expected_runner_id, inventory) -> RemovalReport`, and exact-path no-argument rollback/removal/restore wrappers.

- [ ] **Step 1: Write failing rollback tests**

Rollback verifies both current and previous immutable generations, journals the exact reverse transition, and uses the same atomic activation function. It rejects an unrecorded previous generation or a current binding that has changed.

- [ ] **Step 2: Write failing removal tests**

Removal has two separately authorized phases. The owner first deregisters the exact recorded `runner_id` through GitHub's supported UI/API using a short-lived credential that never enters this toolkit. The local helper runs only after a fresh complete inventory proves that exact ID absent, the exact service stopped, no Worker/queued unique-label job exists, and no other record binds the root. It accepts the rendered exact runner ID and instance root only, backs up the active binding and contract, never calls GitHub or accepts a token, and refuses unexpected content, symlinks, mounted descendants, foreign ownership, or another instance's path.

- [ ] **Step 3: Implement rollback/removal reports and wrappers**

Implement `remove_local_instance(contract, expected_runner_id, inventory)`. It returns `RemovalReport` and refuses if the inventory still contains the ID, is incomplete, or lacks the exact absence/collision proofs. The restore wrapper can recreate only the exact backed-up controller bytes and binding. All wrappers accept no broad path arguments and print bounded status/hashes only. Documentation labels GitHub deregistration as an owner/API prerequisite, never as a helper capability.

- [ ] **Step 4: Run and commit**

```bash
.venv/bin/python -m unittest tests.test_transition tests.test_removal -v
/bin/bash -n templates/rollback-instance.sh
/bin/bash -n templates/remove-instance.sh
/bin/bash -n templates/restore-instance.sh
git add src/runner_guard/transition.py templates/rollback-instance.sh templates/remove-instance.sh templates/restore-instance.sh tests/test_removal.py
git commit -m "feat: add exact rollback and removal flows"
```

## Task 8: Integrate rendering and run disposable macOS lifecycle proof

**Files:**
- Modify: `src/runner_guard/render.py`
- Modify: `src/runner_guard/transition.py`
- Create: `templates/install-instance.sh`
- Create: `tests/test_lifecycle_integration.py`
- Modify: `tests/test_render.py`

**Interfaces:**
- Consumes: the five cleanup components from child plan 2, five lifecycle components from `build_lifecycle_components(source_root)`, explicit template-root and policy-source bytes, a validated contract, and disposable host-verification evidence.
- Produces: the exact 15 rendered payload members and modes, a no-argument install helper, and a disposable end-to-end lifecycle proof; `MANIFEST.json` and `SHA256SUMS` remain artifact-stage-only.

- [ ] **Step 1: Add a failing complete-layout render test**

First pin the renderer boundary alone. The pure render result contains exactly the closed 15-member payload, excludes `ACTIVE.json` and every generation-layout file, and is byte-identical on Linux and macOS. `build_lifecycle_components(source_root) -> tuple[ComponentPayload, ...]` is implemented in `transition.py` and returns exactly `install-instance.sh`, `recover-instance.sh`, `rollback-instance.sh`, `remove-instance.sh`, and `restore-instance.sh`, all mode `0555`. Merge those with child plan 2's five cleanup components and reject any remaining inert fixture sentinel. `render_instance(contract, components, template_root, policy_source)` must receive the resulting exact ten reviewed component payloads plus explicit policy bytes and return the complete immutable tuple passed to `write_rendered_tree()`; it must not discover files from the working directory.

Assert the complete result is exactly: `instance.json` `0444`; the ten components `activate.py`, `cleanup_controller.py`, `job-started.sh`, `job-completed.sh`, `install-instance.sh`, `recover-instance.sh`, `rollback-instance.sh`, `remove-instance.sh`, `restore-instance.sh`, and `residue-audit.py` all `0555`; `workflow-guard.yml` `0444`; `workflow-policy.py` `0555`; and `ADOPTION_CHECKLIST.md` / `AGENT_PROMPT.md` both `0444`.

- [ ] **Step 2: Integrate the lifecycle renderer**

Render absolute static paths only from the validated contract. Separately execute the generated install helper against a disposable fake root and assert that this post-render installation creates the bootstrap tree, initial digest-named generation, canonical generation manifest, and `ACTIVE.json`. The initial generation digest determines its directory and active binding. Fail if any generated destination already exists. The generated no-argument install helper requires root, verifies the host-verification evidence digest and service-stopped state, creates only the exact absent protected/mutable tree, installs the standalone bootstrap and initial generation, and never registers, starts, or removes a runner. Run `/bin/bash -n` on all five lifecycle helper templates.

- [ ] **Step 3: Run the full disposable integration test on macOS**

```bash
.venv/bin/python -m unittest tests.test_lifecycle_integration -v
.venv/bin/python -m pytest tests/test_generation.py tests/test_activation.py tests/test_transition.py tests/test_removal.py -q
```

Expected: two successive generations activate and roll back in a temporary fixture; every interruption case returns an old-or-new verified state; no path outside the fixture changes.

- [ ] **Step 4: Run affected full validation and commit**

```bash
.venv/bin/python -m compileall src tests -q
.venv/bin/python -m unittest discover -s tests -v
.venv/bin/python -m pytest -q
git diff --check
git add src/runner_guard/render.py src/runner_guard/transition.py templates/install-instance.sh tests/test_lifecycle_integration.py tests/test_render.py
git commit -m "test: prove protected controller lifecycle"
```

## Acceptance Gate

- [ ] Every inventory, generation, activation, transition, interruption, rollback, and removal test passes.
- [ ] The active state is always a fully verified old or new generation.
- [ ] Standalone `/usr/bin/python3 -I -S` execution uses the descriptor-opened, hash-verified controller bytes; both pathname-swap races fail safe.
- [ ] The runner account can traverse but not list the protected `0711` chain, and only its declared `0700` mutable roots are writable.
- [ ] Invalid active state never causes automatic fallback.
- [ ] Generated wrappers pass `/bin/bash -n` and contain no broad target or process command.
- [ ] No live runner, service, registration, account, `/Library` instance root, or unrelated process was read or changed.
- [ ] A fresh exact-head review reports zero actionable lifecycle findings.

Required endpoint:

```text
CONTROLLER_LIFECYCLE_ACCEPTED
```
