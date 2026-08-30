# macOS Runner Guard

Secure-by-default building blocks for project-dedicated, self-cleaning GitHub Actions runners on shared macOS hosts.

> [!IMPORTANT]
> This repository currently contains the approved architecture and draft implementation plans under final consistency review. It does not yet provide an installable runner controller or production release.

## Why this project

GitHub's official self-hosted runner is persistent. Reusing one macOS host across jobs and repositories can leave workspaces, caches, build products, and altered state behind. macOS Runner Guard is designed to add deterministic residue cleanup, project-specific routing, bounded host checks, reproducible adoption bundles, and explicit trust boundaries without replacing the official runner.

## Planned profiles

- **Dedicated account — recommended:** one non-admin macOS account and one organization-scoped runner in a project-dedicated restricted runner group per independent trust domain.
- **Shared account, trusted repositories only:** separate labels and roots under one account for routing and cleanup reliability. This profile does not provide credential or privacy isolation between those repositories.

Persistent cleanup is residue control, not VM isolation. Version 1 accepts persistent adoption only for a private organization-owned repository whose dedicated runner group is selected to that repository and restricted to one protected-default-branch workflow. This public project and other public or user-owned repositories remain GitHub-hosted-only. Untrusted code, signing, deployment credentials, and hostile-code workloads should use an ephemeral VM or dedicated host.

## Planned capabilities

- project-dedicated organization registrations, restricted runner groups, and unique routing labels;
- root-protected, per-project controller generations;
- automatic before-job, always-run, and after-job cleanup;
- descriptor-relative, no-follow filesystem operations;
- fail-closed workflow policy checks;
- deterministic toolkit and per-instance ZIP bundles;
- transactional activation, rollback, and residue audits;
- project-specific commissioning that proves unrelated runners were unchanged.

Version `0.1.0` deliberately disables the official runner's automatic application update so the executable bytes remain bound to one reviewed GitHub archive. GitHub requires a disabled-update runner to move to each available release within 30 days, or immediately when a critical security update is required. This first version therefore treats a runner release change as planned replacement: stop and decommission the old registration, quarantine its retained instance, and adopt the reviewed new archive through a wholly new contract with a fresh project slug, label, runner-name prefix, instance/runner/work roots, service identity, and operations namespace. The dedicated account and project-specific runner group may be reused only after fresh evidence proves the old service, registration, Listener, and Worker absent and the selected-workflow policy still exact. In-place binary upgrade and same-namespace reuse are deferred; restoring a quarantine never returns it to service. This is an operational limitation of the first pre-release, not a permanent design goal.

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
