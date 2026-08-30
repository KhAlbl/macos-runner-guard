# macOS Runner Guard Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build and verify a project-neutral toolkit that renders project-dedicated macOS runner guards for private organization repositories with bounded automatic cleanup, exact runner-group workflow authorization, protected controller generations, deterministic evidence bundles, and explicit trust limits.

**Architecture:** Implement four independently reviewable subsystems in order. Pure contract/render/policy logic comes first; the cleanup runtime consumes that contract; the protected controller lifecycle activates immutable cleanup generations; deterministic archives and adoption documentation package only reviewed, non-secret bytes. Live runner adoption remains a separate project-specific ceremony.

**Tech Stack:** Python 3.9+ standard library for installed hooks/controller; Python 3.9–3.14 for the toolkit; PyYAML 6.0.3 only for host-independent workflow parsing; POSIX/macOS descriptor-relative filesystem APIs; `/bin/bash`; JSON Schema 2020-12; pytest and unittest; GitHub Actions on hosted Ubuntu and macOS.

**Spec:** [Approved architecture](../../ARCHITECTURE.md)

## Global Constraints

- Keep the implementation repository-neutral: no private project name, private username, live runner ID, registration material, or private host path may enter source, examples, fixtures, archives, or logs.
- Do not create, register, start, stop, relabel, remove, or modify a live runner while executing these plans.
- Do not create macOS accounts, root services, GitHub Apps, JIT controllers, runner groups, LaunchDaemons, tokens, sudoers rules, signing identities, releases, tags, packages, or attestations.
- Installed job hooks and the cleanup/activation controller must run with `/usr/bin/python3 -I -S` and import only the Python standard library.
- Cleanup must use descriptor-relative, no-follow operations and fail closed when the capability probe cannot prove the required primitives. There is no path-recursive fallback.
- Every pre-mutation filesystem rejection test snapshots its fixture and proves all non-target bytes and metadata remain unchanged. A failure after mutation begins is reported as bounded partial quarantine and never claims whole-target immutability.
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
| Action identity | exact complete member of contract `allowed_actions`; shape alone is insufficient |
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
| `contract.py` | bounded no-follow `read_regular_file_bytes(path, maximum_bytes) -> bytes`; `load_contract(path) -> InstanceContract`; `load_contract_bytes(data) -> InstanceContract`; `validate_contract_set(contracts) -> None`; `canonical_contract_bytes(contract) -> bytes`; `validate_run_identity_fields(run_id, run_attempt)`; `validate_job_identity_fields(contract, run_id, run_attempt, github_job, leg)` |
| `bindings.py` | `BindingRecord(path, mode, byte_count, sha256)`; `build_binding_manifest(schema_version, records) -> bytes`; `binding_manifest_sha256(...) -> str`; accepted schemas are the generation, installation-input, and rendered-payload v1 identities only |
| `policy.py` | `load_workflow_bytes(data) -> Mapping[str, object]`; `check_workflow(document, contract) -> tuple[PolicyViolation, ...]` |
| `render.py` | `load_component_payloads(root) -> tuple[ComponentPayload, ...]`; `load_template_payloads(root) -> tuple[TemplatePayload, ...]`; `render_instance(contract, immutable_components, immutable_templates, policy_source) -> tuple[RenderedFile, ...]`; `merge_component_payloads(base, replacements) -> tuple[ComponentPayload, ...]`; `build_rendered_payload_binding(files) -> bytes`; `rendered_payload_binding_sha256(files) -> str`; `canonical_render_receipt_bytes(files) -> bytes`; `load_render_receipt_bytes(data) -> RenderReceipt`; `write_rendered_tree(output_dir, files) -> None`; the receipt-byte SHA-256 is review evidence while its distinct `rendered_payload_binding_sha256` field is the exact value accepted by the instance builder |
| `components.py` | `build_all_components(source_root: Path) -> tuple[ComponentPayload, ...]`; `write_component_tree(output_dir: Path, components: Sequence[ComponentPayload]) -> None`; descriptor-loads the fixed reviewed cleanup/runtime/lifecycle source inventory from one explicit source root, calls the two child builders, merges through `render.merge_component_payloads()`, and writes exactly eleven mode-`0555` files to one absent output transaction |
| `cleanup.py` / `bundle_runtime.py` | `CleanupPhase` owns `before`, `after`, `preflight`, and `workflow`; `run_cleanup(phase, contract, environment) -> CleanupReceipt`; `build_cleanup_components(sources, modules) -> tuple[ComponentPayload, ...]` returns exactly the five frozen cleanup components; `build_transition_runtime_bundle(modules) -> bytes` and `build_operation_stager_bundle(modules) -> bytes` each accept the exact same closed eleven lifecycle modules, with the latter producing the isolated root-stager program and no ambient import |
| `generation.py` | `open_generation_source(source_root, expected_manifest_sha256) -> OpenedGenerationSource`; `stage_generation(source, controller_root, expected_manifest_sha256, generation_transition_id_sha256) -> GenerationBinding`; `verify_generation(root, expected_sha256) -> GenerationBinding`; the transition ID deterministically binds the sole recoverable staging basename |
| `bootstrap.py` | `canonical_bash_argv_literal(value) -> bytes`; `build_global_bootstrap_plan(stager_source, expected_byte_count, expected_sha256) -> GlobalBootstrapPlan`; `canonical_global_bootstrap_plan_bytes(plan) -> bytes`; `load_global_bootstrap_plan_bytes(data) -> GlobalBootstrapPlan`; `inspect_global_bootstrap_state(plan, adapter) -> GlobalBootstrapStateReport`; `render_owner_bootstrap_command(plan_path, expected_plan_sha256) -> bytes`; the shared canonical single-quoted Bash argv encoder rejects NUL and protects every variable-bearing word in both bootstrap and later registration transcripts; the operator-private plan owns the fixed global initializer and exact retained stager-source binding, while an interrupted context build is abandon-only and owner commands name only final context paths |
| `activation.py` / `transition.py` | `read_active_binding(controller_root) -> GenerationBinding`; `open_active_generation(controller_root) -> ContextManager[OpenedGeneration]`; `execute_opened_generation(opened, phase, environment) -> CleanupReceipt`; `build_filesystem_preparation_plan(contract, accepted_envelope) -> FilesystemPreparationPlan`; `canonical_filesystem_preparation_plan_bytes(plan) -> bytes`; `load_filesystem_preparation_plan_bytes(data) -> FilesystemPreparationPlan`; `filesystem_preparation_plan_sha256(plan) -> str`; `write_filesystem_preparation_plan(output, plan) -> None`; `prepare_filesystem(contract, inventory, accepted_envelope, preparation_plan) -> FilesystemPreparationReceipt`; `install_initial_controller(instance_root, runner_root, controller_source, old_activation, new_activation) -> None`; `upgrade_generation(controller_root, source, expected_manifest_sha256) -> GenerationBinding`; `rollback_generation(controller_root, previous) -> None`; `recover_transition(instance_root, runner_root, preparation_plan, action) -> RecoveryReport`; `build_quarantine_manifest(contract, expected_runner_id, inventory) -> QuarantineManifest`; `canonical_quarantine_manifest_bytes(manifest) -> bytes`; `load_quarantine_manifest_bytes(data) -> QuarantineManifest`; `quarantine_manifest_sha256(manifest) -> str`; `quarantine_local_instance(contract, expected_runner_id, inventory, manifest, operation_authority_sha256) -> RemovalReport`; `restore_quarantined_instance(contract, report, inventory, manifest, operation_authority_sha256) -> RemovalReport`; `build_lifecycle_components(sources: Sequence[ComponentPayload], modules: Sequence[RuntimeModule]) -> tuple[ComponentPayload, ...]` returns exactly the six frozen rendered lifecycle components; the canonical preparation plan is authority-bound and excludes its digest-derived staging basename, quarantine/restore are closed reversible v1 operations with no purge endpoint, preparation and initial `ACTIVE.json` creation have one transition owner each, upgrade owns its complete journaled switch, and install/runtime-result journals are host-side state rather than rendered or staged-source members |
| `inventory.py` | `load_redacted_inventory(path) -> HostInventory`; `canonical_runner_application_lock_bytes(lock) -> bytes`; `load_runner_application_lock_bytes(data) -> RunnerApplicationLock`; `canonical_runner_installation_evidence_bytes(evidence) -> bytes`; `load_runner_installation_evidence_bytes(data) -> RunnerInstallationEvidence`; `collect_runner_application_integrity(application_lock, installation_evidence, state_root_binding, adapter) -> RunnerApplicationIntegrityEvidence`; `build_host_verification_envelope(report, inventory_bytes, branch_policy_bytes, session_policy_bytes, peer_boundary_policy_sha256, state_descriptor_bytes: Sequence[tuple[str, bytes]]) -> bytes`; `load_host_verification_envelope(data, expected_state) -> HostVerificationEnvelope`; the nine exact host-state verifiers: proposal precreation, proposal ready, filesystem prepared, registered pretransition, installed precommissioning, commissioned, decommission ready, quarantined, and recovery; child plans obtain an accepted envelope only through the canonical builder/loader and explicit expected-state validation, every accepted report is wrapped in that bound envelope, and no state accepts caller-authored installation evidence or a stale/current-match assertion; applicable inventories always construct fresh descriptor-relative current-application evidence from the lifecycle-neutral lock |
| `registration.py` | `build_registration_ceremony(contract, accepted_envelope, filesystem_prepared_inventory, organization_routing_evidence, acquisition_receipts) -> RegistrationCeremony`; `canonical_registration_ceremony_bytes(ceremony) -> bytes`; `load_registration_ceremony_bytes(data) -> RegistrationCeremony`; `registration_ceremony_sha256(ceremony) -> str`; `build_registration_command_plan(expected, extraction_report, session_policy, peer_idle_bindings, ceremony) -> RegistrationCommandPlan`; `render_owner_registration_commands(plan) -> bytes`; `build_registration_process_policy(plan, transcript_bytes) -> RegistrationProcessPolicy`; `canonical_registration_process_policy_bytes(policy) -> bytes`; `load_registration_process_policy_bytes(data) -> RegistrationProcessPolicy`; `registration_process_policy_sha256(policy) -> str`; `observe_registration_processes(policy, owner_shell_pid, adapter) -> RegistrationObservationReceipt`; `canonical_registration_observation_receipt_bytes(receipt) -> bytes`; `load_registration_observation_receipt_bytes(data) -> RegistrationObservationReceipt`; `compare_registered_runner_members(expected, extraction_report, policy, adapter) -> RunnerMemberComparisonReceipt`; `canonical_runner_member_comparison_receipt_bytes(receipt) -> bytes`; `load_runner_member_comparison_receipt_bytes(data) -> RunnerMemberComparisonReceipt`; `runner_member_comparison_receipt_sha256(receipt) -> str`; `build_runner_application_lock(expected, extraction_report) -> RunnerApplicationLock`; `build_runner_installation_evidence(expected, extraction_report, application_lock, comparison_receipt, ceremony, policy, accepted_receipt, organization_routing_evidence, acquisition_receipts) -> RunnerInstallationEvidence`; the canonical ceremony derives every non-secret registration mutation input, including fixed `automatic_updates_disabled=true`, from the exact contract, accepted filesystem-prepared envelope/inventory, and authenticated empty-group evidence before command rendering; the in-memory command plan, shared canonical shell-literal encoder, exact transcript containing exactly one literal `--disableupdate`, final policy, at-rest archive/member comparison, lifecycle-neutral application lock, content-free process receipt, fresh post-configuration comparison, and authenticated receipt-bound organization read-back are distinct inputs and all must cross-validate; the observer never receives a token or claims generic shell metadata proves script bytes |
| `operation_package.py` | `build_operation_request(contract, accepted_envelope, operation, typed_inputs) -> OperationRequest`; `canonical_operation_request_bytes(request) -> bytes`; `operation_request_sha256(request) -> str`; `load_operation_request_bytes(data) -> OperationRequest`; `write_operation_request(output, request) -> None`; `canonical_artifact_identity_bytes(identity) -> bytes`; `load_artifact_identity_bytes(data) -> ArtifactIdentity`; `build_operation_input_authority(expected_finalization_identity_sha256, extraction_report_bytes, artifact_identity_bytes) -> OperationInputAuthority`; `canonical_operation_input_authority_bytes(value) -> bytes`; `operation_input_authority_sha256(value) -> str`; `load_operation_input_authority_bytes(data) -> OperationInputAuthority`; `load_verified_operation_input_context(context, payload, expected_operation_input_authority_sha256, expected_finalization_identity_sha256) -> VerifiedOperationInput`; `build_operation_source(contract, request, verified_operation_input, inventory_bytes, branch_policy_bytes, verification_envelope_bytes, preparation_plan_bytes=None) -> VerifiedOperationSource`; `write_operation_source(output, source) -> None`; `load_verified_operation_source(path, expected_authority_sha256) -> VerifiedOperationSource`; `stage_operation_package(contract, source_path, expected_authority_sha256) -> OperationPackageBinding`; `stage_host_recovery_package(contract, source_path, expected_authority_sha256) -> OperationPackageBinding`; `verify_operation_package(contract, slot) -> OperationPackageBinding`; `execute_operation_package(contract, slot) -> bounded receipt`; `retire_operation_package(contract, binding, outcome) -> OperationPackageBinding`; `inspect_operation_package_state(contract, expected_authority_sha256) -> OperationPackageRecoveryReport`; `inspect_operation_execution_resolution(contract, binding, fresh_inventory_bytes) -> OperationExecutionResolution`; `build_operation_recovery_authority(contract, report, action, execution_resolution, transition_recovery_report, stable_host_envelope) -> bytes`; `load_operation_recovery_authority_bytes(data) -> OperationRecoveryAuthority`; `recover_operation_package(contract, report, expected_report_sha256, recovery_authority_bytes, expected_recovery_authority_sha256) -> OperationPackageRecoveryReceipt`. `scripts/build-operation-request.py` atomically emits the canonical request context and its conditional plan; `scripts/build-operation-package.py` consumes those exact members plus the complete authority-bound operation-input context and both independently retained expected digests. The conditional canonical `filesystem-preparation-plan.json` source/destination member is required exactly for preparation-domain requests and is inventoried by the authority. The installed CLI freezes normal `stage`/`execute`, host-transition `stage-recovery`/`execute-recovery`, non-mutating `inspect`/`inspect-execution`/`inspect-host-transition`, and authority-bound `recover`; no privileged execution caller supplies a target, wrapper, free action, or blocked operation authority. |
| `manifest.py` | `VerifiedPayload`, `PayloadMember`, `ManifestRecord`, `AllowedMember`, and `ArtifactPolicy`; complete policy/source-bound collection and control-file construction |
| `archive.py` | `build_canonical_zip(payload, policy, expected_source_binding_sha256) -> ConstructedArchive`; `verify_canonical_zip_bytes(archive_bytes, policy, expected_source_binding_sha256, expected_archive_sha256, expected_manifest_sha256) -> VerifiedArchive`; `verify_canonical_zip(archive_path, policy, expected_source_binding_sha256, expected_archive_sha256, expected_manifest_sha256) -> VerifiedArchive`; `load_finalized_verified_archive(output_child, evidence_output, expected_source_binding_sha256, expected_finalization_identity_sha256) -> tuple[VerifiedArchive, FinalArtifactReceipt]`; `extract_verified_zip(verified, destination, policy, expected_source_binding_sha256) -> ExtractionReport`; `canonical_extraction_report_bytes(report) -> bytes`; `load_extraction_report_bytes(data) -> ExtractionReport`; `inspect_interrupted_extraction(destination, verified, policy, expected_source_binding_sha256) -> ExtractionRecoveryReport`; `recover_interrupted_extraction(destination, report, expected_report_sha256, verified, policy, expected_source_binding_sha256, action) -> ExtractionRecoveryReport`; `build_lifecycle_artifact_identity(verified, extraction) -> bytes`; `build_finalization_identity(verified, output_child, policy, expected_source_binding_sha256) -> bytes`; `load_finalization_identity_bytes(data) -> FinalizationIdentity`; `write_finalization_evidence(evidence_output, policy, identity) -> None`; `load_finalization_evidence(evidence_output, expected_source_binding_sha256, expected_finalization_identity_sha256) -> tuple[ArtifactPolicy, FinalizationIdentity]`; `finalize_verified_archive(verified, output_child, policy, expected_source_binding_sha256, identity) -> FinalArtifactReceipt`; `verify_finalized_artifact(output_child, policy, expected_source_binding_sha256, identity) -> FinalArtifactReceipt`; `inspect_artifact_staging(output_child, evidence_output, artifact_class, expected_source_binding_sha256) -> ArtifactStagingRecoveryReport`; `recover_artifact_staging(output_child, evidence_output, artifact_class, expected_source_binding_sha256, report, expected_report_sha256, action) -> ArtifactStagingRecoveryReport`; `inspect_operation_input_context(verified, extraction_output, context_output, expected_finalization_identity_sha256) -> OperationInputRecoveryReport`; `recover_operation_input_context(verified, extraction_output, context_output, expected_finalization_identity_sha256, report, expected_report_sha256, action) -> OperationInputRecoveryReport`. The pure builder type is never accepted by extraction/finalization; only the independent byte/path verifier creates `VerifiedArchive`. |
| `upstream_runner.py` | `canonical_expected_runner_archive_bytes(expected) -> bytes`; `expected_runner_archive_sha256(expected) -> str`; `load_expected_runner_archive(path) -> ExpectedRunnerArchive`; `write_expected_runner_archive(output, expected) -> None`; `verify_official_runner_archive(path, expected) -> VerifiedRunnerArchive`; `extract_verified_runner_archive(verified, expected, destination) -> RunnerExtractionReport`; `canonical_runner_extraction_report_bytes(report) -> bytes`; `load_runner_extraction_report_bytes(data) -> RunnerExtractionReport`; `write_runner_extraction_report(output, report) -> None`; `canonical_runner_extraction_recovery_report_bytes(report) -> bytes`; `load_runner_extraction_recovery_report_bytes(data) -> RunnerExtractionRecoveryReport`; `write_runner_extraction_recovery_report(output, report) -> None`; `inspect_interrupted_runner_extraction(verified, expected, destination) -> RunnerExtractionRecoveryReport`; `recover_interrupted_runner_extraction(report, expected_report_sha256, verified, expected, destination, action, success_report_output=None) -> RunnerExtractionRecoveryReport`; the proposal CLI is offline and untrusted until independent review, a complete destination can resume only the absent durable success report, while an exact proper subset can reset only to the same empty root; all canonical writers and paths independently parse or cross-check the same retained reviewed identities |

