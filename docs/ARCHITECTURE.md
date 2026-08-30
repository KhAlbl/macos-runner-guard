# macOS Runner Guard — Architecture Design

**Status:** Approved architecture; implementation pending

**Date:** 2026-08-30

**Project:** `macos-runner-guard`

**Repository:** `https://github.com/KhAlbl/macos-runner-guard`

**Scope:** reusable tooling and guidance for safely operating multiple repository-scoped GitHub Actions runners on one macOS host

## 1. Decision Summary

Build a standalone, project-neutral toolkit that renders a separately bound runner package for each repository. It generalizes lessons from a qualified internal runner—dedicated routing, private job storage, automatic residue cleanup, fail-closed path validation, workflow policy checks, deterministic evidence, and rollback—without copying project-specific identities or modifying an installed runner.

The toolkit supports two explicit profiles:

1. **`dedicated-account` — recommended.** Each independent trust domain gets a non-admin macOS account and a private home. This gives useful credential and filesystem separation between projects on the same Mac.
2. **`shared-account-trusted-only` — compatibility profile.** Multiple trusted repositories may run under one existing macOS account, each with disjoint paths and labels. This improves routing and residue control but provides **no privacy or credential isolation** between runners sharing that account.

The toolkit is not a replacement for a VM, a dedicated host, or an ephemeral macOS provider. Persistent cleanup reduces accidental cross-job residue; it cannot contain hostile code running as the same user.

The owner approved publication as `KhAlbl/macos-runner-guard` under the MIT License. Public availability does not authorize an executable release; implementation, release governance, and production-support claims remain separate gates.

## 2. Goals

- Make adoption reproducible for independent repositories without embedding any one project's identities.
- Give every project a repository-scoped runner registration, unique label, unique runner root, unique work root, unique temporary root, and unique cleanup namespace.
- Keep the runner online for routine trusted CI after qualification; stop it only for maintenance, quarantine, a security incident, a special isolation window, or reboot qualification.
- Generate deterministic, reviewable files and a deterministic adoption ZIP with a two-level inventory that covers every payload file by path, mode, byte count, and SHA-256 without a self-hash cycle.
- Keep registration credentials outside the toolkit, repository, logs, evidence, command history, and agent context.
- Make unsafe or ambiguous cleanup fail closed without deleting evidence.
- Preserve all other runners: no label-wide operations, organization-wide mutations, broad process termination, shared root rewriting, or implicit service changes.
- Make the implementation small enough to audit and portable enough that another Codex agent can adopt it with a bounded project-specific plan.

## 3. Non-Goals

- Kubernetes, Actions Runner Controller, GARM, Tart, Anka, Docker-based macOS virtualization, or a custom JIT control plane.
- A shared organization runner or one runner registration used by many repositories.
- Physical-host isolation, malicious-code containment, guaranteed process destruction, or protection from a compromised kernel/account.
- Automatic acquisition or storage of a GitHub registration token.
- Automatic creation of macOS users, Secure Tokens, FileVault identities, keychains, GitHub Apps, private keys, sudoers rules, LaunchDaemons, or root services.
- Reading or asserting the contents of keychains, SSH agents, cloud credentials, or another user's files.
- Replacing GitHub's official runner binary or service lifecycle.
- Changing an adopting application's runtime behavior, public interfaces, data formats, release state, or benchmark methodology.

## 4. Design Provenance and Lessons Reused

A qualified internal implementation informed this architecture. It is not a runtime dependency, and no private repository identity, live runner file, credential, or host evidence belongs in this project.

The generic design preserves these proven lessons:

