# Cleanup Runtime Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the capability-gated, standard-library-only cleanup runtime that safely removes only one macOS Runner Guard instance's bounded job residue before and after GitHub Actions jobs.

**Architecture:** A validated instance contract supplies immutable roots, a job namespace, allowed legs, and cleanup budgets. Stable root-owned `/bin/bash` hooks invoke a generation-bound controller through `/usr/bin/python3 -I -S`; the controller validates canonical job identity, walks directories descriptor-relatively without following links, inventories targets within monotonic budgets, revalidates inode/device identity, and only then deletes. Repository workflow cleanup uses the same controller for the exact job leg, while before/after hooks recover stale or interrupted work without addressing another instance.

**Tech Stack:** macOS, CPython 3.9 or newer at `/usr/bin/python3`, Python standard library only for installed runtime code, `/bin/bash`, JSON Schema draft 2020-12, `unittest`, GitHub Actions YAML

**Spec:** [Approved architecture sections 6.2-6.4, 7, 8, 10, 12, and 13](../../ARCHITECTURE.md)

## Global Constraints

- Installed controller and hook code runs on macOS with CPython 3.9 or newer and uses only the Python standard library.
- Adoption is capability-gated: missing or behaviorally incompatible descriptor-relative/no-follow primitives block host qualification and cleanup.
- Hooks use `/bin/bash` and invoke `/usr/bin/python3 -I -S`; they never use Homebrew Bash or an unqualified Python executable.
- Required `cleanup_limits` fields are `max_entries`, `max_depth`, `max_bytes`, and `max_seconds`; reviewed examples use `200000`, `64`, `53687091200`, and `120`.
- `run_id` full-matches `^[1-9][0-9]{0,19}$`; `run_attempt` full-matches `^[1-9][0-9]{0,9}$`.
- `github_job` full-matches `^[A-Za-z0-9_.-]{1,64}$`.
- `leg` belongs to the contract's non-empty `allowed_legs` and full-matches `^[A-Za-z0-9_.-]{1,32}$`.
- Cleanup operates only below exact contract-bound `temp_root`, `work_root`, and `workspace_root`; it never consumes a deletion target from workflow input.
- Cleanup never follows links, traverses `..`, crosses devices, accepts foreign ownership, removes an unexpected hard link, or falls back to canonical-path recursion.
- Cleanup never uses broad deletion, `pkill`, `killall`, or automatic process killing and never mutates a live host during tests.
- Every rejection test snapshots its complete fixture first and proves the fixture and external link targets remain byte-for-byte unchanged.
- Receipts are bounded, canonical, content-free JSON: no filenames, environment values, file contents, credentials, or payload data.
- Development and unit-test commands use `.venv/bin/python`; `/usr/bin/python3 -I -S` is reserved for rendered standalone-runtime subprocess tests and host capability proof.
- Use `apply_patch` for repository edits. Do not use shell heredocs or shell redirection to create source files.

---

## File Map

- Create `src/runner_guard/capabilities.py`: mandatory macOS descriptor/no-follow probe.
- Create `src/runner_guard/job_identity.py`: canonical GitHub identity parser and basename derivation.
- Create `src/runner_guard/fsops.py`: descriptor ownership, inventory, revalidation, and deletion primitives.
- Create `src/runner_guard/workspace.py`: exact ordinary-worktree cleanup through `/usr/bin/git` without a shell.
- Create `src/runner_guard/cleanup.py`: phase orchestration, monotonic budgets, bounded receipts, and CLI.
- Create `src/runner_guard/residue.py`: read-only residue inventory using the same safe primitives.
- Create `src/runner_guard/bundle_runtime.py`: deterministic single-file standard-library runtime bundler.
- Verify `src/runner_guard/contract.py`: typed cleanup settings on `InstanceContract` remain the child-plan-1 contract.
- Verify `schemas/instance-v1.schema.json`: cleanup limits, job prefix, and enumerated allowed legs remain unchanged.
- Create `templates/job-started.sh` and `templates/job-completed.sh`: stable wrappers.
- Modify `templates/workflow-guard.yml`: exact per-leg job root and `if: always()` cleanup.
- Verify `src/runner_guard/render.py`: retain child-plan-1 renderer interfaces while supplying cleanup components.
- Create `templates/activate.py`: descriptor-bound standalone controller loader used by the protected lifecycle.
- Verify both JSON files under `examples/`: reviewed cleanup values and allowed legs.
- Create `tests/support/tree_snapshot.py`: no-follow fixture snapshot helper.
- Create `tests/test_cleanup_contract.py`, `tests/test_job_identity.py`, `tests/test_capabilities.py`, `tests/test_fsops.py`, `tests/test_workspace.py`, `tests/test_cleanup.py`, `tests/test_residue.py`, `tests/test_hooks.py`, `tests/test_runtime_bundle.py`, `tests/test_cleanup_render.py`, and `tests/test_cleanup_integration.py`.

### Task 1: Verify the Frozen Cleanup Contract and Add Canonical Job Identity

**Files:**
- Verify: `schemas/instance-v1.schema.json`
- Verify: `src/runner_guard/contract.py`
- Create: `src/runner_guard/job_identity.py`
- Verify: `examples/dedicated-account.json`
- Verify: `examples/shared-account-trusted-only.json`
- Create: `tests/test_cleanup_contract.py`
- Create: `tests/test_job_identity.py`

**Interfaces:**
- Consumes: `CleanupLimits`, `InstanceContract`, `load_contract(path: os.PathLike[str]) -> InstanceContract`, `load_contract_bytes(data: bytes) -> InstanceContract`, `canonical_contract_bytes(contract: InstanceContract) -> bytes`, and `validate_job_identity_fields(contract: InstanceContract, run_id: object, run_attempt: object, github_job: object, leg: object) -> tuple[str, str, str, str]` from `runner_guard.contract`; `ContractError` and `JobIdentityError` from `runner_guard.errors`.
- Produces: `JobIdentity.from_environment(contract: InstanceContract, environment: Mapping[str, str]) -> JobIdentity`, `JobIdentity.job_basename() -> str`, and `JobIdentity.run_prefix() -> str`.

- [ ] **Step 1: Pin the already implemented cross-plan cleanup contract**

```python
class CleanupContractTests(unittest.TestCase):
    def test_reviewed_example_limits_are_required(self):
        raw = valid_instance_dict()
        raw["cleanup_limits"] = {
            "max_entries": 200000,
            "max_depth": 64,
            "max_bytes": 53687091200,
            "max_seconds": 120,
        }
        contract = load_contract_bytes(encoded(raw))
        self.assertEqual(contract.cleanup_limits, CleanupLimits(200000, 64, 53687091200, 120))

    def test_missing_cleanup_member_is_rejected(self):
        raw = valid_instance_dict()
        del raw["cleanup_limits"]["max_seconds"]
        with self.assertRaisesRegex(ContractError, "cleanup_limits.max_seconds"):
            load_contract_bytes(encoded(raw))

    def test_legs_are_nonempty_unique_and_canonical(self):
        for legs in ([], ["main", "main"], ["../main"], ["x" * 33]):
            raw = valid_instance_dict()
            raw["allowed_legs"] = legs
            with self.subTest(legs=legs), self.assertRaises(ContractError):
                load_contract_bytes(encoded(raw))
```

- [ ] **Step 2: Run the prerequisite contract assertions**

Run: `.venv/bin/python -m unittest tests.test_cleanup_contract -v`

Expected: PASS because child plan 1 already created the exact fields. Any failure is interface drift: stop and reconcile the prior plan rather than redefining the contract here.

- [ ] **Step 3: Verify the exact schema and typed model without editing them**

Require `cleanup_limits`, `job_prefix`, and `allowed_legs` in the instance `required` array and verify this existing schema shape:

```json
"cleanup_limits": {
  "type": "object",
  "additionalProperties": false,
  "required": ["max_entries", "max_depth", "max_bytes", "max_seconds"],
  "properties": {
    "max_entries": {"type": "integer", "minimum": 1, "maximum": 1000000},
    "max_depth": {"type": "integer", "minimum": 1, "maximum": 128},
    "max_bytes": {"type": "integer", "minimum": 1, "maximum": 1099511627776},
    "max_seconds": {"type": "integer", "minimum": 1, "maximum": 600}
  }
},
"job_prefix": {"type": "string", "pattern": "^[A-Za-z0-9][A-Za-z0-9_.-]{0,31}$"},
"allowed_legs": {
  "type": "array", "minItems": 1, "maxItems": 64, "uniqueItems": true,
  "items": {"type": "string", "pattern": "^[A-Za-z0-9_.-]{1,32}$"}
}
```

