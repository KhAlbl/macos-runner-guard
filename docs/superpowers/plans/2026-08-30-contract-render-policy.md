# Contract, Renderer, and Workflow Policy Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the host-independent contract, workflow-policy, and deterministic renderer core for macOS Runner Guard without touching a live runner or host configuration.

**Architecture:** A strict JSON instance contract is the only input to pure validation, policy evaluation, and rendering. Workflow policy is implemented before rendering because each bundle embeds that reviewed policy source. Cleanup owns phase-specific environment binding, `RunIdentity`, and `JobIdentity`; this child validates only the four untrusted workflow job-identity fields.

**Tech Stack:** CPython 3.9 or newer, Python standard library, JSON Schema 2020-12, PyYAML 6.0.3 only for workflow parsing, `unittest`, GitHub-hosted Ubuntu and macOS runners.

**Spec:** [Approved architecture](../../ARCHITECTURE.md)

## Global Constraints

- Toolkit and package version is exactly `0.1.0`; Python requirement is `>=3.9`.
- Contract schema version is exactly `macos-runner-guard.instance.v1`; schema identifier is exactly `urn:macos-runner-guard:schema:instance:v1`.
- Contract JSON is duplicate-key-free UTF-8 and canonicalizes with sorted keys, compact separators, `ensure_ascii=False`, and one final newline.
- All four cleanup limits and `minimum_free_bytes` are required positive JSON integers. No universal default, fallback, or inherited value exists.
- `temp_root` is exactly `work_root / "_temp"`, the location GitHub Runner exposes as `$RUNNER_TEMP`; a sibling `temp/` path is invalid.
- Job fields full-match run ID `[1-9][0-9]{0,19}`, run attempt `[1-9][0-9]{0,9}`, job `[A-Za-z0-9_.-]{1,64}`, and leg `[A-Za-z0-9_.-]{1,32}`; leg is an exact `allowed_legs` member.
- `allowed_actions` is a non-empty, bytewise-sorted tuple of complete `owner/repository@40-lowercase-hex` identities. The rendered workflow may use only those exact identities; a different owner, repository, ref, SHA, local action, Docker action, or reusable-workflow job is rejected.
- `workspace_root` is exactly `work_root / github_repository / github_repository` and is disjoint from `temp_root`, every guarded job root, and the reserved `_actions`, `_diag`, `_temp`, `_tool`, cache, controller, runner, evidence, and derived global operation-package roots.
- Supported architecture values are exactly `ARM64` and `X64`.
- PyYAML is exactly 6.0.3, only the host-independent policy CLI imports it, and construction uses `yaml.safe_load` only.
- A separately reviewed bootstrap lock hash-locks `pip-tools==7.6.1` and every transitive wheel required on Python 3.9-3.14. Validation requirements hash-lock exactly PyYAML 6.0.3, pytest 8.4.2, and setuptools 82.0.1. Pytest 8.4.2 and setuptools 82.0.1 are the newest selected lines that retain Python 3.9 support; pytest 9.1.1 and setuptools 83/84 require Python 3.10 or newer and are forbidden while Python 3.9 remains supported.
- Lock generation is the only separately reviewed dependency-resolution step. Development and CI do not install this repository at all: they expose `src/` through an explicit `PYTHONPATH`. Every network validation-tool installation uses the matching interpreter-specific lock with `--require-hashes --only-binary=:all:`; no editable install, sdist, or ambient package is accepted.
- Installed cleanup, activation, hook, transition, rollback, removal, restoration, and residue code remains standard-library-only under `/usr/bin/python3 -I -S`; this child does not implement it.
- Rendering is host-independent and fails when its output path exists.
- Root `MANIFEST.json` and `SHA256SUMS` are generated later inside artifact staging; they are not committed or emitted here.
- CI covers Python 3.9-3.14 on Ubuntu and Python 3.11/3.14 on macOS.
- No task may inspect credentials, operate a live runner, use `sudo`, alter services/keychains/agents, or write outside the repository and test-created temporary directories.
- No executable release, package publication, live adoption, or production-support claim is authorized.

---

## Public Shared Interfaces

```text
load_contract(path: Path) -> InstanceContract
read_regular_file_bytes(path: Path, maximum_bytes: int) -> bytes
load_contract_bytes(data: bytes) -> InstanceContract
canonical_contract_bytes(contract: InstanceContract) -> bytes
BindingRecord(path: str, mode: int, byte_count: int, sha256: str)
build_binding_manifest(schema_version: str,
                       records: Sequence[BindingRecord]) -> bytes
binding_manifest_sha256(schema_version: str,
                        records: Sequence[BindingRecord]) -> str
validate_run_identity_fields(run_id: object,
                             run_attempt: object) -> tuple[str, str]
validate_job_identity_fields(contract: InstanceContract, run_id: object,
                             run_attempt: object, github_job: object,
                             leg: object) -> tuple[str, str, str, str]
PolicyViolation(code: str, path: str, message: str)
load_workflow_bytes(data: bytes) -> Mapping[str, object]
check_workflow(document: Mapping[str, object],
               contract: Mapping[str, object]) -> tuple[PolicyViolation, ...]
render_instance(contract: InstanceContract,
                components: Sequence[ComponentPayload],
                templates: Sequence[TemplatePayload],
                policy_source: bytes) -> tuple[RenderedFile, ...]
merge_component_payloads(base: Sequence[ComponentPayload],
                         replacements: Sequence[ComponentPayload]) -> tuple[ComponentPayload, ...]
load_component_payloads(root: Path) -> tuple[ComponentPayload, ...]
load_template_payloads(root: Path) -> tuple[TemplatePayload, ...]
build_rendered_payload_binding(files: Sequence[RenderedFile]) -> bytes
rendered_payload_binding_sha256(files: Sequence[RenderedFile]) -> str
canonical_render_receipt_bytes(files: Sequence[RenderedFile]) -> bytes
load_render_receipt_bytes(data: bytes) -> RenderReceipt
write_rendered_tree(output_dir: Path,
                    files: Sequence[RenderedFile]) -> None
```

No `RunIdentity`, `JobIdentity`, or environment reader belongs in this child.

## File Map

| Path | Responsibility |
|---|---|
| `pyproject.toml` | Package version, Python floor, policy extra, and CLI entry point. |
| `requirements/bootstrap-py39.txt` … `bootstrap-py314.txt` | Per-interpreter reviewed wheel-only hashes for `pip-tools` and its complete bootstrap dependency closure. |
| `requirements/validation.in` / `requirements/validation-py39.txt` … `validation-py314.txt` | Exact validation inputs and per-interpreter reviewed wheel hashes. |
| `requirements/metadata/pypi-wheel-metadata.json` | Canonical checked fixture of the separately reviewed PyPI filenames, sizes, `Requires-Python`, and hashes used by every lock. |
| `schemas/instance-v1.schema.json` | Portable non-secret contract schema. |
| `src/runner_guard/jsonio.py` | Duplicate-safe JSON, canonical UTF-8 bytes, and bounded no-follow regular-file reads. |
| `src/runner_guard/bindings.py` | Lifecycle-neutral `BindingRecord`, canonical frozen binding manifests, and rendered-payload binding digests. |
| `src/runner_guard/contract.py` | Contract types, public loaders, serialization, separation, and job fields. |
| `src/runner_guard/policy.py` | Bounded YAML parsing and workflow checks; rendered standalone source. |
| `src/runner_guard/render.py` | Pure rendering and fail-if-exists writer. |
| `src/runner_guard/cli.py` | Public commands and exit codes. |
| `templates/*`, `examples/*`, `tests/*` | Templates, non-secret examples, and verification. |
| `.github/workflows/ci.yml` | Hosted supported-version validation. |

### Task 1: Establish Package, Lock, Errors, and Canonical JSON

**Files:**
- Create: `pyproject.toml`
- Create: `requirements/bootstrap-py39.txt` through `requirements/bootstrap-py314.txt`
- Create: `requirements/validation.in`
- Create: `requirements/validation-py39.txt` through `requirements/validation-py314.txt`
- Create: `requirements/metadata/pypi-wheel-metadata.json`
- Create: `src/runner_guard/__init__.py`
- Create: `src/runner_guard/errors.py`
- Create: `src/runner_guard/jsonio.py`
- Create: `tests/test_jsonio.py`
- Create: `tests/test_requirements_policy.py`