- Use system `/bin/bash`, not Homebrew Bash, for bootstrap hooks.
- Run cleanup through `/usr/bin/python3 -I -S` so user and site configuration cannot alter imports.
- Use fixed absolute paths supplied by a root-owned instance contract, never deletion targets supplied by workflow code.
- Separate a job's files under a unique run/attempt/job/leg directory.
- Clean before and after a job, with an additional workflow `if: always()` layer.
- Validate paths, ownership, type, link count, device, and traversal bounds before deletion.
- Keep cleanup output bounded and content-free.
- Treat GUI-session and keychain state as outside the per-job cleanup contract.
- Avoid controller byte locks inside repository workflows when they create an upgrade deadlock; verify root ownership, modes, immutability/non-writability, and a stable instance contract instead. Record controller hashes in transition evidence.
- Keep `.env` and `.path` stable when hook paths and the system-first bootstrap path do not change.
- Prefer a normal persistent runner over a custom controller when the workload is trusted and the dedicated-account boundary is sufficient.

## 5. Trust Model

### 5.1 Common boundary

Anyone able to change a same-repository workflow that is permitted to run can execute code as the runner's macOS account. Repository scope and labels control scheduling; they are not host isolation.

All runner instances share the physical Mac's CPU, RAM, storage device, network, kernel, process table, power, and thermal envelope. A resource-heavy project can affect another project's timing even when filesystem identities are separate.

The toolkit therefore permits only trusted, same-repository pull requests and trusted pushes by default. Fork pull requests, untrusted contributors, secret-bearing release jobs, signing jobs, deployment jobs, and workloads requiring hostile-code containment must use an ephemeral VM or a dedicated host.

### 5.2 `dedicated-account`

Required properties:

- one non-admin macOS account per independent trust domain;
- home directory owned by that account, mode `0700`, and not a symlink;
- no passwordless sudo;
- no project GitHub login, SSH key, cloud credential, signing identity, or inherited maintainer keychain credential;
- project-mutable runner, work, temporary, cache, and generated evidence roots that are real directories, owned by that account, mode `0700`, free of allow ACLs, and located beneath a project-specific root-protected instance parent;
- a repository-scoped GitHub runner registration and unique custom label.

This profile reduces cross-project credential and filesystem exposure. It does not prevent code from attacking other processes or shared host resources through operating-system vulnerabilities or deliberately granted permissions.

### 5.3 `shared-account-trusted-only`

Required properties:

- separate runner, work, temp, workspace, static-controller, and job-prefix paths per repository;
- unique repository-scoped registration and custom label;
- same workflow and cleanup controls as the dedicated profile;
- explicit acknowledgement that every runner under the account can potentially access the account's files, credentials, caches, keychain, agents, and sibling runner registration material.

This profile may be used for routing and reliability among repositories trusted at the same level. It must never be described as private, isolated, credential-safe, or equivalent to a dedicated account.

## 6. Architecture

### 6.1 Toolkit source layout

The implementation phase uses this core ownership layout. The detailed implementation plans enumerate the complete test, documentation, fixture, and validation-file inventory added under these roots.

```text
macos-runner-guard/
├── .github/
│   ├── pull_request_template.md
│   └── workflows/ci.yml
├── pyproject.toml
├── requirements/
│   ├── validation.in
│   └── validation.txt
├── src/runner_guard/
│   ├── __init__.py
│   ├── __main__.py
│   ├── activation.py
│   ├── archive.py
│   ├── bundle_runtime.py
│   ├── capabilities.py
│   ├── cleanup.py
│   ├── cli.py
│   ├── contract.py
│   ├── errors.py
│   ├── fsops.py
│   ├── generation.py
│   ├── inventory.py
│   ├── job_identity.py
│   ├── jsonio.py
│   ├── manifest.py
│   ├── policy.py
│   ├── render.py
│   ├── residue.py
│   ├── scanner.py
│   ├── transition.py
│   ├── workspace.py
│   └── zip_format.py
├── schemas/instance-v1.schema.json
├── templates/
│   ├── activate.py
│   ├── job-started.sh
│   ├── job-completed.sh
│   ├── workflow-guard.yml
│   ├── ADOPTION_CHECKLIST.md
│   ├── AGENT_PROMPT.md
│   ├── install-instance.sh
│   ├── recover-instance.sh
│   ├── remove-instance.sh
│   ├── restore-instance.sh
│   └── rollback-instance.sh
├── examples/
│   ├── dedicated-account.json
│   └── shared-account-trusted-only.json
├── scripts/
│   ├── render-instance.py
│   ├── verify-instance.py
│   ├── build-adoption-zip.py
│   ├── inventory-runners.py
│   └── audit-residue.py
├── tests/
│   ├── test_contract.py
│   ├── test_cleanup_safety.py
│   ├── test_instance_separation.py
│   ├── test_workflow_policy.py
│   └── test_bundle_determinism.py
├── docs/
│   ├── ADOPTION.md
│   ├── OPERATIONS.md
│   ├── SECURITY_MODEL.md
│   ├── TROUBLESHOOTING.md
│   └── LESSONS_LEARNED.md
└── AGENT_ADOPTION_PROMPT.md
```

