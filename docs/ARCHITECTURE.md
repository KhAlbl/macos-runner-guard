# macOS Runner Guard — Architecture Design

**Status:** Approved architecture; implementation pending

**Date:** 2026-08-30

**Project:** `macos-runner-guard`

**Repository:** `https://github.com/KhAlbl/macos-runner-guard`

**Scope:** reusable tooling and guidance for safely operating multiple project-dedicated GitHub Actions runners on one macOS host

## 1. Decision Summary

Build a standalone, project-neutral toolkit that renders a separately bound runner package for each repository. It generalizes lessons from a qualified internal runner—dedicated routing, private job storage, automatic residue cleanup, fail-closed path validation, workflow policy checks, deterministic evidence, and rollback—without copying project-specific identities or modifying an installed runner.

The toolkit supports two explicit profiles:

1. **`dedicated-account` — recommended.** Each independent trust domain gets a non-admin macOS account and a private home. This gives useful credential and filesystem separation between projects on the same Mac.
2. **`shared-account-trusted-only` — compatibility profile.** Multiple trusted repositories may run under one existing macOS account, each with disjoint paths and labels. This improves routing and residue control but provides **no privacy or credential isolation** between runners sharing that account. A peer Listener may remain online during a bounded window only through a complete exact idle-peer binding; every peer Worker or unbound Listener blocks the window.

The toolkit is not a replacement for a VM, a dedicated host, or an ephemeral macOS provider. Persistent cleanup reduces accidental cross-job residue; it cannot contain hostile code running as the same user.

The owner approved publication as `KhAlbl/macos-runner-guard` under the MIT License. Public availability does not authorize an executable release; implementation, release governance, and production-support claims remain separate gates.

## 2. Goals

- Make adoption reproducible for independent repositories without embedding any one project's identities.
- Give every accepted project one organization-scoped runner registration in a project-dedicated organization runner group, a unique label, unique runner root, unique work root, unique temporary root, and unique cleanup namespace.
- Keep the runner online for routine trusted CI after qualification; stop it only for maintenance, quarantine, a security incident, a special isolation window, or reboot qualification.
- Generate deterministic, reviewable files and a deterministic adoption ZIP with a two-level inventory that covers every payload file by path, mode, byte count, and SHA-256 without a self-hash cycle.
- Keep registration credentials outside the toolkit, repository, logs, evidence, command history, and agent context.
- Make unsafe or ambiguous cleanup fail closed without deleting evidence.
- Preserve all other runners: no label-wide operations, organization-wide mutations, broad process termination, shared root rewriting, or implicit service changes.
- Make the implementation small enough to audit and portable enough that another Codex agent can adopt it with a bounded project-specific plan.

## 3. Non-Goals

- Kubernetes, Actions Runner Controller, GARM, Tart, Anka, Docker-based macOS virtualization, or a custom JIT control plane.
- A shared organization runner, a default/unrestricted runner group, or one runner registration used by many repositories.
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

Anyone able to place workflow code on a ref that is permitted to use the persistent runner can execute code as the runner's macOS account. Repository scope and labels control scheduling; they are not host isolation. A job-level condition inside a pull-request workflow is not a security boundary because the proposed workflow bytes may themselves come from the pull request.

All runner instances share the physical Mac's CPU, RAM, storage device, network, kernel, process table, power, and thermal envelope. A resource-heavy project can affect another project's timing even when filesystem identities are separate.

The rendered persistent-runner lane therefore accepts only a **private organization-owned repository** whose one project-dedicated organization runner group is selected to that repository, denies public repositories, and is restricted to the exact adopting-repository workflow `.github/workflows/macos-runner-guard.yml` pinned to `refs/heads/<protected-default-branch>`. The workflow itself accepts only pushes to that branch after review and hosted validation. A unique label and push-only YAML are necessary routing controls, but are not authorization controls: proposed PR bytes can edit or add workflow YAML. Public repositories, user-owned repositories, and organization repositories for which this complete group policy cannot be proved are hosted-only in version 1 and must not register or expose a persistent runner through this toolkit.

External, fork, and same-repository PR validation runs on GitHub-hosted infrastructure. The runner-group workflow restriction prevents another workflow proposed by a pull request from selecting the persistent runner even if it knows the labels. A maintainer may merge reviewed bytes through the protected branch, whose resulting push of the selected workflow is the first persistent-runner execution. Secret-bearing release jobs, signing jobs, deployment jobs, and workloads requiring hostile-code containment must use a separately authorized isolated environment. GitHub's own guidance recommends self-hosted runners only with private repositories and warns that persistent runners can be compromised by untrusted workflow code; the group restriction narrows which trusted workflow may schedule this runner but does not create VM isolation.

### 5.2 `dedicated-account`

Required properties:

- one non-admin macOS account per independent trust domain;
- home directory owned by that account, mode `0700`, and not a symlink;
- complete group evidence proving no `admin`/`wheel` membership and a fresh noninteractive sudo probe proving no passwordless sudo;
- no project GitHub login, SSH key, cloud credential, signing identity, or inherited maintainer keychain credential;
- project-mutable runner, work, temporary, cache, and generated evidence roots that are real directories, owned by that account, mode `0700`, free of allow ACLs, and located beneath a project-specific root-protected instance parent;
- a private organization-owned target repository plus one non-default, project-dedicated organization runner group whose repository access is exactly that repository, `allows_public_repositories` is false, `restricted_to_workflows` is true, and `selected_workflows` is exactly `<owner>/<repo>/.github/workflows/macos-runner-guard.yml@refs/heads/<protected-default-branch>`;
- one organization-scoped runner registration assigned only to that group and a unique custom label.

Dedicated adoption also proves both directions of the local filesystem boundary without enumerating peer contents: another local account cannot list/read a runner-owned `0700` mutable child, and the runner UID cannot traverse, list, read, or write the maintainer home or any unrelated runner root. Peer paths are represented in portable/redacted evidence only by an owner-reviewed role plus a domain-separated salted path digest; the owner-held exact-path map is used only by the authorized host verifier and is bound by digest. Unknown access state blocks the dedicated privacy claim, and the toolkit never changes peer permissions automatically.

This profile reduces cross-project credential and filesystem exposure. It does not prevent code from attacking other processes or shared host resources through operating-system vulnerabilities or deliberately granted permissions.

### 5.3 `shared-account-trusted-only`

Required properties:

- separate runner, work, temp, workspace, static-controller, and job-prefix paths per repository;
- the same private organization-owned repository, dedicated organization-runner-group, exact selected-workflow, and public-repository-denial boundary as the dedicated profile;
- a unique organization-scoped registration and custom label;
- same workflow and cleanup controls as the dedicated profile;
- every peer `Runner.Listener` present during verification, transition, registration, commissioning, or qualification bound to an exact official executable/root/service/remote-runner/group/workflow/label identity, authenticated online-idle state, disjoint local paths, and complete queued-job absence; every peer Worker or unbound/mismatched Listener is rejected, and a label alone is never enough;
- explicit acknowledgement that every runner under the account can potentially access the account's files, credentials, caches, keychain, agents, and sibling runner registration material.