```python
@dataclass(frozen=True)
class CleanupLimits:
    max_entries: int
    max_depth: int
    max_bytes: int
    max_seconds: int
```

The existing parser must reject strings and booleans rather than coercing them and preserve allowed legs as a tuple in input order. If these assertions fail, return to child plan 1; do not patch the parser within this task.

- [ ] **Step 4: Run the contract tests**

Run: `.venv/bin/python -m unittest tests.test_cleanup_contract -v`

Expected: PASS.

- [ ] **Step 5: Write failing canonical identity tests**

```python
class JobIdentityTests(unittest.TestCase):
    def test_valid_identity_derives_exact_and_run_prefix_names(self):
        identity = JobIdentity.from_environment(
            contract_with(repository="KhAlbl/macos-runner-guard", job_prefix="guard-job", allowed_legs=("main", "3.14")),
            valid_env(run_id="123456789", attempt="2", job="memory-abi-wheel", leg="3.14"),
        )
        self.assertEqual(identity.job_basename(), "guard-job-123456789-2-memory-abi-wheel-3.14")
        self.assertEqual(identity.run_prefix(), "guard-job-123456789-2-")

    def test_noncanonical_values_are_rejected(self):
        cases = {
            "GITHUB_RUN_ID": ["0", "01", "+1", " 1", "1\n", "1" * 21],
            "GITHUB_RUN_ATTEMPT": ["0", "01", "-1", "1 ", "1\n", "1" * 11],
            "GITHUB_JOB": ["", "a/b", "job name", "x" * 65],
            "RUNNER_GUARD_LEG": ["", "../main", "main/other", "x" * 33],
        }
        for variable, values in cases.items():
            for value in values:
                environment = valid_env(run_id="9", attempt="1", job="build", leg="main")
                environment[variable] = value
                with self.subTest(variable=variable, value=repr(value)), self.assertRaises(JobIdentityError):
                    JobIdentity.from_environment(contract_with(allowed_legs=("main",)), environment)

    def test_well_formed_unlisted_leg_is_rejected(self):
        with self.assertRaisesRegex(JobIdentityError, "leg_not_allowed"):
            JobIdentity.from_environment(contract_with(allowed_legs=("main",)), valid_env(leg="3.14"))
```

- [ ] **Step 6: Run identity tests to prove the gap**

Run: `.venv/bin/python -m unittest tests.test_job_identity -v`

Expected: FAIL because `runner_guard.job_identity` does not exist.

- [ ] **Step 7: Implement canonical identity parsing**

```python
from .contract import validate_job_identity_fields


@dataclass(frozen=True)
class JobIdentity:
    prefix: str
    run_id: str
    run_attempt: str
    github_job: str
    leg: str

    @classmethod
    def from_environment(
        cls,
        contract: InstanceContract,
        environment: Mapping[str, str],
    ) -> "JobIdentity":
        if environment.get("GITHUB_REPOSITORY") != contract.repository:
            raise JobIdentityError("repository_mismatch")
        run_id = environment.get("GITHUB_RUN_ID", "")
        run_attempt = environment.get("GITHUB_RUN_ATTEMPT", "")
        github_job = environment.get("GITHUB_JOB", "")
        leg = environment.get("RUNNER_GUARD_LEG", "")
        values = validate_job_identity_fields(
            contract,
            run_id,
            run_attempt,
            github_job,
            leg,
        )
        return cls(contract.job_prefix, *values)
```

Implement basenames by joining only validated components; accept no caller-supplied path.

- [ ] **Step 8: Run focused tests and commit**

Run: `.venv/bin/python -m unittest tests.test_cleanup_contract tests.test_job_identity -v`

Expected: PASS.

```bash
git add src/runner_guard/job_identity.py tests/test_cleanup_contract.py tests/test_job_identity.py
git commit -m "feat: bind environment to canonical job identity"
```

### Task 2: macOS Descriptor Capability Probe

**Files:**
- Create: `src/runner_guard/capabilities.py`
- Create: `tests/test_capabilities.py`

**Interfaces:**
- Consumes: `/usr/bin/python3` runtime facts and injectable `CapabilityAdapter` methods.
- Produces: `CapabilityReport`, `probe_cleanup_capabilities(adapter: CapabilityAdapter) -> CapabilityReport`, and `require_cleanup_capabilities(adapter: CapabilityAdapter) -> CapabilityReport`.

- [ ] **Step 1: Write failing capability tests**

```python
class CapabilityTests(unittest.TestCase):
    def test_complete_macos_cpython_capability_set_passes(self):
        report = probe_cleanup_capabilities(FakeCapabilityAdapter.complete())
        self.assertTrue(report.supported)
        self.assertEqual(report.missing, ())

    def test_each_missing_primitive_fails_closed(self):
        required = (
            "cpython_3_9_or_newer", "darwin", "o_directory", "o_nofollow",
            "open_dir_fd", "stat_dir_fd", "stat_follow_symlinks_false",
            "unlink_dir_fd", "rmdir_dir_fd", "scandir_fd", "stable_dev_ino",
        )
        for name in required:
            with self.subTest(name=name), self.assertRaisesRegex(CapabilityError, name):
                require_cleanup_capabilities(FakeCapabilityAdapter.complete().without(name))

    def test_runtime_identity_is_reported_without_a_path(self):
        report = probe_cleanup_capabilities(FakeCapabilityAdapter.complete())
        self.assertEqual(report.implementation, "CPython")
        self.assertGreaterEqual(report.version[:2], (3, 9))
        self.assertEqual(report.platform, "darwin")
```

- [ ] **Step 2: Run probe tests to prove the gap**

Run: `.venv/bin/python -m unittest tests.test_capabilities -v`

Expected: FAIL because the probe is absent.

- [ ] **Step 3: Implement behavioral probing**

```python
@dataclass(frozen=True)
class CapabilityReport:
    supported: bool
    missing: Sequence[str]
    implementation: str
    version: Tuple[int, int, int]
    platform: str

def require_cleanup_capabilities(adapter: CapabilityAdapter) -> CapabilityReport:
    report = probe_cleanup_capabilities(adapter)
    if not report.supported:
        raise CapabilityError("capability_missing:" + ",".join(report.missing))
    return report
```

The system adapter creates a private fixture, exercises `open`, `stat`, `scandir`, `unlink`, and `rmdir` through directory descriptors, proves non-following stat and `O_NOFOLLOW` symlink rejection, and compares stable `st_dev`/`st_ino`. It closes every descriptor before removing only its own fixture. The report contains capability names and runtime facts, never a path.

- [ ] **Step 4: Run under the supported runtime and commit**

Run: `.venv/bin/python -m unittest tests.test_capabilities -v`

Expected: PASS. The fake-adapter unit tests run inside the development environment; the system-runtime proof is exercised later by spawning `/usr/bin/python3 -I -S` against the rendered standalone probe.

```bash
git add src/runner_guard/capabilities.py tests/test_capabilities.py
git commit -m "feat: gate cleanup on descriptor capabilities"
```

### Task 3: Descriptor-Relative Filesystem Operations

**Files:**
- Create: `src/runner_guard/fsops.py`
- Create: `tests/support/tree_snapshot.py`
- Create: `tests/test_fsops.py`

**Interfaces:**
- Consumes: capability approval, `InstanceContract.account_name`, absolute contract paths, and expected device.
- Produces: `AccountAdapter.current_uid() -> int`, `AccountAdapter.uid_for_account(account_name: str) -> int`, `SystemAccountAdapter` backed by `os.getuid()` and `pwd.getpwnam()`, `resolve_runner_uid(contract: InstanceContract, adapter: AccountAdapter) -> int`, `BudgetProtocol` with `charge_entry(depth: int, size: int) -> None` and `checkpoint() -> None`, `OpenedDirectory`, `EntryIdentity`, `TreeInventory`, `open_absolute_directory_no_follow(path: str) -> OpenedDirectory`, `inventory_child_tree(parent: OpenedDirectory, name: str, expected_uid: int, budget: BudgetProtocol) -> TreeInventory`, and `delete_inventory(parent: OpenedDirectory, inventory: TreeInventory, budget: BudgetProtocol) -> tuple[int, int]`.

- [ ] **Step 1: Create a no-follow fixture snapshot helper**