Types referenced across plans live in the module that owns the behavior; child plans must import rather than redefine them. `contract.py` owns `InstanceContract`, `InstanceLayout`, and `CleanupLimits`; `policy.py` owns `PolicyViolation`; `render.py` owns `ComponentPayload`, `TemplatePayload`, `RenderedFile`, and `RenderReceipt` plus its canonical loader/serializer; `cleanup.py` owns `CleanupPhase` and cleanup receipts; `generation.py` owns generation bindings; `bootstrap.py` owns `GlobalBootstrapPlan` and `GlobalBootstrapStateReport`; `inventory.py` owns host evidence/reports/envelopes including the lifecycle-neutral `RunnerApplicationMemberLock`, versioned closed runner live-state namespace policy records, `RunnerApplicationLock`, `RunnerApplicationIntegrityEvidence`, and `RunnerInstallationEvidence` schemas/loaders plus the fresh complete current-application collector; `registration.py` owns `RegistrationCeremony`, in-memory-only `RegistrationCommandPlan`, registration policy/observation records, the child-plan-4 reviewed-archive-to-application-lock adapter, and the only installation-evidence constructor; `transition.py` owns preparation/generation/install transition plans, bindings, receipts, recovery reports, `QuarantineManifest`, member records, and tagged `RemovalReport`; `operation_package.py` owns operation requests, the neutral artifact-identity and operation-input-authority interchange schemas, the nonserializable `VerifiedOperationInput`, authorities, and operation-package records; `manifest.py` owns payload, manifest, allowed-member, and artifact-policy records; `archive.py` owns `ConstructedArchive`, `VerifiedArchive`, `ExtractionReport`, `ExtractionRecoveryReport`, `FinalizationIdentity`, `ArtifactStagingRecoveryReport`, `OperationInputRecoveryReport`, and `FinalArtifactReceipt`, including the child-plan-4 executable final-artifact-to-operation-input context bridge that imports the already accepted neutral schemas; `upstream_runner.py` owns reviewed upstream-runner archive/extraction/recovery records.

