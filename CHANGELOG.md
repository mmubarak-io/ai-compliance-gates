# Changelog

All notable changes to this project are documented here.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

Adopters should pin to a release tag (`@v1`). `main` may change without notice.

Releases increment the patch: `v1.0.0`, `v1.0.1`, `v1.0.2`. The floating `v1`
tag always points at the newest of them, so `@v1` follows the line without
anyone having to re-pin. A minor bump is reserved for a change that adds
something an adopter has to opt into; a major for one that breaks a pinned
`@v1` build.

## [1.0.0] - 2026-08-29

First release. Everything below is what `v1.0.0` ships. The fail-open
fixes are called out because they change gate behaviour in ways an
adopter needs to know about.

### Added: the root action and Marketplace listing

- **A root `action.yml`, so the toolkit is listable in GitHub Marketplace.**
  `uses: mmubarak-io/ai-compliance-gates@v1` now runs every gate in one step.
  GitHub lists one action per repository and only the metadata file at its
  root, so the four actions under `.github/actions/` remain usable through
  `uses:` but were never listable; nothing about them changes. The root action
  re-invokes the engines rather than delegating to them, because `uses:`
  cannot take a dynamic ref: a delegating root action tagged `v1.0.1` would
  still run children pinned at `@v1`, so the version an adopter pinned would
  not be the version that ran. `TestRootActionParity` fails the build if the
  two copies drift on flags, engine, conftest checksum or a pinned SHA.
- `evidence-artifact-name` input on the root action. Two invocations in one
  workflow run (a matrix, say) would otherwise both upload an artifact called
  `evidence-pack`, and the second upload is rejected.
- `docs/RELEASING.md`: the Marketplace publishing runbook and its
  prerequisites.

### Fixed: a run that requests no gates

- **Requesting no gates is now a misconfiguration, not a pass.** Calling
  `gates.yml` (or the new root action) with none of `data-path`,
  `model-cards-path` or `terraform-paths` skipped every gate, wrote an
  evidence pack recording three `NOT_RUN` gates and an overall `FAIL`, and
  still exited `0`, because the evidence pack deliberately leaves failing the
  build to the gates themselves. The job went green with a compliance
  artifact attached and nothing checked. Both entry points now exit `2` up
  front. If you call the workflow with no inputs today, it will start
  failing; that failure is the bug being fixed.

### Changed: every gate reports in one pass

- **Every gate now reports in one pass.** A failing gate no longer skips the
  gates after it (`!cancelled()` rather than a bare `if:`), so one run
  surfaces every violation instead of one per iteration, and the evidence
  pack stays complete precisely when something is wrong, which is when an
  auditor most needs it. Nothing that passed before fails now.
- The evidence pack and artifact upload run on `!cancelled()` instead of
  `always()`. A cancelled run no longer fabricates an audit record for work
  it abandoned.

### Fixed: gates that reported success without checking

These are the reason this release exists. Each one could produce a green
check, and an evidence pack asserting compliance, for a check that never
actually ran.

- **Scanning nothing is no longer a pass.** `pii_scanner.py` and
  `model_card_validator.py` now exit `2` (misconfigured) when the target
  resolves to zero files. Previously a mistyped `data-path` produced a
  passing gate. Pass `--allow-empty` (or the `allow-empty` action input) if
  scanning nothing is genuinely expected.
- **`--output-format json` now exits non-zero on findings.** The JSON branch
  of `pii_scanner.py` returned `0` even when PII was detected, silently
  disarming the gate for anyone consuming JSON in CI.
- **Model cards are discovered recursively.** The validator used a
  non-recursive glob, so cards in nested directories were skipped and the
  gate passed without seeing them.
- **The policy pack evaluates files together.** `conftest` now runs with
  `--combine`. Per-file evaluation made `POL-PRIV-004` fire a false positive
  on every file lacking a diagnostic setting, and made `POL-PRIV-003` pass
  silently whenever `variables.tf` wasn't in the path list.
- **Terraform paths that match nothing now fail.** The policy action exits
  `2` instead of checking an empty file list.
- **Gates that never ran are recorded.** The evidence pack lists every gate
  with an explicit `NOT_RUN`, `NO_DATA` or `MALFORMED_REPORT` status instead
  of omitting it. A crashed gate used to vanish from the artifact entirely,
  leaving a pack that read as a clean bill of health.
- **Scanning part of the target is not a pass either.** The PII scanner
  discovered `*.csv` only, so a data directory holding one CSV beside fifty
  JSONL or Parquet shards scanned the CSV, found nothing, and returned a green
  check for the whole directory. It now reads CSV, TSV, JSON, JSON Lines and
  Parquet, including `.gz`/`.bz2`/`.xz`/`.zst` variants.