`MANIFEST.json` and `SHA256SUMS` are generated inside fresh artifact-staging directories and archives. They are not committed at the repository root, which avoids a control-file update on every source commit while preserving complete artifact coverage.

### 6.2 Instance contract

Each project is described by a schema-validated `instance.json`. It contains only non-secret configuration:

- toolkit and contract schema versions;
- GitHub owner and repository;
- normalized project slug;
- selected trust profile;
- macOS account and exact home path;
- runner name prefix and unique routing label;
- exact runner, work, temp, workspace, controller, evidence, and cache roots;
- exact per-job directory prefix;
- an enumerated allowed-leg set, not an arbitrary regular expression;
- platform and architecture;
- cleanup entry, depth, byte, and time bounds;
- a measured project-specific free-disk floor;
- service identity pattern;
- routine availability policy;
- whether reboot/login/reconnect has been qualified.

Pure contract validation and rendering are host-independent. They reject:

- `/`, `/Users`, a home directory itself, or a root declared by another supplied instance contract as a mutable target;
- overlapping mutable roots declared in the supplied contract set;
- path traversal, unresolved variables, non-absolute paths, and control characters;
- account/home/profile combinations that contradict the claimed trust level;
- a `minimum_free_bytes` value with no recorded adopting-project measurement evidence. The pure validator checks only the explicit positive value; the adoption review verifies the measurement provenance.

A separate host-bound verifier runs immediately before adoption or transition. It walks the declared ancestor chains without following links and reads a fresh, redacted inventory; it rejects symlinked or structurally invalid ancestors and collisions with live runner roots, labels, service/name prefixes, queued jobs, or active workers. For `dedicated-account`, any live reuse of the proposed account is also a collision. For `shared-account-trusted-only`, the acknowledged account reuse is allowed only when roots, labels, name prefixes, and service identities remain disjoint. Failure or lack of permission to prove the live inventory blocks adoption. Host inventory never changes rendered bytes and is never embedded in a portable ZIP.

### 6.3 Generated project bundle

For an instance named `<project-slug>`, the renderer creates:

```text
instances/<project-slug>/
├── instance.json
├── activate.py
├── cleanup_controller.py
├── job-started.sh
├── job-completed.sh
├── workflow-guard.yml
├── workflow-policy.py
├── install-instance.sh
├── recover-instance.sh
├── rollback-instance.sh
├── remove-instance.sh
├── restore-instance.sh
├── residue-audit.py
├── ADOPTION_CHECKLIST.md
└── AGENT_PROMPT.md
```

The renderer output is a closed 15-member inventory with exact modes:

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

The pure renderer receives four explicit inputs: the validated contract, the exact ten component payloads, the reviewed template root, and the exact reviewed workflow-policy source bytes. It never discovers policy bytes through package paths, import state, the current directory, or live host state. The generated policy embeds canonical contract bytes through a deterministic hexadecimal bytes expression—not raw JSON as Python syntax—and its standalone CLI parses those bytes before evaluating a bounded workflow input. The renderer creates this payload only. Fresh per-instance artifact staging then adds `MANIFEST.json` and `SHA256SUMS` under the deterministic artifact contract in section 11. The resulting archive is self-contained but remains a proposal until a project-specific review verifies repository workflows, host paths, account state, runner inventory, and registration scope.

### 6.4 Root-protected instance and controller generations

Each project gets a root-protected instance parent outside the runner account's home, runner application, work, cache, and temporary trees. The default location is:

