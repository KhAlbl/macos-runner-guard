# Deterministic Artifacts and Adoption Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Produce byte-for-byte reproducible toolkit-source and per-instance ZIPs, verify them independently before safe extraction, and publish precise adoption/operations guidance without exposing live host state or claiming a release.

**Architecture:** Payload bytes are collected into typed regular-file members. `MANIFEST.json` covers the payload and excludes the two control files; `SHA256SUMS` covers payload plus manifest and excludes itself. A custom canonical ZIP32/STORE writer sets every header field. A separate parser verifies local and central records before descriptor-relative extraction into a new directory. Documentation and agent prompts consume only verified bundles.

**Tech Stack:** Python 3.9+ standard library (`hashlib`, `struct`, `zlib`, descriptor-relative `os` APIs); canonical JSON; ZIP32 specification; unittest/pytest; Markdown; final-byte secret scanning.

**Spec:** [Approved architecture §§10–15](../../ARCHITECTURE.md)

## Global Constraints

- Prerequisites: the first three child plans are accepted on the same exact branch head.
- Artifact manifest version is `macos-runner-guard.manifest.v1`; toolkit version is `0.1.0`.
- Build only from a clean source checkout or a fresh rendered-instance staging tree—never from a live runner root.
- `MANIFEST.json` and `SHA256SUMS` exist only in fresh staging and archives; they are not committed at repository root.
- Archive limits are fixed at 256 total members including `MANIFEST.json` and `SHA256SUMS`, 8,388,608 bytes per member, 67,108,864 total uncompressed bytes, and 83,886,080 total archive bytes. Payload collection therefore permits at most 254 members.
- Fixed member time is `1980-01-01T00:00:00`; member names are portable ASCII and bytewise sorted.
- ZIP32, STORE, regular-file members only; no ZIP64, compression, encryption, data descriptors, extras, comments, or explicit directory entries.
- Do not publish a package, release, tag, asset, attestation, or hosted download in this plan.
- Do not call a digest a signature, authentication, origin proof, or trust root.
- All extraction tests use new disposable roots and prove no outside path changed after rejection.
- Run development, unit-test, and builder commands with the child-plan-1 `.venv/bin/python` validation environment.

## Frozen Archive Interfaces and Fields

| Type | Exact fields |
|---|---|
| `PayloadMember` | `path: PurePosixPath`, `mode: int`, `data: bytes` |
| `ManifestRecord` | `path: str`, `mode: str`, `byte_count: int`, `sha256: str` |
| `ArtifactPolicy` | `artifact_class: Literal["toolkit-source", "instance"]`, `maximum_members: int = 256`, `maximum_member_bytes: int = 8_388_608`, `maximum_total_bytes: int = 67_108_864`, `maximum_archive_bytes: int = 83_886_080`, `allowed_paths: tuple[str, ...]` |
| `VerifiedArchive` | frozen record with `artifact_class: str`, `members: tuple[ManifestRecord, ...]`, `manifest_sha256: str`, `archive_sha256: str`, `total_uncompressed_bytes: int`, `archive_bytes: bytes = field(repr=False)` |
| `ExtractionReport` | `destination_device: int`, `destination_inode: int`, `member_count: int`, `manifest_sha256: str`, `archive_sha256: str` |

| Module | Exact interface |
|---|---|
| `manifest.py` | `collect_payload(root, policy) -> tuple[PayloadMember, ...]`; `build_manifest(members, artifact_class) -> bytes`; `build_sha256sums(members, manifest_bytes) -> bytes` |
| `archive.py` | `write_canonical_zip(members, destination, policy) -> VerifiedArchive`; `verify_canonical_zip(archive_path, policy) -> VerifiedArchive`; `extract_verified_zip(verified, destination, policy) -> ExtractionReport` |
| `scanner.py` | `scan_verified_tree(root, extraction, policy) -> ScanReceipt` where `ScanReceipt` has `member_count: int`, `text_member_count: int`, and `finding_codes: tuple[str, ...]` |

`ArtifactPolicy.maximum_members` counts payload plus both control files. `collect_payload()` rejects the 255th payload member; `write_canonical_zip()` rejects any total count above 256.