```python
def snapshot_tree(root: Path) -> Sequence[Tuple[str, int, int, str]]:
    records = []
    for directory, names, files in os.walk(root, topdown=True, followlinks=False):
        names.sort()
        files.sort()
        for name in names + files:
            path = Path(directory, name)
            metadata = path.lstat()
            digest = "symlink:" + os.readlink(path) if stat.S_ISLNK(metadata.st_mode) else file_digest_or_empty(path, metadata)
            records.append((str(path.relative_to(root)), metadata.st_mode, metadata.st_nlink, digest))
    return tuple(records)
```

Every rejection test below records this snapshot before invoking the runtime and compares it afterward. Link targets receive a separate snapshot.

- [ ] **Step 2: Write failing safe-open tests**

```python
def test_nested_symlink_ancestor_is_rejected_unchanged():
    fixture = make_symlink_ancestor_fixture()
    before = snapshot_tree(fixture.root)
    external_before = snapshot_tree(fixture.external)
    with self.assertRaisesRegex(FilesystemSafetyError, "symlink_ancestor"):
        open_absolute_directory_no_follow(str(fixture.symlinked_child))
    self.assertEqual(snapshot_tree(fixture.root), before)
    self.assertEqual(snapshot_tree(fixture.external), external_before)

def test_dot_components_and_relative_paths_are_rejected():
    for value in ("relative/root", "/safe/../unsafe", "/safe/./child"):
        with self.subTest(value=value), self.assertRaises(FilesystemSafetyError):
            open_absolute_directory_no_follow(value)
```

- [ ] **Step 3: Run safe-open tests to prove the gap**

Run: `.venv/bin/python -m unittest tests.test_fsops.DescriptorOpenTests -v`

Expected: FAIL because `runner_guard.fsops` is absent.

- [ ] **Step 4: Implement component-by-component opening**

```python
@dataclass
class OpenedDirectory:
    fd: int
    device: int
    inode: int
    uid: int
    mode: int

    def close(self) -> None:
        if self.fd >= 0:
            os.close(self.fd)
            self.fd = -1

def open_absolute_directory_no_follow(path: str) -> OpenedDirectory:
    parts = validate_absolute_components(path)
    current = os.open("/", os.O_RDONLY | os.O_DIRECTORY | os.O_NOFOLLOW)
    try:
        for component in parts:
            child = os.open(component, os.O_RDONLY | os.O_DIRECTORY | os.O_NOFOLLOW, dir_fd=current)
            os.close(current)
            current = child
        metadata = os.fstat(current)
        return OpenedDirectory(current, metadata.st_dev, metadata.st_ino, metadata.st_uid, stat.S_IMODE(metadata.st_mode))
    except BaseException:
        os.close(current)
        raise
```

Map `ELOOP`, `ENOTDIR`, and identity mismatch to stable reason codes without returning the path.

- [ ] **Step 5: Write failing account-identity tests**

```python
def test_runner_uid_comes_from_account_database_and_matches_process() -> None:
    adapter = FakeAccountAdapter(current_uid=502, account_uids={"acmerunner": 502})
    self.assertEqual(resolve_runner_uid(contract_with(account_name="acmerunner"), adapter), 502)

def test_missing_or_different_execution_account_is_rejected_without_mutation() -> None:
    fixture = make_regular_tree_fixture()
    before = snapshot_tree(fixture.root)
    adapters = (
        FakeAccountAdapter(current_uid=502, account_uids={}),
        FakeAccountAdapter(current_uid=501, account_uids={"acmerunner": 502}),
    )
    for adapter in adapters:
        with self.subTest(adapter=adapter), self.assertRaises(FilesystemSafetyError):
            resolve_runner_uid(contract_with(account_name="acmerunner"), adapter)
        self.assertEqual(snapshot_tree(fixture.root), before)
```

Implement `SystemAccountAdapter.uid_for_account()` with `pwd.getpwnam(account_name).pw_uid`, translate `KeyError` to `account_not_found`, and compare that UID with `os.getuid()`. No contract field may store or override the UID. Pass the resolved value to every mutable-root and candidate ownership check; root-owned bootstrap/controller checks require UID `0` explicitly.

- [ ] **Step 6: Write failing inventory/revalidation tests**

```python
def test_symlink_and_hard_link_are_rejected_unchanged():
    for fixture in (make_symlink_candidate_fixture(), make_hard_link_fixture()):
        before = snapshot_tree(fixture.root)
        external_before = snapshot_tree(fixture.external)
        with self.subTest(kind=fixture.kind), self.assertRaises(FilesystemSafetyError):
            inventory_child_tree(fixture.parent, fixture.name, os.getuid(), UnlimitedTestBudget())
        self.assertEqual(snapshot_tree(fixture.root), before)
        self.assertEqual(snapshot_tree(fixture.external), external_before)

def test_inode_substitution_before_delete_is_rejected_unchanged():
    fixture = make_regular_tree_fixture()
    inventory = inventory_child_tree(fixture.parent, fixture.name, os.getuid(), UnlimitedTestBudget())
    fixture.replace_root_with_same_named_directory()
    replacement = snapshot_tree(fixture.root)
    with self.assertRaisesRegex(FilesystemSafetyError, "identity_changed"):
        delete_inventory(fixture.parent, inventory, UnlimitedTestBudget())
    self.assertEqual(snapshot_tree(fixture.root), replacement)
```

- [ ] **Step 7: Implement two-pass inventory and deletion**

```python
@dataclass(frozen=True)
class EntryIdentity:
    components: Sequence[str]
    device: int
    inode: int
    uid: int
    mode: int
    links: int
    size: int

@dataclass(frozen=True)
class TreeInventory:
    root: EntryIdentity
    postorder: Sequence[EntryIdentity]
    entries: int
    bytes: int
```

Inventory with `os.scandir(directory_fd)`, non-following stat, and child opens using `O_DIRECTORY | O_NOFOLLOW`. Reject links and special files; require one link for regular files; require expected UID and starting device. Before each `os.unlink(name, dir_fd=parent_fd)` or `os.rmdir(name, dir_fd=parent_fd)`, reopen the parent descriptor chain and compare device, inode, type, UID, mode, link count, and size to the recorded identity.

- [ ] **Step 8: Run filesystem tests and commit**

Run: `.venv/bin/python -m unittest tests.test_fsops -v`

Expected: PASS; every rejection fixture and external target is unchanged.

```bash
git add src/runner_guard/fsops.py tests/support/tree_snapshot.py tests/test_fsops.py
git commit -m "feat: add no-follow filesystem primitives"
```

### Task 4: Monotonic Budgets and Bounded Receipts

**Files:**
- Create: `src/runner_guard/cleanup.py`
- Create: `tests/test_cleanup.py`

**Interfaces:**
- Consumes: `CleanupLimits`, `JobIdentity`, and Task 3 filesystem interfaces.
- Produces: `CleanupPhase`, `CleanupReceipt`, `CleanupBudget`, `CleanupRuntime(clock_ns: Callable[[], int], account_adapter: AccountAdapter, command_runner: CommandRunner)`, `run_cleanup(phase: CleanupPhase, contract: InstanceContract, environment: Mapping[str, str]) -> CleanupReceipt`, and `main(argv: Optional[Sequence[str]] = None) -> int`.

- [ ] **Step 1: Write failing budget tests**

```python
class CleanupBudgetTests(unittest.TestCase):
    def test_limits_are_inclusive_and_the_next_unit_fails(self):
        limits = CleanupLimits(max_entries=2, max_depth=3, max_bytes=10, max_seconds=120)
        budget = CleanupBudget(limits, SequenceClock([0, 0, 0, 0]))
        budget.charge_entry(depth=3, size=4)
        budget.charge_entry(depth=1, size=6)
        self.assertEqual((budget.entries, budget.bytes), (2, 10))
        with self.assertRaisesRegex(CleanupBoundError, "entry_limit"):
            budget.charge_entry(depth=1, size=0)

    def test_depth_byte_and_deadline_overflow_fail(self):
        limits = CleanupLimits(200000, 64, 53687091200, 120)
        actions = (
            (lambda: CleanupBudget(limits, SequenceClock([0, 0])).charge_entry(65, 0), "depth_limit"),
            (lambda: CleanupBudget(limits, SequenceClock([0, 0])).charge_entry(1, 53687091201), "byte_limit"),
            (lambda: CleanupBudget(limits, SequenceClock([0, 120_000_000_001])).checkpoint(), "deadline"),
        )
        for action, reason in actions:
            with self.subTest(reason=reason), self.assertRaisesRegex(CleanupBoundError, reason):
                action()
```