**Interfaces:**
- Consumes: UTF-8 JSON bytes.
- Produces: `__version__ == "0.1.0"`; `ValidationIssue`; `GuardError`; `JsonFormatError`; `ContractError`; `load_json_object`; `canonical_json_bytes`.

- [ ] **Step 1: Write failing duplicate and UTF-8 tests**

```python
class JsonIoTests(unittest.TestCase):
    def test_duplicate_key_is_rejected(self) -> None:
        with self.assertRaisesRegex(JsonFormatError, "^duplicate_json_key$"):
            load_json_object(b'{"profile":"a","profile":"b"}\n')

    def test_duplicate_sensitive_looking_key_is_never_echoed(self) -> None:
        data = b'{"github_' + b'pat_example":"a","github_' + b'pat_example":"b"}\n'
        with self.assertRaisesRegex(JsonFormatError, "^duplicate_json_key$") as raised:
            load_json_object(data)
        self.assertNotIn("github_", str(raised.exception))

    def test_canonical_bytes_are_sorted_utf8(self) -> None:
        self.assertEqual(canonical_json_bytes({"z": "\u0627", "a": 1}),
                         '{"a":1,"z":"\u0627"}\n'.encode("utf-8"))
```

- [ ] **Step 2: Run and confirm missing imports**

Run: `python3 -I -S -m unittest discover -s tests -p 'test_jsonio.py' -v`

Expected: FAIL because the package modules do not exist.

- [ ] **Step 3: Add exact metadata and errors**

```toml
[build-system]
requires = ["setuptools==82.0.1"]
build-backend = "setuptools.build_meta"

[project]
name = "macos-runner-guard"
version = "0.1.0"
requires-python = ">=3.9"
dependencies = []

[project.optional-dependencies]
policy = ["PyYAML==6.0.3"]
```

```python
@dataclass(frozen=True, order=True)
class ValidationIssue:
    code: str
    path: str
    message: str


class GuardError(ValueError):
    pass


class JsonFormatError(GuardError):
    pass


class ContractError(GuardError):
    pass
```

- [ ] **Step 4: Implement canonical JSON**

```python
def load_json_object(data: bytes) -> Dict[str, object]:
    text = data.decode("utf-8", errors="strict")
    value = json.loads(text, object_pairs_hook=_reject_duplicate_pairs)
    if not isinstance(value, dict):
        raise JsonFormatError("top-level JSON value must be an object")
    return value


def canonical_json_bytes(value: object) -> bytes:
    text = json.dumps(value, ensure_ascii=False, allow_nan=False,
                      sort_keys=True, separators=(",", ":"))
    return (text + "\n").encode("utf-8")
```

`_reject_duplicate_pairs` raises `JsonFormatError` on the first repeated key. Convert UTF-8, JSON, recursion, non-object, non-finite-number, and unsupported-type errors to stable bounded messages without echoing input.

- [ ] **Step 5: Create and prove the reviewed lock inputs before any network installation**

As part of this task—before creating `.venv` or installing any network package—create the six `requirements/bootstrap-py<minor>.txt` files, the canonical `requirements/metadata/pypi-wheel-metadata.json` fixture, and `tests/test_requirements_policy.py`. Each bootstrap lock is complete for its exact CPython minor (`3.9` through `3.14`), platform-marked where required, wheel-only, and closes `pip-tools==7.6.1` plus its applicable transitive requirements. Obtain every retained filename, byte count, `Requires-Python`, and SHA-256 from the official PyPI JSON API in a separate read-only review; include only applicable wheel hashes and reject sdists. The stdlib-only test parses all six locks and independently compares their closed names, versions, markers, wheel filenames, sizes, compatibility claims, and hashes with the checked canonical metadata fixture. It also rejects a missing interpreter lock, metadata entry, sdist, unreviewed hash, incompatible `Requires-Python`, or unhashed network-install command. No implementation command may install a network package from a bare version constraint.

`requirements/validation.in` contains exactly:

```text
PyYAML==6.0.3
pytest==8.4.2
setuptools==82.0.1
```

First run the lock proof without site packages:

```bash
python3 -I -S -m unittest discover -s tests -p 'test_requirements_policy.py' -v
```

Then, for each minor `39 310 311 312 313 314`, use an exact matching interpreter in a fresh disposable environment, install only `requirements/bootstrap-py<minor>.txt`, and generate `requirements/validation-py<minor>.txt` from the one `validation.in`. `pip-compile` runs under that same minor; a lock generated under one interpreter is never relabelled for another.

```bash
test ! -e .lock-venv-py<minor>
python<major.minor> -m venv .lock-venv-py<minor>
.lock-venv-py<minor>/bin/python -m pip install \
  --require-hashes --only-binary=:all: \
  -r requirements/bootstrap-py<minor>.txt
.lock-venv-py<minor>/bin/pip-compile \
  --generate-hashes --allow-unsafe --pip-args='--only-binary=:all:' \
  --output-file=requirements/validation-py<minor>.txt \
  requirements/validation.in
```

After all six files exist, rerun the stdlib-only lock proof and require every direct/transitive requirement and retained wheel hash to match official metadata for its interpreter. Record that pytest 8.4.2 and setuptools 82.0.1 preserve Python 3.9; reject pytest 9.1.1 and setuptools 83/84 because they require Python 3.10 or newer. Delete the six disposable lock-generation environments only after their lock bytes have been reviewed; they are never committed.

The Task-1 requirements test owns the repository scan that rejects every network-capable `pip install` command lacking both `--require-hashes` and `--only-binary=:all:`. Development and CI do not install this checkout in editable mode: they set `PYTHONPATH` to the explicit reviewed `src/` root. This prevents setuptools metadata or build products from modifying the source tree before clean-source artifact construction. It must catch a reintroduced editable install or bare `pip-tools==...` bootstrap. Task 6 extends this same test for the final workflow; it does not create the safety boundary after the locks have already been used.

- [ ] **Step 6: Install, test, and commit**

```bash
python3 -m venv --clear .venv
validation_minor="$(PYTHONPATH="$PWD/src" .venv/bin/python -c 'import sys; print(f"{sys.version_info.major}{sys.version_info.minor}")')"
.venv/bin/pip install --require-hashes --only-binary=:all: \
  -r "requirements/validation-py${validation_minor}.txt"
PYTHONPATH="$PWD/src" .venv/bin/python -m unittest tests.test_jsonio tests.test_requirements_policy -v
git add pyproject.toml requirements src/runner_guard \
  tests/test_jsonio.py tests/test_requirements_policy.py
git commit -m "feat: establish contract serialization boundary"
```

Expected: all JSON tests pass.

### Task 2: Define Contract Loaders, Separation, and Job Fields

**Files:**
- Create: `schemas/instance-v1.schema.json`
- Create: `src/runner_guard/contract.py`
- Create: `tests/test_contract.py`

**Interfaces:**
- Consumes: contract bytes/path, a contract sequence, and four untrusted job fields.
- Produces: `CleanupLimits`; `InstanceLayout`; `InstanceContract` with `repository`; `load_contract`; `load_contract_bytes`; `canonical_contract_bytes`; `validate_contract_set`; `validate_run_identity_fields`; `validate_job_identity_fields`.

Require exactly these fields and reject extras:

```text
schema_version, toolkit_version, project_slug, profile,
shared_account_risk_acknowledged, github_owner, github_repository,
default_branch, account_name, account_home, runner_name_prefix,
runner_label, service_name_prefix, instance_root, runner_root,
work_root, temp_root, workspace_root, controller_root, cache_root,
evidence_root, job_prefix, allowed_legs, platform, architecture,
allowed_actions, cleanup_limits, minimum_free_bytes, routine_availability,
reboot_login_reconnect_qualified
```

- [ ] **Step 1: Write fixture and public-interface tests**