`verify_canonical_zip()` opens the source once without following links, rejects a file larger than `maximum_archive_bytes` before allocation, reads at most 83,886,080 bytes, and parses those exact bytes. `VerifiedArchive.archive_bytes` is that immutable verified `bytes` object, uses `field(repr=False)`, and is excluded from receipts/logging. `extract_verified_zip()` accepts only a `VerifiedArchive`, recomputes its archive SHA-256, reparses its stored bytes independently, and never opens or refers to the original archive path. This closes the verification-to-extraction path-swap boundary with a fixed 80 MiB archive-input byte ceiling; it does not claim an 80 MiB process-RSS ceiling.

Every local header uses version-needed `10`, flags `0`, method `0`, DOS time `0`, DOS date `0x0021`, exact CRC-32 and sizes, and zero extra bytes. Every central header uses UNIX version-made-by `(3 << 8) | 30`, version-needed `10`, the same flags/method/time/date/CRC/sizes/name, zero extra/comment/disk/internal attributes, and external attributes `(stat.S_IFREG | mode) << 16`. The end record uses disk `0`, identical entry counts, exact central size/offset, and zero comment bytes.

## Task 1: Collect a canonical payload inventory

**Files:**
- Create: `src/runner_guard/manifest.py`
- Create: `tests/test_manifest.py`

**Interfaces:**
- Consumes: a new source/rendered root and an explicit `ArtifactPolicy` exact-path allow-list.
- Produces: `collect_payload(root, policy) -> tuple[PayloadMember, ...]`, bytewise sorted and bounded without mutating the source tree.

- [ ] **Step 1: Write failing inventory tests**

Accept only regular files with modes `0444`, `0555`, or source modes `0644`/`0755` according to the artifact-class allow-list. Reject symlinks, hard-linked files, sockets/devices/FIFOs, unreadable files, non-ASCII/control paths, absolute paths, empty/`.`/`..` components, backslashes, duplicate byte names, macOS casefold collisions, and Unicode normalization aliases.

```python
def test_collect_payload_is_bytewise_sorted_and_complete(self):
    members = collect_payload(self.fixture, self.policy)
    self.assertEqual(
        [member.path.as_posix() for member in members],
        sorted(path.as_posix() for path in self.expected_paths),
    )
```

Run:

```bash
.venv/bin/python -m unittest tests.test_manifest -v
```

Expected: FAIL with missing `runner_guard.manifest`.

- [ ] **Step 2: Implement descriptor-relative collection**

Open the root once, walk using descriptor-relative non-following operations, revalidate device/inode, and copy each member into memory only after enforcing the `ArtifactPolicy` member-count, per-member, total-byte, and exact-path allow-list. Do not use `Path.rglob()` as the security boundary.

- [ ] **Step 3: Add mutation and unchanged-tree proofs**

For each rejected shape, snapshot the complete fixture and prove collection performed no mutation.

- [ ] **Step 4: Run and commit**

```bash
.venv/bin/python -m unittest tests.test_manifest.PayloadCollectionTests -v
git add src/runner_guard/manifest.py tests/test_manifest.py
git commit -m "feat: collect canonical artifact payloads"
```

## Task 2: Build the manifest and checksum control files

**Files:**
- Modify: `src/runner_guard/manifest.py`
- Modify: `tests/test_manifest.py`

**Interfaces:**
- Consumes: validated `PayloadMember` records and an allowed artifact class.
- Produces: canonical `build_manifest(members, artifact_class) -> bytes` and `build_sha256sums(members, manifest_bytes) -> bytes` with the specified non-circular coverage.

- [ ] **Step 1: Write failing control-file tests**

`MANIFEST.json` contains exactly `schema_version`, `toolkit_version`, `artifact_class`, and `members`. Each member contains `path`, four-digit octal `mode`, `byte_count`, and lowercase SHA-256. It excludes `MANIFEST.json` and `SHA256SUMS`. `SHA256SUMS` contains every payload member plus `MANIFEST.json`, bytewise sorted, as `<64-lowercase-hex>  <ASCII-path>\n`, and excludes itself.