```text
/Library/Application Support/macos-runner-guard/instances/<project-slug>/
```

The global toolkit parent, its `instances/` child, and each project instance parent are real, non-symlink `root:wheel` directories with exact mode `0711`: runner accounts may traverse an exact known path but cannot list, create, rename, or remove instance entries. They have no allow ACL. Every controller ancestor below the instance parent uses the same root ownership, no-ACL rule, and mode `0711` unless a stricter non-writable mode remains traversable. The host-bound verifier rechecks owner, type, mode, ACL, symlink absence, and positive traversal through this complete ancestor chain.

The project instance parent contains separate children:

- `runner/`, `work/`, `temp/`, `cache/`, and redacted operational output as real, non-symlink directories owned by the runner account, exact mode `0700`, with no allow ACL and created under `umask 077`;
- `controller/` and every generation ancestor owned by `root:wheel`, mode `0711`, no allow ACL, and never writable by the runner account; controller files use exact reviewed `0444` or `0555` modes.

The root-owned instance parent prevents the runner account from renaming or replacing the complete runner root. The runner's `.env` and `.path` activation files must be regular, non-symlink, single-link, root-owned, exact-mode, immutable files with reviewed byte contracts. Their paths remain inside the application root because GitHub reads them there, but the protected parent and immutable flags prevent replacement. Changes require the service to be stopped and a reviewed root transition. GitHub hook targets themselves live under the separate root-owned controller root, not in the runner application directory.

Controller updates use one fixed generation model:

```text
controller/
├── bootstrap/
│   ├── activate.py
│   ├── job-started.sh
│   └── job-completed.sh
├── generations/
│   └── <generation-manifest-sha256>/
│       ├── cleanup_controller.py
│       ├── instance.json
│       └── GENERATION_MANIFEST.json
└── ACTIVE.json
```

The minimal bootstrap hooks are stable, root-owned `/bin/bash` wrappers named by `.env`; they invoke root-owned `activate.py` through `/usr/bin/python3 -I -S`. The toolkit builds that bootstrap as a deterministic, standard-library-only standalone program, so it never imports an installed package or user/site path at runtime. The bootstrap opens and validates the root-owned `ACTIVE.json`, binds the declared immutable generation directory and manifest by descriptor and inode/device identity, opens the selected cleanup controller without following links, hashes and compiles the already-opened verified bytes, executes them in an isolated namespace, and calls one explicit entry point while retaining the descriptor binding. A path swap after verification cannot change the executed bytes. A generation is fully staged, hashed, ownership/mode checked, recursively synced, and made read-only before it can be activated. `ACTIVE.json` records the generation name and manifest hash and is replaced through one same-directory atomic rename followed by directory `fsync`. No mutable controller binary is shared between projects.

An interrupted transition cannot expose a partially staged generation: bootstrap ignores staging names and reads only `ACTIVE.json`. On startup, an orphan staging entry or transition journal causes a fail-closed recovery report; the active generation remains unchanged if its manifest is valid. An invalid active manifest never triggers an automatic fallback. A reviewed recovery helper either completes cleanup of the orphan or atomically reactivates the previously recorded generation.

Workflow code verifies the complete protected ancestor chain, `.env`/`.path` byte contracts, immutable flags, bootstrap ownership/mode, active-manifest identity, and non-writability. Transition evidence records exact generation and activation SHA-256 values. Hashes are integrity identifiers, not signatures or authentication.

## 7. Cleanup Contract

### 7.1 Job directory

Every job uses a root of the form:

```text
<temp-root>/<prefix>-<run-id>-<run-attempt>-<github-job>-<leg>
```

The workflow knows the full job and leg name. The completed hook only needs the validated current-run prefix, allowing it to remove all serial matrix-leg roots for that run and attempt.

`run-id`, `run-attempt`, job name, and leg are validated with full-match, length, canonical-number, and allow-list rules before any filesystem action.

### 7.2 Three cleanup layers

