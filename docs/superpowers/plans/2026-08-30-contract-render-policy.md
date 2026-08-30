# Contract, Renderer, and Workflow Policy Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the host-independent contract, workflow-policy, and deterministic renderer core for macOS Runner Guard without touching a live runner or host configuration.

**Architecture:** A strict JSON instance contract is the only input to pure validation, policy evaluation, and rendering. Workflow policy is implemented before rendering because each bundle embeds that reviewed policy source. Cleanup owns environment binding and `JobIdentity`; this child validates only the four untrusted job-identity fields.

**Tech Stack:** CPython 3.9 or newer, Python standard library, JSON Schema 2020-12, PyYAML 6.0.3 only for workflow parsing, `unittest`, GitHub-hosted Ubuntu and macOS runners.

**Spec:** [Approved architecture](../../ARCHITECTURE.md)

## Global Constraints

- Toolkit and package version is exactly `0.1.0`; Python requirement is `>=3.9`.
- Contract schema version is exactly `macos-runner-guard.instance.v1`; schema identifier is exactly `urn:macos-runner-guard:schema:instance:v1`.
- Contract JSON is duplicate-key-free UTF-8 and canonicalizes with sorted keys, compact separators, `ensure_ascii=False`, and one final newline.
- All four cleanup limits and `minimum_free_bytes` are required positive JSON integers. No universal default, fallback, or inherited value exists.
- Job fields full-match run ID `[1-9][0-9]{0,19}`, run attempt `[1-9][0-9]{0,9}`, job `[A-Za-z0-9_.-]{1,64}`, and leg `[A-Za-z0-9_.-]{1,32}`; leg is an exact `allowed_legs` member.
- Supported architecture values are exactly `ARM64` and `X64`.
- PyYAML is exactly 6.0.3, only the host-independent policy CLI imports it, and construction uses `yaml.safe_load` only.
- Validation requirements hash-lock exactly PyYAML 6.0.3, pytest 8.4.2, and setuptools 82.0.1. Pytest 8.4.2 and setuptools 82.0.1 are the newest selected lines that retain Python 3.9 support; pytest 9.1.1 and setuptools 83/84 require Python 3.10 or newer and are forbidden while Python 3.9 remains supported.
- Every package installation after lock creation uses `--no-build-isolation`; validation dependencies use `--require-hashes --only-binary=:all:`.
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
load_contract_bytes(data: bytes) -> InstanceContract
canonical_contract_bytes(contract: InstanceContract) -> bytes
validate_job_identity_fields(contract: InstanceContract, run_id: object,
                             run_attempt: object, github_job: object,
                             leg: object) -> tuple[str, str, str, str]
PolicyViolation(code: str, path: str, message: str)
check_workflow(document: Mapping[str, object],
               contract: Mapping[str, object]) -> tuple[PolicyViolation, ...]
render_instance(contract: InstanceContract,
                components: Sequence[ComponentPayload],
                template_root: Path,
                policy_source: bytes) -> tuple[RenderedFile, ...]
write_rendered_tree(output_dir: Path,
                    files: Sequence[RenderedFile]) -> None
