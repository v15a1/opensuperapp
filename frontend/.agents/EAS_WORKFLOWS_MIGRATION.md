# Migrating CI/CD from GitHub Actions to EAS Workflows

Status: IN PROGRESS. Created 2026-06-04. Phase 1 (4 workflow files) + Phase 0
Step 4 (eas.json `environment` mapping) implemented 2026-06-04 and locally
validated (YAML 1.2 parse, tag globs, eas.json JSON, test command exits 0).
Remaining: Phase 0 Steps 1-3,5 (interactive auth / dashboard / env:push /
credentials), Phase 2 validation, Phase 3 decommission, Phase 4 cleanup.

Goal: move the release pipeline off GitHub Actions and onto EAS Workflows, so
the build/update run on Expo infrastructure with EAS-injected env. This removes
the blocker we hit on GitHub Actions: `eas build`/`eas update` evaluate
`app.config.ts` on the runner, which lacks `.env`, so config eval throws
(`iosUrlScheme` validation against the placeholder `example.scheme`). On EAS,
config eval happens where the environment variables are injected.

Preserve the original four lanes:

| Lane (tag) | Channel | Native build? | Action |
|---|---|---|---|
| `dev/*` | development | no | OTA update only |
| `dev-native/*` | development | yes | native build + OTA |
| `release/*` | production | no | OTA update only |
| `release-native/*` | production | yes | native build + OTA |

## Verified facts (Expo docs, 2026-06-04)

- Workflows live in `.eas/workflows/*.yml` at the **EAS project root = `frontend/`**
  (not the repo root, unlike GitHub Actions).
- `on.push.tags` supports glob patterns → tag lanes work.
- Job types: `build`, `update`, `maestro`; custom jobs (no `type:`, use `steps:`)
  run shell commands (for jest).
- Jobs chain with `needs:`; step/job outputs via `set-output`.
- `update` job takes `environment: development|production`; `build` pulls env from
  its `eas.json` profile's `environment`. Both inject env at config-eval time on
  EAS infra — this is what fixes the blocker.
- `eas env:push <environment> --path .env` migrates the local `.env` into an EAS
  environment.

## Design decision: 4 files, one per lane

One file per lane keyed on a tag glob, rather than one file with `if`
conditionals. Reason: the docs only show `if:` equality (`github.ref_name == 'main'`),
not prefix functions like `startsWith()`, and `dev/1.0.0` won't equal `dev`. Glob
triggers are confirmed, so this is the robust choice. `dev/*` and `dev-native/*`
are distinct prefixes (`dev-native/...` starts with `dev-`, not `dev/`), so the
globs don't collide.

---

## Phase 0 — One-time setup (dashboard + CLI)

### Step 1 — Confirm EAS CLI auth
```bash
cd frontend
npx eas-cli@latest whoami      # if needed: npx eas-cli@latest login
```

### Step 2 — Connect the repo via the Expo GitHub App
- expo.dev -> project -> GitHub -> Install GitHub App -> connect `v15a1/opensuperapp`.
- Monorepo: set the **base directory** to `frontend` in the connection settings.
  This makes pushes run the workflows in `frontend/.eas/workflows/`.
- After this, pushes trigger workflows directly. No `EXPO_TOKEN` / GitHub secrets needed.

### Step 3 — Migrate `.env` into EAS environments
```bash
cd frontend
npx eas-cli@latest env:push development --path .env
npx eas-cli@latest env:push production  --path .env
```
Then in the dashboard review per-environment values (dev vs prod URLs, client IDs)
and set visibility: `secret` for tokens/keys/`FIREBASE_*_B64`, `sensitive` or
`plaintext` for the rest. Note: `EXPO_PUBLIC_*` ships in the client bundle — public
regardless of visibility setting.

### Step 4 — Map `eas.json` build profiles to environments (eas.json edit)
```jsonc
"build": {
  "development": { "developmentClient": true, "distribution": "internal",
    "channel": "development", "environment": "development",
    "env": { "EAS_BUILD_PROFILE": "development" } },
  "production": { "autoIncrement": true,
    "channel": "production", "environment": "production",
    "env": { "EAS_BUILD_PROFILE": "production" } }
}
```

