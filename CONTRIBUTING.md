# Contributing

Thank you for helping improve macOS Runner Guard.

The project is currently design-first. External implementation pull requests are not yet accepted; issues and review feedback are welcome. The implementation lane opens only after stable GitHub-hosted PR check names exist and the `main` ruleset requires those checks, at least one approval, stale-approval dismissal, latest-push approval by someone other than its pusher, and resolution of review conversations. This public repository remains GitHub-hosted-only: it will not expose a persistent self-hosted runner to pull-request workflow bytes. An adopting private organization repository may use the generated persistent lane only after a dedicated organization runner group is independently proved selected to that repository and restricted to the exact protected-default-branch workflow. Before submitting implementation work after that gate opens:

1. read [the architecture](docs/ARCHITECTURE.md);
2. open an issue describing the bounded change and its trust-boundary impact;
3. keep credentials, live runner metadata, private paths, and project-specific evidence out of commits;
4. include deterministic tests for success, rejection, cleanup, rollback, and unrelated-runner preservation;
5. do not weaken a fail-closed guard merely to make a host pass.

Security vulnerabilities should follow [SECURITY.md](SECURITY.md), not a public issue.

## Branch governance

`main` is protected by the active GitHub ruleset **Protect main via pull requests** (ruleset ID `21842288`). It requires changes to arrive through a pull request, blocks branch deletion and non-fast-forward updates, and has no bypass actors. During the planning phase it intentionally requires zero approving reviews and no status checks. That minimum prevents direct history changes but is not sufficient for external implementation. The contribution lane remains closed until the strengthened live ruleset and hosted-only PR workflow are independently re-read and recorded. A label or push-only self-hosted workflow would not close this gate because proposed workflow bytes could target a repository-visible runner.