- [ ] **Step 2: Run budget tests to prove the gap**

Run: `.venv/bin/python -m unittest tests.test_cleanup.CleanupBudgetTests -v`

Expected: FAIL because budget types are absent.

- [ ] **Step 3: Implement integer nanosecond accounting**

```python
class CleanupBudget:
    def __init__(self, limits: CleanupLimits, clock_ns: Callable[[], int]) -> None:
        self.limits = limits
        self.clock_ns = clock_ns
        self.started_ns = clock_ns()
        self.deadline_ns = self.started_ns + limits.max_seconds * 1_000_000_000
        self.entries = 0
        self.bytes = 0

    def checkpoint(self) -> None:
        if self.clock_ns() > self.deadline_ns:
            raise CleanupBoundError("deadline")

    def charge_entry(self, depth: int, size: int) -> None:
        self.checkpoint()
        next_entries = self.entries + 1
        next_bytes = self.bytes + size
        if depth > self.limits.max_depth:
            raise CleanupBoundError("depth_limit")
        if next_entries > self.limits.max_entries:
            raise CleanupBoundError("entry_limit")
        if next_bytes > self.limits.max_bytes:
            raise CleanupBoundError("byte_limit")
        self.entries = next_entries
        self.bytes = next_bytes
```

Call `checkpoint()` before and after each scan, before every descent, and immediately before every mutation.

- [ ] **Step 4: Write failing bounded-receipt tests**

```python
class CleanupReceiptTests(unittest.TestCase):
    def test_receipt_is_canonical_content_free_and_bounded(self):
        receipt = CleanupReceipt.pass_result("before", scanned=17, removed=3, removed_bytes=4096, elapsed_ms=12)
        encoded = receipt.to_line()
        self.assertLessEqual(len(encoded.encode("utf-8")), 512)
        self.assertEqual(encoded, '{"elapsed_ms":12,"phase":"before","reason":null,"removed":3,"removed_bytes":4096,"scanned":17,"status":"pass"}')
        for forbidden in ("/Users/", "GITHUB_", "token", "secret", "filename"):
            self.assertNotIn(forbidden, encoded)

    def test_reason_must_come_from_the_closed_enumeration(self):
        with self.assertRaises(ValueError):
            CleanupReceipt.blocked("before", "caller supplied detail", 0).to_line()
```

- [ ] **Step 5: Implement receipts and CLI exits**

```python
class CleanupPhase(enum.Enum):
    BEFORE = "before"
    AFTER = "after"
    WORKFLOW = "workflow"

ALLOWED_REASONS = frozenset({
    "capability_missing", "contract_invalid", "identity_invalid", "root_invalid",
    "candidate_invalid", "entry_limit", "depth_limit", "byte_limit", "deadline",
    "identity_changed", "mutation_failed", "workspace_invalid", "git_clean_failed",
})

@dataclass(frozen=True)
class CleanupReceipt:
    phase: str
    status: str
    scanned: int
    removed: int
    removed_bytes: int
    elapsed_ms: int
    reason: Optional[str]

    def to_line(self) -> str:
        if self.reason is not None and self.reason not in ALLOWED_REASONS:
            raise ValueError("unregistered_reason")
        value = json.dumps(dataclasses.asdict(self), sort_keys=True, separators=(",", ":"))
        if len(value.encode("utf-8")) > 512:
            raise ValueError("receipt_too_large")
        return value
```

The shared public function has the exact cross-plan signature and no test-only parameters:

```python
def run_cleanup(
    phase: CleanupPhase,
    contract: InstanceContract,
    environment: Mapping[str, str],
) -> CleanupReceipt:
    runtime = CleanupRuntime(
        clock_ns=time.monotonic_ns,
        account_adapter=SystemAccountAdapter(),
        command_runner=SystemCommandRunner(),
    )
    return runtime.run(phase, contract, environment)
```

Deterministic tests instantiate `CleanupRuntime` with fake adapters. The installed entry point accepts only `--phase before|after|workflow`; the descriptor-bound activation layer loads contract bytes with `load_contract_bytes()` and calls `run_cleanup(CleanupPhase(value), contract, environment)`. It emits exactly one line. Exit `0` only for pass, `1` for controlled rejection, and `2` for a bounded internal failure; never print a traceback.

- [ ] **Step 6: Run focused tests and commit**

Run: `.venv/bin/python -m unittest tests.test_cleanup.CleanupBudgetTests tests.test_cleanup.CleanupReceiptTests -v`

Expected: PASS.

```bash
git add src/runner_guard/cleanup.py tests/test_cleanup.py
git commit -m "feat: enforce cleanup budgets and receipts"
```

### Task 5: Before, After, and Exact Workflow Cleanup

**Files:**
- Modify: `src/runner_guard/cleanup.py`
- Create: `src/runner_guard/workspace.py`
- Modify: `tests/test_cleanup.py`
- Create: `tests/test_workspace.py`

**Interfaces:**
- Consumes: `JobIdentity`, `CleanupBudget`, `open_absolute_directory_no_follow`, `inventory_child_tree`, and `delete_inventory`.
- Produces: `plan_before_cleanup(contract: InstanceContract, temp_root: OpenedDirectory, expected_uid: int, budget: CleanupBudget) -> Sequence[TreeInventory]`, `plan_current_run_cleanup(contract: InstanceContract, identity: JobIdentity, temp_root: OpenedDirectory, expected_uid: int, budget: CleanupBudget) -> Sequence[TreeInventory]`, `plan_exact_job_cleanup(contract: InstanceContract, identity: JobIdentity, temp_root: OpenedDirectory, expected_uid: int, budget: CleanupBudget) -> Optional[TreeInventory]`, and `clean_validated_workspace(contract: InstanceContract, runner_uid: int, runner: CommandRunner) -> None`.

- [ ] **Step 1: Write failing before-phase tests**

```python
class BeforeCleanupTests(CleanupFixtureTestCase):
    def test_before_removes_only_valid_stale_instance_directories(self):
        stale = self.make_job_root("guard-job-11-1-build-main")
        unrelated = self.make_directory("github-managed")
        receipt = self.run_phase("before", run_id="12", attempt="1", job="build", leg="main")
        self.assertEqual(receipt.status, "pass")
        self.assertFalse(stale.exists())
        self.assertTrue(unrelated.exists())

    def test_malformed_matching_candidate_blocks_all_deletion(self):
        valid = self.make_job_root("guard-job-11-1-build-main")
        malformed = self.make_directory("guard-job-not-canonical")
        before = snapshot_tree(self.fixture_root)
        receipt = self.run_phase("before")
        self.assertEqual((receipt.status, receipt.reason), ("blocked", "candidate_invalid"))
        self.assertEqual(snapshot_tree(self.fixture_root), before)
        self.assertTrue(valid.exists() and malformed.exists())
```

- [ ] **Step 2: Run before tests to prove the gap**

Run: `.venv/bin/python -m unittest tests.test_cleanup.BeforeCleanupTests -v`

Expected: FAIL because phase planning is absent.

- [ ] **Step 3: Implement all-candidates-first before planning**

Scan only the opened `temp_root` descriptor. Ignore names that do not begin with `job_prefix + "-"`. Parse every matching complete basename through the canonical validators and allowed-leg membership. Sort by ASCII basename, inventory all candidates, and charge all budgets before deleting any candidate. A malformed candidate blocks the phase and preserves every candidate.

- [ ] **Step 4: Write failing after/workflow tests**

```python
class CurrentCleanupTests(CleanupFixtureTestCase):
    def test_after_removes_all_current_run_legs_and_preserves_other_run(self):
        for name in ("guard-job-20-2-test-3.13", "guard-job-20-2-test-3.14"):
            self.make_job_root(name)
        other = self.make_job_root("guard-job-21-1-test-3.14")
        receipt = self.run_phase("after", run_id="20", attempt="2", job="test", leg="3.14")
        self.assertEqual(receipt.removed, 2)
        self.assertTrue(other.exists())

    def test_workflow_removes_only_exact_leg(self):
        exact = self.make_job_root("guard-job-20-2-test-3.14")
        sibling = self.make_job_root("guard-job-20-2-test-3.13")
        receipt = self.run_phase("workflow", run_id="20", attempt="2", job="test", leg="3.14")
        self.assertFalse(exact.exists())
        self.assertTrue(sibling.exists())
        self.assertEqual(receipt.removed, 1)

    def test_absence_is_idempotent_success(self):
        for phase in ("before", "after", "workflow"):
            with self.subTest(phase=phase):
                self.assertEqual(self.run_phase(phase).status, "pass")
```

