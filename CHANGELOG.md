# Changelog

User-facing changes of the sandbox service, the most recent first.

## 0.1.0 - 2026-11-10

- Initial version: a REST API that creates sandboxes from Docker Hub images, reachable through the biscuit-ssh bastion
- Sandboxes: tokens restricted to the requesting address, and containers removed at token expiry or deletion
- Sandboxes: no network, bounded resources, reduced capabilities, and no privilege escalation
- Renewals: each must extend the sandbox's life by at least 30 seconds
- Configuration: the signing key must be an Ed25519 key
- Service: `--check` validates the configuration, the signing key, Docker, and the TLS certificate, then exits