1. **Before-job hook:** removes only valid stale directories carrying that instance's prefix under that instance's temp root. It is recovery for cancellation, restart, or a failed previous completion hook.
2. **Workflow `if: always()` cleanup:** removes only the exact current leg root and cleans only a validated ordinary checkout belonging to that instance.
3. **Completed hook:** removes every valid job root matching the current run/attempt prefix.

The official runner may also clean portions of its work area. The toolkit's cleanup is defense in depth and provides a deterministic policy and receipt.

### 7.3 Filesystem safety

Cleanup uses descriptor-relative, no-follow operations with inode/device revalidation on macOS. There is no canonical-path, `lstat`, or path-recursive fallback. Adoption first runs a capability probe against `/usr/bin/python3`; it requires CPython 3.9 or newer, the Apple Command Line Tools that supply that interpreter, `dir_fd` support for the required `os` operations, `O_DIRECTORY`, `O_NOFOLLOW`, descriptor-based scanning, non-following stat, and stable inode/device fields. If any primitive is absent or behaves differently, adoption and cleanup fail closed.

Before deletion cleanup proves:

- every root canonicalizes to the exact contract path;
- no validated root or ancestor is a symlink;
- target entries are on the expected device and owned by the runner UID;
- regular files do not have unexpected hard links;
- directory traversal stays below configured entry, depth, byte, and time limits;
- the target is below the opened project temp/work root;
- no target is a runner binary, registration file, diagnostic root, action cache, another runner root, another user path, or global temporary path.

Malformed matching entries are preserved and cause a nonzero result. Absence is success. Before and after phases are idempotent.

The controller never uses a broad `rm -rf`, `pkill`, `killall`, label-wide removal, unresolved environment variable, or process-killing rule. Unknown detached processes cause quarantine and operator review; they are not automatically terminated.

Cleanup emits only bounded status receipts and counts. It never prints filenames containing user data, file contents, environment values, credentials, or evidence payloads.

GitHub job hooks have no platform-provided execution timeout. The cleanup controller therefore enforces its own monotonic deadline in addition to entry, depth, and byte limits; exceeding any bound fails closed and preserves the unproven target.

## 8. Workflow Guard

The rendered workflow fragment and policy checker enforce these defaults:

- `permissions: contents: read`;
- push to the approved default branch and pull requests targeting it;
- same-repository PR condition before runner code executes;
- no `pull_request_target`;
- no repository secrets, write permissions, deployment environment, PR comments, or status writes;
- unique labels including `self-hosted`, `macOS`, architecture, and the instance label;
- commit-pinned Actions;
- checkout cleaning and `persist-credentials: false`;
- per-ref, non-cancelling concurrency;
- explicit job timeouts and serial matrices where the host has one matching runner;
- system `/bin/bash`;
- job-specific `TMPDIR`, cache, bytecode, venv, distribution, and report paths below the exact job root;
- cache and artifact upload disabled by default;
- a project-measured free-space threshold;
- `if: always()` cleanup;
- no automatic rerun.

Manual reruns follow ordinary GitHub semantics after the cause is understood. A performance failure must be investigated before rerunning.

The workflow fragment is deliberately not a complete universal CI workflow. Each adopting repository integrates its own validation commands, then the generated policy checker verifies the resulting full workflow against the instance contract.

## 9. Installation, Upgrade, and Rollback

### 9.1 One-time project setup

The adoption procedure:

1. Audit all registered runners, labels, accounts, roots, services, active workers, disk, ACLs, and queued jobs read-only.
2. Select and record a trust profile.
3. Create or verify the macOS account without exposing a password to an agent.
4. Render and independently review the project bundle.
5. Download GitHub's current official macOS runner archive from the immutable release URL.
6. Verify the presented version, regular-file shape, absence of symlinks, byte count, and official SHA-256 before extraction.
7. Create the root-protected instance parent, then extract into its new, private, fail-if-exists mutable runner child; never overlay an existing runner and never use `--replace`.
8. The owner obtains a short-lived repository registration token and enters it privately with shell history disabled. The toolkit never receives it.
9. Register only the exact repository with the unique custom label and work directory.
10. Install only the official user-level runner service for the chosen account; no root service or LaunchDaemon.
11. Install the reviewed project-specific static controller through a root transition helper.
12. Run one exact-head commissioning job, verify receipts and residue, compare unrelated runners, and record evidence.

