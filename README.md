# macOS Runner Guard

Secure-by-default building blocks for repository-scoped, self-cleaning GitHub Actions runners on shared macOS hosts.

> [!IMPORTANT]
> This repository currently contains the approved architecture and draft implementation plans under final consistency review. It does not yet provide an installable runner controller or production release.

## Why this project

GitHub's official self-hosted runner is persistent. Reusing one macOS host across jobs and repositories can leave workspaces, caches, build products, and altered state behind. macOS Runner Guard is designed to add deterministic residue cleanup, project-specific routing, bounded host checks, reproducible adoption bundles, and explicit trust boundaries without replacing the official runner.

## Planned profiles

- **Dedicated account — recommended:** one non-admin macOS account and one repository-scoped runner per independent trust domain.
- **Shared account, trusted repositories only:** separate labels and roots under one account for routing and cleanup reliability. This profile does not provide credential or privacy isolation between those repositories.

Persistent cleanup is residue control, not VM isolation. Public forks, untrusted code, signing, deployment credentials, and hostile-code workloads should use an ephemeral VM or dedicated host.

## Planned capabilities

- repository-scoped registrations with unique routing labels;
- root-protected, per-project controller generations;
- automatic before-job, always-run, and after-job cleanup;
- descriptor-relative, no-follow filesystem operations;
- fail-closed workflow policy checks;
- deterministic toolkit and per-instance ZIP bundles;
- transactional activation, rollback, and residue audits;
- project-specific commissioning that proves unrelated runners were unchanged.

## Documentation

- [Architecture](docs/ARCHITECTURE.md)
- [Implementation plan](docs/superpowers/plans/2026-08-30-macos-runner-guard-implementation.md)
- [Security policy](SECURITY.md)
- [Contributing](CONTRIBUTING.md)

## Roadmap

1. ~~Approve the published architecture.~~ Complete.
2. Draft and independently review the detailed implementation plan; merge pending.
3. Implement the contract schema, renderer, cleanup engine, hooks, policy checks, and deterministic bundle builder through the four reviewed child plans.
4. Complete independent safety and security review.
5. Publish the first reviewed pre-release.

No live runner files, registration credentials, runner IDs, private host evidence, or project-specific secrets belong in this repository.

## License

[MIT](LICENSE)
