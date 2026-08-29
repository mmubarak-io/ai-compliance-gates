# Releasing and publishing to GitHub Marketplace

Maintainer runbook. Contributors do not need this.

## What GitHub actually lists

GitHub lists **one action per repository, and only the `action.yml` at the
root**. From the docs: "Repositories may include other actions metadata files
in sub-folders, but they will not be automatically listed in the marketplace."

So [`action.yml`](../action.yml) at the repository root is the listing. It runs
every gate in one step. The four actions under `.github/actions/` stay fully
usable through `uses:`, they are simply invisible to Marketplace search.
Listing them individually would mean four separate repositories, not four
listings from this one.

The root action re-invokes the engines rather than delegating to those four
actions, because `uses:` cannot take a dynamic ref. A delegating root action
tagged `v1.2.0` would still run children pinned at `@v1`, so the version an
adopter pinned would not be the version that ran, which is not a defensible
property for an audit trail. The cost of inlining is drift, and
`TestRootActionParity` is what holds the two in step: it fails the build when
the root action and its standalone twin stop passing the same flags, stop
calling the same engine, or disagree on the conftest checksum or a pinned
third-party SHA.

## One-time setup

1. The repository must be public. It is.
2. Enable two-factor authentication on the publishing account. Releases cannot
   be published to Marketplace without it.
3. Accept the GitHub Marketplace Developer Agreement. The checkbox in the
   release form stays disabled until the account owner has, and the form links
   to it.
4. The action `name` must be unique across Marketplace, must not match a
   GitHub user or organisation you do not own, must not match a Marketplace
   category, and must not use a reserved GitHub feature name.
   `AI Compliance Gates` was free at the time of writing; confirm before
   publishing:

   ```bash
   curl -s -o /dev/null -w '%{http_code}\n' \
     https://github.com/marketplace/actions/ai-compliance-gates   # 404 = free
   ```

## Every release

1. Land the change on `main`. **CI must be green**, including
   `marketplace-action`, which runs the listed action end to end: clean
   fixtures must pass and report a verifiable digest, a run given no gate at
   all must fail, and a PII-laden fixture must be blocked.
2. Update [CHANGELOG.md](../CHANGELOG.md). Verify **after** this edit, not
   before: `TestPolicyRegister` scans documentation for policy ids.
3. Re-run the suite and the checks CI cannot infer:

   ```bash
   pytest tests/ -v
   conftest test --parser hcl2 --combine -p policy infrastructure/modules/ml-platform/*.tf
   terraform fmt -check -recursive infrastructure/
   ```

4. Tag and move the floating major tag:

   ```bash
   git tag -a v1.0.1 -m "..." && git push origin v1.0.1
   git tag -f v1 && git push -f origin v1
   ```

   Releases increment the patch: `v1.0.0`, `v1.0.1`, `v1.0.2`. `v1` floats to
   the newest so `@v1` follows the line. Bump the minor only for something an
   adopter has to opt into, the major only for something that breaks a pinned
   `@v1` build. Move `v1` in the same session as the release: between the two,
   `@v1` serves the previous commit while the README documents the new one.

5. Publish the release:
   - Open the repository home page. With `action.yml` on the default branch, a
     banner offers to publish the action.
   - **Draft a release**.
   - Tick **Publish this Action to the GitHub Marketplace**.
   - Clear any metadata validation errors the form reports.
   - Primary category: **Security**. Secondary: **Continuous integration**.
   - Choose the version tag, add a title and notes, then **Publish release**.
     This prompts for 2FA.

   Publishing is immediate. GitHub does not review the action.

6. Confirm the listing renders at
   <https://github.com/marketplace/actions/ai-compliance-gates>, and that the
   README badge resolves. Only now tick the Marketplace item in the README
   roadmap: until the release is published, the badge 404s and the box is a
   claim the repository cannot back.

## Removing a listing

Marketplace listings are delisted from the repository's Marketplace settings
page. Delisting does not delete tags, and adopters pinned to `@v1` keep
working, because `uses:` resolves against the repository, not the listing.