Official runner auto-update remains enabled unless a separate reviewed policy decides otherwise. It may mutate the runner-owned application binaries, but it cannot replace the root-protected instance parent, immutable `.env`/`.path`, bootstrap hooks, or controller generations. The recorded minimum/current version is a commissioning observation, not a permanent byte lock.

### 9.2 Root transition helper

Every install or upgrade helper is generated for one exact instance and one exact old/new state. It:

- requires root and accepts no broad target arguments;
- requires only the target runner service stopped and its Listener/Worker absent;
- targets only the instance's root-protected controller and, during a separately declared bootstrap transition, its protected activation files;
- validates the complete ancestor chain and exact existing/replacement identities;
- preserves the previous active-manifest bytes and metadata;
- stages a complete immutable generation, verifies its manifest and every member, recursively syncs it, then activates it through the single atomic `ACTIVE.json` rename and directory `fsync` described in section 6.4;
- records an explicit transition journal outside the active namespace and supplies fail-closed interrupted-transition recovery;
- leaves the prior immutable generation intact for rollback;
- leaves registration files, credentials, runner binaries, and all other runner roots untouched; `.env` and `.path` remain untouched for ordinary generation updates;
- never starts the service;
- emits bounded hashes and status only.

The generated rollback helper verifies both generation manifests and atomically replaces only `ACTIVE.json` with the preserved previous generation binding. Bootstrap or activation-file changes are a rarer, separately reviewed transition with their own byte-for-byte rollback package.

### 9.3 Removal

Removal is deliberately two-step. First, the owner uses GitHub's supported API/UI flow to deregister the exact recorded runner ID; the toolkit never accepts or handles the removal token. Only after a fresh inventory proves that exact registration absent may the generated local helper remove the exact instance. It never removes by a shared label, organization scope, or path glob. Local cleanup is bound to the exact instance root and refuses symlinked, foreign-owned, or unexpected content.

## 10. Routine Operations

- After qualification, leave the routine runner online so repository CI works automatically without per-job owner action.
- Stop it only for controller maintenance, residue quarantine, a security incident, an explicitly isolated benchmark, or reboot qualification.
- A job routed to an unavailable unique label remains queued and GitHub fails it after 24 hours. Operators must treat a long queue as an offline/routing incident rather than dispatching duplicates.
- A user-level LaunchAgent may require the runner account to log in after a reboot. Reboot/login/reconnect qualification is tracked per instance.
- Review diagnostic logs and disk growth periodically; persistent runner diagnostics are not deleted by job cleanup.
- Job hooks execute synchronously and GitHub supplies no hook timeout. The controller's internal deadline is therefore part of the availability boundary.
- A cleanup failure quarantines that runner instance. Do not rerun automatically or weaken the guard.
- Check the unique-label queue before maintenance and take two idle observations before mutating a runner root.
- Recheck for any global `Runner.Worker` immediately before and after host mutation. Never stop another project's worker to create a maintenance window.

## 11. Deterministic Artifact Contract

The project produces two artifact classes:

1. a complete toolkit-source ZIP containing the reviewed source, schema, templates, scripts, tests, documentation, adoption prompt, validation locks, and reviewed `.github` governance/workflow files needed to reproduce rendering;
2. a separately rendered per-instance ZIP containing only one non-secret project contract and its generated adoption files.

Neither archive is built from a live runner root. A sample instance is test evidence, not a substitute for the transferable toolkit-source archive.

The canonical ZIP profile is ZIP32 with stored/uncompressed members (`STORE`), no data descriptors, a fixed DOS timestamp, UNIX creator/version fields, explicit regular-file modes, zero extra fields, zero member/archive comments, no encryption, no explicit directory entries, and bytewise-sorted portable ASCII member paths using `/`. Generated member paths are ASCII by construction. The builder sets every local and central-directory field explicitly rather than relying on environment defaults. Fixed limits are 256 total members including both control files, 8,388,608 bytes per member, 67,108,864 uncompressed bytes, and 83,886,080 archive bytes.