The renderer returns exactly these 16 payload members and modes. The eleven component inputs are executable `0555` files; the remaining five members are derived from explicit retained contract, template, and policy-source bytes. All input directories/files are descriptor-bound before pure rendering. Artifact staging, not rendering, later adds `MANIFEST.json` and `SHA256SUMS`.

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

Read ruleset ID `21842288` through the GitHub API. Require active enforcement on `refs/heads/main`, no bypass actors, and exact `pull_request`, `deletion`, and `non_fast_forward` rules. Record the live ruleset response in the implementation issue without copying account credentials or local host evidence. Stop if the ruleset is absent or weaker. This minimum authorizes only the owner-coordinated implementation lane; do not solicit or accept external implementation contributions until Task 7 strengthens the ruleset after stable CI check names exist.

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
ARTIFACT_SUBSYSTEM_ACCEPTED
```

Evidence includes two closed-fixture archive hashes, manifest coverage, adversarial ZIP rejection totals, secret-scan result, and documentation review. These are subsystem/non-release identities only; the final toolkit-source and instance artifacts are built after Task 6 commits the umbrella integration files.

## Task 6: Run the integrated verification gate

**Files:**
- Create: `src/runner_guard/components.py`
- Create: `scripts/build-components.py`
- Create: `scripts/validate.py`
- Create: `tests/test_end_to_end.py`
- Create: `tests/test_components.py`
- Create: `tests/test_public_inventory.py`
- Modify: `src/runner_guard/git_source.py`
- Modify: `tests/test_bundle_determinism.py`

- [ ] **Step 1: Write a failing end-to-end test**

The test first invokes `scripts/build-components.py --source-root <explicit-reviewed-source-root> --output <absent-component-directory>` twice from source roots with identical reviewed bytes but different absolute paths, mtimes, enumeration order, and umasks; it requires two byte-identical exact eleven-file component trees, then feeds each through the public render CLI rather than a test-fixture sentinel. It renders two disjoint sample contracts, runs before/after cleanup only within their fixture roots, stages and activates two controller generations, builds closed fixture forms of both artifact classes twice, and proves project A cannot address project B. For the instance fixture it invokes the artifact subsystem's public `scripts/prepare-operation-input.py prepare` bridge against the real finalized archive/evidence directory, the independently computed expected finalization-identity SHA-256, fresh extraction root, and fresh atomic context directory; it does not manually serialize any of the three context records. It independently retains the emitted operation-input-authority SHA-256 and the exact extracted payload root whose device/inode those records bind. The test invokes the sole request builder for the first preparation operation and feeds both members of its atomic request context plus the complete operation-input context into the package builder:

```text
scripts/build-operation-request.py --operation prepare_filesystem --contract <contract> --verification-envelope <envelope> --output <new-request-context>
scripts/build-operation-package.py --contract <contract> --operation-request <new-request-context/operation-request.json> --operation-input-context <context> --expected-operation-input-authority-sha256 <independently-retained-digest> --expected-finalization-identity-sha256 <independently-retained-digest> --inventory <inventory> --protected-branch-policy <policy> --verification-envelope <envelope> --payload <that-exact-root> --filesystem-preparation-plan <new-request-context/filesystem-preparation-plan.json> --output <new-source-package>
```

The disposable stager model consumes that exact source package through normal `stage` then `execute`, exercises one later package plus one host-transition-recovery package, interrupts the complete first-project initializer immediately before its outer rename, and proves the matching retry finishes while a substituted request context, plan, source, or blocked authority fails. These are model-root tests only and make no `/Library`, root, service, or registered-runner claim. Model-only artifact fixtures, a copied root, a manually transcribed request/plan/identity/operation-input authority, identity plus payload substituted self-consistently while retaining the reviewed digests, or an invocation missing any mandatory operation input cannot satisfy this bridge. It does not claim to build the final clean-source toolkit artifact while its own integration files are still uncommitted.

Run:

```bash
PYTHONPATH="$PWD/src" .venv/bin/python -m unittest tests.test_end_to_end -v
```

Expected: FAIL because the public component assembler and integrated validation driver do not exist.

- [ ] **Step 2: Add the deterministic public component assembler and validation driver**

`runner_guard.components.build_all_components()` descriptor-opens the explicit source root and accepts exactly this closed source inventory, every entry a regular single-link mode-`0644` UTF-8 file ending in one LF:

- cleanup templates: `templates/activate.py`, `templates/job-started.sh`, `templates/job-completed.sh`;
- cleanup runtime modules: `src/runner_guard/capabilities.py`, `jsonio.py`, `contract.py`, `errors.py`, `job_identity.py`, `fsops.py`, `workspace.py`, `cleanup.py`, and `residue.py`;
- lifecycle wrapper templates: `templates/install-instance.sh`, `templates/recover-instance.sh`, `templates/rollback-instance.sh`, `templates/remove-instance.sh`, and `templates/restore-instance.sh`;
- lifecycle runtime modules: `src/runner_guard/errors.py`, `jsonio.py`, `contract.py`, `bindings.py`, `fsops.py`, `generation.py`, `activation_files.py`, `inventory.py`, `activation.py`, `transition.py`, and `operation_package.py`.

The abbreviated module names after the first path stay under `src/runner_guard/`. Shared modules are opened once and their retained bytes are supplied to both closed builders. The assembler does not inventory unrelated repository paths: an unrelated tracked file is outside this input namespace, while a missing, duplicated alias, symlinked, nonregular, remoded, non-UTF-8, non-final-LF, or changed-during-read required input is rejected. It revalidates every opened identity before returning, invokes only `build_cleanup_components(cleanup_sources, cleanup_modules)` and `build_lifecycle_components(lifecycle_sources, lifecycle_modules)`, and combines them only through `render.merge_component_payloads()`. `write_component_tree()` commits exactly the eleven expected mode-`0555` `ComponentPayload` records to an absent output directory through a same-parent hidden-directory transaction; it accepts no overwrite and discovers no current-directory/package path. `scripts/build-components.py` exposes only `--source-root` and `--output`, prints the component count plus a canonical component-tree digest, and performs no host or GitHub action. Tests use two different explicit source roots, output collisions, source swaps, every required-input mutation, an unrelated-file control, interrupted output transaction, and the exact eleven-file/mode contract. Final clean-build review—not this pure assembler—binds the source root to the exact reviewed Git commit/tree.

`scripts/validate.py` invokes module entry points directly, including this public assembly route, emits a bounded JSON summary without paths containing user data, and returns nonzero on any skipped required gate. It accepts only an output directory that does not yet exist.

- [ ] **Step 3: Add exact public inventory tests**

`tests/test_public_inventory.py` pins package modules, CLI entry points, schemas, templates, example contracts, documentation files, and the complete final artifact `AllowedMember` inventory. Update the artifact policy and determinism fixtures so `scripts/validate.py`, `tests/test_end_to_end.py`, and `tests/test_public_inventory.py` are ordinary required members rather than future placeholders. Reject an undocumented extra executable or missing planned component. It also scans every checked-in implementation-plan `bash` block and rejects any repository `.venv/bin/python` command not explicitly prefixed with `PYTHONPATH="$PWD/src"`; isolated installed/runtime `/usr/bin/python3 -I -S` commands are a separate closed execution boundary and are not rewritten to import the checkout.

- [ ] **Step 4: Run focused final-content validation before committing**

Run:

```bash
PYTHONPATH="$PWD/src" .venv/bin/python -m compileall src tests scripts -q
PYTHONPATH="$PWD/src" .venv/bin/python -m unittest discover -s tests -v
PYTHONPATH="$PWD/src" .venv/bin/python -m pytest -q
PYTHONPATH="$PWD/src" .venv/bin/python scripts/validate.py --output "$(mktemp -d)/validation"
git diff --check
```

Expected: every required non-clean-source gate passes; no test is silently skipped except an explicitly platform-gated macOS integration test on Linux. Do not invoke the real toolkit-source builder from this dirty worktree.

- [ ] **Step 5: Commit the integration gate**

```bash
git add src/runner_guard/components.py scripts/build-components.py scripts/validate.py \
  tests/test_components.py tests/test_end_to_end.py tests/test_public_inventory.py \
  src/runner_guard/git_source.py tests/test_bundle_determinism.py
