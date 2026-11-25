# Support Policy & LTS Lifecycle

This document outlines the support policy and maintenance lifecycle for **Gaet**.

---

## Active LTS Releases

| Version | Engine | Status | Support Window | Policy |
|:---|:---|:---|:---|:---|
| **Latest release** | Go | **Recommended (Active LTS)** | Aug 2026 – Aug 2027+ | Primary production engine & active feature development. |
| **`v1.0.0 LTS`** | Python | **Alternative (Legacy)** | Aug 2026 – Aug 2027 | Maintenance mode. Security & critical bug fixes only. |

---

## Engine Roadmap & Transition Policy

- **Go Engine (latest release)**: Designated as the primary engine for Gaet. All future feature development, performance improvements, and active maintenance take place on the Go single-binary architecture.
- **Python Engine (`v1.0.0 LTS`)**: Remains supported as a legacy alternative until **August 2027**. After August 2027, the Python implementation reaches End-of-Life (EOL), and users will be required to run the Go binary.

---

## Scope of Support for the Go Engine

During its active LTS lifecycle, the Go Engine receives:

1. **Security Patches**: Rapid patch updates (`vX.Y.Z`) for credential handling or security issues.
2. **Bug Fixes**: Fixes for CLI errors, OS compatibility issues (Linux, macOS, Windows), or sync failures.
3. **PostgreSQL Compatibility**: Maintenance updates for new PostgreSQL server versions and client tools (`psql`, `pg_dump`, `pg_restore`).

---

## Reporting Issues

- **Bug Reports**: Submit issues via [GitHub Issues](https://github.com/ghaniyyrahmans/gaet/issues).
- **Security Disclosures**: Refer to [SECURITY.md](SECURITY.md) for vulnerability reporting.