Before writing or extracting, the verifier accepts regular-file members only and rejects duplicate names, symlinks, hard-link or special-file metadata, absolute paths, empty components, `.`/`..`, backslashes, control characters, non-ASCII names, macOS case-fold collisions, Unicode-normalization aliases, unsupported flags, inconsistent local/central headers, ZIP64, and configured count/size expansion limits. It opens an input archive once, retains the exact bounded verified bytes in an immutable verification result, and extraction consumes only that result; replacing the source path after verification cannot affect extracted bytes. Extraction reparses the retained bytes and uses descriptor-relative creation in a new fail-if-exists directory rather than `extractall`.

The builder:

- applies the canonical ZIP profile above;
- excludes ownership-dependent metadata, extended attributes, credentials, logs, workspaces, caches, and host evidence;
- writes canonical UTF-8 JSON with sorted keys and a final newline;
- records every payload member path, mode, byte count, and SHA-256 in `MANIFEST.json`; the manifest explicitly excludes itself and `SHA256SUMS` to avoid a circular self-hash;
- creates `SHA256SUMS` containing every payload member plus `MANIFEST.json`, while excluding only `SHA256SUMS` itself;
- verifies the archive by extracting to a fresh directory, comparing every payload member to the manifest, validating `MANIFEST.json` through `SHA256SUMS`, and validating the full archive through the adjacent archive-digest record;
- scans the final extracted bytes and archive member names for secrets and forbidden live-state files.

The durable local output is written under:

```text
<project-root>/dist/
```

Each filename contains the toolkit version, artifact class or instance slug, and manifest digest prefix. A separate adjacent archive-digest record binds the complete ZIP, including both control files. A SHA-256 binds bytes to the recorded acquisition path; it is not a signature.

## 12. Validation Strategy

### 12.1 Unit and safety tests

The implementation must prove rejection of:

- overlapping paths, labels, prefixes, or service identities;
- account reuse under `dedicated-account`, while permitting explicitly acknowledged shared-account reuse only with otherwise disjoint instances;
- root, `/Users`, a home root, a sibling runner root, or another user's path as a cleanup target;
- symlinked roots, nested symlink ancestors, symlinked candidates, hard-linked files, foreign ownership, wrong device, wrong `0711` protected-ancestor or `0700` mutable-root modes, writable static roots or ancestors, allow ACLs, mutable activation files, and path traversal;
- a missing or insufficient `/usr/bin/python3` descriptor/no-follow capability;
- malformed run IDs, attempts, job names, and legs;
- uncontrolled depth, entry count, byte count, or elapsed time;
- broad deletion and process-killing primitives;
- generic self-hosted routing, missing unique label, `pull_request_target`, write permissions, secrets, comments, deployments, unpinned Actions, credential-persisting checkout, global cross-PR concurrency, parallel use of one runner, uncontrolled caches, and artifact upload;
- mixed or partially staged controller generations, invalid active manifests, interrupted-transition journals, and activation of an unsynced generation;
- bundle members absent from or extra to the manifest, wrong mode, wrong size, wrong hash, non-deterministic order, noncanonical ZIP fields, duplicate/unsafe member names, special members, header disagreement, or non-normalized timestamp;
- secret-like values and forbidden live-runner files in the final ZIP.

### 12.2 Positive tests

The implementation must prove:

- valid dedicated and shared-account contracts render deterministically;
- a runner account can traverse its exact protected instance path but another local account cannot list or read any `0700` mutable child;
- pure rendering remains byte-identical regardless of host inventory, while the separate host verifier blocks a live collision;
- two project instances are disjoint and project A cleanup cannot address or remove project B data;
- before cleanup removes valid stale roots only;
- after cleanup removes all current-run leg roots and leaves another run untouched;
- interrupted-job recovery and repeated cleanup are safe;
- unrelated GitHub temp entries and runner-managed roots remain untouched;
- workflow policy passes only for the fully guarded form;
- a fully staged immutable controller generation activates through one atomic manifest switch, survives simulated interruption, and rolls back through the same mechanism;
- transition and rollback helpers pass Bash syntax and mutation tests;
- the same input produces byte-identical directory trees, manifests, toolkit-source ZIPs, and per-instance ZIPs across supported builder environments.