```python
def valid_contract(architecture: str = "ARM64") -> dict[str, object]:
    root = "/Library/Application Support/macos-runner-guard/instances/acme-widget"
    return {
        "schema_version": "macos-runner-guard.instance.v1",
        "toolkit_version": "0.1.0", "project_slug": "acme-widget",
        "profile": "dedicated-account", "shared_account_risk_acknowledged": False,
        "github_owner": "acme", "github_repository": "widget",
        "default_branch": "main", "account_name": "acmewidgetrunner",
        "account_home": "/Users/acmewidgetrunner",
        "runner_name_prefix": "acme-widget-ci-", "runner_label": "acme-widget-ci",
        "service_name_prefix": "actions.runner.acme.",
        "instance_root": root, "runner_root": root + "/runner",
        "work_root": root + "/work", "temp_root": root + "/work/_temp",
        "workspace_root": root + "/work/widget/widget",
        "controller_root": root + "/controller", "cache_root": root + "/cache",
        "evidence_root": root + "/evidence", "job_prefix": "acme-widget-job",
        "allowed_legs": ["main", "3.11", "3.12", "3.13", "3.14"],
        "allowed_actions": ["actions/checkout@fbc6f3992d24b796d5a048ff273f7fcc4a7b6c09"],
        "platform": "macOS", "architecture": architecture,
        "cleanup_limits": {"max_entries": 200000, "max_depth": 64,
                           "max_bytes": 53687091200, "max_seconds": 120},
        "minimum_free_bytes": 53687091200,
        "routine_availability": "online-after-qualification",
        "reboot_login_reconnect_qualified": False,
    }


class ContractTests(unittest.TestCase):
    def test_loaders_and_canonical_bytes_agree(self) -> None:
        data = encoded(valid_contract())
        with temporary_contract(data) as path:
            self.assertEqual(load_contract(path), load_contract_bytes(data))
        self.assertEqual(canonical_contract_bytes(load_contract_bytes(data)), data)

    def test_architecture_enum_accepts_both_values(self) -> None:
        for value in ("ARM64", "X64"):
            self.assertEqual(load_contract_bytes(encoded(valid_contract(value))).architecture,
                             value)

    def test_repository_property_is_canonical_owner_slash_name(self) -> None:
        contract = load_contract_bytes(encoded(valid_contract()))
        self.assertEqual(contract.repository, "acme/widget")

    def test_job_fields_return_validated_strings_only(self) -> None:
        contract = load_contract_bytes(encoded(valid_contract()))
        self.assertEqual(validate_job_identity_fields(
            contract, "123", "2", "memory-abi", "3.14"),
            ("123", "2", "memory-abi", "3.14"))
```

- [ ] **Step 2: Run and confirm missing contract module**

Run: `PYTHONPATH="$PWD/src" .venv/bin/python -m unittest tests.test_contract -v`

Expected: FAIL because `runner_guard.contract` does not exist.

- [ ] **Step 3: Add exact schema and frozen types**

Use draft 2020-12, exact `$id`, `additionalProperties: false`, the complete required/property sets, all four required positive cleanup integers, required positive `minimum_free_bytes`, profile acknowledgement conditions, and these exact ASCII-only string rules in both schema and parser:

| Field | Exact rule |
|---|---|
| `project_slug` | lowercase ASCII, 1–32 bytes, full-match `[a-z0-9](?:[a-z0-9-]{0,30}[a-z0-9])?` |
| `github_owner` | exact API-canonical ASCII case, 1–39 bytes, full-match `[A-Za-z0-9](?:[A-Za-z0-9-]{0,37}[A-Za-z0-9])?`, with `--` rejected |
| `github_repository` | exact API-canonical ASCII case, 1–100 bytes, full-match `[A-Za-z0-9](?:[A-Za-z0-9._-]{0,98}[A-Za-z0-9_-])?`, with `..`, case-insensitive `.git`, and normalization/casefold aliases rejected |
| `default_branch` | ASCII, 1–128 bytes, full-match `[A-Za-z0-9](?:[A-Za-z0-9._/-]{0,126}[A-Za-z0-9_-])?`, plus the complete Git-ref exclusions below |
| `account_name` | lowercase ASCII, 1–31 bytes, full-match `[a-z_][a-z0-9_-]{0,30}`, with `_` alone rejected |
| `runner_label` | lowercase ASCII, 1–64 bytes, full-match `[a-z0-9](?:[a-z0-9._-]{0,62}[a-z0-9])?` |
| `runner_name_prefix` | exactly `project_slug + "-ci-"` |
| `service_name_prefix` | exactly `"actions.runner." + github_owner + "."`; the exact accepted service identity is this organization-scope prefix plus the independently joined exact runner name, and authentic org-scope service fixtures pin that derivation rather than assuming repository-scope naming |
| `job_prefix` | exactly `project_slug + "-job"` |

All values must already be NFC. Lowercase-only fields must equal their ASCII casefold form; `github_owner` and `github_repository` instead preserve the exact authenticated API spelling, and a separately derived ASCII-casefold key is used only for collision/alias checks. A case variant is never silently normalized and cannot satisfy later exact metadata equality. `default_branch` additionally rejects controls/space, backslash, tilde, caret, colon, question mark, asterisk, `[`, consecutive slash or dot, `@{`, a leading/trailing slash or dot, a `.lock` suffix, and a `.` or `..` path segment. This deliberately supports a conservative subset of valid Git refs. It must exactly equal authenticated repository metadata later; contract parsing does not shell out to Git.

Architecture is:

```json
{"type":"string","enum":["ARM64","X64"]}
```

Define frozen `CleanupLimits`, `InstanceLayout`, and `InstanceContract`; every schema field is typed, path fields use `PurePosixPath`, and `allowed_legs` uses `tuple[str, ...]`. Define the cleanup-consumed canonical repository property exactly:

```python
@property
def repository(self) -> str:
    return f"{self.github_owner}/{self.github_repository}"
```

- [ ] **Step 4: Implement exact public loaders and canonical output**

```python
def load_contract(path: Path) -> InstanceContract:
    return load_contract_bytes(read_regular_file_bytes(path, 1_000_000))


def load_contract_bytes(data: bytes) -> InstanceContract:
    return _parse_contract(load_json_object(data))


def canonical_contract_bytes(contract: InstanceContract) -> bytes:
    return canonical_json_bytes(_contract_to_mapping(contract))
```

`read_regular_file_bytes()` opens each absolute or working-directory-relative component descriptor-relatively, rejects symlink ancestors and leaves, requires a regular single-link file, enforces the byte limit before allocation, reads from the retained descriptor, and revalidates device/inode/type/link-count/size after reading. It never validates by pathname and then reopens that path.

Validate every raw string before converting paths or rendering. Apply the exact schema/parser rules above byte-for-byte; reject YAML-leading/metacharacter candidates (`&`, `*`, `!`, `#`, colon, comma, braces, brackets), controls/newlines, separators, overlengths, slash/dot aliases, NFC/casefold collisions, and any schema/parser disagreement. Include an authentic mixed-case owner fixture such as `KhAlbl`, prove it round-trips unchanged, and reject a case-variant contract against differently cased authenticated metadata while still treating case variants as collision aliases. Validate raw path text before `PurePosixPath`; require leading `/`; reject `//`, `~`, `$`, backslash, controls, `.`, and `..`. Never call `resolve()` or inspect the host. Require `account_home == PurePosixPath("/Users") / account_name`; exact instance/account roots; direct runner/work/controller/cache/evidence children; `temp_root == work_root / "_temp"`; and `workspace_root == work_root / github_repository / github_repository`. Derive the protected global operation-package path from the fixed toolkit root and validated project slug; it is not a caller-supplied contract field. Reject a repository component or any derived path that overlaps `_temp`, a guarded job prefix, `_actions`, `_diag`, `_tool`, cache, controller, runner, evidence, global operations, or another reserved root. Add mutations for every string boundary, derived-field mismatch, overlap, and sibling `instance_root / "temp"`.

Parse `allowed_actions` as a non-empty, unique, bytewise-sorted tuple and require every member to full-match `[A-Za-z0-9_.-]+/[A-Za-z0-9_.-]+@[0-9a-f]{40}`. The reviewed examples allow exactly `actions/checkout@fbc6f3992d24b796d5a048ff273f7fcc4a7b6c09`; changing any owner, repository, case, separator, or SHA is a contract change.

