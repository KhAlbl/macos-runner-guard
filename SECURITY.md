# Security Policy

## Supported versions

The project is in its architecture phase and has no supported executable release yet. Security guarantees described in the architecture are design requirements, not claims about deployed code.

## Reporting a vulnerability

Please report suspected vulnerabilities privately through [GitHub private vulnerability reporting](https://github.com/KhAlbl/macos-runner-guard/security/advisories/new).

Do not include registration credentials, personal access tokens, runner credential files, keychain contents, SSH material, cloud credentials, private repository contents, or sensitive host evidence in a public issue.

## Trust boundary

This project is intended to reduce accidental residue and cross-project exposure on persistent macOS hosts. It does not provide VM-level containment, physical-host isolation, or a safe execution boundary for malicious same-user code.