- [ ] **Step 5: Implement current-run and exact selectors**

For after, select only basenames beginning with `identity.run_prefix()` and validate the entire basename before inventory. For workflow, select only `identity.job_basename()`. Both perform a full planning pass before the first mutation.

- [ ] **Step 6: Write failing rejection/no-mutation tests**

```python
def test_every_preflight_rejection_preserves_fixture(self):
    builders = (
        self.symlinked_temp_root, self.symlink_candidate, self.nested_symlink_ancestor,
        self.foreign_owned_candidate, self.unexpected_hard_link, self.cross_device_candidate,
        self.depth_over_limit, self.entries_over_limit, self.bytes_over_limit,
        self.deadline_over_limit,
    )
    for build in builders:
        with self.subTest(case=build.__name__):
            case = build()
            before = snapshot_tree(case.root)
            external_before = snapshot_tree(case.external)
            self.assertEqual(case.run().status, "blocked")
            self.assertEqual(snapshot_tree(case.root), before)
            self.assertEqual(snapshot_tree(case.external), external_before)
```

Use mocked metadata for foreign UID and device cases so tests need neither root nor a second volume.

- [ ] **Step 7: Write failing workspace-cleaner tests**

```python
def test_workspace_clean_uses_exact_git_argv_without_shell():
    runner = RecordingCommandRunner(returncode=0)
    clean_validated_workspace(valid_contract(), runner)
    self.assertEqual(runner.argv, ("/usr/bin/git", "-C", valid_contract().workspace_root, "clean", "-ffdx"))
    self.assertEqual(runner.timeout, 60)
    self.assertNotIn("HOME", runner.environment)

def test_invalid_worktree_is_preserved_without_git(self):
    fixture = make_symlinked_git_directory_fixture()
    before = snapshot_tree(fixture.root)
    runner = RecordingCommandRunner(returncode=0)
    with self.assertRaisesRegex(WorkspaceError, "workspace_invalid"):
        clean_validated_workspace(fixture.contract, runner)
    self.assertEqual(runner.calls, 0)
    self.assertEqual(snapshot_tree(fixture.root), before)
```

- [ ] **Step 8: Implement exact ordinary-worktree cleanup**

```python
def clean_validated_workspace(
    contract: InstanceContract,
    runner_uid: int,
    runner: CommandRunner,
) -> None:
    workspace = open_absolute_directory_no_follow(contract.workspace_root)
    try:
        verify_workspace_identity(workspace, contract)
        verify_real_owned_git_directory(workspace, runner_uid)
        result = runner.run(
            ("/usr/bin/git", "-C", contract.workspace_root, "clean", "-ffdx"),
            environment={"PATH": "/usr/bin:/bin", "LC_ALL": "C"},
            timeout=60,
        )
        if result.returncode != 0:
            raise WorkspaceError("git_clean_failed")
    finally:
        workspace.close()
```

The production command runner calls `subprocess.run` with an argument tuple, `shell=False`, no stdin, captured bounded output that is discarded, and timeout 60. Only workflow phase invokes it.

- [ ] **Step 9: Run cleanup/workspace tests and commit**

Run: `.venv/bin/python -m unittest tests.test_cleanup tests.test_workspace -v`

Expected: PASS; every rejected fixture is unchanged and receipts stay within 512 UTF-8 bytes.

```bash
git add src/runner_guard/cleanup.py src/runner_guard/workspace.py tests/test_cleanup.py tests/test_workspace.py
git commit -m "feat: implement bounded cleanup phases"
```

### Task 6: Read-Only Residue Audit

**Files:**
- Create: `src/runner_guard/residue.py`
- Create: `scripts/audit-residue.py`
- Create: `tests/test_residue.py`

**Interfaces:**
- Consumes: `InstanceContract`, safe open/inventory functions, and `CleanupBudget`.
- Produces: `ResidueReport`, `audit_residue(contract: InstanceContract, clock_ns: Callable[[], int] = time.monotonic_ns) -> ResidueReport`, and `main(argv: Optional[Sequence[str]] = None) -> int`.

- [ ] **Step 1: Write failing mutation-proof residue tests**

```python
def test_empty_instance_reports_clean_without_mutation():
    fixture = make_instance_fixture()
    before = snapshot_tree(fixture.root)
    report = audit_residue(fixture.contract)
    self.assertEqual(report.to_line(), '{"job_roots":0,"status":"clean","workspace_entries":0}')
    self.assertEqual(snapshot_tree(fixture.root), before)

def test_residue_and_malformed_candidate_do_not_mutate(self):
    for fixture in (make_valid_residue_fixture(), make_symlink_residue_fixture()):
        before = snapshot_tree(fixture.root)
        external_before = snapshot_tree(fixture.external)
        report = audit_residue(fixture.contract)
        self.assertIn(report.status, ("residue", "blocked"))
        self.assertEqual(snapshot_tree(fixture.root), before)
        self.assertEqual(snapshot_tree(fixture.external), external_before)
```

- [ ] **Step 2: Run residue tests to prove the gap**

Run: `.venv/bin/python -m unittest tests.test_residue -v`

Expected: FAIL because the residue module is absent.

- [ ] **Step 3: Implement bounded read-only audit**

```python
@dataclass(frozen=True)
class ResidueReport:
    status: str
    job_roots: int
    workspace_entries: int

    def to_line(self) -> str:
        return json.dumps(dataclasses.asdict(self), sort_keys=True, separators=(",", ":"))
```

Open roots with the same descriptor checks, inventory under the same limits, and perform no unlink, rmdir, Git, subprocess, or write. Return exit `0` for clean, `1` for proven residue, and `2` for blocked. `scripts/audit-residue.py --contract CONTRACT` is a thin development/operator wrapper around this module; it accepts no root override and emits only `ResidueReport.to_line()`.

- [ ] **Step 4: Run tests and commit**

Run: `.venv/bin/python -m unittest tests.test_residue -v`

Expected: PASS.

```bash
git add src/runner_guard/residue.py scripts/audit-residue.py tests/test_residue.py
git commit -m "feat: add read-only residue audit"
```

### Task 7: Stable Hooks and Workflow Always Cleanup

**Files:**
- Create: `templates/job-started.sh`
- Create: `templates/job-completed.sh`
- Modify: `templates/workflow-guard.yml`
- Create: `tests/test_hooks.py`

**Interfaces:**
- Consumes: renderer substitutions `@@ACTIVATE_PY@@`, `@@JOB_PREFIX@@`, and `@@ALLOWED_LEGS_JSON@@`; generation bootstrap CLI `activate.py --phase before|after|workflow`.
- Produces: stable started/completed wrappers and a workflow fragment with exact per-leg storage and always-run cleanup.

- [ ] **Step 1: Write failing exact-hook tests**

```python
class HookTemplateTests(unittest.TestCase):
    def test_hooks_have_one_system_python_execution(self):
        phases = {"job-started.sh": "before", "job-completed.sh": "after"}
        for filename, phase in phases.items():
            text = Path("templates", filename).read_text(encoding="utf-8")
            self.assertTrue(text.startswith("#!/bin/bash\nset -euo pipefail\numask 077\n"))
            self.assertIn(f'exec /usr/bin/python3 -I -S "@@ACTIVATE_PY@@" --phase {phase}', text)
            for forbidden in ("/opt/homebrew", "eval", "rm -", "pkill", "killall"):
                self.assertNotIn(forbidden, text)
```

- [ ] **Step 2: Run hook tests to prove the gap**

Run: `.venv/bin/python -m unittest tests.test_hooks.HookTemplateTests -v`

Expected: FAIL because hook templates are absent.

- [ ] **Step 3: Create exact stable wrappers**

`templates/job-started.sh`:

```bash
#!/bin/bash
set -euo pipefail
umask 077
exec /usr/bin/python3 -I -S "@@ACTIVATE_PY@@" --phase before
```

`templates/job-completed.sh`:

```bash
#!/bin/bash
set -euo pipefail
umask 077
exec /usr/bin/python3 -I -S "@@ACTIVATE_PY@@" --phase after
```

- [ ] **Step 4: Write failing workflow-fragment tests**

