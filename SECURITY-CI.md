# Security CI (Ycash fork)

The fork's security checks, beside its tests (`.github/workflows/ycash-fork.yml`). Upstream
librustzcash's own workflows stay in the tree, scoped to upstream's `main` (each file's
"Ycash (2026-10-06)" note). Nothing below opens a pull request or an issue except Semgrep's rolling
bucket PRs and Dependabot's grouped security PRs.

| Leg | Workflow / job | Runs on | Gate | Results |
|---|---|---|---|---|
| Semgrep (Community Edition, pinned rule packs + `contrib/semgrep/rules/`) | `semgrep.yml` | PR (diff-aware), push, nightly 02:17 UTC | PR fails on a new **critical** finding | step summary, code scanning (`semgrep`), at most four rolling PRs `semgrep/<bucket>` |
| CodeQL `security-extended`, Rust + Actions, `build-mode: none` | `codeql.yml` | PR, push to `upgrade/vault`, weekly | none (alerts only) | code scanning (`/language:rust`, `/language:actions`) |
| zizmor 1.30.1 (workflow security, default persona) | `zizmor.yml` (upstream's, version pinned) | PR, push | all workflows clean at 1.30.1; `.github/zizmor.yml` lists the upstream-workflow exceptions with reasons | code scanning (`zizmor`) |
| gitleaks 8.30.1 (secrets) | `secops.yml` `gitleaks` | PR (the PR's commits), push, nightly 03:43 UTC (full history) | fails on a leak | step summary, code scanning (`gitleaks`) |
| OSV-Scanner 2.6.0 (`Cargo.lock`, `contrib/semgrep/requirements.txt`) | `secops.yml` `osv-scanner` | PR, push, nightly | fails on a known vulnerability not in `osv-scanner.toml` | step summary, code scanning (`osv-scanner`) |
| cargo-deny 0.20.2 `check advisories bans sources --locked` | `secops.yml` `cargo-deny` | PR, push, nightly | fails on an advisory not ignored in `deny.toml`, a banned crate or an unknown source | step summary, code scanning (`cargo-deny`) |
| cargo vet (supply-chain review) | `audits.yml` (upstream's) | upstream's `main` only | upstream's store | upstream |
| Dependabot (security updates only) | `.github/dependabot.yml` | GitHub | one grouped `security` PR per ecosystem (cargo, github-actions, pip); version updates off | PRs |

Tools are installed at exact releases and checked against the sha256 recorded in the workflow;
actions are pinned to full commit SHAs. The same pins are used across the Yellowback repositories.

## Ycash-specific Semgrep rules (`contrib/semgrep/rules/ycash.yaml`)

- `ycash.consensus.zcash-only-branch-id` (critical): a Zcash-only mainnet branch ID that Ycash replaced.
- `ycash.consensus.branch-id-literal-outside-definition` (high): Ycash's own branch IDs
  (`0x374d694f` Ycash, `0x6d5b7a31` Vault) written as a literal anywhere but their one definition in
  `zcash_protocol::consensus`.
- `ycash.parse.panic-in-transaction-reader` (high): `unwrap`/`expect`/`panic!`/`unreachable!` in a
  `read*`/`parse*` function of the transaction, block, transparent and encoding parsers, which read
  untrusted bytes.

## Accepting a finding

Every acceptance carries a reason a reviewer can check.

- **Semgrep**: `contrib/semgrep/accepted.json` (one finding by fingerprint, or a rule over paths), or
  `config.json` `exclude_rules` / `audit_rules` / `bucket_overrides` (`contrib/semgrep/README.md`).
- **OSV-Scanner**: an `[[IgnoredVulns]]` entry in `osv-scanner.toml` with `reason` and an
  `ignoreUntil` at most 90 days out; when it lapses the scan fails and the entry is re-checked.
- **cargo-deny**: an `{ id, reason }` entry in `deny.toml` `[advisories] ignore`, mirrored in
  `osv-scanner.toml` (its expiry is the re-check date). Prefer a `Cargo.lock` bump when a
  semver-compatible fix exists.
- **gitleaks**: a fingerprint line in `.gitleaksignore` with a `#` reason above it, only for test
  vectors and public constants. A real secret is rotated, never ignored.
- **zizmor**: a rule/file entry in `.github/zizmor.yml` with the reason in its header comment.
- **CodeQL**: dismiss the alert in code scanning with a reason.

## Repository settings

Code scanning must be enabled (free for public repositories) for the SARIF uploads. Semgrep's
rolling PRs need "Allow GitHub Actions to create and approve pull requests", and optionally the
`SEMGREP_PR_TOKEN` secret so CI runs on them (`.github/workflows/semgrep.yml` header).