```

No `JobIdentity` type or environment reader belongs in this child.

## File Map

| Path | Responsibility |
|---|---|
| `pyproject.toml` | Package version, Python floor, policy extra, and CLI entry point. |
| `requirements/validation.in` / `requirements/validation.txt` | Exact inputs and reviewed wheel hashes. |
| `schemas/instance-v1.schema.json` | Portable non-secret contract schema. |
| `src/runner_guard/jsonio.py` | Duplicate-safe JSON and canonical UTF-8 bytes. |
| `src/runner_guard/contract.py` | Contract types, public loaders, serialization, separation, and job fields. |
| `src/runner_guard/policy.py` | Bounded YAML parsing and workflow checks; rendered standalone source. |
| `src/runner_guard/render.py` | Pure rendering and fail-if-exists writer. |
| `src/runner_guard/cli.py` | Public commands and exit codes. |
| `templates/*`, `examples/*`, `tests/*` | Templates, non-secret examples, and verification. |
| `.github/workflows/ci.yml` | Hosted supported-version validation. |

### Task 1: Establish Package, Lock, Errors, and Canonical JSON

**Files:**
- Create: `pyproject.toml`
- Create: `requirements/validation.in`
- Create: `requirements/validation.txt`
- Create: `src/runner_guard/__init__.py`
- Create: `src/runner_guard/errors.py`
- Create: `src/runner_guard/jsonio.py`
- Create: `tests/test_jsonio.py`

**Interfaces:**
- Consumes: UTF-8 JSON bytes.
- Produces: `__version__ == "0.1.0"`; `ValidationIssue`; `GuardError`; `JsonFormatError`; `ContractError`; `load_json_object`; `canonical_json_bytes`.

- [ ] **Step 1: Write failing duplicate and UTF-8 tests**

```python
class JsonIoTests(unittest.TestCase):
    def test_duplicate_key_is_rejected(self) -> None:
        with self.assertRaisesRegex(JsonFormatError, "duplicate JSON key: profile"):
            load_json_object(b'{"profile":"a","profile":"b"}\n')

    def test_canonical_bytes_are_sorted_utf8(self) -> None:
        self.assertEqual(canonical_json_bytes({"z": "\u0627", "a": 1}),
                         '{"a":1,"z":"\u0627"}\n'.encode("utf-8"))
```

- [ ] **Step 2: Run and confirm missing imports**

Run: `PYTHONPATH=src python -m unittest tests.test_jsonio -v`

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

- [ ] **Step 5: Generate and verify the exact hash lock**

`requirements/validation.in` contains exactly:

```text
PyYAML==6.0.3
pytest==8.4.2
setuptools==82.0.1
```

Run: `test ! -e .venv && python3 -m venv .venv && .venv/bin/python -m pip install 'pip-tools==7.6.1'`

Run: `.venv/bin/pip-compile --generate-hashes --allow-unsafe --pip-args='--only-binary=:all:' --output-file=requirements/validation.txt requirements/validation.in`

Expected: the three direct requirements and their exact transitive requirements appear. Compare every retained wheel hash with official PyPI file metadata and require compatible wheels for every supported matrix leg. Record that pytest 8.4.2 and setuptools 82.0.1 preserve Python 3.9; reject pytest 9.1.1 and setuptools 83/84 because they require Python 3.10 or newer.

- [ ] **Step 6: Install, test, and commit**

```bash
python3 -m venv --clear .venv
.venv/bin/pip install --require-hashes --only-binary=:all: -r requirements/validation.txt
.venv/bin/pip install --no-build-isolation --no-deps -e .
.venv/bin/python -m unittest tests.test_jsonio -v
git add pyproject.toml requirements src/runner_guard tests/test_jsonio.py
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
- Produces: `CleanupLimits`; `InstanceLayout`; `InstanceContract` with `repository`; `load_contract`; `load_contract_bytes`; `canonical_contract_bytes`; `validate_contract_set`; `validate_job_identity_fields`.

Require exactly these fields and reject extras:

```text
schema_version, toolkit_version, project_slug, profile,
shared_account_risk_acknowledged, github_owner, github_repository,
default_branch, account_name, account_home, runner_name_prefix,
runner_label, service_name_prefix, instance_root, runner_root,
work_root, temp_root, workspace_root, controller_root, cache_root,
evidence_root, job_prefix, allowed_legs, platform, architecture,
cleanup_limits, minimum_free_bytes, routine_availability,
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
        "service_name_prefix": "actions.runner.acme-widget.",
        "instance_root": root, "runner_root": root + "/runner",
        "work_root": root + "/work", "temp_root": root + "/temp",
        "workspace_root": root + "/work/widget/widget",
        "controller_root": root + "/controller", "cache_root": root + "/cache",
        "evidence_root": root + "/evidence", "job_prefix": "acme-widget-job",
        "allowed_legs": ["main", "3.11", "3.12", "3.13", "3.14"],
        "platform": "macOS", "architecture": architecture,
        "cleanup_limits": {"max_entries": 20000, "max_depth": 32,
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

Run: `PYTHONPATH=src python -m unittest tests.test_contract -v`

Expected: FAIL because `runner_guard.contract` does not exist.

- [ ] **Step 3: Add exact schema and frozen types**

Use draft 2020-12, exact `$id`, `additionalProperties: false`, the complete required/property sets, all four required positive cleanup integers, required positive `minimum_free_bytes`, profile acknowledgement conditions, and Global Constraint patterns. Architecture is:

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
    return load_contract_bytes(path.read_bytes())


def load_contract_bytes(data: bytes) -> InstanceContract:
    return _parse_contract(load_json_object(data))


def canonical_contract_bytes(contract: InstanceContract) -> bytes:
    return canonical_json_bytes(_contract_to_mapping(contract))
```

Validate raw path text before `PurePosixPath`; require leading `/`; reject `//`, `~`, `$`, backslash, controls, `.`, and `..`. Reject invalid repository and branch shapes. Never call `resolve()` or inspect the host. Require exact instance/account roots, direct runner/work/temp/controller/cache/evidence children, and workspace as a strict work-root descendant.

- [ ] **Step 5: Implement job-field validation only**

```python
RUN_ID_RE = re.compile(r"[1-9][0-9]{0,19}\Z", re.ASCII)
RUN_ATTEMPT_RE = re.compile(r"[1-9][0-9]{0,9}\Z", re.ASCII)
JOB_RE = re.compile(r"[A-Za-z0-9_.-]{1,64}\Z", re.ASCII)
LEG_RE = re.compile(r"[A-Za-z0-9_.-]{1,32}\Z", re.ASCII)


def validate_job_identity_fields(contract: InstanceContract, run_id: object,
                                 run_attempt: object, github_job: object,
                                 leg: object) -> tuple[str, str, str, str]:
    values = (run_id, run_attempt, github_job, leg)
    patterns = (RUN_ID_RE, RUN_ATTEMPT_RE, JOB_RE, LEG_RE)
    codes = ("run_id_invalid", "run_attempt_invalid",
             "github_job_invalid", "leg_invalid")
    for value, pattern, code in zip(values, patterns, codes):
        if not isinstance(value, str) or pattern.fullmatch(value) is None:
            raise ContractError(code)
    if leg not in contract.allowed_legs:
        raise ContractError("leg_not_allowed")
    return run_id, run_attempt, github_job, leg
```

Test zero, sign, padding, whitespace, newline, overlength, Unicode, controls, non-string values, and unlisted legs. Do not read environment, create a basename, or define `JobIdentity`.

- [ ] **Step 6: Implement separation and schema parity**

Compare repository, label, runner/service/job prefixes, account, and roots with NFC casefolding. Reject exact/nested cross-contract roots. Reject account reuse when either profile is dedicated; permit only two acknowledged shared profiles with all other identities disjoint. Test every collision class. Load the schema and compare required/property/cleanup sets, constants, enums, patterns, and profile conditions with parser tests.

- [ ] **Step 7: Run and commit**

```bash
PYTHONPATH=src python -m unittest tests.test_contract -v
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

Run: `PYTHONPATH=src python -m unittest tests.test_workflow_policy -v`

Expected: FAIL because `runner_guard.policy` does not exist.

- [ ] **Step 3: Scan tokens before compose and safe construction**

Decode at most 2,000,000 UTF-8 bytes. Iterate `yaml.scan(text)` first with a 100,000-token limit. Reject `AliasToken`, `AnchorToken`, `TagToken`, merge scalar `<<`, nonstandard directives, and scalar values above 1,000,000 characters. No compose/load call occurs before this scan passes.

Then call `yaml.compose(text)` and walk at most 100,000 nodes to depth 64, rejecting duplicate mapping keys, nonstandard tags, and cycles. Only after both gates call `yaml.safe_load(text)`. Normalize root boolean `True` to `on` only if literal `on` is absent.

Add tests for anchors, aliases, merges, custom tags, nested alias bombs, token/node/depth/scalar bounds, duplicate keys, ambiguous `on`, malformed UTF-8, and parser call order. Patch `yaml.compose` and `yaml.safe_load` in forbidden-token tests and assert neither was called.

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
        _check_permissions, _check_events, _check_same_repository_pr,
        _check_routing, _check_actions, _check_checkout,
        _check_concurrency, _check_job_limits, _check_shell,
        _check_forbidden_capabilities, _check_prepare, _check_cleanup,
    )
    return tuple(sorted(issue for check in checks
                        for issue in check(document, contract)))
```

Implement every named `_check_*` helper with signature `(document: Mapping[str, object], contract: Mapping[str, object]) -> tuple[PolicyViolation, ...]`. In listed order they enforce root permissions, events, same-repository PRs, routing, Action identities, checkout, concurrency, limits/matrices, shell, forbidden capabilities, preparation, and cleanup. The exact rules are: root `contents: read`; only push/PR to default branch; same-repository PR condition; exact self-hosted/macOS/architecture/unique-label route; 40-hex Action SHAs; checkout clean and credentials false; per-ref non-cancelling concurrency; positive timeout; serial non-fail-fast matrix; exact system Bash; no environment/write/secret/comment/status/deployment/cache/upload/rerun; exact early prepare with contract byte floor and allowed leg; final `if: always()` cleanup.

Add one mutation per rule. Diagnostics expose stable code and structural YAML path only, never values, expressions, commands, credentials, or contents.

Implement the exact standalone `main()` contract above. Convert parser/input failures into closed stable codes, never exception strings. Unit tests cover success, every exit class, too-large input, symlink/nonregular workflow input, canonical binding rejection, deterministic sorted codes, and content-free stdout/stderr.

- [ ] **Step 5: Prove safe construction and commit**

```bash
! rg -n 'yaml\.load\(|UnsafeLoader|FullLoader|object/apply|object/new' \
  src/runner_guard/policy.py tests/test_workflow_policy.py
PYTHONPATH=src python -m unittest tests.test_workflow_policy -v
git add src/runner_guard/policy.py tests/test_workflow_policy.py
git commit -m "feat: enforce bounded workflow policy"
```

Expected: routing, mutation, and bomb tests pass.

### Task 4: Build the Exact Ten-Component Renderer

**Files:**
- Create: `src/runner_guard/render.py`
- Create: `templates/workflow-guard.yml`
- Create: `templates/ADOPTION_CHECKLIST.md`
- Create: `templates/AGENT_PROMPT.md`
- Create: `tests/test_render.py`
- Create: `tests/fixtures/components/*`

**Interfaces:**
- Consumes: `InstanceContract`, exactly ten `ComponentPayload` records, a template root, and the exact reviewed `policy.py` bytes supplied explicitly as `policy_source`.
- Produces: `ComponentPayload`; `RenderedFile`; exact `render_instance(contract, components, template_root, policy_source)`; separate `write_rendered_tree(output_dir, files)`.

```text
activate.py
cleanup_controller.py
job-started.sh
job-completed.sh
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
        policy_source = Path("src/runner_guard/policy.py").read_bytes()
        self.assertEqual(render_instance(contract, components(), Path("templates"),
                                         policy_source),
                         render_instance(contract, tuple(reversed(components())),
                                         Path("templates"), policy_source))

    def test_architecture_is_exactly_rendered(self) -> None:
        for architecture in ("ARM64", "X64"):
            contract = load_contract_bytes(encoded(valid_contract(architecture)))
            files = render_instance(
                contract,
                components(),
                Path("templates"),
                Path("src/runner_guard/policy.py").read_bytes(),
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

Run: `PYTHONPATH=src python -m unittest tests.test_render -v`

Expected: FAIL because `runner_guard.render` does not exist.

- [ ] **Step 3: Implement records and exact component checks**

```python
REQUIRED_COMPONENTS = frozenset({
    "activate.py", "cleanup_controller.py", "job-started.sh", "job-completed.sh",
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
```

Reject missing/extra/duplicate files, manifest/checksum control names, absolute/traversal/backslash/control/non-ASCII paths, normalization/casefold aliases, and every component whose mode differs from the exact `REQUIRED_COMPONENT_MODES` entry. The complete result must have exactly the 15 paths and modes in `RENDERED_OUTPUT_MODES`; no source-mode allowance applies to rendered instance payloads.

- [ ] **Step 4: Create exact-marker templates**

Workflow markers occur once each: `@@DEFAULT_BRANCH@@`, `@@RUNNER_LABEL@@`, `@@ARCHITECTURE@@`, `@@PROJECT_SLUG@@`, `@@MINIMUM_FREE_BYTES@@`, `@@JOB_PREFIX@@`, `@@ALLOWED_LEGS_JSON@@`. Checklist/agent markers for project, repository, profile, and label occur once each. Reject missing, repeated, or residual markers.

- [ ] **Step 5: Implement exact pure rendering after policy exists**

```python
def render_instance(contract: InstanceContract,
                    components: Sequence[ComponentPayload],
                    template_root: Path,
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
    rendered_files = list(validated)
    rendered_files.append(RenderedFile("instance.json", 0o444, contract_bytes))
    rendered_files.extend(_render_templates(contract, template_root, policy_source))
    return tuple(sorted(rendered_files,
                        key=lambda item: item.relative_path.encode("ascii")))
```

Implement `_validate_components(components) -> tuple[RenderedFile, ...]` with Step 3's exact ten-file and exact-mode rules. Implement `_render_templates(contract, template_root, policy_source) -> tuple[RenderedFile, ...]` with Step 4's marker rules; it emits `workflow-guard.yml` mode `0444`, `workflow-policy.py` mode `0555`, `ADOPTION_CHECKLIST.md` mode `0444`, and `AGENT_PROMPT.md` mode `0444`, replacing the one binding with the deterministic `bytes.fromhex()` assignment above. Reject a non-`bytes` policy source, a missing/repeated binding, and any output inventory or mode mismatch. Root manifest/checksum files remain absent.

- [ ] **Step 6: Implement writer separately**

Create output mode 0700 with `exist_ok=False`. Beneath it use descriptor-relative `O_CREAT | O_EXCL | O_WRONLY`, `O_NOFOLLOW` where supported, exact modes, and file/directory `fsync`. Preserve incomplete output on failure; never overlay or retry.

- [ ] **Step 7: Add rejection/race tests and commit**

Test every component/path/exact-mode/marker rejection, existing output, file precreation race, ARM64/X64 rendering, policy embedding, explicit policy-source mutation, reverse-order determinism, UTF-8 contract bytes, and the exact 15-path/mode result. Compile the generated `workflow-policy.py` with `py_compile`, then execute it in the validation virtual environment against one passing and one mutated workflow; require the exact receipt and exit-code contract with no source-tree import dependency.

```bash
PYTHONPATH=src python -m unittest tests.test_render tests.test_workflow_policy -v
policy_test_dir="$(mktemp -d)"
PYTHONPATH=src .venv/bin/python -m runner_guard render \
  --contract examples/dedicated-account.json \
  --components tests/fixtures/components \
  --policy-source src/runner_guard/policy.py \
  --output "$policy_test_dir/rendered"
PYTHONPYCACHEPREFIX="$policy_test_dir/pycache" \
  .venv/bin/python -m py_compile "$policy_test_dir/rendered/workflow-policy.py"
.venv/bin/python "$policy_test_dir/rendered/workflow-policy.py" \
  "$policy_test_dir/rendered/workflow-guard.yml"
git add src/runner_guard/render.py templates tests/test_render.py tests/fixtures/components
git commit -m "feat: render deterministic instance payloads"
```

### Task 5: Add CLI, Examples, and Exit Codes

**Files:**
- Create: `src/runner_guard/cli.py`
- Create: `src/runner_guard/__main__.py`
- Create: `scripts/render-instance.py`
- Create: `examples/dedicated-account.json`
- Create: `examples/shared-account-trusted-only.json`
- Create: `tests/test_cli.py`

**Interfaces:**
- Consumes: contract/peer/workflow paths, one component directory, one explicit policy-source file, and one absent output path.
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
                           "--policy-source", "src/runner_guard/policy.py",
                           "--output", str(output)])
        self.assertEqual(result, 2)
```

- [ ] **Step 2: Run and confirm missing CLI**

Run: `PYTHONPATH=src python -m unittest tests.test_cli -v`

Expected: FAIL because `runner_guard.cli` does not exist.

- [ ] **Step 3: Implement only three commands**

```text
validate-contract CONTRACT [--peer-contract CONTRACT]
render --contract CONTRACT --components DIRECTORY --policy-source FILE --output DIRECTORY
check-workflow --contract CONTRACT WORKFLOW
```

The component directory contains exactly the ten Task 4 regular, non-symlink files, each at exact mode `0555`, with no extra entry. `--policy-source` must be one explicit regular, non-symlink UTF-8 file whose bytes are passed directly to `render_instance()`; the renderer never discovers it from package or working-directory state. Do not accept deletion targets, credentials, service actions, host mutations, or overwrite flags. `scripts/render-instance.py` delegates only rendering to CLI logic; contract validation and workflow checking use `python -m runner_guard`. Lifecycle host verification owns its own script outside this child.

- [ ] **Step 4: Add examples for both architectures**

Dedicated example uses ARM64. Acknowledged shared example uses X64, `acme-docs`, account `trustedcirunner`, and disjoint identities/roots. Numeric values are labeled sample measurements in docs, never defaults.

- [ ] **Step 5: Run and commit**

```bash
PYTHONPATH=src python -m runner_guard validate-contract examples/dedicated-account.json
PYTHONPATH=src python -m runner_guard validate-contract examples/shared-account-trusted-only.json
PYTHONPATH=src python -m unittest tests.test_cli -v
git add src/runner_guard/cli.py src/runner_guard/__main__.py scripts examples tests/test_cli.py
git commit -m "feat: expose contract rendering commands"
```

### Task 6: Add Hosted CI, Documentation, and Final Gates

**Files:**
- Create: `.github/workflows/ci.yml`
- Create: `.github/pull_request_template.md`
- Modify: `tests/test_workflow_policy.py`
- Create: `docs/CONTRACT.md`
- Create: `docs/WORKFLOW_POLICY.md`

**Interfaces:**
- Consumes: committed source, exact lock, examples, templates, and tests.
- Produces: Ubuntu Python 3.9-3.14 and macOS Python 3.11/3.14 checks; public docs.

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
```

- [ ] **Step 2: Run and confirm CI is absent**

Run: `PYTHONPATH=src python -m unittest tests.test_workflow_policy -v`

Expected: FAIL with `FileNotFoundError`.

- [ ] **Step 3: Add exact hosted workflow**

```text
actions/checkout@fbc6f3992d24b796d5a048ff273f7fcc4a7b6c09
actions/setup-python@ece7cb06caefa5fff74198d8649806c4678c61a1
```

Use `contents: read`, push/PR main, per-ref non-cancelling concurrency, no secrets/cache/artifact upload, Ubuntu `[3.9,3.10,3.11,3.12,3.13,3.14]`, macOS `[3.11,3.14]`, 20-minute timeouts, and `fail-fast: false`.

```bash
python -m pip install --require-hashes --only-binary=:all: -r requirements/validation.txt
python -m pip install --no-build-isolation --no-deps -e .
python -m compileall src tests scripts -q
python -m unittest discover -s tests -v
python -m pytest -q
python -m runner_guard validate-contract examples/dedicated-account.json \
  --peer-contract examples/shared-account-trusted-only.json
git diff --check
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

`docs/CONTRACT.md` lists every field, pattern, architecture, profile condition, path relation, and required measured limit. `docs/WORKFLOW_POLICY.md` lists stable codes, token/compose/load bounds, bomb rejection, PyYAML confinement, same-repository assumption, and trust limits. Neither claims host qualification.

- [ ] **Step 5: Run complete validation and determinism**

```bash
test -x .venv/bin/python
.venv/bin/pip install --require-hashes --only-binary=:all: -r requirements/validation.txt
.venv/bin/pip install --no-build-isolation --no-deps -e .
.venv/bin/python -m compileall src tests scripts -q
.venv/bin/python -m unittest discover -s tests -v
.venv/bin/python -m pytest -q
.venv/bin/python -m runner_guard validate-contract \
  examples/dedicated-account.json --peer-contract examples/shared-account-trusted-only.json
git diff --check
```

Render identical inputs into two fresh roots and require `diff -ru` to emit nothing. Record sorted hashes as test evidence only; never commit root manifest/checksum files.

- [ ] **Step 6: Run interface, dependency, placeholder, and scope scans**

```bash
rg -n 'def (load_contract|load_contract_bytes|canonical_contract_bytes|validate_job_identity_fields|check_workflow|render_instance|write_rendered_tree)\b|class PolicyViolation\b' src tests docs
old_job_api='validate_job_'"'"'identity\\b|class Job'"'"'Identity\\b'
! rg -n "$old_job_api|yaml\.load\(|UnsafeLoader|FullLoader" src tests docs requirements pyproject.toml
placeholder_pattern='T''BD|T''ODO|F''IXME|implement '"'"'later|fill in '"'"'details|similar to '"'"'Task|add '"'"'appropriate'
! rg -n "$placeholder_pattern" src tests docs requirements pyproject.toml
! rg -n 'registration[_ -]?token|PRIVATE KEY|gh auth|launchctl|svc\.sh|config\.sh|sudo |pkill|killall|rm -rf' src templates examples scripts tests
test ! -e MANIFEST.json
test ! -e SHA256SUMS
```

Expected: required interfaces appear; forbidden names, dependencies, and scopes are absent.

- [ ] **Step 7: Commit and request exact-head review**

```bash
git add .github/workflows/ci.yml .github/pull_request_template.md \
  tests/test_workflow_policy.py docs/CONTRACT.md docs/WORKFLOW_POLICY.md
git commit -m "ci: validate contract core across supported Python versions"
```

Request review of schema/parser parity, interfaces, ARM64/X64 routing, contract separation, job-field ownership, YAML bomb defenses, policy mutations, task order, renderer fail-if-exists behavior, dependency hashes, exact Action identities, matrices, and absence of host mutation. Do not merge or release from this child alone.

## Spec Coverage Matrix

| Requirement | Task |
|---|---|
| Exact versions, UTF-8 canonical JSON, hash lock | 1 |
| Public contract loaders/serializer and v1 schema | 2 |
| Required cleanup limits and no default disk floor | 2 |
| ARM64 and X64 contract/routing support | 2, 3, 4, 5 |
| Contract-owned field validation, no `JobIdentity` | 2 |
| Cross-instance/account separation | 2 |
| Bounded token scan, compose walk, safe load, bomb tests | 3 |
| Workflow policy implemented before rendering | 3 |
| Exact ten-component deterministic renderer | 4 |
| Fail-if-exists writer and control-file exclusion | 4 |
| CLI and examples | 5 |
| Ubuntu 3.9-3.14 and macOS 3.11/3.14 | 6 |
| No live host mutation or qualification claim | Global Constraints, 6 |

## Explicitly Deferred Subsystems

- Real macOS descriptor/no-follow capability probing.
- Cleanup environment binding, `JobIdentity`, traversal/deletion, and job hooks.
- Activation, immutable generations, transition, recovery, rollback, removal, and restoration.
- Host inventory, ACL/ownership checks, queues, unrelated-runner comparison, registration, and service lifecycle.
- ZIP32 STORE archives, extraction, staged `MANIFEST.json`, staged `SHA256SUMS`, SBOM, signatures, attestations, packages, and releases.

These are subsystem boundaries, not implemented capabilities.