- [ ] **Step 2: Implement canonical bytes**

Use the umbrella JSON contract: sorted keys, compact separators, UTF-8, `ensure_ascii=False`, and one final LF. Reject an unknown artifact class; allowed values are `toolkit-source` and `instance`.

- [ ] **Step 3: Verify the no-cycle relationship**

Tests independently recompute every payload hash, parse the manifest, then verify `SHA256SUMS` binds the manifest. A one-byte mutation of any payload or manifest line fails.

- [ ] **Step 4: Run and commit**

```bash
.venv/bin/python -m unittest tests.test_manifest.ControlFileTests -v
git add src/runner_guard/manifest.py tests/test_manifest.py
git commit -m "feat: add artifact manifest and checksum controls"
```

## Task 3: Write a canonical ZIP32/STORE archive

**Files:**
- Create: `src/runner_guard/zip_format.py`
- Create: `src/runner_guard/archive.py`
- Create: `tests/test_archive_format.py`

**Interfaces:**
- Consumes: validated payload members, a nonexistent destination, and `ArtifactPolicy`.
- Produces: `write_canonical_zip(members, destination, policy) -> VerifiedArchive` using the exact ZIP32/STORE byte profile.

- [ ] **Step 1: Write failing byte-layout tests**

Pin every field listed in “Frozen Archive Interfaces and Fields,” local/central record order, end record, member offsets, no trailing data, and adjacent digest line format `<sha256>  <archive-filename>\n`.

- [ ] **Step 2: Implement explicit binary records**

Use `struct.pack()` and `zlib.crc32()`; do not use `zipfile.ZipFile` for writing. `write_canonical_zip()` constructs `MANIFEST.json` and `SHA256SUMS` from the supplied payload, appends those two controls, and applies the policy to the resulting total. Refuse an existing destination, nonregular parent, symlink ancestor, ZIP32 overflow, more than 254 payload members, more than 256 total members, or a member outside fixed limits.

- [ ] **Step 3: Add cross-environment golden construction**

Build the same three-member fixture with Python 3.9–3.14; compare the complete archive bytes and digest. Store only the small expected hex-header fixture, not a generated binary in Git.

- [ ] **Step 4: Run and commit**

```bash
.venv/bin/python -m unittest tests.test_archive_format.CanonicalWriterTests -v
git add src/runner_guard/zip_format.py src/runner_guard/archive.py tests/test_archive_format.py
git commit -m "feat: write canonical ZIP32 artifacts"
```

## Task 4: Verify archives with an independent parser

**Files:**
- Modify: `src/runner_guard/zip_format.py`
- Modify: `src/runner_guard/archive.py`
- Create: `tests/test_archive_rejections.py`

**Interfaces:**
- Consumes: an untrusted archive path and explicit `ArtifactPolicy` limits.
- Produces: `verify_canonical_zip(archive_path, policy) -> VerifiedArchive` only after an independent local/central-record and control-file verification; the result owns the exact immutable bounded bytes that were verified.

- [ ] **Step 1: Write adversarial archive fixtures**

Construct byte-level mutations for duplicate names, absolute/traversal/backslash names, empty components, control/non-ASCII names, casefold and normalization aliases, symlink/special external mode, encryption flag, data-descriptor flag, compression, extras/comments, directory entry, header disagreement, overlapping data, bad offset, bad CRC/size, truncated records, multiple end records, ZIP64 sentinel/record, count/size expansion, trailing bytes, and source-path replacement immediately after verification. Assert `repr(verified)` and every receipt/exception omit both retained archive bytes and payload excerpts.

- [ ] **Step 2: Implement an independent local/central parser**

Open the archive with no-follow regular-file checks, enforce `maximum_archive_bytes` before allocation/read completion, and parse the retained immutable bytes with `struct.unpack_from()` and explicit offset arithmetic. Do not call `zipfile.extractall()` or reuse writer record objects as verifier truth. Compare local and central fields, recompute CRC and SHA-256, validate both control files, and return those exact bytes in `VerifiedArchive`.

- [ ] **Step 3: Prove every mutation fails before extraction**

Each test records that no destination exists and no external fixture path changed.