- [ ] **Step 5: Implement job-field validation only**

```python
RUN_ID_RE = re.compile(r"[1-9][0-9]{0,19}\Z", re.ASCII)
RUN_ATTEMPT_RE = re.compile(r"[1-9][0-9]{0,9}\Z", re.ASCII)
JOB_RE = re.compile(r"[A-Za-z0-9_.-]{1,64}\Z", re.ASCII)
LEG_RE = re.compile(r"[A-Za-z0-9_.-]{1,32}\Z", re.ASCII)


def validate_job_identity_fields(contract: InstanceContract, run_id: object,
                                 run_attempt: object, github_job: object,
                                 leg: object) -> tuple[str, str, str, str]:
    run_id, run_attempt = validate_run_identity_fields(run_id, run_attempt)
    values = (github_job, leg)
    patterns = (JOB_RE, LEG_RE)
    codes = ("github_job_invalid", "leg_invalid")
    for value, pattern, code in zip(values, patterns, codes):
        if not isinstance(value, str) or pattern.fullmatch(value) is None:
            raise ContractError(code)
    if leg not in contract.allowed_legs:
        raise ContractError("leg_not_allowed")
    return run_id, run_attempt, github_job, leg


def validate_run_identity_fields(run_id: object,
                                 run_attempt: object) -> tuple[str, str]:
    values = (run_id, run_attempt)
    patterns = (RUN_ID_RE, RUN_ATTEMPT_RE)
    codes = ("run_id_invalid", "run_attempt_invalid")
    for value, pattern, code in zip(values, patterns, codes):
        if not isinstance(value, str) or pattern.fullmatch(value) is None:
            raise ContractError(code)
    return run_id, run_attempt
```

Test zero, sign, padding, whitespace, newline, overlength, Unicode, controls, non-string values, and unlisted legs. Do not read environment, create a basename, or define `JobIdentity`.

- [ ] **Step 6: Implement separation and schema parity**

Compare repository, project slug, label, runner/service/job prefixes, account, declared roots, and the derived global operation-package root with NFC casefolding. Reject an exact or nested cross-contract root and reject two contracts that derive the same `operations/<project-slug>/` namespace even when their declared instance roots differ. Reject account reuse when either profile is dedicated; permit only two acknowledged shared profiles with all other identities disjoint. Test every collision class. Load the schema and compare required/property/cleanup sets, constants, enums, patterns, and profile conditions with parser tests.

- [ ] **Step 7: Run and commit**

```bash
PYTHONPATH="$PWD/src" .venv/bin/python -m unittest tests.test_contract -v
git add schemas/instance-v1.schema.json src/runner_guard/contract.py tests/test_contract.py
git commit -m "feat: define instance contract v1"
```

Expected: all contract tests pass for ARM64 and X64.

### Task 3: Implement Bounded Workflow Policy Before Rendering

**Files:**
- Create: `src/runner_guard/policy.py`
- Create: `tests/test_workflow_policy.py`

**Interfaces:**
- Consumes: workflow UTF-8 YAML bytes and canonical contract mapping.
- Produces: public `PolicyViolation`; `load_workflow_bytes(data)`; public `check_workflow(document, contract)`; `main(argv=None) -> int`, with exact standalone behavior defined below.

`policy.py` contains the exact one-line sentinel `INSTANCE_CONTRACT_JSON = None`. Rendering replaces only that line with `INSTANCE_CONTRACT_JSON = bytes.fromhex("<lowercase-canonical-contract-hex>")`. Raw JSON is never inserted as Python syntax. The policy module's `_load_embedded_contract()` rejects a missing/non-`bytes` binding, duplicate JSON keys, a non-object root, invalid UTF-8, and bytes that are not the exact canonical encoding of the parsed mapping.

The generated `workflow-policy.py WORKFLOW` interface accepts exactly one positional workflow path, reads at most 2,000,000 bytes through a no-follow regular-file descriptor, and emits no path or YAML value. It prints exactly `WORKFLOW_POLICY=pass\n` and returns `0`, or `WORKFLOW_POLICY=fail codes=<sorted-comma-codes>\n` and returns `2` for usage, input, or policy rejection. An unbound contract or unexpected internal failure prints only `WORKFLOW_POLICY=fail codes=internal_error\n` and returns `3`. The module ends with `raise SystemExit(main())` under the ordinary `__main__` guard.

- [ ] **Step 1: Write routing, mutation, and YAML-bomb tests**

```python
class WorkflowPolicyTests(unittest.TestCase):
    def test_both_architecture_routes_pass(self) -> None:
        for architecture in ("ARM64", "X64"):
            contract = valid_contract(architecture)
            self.assertEqual(check_workflow(guarded_workflow(contract), contract), ())

    def test_wrong_architecture_fails(self) -> None:
        contract = valid_contract("X64")
        workflow = guarded_workflow(contract)
        workflow["jobs"]["verify"]["runs-on"] = ["self-hosted", "macOS", "ARM64"]
        self.assertIn("routing.labels",
                      {item.code for item in check_workflow(workflow, contract)})

    def test_anchor_alias_merge_and_bomb_fail_before_construction(self) -> None:
        samples = (
            b"a: &x [1]\nb: *x\n",
            b"a: &x {k: v}\nb: {<<: *x}\n",
            b"a: &a [x,x,x,x]\nb: &b [*a,*a,*a,*a]\nc: [*b,*b,*b,*b]\n",
        )
        for data in samples:
            with self.assertRaisesRegex(ValueError, "yaml_forbidden_token"):
                load_workflow_bytes(data)
```

- [ ] **Step 2: Run and confirm missing policy**

Run: `PYTHONPATH="$PWD/src" .venv/bin/python -m unittest tests.test_workflow_policy -v`

Expected: FAIL because `runner_guard.policy` does not exist.

- [ ] **Step 3: Scan tokens before compose and safe construction**

Decode at most 2,000,000 UTF-8 bytes. Iterate `yaml.scan(text)` first with a 100,000-token limit. Reject `AliasToken`, `AnchorToken`, `TagToken`, merge scalar `<<`, nonstandard directives, and scalar values above 1,000,000 characters. While scanning, track block/flow collection start/end tokens with a nonrecursive counter and reject nesting above 64 or an unbalanced token stream before composition. No compose/load call occurs before this scan passes.

Then call `yaml.compose(text)` and walk at most 100,000 nodes to depth 64, rejecting duplicate scalar spellings, nonstandard tags, cycles, and distinct scalar keys that PyYAML would construct as the same key. At the document root, the event key must be the exact plain scalar `on`; `true`, `yes`, case variants, quoted aliases, and any combination or ordering that would resolve/collide with boolean `True` are rejected before `yaml.safe_load`. Only after these gates call `yaml.safe_load(text)`; normalize the one already-proven root `True` key to string `on` without overwriting another key.

Add tests for anchors, aliases, merges, custom tags, nested alias bombs, token/node/depth/scalar bounds, duplicate keys, malformed UTF-8, and parser call order. Plain nested mappings and sequences at depth 64 pass the scan; depth 65 is rejected while patched `yaml.compose` and `yaml.safe_load` both remain uncalled. Explicit event-key mutations cover `on` plus `true`, `on` plus `yes`, `true` only, `yes` only, case variants, quoted variants, and reversed ordering; no case may reach `safe_load` unless the root event key is exactly the approved `on`. Patch `yaml.compose` and `yaml.safe_load` in forbidden-token tests and assert neither was called where the corresponding earlier gate must stop construction.

- [ ] **Step 4: Implement public violations and checks**

```python
INSTANCE_CONTRACT_JSON = None


@dataclass(frozen=True, order=True)
class PolicyViolation:
    code: str
    path: str
    message: str


def check_workflow(document: Mapping[str, object],
                   contract: Mapping[str, object]) -> tuple[PolicyViolation, ...]:
    checks = (
        _check_permissions, _check_events, _check_routing,
        _check_actions, _check_checkout,
        _check_no_concurrency, _check_job_limits, _check_shell,
        _check_forbidden_capabilities, _check_prepare, _check_cleanup,
    )
    return tuple(sorted(issue for check in checks
                        for issue in check(document, contract)))
```