git commit -m "test: add integrated runner guard verification"
```

- [ ] **Step 6: Build and verify the final artifacts from the exact clean commit**

Create two independent clean Git checkouts of the exact Task 6 commit/tree. For each checkout create a separate private validation root **outside** the checkout and place its virtual environment, `TMPDIR`, `XDG_CACHE_HOME`, `PIP_CACHE_DIR`, `PYTHONPYCACHEPREFIX`, validation reports, component output, rendered output, artifacts, extraction root, and operation-input context only below that external root. Install validation tools there from the matching per-interpreter hash lock, set explicit `PYTHONPATH=<checkout>/src`, and run pytest with `-p no:cacheprovider`; no editable/local package install is permitted. Before validation, record a descriptor-derived snapshot of every non-`.git` checkout entry including ignored/untracked paths, type, mode, byte count, and SHA-256. After validation require that snapshot byte-for-byte unchanged, `git status --porcelain=v1 --untracked-files=all` empty, the ignored-file inventory empty, and the exact commit/tree unchanged. A validation command that creates `.venv`, bytecode, pytest cache, build output, or any other entry inside the checkout fails the gate rather than being cleaned.

Run the complete validation driver under that external environment, invoke the public component assembler into a fresh external private directory, feed that exact directory to the public renderer for the same reviewed sample instance, and build both final artifact classes through `scripts/build-adoption-zip.py`. In each clean checkout also invoke `scripts/build-operation-stager.py --source-root <that-clean-checkout> --output <external-private-root>/stage-operation.py`; require byte-identical standalone stager bytes, record filename/byte count/SHA-256 plus the builder source commit/tree, and run each output under `/usr/bin/python3 -I -S` with source/site/current-directory imports unavailable. Require byte-identical component trees, rendered trees, toolkit ZIPs, instance ZIPs, manifests, adjacent digest records, and atomic finalization-evidence directory members (`artifact-policy.json` and `finalization-identity.json`) across the two roots. Independently verify every final output through those preserved evidence directories.

For each independently built stager, invoke `scripts/plan-global-bootstrap.py --stager-source <that-root>/stage-operation.py --expected-byte-count <measured> --expected-sha256 <verified> --output <fresh-private-root>/bootstrap-context`. Require the exact atomic context members `bootstrap-plan.json`, `bootstrap-state-report.json`, and `OWNER_COMMANDS.txt`; capture shell argv for adversarial safe paths and prove it is exactly `/usr/bin/python3 -I -S -c PROGRAM -- PLAN SIZE HASH`; prove a non-root execution of the rendered owner command exits before mutation; verify the transcript names only the final `<bootstrap-context>/bootstrap-plan.json` and never its hidden transaction sibling; and inject/retain one interrupted-context fixture to prove abandon-only behavior plus success in a different fresh output. Compare the portable toolkit version, fixed destinations/inventory, source byte count/SHA-256, inline-program SHA-256, canonical quoting, and command grammar across both contexts. The absolute stager source path and its parent/leaf device/inode values are intentionally host-local and must differ or be independently verified rather than falsely compared as reproducible. Record every plan/report/transcript byte count and SHA-256 plus the inline-program digest.

Then run `scripts/prepare-operation-input.py prepare` with the independently computed expected finalization-identity SHA-256 into a different fresh private extraction root and fresh atomic context directory in each checkout. Require each context to contain exactly `extraction-report.json`, `artifact-identity.json`, and `operation-input-authority.json`; independently retain the reported operation-input-authority SHA-256. Invoke `scripts/build-operation-request.py --operation prepare_filesystem --contract <contract> --verification-envelope <envelope> --output <new-request-context>` in each root; require its atomic context to contain exact canonical `operation-request.json` and `filesystem-preparation-plan.json` members and compare their bytes/digests. Feed each complete operation-input context plus its own exact bound root and both independently retained expected digests, `<new-request-context>/operation-request.json`, `<new-request-context>/filesystem-preparation-plan.json`, and the complete inventory, protected-branch-policy, verification-envelope, and absent output arguments to `build-operation-package.py`. Missing, manually serialized, or substituted request/plan/context bytes fail. Those two host-local artifact identities and operation-input-authority bytes are expected to differ in destination device/inode; compare every portable archive/member/installation-input/request/plan field for equality, verify each destination binding independently, and never claim the host-local authority/identity bytes are reproducible across roots. Retain each original expected pair while mutation-testing identity-only, payload-only/in-place, self-consistent identity-plus-payload, extraction-report, context-copy/root, context-mode/inventory, and finalization substitutions; all must fail before package creation. Record the exact source commit/tree and every stager/component/rendered/final/evidence/operation-input/request-context/operation-package filename, byte count, member count, standalone-stager SHA-256 and isolated-test result, request/plan SHA-256, artifact-policy SHA-256, finalization-identity SHA-256, operation-input-authority SHA-256, each host-local lifecycle-artifact-identity SHA-256, manifest SHA-256, and archive SHA-256.

If any validation or artifact comparison fails, do not edit a clean build root or reuse its outputs. Make the smallest correction in the implementation worktree, commit it, discard all prior final artifact identities, and repeat this entire step from two new clean checkouts of the new exact commit.

## Task 7: Complete independent review without publishing

- [ ] **Step 1: Run focused independent reviews**

Require separate review of:

1. descriptor-relative cleanup and archive extraction;
2. controller activation/recovery/rollback atomicity;
3. workflow and supply-chain policy;
4. public documentation and trust claims.

Every review is bound to the exact head/tree and the final artifact identities produced from that tree in Task 6 Step 6. Actionable findings require a new commit, full affected validation, fresh two-root artifact builds, and a fresh review of the new exact head and artifacts.

- [ ] **Step 2: Harden the contribution ruleset before external implementation**

Until this gate is complete, do not invite or accept external implementation contributions. Confirm the active `main` ruleset has no bypass actors and includes `pull_request`, `deletion`, and `non_fast_forward`. After the GitHub-hosted PR workflow establishes stable exact check names, update the ruleset separately to require those hosted checks, at least one approving review, dismissal of stale approvals, approval of the latest push by someone other than its pusher, and resolution of all review conversations. Re-read the live ruleset and attach redacted evidence before opening the contribution lane. This public toolkit repository remains hosted-only. For any later adopting repository, persistent execution is accepted only after the private-organization and dedicated-group evidence defined by the lifecycle plan proves selected-repository access, public-repository denial, and `restricted_to_workflows` with exactly `<owner>/<repo>/.github/workflows/macos-runner-guard.yml@refs/heads/<protected-default-branch>`; the path is a fixed version-1 destination, and a label or push-only event is not sufficient. Do not mix the governance API mutation into an implementation-fix commit.

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
