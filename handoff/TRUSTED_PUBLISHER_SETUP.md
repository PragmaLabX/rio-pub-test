# Setting up Trusted Publisher (OIDC) for tag-based publishing

Prerequisite: `publish.yml` is already in `.github/workflows/`, and the
package has been published to npm manually at least once
(`npm login && npm publish`) — Trusted Publisher is configured on an
already-existing package's settings.

## 1. npm — Trusted Publisher

npmjs.com → **Packages** → `<package>` → **Settings** → **Trusted
publishing** → **GitHub Actions** button:

| Field | Value |
|---|---|
| Organization or user | `<org-or-user>` |
| Repository | `<repo>` |
| Workflow filename | `publish.yml` |
| Environment name | `npm-release` |
| Allowed actions | `npm publish` |

Requirements: npm CLI >= 11.5.1 (the workflow pulls it in via
`npm install -g npm@latest`), Node.js >= 22.14.0, `permissions: id-token:
write` in the workflow — already present in `publish.yml`.

On the same page, select **"Require two-factor authentication and
disallow bypass 2fa tokens"** — this does not block automated publishing
via Actions (OIDC is a separate authentication mechanism); it only closes
off alternative manual publish paths that bypass the pipeline.

## 2. GitHub — `npm-release` Environment

Repository Settings → **Environments** → New environment → `npm-release`:

- **Required reviewers** — who must manually approve each release.
- **Deployment branches and tags** → Selected branches and tags → add
  `v*.*.*` (by default an environment only allows deployment from
  branches, not tags — the workflow won't run without this).

The name `npm-release` must match in three places: `environment:` in
`publish.yml`, the Environment name on GitHub, and the "Environment name"
field in npm's Trusted Publisher settings.

## 3. (recommended) Tag protection rule

Settings → **Rules** → **Rulesets** → New tag ruleset, pattern `v*.*.*` —
restrict who can create/push such tags. This is the only trigger for
publishing, so it shouldn't be left open to every contributor.