Implement every named `_check_*` helper with signature `(document: Mapping[str, object], contract: Mapping[str, object]) -> tuple[PolicyViolation, ...]`. In listed order they enforce root permissions, events, routing, Action identities, checkout, absence of concurrency replacement semantics, limits/matrices, shell, forbidden capabilities, preparation, and cleanup. The exact rules are: root `contents: read`; the event mapping is exactly a push to the contract's protected default branch; `pull_request`, `pull_request_target`, `workflow_dispatch`, `schedule`, `workflow_call`, and every other event are forbidden in this persistent self-hosted workflow; exact self-hosted/macOS/architecture/unique-label route; every step `uses` value is an exact complete member of `allowed_actions`; no job-level `uses`, local action, Docker action, mutable ref, or merely well-shaped unknown SHA; checkout clean and credentials false; no workflow-level or job-level `concurrency` key, because GitHub replaces an older pending run even when `cancel-in-progress` is false; positive timeout; serial non-fail-fast matrix; exact system Bash; no environment/write/secret/comment/status/deployment/cache/upload/rerun; exact early storage preflight plus prepare with contract byte floor and allowed leg; exact `$GITHUB_WORKSPACE` equality to the derived contract workspace before cleanup; final `if: always()` cleanup. One uniquely labelled persistent runner supplies serial execution without deleting pending runs.

Add one mutation per rule. Event mutations add each forbidden PR/manual/scheduled/reusable event, combine it with push, change the protected branch, or remove push; every case fails before a job can select the unique self-hosted label. Concurrency mutations add a workflow or job group with either `cancel-in-progress` value and prove rejection, including the superficially “non-cancelling” form. Action tests separately mutate owner, repository, SHA, case, local path, Docker scheme, and reusable-workflow job and prove rejection even when the replacement is 40 lowercase hex. Diagnostics expose stable code and structural YAML path only, never values, expressions, commands, credentials, or contents.

Implement the exact standalone `main()` contract above. Convert parser/input failures into closed stable codes, never exception strings. Unit tests cover success, every exit class, too-large input, symlink/nonregular workflow input, canonical binding rejection, deterministic sorted codes, and content-free stdout/stderr.

- [ ] **Step 5: Prove safe construction and commit**

```bash
! rg -n 'yaml\.load\(|UnsafeLoader|FullLoader|object/apply|object/new' \
  src/runner_guard/policy.py tests/test_workflow_policy.py
PYTHONPATH="$PWD/src" .venv/bin/python -m unittest tests.test_workflow_policy -v
git add src/runner_guard/policy.py tests/test_workflow_policy.py
git commit -m "feat: enforce bounded workflow policy"
```

Expected: routing, mutation, and bomb tests pass.

### Task 4: Build the Exact Eleven-Component Renderer

**Files:**
- Create: `src/runner_guard/bindings.py`
- Create: `src/runner_guard/render.py`
- Create: `templates/rendered/workflow-guard.yml`
- Create: `templates/rendered/ADOPTION_CHECKLIST.md`
- Create: `templates/rendered/AGENT_PROMPT.md`
- Create: `tests/test_render.py`
- Create: `tests/test_bindings.py`
- Create: `tests/fixtures/components/*`

**Interfaces:**
- Consumes: `InstanceContract`, exactly eleven `ComponentPayload` records, exactly three already-opened immutable `TemplatePayload` records, and the exact already-opened reviewed `policy.py` bytes supplied explicitly as `policy_source`.
- Produces: shared lifecycle-neutral `BindingRecord(path, mode, byte_count, sha256)`, `build_binding_manifest(schema_version, records) -> bytes`, and `binding_manifest_sha256(...)`; `ComponentPayload`; `TemplatePayload`; `RenderedFile`; exact `render_instance(contract, components, templates, policy_source)`; `build_rendered_payload_binding(files) -> bytes`; `rendered_payload_binding_sha256(files) -> str`; separate `write_rendered_tree(output_dir, files)`.

```text
activate.py
cleanup_controller.py
job-started.sh
job-completed.sh
transition_controller.py
install-instance.sh
recover-instance.sh
rollback-instance.sh
remove-instance.sh
restore-instance.sh
residue-audit.py
```

- [ ] **Step 1: Write order, embedding, architecture, and output tests**

```python
class RenderTests(unittest.TestCase):
    def test_order_cannot_change_output(self) -> None:
        contract = load_contract_bytes(encoded(valid_contract()))
        policy_source = reviewed_policy_bytes()
        self.assertEqual(render_instance(contract, components(), templates(),
                                         policy_source),
                         render_instance(contract, tuple(reversed(components())),
                                         tuple(reversed(templates())), policy_source))

    def test_architecture_is_exactly_rendered(self) -> None:
        for architecture in ("ARM64", "X64"):
            contract = load_contract_bytes(encoded(valid_contract(architecture)))
            files = render_instance(
                contract,
                components(),
                templates(),
                reviewed_policy_bytes(),
            )
            workflow = next(item.content for item in files
                            if item.relative_path == "workflow-guard.yml")
            self.assertIn(architecture.encode("ascii"), workflow)

    def test_existing_output_is_rejected(self) -> None:
        with tempfile.TemporaryDirectory() as parent:
            output = Path(parent) / "instance"
            output.mkdir()
            with self.assertRaisesRegex(FileExistsError, "render output already exists"):
                write_rendered_tree(output, ())
```

- [ ] **Step 2: Run and confirm missing renderer**

Run: `PYTHONPATH="$PWD/src" .venv/bin/python -m unittest tests.test_render -v`

Expected: FAIL because `runner_guard.render` does not exist.

- [ ] **Step 3: Implement records and exact component checks**

```python
REQUIRED_COMPONENTS = frozenset({
    "activate.py", "cleanup_controller.py", "job-started.sh", "job-completed.sh",
    "transition_controller.py",
    "install-instance.sh", "recover-instance.sh", "rollback-instance.sh",
    "remove-instance.sh", "restore-instance.sh", "residue-audit.py",
})

REQUIRED_COMPONENT_MODES = {
    name: 0o555 for name in REQUIRED_COMPONENTS
}

RENDERED_OUTPUT_MODES = {
    "instance.json": 0o444,
    **REQUIRED_COMPONENT_MODES,
    "workflow-guard.yml": 0o444,
    "workflow-policy.py": 0o555,
    "ADOPTION_CHECKLIST.md": 0o444,
    "AGENT_PROMPT.md": 0o444,
}


@dataclass(frozen=True, order=True)
class ComponentPayload:
    relative_path: str
    mode: int
    content: bytes


@dataclass(frozen=True, order=True)
class RenderedFile:
    relative_path: str
    mode: int
    content: bytes

TemplatePayload = ComponentPayload
```

Reject missing/extra/duplicate files, manifest/checksum control names, absolute/traversal/backslash/control/non-ASCII paths, normalization/casefold aliases, and every component whose mode differs from the exact `REQUIRED_COMPONENT_MODES` entry. The complete result must have exactly the 16 paths and modes in `RENDERED_OUTPUT_MODES`; no source-mode allowance applies to rendered instance payloads.

- [ ] **Step 4: Create exact-marker templates**

The exact three rendered-output templates live in the dedicated `templates/rendered/` directory; helper/component templates live elsewhere and are never discovered through this loader. Workflow markers occur once each: `@@DEFAULT_BRANCH_YAML@@`, `@@RUNNER_LABEL_YAML@@`, `@@ARCHITECTURE_YAML@@`, `@@PROJECT_SLUG_YAML@@`, `@@MINIMUM_FREE_BYTES@@`, `@@TEMP_ROOT_YAML@@`, `@@WORKSPACE_ROOT_YAML@@`, `@@JOB_PREFIX_YAML@@`, and `@@ALLOWED_LEGS_JSON@@`. Every string marker is replaced only by `json.dumps(value, ensure_ascii=True, separators=(",", ":"))` bytes, which are valid deterministic quoted YAML 1.2 scalars; the positive integer marker is rendered as canonical ASCII decimal and allowed legs as canonical JSON. No unquoted caller string is substituted. Checklist/agent markers use the same deterministic JSON-quoted or escaped-inline contract and never raw replacement. Tests parse the rendered YAML and assert exact scalar values as well as exact bytes.