```python
def test_workflow_uses_private_root_and_always_cleanup():
    workflow = load_yaml("templates/workflow-guard.yml")
    job = workflow["jobs"]["guarded-job"]
    self.assertEqual(job["strategy"]["matrix"]["leg"], "@@ALLOWED_LEGS_JSON@@")
    self.assertEqual(job["env"]["RUNNER_GUARD_LEG"], "${{ matrix.leg }}")
    text = canonical_yaml_text(workflow)
    self.assertIn("$RUNNER_TEMP/@@JOB_PREFIX@@-$GITHUB_RUN_ID-$GITHUB_RUN_ATTEMPT-$GITHUB_JOB-$RUNNER_GUARD_LEG", text)
    cleanup = job["steps"][-1]
    self.assertEqual(cleanup["if"], "always()")
    self.assertIn("/usr/bin/python3 -I -S", cleanup["run"])
    self.assertIn("--phase workflow", cleanup["run"])
```

- [ ] **Step 5: Implement exact per-leg storage and always cleanup**

The fragment creates the exact job root mode `0700`, failing if it exists. It creates `tmp`, `cache`, `pycache`, `venv`, `dist`, and `reports` below it and exports `TMPDIR`, `XDG_CACHE_HOME`, `PIP_CACHE_DIR`, and `PYTHONPYCACHEPREFIX` there. Its final step is:

```yaml
- name: Clean exact guarded job storage
  if: always()
  shell: /bin/bash --noprofile --norc -euo pipefail {0}
  run: >-
    /usr/bin/python3 -I -S "@@ACTIVATE_PY@@"
    --phase workflow
```

The fragment contains no broad deletion, global `/tmp`, cache upload, artifact upload, or process termination.

- [ ] **Step 6: Validate templates and commit**

Run: `/bin/bash -n templates/job-started.sh && /bin/bash -n templates/job-completed.sh`

Expected: exit `0` with no output.

Run: `.venv/bin/python -m unittest tests.test_hooks -v`

Expected: PASS.

```bash
git add templates/job-started.sh templates/job-completed.sh templates/workflow-guard.yml tests/test_hooks.py
git commit -m "feat: add stable cleanup hooks and workflow layer"
```

### Task 8: Deterministic Standalone Runtime and Descriptor-Bound Activation

**Files:**
- Create: `src/runner_guard/bundle_runtime.py`
- Create: `templates/activate.py`
- Create: `tests/test_runtime_bundle.py`

**Interfaces:**
- Consumes: exact UTF-8 source bytes for `runner_guard.capabilities`, `runner_guard.contract`, `runner_guard.errors`, `runner_guard.job_identity`, `runner_guard.fsops`, `runner_guard.workspace`, `runner_guard.cleanup`, and `runner_guard.residue`; canonical active/generation manifest bytes from the lifecycle plan.
- Produces: `RuntimeModule(name: str, source: bytes)`, `build_runtime_bundle(modules: Sequence[RuntimeModule]) -> bytes`, standalone `run_generation_entry(phase: str, contract_bytes: bytes, environment: Mapping[str, str]) -> CleanupReceipt`, and bootstrap `execute_verified_controller(controller_fd: int, expected_sha256: str, phase: str, contract_bytes: bytes, environment: Mapping[str, str]) -> CleanupReceipt`.

- [ ] **Step 1: Write failing deterministic-bundle tests**

```python
class RuntimeBundleTests(unittest.TestCase):
    def test_bundle_is_order_independent_and_contains_no_external_import_path(self) -> None:
        first = build_runtime_bundle(runtime_modules())
        second = build_runtime_bundle(tuple(reversed(runtime_modules())))
        self.assertEqual(first, second)
        self.assertNotIn(b"site-packages", first)
        self.assertNotIn(b"PYTHONPATH", first)
        self.assertNotIn(b"/Users/", first)

    def test_duplicate_missing_or_extra_runtime_module_is_rejected(self) -> None:
        cases = (
            runtime_modules_without("runner_guard.fsops"),
            runtime_modules() + (runtime_modules()[0],),
            runtime_modules() + (RuntimeModule("runner_guard.unreviewed", b"VALUE = 1\n"),),
        )
        for modules in cases:
            with self.subTest(names=tuple(item.name for item in modules)), self.assertRaises(RuntimeBundleError):
                build_runtime_bundle(modules)
```

- [ ] **Step 2: Run bundle tests to prove the gap**

Run: `.venv/bin/python -m unittest tests.test_runtime_bundle.RuntimeBundleTests -v`

Expected: FAIL because `runner_guard.bundle_runtime` is absent.

- [ ] **Step 3: Implement the deterministic in-memory module bundle**

The builder validates the exact closed module-name set above, rejects duplicate names, requires UTF-8 ending in one LF, sorts by ASCII module name, and embeds a canonical JSON source map. The emitted single file uses a standard-library `importlib.abc.MetaPathFinder`/`Loader` that serves only the embedded `runner_guard` package and closed module set. It removes its finder after importing `runner_guard.cleanup`; it does not read `PYTHONPATH`, site packages, the current directory, or installed toolkit files. At top level it binds `CleanupPhase`, `CleanupReceipt`, `load_contract_bytes`, and `run_cleanup` from embedded modules before defining the entry point, allowing the isolated activation namespace to verify the returned receipt against the exact embedded type.

The final generated function is exact:

```python
def run_generation_entry(
    phase: str,
    contract_bytes: bytes,
    environment: Mapping[str, str],
) -> CleanupReceipt:
    contract = load_contract_bytes(contract_bytes)
    return run_cleanup(CleanupPhase(phase), contract, environment)
```

The emitted bundle ends with one LF and contains no timestamp, source absolute path, hash seed output, or host identity. When invoked as a script it accepts only `--capability-check-only`, calls the embedded `require_cleanup_capabilities(SystemCapabilityAdapter())`, emits its bounded canonical report, and exits without opening a runner or instance root.

- [ ] **Step 4: Write failing verified-execution and race tests**

```python
class RuntimeActivationTests(unittest.TestCase):
  def test_verified_controller_bytes_execute_in_isolated_namespace(self) -> None:
    bundle = build_runtime_bundle(runtime_modules())
    controller_fd = open_read_only_fixture(bundle)
    receipt = execute_verified_controller(
        controller_fd,
        hashlib.sha256(bundle).hexdigest(),
        "before",
        canonical_contract_bytes(valid_contract()),
        valid_env(run_id="7", attempt="1", job="build", leg="main"),
    )
    self.assertEqual(receipt.status, "pass")

  def test_path_swap_after_controller_open_cannot_change_executed_bytes(self) -> None:
    fixture = opened_controller_swap_fixture()
    original_before = snapshot_tree(fixture.root)
    fixture.swap_path_to_unreviewed_bytes()
    with self.assertRaisesRegex(ActivationError, "controller_identity_changed"):
        fixture.execute_after_parent_identity_change()
    self.assertEqual(snapshot_tree(fixture.external), fixture.external_before)
    self.assertNotEqual(snapshot_tree(fixture.root), original_before)

  def test_hash_mismatch_rejects_before_compile_or_exec(self) -> None:
    with mock.patch("builtins.compile") as compile_spy:
        with self.assertRaisesRegex(ActivationError, "controller_hash_mismatch"):
            execute_verified_controller(open_read_only_fixture(b"VALUE = 1\n"), "0" * 64, "before", b"{}\n", {})
    compile_spy.assert_not_called()
```

The swap test deliberately changes only the named fixture path and proves the external target is untouched. A second race test swaps the pathname after the controller descriptor opens but leaves the opened descriptor identity stable; execution must use the already opened verified bytes rather than reopening the pathname.

- [ ] **Step 5: Implement descriptor-bound `activate.py` execution**

`templates/activate.py` accepts only `--phase before|after|workflow`. Rendered absolute controller-root constants are inside the root-owned template; no instance path or controller path is accepted from argv or environment. It opens the controller root, `ACTIVE.json`, digest-named generation directory, `GENERATION_MANIFEST.json`, `cleanup_controller.py`, and `instance.json` descriptor-relatively with `O_NOFOLLOW`. It validates manifest membership, file mode `0555` for `cleanup_controller.py`, file mode `0444` for `instance.json`, UID `0`, single links, byte counts, SHA-256 values, and generation-directory device/inode before and after reads. It constructs the environment mapping from exactly `GITHUB_REPOSITORY`, `GITHUB_RUN_ID`, `GITHUB_RUN_ATTEMPT`, `GITHUB_JOB`, and `RUNNER_GUARD_LEG`; it never passes the complete process environment into generation code.