- **Data the scanner cannot read blocks the gate.** Whatever is left over (a
  `.xlsx`, an `.avro`, a bare archive, a format excluded by config) is listed
  as unscanned and exits `2`. Passing would claim coverage over files the gate
  never opened. `--allow-unscanned` (action input `allow-unscanned`) records
  the gap and continues.
- **A gate that covered less than its target is `PARTIAL`** in the evidence
  pack, never `PASS`, and pulls the pack's `overall_result` with it. The
  Markdown summary names the gates involved.

### Fixed: detection accuracy

- Column-name heuristics match on token boundaries, so `name` no longer
  flags `filename` or `hostname`.
- UAE phone detection uses digit lookarounds instead of `\b`, so a long
  numeric id such as `990501234567` is no longer read as a phone number.
- Risk scores take the highest risk per column instead of summing every
  finding, which double-counted a column caught by both name and value.
- `disable_patterns` now also suppresses Presidio's built-in recognizers,
  and configured `risk` values are honoured on the Presidio path. The same
  config previously produced different results depending on whether
  Presidio happened to be installed.

### Security

- **The reference module now defaults to deny at the network layer.**
  `azurerm_storage_account` gained `network_rules { default_action = "Deny" }`
  and `azurerm_key_vault` gained `network_acls { default_action = "Deny" }`.
  `public_network_access_enabled = false` alone still left both services
  reachable through network rules that default to Allow.
- **Two new POL-SEC-001 rules** enforce exactly that, so the policy pack
  catches the gap itself instead of relying on an external scanner.
- **The conftest download is checksum-verified.** `policy-check` pins the
  SHA-256 of the release tarball and refuses to install on mismatch. Bumping
  `conftest-version` now requires supplying the matching `conftest-sha256`.
- **Action inputs are passed through `env`,** not interpolated into shell
  script bodies. Terraform path globs are expanded without `eval`, so an
  input containing shell metacharacters is treated as data.
- All composite action steps run under `set -euo pipefail`.

### Added

- `--allow-empty` flag and matching `allow-empty` action inputs.
- `--fail-on-violation` flag on the evidence pack generator.
- Config validation for `compliance.config.yml`: malformed patterns, bad
  risk values and mis-shaped schemas now produce a specific message naming
  the offending key, instead of a `KeyError` surfacing as a bogus result.
- `examples/terraform/non_compliant/`, a deliberately non-compliant
  Terraform fixture, split across two files so the cross-file rules are
  actually exercised. CI blocks on it as a negative test.
- Python 3.11, 3.12 and 3.13 CI matrix. The scripts used `str | None`
  annotations, which require 3.10+, while nothing declared a minimum at all.
- **Checksummed evidence packs.** Each pack carries a `sha256` of its own
  canonical content, and every gate records the digest of the report it was
  summarized from. `--verify <pack>` recomputes both and exits non-zero if
  either has moved, so a pack edited after generation, or paired with a
  different set of gate outputs, no longer verifies. This is a checksum, not
  a signature: it detects post-hoc edits, not who made them. Signed
  attestation stays on the roadmap.
- The `evidence-pack` action verifies the pack it just wrote and exposes
  `content-digest` and `overall-result` as step outputs, so the digest can be
  fed straight to `actions/attest-build-provenance`.
- `pii_scan.scan_extensions` in `compliance.config.yml`, to narrow which
  formats the scanner opens. Narrowing is treated as a coverage decision:
  excluded files are reported as unscanned, not silently dropped.
- A provenance note on `infrastructure/`, naming the public sources the
  `azurerm_*` arguments and UAE region names come from, and why POL-PRIV-003
  requires a concrete region to be enforceable at all.
- Community health files: `SECURITY.md`, `CODE_OF_CONDUCT.md`,
  issue and PR templates, `CODEOWNERS`, Dependabot config.

### Changed

- **Minimum Python is now 3.11.** Python 3.9 reached end of life in October
  2025, and pandas 3.x requires 3.11 or newer. A toolkit whose job is to
  assert compliance should not run its gates on an unpatched interpreter.
  The CI matrix is now 3.11, 3.12 and 3.13.
- **The composite actions install from `requirements.txt`.** They previously
  ran `pip install pandas pyyaml` with no constraint, so an adopter's gate
  used whatever version happened to be newest that day and could drift from
  the versions this project tests. The gates that only need PyYAML use the
  same file as a constraints file.