This profile may be used for routing and reliability among repositories trusted at the same level. Accepting a bound idle peer is an operational compatibility rule, not isolation; a same-UID peer can still access sibling state, and any Worker or incomplete binding blocks the window. It must never be described as private, isolated, credential-safe, or equivalent to a dedicated account.

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
│   ├── bootstrap-py39.txt ... bootstrap-py314.txt
│   ├── validation.in
│   ├── validation-py39.txt ... validation-py314.txt
│   └── metadata/pypi-wheel-metadata.json
├── src/runner_guard/
│   ├── __init__.py
│   ├── __main__.py
│   ├── activation.py
│   ├── activation_files.py
│   ├── archive.py
│   ├── bindings.py
│   ├── bootstrap.py
│   ├── bundle_runtime.py
│   ├── capabilities.py
│   ├── cleanup.py
│   ├── cli.py
│   ├── components.py
│   ├── contract.py
│   ├── errors.py
│   ├── fsops.py
│   ├── generation.py
│   ├── git_source.py
│   ├── inventory.py
│   ├── job_identity.py
│   ├── jsonio.py
│   ├── manifest.py
│   ├── operation_package.py
│   ├── policy.py
│   ├── registration.py
│   ├── render.py
│   ├── residue.py
│   ├── scanner.py
│   ├── transition.py
│   ├── upstream_runner.py
│   ├── workspace.py
│   └── zip_format.py
├── schemas/instance-v1.schema.json
├── templates/
│   ├── activate.py
│   ├── job-started.sh
│   ├── job-completed.sh
│   ├── install-instance.sh
│   ├── recover-instance.sh
│   ├── remove-instance.sh
│   ├── restore-instance.sh
│   ├── rollback-instance.sh
│   └── rendered/
│       ├── workflow-guard.yml
│       ├── ADOPTION_CHECKLIST.md
│       └── AGENT_PROMPT.md
├── examples/
│   ├── dedicated-account.json
│   └── shared-account-trusted-only.json
├── scripts/
│   ├── build-operation-package.py
│   ├── build-operation-recovery-authority.py
│   ├── build-operation-request.py
│   ├── build-operation-stager.py
│   ├── plan-global-bootstrap.py
│   ├── monitor-registration.py
│   ├── build-runner-installation-evidence.py
│   ├── build-components.py
│   ├── check-committed-diff.py
│   ├── render-instance.py
│   ├── verify-instance.py
│   ├── build-adoption-zip.py
│   ├── prepare-operation-input.py
│   ├── inventory-runner-archive.py
│   ├── verify-runner-archive.py
│   ├── inventory-runners.py
│   ├── audit-residue.py
│   └── validate.py
├── tests/
│   ├── fixtures/components/...
│   ├── support/tree_snapshot.py
│   ├── test_activation.py
│   ├── test_activation_files.py
│   ├── test_archive_extraction.py
│   ├── test_archive_format.py
│   ├── test_archive_rejections.py
│   ├── test_artifact_scanner.py
│   ├── test_bindings.py
│   ├── test_bootstrap.py
│   ├── test_bundle_determinism.py
│   ├── test_capabilities.py
│   ├── test_cleanup.py
│   ├── test_components.py
│   ├── test_cleanup_contract.py
│   ├── test_cleanup_integration.py
│   ├── test_cleanup_render.py
│   ├── test_cli.py
│   ├── test_ci_diff.py
│   ├── test_contract.py
│   ├── test_documentation_contract.py
│   ├── test_end_to_end.py
│   ├── test_fsops.py
│   ├── test_generation.py
│   ├── test_hooks.py
│   ├── test_host_verification.py
│   ├── test_inventory_cli.py
│   ├── test_job_identity.py
│   ├── test_jsonio.py
│   ├── test_lifecycle_integration.py
│   ├── test_lifecycle_runtime_bundle.py
│   ├── test_manifest.py
│   ├── test_operation_package.py
│   ├── test_public_inventory.py
│   ├── test_removal.py
│   ├── test_render.py
│   ├── test_registration.py
│   ├── test_requirements_policy.py
│   ├── test_residue.py
│   ├── test_runtime_bundle.py
│   ├── test_transition.py
│   ├── test_upstream_runner.py
│   ├── test_workflow_policy.py
│   └── test_workspace.py
├── docs/
│   ├── ADOPTION.md
│   ├── CONTRACT.md
│   ├── OPERATIONS.md
│   ├── SECURITY_MODEL.md
│   ├── TROUBLESHOOTING.md
│   ├── WORKFLOW_POLICY.md
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
- a closed bytewise-sorted tuple of complete approved Action `owner/repository@SHA` identities;
- platform and architecture;
- cleanup entry, depth, byte, and time bounds;
- a measured project-specific free-disk floor;
- service identity pattern;
- routine availability policy;
- whether reboot/login/reconnect has been qualified.

Pure contract validation and rendering are host-independent. They reject:

- `/`, `/Users`, a home directory itself, or a root declared by another supplied instance contract as a mutable target;
- overlapping mutable roots declared in the supplied contract set;
- duplicate project slugs or any collision in the derived global `operations/<project-slug>/` namespace across the supplied contract set;
- path traversal, unresolved variables, non-absolute paths, and control characters;
- account/home/profile combinations that contradict the claimed trust level;
- any `temp_root` other than the exact GitHub runner temp location `work_root / "_temp"`;
- any workspace other than `work_root / github_repository / github_repository`, or any overlap with temp, guarded-job, action, diagnostic, tool, cache, controller, runner, evidence, or global operation-package namespaces;
- a `minimum_free_bytes` value with no recorded adopting-project measurement evidence. The pure validator checks only the explicit positive value; the adoption review verifies the measurement provenance.

Host-bound verification has nine explicit states. Six form the normal adoption path: `proposal_precreation`, `proposal_ready`, `filesystem_prepared`, `registered_pretransition`, `installed_precommissioning`, and `commissioned`. `verify_proposed_identifiers()` proves the fresh account and target identities absent without fabricating a UID; `verify_proposed_adoption()` proves the created account, two-way peer boundary, exact eligible organization group with an empty runner list, and zero target registration before filesystem preparation; `verify_filesystem_prepared()` proves the exact empty roots created by the bounded preparation operation while registration, service, and controller remain absent; `verify_registered_pretransition()` joins one exact local name/root/service binding to the sole authenticated organization runner in the accepted project-dedicated group and binds the official pre-controller `.env`/`.path`; `verify_installed_precommissioning()` proves final controller/activation bytes without requiring a job that has not run; and, after exactly one bounded commissioning job, `verify_commissioned_binding()` adds same-registration reconnect/online-idle evidence and alone qualifies routine persistence. Three additional states prevent lifecycle deadlocks: `decommission_ready` proves the historically bound registration and service absent and the dedicated group empty while the complete stopped instance remains verified; `quarantined` proves the same registration/group absence, the live instance absent, and one exact same-parent quarantine binding; and `recovery` binds a fresh host-transition recovery report to an allowed originating operation/state and the previous accepted envelope without pretending a partially transitioned host still satisfies its old normal state.

Every state binds authenticated repository metadata to the contract's exact default branch and a separately reviewed protected-branch policy. Complete repository-ruleset evidence must prove active exact-branch inclusion, zero bypass actors, deletion/non-fast-forward protection, at least one approving review, stale-review dismissal, a different actor's approval after the latest push, conversation resolution, strict current-branch checks, and the exact non-empty hosted-check `(context, integration_id)` tuple. An inactive/weaker/excluding ruleset, omitted/extra/wrong-integration check, incomplete endpoint, or current live configuration that lacks those controls blocks external implementation and persistent-runner adoption. All nine states walk declared ancestor chains without following links and consume a fresh, redacted inventory.