- [ ] **Step 4: Run and commit**

```bash
.venv/bin/python -m unittest tests.test_archive_rejections -v
git add src/runner_guard/zip_format.py src/runner_guard/archive.py tests/test_archive_rejections.py
git commit -m "feat: verify canonical archives fail closed"
```

## Task 5: Extract only a verified archive

**Files:**
- Modify: `src/runner_guard/archive.py`
- Create: `tests/test_archive_extraction.py`

**Interfaces:**
- Consumes: a `VerifiedArchive` containing immutable verified bytes, a nonexistent destination, and the same `ArtifactPolicy`.
- Produces: `extract_verified_zip(verified, destination, policy) -> ExtractionReport` bound to the safely created destination identity; the original source pathname is never reopened.

- [ ] **Step 1: Write failing extraction-safety tests**

Require a nonexistent destination beneath a verified parent. Reject a symlinked parent/leaf, path appearing during creation, foreign device/owner where modeled, existing destination, mode drift, mutated `VerifiedArchive` metadata/bytes, and any raw path or archive object that has not passed full verification. Add a regression that verifies archive path A, replaces A before extraction, and proves extraction uses only the retained verified bytes and never the replacement.

- [ ] **Step 2: Implement descriptor-relative creation**

First recompute `sha256(verified.archive_bytes)`, compare it to `verified.archive_sha256`, and independently reparse those retained bytes under the same policy. Then create the destination mode `0700`, create ancestors relative to an opened destination descriptor, create each member with `O_CREAT|O_EXCL|O_NOFOLLOW`, write/sync bytes, apply exact mode, sync directories, and verify the extracted tree against the manifest before returning `ExtractionReport(destination_device, destination_inode, member_count, manifest_sha256, archive_sha256)`.

- [ ] **Step 3: Add interrupted extraction cleanup policy**

An interruption leaves the fail-if-exists destination quarantined. Do not delete it automatically; return a bounded recovery report with no filenames containing user data.

- [ ] **Step 4: Run and commit**

```bash
.venv/bin/python -m unittest tests.test_archive_extraction -v
git add src/runner_guard/archive.py tests/test_archive_extraction.py
git commit -m "feat: extract verified archives without following links"
```

## Task 6: Write adoption, operations, security, and troubleshooting guides

**Files:**
- Create: `docs/ADOPTION.md`
- Create: `docs/OPERATIONS.md`
- Create: `docs/SECURITY_MODEL.md`
- Create: `docs/TROUBLESHOOTING.md`
- Create: `docs/LESSONS_LEARNED.md`
- Create: `AGENT_ADOPTION_PROMPT.md`
- Modify: `README.md`
- Create: `tests/test_documentation_contract.py`

**Interfaces:**
- Consumes: the approved architecture, implemented operational contracts, known trust limitations, and exact adoption responsibilities.
- Produces: five bounded guides, `AGENT_ADOPTION_PROMPT.md`, README adoption links, and a machine-checked documentation contract.

- [ ] **Step 1: Write failing documentation-contract tests**

Require explicit sections for both trust profiles, repository/account/host boundaries, owner-only password/token/root steps, online routine operation, cleanup failure quarantine, 24-hour unmatched-label queue behavior, reboot gap, disk measurement, rollback, exact removal, unrelated-runner proof, two-failure RCA stop rule, and digest-not-signature wording. Reject language claiming VM isolation, hostile-process cleanup, automatic registration, production support, or a released package.

- [ ] **Step 2: Write the five guides**

Separate repository changes, host changes, and owner-only ceremonies. Include exact verification commands only for generated instance paths supplied by a validated contract; never include a real username or runner identity.

- [ ] **Step 3: Write the adoption-agent prompt**

The prompt requires read-only inventory, honest profile choice, isolated worktree, schema-valid contract, local render/tests, separate host authorization, exact-head CI, independent review, two idle observations before maintenance, unrelated-worker preservation, and online routine operation after qualification.

- [ ] **Step 4: Run link/content tests and commit**

```bash
.venv/bin/python -m unittest tests.test_documentation_contract -v
git diff --check
git add docs README.md AGENT_ADOPTION_PROMPT.md tests/test_documentation_contract.py
git commit -m "docs: add bounded runner adoption guidance"
```

