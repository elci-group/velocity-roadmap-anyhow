# Roadmap: Replace `anyhow` in Velocity

## Status

**Draft** — awaiting prioritization.

## Bottom line

`anyhow` is used in 12 files across Velocity, mostly for ad-hoc error propagation and the `anyhow!("...")` convenience macro. Replacing it removes a CVE-flagged, broad-error dependency and replaces it with typed errors that are easier to diagnose in production.

This is the highest-effort roadmap of the three because error types touch almost every async boundary and CLI surface.

## Quick facts

| | |
|---|---|
| Crate | `anyhow` v1.0.104 |
| Amber recommendation | `security_block` (1 CVE, broad usage) |
| Amber classification | Security Critical |
| Affected files | 12 |
| Call sites | 37 |
| Unique APIs used | 2 (`anyhow::Result`, `anyhow!`) |

## Documents

- [`ROADMAP.md`](./ROADMAP.md) — detailed migration plan, risks, and acceptance criteria.