### 12.3 Project adoption qualification

Generic tests do not qualify a real runner. Every adopting repository separately proves:

- exact repository scope, label, account, roots, and service identity;
- exact-head workflow and job identities;
- a successful project-specific CI canary;
- started/completed cleanup receipts;
- no residue under bounded instance roots;
- unchanged unrelated runner registrations, labels, services, roots, and listeners;
- expected online/offline behavior and, when scheduled, reboot/login/reconnect behavior.

## 13. Adoption Agent Contract

`AGENT_PROMPT.md` will instruct another Codex agent to:

1. inventory live state read-only;
2. verify authorization and select a profile honestly;
3. fill a schema-valid instance contract without guessing paths or identities;
4. render and test in an isolated project worktree;
5. request owner action only for password/token/root ceremonies;
6. never copy another project's constants or modify another runner;
7. stop after two similar failures and return an RCA instead of adding speculative machinery;
8. obtain exact-head CI and an independent review before merge;
9. keep a project-specific evidence record without secrets;
10. leave the runner online after successful routine qualification unless the instance policy says otherwise.

The prompt will distinguish repository changes, host changes, and owner-only steps so an agent cannot mistake documentation or an online runner for authorization.

## 14. Delivery Phases

After this design is approved:

1. Write a detailed implementation plan using the Superpowers planning workflow.
2. Implement schema, renderer, cleanup library, hooks, policy checks, bundle builder, docs, and tests in this standalone repository.
3. Run independent code, security, cleanup-safety, and deterministic-artifact reviews.
4. Build and verify the complete toolkit-source ZIP and one separate non-secret sample per-instance ZIP.
5. Provide the owner with both ZIPs, their manifests and archive digests, the exact local source commit/tree, and an adoption prompt.
6. Do not publish an executable release or package until release governance, security validation, and the release process receive separate approval.
7. Do not retrofit any live project automatically. Each project adopts through its own reviewed PR and bounded host ceremony.

## 15. External References

The implementation and adoption guides will track current official GitHub documentation and re-verify it before any real registration:

- [Security for GitHub Actions](https://docs.github.com/en/actions/reference/security/secure-use)
- [Self-hosted runners reference](https://docs.github.com/en/actions/reference/runners/self-hosted-runners)
- [Running scripts before or after a job](https://docs.github.com/en/actions/how-tos/manage-runners/self-hosted-runners/run-scripts)
- [Using self-hosted runners in a workflow](https://docs.github.com/en/actions/how-tos/manage-runners/self-hosted-runners/use-in-a-workflow)
- [Using labels with self-hosted runners](https://docs.github.com/en/actions/how-tos/manage-runners/self-hosted-runners/apply-labels)
- [Adding self-hosted runners](https://docs.github.com/en/actions/how-tos/manage-runners/self-hosted-runners/add-runners)
- [Configuring the self-hosted runner application as a service](https://docs.github.com/en/actions/how-tos/manage-runners/self-hosted-runners/configure-the-application)
- [Monitoring and troubleshooting self-hosted runners](https://docs.github.com/en/actions/how-tos/manage-runners/self-hosted-runners/monitor-and-troubleshoot)
- [Removing self-hosted runners](https://docs.github.com/en/actions/how-tos/manage-runners/self-hosted-runners/remove-runners)

## 16. Deferred Decisions

The public repository name, owner, MIT license, initial contribution guidance, and private vulnerability-reporting route are now established. These remaining decisions are intentionally deferred and do not block local implementation after spec approval:

- formal maintainer and release governance;
- signed releases or attestations;
- public package distribution;
- optional VM/JIT providers;
- zero-login boot through a LaunchDaemon;
- integration with Linear, PMC, or another external tracker.

No deferred item may be silently claimed as implemented or supported.