### Step 5 — Ensure native build credentials exist (only for `*-native` lanes)
```bash
npx eas-cli@latest credentials   # iOS dist cert/provisioning + Android keystore
```
Android keystore can auto-generate; iOS needs the Apple account. Do this once
interactively so non-interactive workflow builds don't stall.

---

## Phase 1 — Create the four workflow files (`frontend/.eas/workflows/`)

Shared test job (runs jest on EAS):
```yaml
  test:
    steps:
      - uses: eas/checkout
      - uses: eas/install_node_modules
      - name: Run unit tests
        run: npm test -- --watchAll=false --ci --passWithNoTests
```

### `update-development.yml`
```yaml
name: Update development (OTA)
on:
  push:
    tags: ['dev/*']
jobs:
  test:
    steps:
      - uses: eas/checkout
      - uses: eas/install_node_modules
      - name: Run unit tests
        run: npm test -- --watchAll=false --ci --passWithNoTests
  update:
    needs: [test]
    type: update
    environment: development
    params:
      channel: development
      message: ${{ github.ref_name }}
```

### `update-production.yml`
Identical to above but: `tags: ['release/*']`, `environment: production`, `channel: production`.

### `release-development.yml` (native)
```yaml
name: Native release development
on:
  push:
    tags: ['dev-native/*']
jobs:
  test:
    steps:
      - uses: eas/checkout
      - uses: eas/install_node_modules
      - name: Run unit tests
        run: npm test -- --watchAll=false --ci --passWithNoTests
  build_android:
    needs: [test]
    type: build
    params: { platform: android, profile: development }
  build_ios:
    needs: [test]
    type: build
    params: { platform: ios, profile: development }
  update:
    needs: [build_android, build_ios]
    type: update
    environment: development
    params:
      channel: development
      message: ${{ github.ref_name }}
```

### `release-production.yml`
Identical to `release-development.yml` but: `tags: ['release-native/*']`,
`profile: production` (both builds), `environment: production`, `channel: production`.

---

## Phase 2 — Validate (cheapest lane first)
```bash
cd frontend
npx eas-cli@latest workflow:run update-development.yml   # manual, no tag
# or the real trigger:
git tag dev/0.0.1 && git push origin dev/0.0.1
```
Watch the Workflows page on the Expo dashboard. Validate OTA lanes (no build
minutes) before native lanes.

## Phase 3 — Decommission GitHub Actions
After the EAS lanes work, delete `.github/workflows/release.yml` (repo root).
Optional: keep a trimmed GitHub Actions workflow that runs only jest on PRs
(GitHub minutes are cheap, fast pre-merge feedback), leaving EAS for build/update/E2E.

## Phase 4 — Cleanup
```bash
git push origin :refs/tags/release/0.0.1-dev   # delete the earlier test tag
```

---

## Open items to verify on first run

1. Monorepo postinstall: `postinstall` runs `write-firebase-files` (needs
   `FIREBASE_*_B64`, now in EAS env; guarded by `|| true`) and `npm install --prefix ..`
   (repo-root package). Confirm `eas/install_node_modules` handles both with base-dir `frontend`.
2. Custom-job env: if the test job's postinstall needs vars, attach `environment:`
   to the test job too. Verify.
3. Single-file consolidation: if `if:` supports `startsWith(github.ref_name, 'dev-native/')`,
   the 4 files could collapse to 2 (one per channel). Kept 4 for safety — unconfirmed.
4. iOS non-interactive builds depend on Step 5 credentials being present.

## Sources
- https://docs.expo.dev/eas/workflows/syntax/
- https://docs.expo.dev/eas/workflows/pre-packaged-jobs/
- https://docs.expo.dev/eas/workflows/get-started/
- https://docs.expo.dev/eas/environment-variables/
- https://docs.expo.dev/eas/environment-variables/manage/
- https://docs.expo.dev/eas/workflows/automating-eas-cli/
