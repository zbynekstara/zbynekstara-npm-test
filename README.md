# zbynekstara-npm-test — release pipeline validation

A throwaway mirror of the JointJS Changesets release pipeline, used to validate the
**whole flow end-to-end against real npm + real GitHub Actions** with a **safe token**
(scoped to `@zbynekstara-test`, zero access to `@joint`) before wiring up the real repo.

It mirrors the joint structure: a `packages/` folder with two packages —

- **`@zbynekstara-test/core`** — the "core" package the `vX.Y.Z` tag + GitHub Release track.
- **`@zbynekstara-test/dep`** — depends on core via `workspace:~`, so the dependency
  cascade (a core minor/major pushes `dep` out of range → `dep` republishes) is exercised.

The pipeline (identical logic to joint, package names swapped):

- `.changeset/` — Changesets config (`changelog: false`; we render our own).
- `scripts/release-info.mjs`, `scripts/changelog-from-changesets.mjs` — versioning info + CHANGELOG/RELEASE_NOTES renderer.
- `package.json` scripts: `release-status`, `release-version`, `release-publish`.
- `.github/workflows/`: `test-pr.yml` (changeset gate / required check), `release.yml` (Phase 1), `publish.yml` (Phase 2).

---

## One-time setup

### 1. Own the npm scope + a SAFE token
- Make sure you control the **`@zbynekstara-test`** scope on npm — either it's your username, or create a **free npm org** named `zbynekstara-test` (free orgs allow unlimited *public* packages).
- Create a **granular access token** scoped to **only `@zbynekstara-test`**, **Read and write**, **Bypass 2FA** checked, with an expiry. This token cannot touch `@joint`, so it's safe to test with.

### 2. Create the GitHub repo (public → everything is free)
```bash
cd ~/code/zbynekstara-npm-test
gh repo create <you>/zbynekstara-npm-test --public --source=. --remote=origin --push
```
- Default branch must be **`master`** (the workflows use it). If GitHub made it `main`, rename to `master` (Settings → Branches, or `git push origin master && gh repo edit --default-branch master`).

### 3. Secrets (Settings → Secrets and variables → Actions)
- `NPM_TOKEN` = the safe granular token from step 1.
- **GitHub App** (recommended, mirrors joint): create a free App with **contents: write** + **pull-requests: write**, install it on this repo, and add `RELEASE_APP_ID` + `RELEASE_APP_PRIVATE_KEY`.
  - *Simpler PAT alternative:* replace the `Generate GitHub App token` step in both workflows with a classic PAT — set `token`/`GH_TOKEN` to `${{ secrets.RELEASE_TOKEN }}` and add that secret. (A non-`GITHUB_TOKEN` credential is required so the opened PR triggers `test-pr.yml`.)

### 4. Repo settings
- **Allow auto-merge**: Settings → General → Pull Requests → check "Allow auto-merge".
- **Branch protection** on `master`: Settings → Branches → add rule → require the **`test`** status check (so auto-merge waits for CI). Allow the App/PAT to bypass if needed.

---

## Validate

### A. Stable release (`latest`)
1. Actions → **Release (prepare)** → Run workflow → `dist_tag: latest`.
2. Watch: a `release/pending` PR opens, `test-pr` runs (proves the App/PAT triggers CI), and it **auto-merges** on green.
3. Merge triggers **Release (publish)**. Confirm it:
   - publishes to npm — `npm view @zbynekstara-test/core` and `@zbynekstara-test/dep`;
   - resolves the workspace range — `npm view @zbynekstara-test/dep dependencies` shows `@zbynekstara-test/core: ~1.1.0` (**not** `workspace:~`);
   - cut the tag `v1.1.0` and a GitHub Release;
   - created/updated the **`prod`** branch;
   - the `prepublishOnly` guards did **not** block it (they'd have `exit 1` under raw `npm publish`).

### B. Core-unchanged case
Add a changeset that bumps only `dep` (`yarn changeset` → pick `dep`, patch), commit to `master`, then run **Release (prepare)** again. Confirm: `dep` republishes, **no** new tag/Release is cut, but `prod` still advances.

### C. Prerelease channel
Run **Release (prepare)** with `dist_tag: alpha`. Confirm packages publish under the `alpha` dist-tag (`npm view @zbynekstara-test/core dist-tags`) and the GitHub Release is marked **pre-release**.

---

## Local smoke test (no npm, no token)
```bash
yarn install
yarn release-status                     # shows the pending plan
node scripts/release-info.mjs           # release_title / core_version / released
node scripts/changelog-from-changesets.mjs --dry-run   # preview CHANGELOG + RELEASE_NOTES
```

## Cleanup when done
```bash
npm unpublish @zbynekstara-test/core --force   # within 72h of publish
npm unpublish @zbynekstara-test/dep --force
gh repo delete <you>/zbynekstara-npm-test --yes
```

Once this all passes, wire the real `@joint`-scoped `NPM_TOKEN` into `clientIO/joint` with confidence.