Persistent adoption additionally requires authenticated metadata proving the repository is private and organization-owned, plus complete organization runner-group, selected-repository, workflow-restriction, organization-runner, per-group runner, and selected-repository endpoint evidence. The one non-default group must be selected to exactly the target repository, deny public repositories, and restrict access to exactly `<owner>/<repo>/.github/workflows/macos-runner-guard.yml@refs/heads/<default-branch>`. The target runner must be absent in proposal/filesystem-prepared/decommission/quarantined states, and must belong to exactly that group in registered/installed/commissioned states and applicable recovery states; unexpected presence in an absence state or missing/multiple membership in a presence state fails closed. The destination workflow path is a version-1 constant, not a contract- or caller-selected path; the renderer's `workflow-guard.yml` is integrated and independently policy-checked at that exact repository path. A user-owned/public/internal repository, default or broadly visible group, extra repository/workflow, mutable or ambiguous workflow ref, wrong state-specific membership/absence, or incomplete group capability is hosted-only and fails persistent adoption. `selected_workflows` is a field of the complete runner-group response, not a separate invented endpoint. Embedded endpoint/complete strings are consistency data, not authority: the root-owned transition package must bind response hashes plus pagination, HTTP-status, and API-version facts from a separately reviewed authenticated acquisition procedure. Repository/group/scope/runner-ID authority never comes from invented local fields or private `.runner` metadata. A separately reviewed policy is bound to the exact macOS build and minimal Apple platform GUI-session process identities. Process evidence serializes only basename, signing identifier/team/platform flag, and executable hash—never path, argv, environment, or open-file data—and rejects shells, agents, applications, unknown processes, and foreign Listeners/Workers. For `dedicated-account`, any unrelated live reuse of the proposed account is a collision. For `shared-account-trusted-only`, acknowledged account reuse is allowed only when roots, labels, name prefixes, and services are disjoint; a peer Listener may remain only through a complete `PeerIdleRunnerBinding` joining its exact official process/root/service/local binding to authenticated online-idle organization runner/group/workflow evidence and queued-job absence, while every peer Worker or unbound Listener is rejected. Process evidence separately accepts only the exact target process identities allowed by the named state. The toolkit never stops a peer automatically. Failure or lack of permission to prove the required state blocks the operation. This is process hygiene, not same-UID isolation. Host inventory never changes rendered bytes and is never embedded in a portable ZIP.

### 6.3 Generated project bundle

For an instance named `<project-slug>`, the renderer creates:

```text
instances/<project-slug>/
├── instance.json
├── activate.py
├── cleanup_controller.py
├── job-started.sh
├── job-completed.sh
├── transition_controller.py
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

The renderer output is a closed 16-member inventory with exact modes:

| Rendered path | Exact mode |
|---|---:|
| `instance.json` | `0444` |
| `activate.py` | `0555` |
| `cleanup_controller.py` | `0555` |
| `job-started.sh` | `0555` |
| `job-completed.sh` | `0555` |
| `transition_controller.py` | `0555` |
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

The pure renderer receives four immutable inputs: the validated contract, the exact eleven already-opened component payloads, the exact three already-opened payloads from the dedicated rendered-template directory, and the exact already-opened reviewed workflow-policy source bytes. The public deterministic precursor is `scripts/build-components.py --source-root REVIEWED_SOURCE_ROOT --output ABSENT_COMPONENT_DIRECTORY`: it descriptor-loads the fixed cleanup/runtime/lifecycle source inventory from that explicit root, calls the two owned child builders, merges through the one shared render helper, and commits exactly eleven mode-`0555` files to an absent output transaction. It discovers no current-directory or installed-package source. The final clean-build procedure separately proves `REVIEWED_SOURCE_ROOT` is the exact clean reviewed commit/tree. The component set includes a deterministic standalone `transition_controller.py` bundle built from the closed reviewed lifecycle module set; root wrappers invoke that bundled runtime through `/usr/bin/python3 -I -S` and never import the source checkout or an installed package. CLI loaders retain no-follow directory/file descriptors, bind every member and byte before rendering, and never validate then reopen a mutable pathname. The renderer never discovers policy or template bytes through package paths, import state, the current directory, or live host state. A closed per-component marker map is rendered in topological order: contract-derived paths first, then the canonical initial-generation manifest, then a non-self-referential installation-input digest that excludes `install-instance.sh` and artifact controls, then the install helper. Host evidence and the final artifact manifest are bound later by a root-owned execution package and never alter portable output. The generated policy embeds canonical contract bytes through a deterministic hexadecimal bytes expression—not raw JSON as Python syntax—and its standalone CLI parses those bytes before evaluating a bounded workflow input. The renderer creates this payload only. Fresh per-instance artifact staging then adds `MANIFEST.json` and `SHA256SUMS` under the deterministic artifact contract in section 11. The resulting archive is self-contained but remains a proposal until a project-specific review verifies repository workflows, host paths, account state, runner inventory, and registration scope.

### 6.4 Root-protected instance and controller generations

Each project gets a root-protected instance parent outside the runner account's home, runner application, work, cache, and temporary trees. The default location is:

```text
/Library/Application Support/macos-runner-guard/instances/<project-slug>/
```

The global toolkit parent, its `instances/` and `operations/` children, each project instance parent, and each `operations/<project-slug>/` parent are real, non-symlink `root:wheel` directories with exact mode `0711`: runner accounts may traverse an exact known path but cannot list, create, rename, or remove entries. They have no allow ACL, xattr/resource fork, or prohibited/unknown flag. Every controller ancestor below the instance parent uses the same root ownership, no-ACL rule, and mode `0711` unless a stricter non-writable mode remains traversable. Every operation-package top-level state—staging, current, completed, quarantine, recovery, and failed archive—is root-owned mode `0700`, so retirement is a same-mode atomic rename with no hidden chmod state. Completed/failed records are immutable only through the closed version-1 interface; root could still modify their mode-`0700` bytes, while the runner account can access neither. The host-bound verifier rechecks owner, type, mode, ACL, flags, xattrs, symlink absence, and positive traversal through the applicable complete ancestor chain.

The project instance parent contains these mutable children:

- `runner/`, `work/`, `cache/`, and redacted operational output as real, non-symlink directories owned by the runner account, exact mode `0700`, with no allow ACL and created under `umask 077`;
- the contract's `temp_root` is not a sibling directory: it is exactly `work/_temp`, matching GitHub Runner's `$RUNNER_TEMP`, and is a real, non-symlink directory owned by the runner account, exact mode `0700`;
- `controller/` and every generation ancestor owned by `root:wheel`, mode `0711`, no allow ACL, and never writable by the runner account; controller files use exact reviewed `0444` or `0555` modes.

The root-owned instance parent prevents the runner account from renaming or replacing the complete runner root. The runner's `.env` and `.path` activation files must be regular, non-symlink, single-link, `root:wheel`, mode `0444`, and immutable, with bytes produced by the lifecycle's deterministic activation-file builders. `.env` contains exactly the two sorted hook assignments, each terminated by one LF, and no other line:

```text
ACTIONS_RUNNER_HOOK_JOB_COMPLETED=<absolute-controller-bootstrap>/job-completed.sh
ACTIONS_RUNNER_HOOK_JOB_STARTED=<absolute-controller-bootstrap>/job-started.sh
```

`.path` contains exactly one system-first line with one trailing LF: `/usr/bin:/bin:/usr/sbin:/sbin:/opt/homebrew/bin` for `ARM64`, or `/usr/bin:/bin:/usr/sbin:/sbin:/usr/local/bin` for `X64`. Their paths remain inside the application root because GitHub reads them there, but the protected parent and immutable flags prevent replacement. Registration is performed under that same sanitized PATH and with no selected environment values so the official runner initially creates a known empty `.env` and the expected `.path`; the reviewed bootstrap transition then replaces only those known activation bytes, binds their SHA-256 values in transition evidence, and provides byte-for-byte rollback. Changes require the service to be stopped and a separately reviewed root transition. GitHub hook targets themselves live under the separate root-owned controller root, not in the runner application directory. These activation files are transition outputs, not members of the closed portable 16-file renderer payload.

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

The minimal bootstrap hooks are stable, root-owned `/bin/bash` wrappers named by `.env`; they invoke root-owned `activate.py` through `/usr/bin/python3 -I -S`. The toolkit builds that bootstrap as a deterministic, standard-library-only standalone program, so it never imports an installed package or user/site path at runtime. The bootstrap opens and validates the root-owned `ACTIVE.json`, binds the declared immutable generation directory and manifest by descriptor and inode/device identity, opens the selected cleanup controller without following links, hashes and compiles the already-opened verified bytes, executes them in an isolated namespace, and calls one explicit entry point while retaining the descriptor binding. A path swap after verification cannot change the executed bytes. A generation is staged in a root-owned mode-`0700` directory, hashed, recursively synced, and finalized as a root-owned mode-`0711` directory before it can be activated. `GENERATION_MANIFEST.json` is regular, single-link, root-owned, and mode `0444`; the two generation payloads retain their frozen `0444`/`0555` modes. `ACTIVE.json` records the generation name and manifest hash and is replaced through one same-directory atomic rename followed by directory `fsync`. No mutable controller binary is shared between projects.

An interrupted transition cannot expose a partially staged generation: bootstrap ignores staging names and reads only `ACTIVE.json`. On startup, an orphan staging entry or transition journal causes a fail-closed recovery report; the active generation remains unchanged if its manifest is valid. An invalid active manifest never triggers an automatic fallback. A reviewed recovery helper may clean only an exact journal/plan-bound safe known incomplete staging subset, or atomically reactivate the previously recorded generation when the closed recovery state authorizes it. An unjournaled, multiply bound, foreign, extra, or otherwise unsafe orphan is `blocked_ambiguous`, remains preserved, and is inspect-only.

Workflow code verifies the complete protected ancestor chain, `.env`/`.path` byte contracts, immutable flags, bootstrap ownership/mode, active-manifest identity, and non-writability. Transition evidence records exact generation and activation SHA-256 values. Hashes are integrity identifiers, not signatures or authentication.

## 7. Cleanup Contract

### 7.1 Job directory

Every job uses a root of the form:

```text
<temp-root>/<prefix>-<run-id>-<run-attempt>-<github-job>-<leg>
```

The workflow knows the full job and leg name. The completed hook derives only the validated current-run prefix from GitHub's documented default repository, run-ID, and run-attempt variables, allowing it to remove all serial matrix-leg roots for that run and attempt. The before hook needs only the documented repository identity and scans the instance prefix. Hooks never depend on a workflow-defined matrix or custom environment value.

The workflow validates `run-id`, `run-attempt`, job name, and leg with full-match, length, canonical-number, and allow-list rules before any filesystem action. Hook phases validate only the documented fields required by that phase.

### 7.2 Three cleanup layers

1. **Before-job hook:** removes only valid stale directories carrying that instance's prefix under that instance's temp root and a descriptor-bound stale checkout at the exact derived workspace. It is recovery for cancellation, restart, or a failed previous completion hook.
2. **Workflow `if: always()` cleanup:** removes only the exact current leg root and cleans only a validated ordinary checkout belonging to that instance.
3. **Completed hook:** removes every valid job root matching the current run/attempt prefix and repeats the exact workspace cleanup idempotently.

Before checkout, a non-mutating workflow preflight binds the full job identity, verifies `$RUNNER_TEMP` and `$GITHUB_WORKSPACE` equal the exact contract-derived paths, revalidates root security metadata, and measures available bytes with `fstatvfs` on the retained temp-root descriptor. Exactly the contract threshold passes; an unavailable or smaller result fails closed.

The official runner may also clean portions of its work area. The toolkit's cleanup is defense in depth and provides a deterministic policy and receipt.

### 7.3 Filesystem safety

Cleanup uses descriptor-relative, no-follow operations with inode/device revalidation on macOS. There is no canonical-path, `lstat`, or path-recursive fallback. Adoption first runs a capability probe against `/usr/bin/python3`; it requires CPython 3.9 or newer, the Apple Command Line Tools that supply that interpreter, `dir_fd` support for the required `os` operations, `O_DIRECTORY`, `O_NOFOLLOW`, descriptor-based scanning, non-following stat, and stable inode/device fields. If any primitive is absent or behaves differently, adoption and cleanup fail closed.

Before deletion cleanup proves:

- every root canonicalizes to the exact contract path;
- no validated root or ancestor is a symlink;
- each mutable root still has the expected device/inode, runner owner, exact mode `0700`, no allow ACL, and no prohibited filesystem flags at the start of every phase and immediately before mutation;
- target entries are on the expected device and owned by the runner UID;
- regular files do not have unexpected hard links;
- directory traversal stays below configured entry, depth, byte, and time limits;
- the target is below the opened project temp/work root;
- no target is a runner binary, registration file, diagnostic root, action cache, another runner root, another user path, or global temporary path.

Malformed matching entries are preserved and cause a nonzero result. Absence is success. Before and after phases are idempotent.

The controller never uses a broad `rm -rf`, `pkill`, `killall`, label-wide removal, unresolved environment variable, or process-killing rule. Unknown detached processes cause quarantine and operator review; they are not automatically terminated.

Cleanup emits only bounded status receipts and counts. It never prints filenames containing user data, file contents, environment values, credentials, or evidence payloads.

GitHub job hooks have no platform-provided execution timeout. The cleanup controller therefore enforces its own monotonic deadline in addition to entry, depth, name-byte, and byte limits. Every top-level entry—including unrelated GitHub temp entries—is charged before collection or sorting, and every candidate is fully validated and budgeted before the first mutation. A planning-time bound failure preserves the complete target. Once deletion starts, a deadline or filesystem failure may leave only the already validated subset removed. That result is reported as bounded partial cleanup, quarantines the instance, and never claims that the target remained unchanged. No path outside the retained parent can be followed or traversed.

macOS does not provide an inode-conditional unlink primitive. Descriptor binding and final revalidation prevent path escape and symlink following, but cannot promise that a concurrently hostile same-UID process will not replace a leaf between the final stat and unlink. This toolkit is for reviewed code that has reached the protected default branch; unexpected same-UID Workers, shells, agents, or applications block adoption and maintenance. Hostile-code containment requires an ephemeral VM or dedicated host.

## 8. Workflow Guard

The rendered workflow fragment and policy checker enforce these defaults:

- `permissions: contents: read`;
- push to the approved protected default branch only;
- no `pull_request`, `pull_request_target`, `workflow_dispatch`, `schedule`, reusable-workflow, or other event in the persistent self-hosted lane;
- PR and fork validation belongs to a separate GitHub-hosted workflow whose jobs cannot select the project runner label;
- no repository secrets, write permissions, deployment environment, PR comments, or status writes;
- unique labels including `self-hosted`, `macOS`, architecture, and the instance label;
- only exact Action identities from the contract's reviewed closed allowlist; a merely 40-hex but unknown Action, local/Docker action, and reusable-workflow job are rejected;
- checkout cleaning and `persist-credentials: false`;
- no workflow-level or job-level `concurrency` key: GitHub replaces an older pending member of a concurrency group even when `cancel-in-progress` is false, while the one unique runner label already serializes jobs without dropping queued runs;
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

The adoption procedure is an explicit state machine:

1. Audit authenticated repository metadata, current workflows, rulesets, required hosted checks, all registered runners/groups/labels, accounts, roots, services, active workers, disk, ACLs, flags, xattrs/resource forks, and queued jobs read-only. Select and record the trust profile, fixed contract identifiers, and protected-branch policy. This planning observation does not claim an accepted lifecycle state.
2. Before any host mutation or runner-group exposure, render and integrate the complete adopting workflow at the fixed path `.github/workflows/macos-runner-guard.yml` through a hosted-only pull request. Review the full workflow, require its separate hosted checks green, strengthen and reread the exact default-branch ruleset, and merge it to the protected default branch. The public toolkit repository itself never takes this persistent path.
3. The organization owner creates or configures one non-default project-dedicated organization runner group: visibility `selected`, selected repositories exactly the authenticated private target repository, `allows_public_repositories=false`, `restricted_to_workflows=true`, and selected workflows exactly `<owner>/<repo>/.github/workflows/macos-runner-guard.yml@refs/heads/<default-branch>`. Obtain a fresh authenticated read-back of the group, selected repository, and empty per-group runner list. The toolkit verifies owner-exported evidence but never creates or edits the group.
4. Review and cancel any bootstrap workflow run queued by the integration merge before a matching runner exists. Require no queued/waiting/in-progress job able to target the unique label, then run `verify_proposed_identifiers()`. A fresh dedicated account must be proven absent; an acknowledged shared account is handled under its stricter profile. The accepted group is already exact and empty, and every proposed runner/service/path identity must be collision-free.
5. Create or verify the macOS account without exposing a password to an agent, then immediately run `verify_proposed_adoption()` and require the target organization runner absent, complete non-admin/no-passwordless-sudo evidence, the two-way peer filesystem boundary, and every proposal-ready identity collision-free. A failure leaves no runner or instance filesystem; account disposition is a separate owner decision.
6. Render the complete project bundle and build/independently verify its instance artifact plus atomic finalization-evidence directory. Record the independently reviewed SHA-256 of the exact `finalization-identity.json` bytes outside that acquisition path. Invoke only `prepare-operation-input.py prepare`, supplying both the expected source-binding digest and that external finalization-identity digest, to authenticate the evidence before extracting the exact final archive into a fresh private root and atomically creating the matching context directory containing exactly `extraction-report.json`, canonical lifecycle `artifact-identity.json`, and `operation-input-authority.json`. Independently retain the canonical operation-input-authority SHA-256 reported only after the final context/root reread. Every operation-source build must supply both retained expected digests and the complete still-bound context/root; neither value is derived from the context under inspection. A self-consistent replacement artifact/evidence/identity/payload set therefore cannot authorize itself merely by copying the public source binding or selecting a new matching digest. Preserve the root and context together; their device/inode binding is required by every operation-source build. No extracted helper is privileged.
7. If the global toolkit stager is not already installed at the reviewed exact digest, run `plan-global-bootstrap.py` against the operator-private stager file and separately retained expected byte count/SHA-256 into one fresh private absent context. Review the canonical `bootstrap-plan.json`, `bootstrap-state-report.json`, and non-executable `OWNER_COMMANDS.txt`; the transcript must name the final context's plan path, never its hidden transaction sibling. An interrupted context build is abandon-only and retry uses a different fresh private output. The owner pastes exactly the reviewed `/usr/bin/python3 -I -S -c <frozen-inline-program>` command. Its frozen program, final plan path, byte count, and SHA-256 are each encoded as one canonical single-quoted Bash argv word, with apostrophes escaped through the exact quote-boundary form; selected paths containing NUL/control/CR/LF are rejected before rendering. Captured argv must equal exactly `python3 -I -S -c PROGRAM -- PLAN SIZE HASH`, so spaces or shell metacharacters in an otherwise valid private path cannot become syntax. That inline program loads and hashes only the final canonical plan, reopens its sole stager-source parent/leaf no-follow, and accepts no destination/action. Root copies but never executes the source and verifies the staged root-owned copy before publication. It builds the complete closed global toolkit/instances/operations tree under one digest-derived hidden sibling of `/Library/Application Support/macos-runner-guard`, syncs and verifies it, and atomically renames it to the absent final basename. An identical root-initializer retry may remove/rebuild only a safe known incomplete initializer or finish one complete exact initializer; unknown or colliding state is preserved. If the final global tree already exists, the ceremony is verification-only and never enumerates or changes project children. It never creates an instance or project `current` package.
8. Build a `prepare_filesystem` operation source from the preserved complete operation-input context, its exact still-bound extraction root, both independently retained expected context/finalization digests, the canonical `FilesystemPreparationPlan`, and the accepted `proposal_ready` envelope; a copied root, substituted plan, manually transcribed identity/authority, or self-consistent replacement pair is rejected. Then invoke only the installed stager. For this first project operation, the stager atomically initializes the absent root-owned `operations/<project-slug>/` together with its private `evidence/` child and complete `current` package; the global bootstrap does not create project-specific operation paths. Run the exact `stage` endpoint and then the separate exact `execute` endpoint. The bound operation creates only the absent root-protected instance parent and exact empty runner-owned `runner/`, plus `work/`, `work/_temp`, `cache/`, and evidence roots. Run `verify_filesystem_prepared()` and bind its accepted envelope before downloading or registering anything; no registration, service, or controller state exists yet.
9. Download GitHub's current official macOS runner archive from the immutable release URL; bind the reviewed URL, version, regular single-link archive-file shape, byte count, official SHA-256, and complete reviewed tar member manifest while retaining one no-follow descriptor or copying those exact opened bytes into a fresh private `O_CREAT|O_EXCL` staging file. The member manifest records regular files, directories, and the small exact set of relative in-archive symlinks legitimately shipped by GitHub; each link target is normalized, cannot escape, and resolves to a listed regular member. Extract only those same verified bytes, never a reopened pathname. A source-path swap cannot affect extraction.
10. Extract descriptor-safely only into that exact pre-created, descriptor-bound, empty runner directory and write the canonical complete `RunnerExtractionReport` to an absent mode-`0444` file in a private operator-controlled review directory with create-exclusive/sync/reread verification. Record its exact SHA-256 outside the runner tree. Immediately before the registration ceremony, fresh verification must reopen the destination descriptor-relatively, compare every member to the retained archive/expected record and the independently retained report digest, and require no Worker and no unrelated same-UID process unless that exact idle peer Listener is already bound by the reviewed `PeerIdleRunnerBinding` exception described in the next step. `build_registration_ceremony()` first requires the exact accepted state-`filesystem_prepared` envelope and its digest-bound canonical inventory, independently authenticates the complete empty eligible-group evidence and acquisition receipts, and derives every non-secret mutation input: organization URL/account/runner root/unique label from the contract, group ID/name from the sole eligible group, runner name as the contract prefix plus the first 12 hexadecimal characters of the accepted-envelope SHA-256, work directory as the exact lexical `../work` relationship between the validated sibling roots, and `automatic_updates_disabled = true`. A raw caller-supplied substitute for any of those fields is rejected before command rendering. `build_registration_command_plan()` next binds that ceremony to the reviewed archive/report, exact macOS/session policy, at-rest executable identities, and idle-peer bindings in one nonserializable in-memory plan. `render_owner_registration_commands(plan)` deterministically renders the canonically quoted transcript, including the fixed literal `--disableupdate`, and only `build_registration_process_policy(plan, transcript_bytes)` may verify those exact bytes and hash them into the final canonical `RegistrationProcessPolicy`; no partial policy is serializable or accepted. `scripts/monitor-registration.py` observes but never spawns, signals, or records the ceremony; it accepts only the separately reviewed Apple GUI-session baseline plus the owner-controlled single-purpose shell, one generic system `/bin/bash` configuration child and its archive-identity `bin/Runner.Listener` descendant. The command-plan constructor separately verifies the at-rest `config.sh` bytes; the observer proves only executable identity, parentage, and lifetime, and does not claim which script the shell interpreted; the monitor never reads live argv or environment, while the separately reviewed command transcript and post-registration authenticated group read-back bind the intended non-secret ceremony fields. The token value is never received, compared, or recorded by the toolkit. The monitor writes one canonical content-free `RegistrationObservationReceipt` to an absent private output and returns nonzero on any unbound process, peer Worker, foreign descendant, parentage/hash drift, observation gap, or restart; the owner then aborts only the exact ceremony tree and treats the token as exposed until expiry/revocation. An authentic fixture pins the separately verified at-rest `config.sh` identity and the observed reviewed `Runner.Listener` identity; a PTY fake alone is insufficient. The extraction report is integrity evidence, not root protection, authentication, or a substitute for this fresh comparison. Reject a missing, nonempty, replaced, or identity-drifted destination; create every member with fail-if-exists operations. Reject absolute/traversal names, hard links, devices/FIFOs/sockets, unreviewed symlinks, escaping/dangling link targets, member drift, and expansion-limit violations. Never overlay an existing runner and never use `--replace`. If extraction is complete but the process ended before that report was durable, explicit report-bound recovery may reread the complete tree and write only the missing report; it never resets complete verified bytes. Only a verified proper subset may be reset to the same empty root.
11. The organization owner obtains a short-lived **organization** registration token and enters it in a quiet window through a tested single-purpose `/bin/bash --noprofile --norc` ceremony using `read -r -s`, no tracing/transcript/history, and immediate `unset`. The toolkit and agent never receive or print it. Because the official `config.sh --token` interface briefly places the token in a process argument, the owner obtains a host-wide observation immediately before launch and continuously for the exact config-child lifetime: no unrelated `Runner.Worker` anywhere and no untrusted interactive shell/agent/process with same-host process-inspection capability may run. An idle peer Listener under either the same trusted-only UID or another account may remain only when its complete `PeerIdleRunnerBinding` is part of the reviewed registration policy and fresh observation proves the exact process, root, service, authenticated remote online-idle/group/workflow binding, unique labels, and no matching queued job; every Worker or unbound peer Listener blocks the ceremony. Otherwise the owner separately stops that peer without toolkit automation. If the monitor fails, the owner—not the toolkit—terminates only the exact owned ceremony tree, treats the token as exposed until expired/revoked, preserves bounded registration state, and stops. Invoke official `config.sh` only with the exact non-secret arguments rendered from the reviewed canonical `RegistrationCeremony`; the owner does not type or select organization URL, group, account/root, runner name, label, work path, or update mode. The exact command includes `--disableupdate`, never `--replace` or `--ephemeral`. Use the architecture-specific sanitized system-first PATH and no selected environment values. Verify the official runner produced the known pre-transition `.env`/`.path` bytes, immediately unset the token, exit that single-purpose shell, and only then obtain a fresh authenticated read-back proving the exact runner ID is assigned only to that group and retain the accepted registration-observation receipt digest. Run the sole `build-runner-installation-evidence.py` context producer with the exact expected archive/report/ceremony/policy/accepted receipt plus complete routing export and acquisition receipts; it freshly descriptor-compares expected members, writes the canonical comparison receipt and installation evidence atomically, and rejects caller-authored match booleans or digest-only routing assertions.
12. Install only the official user-level runner service for the chosen account, leave it stopped, and never create a root service or LaunchDaemon.
13. Run `verify_registered_pretransition()` and require the accepted project-dedicated group's complete runner list to equal exactly the singleton matching organization runner, that runner assigned to no other group, the stopped service, exact registered roots, known official pre-transition activation bytes, and no controller state.
14. Build a new `install_controller` operation source from the same preserved operation-input context, its independently retained authority/finalization digests, and exact bound extraction root plus the fresh accepted `registered_pretransition` envelope; if either root or context was intentionally discarded, rerun the public bridge into a new root/context with the same independently reviewed finalization-identity digest and independently review/retain the new host-local operation-input-authority digest before proceeding. Then invoke only the installed stager. The bound operation installs the reviewed static controller, initial generation, and final activation bytes and cannot recreate/overwrite runner binaries, registration metadata, service files, work roots, or cache roots. Host evidence remains outside portable rendered bytes.
15. Run `verify_installed_precommissioning()` against the final controller and activation-file identities; this structural state authorizes exactly one bounded commissioning job and requires no post-job fact.
16. Select exactly one reviewed protected-default-branch run for commissioning. This may be an explicitly authorized manual rerun of the cancelled integration run or a later reviewed default-branch push. Require its exact commit/workflow identity and cancel or preserve for review every stale or unexpected unique-label run; never start the service merely because work is queued.
17. Run that one commissioning job, verify receipts and residue, compare unrelated runners, and record the same-registration reconnect/online-idle evidence.
18. Run `verify_commissioned_binding()` against the fresh post-job organization/group inventory and persistence evidence before describing the runner as routinely qualified.

Version 1 disables official runner auto-update during the bound registration ceremony. Every archive-supplied application member remains an exact byte/type/mode/link lock to the reviewed official archive, and every accepted target Listener or Worker must join to the archive-bound executable identity. A complete descriptor walk also applies the versioned closed live-state namespace policy frozen from an authentic configured-and-serviced fixture: exact registration/activation/service metadata shapes and a bounded non-executable diagnostic-log subtree are permitted, credential contents are never opened or serialized, and every unknown/extra file, directory, or symlink fails closed. Any archive-member, namespace-policy, or executable drift blocks commissioning, routine-state verification, recovery, and later host mutation; it is never reclassified as a warning or silently rebound. GitHub requires a disabled-update runner to move to each available release within 30 days and may require an immediate critical security update. The operator must therefore schedule replacement before that deadline: stop and decommission the old registration, quarantine its retained instance, and perform a wholly fresh reviewed adoption of the new archive under a wholly new contract with a fresh unused project slug, routing label, runner-name prefix, instance/runner/work roots, service identity, and operations namespace. The dedicated account and project-specific runner group may be reused only after fresh evidence proves the old service, registration, Listener, and Worker absent and the selected-workflow policy still exact. Version 1 has no in-place runner-binary upgrade, no same-namespace re-adoption, and no route from quarantine restore back to service; those are explicit future-version work. If the deadline or a critical update cannot be met, keep the old runner offline rather than weakening the byte lock.

### 9.2 Root operation staging and host transition

There is no per-change privileged install program. Repository-side tooling produces one immutable operation source bound to a canonical `OperationRequest`, the verified instance artifact/extraction identity, and a fresh accepted host-verification envelope. Fixed no-argument wrappers select no path or action; the reviewed transition authority inside the operation package is the sole selector.

No extracted, runner-owned, or caller-writable helper is executed with `sudo`. The separately reviewed one-time bootstrap is an owner-entered sequence of fixed built-in commands, not a toolkit helper run as root. It accepts only the operator-private stager source file and its independently retained expected size/hash, copies but never executes those bytes, and has a fixed destination. It constructs the complete closed global root, toolkit/version/stager, empty `instances/`, and empty `operations/` under the digest-derived hidden sibling of the existing `/Library/Application Support` parent, syncs and verifies every boundary, then atomically publishes the whole tree. Its exact-digest retry and ambiguity rules are frozen in the lifecycle plan. It does not create a project operation package or project operations parent. Before any install, recover, rollback, quarantine, or restore ceremony, the portable bundle is verified and extracted descriptor-safely, then `build-operation-package.py` produces one closed source package only after loading the exact three-member operation-input context with its still-bound extraction root and independently retained expected operation-input-authority and finalization-identity digests. A loose artifact-identity file is never a package-builder authority. The source package and transition authority carry the exact canonical operation-input authority so the trust bridge survives staging. The root-installed exact-digest stager freezes these endpoints: normal `stage` and `execute`; host-transition `stage-recovery` and `execute-recovery`; non-mutating `inspect`, `inspect-execution`, and `inspect-host-transition`; and authority-bound operation-package `recover`. Normal staging accepts only `--contract`, `--source-package`, and `--expected-authority-sha256`; normal execution accepts only `--contract` and `--expected-authority-sha256`. Host-transition-recovery staging has the same three inputs as normal staging, host-transition-recovery execution has the same two inputs as normal execution, and the exact blocked authority is read from the canonical recovery request inside the transition authority rather than supplied by the caller. Inspection and privileged execution accept only the exact report/evidence inputs frozen by the lifecycle plan; no privileged endpoint accepts a target path, wrapper, or free-form action. The nonprivileged authority builder separately selects one closed reviewed recovery literal and performs no mutation. On the first accepted `prepare_filesystem` package only, `stage` creates `operations/.<project-slug>.initializing`, its fixed private `evidence/` child, and the complete verified `current` package inside that one hidden tree, syncs it, then atomically renames the whole initializer to the absent root-owned `operations/<project-slug>/`. A crash leaves either a bounded initializer or a complete final parent/current package, never an empty final parent. Before that outer rename, no durable project evidence namespace is claimed. Only an identical retry of `stage`, bound to the same retained source package, proposal envelope, project slug, and authority, may descriptor-remove/rebuild one verified incomplete initializer or finish the outer rename of one complete exact initializer. An unknown/mismatched initializer or simultaneous hidden/final parent is preserved as ambiguity. Every later operation requires those parents exact and pre-existing.

For subsequent operations the stager creates `.current.staging` descriptor-relatively as `root:wheel` mode `0700`, copies only already-retained verified bytes, writes and verifies the internal authority anchor, and atomically renames the complete directory to `current`. The separate `execute` endpoint loads only that derived verified `current` package, writes durable execution/outcome journals, invokes the selected wrapper, and retires the whole package. Recovery inspection/receipts are immutable content-addressed single-level directories below `evidence/`; each record is itself committed by a hidden-directory transaction, so an interrupted or old report cannot block the next state report. Descriptor-bound checks before and after every creation require the entire ancestor, parent, and file chain to have exact owner/group/type/mode, no allow ACL, no xattr/resource fork, no set-ID bit, and no prohibited or unknown flag. The selected no-argument wrapper must be regular/single-link, exact byte count/SHA-256, `root:wheel`, and mode `0555`; the staged source package holds canonical root-owned authority bytes, the host verification envelope, its bound inventory/policy inputs, and the exact manifest-verified payload. Execution journals and the quarantine-only `QUARANTINE_MANIFEST.json` are absent from staged source, excluded from source authority, and may be created only by the root executor after complete package verification. Portable helper bytes contain no host evidence and no self-referential final-manifest hash. Only the verified staged wrapper and standalone transition runtime are executed.

For a controller install, upgrade, rollback, or explicit recovery, that runtime validates the complete ancestor chain and exact old/new generation identities, preserves the previous active-manifest bytes, stages and recursively syncs a complete immutable generation, and changes activation only through the single `ACTIVE.json` replacement and parent `fsync` from section 6.4. Initial installation is one closed `install_controller` transaction and the sole owner of first `ACTIVE.json` creation: a complete pre-mutation journal binds the expected controller inventory and exact authentic old plus reviewed new `.env`/`.path` bytes/modes/flags; the complete controller is committed by one staging-directory rename; then activation files advance through replacement, ownership/mode, immutable-flag, and parent-sync phases. The authoritative journal record is never rewritten in place: every phase is a complete canonical same-parent atomic replacement through one exact derived temporary, so an interruption leaves a valid prior/new record plus at most one bounded temporary. Its report-bound recovery may only restore the exact pre-state or complete the exact post-state. Later generation changes use their separate live-controller journal with the same complete-record atomic replacement rule. The fixed `rollback-instance.sh` wrapper merely dispatches the authority-bound `rollback_generation` request—it does not contain or select a previous generation itself. Registration files, credentials, runner binaries, and other runner roots remain outside the mutation set. No operation starts the service, and outputs are bounded hashes/status only.

Success same-mode atomically retires the whole mode-`0700` `current` operation directory to a digest-named mode-`0700` completed record. Failure is preserved as an exact quarantine/current recovery state, and another normal operation cannot start while ambiguity or unresolved quarantine remains. The one exception is a non-recursive host-transition-recovery slot bound to the exact blocked authority: `.recovery.staging` → `recovery-current` → `recovery-completed-*` or `recovery-quarantine-*`. It accepts only explicit preparation, generation, or install recovery requests for that blocked package. A failed recovery cannot create another slot; after independent host restoration, fresh stable evidence may archive the recovery quarantine and then the original quarantine to closed failed records. Completed/failed records are durable and immutable by the version-1 interface, not filesystem-read-only. Version 1 intentionally provides no completed/failed evidence-deletion endpoint, so operators must monitor storage. This external location supports initial preparation and later forensic restore even when the instance path is absent. A source race can cause staging/type/hash failure, never intentional execution of replacement bytes. Adapter/barrier tests model this boundary; actual privileged race, ACL, flag, xattr, and exact-executed-digest proof is a separate authorized macOS qualification, not a unit-test claim.

### 9.3 Removal

Removal is deliberately staged. While the verified runner tree still exists, the owner first stops and uninstalls only its exact official user LaunchAgent through the official service lifecycle, proves the exact service/plist identity absent, and records a bounded preserve-or-exact-remove disposition for that service's log directory. The toolkit never edits a service or log path. The owner then uses GitHub's supported API/UI flow to deregister the exact authority-bound runner ID; the toolkit never accepts or handles the removal token. Fresh absence evidence must pass `verify_decommission_ready()` while the complete stopped instance remains present. A `quarantine_instance` operation source bound to that envelope is staged and executed only by the installed stager, building the strict bounded canonical `QuarantineManifest`, persisting/rereading exact runtime `QUARANTINE_MANIFEST.json` in the root-owned current operation package before mutation, and atomically renaming the complete verified instance into a same-parent digest-derived quarantine. The manifest records every non-followed member's relative path/type/ownership/mode/device/inode/link/regular-byte digest/symlink target/ACL/xattr/resource-fork/flag facts, rejects unsupported shapes and limits, and contains neither its own digest nor the derived quarantine name. Fresh evidence must then pass `verify_quarantined_binding()` by deriving the originating completed operation authority, descriptor-loading its retained canonical manifest, and recomputing it against the quarantine tree before a separately authorized `restore_quarantine` package is staged. Restore is forensic/filesystem-only: it returns the exact tree to `decommission_ready` for audit or re-quarantine and cannot be registered, serviced, or called operational. Version 1 exposes no destructive quarantine-purge endpoint. Operational re-adoption requires either a newly reviewed unused project slug/operations namespace while the old slug remains permanently retired, or a separately reviewed external/later-version disposal contract that removes the exact quarantine and old operations/evidence namespace; only then may fresh absence proofs and a wholly new instance/registration/controller ceremony begin. No phase removes by a shared label, organization scope, path glob, or broad service operation, and no partial controller-only backup is called reconstructable.

For the first re-adoption option, "wholly new" means a new reviewed contract with a fresh routing label, runner-name prefix, instance/runner/work roots, service identity, and operations namespace as well as the fresh project slug; every old identity remains retired. The dedicated account and project-specific runner group may be reused only after fresh evidence proves the old service, registration, Listener, and Worker absent and authenticated read-back proves the selected-workflow policy still exact.

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

Before writing or extracting, the verifier accepts regular-file members only and rejects duplicate names, symlinks, hard-link or special-file metadata, absolute paths, empty components, `.`/`..`, backslashes, control characters, non-ASCII names, macOS case-fold collisions, Unicode-normalization aliases, unsupported flags, inconsistent local/central headers, ZIP64, and configured count/size expansion limits. Construction is pure and returns a distinct non-finalizable `ConstructedArchive`; an independent parser consumes those immutable bytes plus explicit expected manifest/archive digests and alone returns `VerifiedArchive`. For a persisted artifact, the path adapter opens the archive once, retains the exact bounded bytes, and applies the same parser. Extraction accepts only the verified type and reparses its retained bytes, so replacing a source path cannot affect extraction. It uses descriptor-relative creation in a new fail-if-exists directory rather than `extractall`.

The builder:

- applies the canonical ZIP profile above;
- excludes ownership-dependent metadata, extended attributes, credentials, logs, workspaces, caches, and host evidence;
- writes canonical UTF-8 JSON with sorted keys and a final newline;
- records every payload member path, mode, byte count, and SHA-256 in `MANIFEST.json`; the manifest explicitly excludes itself and `SHA256SUMS` to avoid a circular self-hash;
- creates `SHA256SUMS` containing every payload member plus `MANIFEST.json`, while excluding only `SHA256SUMS` itself;
- independently verifies the constructed immutable bytes against separately derived expected manifest and archive digests before finalization; then extracts only the verified type to a fresh directory, compares every payload member to the manifest, validates `MANIFEST.json` through `SHA256SUMS`, and reopens the final artifact to validate its full bytes through the adjacent archive-digest record;
- scans immutable collected content for secret/token/private-path values before ZIP construction and separately scans member path components for forbidden live-runner state names. Source, tests, and documentation receive no broad exemption, but path-only state words may be documented as content without making a toolkit-source bundle impossible;
- creates no ZIP or digest on that pre-write content finding. A defensive post-extraction content incident triggers exact private cleanup; if cleanup cannot be proven, the bytes remain only in a private security-incident quarantine and are never represented as a distributable artifact. The report states this honestly rather than claiming impossible zero retention after a filesystem failure.

The durable local output uses pre-created private aggregation/class parents and two distinct fail-if-exists children per build: one artifact directory and one atomic finalization-evidence directory.

```text
<project-root>/dist/
├── artifacts/
│   ├── toolkit/
│   └── instances/
│       └── <project-slug>/
└── evidence/
    ├── toolkit/
    └── instances/
        └── <project-slug>/
