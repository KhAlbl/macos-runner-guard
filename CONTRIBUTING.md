# Contributing

Thank you for helping improve macOS Runner Guard.

The project is currently design-first. Before submitting implementation work:

1. read [the architecture](docs/ARCHITECTURE.md);
2. open an issue describing the bounded change and its trust-boundary impact;
3. keep credentials, live runner metadata, private paths, and project-specific evidence out of commits;
4. include deterministic tests for success, rejection, cleanup, rollback, and unrelated-runner preservation;
5. do not weaken a fail-closed guard merely to make a host pass.

Security vulnerabilities should follow [SECURITY.md](SECURITY.md), not a public issue.

## Branch governance

`main` is protected by the active GitHub ruleset **Protect main via pull requests** (ruleset ID `21842288`). It requires changes to arrive through a pull request, blocks branch deletion and non-fast-forward updates, and has no bypass actors. During the planning phase it intentionally requires zero approving reviews and no status checks; those requirements will be strengthened only after stable CI and maintainer review capacity exist.