## Task 7: Implement the final-byte scanner

**Files:**
- Create: `src/runner_guard/scanner.py`
- Create: `tests/test_artifact_scanner.py`

**Interfaces:**
- Consumes: a descriptor-bound extracted tree, its matching `ExtractionReport`, and `ScanPolicy`.
- Produces: `scan_verified_tree(root, extraction, policy) -> ScanReceipt` containing counts and finding codes only.

- [ ] **Step 1: Write failing bounded scanner tests**

Create table-driven tests for GitHub token prefixes, bearer and three-segment JWT forms, PEM/private-key headers, `.runner`, `.credentials`, `.credentials_rsaparams`, `_diag`, `_work`, keychain database names, SSH key names, absolute `/Users/<name>` paths outside explicitly marked synthetic examples, and a configurable tuple of forbidden private project names. Include punctuation, Unicode separators, minimum-length boundaries, placeholders, and complete-redaction assertions.

The scanner configuration is `ScanPolicy(forbidden_names, maximum_text_bytes=8_388_608, allowed_synthetic_user="example-runner")`. Unknown/binary content is rejected with `binary_member_forbidden`; it is never skipped.

- [ ] **Step 2: Implement bounded scanning and a content-free receipt**

`scan_verified_tree(root, extraction, policy)` requires the `ExtractionReport` returned for that exact descriptor-bound destination. It revalidates the destination device/inode, validates each member against the verified manifest, decodes strict UTF-8, limits bytes before decoding, and returns only `ScanReceipt(member_count, text_member_count, finding_codes)`. Exceptions contain finding codes and ordinal indexes, never paths containing private names or matched bytes.

- [ ] **Step 3: Mutation-prove coverage and redaction**

For every supported secret family, weaken or remove its rule in a test copy and prove the fixture would pass; then prove the real scanner rejects it. Snapshot the tree before each test and prove scanning never mutates it.

- [ ] **Step 4: Run and commit**

```bash
.venv/bin/python -m unittest tests.test_artifact_scanner -v
git add src/runner_guard/scanner.py tests/test_artifact_scanner.py
git commit -m "security: scan verified artifact bytes"
```

## Task 8: Build both artifact classes deterministically

**Files:**
- Create: `scripts/build-adoption-zip.py`
- Create: `tests/test_bundle_determinism.py`
- Modify: `pyproject.toml`

**Interfaces:**
- Consumes: a clean tracked toolkit source or exact rendered-instance tree, artifact/scanner policies, and new output directories.
- Produces: deterministic toolkit/instance ZIPs, verified manifests, adjacent digest records, extraction/scanner evidence, and no publication side effect.

- [ ] **Step 1: Write failing exact-inventory tests**

For toolkit `0.1.0`, the policy includes the complete tracked sets under `src/runner_guard/`, `schemas/`, `templates/`, `scripts/`, `tests/`, `docs/`, and `requirements/`, plus `README.md`, `LICENSE`, `SECURITY.md`, `CONTRIBUTING.md`, `AGENT_ADOPTION_PROMPT.md`, and `pyproject.toml`. The only permitted `.github` members are `.github/pull_request_template.md` and `.github/workflows/ci.yml`. Any other `.github` file requires an explicit policy/test update and focused governance review. Downloaded Actions outputs are never members.

The instance policy accepts exactly one rendered `instances/<validated-project-slug>/` tree containing these 15 members and exact modes: `instance.json` `0444`; `activate.py`, `cleanup_controller.py`, `job-started.sh`, `job-completed.sh`, `install-instance.sh`, `recover-instance.sh`, `rollback-instance.sh`, `remove-instance.sh`, `restore-instance.sh`, and `residue-audit.py` all `0555`; `workflow-guard.yml` `0444`; `workflow-policy.py` `0555`; and `ADOPTION_CHECKLIST.md` / `AGENT_PROMPT.md` both `0444`. The artifact writer then adds `MANIFEST.json` and `SHA256SUMS`. Reject an untracked, missing, extra, ignored, dirty, symlinked, nonregular, or wrong-mode source member.