Component substitution is a separate, closed, topologically ordered operation. `COMPONENT_MARKERS` pins the exact marker set permitted in each of the eleven component inputs. Child plan 1 owns only the lifecycle-neutral canonical `BindingRecord`/manifest primitive; child plan 3 later imports the same primitive rather than defining a competing manifest serializer. The renderer first renders contract-derived static paths in the cleanup, standalone transition-controller fixture, and non-install wrapper components; then uses `build_binding_manifest("macos-runner-guard.generation.v1", ...)` for the exact initial cleanup-controller/instance records; then uses `build_binding_manifest("macos-runner-guard.installation-input.v1", ...)` for the lifecycle plan's exact eleven-member installation-input inventory; and finally renders `install-instance.sh` with those two non-self-referential digests plus the activation-file digests. After all 16 final `RenderedFile` bytes are retained, `build_rendered_payload_binding()` maps every file to one bytewise-ordered `BindingRecord` and serializes `macos-runner-guard.rendered-payload.v1`; its SHA-256 is the non-circular source binding used by the artifact plan. This binding is not a member of the rendered tree and cannot hash itself. The generic builder accepts only these three frozen schema-version strings, bytewise-sorted unique ASCII paths, exact regular-file modes, nonnegative bounded counts, and lowercase SHA-256; it has no filesystem or lifecycle dependency. Host inventory/evidence never participates in rendering. A marker may occur only in its declared component, exactly once unless its frozen record says otherwise. Reject a missing, repeated, unexpected, residual, cyclic, or self-digest marker. Tests pin complete canonical bytes, the per-file marker map, every derived digest input, the 16-record rendered binding, and lifecycle reuse of this one serializer.

- [ ] **Step 5: Implement exact pure rendering after policy exists**

```python
def render_instance(contract: InstanceContract,
                    components: Sequence[ComponentPayload],
                    templates: Sequence[TemplatePayload],
                    policy_source: bytes) -> tuple[RenderedFile, ...]:
    contract_bytes = canonical_contract_bytes(contract)
    binding = b"INSTANCE_CONTRACT_JSON = None"
    if policy_source.count(binding) != 1:
        raise RenderError("policy_binding_count_invalid")
    replacement = (
        b'INSTANCE_CONTRACT_JSON = bytes.fromhex("'
        + contract_bytes.hex().encode("ascii")
        + b'")'
    )
    validated = _validate_components(components)
    rendered_files = list(_render_components(contract, validated, contract_bytes))
    rendered_files.append(RenderedFile("instance.json", 0o444, contract_bytes))
    rendered_files.extend(_render_templates(contract, templates, policy_source))
    return tuple(sorted(rendered_files,
                        key=lambda item: item.relative_path.encode("ascii")))
```

Implement `_validate_components(components) -> tuple[ComponentPayload, ...]` with Step 3's exact eleven-file and exact-mode rules. Implement `_render_components(contract, components, contract_bytes) -> tuple[RenderedFile, ...]` with the frozen topological marker and digest rules above. Implement `_render_templates(contract, templates, policy_source) -> tuple[RenderedFile, ...]` with the rendered-template marker rules; it accepts exactly `workflow-guard.yml`, `ADOPTION_CHECKLIST.md`, and `AGENT_PROMPT.md` as immutable bytes and emits those plus `workflow-policy.py`, replacing the one binding with the deterministic `bytes.fromhex()` assignment above. Implement the rendered-payload binding only from that retained final tuple. `RenderReceipt` has exactly `schema_version="macos-runner-guard.render-receipt.v1"`, `member_count=16`, and `rendered_payload_binding_sha256`; `canonical_render_receipt_bytes()` emits sorted-key compact ASCII JSON plus one LF, and the strict loader rejects noncanonical bytes, unknown/missing fields, a count other than 16, or a malformed digest. The public render CLI writes exactly those bytes to stdout only after the absent output tree is fully synced and reread; it emits no path. Review records both SHA-256 of the canonical receipt bytes and its distinct `rendered_payload_binding_sha256` field. The later instance builder accepts only that field as `--expected-rendered-binding-sha256` and independently recomputes it from the descriptor-collected 16 files; it never substitutes the receipt-byte digest. Reject a non-`bytes` policy source, a missing/repeated binding, a missing/extra/duplicate template, or any output inventory/mode mismatch. Require no `@@...@@` token anywhere in final bytes. Root manifest/checksum files remain absent.

`render.py` also owns `merge_component_payloads(base, replacements)`. It indexes both immutable sequences by exact ASCII `relative_path`, rejects a duplicate in either sequence, requires every replacement path to exist in the base inventory, substitutes exactly those records, preserves the base cardinality and closed required path set, and returns the result sorted by ASCII path bytes. It accepts no output path and performs no I/O. Child plans 2 and 3 import this owner function rather than defining local merge behavior. Tests reject a new path, missing base component, duplicate, wrong mode, and count drift; reversed input order produces identical bytes.

- [ ] **Step 6: Implement writer separately**

Create output mode 0700 with `exist_ok=False`. Beneath it use descriptor-relative `O_CREAT | O_EXCL | O_WRONLY`, `O_NOFOLLOW` where supported, then call `os.fchmod(fd, expected_mode)` before sync so the process umask cannot alter the frozen mode. Verify the mode through `fstat`, sync files and directories, preserve incomplete output on failure, and never overlay or retry. Tests run under at least two different umasks and require byte- and mode-identical output.

- [ ] **Step 7: Add rejection/race tests and commit**

Test every component/path/exact-mode/marker rejection, existing output, file precreation race, ARM64/X64 rendering, policy embedding, explicit policy-source mutation, reverse-order determinism, UTF-8 contract bytes, hostile YAML-scalar candidates, the exact 16-path/mode result, and independent recomputation of the exact rendered-payload binding. Mutate one byte/mode/path, forge a receipt digest, or pass a binding from another rendered instance and prove rejection. Add barrier-controlled swaps of the contract, peer contract, workflow, policy file, component root/member, and template root/member after input opening; rendering and checking must consume only the retained verified bytes, or fail before output, never use replacement bytes. Compile the generated `workflow-policy.py` and the child-plan-1 inert `transition_controller.py` fixture with `py_compile`; execute only `workflow-policy.py` here against bounded passing and rejected fixture inputs. Child plan 3 replaces the inert transition fixture with the standalone lifecycle bundle and owns its isolated execution proof, avoiding a dependency on code that does not yet exist.

```bash
PYTHONPATH="$PWD/src" .venv/bin/python -m unittest tests.test_render tests.test_workflow_policy -v
git add src/runner_guard/bindings.py src/runner_guard/render.py templates \
  tests/test_bindings.py tests/test_render.py tests/fixtures/components
git commit -m "feat: render deterministic instance payloads"
```

The API-level tests write the rendered tuple with `write_rendered_tree()`, compile both generated Python files in a disposable root, and execute only `workflow-policy.py` against bounded fixtures. The public `runner_guard render` command does not exist until Task 5, so Task 4 must not invoke it.

### Task 5: Add CLI, Examples, and Exit Codes

**Files:**
- Create: `src/runner_guard/cli.py`
- Create: `src/runner_guard/__main__.py`
- Create: `scripts/render-instance.py`
- Create: `examples/dedicated-account.json`
- Create: `examples/shared-account-trusted-only.json`
- Create: `tests/test_cli.py`

**Interfaces:**
- Consumes: contract/peer/workflow paths, one component directory, one dedicated three-file rendered-template directory, one explicit policy-source file, and one absent output path.
- Produces: `main(argv=None) -> int`; exit 0 pass, 2 validated failure, 3 internal failure.

- [ ] **Step 1: Write failing CLI tests**