```

Before every build, the nonprivileged caller creates `dist/`, `dist/artifacts/`, and `dist/evidence/`; for an instance build it additionally creates only the two `instances/` parents. All are real, non-symlink directories owned by the current account, exact mode `0700`, and opened descriptor-relatively. The caller never pre-creates the `toolkit/` or `<project-slug>/` output/evidence children; those four names are fail-if-exists transaction children. Directory link counts are filesystem-dependent and are not frozen; regular evidence/artifact files must remain single-link. The builder never creates or repairs aggregation parents. Each invocation owns exactly one absent artifact child and one absent evidence child under the matching parents and never writes into a child produced by another invocation. The evidence child is committed by a same-parent hidden-directory transaction and contains exactly mode-`0444` `artifact-policy.json` and `finalization-identity.json`; an interrupted hidden evidence or artifact staging child is handled only by the closed artifact recovery interface. Each archive filename contains the toolkit version, artifact class or instance slug, and manifest digest prefix. The manifest records the exact artifact-policy and independently verified source-binding digests. A mode-`0444` adjacent archive-digest record binds the complete mode-`0444` ZIP, including both control files; finalization rereads and reparses that exact record and revalidates both file identities before success. A SHA-256 binds bytes to the recorded acquisition path; it is not a signature.

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
- generic self-hosted routing, missing unique label, any PR/manual/scheduled/reusable event in the persistent lane, write permissions, secrets, comments, deployments, unpinned Actions, credential-persisting checkout, any workflow/job concurrency key that can replace a pending run, parallel matrix execution on one runner, uncontrolled caches, and artifact upload;
- mixed or partially staged controller generations, invalid active manifests, interrupted-transition journals, and activation of an unsynced generation;
- bundle members absent from or extra to the manifest, wrong mode, wrong size, wrong hash, non-deterministic order, noncanonical ZIP fields, duplicate/unsafe member names, special members, header disagreement, or non-normalized timestamp;
- secret-like values and forbidden live-runner files in the final ZIP.

### 12.2 Positive tests

The implementation must prove:

- valid dedicated and shared-account contracts render deterministically;
- a runner account can traverse its exact protected instance path but another local account cannot list or read any `0700` mutable child, while the runner account cannot traverse/list/read/write the maintainer home or an unrelated runner root;
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

- exact private organization repository, dedicated group/selected-repository/selected-workflow scope, authenticated default branch, active protected-branch/ruleset and hosted-check identities, organization runner ID, label, account, roots, and organization-scope service identity;
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
5. Provide the owner with an exact adoption manifest recording both ZIP filenames/byte counts/SHA-256 values and manifest identities; artifact-policy, source-binding, and independently retained finalization-identity plus operation-input-authority digests; the render-receipt byte digest and distinct rendered-payload binding; the standalone operation-stager filename/byte count/SHA-256 plus builder source commit/tree and isolated test result; the canonical global-bootstrap plan and owner-command transcript filenames/digests; the registration-process policy/monitor/receipt schema identities; every operation-request/conditional-plan/context filename and digest; dependency-lock hashes and platform/test matrix; exact generated child paths/basenames; limitations; and the adoption-prompt filename/SHA-256. A digest binds reviewed bytes but is not a signature or proof of publisher identity.
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
- [GitHub Actions Runner v2.336.0 work-option parsing](https://github.com/actions/runner/blob/v2.336.0/src/Runner.Listener/CommandSettings.cs)
- [GitHub Actions Runner v2.336.0 root-relative work-path resolution](https://github.com/actions/runner/blob/v2.336.0/src/Runner.Common/HostContext.cs)
- [GitHub Actions Runner v2.336.0 configuration writes](https://github.com/actions/runner/blob/v2.336.0/src/Runner.Listener/Configuration/ConfigurationManager.cs)
- [GitHub Actions Runner v2.336.0 configuration storage](https://github.com/actions/runner/blob/v2.336.0/src/Runner.Common/ConfigurationStore.cs)
- [GitHub Actions Runner v2.336.0 macOS service template](https://github.com/actions/runner/blob/v2.336.0/src/Misc/layoutbin/darwin.svc.sh.template)
- [GitHub Actions Runner v2.336.0 runtime configuration refresh](https://github.com/actions/runner/blob/v2.336.0/src/Runner.Listener/RunnerConfigUpdater.cs)

## 16. Deferred Decisions

The public repository name, owner, MIT license, initial contribution guidance, and private vulnerability-reporting route are now established. These remaining decisions are intentionally deferred and do not block local implementation after spec approval:

- formal maintainer and release governance;
- signed releases or attestations;
- public package distribution;
- optional VM/JIT providers;
- zero-login boot through a LaunchDaemon;
- integration with Linear, PMC, or another external tracker.

No deferred item may be silently claimed as implemented or supported.