- [ ] **Step 2: Implement the builder only after documentation exists**

```text
.venv/bin/python scripts/build-adoption-zip.py toolkit --source <clean-root> --output <new-dist>
.venv/bin/python scripts/build-adoption-zip.py instance --rendered <new-instance-root> --output <new-dist>
```

The CLI requires the source commit to equal `HEAD`, `git status --porcelain=v1 --untracked-files=all` to be empty, and the output directory not to exist. Instance staging is not required to be a Git checkout but must match the rendered public inventory exactly. The CLI uses `ArtifactPolicy`; it does not infer allowed members from whatever happens to be present.

Filenames are `macos-runner-guard-0.1.0-toolkit-<manifest12>.zip` and `macos-runner-guard-0.1.0-<project-slug>-<manifest12>.zip`. Each ZIP gets one adjacent `<filename>.sha256` file containing `<64-lowercase-hex>  <filename>\n`.

- [ ] **Step 3: Integrate verification, extraction, and scanning before success**

After writing, verify the archive with the independent parser, extract into a new disposable directory, compare the extraction report to the writer report, run `scan_verified_tree()`, and only then write the adjacent digest. Any failure preserves the ZIP as explicitly failed evidence but emits no success digest.

- [ ] **Step 4: Build twice from independent directories**

Copy the same approved source and instance bytes into two new roots with different parent names, mtimes, and umasks. Build with Python 3.9–3.14 on Ubuntu and the two macOS lanes. Both ZIP classes and adjacent digest files must be byte-identical. Assert that all 256-member, per-member, uncompressed-total, and 83,886,080-byte archive boundaries count both control files.

- [ ] **Step 5: Run and commit**

```bash
.venv/bin/python -m unittest tests.test_bundle_determinism tests.test_artifact_scanner -v
git add scripts/build-adoption-zip.py tests/test_bundle_determinism.py pyproject.toml
git commit -m "feat: build verified deterministic adoption bundles"
```

## Task 9: Complete subsystem reproducibility and independent review

**Interfaces:**
- Consumes: the complete artifact subsystem at one exact clean source commit/tree and two independently created build roots.
- Produces: immutable validation evidence, reproducibility hashes, three fresh review results, and either the required accepted endpoint or an explicit blocker; it creates no release.

- [ ] **Step 1: Run the full artifact gate**

```bash
.venv/bin/python -m compileall src tests scripts -q
.venv/bin/python -m unittest discover -s tests -v
.venv/bin/python -m pytest -q
git diff --check
```

Expected: all artifact, extraction, scanner, documentation, and determinism tests pass; toolkit and instance archives reproduce byte-for-byte in two clean directories. The umbrella plan creates `scripts/validate.py` only after all four child plans, so this child plan must not call it.

- [ ] **Step 2: Record immutable test evidence**

Record source commit/tree, toolkit archive filename/hash, instance archive filename/hash, both manifest hashes, member counts, total sizes, Python/OS matrix, scanner result, and validation totals. Keep the test archives local and clearly label them non-release evidence.

- [ ] **Step 3: Obtain fresh independent reviews**

Require archive-format/safe-extraction review, supply-chain/secret-scan review, and adoption-claims review on the exact head. Any fix invalidates prior exact-head review and both archive hashes.

- [ ] **Step 4: Commit only review-driven corrections**

If review changes executable bytes, rerun every artifact gate and rebuild twice. If it changes documentation bytes, rebuild both toolkit archives because documentation is payload. Do not create a release in this plan.

## Acceptance Gate

- [ ] Both artifact classes are byte-identical across independent clean builds.
- [ ] Every archive member and control file is covered exactly as specified.
- [ ] Every adversarial ZIP shape fails before extraction and without outside mutation.
- [ ] Final extracted bytes pass the secret/forbidden-state scan.
- [ ] Adoption documentation states the trust boundary and owner-only actions accurately.
- [ ] Exact-head independent reviews have zero actionable findings.
- [ ] No tag, release, package publication, live adoption, or credential operation occurred.

Required endpoint:

```text
ARTIFACTS_AND_ADOPTION_ACCEPTED
```