```python
class CliTests(unittest.TestCase):
    def test_validate_success_receipt(self) -> None:
        stdout = io.StringIO()
        with contextlib.redirect_stdout(stdout):
            result = main(["validate-contract", "examples/dedicated-account.json"])
        self.assertEqual(result, 0)
        self.assertEqual(stdout.getvalue(), "CONTRACT_VALIDATION=pass\n")

    def test_existing_output_returns_two(self) -> None:
        with temporary_existing_directory() as output:
            result = main(["render", "--contract", "examples/dedicated-account.json",
                           "--components", "tests/fixtures/components",
                           "--templates", "templates/rendered",
                           "--policy-source", "src/runner_guard/policy.py",
                           "--output", str(output)])
        self.assertEqual(result, 2)
```

- [ ] **Step 2: Run and confirm missing CLI**

Run: `PYTHONPATH="$PWD/src" .venv/bin/python -m unittest tests.test_cli -v`

Expected: FAIL because `runner_guard.cli` does not exist.

- [ ] **Step 3: Implement only three commands**

```text
validate-contract CONTRACT [--peer-contract CONTRACT]
render --contract CONTRACT --components DIRECTORY --templates DIRECTORY --policy-source FILE --output DIRECTORY
check-workflow --contract CONTRACT WORKFLOW
```

The CLI opens contract, peer contract, workflow, policy source, component directory, and the mandatory `--templates` directory through bounded descriptor-relative/no-follow readers. `load_component_payloads()` and `load_template_payloads()` retain the root descriptors while inventorying, open each member with `O_NOFOLLOW`, read bounded bytes, and revalidate each identity and the closed directory inventory before returning immutable records. It validates and retains every exact byte before pure logic or output creation. The component directory contains exactly the eleven Task 4 regular, single-link, non-symlink files at mode `0555`; the dedicated rendered-template directory contains exactly the three reviewed regular, single-link templates and no extra entry. Cleanup/lifecycle helper templates are component inputs and cannot appear in this directory. `--policy-source` supplies one explicit bounded regular, single-link file whose retained bytes are passed directly to `render_instance()`; no validator or renderer reopens an input pathname or discovers bytes from package/current-directory state. On render success stdout is exactly the canonical `RenderReceipt` line; failure never emits a success receipt. Tests pin the complete success bytes, independently parse them, and reject changed member count/digest/schema/whitespace plus a receipt from another rendered tree. Do not accept deletion targets, credentials, service actions, host mutations, or overwrite flags. `scripts/render-instance.py` delegates only rendering to CLI logic; contract validation and workflow checking use `python -m runner_guard`. Lifecycle host verification owns its own script outside this child.

- [ ] **Step 4: Add examples for both architectures**

Dedicated example uses ARM64. Acknowledged shared example uses X64, `acme-docs`, account `trustedcirunner`, and disjoint identities/roots. Numeric values are labeled sample measurements in docs, never defaults.

- [ ] **Step 5: Run and commit**

```bash
PYTHONPATH="$PWD/src" .venv/bin/python -m runner_guard validate-contract examples/dedicated-account.json
PYTHONPATH="$PWD/src" .venv/bin/python -m runner_guard validate-contract examples/shared-account-trusted-only.json
policy_test_dir="$(mktemp -d)"
PYTHONPATH="$PWD/src" .venv/bin/python -m runner_guard render \
  --contract examples/dedicated-account.json \
  --components tests/fixtures/components \
  --templates templates/rendered \
  --policy-source src/runner_guard/policy.py \
  --output "$policy_test_dir/rendered"
PYTHONPYCACHEPREFIX="$policy_test_dir/pycache" \
  PYTHONPATH="$PWD/src" .venv/bin/python -m py_compile \
    "$policy_test_dir/rendered/workflow-policy.py" \
    "$policy_test_dir/rendered/transition_controller.py"
PYTHONPATH="$PWD/src" .venv/bin/python "$policy_test_dir/rendered/workflow-policy.py" \
  "$policy_test_dir/rendered/workflow-guard.yml"
PYTHONPATH="$PWD/src" .venv/bin/python -m unittest tests.test_cli -v
git add src/runner_guard/cli.py src/runner_guard/__main__.py scripts examples tests/test_cli.py
git commit -m "feat: expose contract rendering commands"
```

### Task 6: Add Hosted CI, Documentation, and Final Gates

**Files:**
- Create: `.github/workflows/ci.yml`
- Create: `.github/pull_request_template.md`
- Create: `scripts/check-committed-diff.py`
- Create: `tests/test_ci_diff.py`
- Modify: `tests/test_workflow_policy.py`
- Modify: `tests/test_requirements_policy.py`
- Create: `docs/CONTRACT.md`
- Create: `docs/WORKFLOW_POLICY.md`

**Interfaces:**
- Consumes: committed source, exact lock, examples, templates, and tests.
- Produces: Ubuntu Python 3.9-3.14 and macOS Python 3.11/3.14 checks; an event-bound committed-diff verifier; public docs.

- [ ] **Step 1: Write failing repository-CI test**

```python
def test_repository_ci_is_read_only_and_action_pinned(self) -> None:
    document = load_workflow_bytes(Path(".github/workflows/ci.yml").read_bytes())
    self.assertEqual(document["permissions"], {"contents": "read"})
    for value in collect_uses_values(document):
        self.assertRegex(value, r"^[^@]+@[0-9a-f]{40}$")

def test_pull_request_template_requires_review_evidence_and_safe_scope(self) -> None:
    template = Path(".github/pull_request_template.md").read_text("utf-8")
    for heading in ("## Architecture impact", "## Test evidence",
                    "## Scope exclusions"):
        self.assertIn(heading, template)
    self.assertIn("no credentials, tokens, private keys, or live host data", template)
    self.assertEqual(
        {path.as_posix() for path in Path(".github").rglob("*") if path.is_file()},
        {".github/workflows/ci.yml", ".github/pull_request_template.md"},
    )

def test_all_network_package_installs_are_hash_and_wheel_bound(self) -> None:
    assert_all_locks_match_reviewed_pypi_metadata(
        Path("requirements"),
        Path("requirements/metadata/pypi-wheel-metadata.json"),
        supported_minors=("39", "310", "311", "312", "313", "314"),
    )
    violations = find_unhashed_network_installs(repository_text_files())
    self.assertEqual(violations, ())
```

- [ ] **Step 2: Run and confirm CI is absent**

Run: `PYTHONPATH="$PWD/src" .venv/bin/python -m unittest tests.test_workflow_policy tests.test_requirements_policy tests.test_ci_diff -v`

Expected: FAIL with `FileNotFoundError`.

- [ ] **Step 3: Add exact hosted workflow**

```text
actions/checkout@fbc6f3992d24b796d5a048ff273f7fcc4a7b6c09
actions/setup-python@ece7cb06caefa5fff74198d8649806c4678c61a1
```

Use `contents: read`, push/PR main, no workflow/job concurrency replacement key, no secrets/cache/artifact upload, Ubuntu `[3.9,3.10,3.11,3.12,3.13,3.14]`, macOS `[3.11,3.14]`, 20-minute timeouts, and `fail-fast: false`. Hosted jobs may run concurrently; preserving every pending run is more important than artificial serialization. Checkout freezes `persist-credentials: false`, `clean: true`, `fetch-depth: 0`, and exact `ref: ${{ github.event.pull_request.head.sha || github.sha }}`. Workflow-policy mutation tests reject omission or weakening of any of those four controls. The workflow passes the event's exact PR base/head or push before/after object IDs to the verifier; it never substitutes the local working tree or guesses a parent.

`scripts/check-committed-diff.py --event <pull_request|push> --base-sha SHA --head-sha SHA` uses only stdlib plus the reviewed system Git. It detects the repository object format, requires canonical full lowercase object IDs of that format, requires `HEAD` equal `--head-sha`, and proves both objects are commits. For a pull request it requires one merge base and runs `git diff --check <base-sha>...<head-sha>` so a branch behind current base is still checked from the real fork point. For a push it rejects an all-zero/deletion/missing `before`, requires `base-sha` be an ancestor of `head-sha`, and runs `git diff --check <base-sha>..<head-sha>`. Missing/shallow/unrelated objects, malformed IDs, unexpected event, changed HEAD, multiple/no merge base, or Git failure is nonzero. `tests/test_ci_diff.py` creates disposable repositories and proves committed whitespace fails for both PR and push while clean ranges pass; it also covers a behind-base PR, all-zero push, missing/shallow object, unrelated histories, non-ancestor push, malformed SHA, and wrong checkout head. Workflow-policy tests require the exact full-history/ref/event-field wiring and reject bare `git diff --check`.