`execute_verified_controller()` reads through the already verified controller descriptor with a fixed maximum byte count, hashes the exact bytes, checks `fstat()` identity before and after, and only then calls:

```python
code = compile(controller_bytes, "<verified-cleanup-controller>", "exec", dont_inherit=True)
namespace = {
    "__name__": "__runner_guard_generation__",
    "__file__": "<verified-cleanup-controller>",
    "__package__": None,
    "__builtins__": builtins.__dict__,
}
exec(code, namespace, namespace)
entry = namespace.get("run_generation_entry")
if not callable(entry):
    raise ActivationError("controller_entry_missing")
receipt = entry(phase, contract_bytes, dict(environment))
receipt_type = namespace.get("CleanupReceipt")
if not isinstance(receipt_type, type) or not isinstance(receipt, receipt_type):
    raise ActivationError("controller_receipt_invalid")
return receipt
```

No controller module is imported by path. No verified file is reopened by pathname. Bootstrap output is the receipt line only.

- [ ] **Step 6: Prove `/usr/bin/python3 -I -S` standalone execution**

Add this subprocess test. It renders a disposable active-generation fixture containing only `activate.py`, `ACTIVE.json`, `GENERATION_MANIFEST.json`, `cleanup_controller.py`, and `instance.json`:

```python
class StandaloneRuntimeTests(unittest.TestCase):
  def test_rendered_generation_runs_with_system_python_in_isolation(self) -> None:
    with tempfile.TemporaryDirectory() as parent:
        fixture = render_active_generation_fixture(Path(parent))
        result = subprocess.run(
            ("/usr/bin/python3", "-I", "-S", str(fixture / "activate.py"), "--phase", "before"),
            cwd="/",
            env={"PATH": "/usr/bin:/bin", **valid_env(run_id="7", attempt="1", job="build", leg="main")},
            stdin=subprocess.DEVNULL,
            stdout=subprocess.PIPE,
            stderr=subprocess.PIPE,
            check=False,
            timeout=130,
        )
        self.assertEqual(result.returncode, 0, result.stderr.decode("utf-8", "replace"))
        self.assertEqual(json.loads(result.stdout)["status"], "pass")

  def test_rendered_capability_probe_runs_with_system_python(self) -> None:
    with tempfile.TemporaryDirectory() as parent:
        bundle = Path(parent, "cleanup_controller.py")
        bundle.write_bytes(build_runtime_bundle(runtime_modules()))
        bundle.chmod(0o555)
        result = subprocess.run(
            ("/usr/bin/python3", "-I", "-S", str(bundle), "--capability-check-only"),
            cwd="/",
            env={"PATH": "/usr/bin:/bin"},
            stdin=subprocess.DEVNULL,
            stdout=subprocess.PIPE,
            stderr=subprocess.PIPE,
            check=False,
            timeout=15,
        )
        report = json.loads(result.stdout)
        self.assertEqual(result.returncode, 0, result.stderr.decode("utf-8", "replace"))
        self.assertTrue(report["supported"])
        self.assertEqual(report["platform"], "darwin")
```

Expected: exit `0`, one bounded pass receipt, and no import from the repository, `.venv`, user site, or current directory. The test creates and removes only its exact disposable fixture.

- [ ] **Step 7: Run tests and commit**

Run: `.venv/bin/python -m unittest tests.test_runtime_bundle -v`

Expected: PASS, including deterministic bytes, hash-before-exec, descriptor/path-swap resistance, and standalone system-Python execution.

```bash
git add src/runner_guard/bundle_runtime.py templates/activate.py tests/test_runtime_bundle.py
git commit -m "feat: bundle and activate standalone cleanup runtime"
```

### Task 9: Deterministic Renderer Integration

**Files:**
- Verify: `src/runner_guard/render.py`
- Verify: `examples/dedicated-account.json`
- Verify: `examples/shared-account-trusted-only.json`
- Create: `tests/test_cleanup_render.py`

**Interfaces:**
- Consumes: `ComponentPayload(relative_path: str, mode: int, content: bytes)`, `RenderedFile(relative_path: str, mode: int, content: bytes)`, `render_instance(contract: InstanceContract, components: Sequence[ComponentPayload], template_root: Path, policy_source: bytes) -> tuple[RenderedFile, ...]`, `write_rendered_tree(output_dir: Path, files: Sequence[RenderedFile]) -> None`, `build_runtime_bundle(modules: Sequence[RuntimeModule]) -> bytes`, and `canonical_contract_bytes(contract) -> bytes`. The renderer's returned tuple is sorted by ASCII `relative_path`.
- Produces: `build_cleanup_components(source_root: Path) -> tuple[ComponentPayload, ...]`; its returned tuple contains exactly five mode-`0555` replacements: `activate.py`, `cleanup_controller.py`, `job-completed.sh`, `job-started.sh`, and `residue-audit.py`. After merging with the five lifecycle fixtures, rendering produces exactly the 15-path/mode inventory frozen below.

- [ ] **Step 1: Write failing renderer/determinism tests**

```python
def test_rendered_cleanup_files_are_bound_and_deterministic() -> None:
    contract = load_contract(Path("examples/dedicated-account.json"))
    components = merge_component_payloads(
        load_component_fixtures(Path("tests/fixtures/components")),
        build_cleanup_components(Path.cwd()),
    )
    policy_source = Path("src/runner_guard/policy.py").read_bytes()
    first = render_instance(contract, components, Path("templates"), policy_source)
    second = render_instance(
        contract, tuple(reversed(components)), Path("templates"), policy_source
    )
    self.assertEqual(first, second)
    by_name = {item.relative_path: item for item in first}
    self.assertEqual(by_name["cleanup_controller.py"].mode, 0o555)
    self.assertEqual(by_name["instance.json"].content, canonical_contract_bytes(contract))
    self.assertFalse(any(b"@@" in item.content for item in first))

def test_written_render_trees_are_byte_identical() -> None:
    contract = load_contract(Path("examples/dedicated-account.json"))
    components = merge_component_payloads(
        load_component_fixtures(Path("tests/fixtures/components")),
        build_cleanup_components(Path.cwd()),
    )
    files = render_instance(
        contract,
        components,
        Path("templates"),
        Path("src/runner_guard/policy.py").read_bytes(),
    )
    with tempfile.TemporaryDirectory() as parent:
        first = Path(parent, "first")
        second = Path(parent, "second")
        write_rendered_tree(first, files)
        write_rendered_tree(second, files)
        self.assertEqual(snapshot_tree(first), snapshot_tree(second))

def test_examples_retain_reviewed_limits() -> None:
    expected = CleanupLimits(200000, 64, 53687091200, 120)
    for name in ("dedicated-account.json", "shared-account-trusted-only.json"):
        self.assertEqual(load_contract(Path("examples", name)).cleanup_limits, expected)
```

- [ ] **Step 2: Run renderer tests to prove the gap**

Run: `.venv/bin/python -m unittest tests.test_cleanup_render -v`

Expected: FAIL because `build_cleanup_components()` is absent.

- [ ] **Step 3: Build exact components without changing the renderer contract**

`build_cleanup_components()` constructs the standalone controller bytes through `build_runtime_bundle()` and returns the exact component records:

```python
return (
    ComponentPayload("activate.py", 0o555, activate_bytes),
    ComponentPayload("cleanup_controller.py", 0o555, controller_bytes),
    ComponentPayload("job-completed.sh", 0o555, completed_hook_bytes),
    ComponentPayload("job-started.sh", 0o555, started_hook_bytes),
    ComponentPayload("residue-audit.py", 0o555, residue_bundle_bytes),
)
```

`merge_component_payloads(base, replacements) -> tuple[ComponentPayload, ...]` indexes both sequences by `relative_path`, requires every replacement path to exist in the child-plan-1 fixture set, replaces exactly those five records, rejects duplicates, and returns all ten required component records sorted by ASCII path. This prevents the cleanup sub-plan from weakening the renderer's complete-component requirement while lifecycle helpers remain inert reviewed fixtures.

`render_instance(contract, components, template_root, policy_source)` remains the child-plan-1 interface; do not add a destination parameter or return a new result type. The policy bytes are explicit reviewed input and the renderer must never discover them through `__file__`, the current directory, import state, or another ambient path. Call `write_rendered_tree()` separately when a filesystem tree is required. Unknown or residual template markers fail rendering. Rendering never reads live host state.

