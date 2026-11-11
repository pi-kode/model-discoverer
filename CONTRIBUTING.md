# Contributing to @pi-kode/model-discoverer

Thank you for contributing! This guide covers local development, testing, coding conventions, and the release workflow.

## Prerequisites

- **Node.js** `>= 22`
- **Earendil Pi** `1.5.2+` (for manual end-to-end testing in Pi)

## Local setup

Clone the repository and install dev dependencies without running lifecycle scripts:

```sh
git clone https://github.com/pi-kode/model-discoverer.git
cd model-discoverer
npm install --ignore-scripts
```

To test the extension inside Pi from your local checkout:

```sh
# Install locally into Pi
pi install .

# Or run a one-off session without installing
pi -e .
```

After making changes, run `/reload` inside Pi to pick up updated TypeScript sources.

## Project structure

- `src/index.ts` — Extension entrypoint: reads `models.json`, registers provider refresh hooks, commands (`/discover-models`, `/configure-model`), and lifecycle handlers (`session_start`, `session_shutdown`).
- `src/models-json.ts` — Parses `discoverModels: true` entries from `models.json` (with JSONC `//` comment and trailing comma stripping).
- `src/discovery.ts` — Fetches and normalizes custom OpenAI/Anthropic `/models` catalogs, enforcing size/page/model bounds, pagination rules, and sanitized errors.
- `src/refresh-hook.ts` — Bridges Pi's `refreshModels` hook: offline cache restoration and online catalog persistence.
- `src/refresh.ts` — Orchestrates refreshes across native and custom providers via Pi's `ModelRegistry`.
- `src/configure-model.ts` — Interactive `/configure-model` flow and `applyModelOverride` helper for editing `modelOverrides` in `models.json`.
- `src/validate.ts` — Shared validation helpers (`record`, `endpointUrl`).
- `test/*.test.ts` — Unit and loader tests using `node:test` and `tsx`.

## Running checks

Run the test suite and TypeScript type checker before opening a pull request:

```sh
npm test
npm run typecheck
```

Tests use mocked HTTP (`fetch`) and temporary config directories; they do not read your saved Pi credentials or make real network requests.

To inspect the exact tarball contents that will be published to npm:

```sh
npm pack --dry-run
```

## Coding & security guidelines

- **Strict TypeScript & ESM**: Keep `tsconfig.json` strictness intact and use explicit `.ts` extensions on relative imports.
- **No secret leakage**: Never include raw config values, API keys, headers, URLs, or upstream response bodies in thrown error messages or UI notifications.
- **Network safety**: Preserve HTTPS-only (plus loopback HTTP) validation, same-origin checks between `baseUrl` and `modelsUrl`, `redirect: "error"`, and `AbortSignal` propagation.
- **Minimal dependencies**: Avoid adding runtime dependencies; the extension runs directly against Node 22+ built-ins and Pi's peer dependencies (`@earendil-works/pi-ai`, `@earendil-works/pi-coding-agent`).
- **Tests for behavior changes**: Add or update tests in `test/` for any new capability, edge case, or bug fix.

## Pull requests

1. Create a feature or fix branch from `main`.
2. Keep changes focused and well-tested (`npm test && npm run typecheck`).
3. Open a pull request against `main`; the CI workflow (`.github/workflows/ci.yml`) will automatically run tests and type checks.

## Releasing & publishing

Packages are published to the `@pi-kode` npm organization (`https://www.npmjs.com/settings/pi-kode/packages`) automatically by `.github/workflows/publish.yaml` using **npm OIDC Trusted Publishing** (no `NPM_TOKEN` secret required).

Publishing is triggered **only** when a tag matching `vX.X.X-release` (for example `v0.2.0-release`) is pushed to GitHub.

### Release steps

1. Bump `"version"` in `package.json` (and keep `package-lock.json` in sync), e.g. `0.2.0`, and merge the version bump into `main`.
2. Create and push an annotated tag matching `v<version>-release`:

   ```sh
   git tag -a v0.2.0-release -m "Release v0.2.0"
   git push origin v0.2.0-release
   ```

3. The `Publish` workflow runs the `verify` job to validate that:
   - the tag strictly matches `^v[0-9]+\.[0-9]+\.[0-9]+-release$`,
   - the tag version matches `package.json` `"version"`,
   - the package name is scoped under `@pi-kode/`,
   - `npm test` and `npm run typecheck` pass.
4. Once `verify` passes, the `publish` job pauses at the `npm-publish` GitHub Actions environment. Open the workflow run in GitHub Actions and click **Review deployments** → **Approve and deploy** to run `npm publish --provenance --access public`.

### One-time release setup (maintainers)

1. **GitHub Actions environment** (in repository **Settings → Environments**):
   - Create an environment named `npm-publish`.
   - Enable **Required reviewers** and add the maintainer(s) authorized to approve releases.
2. **npm Trusted Publishing** (in package settings for `@pi-kode/model-discoverer` under `https://www.npmjs.com/settings/pi-kode/packages`):
   - **Organization / user**: `pi-kode`
   - **Repository**: `model-discoverer`
   - **Workflow filename**: `publish.yaml`
   - **Environment**: `npm-publish`