- Dependency floors stay permissive (`pandas>=2.0.0`, `pyyaml>=6.0`,
  `pyarrow>=14.0.0`). Both pandas 2.3.3 and pandas 3.0.5 pass the full suite
  on 3.12, so narrowing the floor would drop working configurations for no
  gain. `pyarrow` ships by default because without a Parquet reader a
  `.parquet` dataset counts as unscanned: honest, but it blocks the build.

### Changed (policy register)

- **Policy ids renumbered densely.** The register jumped `POL-PRIV-001`,
  `006`, `007`, `010`, which reads as though policies 002 through 005 exist
  and are being withheld. They never existed. Renumbered so the register is
  self-contained:

  | Was | Now | Policy |
  |-----|-----|--------|
  | `POL-PRIV-006` | `POL-PRIV-002` | Data Retention |
  | `POL-PRIV-007` | `POL-PRIV-003` | Data Residency (UAE) |
  | `POL-PRIV-010` | `POL-PRIV-004` | Audit Logging |

  Every other id is unchanged, and the ordering of the register is the same.
  This is done before the first announcement on purpose: policy ids are
  written into `evidence_pack.json`, which adopters retain as an audit
  record, so renumbering after adoption would break the continuity of
  somebody else's compliance evidence.

### Fixed (policy register)

- **`POL-GOV-001` was emitted by a gate but missing from the register.** The
  Rego pack can fail a build with `[POL-GOV-001] compliance_tags must declare
  ...`, yet `templates/policy_mapping.yml`, which calls itself the single
  source of truth, had no entry for it. Added, with its regulation basis and
  enforcing check.
- Added `TestPolicyRegister`, which asserts every policy id emitted anywhere
  in the toolkit is defined in the register, that no register entry is
  unenforced, and that each entry cites a regulation and a check. The drift
  that produced the bug above now fails CI.

### Changed (namespace)

- **The project moved to the `mmubarak-io` GitHub namespace.** Every
  reference a consumer touches changed with it:

  ```yaml
  uses: mmubarak-io/ai-compliance-gates/.github/workflows/gates.yml@v1
  ```

  GitHub redirects the previous owner path, but that redirect only holds
  while the old username stays unclaimed, so update any pinned reference
  rather than relying on it.

### Security (supply chain)

- **Third-party actions are pinned to commit SHAs**, not tags, in both
  `ci.yml` and the reusable `gates.yml`. A tag is mutable: whoever
  controls the upstream repository can repoint `v7` at different code, and
  every consumer picks it up silently on the next run. A toolkit that already
  verifies the conftest tarball by SHA-256 should not trust its own CI to a
  mutable reference. Each pin carries the version as a trailing comment, so
  Dependabot still tracks and proposes updates.

### Changed (actions)

- Bumped the third-party actions this project uses: `actions/checkout` 4 to 7,
  `actions/setup-python` 5 to 7, `actions/upload-artifact` 4 to 7,
  `hashicorp/setup-terraform` 3 to 4, and `aquasecurity/trivy-action` 0.33.1
  to 0.36.0. The v4 generation targets Node 20, which GitHub has deprecated
  and already force-runs on Node 24.
- **Self-hosted runner requirement.** Those major versions run on Node 24 and
  need an Actions Runner of at least 2.327.1. GitHub-hosted runners are well
  past that, but if you consume `gates.yml` on self-hosted runners,
  update them before moving to this release.
- The Trivy version stays explicitly pinned to `v0.74.0` rather than taking
  trivy-action's default, so the scan is reproducible across action bumps.

### Changed (CI)

- The Trivy job now actually scans. It had two independent faults: the step
  referenced `aquasecurity/trivy-action@0.28.0` while the tag is `v0.28.0`,
  so the job died at setup; and after fixing that, trivy-action `v0.33.1`
  defaults to trivy `v0.65.0`, which has no GitHub release, so its installer
  exited 1 and every scan step was silently skipped. Now pinned to
  trivy-action `v0.33.1` with `version: v0.74.0`, and
  `examples/terraform/non_compliant` excluded because that fixture is
  insecure by design.
- `terraform fmt -check` passes. It had never been run against
  `infrastructure/`, so the job failed and `init` and `validate` never ran.

### Changed

- The evidence pack format is version `1.2`: gates carry `report_status`,
  `report_path` and `report_sha256`; the pack carries `gates_not_run` and an
  `integrity` block; and `overall_result` can be `PARTIAL` as well as `PASS`
  or `FAIL`.
- Conftest runs once per check rather than twice; GitHub annotations are
  rendered from the JSON report.
- Reusable workflow and README examples reference `@v1` instead of `@main`.

[1.0.0]: https://github.com/mmubarak-io/ai-compliance-gates/releases/tag/v1.0.0