The exact renderer result is:

| Path | Mode |
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

`MANIFEST.json` and `SHA256SUMS` are absent from this result and are added only by artifact staging.

- [ ] **Step 4: Run renderer tests and commit**

Run: `.venv/bin/python -m unittest tests.test_cleanup_render -v`

Expected: PASS with byte-identical `RenderedFile` tuples and `cleanup_controller.py` mode `0555`.

```bash
git add src/runner_guard/bundle_runtime.py tests/test_cleanup_render.py
git commit -m "feat: integrate standalone cleanup rendering"
```

### Task 10: Cross-Instance and Interruption Recovery

**Files:**
- Create: `tests/test_cleanup_integration.py`
- Modify: `src/runner_guard/cleanup.py`
- Modify: `src/runner_guard/residue.py`

**Interfaces:**
- Consumes: complete cleanup, residue, contract, identity, and renderer interfaces from Tasks 1-8.
- Produces: verified separation and recovery behavior without a new public runtime interface.

- [ ] **Step 1: Write failing cross-instance tests**

```python
def test_project_a_cannot_address_project_b():
    host = make_two_instance_fixture(prefix_a="alpha-job", prefix_b="beta-job")
    host.alpha.make_job_root("alpha-job-5-1-test-main")
    beta = host.beta.make_job_root("beta-job-8-1-test-main", payload=b"keep")
    beta_before = snapshot_tree(host.beta.root)
    receipt = host.alpha.run("before", run_id="9", attempt="1", job="test", leg="main")
    self.assertEqual(receipt.status, "pass")
    self.assertEqual(snapshot_tree(host.beta.root), beta_before)
    self.assertTrue(beta.exists())

def test_overlapping_contract_roots_are_rejected_unchanged():
    host = make_overlapping_contract_fixture()
    before = snapshot_tree(host.root)
    with self.assertRaisesRegex(ContractError, "root_overlap"):
        validate_contract_set((host.alpha_contract, host.beta_contract))
    self.assertEqual(snapshot_tree(host.root), before)
```

- [ ] **Step 2: Write failing interruption tests**

```python
def test_next_before_removes_root_left_when_after_never_ran():
    fixture = make_instance_fixture()
    stale = fixture.make_job_root("guard-job-40-1-build-main", payload=b"interrupted")
    receipt = fixture.run("before", run_id="41", attempt="1", job="build", leg="main")
    self.assertEqual(receipt.status, "pass")
    self.assertFalse(stale.exists())

def test_deadline_during_inventory_preserves_entire_candidate():
    fixture = make_instance_fixture(clock=SequenceClock([0, 0, 120_000_000_001]))
    fixture.make_job_root("guard-job-40-1-build-main", nested_files=3)
    before = snapshot_tree(fixture.root)
    receipt = fixture.run("before", run_id="41", attempt="1", job="build", leg="main")
    self.assertEqual((receipt.status, receipt.reason), ("blocked", "deadline"))
    self.assertEqual(snapshot_tree(fixture.root), before)

def test_repeated_phases_and_audit_are_idempotent():
    fixture = make_instance_fixture()
    for operation in (fixture.before, fixture.before, fixture.after, fixture.after):
        self.assertEqual(operation().status, "pass")
    self.assertEqual(audit_residue(fixture.contract).status, "clean")
```

- [ ] **Step 3: Add static safety-source tests**

```python
def test_filesystem_runtime_has_no_broad_deletion_or_process_killing():
    sources = [Path(path).read_text(encoding="utf-8") for path in (
        "src/runner_guard/cleanup.py", "src/runner_guard/fsops.py",
        "templates/job-started.sh", "templates/job-completed.sh",
        "templates/workflow-guard.yml",
    )]
    for token in ("rm -rf", "shutil.rmtree", "pkill", "killall", "os.kill(", "shell=True"):
        self.assertFalse(any(token in source for source in sources), token)
```

`workspace.py` may use `subprocess.run` only with the fixed argument tuple and `shell=False` proven in Task 5.

- [ ] **Step 4: Run integration tests and apply the smallest correction**

Run: `.venv/bin/python -m unittest tests.test_cleanup_integration -v`

Expected: tests initially expose any cross-instance or interruption gap. Modify only `cleanup.py` or `residue.py` needed to satisfy the exact assertions, then rerun to PASS.

- [ ] **Step 5: Commit**

```bash
git add src/runner_guard/cleanup.py src/runner_guard/residue.py tests/test_cleanup_integration.py
git commit -m "test: prove cleanup isolation and recovery"
```

### Task 11: Final Cleanup Runtime Verification

**Files:**
- Verify only: every path in the File Map

**Interfaces:**
- Consumes: complete cleanup implementation.
- Produces: exact evidence for independent cleanup-safety review; no release or live-host transition.

- [ ] **Step 1: Run whitespace, Python, and shell syntax checks**

Run: `git diff --check`

Expected: exit `0`.

Run: `.venv/bin/python -m compileall -q src tests`

Expected: exit `0`.

Run: `/bin/bash -n templates/job-started.sh && /bin/bash -n templates/job-completed.sh`

Expected: exit `0` with no output.

- [ ] **Step 2: Run the complete focused suite**

```bash
.venv/bin/python -m unittest \
  tests.test_cleanup_contract \
  tests.test_job_identity \
  tests.test_capabilities \
  tests.test_fsops \
  tests.test_workspace \
  tests.test_cleanup \
  tests.test_residue \
  tests.test_hooks \
  tests.test_runtime_bundle \
  tests.test_cleanup_render \
  tests.test_cleanup_integration -v
```

Expected: all focused tests PASS; no test requires root, reads another user's files, or mutates a live runner.

- [ ] **Step 3: Exercise deterministic render/write twice**

Run: `.venv/bin/python -m unittest tests.test_cleanup_render -v`

Expected: PASS, including the two fresh rendered trees with identical snapshots and modes.

- [ ] **Step 4: Run the capability proof without live cleanup mutation**

Run: `.venv/bin/python -m unittest tests.test_runtime_bundle.StandaloneRuntimeTests.test_rendered_capability_probe_runs_with_system_python -v`

Expected: PASS after the test launches the rendered standalone probe with `/usr/bin/python3 -I -S`, receives one bounded JSON line with `supported=true`, CPython 3.9 or newer, platform `darwin`, and no paths, and removes only its private fixture.

- [ ] **Step 5: Inspect final scope and unsafe primitives**

Run: `git diff --name-only origin/main HEAD`

Expected: only paths in this plan's File Map plus shared scaffolding approved by the parent plan.

Run: `rg -n 'rm -rf|shutil\.rmtree|pkill|killall|os\.kill\(|shell=True' src/runner_guard templates tests examples`

Run: `! rg --pcre2 -n '/Users/(?!example-runner|acmewidgetrunner|trustedcirunner)([^/[:space:]]+)' src/runner_guard templates tests examples`

Expected: no runtime/template unsafe-primitive or private-identity matches. Tests may contain the forbidden tokens only inside static rejection assertions.

- [ ] **Step 6: Record the verified state and request independent review**

If verification required a test-only correction, commit it:

```bash
git add tests
git commit -m "test: finalize cleanup runtime verification"
```

If no correction was required, create no empty commit. Request independent review focused on descriptor safety, two-pass rejection immutability, budgets, receipt privacy, cross-instance separation, hook bootstrap paths, and renderer determinism. Do not install on a host, mutate a live runner, publish a package, or create a release.

## Spec Coverage Review

- Cleanup fields, allowed legs, and canonical identity: Task 1.
- CPython 3.9+, `/usr/bin/python3`, macOS, descriptor/no-follow gate: Task 2.
- Descriptor-relative inventory, inode/device revalidation, and link/device/ownership rejection: Task 3.
- Monotonic entry/depth/byte/time budgets and bounded receipts: Task 4.
- Before, current-run after, exact workflow, and validated checkout cleanup: Task 5.
- Read-only residue reporting: Task 6.
- Stable `/bin/bash` hooks, `/usr/bin/python3 -I -S`, exact job storage, and `if: always()` cleanup: Task 7.
- Deterministic standalone bundling, descriptor-bound activation, and path-swap resistance: Task 8.
- Contract-bound deterministic rendering and `0555` cleanup-controller mode: Task 9.
- Cross-instance isolation, interruption recovery, idempotency, and no process killing: Task 10.
- Full focused verification without live-host mutation: Task 11.