```bash
python_minor="$(python -c 'import sys; print(f"{sys.version_info.major}{sys.version_info.minor}")')"
python -m pip install --require-hashes --only-binary=:all: \
  -r "requirements/validation-py${python_minor}.txt"
export PYTHONPATH="$GITHUB_WORKSPACE/src"
python -m compileall src tests scripts -q
python -m unittest discover -s tests -v
python -m pytest -q
python -m runner_guard validate-contract examples/dedicated-account.json \
  --peer-contract examples/shared-account-trusted-only.json
python scripts/check-committed-diff.py \
  --event "$GITHUB_EVENT_NAME" \
  --base-sha "$RUNNER_GUARD_EVENT_BASE_SHA" \
  --head-sha "$RUNNER_GUARD_EVENT_HEAD_SHA"
test -z "$(git status --porcelain=v1 --untracked-files=all)"
```

- [ ] **Step 4: Document exact behavior**

Create `.github/pull_request_template.md` with these exact required sections and safety acknowledgement:

```markdown
## Architecture impact

<!-- State which approved architecture section this change implements and whether any contract changes. -->

## Test evidence

<!-- List exact commands, outcomes, commit, and hosted check URLs. -->

## Scope exclusions

- [ ] This PR contains no credentials, tokens, private keys, or live host data.
- [ ] This PR does not operate a live runner or claim host qualification.
- [ ] Deferred subsystems remain excluded or are explicitly identified.
```

`docs/CONTRACT.md` lists every field, pattern, architecture, profile condition, path relation, and required measured limit. `docs/WORKFLOW_POLICY.md` lists stable codes, token/compose/load bounds, bomb rejection, PyYAML confinement, the protected-default-branch push-only self-hosted boundary, the separate hosted PR-validation requirement, and trust limits. Neither claims host qualification.

- [ ] **Step 5: Run complete validation and determinism**

```bash
test -x .venv/bin/python
validation_minor="$(PYTHONPATH="$PWD/src" .venv/bin/python -c 'import sys; print(f"{sys.version_info.major}{sys.version_info.minor}")')"
.venv/bin/pip install --require-hashes --only-binary=:all: \
  -r "requirements/validation-py${validation_minor}.txt"
export PYTHONPATH="$PWD/src"
PYTHONPATH="$PWD/src" .venv/bin/python -m compileall src tests scripts -q
PYTHONPATH="$PWD/src" .venv/bin/python -m unittest discover -s tests -v
PYTHONPATH="$PWD/src" .venv/bin/python -m pytest -q
PYTHONPATH="$PWD/src" .venv/bin/python -m runner_guard validate-contract \
  examples/dedicated-account.json --peer-contract examples/shared-account-trusted-only.json
git diff --check
test -z "$(git status --porcelain=v1 --untracked-files=all)"
```

Render identical inputs into two fresh roots and require `diff -ru` to emit nothing. Record sorted hashes as test evidence only; never commit root manifest/checksum files.

- [ ] **Step 6: Run interface, dependency, placeholder, and scope scans**

```bash
rg -n 'def (load_contract|load_contract_bytes|canonical_contract_bytes|validate_run_identity_fields|validate_job_identity_fields|check_workflow|render_instance|write_rendered_tree)\b|class PolicyViolation\b' src tests docs
old_job_api='validate_job_''identity\b|class Job''Identity\b'
old_job_api_probe="$(printf '%s\n' 'validate_job_''identity(value)' 'class Job''Identity:')"
test "$(printf '%s\n' "$old_job_api_probe" | rg -c "$old_job_api")" -eq 2
test -f src/runner_guard/contract.py
test -f tests/test_contract.py
test -f docs/CONTRACT.md
! rg -n "$old_job_api" src/runner_guard/contract.py tests/test_contract.py docs/CONTRACT.md
! rg -n 'yaml\.load\(|UnsafeLoader|FullLoader' src tests docs requirements pyproject.toml
placeholder_pattern='T''BD|T''ODO|F''IXME|implement ''later|fill in ''details|similar to ''Task|add ''appropriate'
placeholder_probe="$(printf '%s\n' 'T''BD' 'T''ODO' 'F''IXME' 'implement ''later' 'fill in ''details' 'similar to ''Task' 'add ''appropriate')"
test "$(printf '%s\n' "$placeholder_probe" | rg -c "$placeholder_pattern")" -eq 7
! rg -n "$placeholder_pattern" src tests docs requirements pyproject.toml
! rg -n 'registration[_ -]?token|PRIVATE KEY|gh auth|launchctl|svc\.sh|config\.sh|sudo |pkill|killall|rm -rf' src templates examples scripts tests
PYTHONPATH="$PWD/src" .venv/bin/python -m unittest tests.test_requirements_policy -v
test ! -e MANIFEST.json
test ! -e SHA256SUMS
```

Expected: required interfaces appear; forbidden names and scopes are absent; the syntax-aware repository requirements test accepts the documented multiline hash-bound installs and rejects every mutated unhashed, sdist-permitting, or bare network install.

- [ ] **Step 7: Commit and request exact-head review**

```bash
git add .github/workflows/ci.yml .github/pull_request_template.md \
  scripts/check-committed-diff.py tests/test_ci_diff.py \
  tests/test_workflow_policy.py tests/test_requirements_policy.py \
  docs/CONTRACT.md docs/WORKFLOW_POLICY.md
git commit -m "ci: validate contract core across supported Python versions"
```

Request review of schema/parser parity, interfaces, ARM64/X64 routing, contract separation, job-field ownership, YAML bomb defenses, policy mutations, task order, renderer fail-if-exists behavior, dependency hashes, exact Action identities, matrices, and absence of host mutation. Do not merge or release from this child alone.

## Spec Coverage Matrix

| Requirement | Task |
|---|---|
| Exact versions, UTF-8 canonical JSON, bootstrap and validation hash locks | 1, 6 |
| Public contract loaders/serializer and v1 schema | 2 |
| Required cleanup limits and no default disk floor | 2 |
| ARM64 and X64 contract/routing support | 2, 3, 4, 5 |
| Contract-owned field validation, no `JobIdentity` | 2 |
| Cross-instance/account separation | 2 |
| Bounded token scan, compose walk, safe load, bomb tests | 3 |
| Workflow policy implemented before rendering | 3 |
| Exact eleven-component deterministic renderer | 4 |
| Fail-if-exists writer and control-file exclusion | 4 |
| CLI and examples | 5 |
| Ubuntu 3.9-3.14 and macOS 3.11/3.14 | 6 |
| No live host mutation or qualification claim | Global Constraints, 6 |

## Explicitly Deferred Subsystems

- Real macOS descriptor/no-follow capability probing.
- Cleanup environment binding, `RunIdentity`, `JobIdentity`, traversal/deletion, and job hooks.
- Activation, immutable generations, transition, recovery, rollback, removal, and restoration.
- Host inventory, ACL/ownership checks, queues, unrelated-runner comparison, registration, and service lifecycle.
- ZIP32 STORE archives, extraction, staged `MANIFEST.json`, staged `SHA256SUMS`, SBOM, signatures, attestations, packages, and releases.

These are subsystem boundaries, not implemented capabilities.

## Acceptance Gate

- [ ] Exact contract/schema/parser parity, conservative string grammar, path derivation, and cross-contract separation tests pass.
- [ ] The standalone policy rejects every forbidden event, capability, Action identity, and concurrency-replacement mutation.
- [ ] The renderer emits exactly 16 deterministic portable members and performs no host read or mutation.
- [ ] Hosted validation passes on the frozen Ubuntu/macOS Python matrix with hash-locked dependencies.
- [ ] A fresh exact-head review reports zero actionable contract/render/policy findings.

Required endpoint:

```text
CONTRACT_RENDER_POLICY_ACCEPTED
```
