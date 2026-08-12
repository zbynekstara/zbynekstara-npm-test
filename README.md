# zbynekstara-npm-test — release pipeline validation

A throwaway mirror of the JointJS Changesets release pipeline, used to validate the
**whole flow end-to-end against real npm + real GitHub Actions** with a **safe token**
(scoped to `@zbynekstara-test`, zero access to `@joint`) before wiring up the real repo.

It mirrors the joint structure: a `packages/` folder with two packages —

- **`@zbynekstara-test/core`** — stands in for `@joint/core`.
- **`@zbynekstara-test/dep`** — depends on core via `workspace:~`, so the dependency
  cascade (a core minor/major pushes `dep` out of range → `dep` gets a patch release) is
  exercised.

The pipeline (identical logic to joint, package names swapped):

- `.changeset/config.json` — Changesets config. Default changelog generator
  (`@changesets/cli/changelog`), so **each package keeps its own `CHANGELOG.md`**;
  `changedFilePatterns` defines what counts as releasable.
- `.github/workflows/release.yml` — the whole release, on every push to `master`.
  One workflow, three jobs: `select-mode` → `version` **or** `publish`.
- `.github/workflows/test-pr.yml` — CI plus the `changeset status` gate.
- No custom release scripts and no root `CHANGELOG` — the official
  `changesets/action` sub-actions do the versioning, publishing, tagging and GitHub
  Releases.

## How the release works

`release.yml` runs on every push to `master` and `changesets/action/select-mode@v2`
picks the mode:

| Repo state | Mode | What happens |
| --- | --- | --- |
| Pending changesets in `.changeset/` | `version` | Opens/updates the `changeset-release/master` PR titled **Version Packages**: version bumps, per-package `CHANGELOG.md` entries, consumed changesets deleted. |
| `master` versions ahead of npm | `publish` | Builds (`yarn dist`), runs `changeset publish` (→ `yarn npm publish`), then a git tag + GitHub Release **per published package**. |
| Neither | `none` | Nothing. |

Nothing reaches npm until a maintainer merges the Version Packages PR.

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

### 3. Secret (Settings → Secrets and variables → Actions)
- `NPM_TOKEN` = the safe granular token from step 1. That's the **only** secret needed —
  no GitHub App, no PAT (the sub-actions use the built-in `GITHUB_TOKEN`).

### 4. Repo settings
- **Settings → Actions → General → "Allow GitHub Actions to create and approve pull
  requests"** must be **enabled**, otherwise the `version` job cannot open the release PR.
- **Branch protection:** leave `test` *un*required on `master`, or give the release PR an
  explicit bypass. The release PR is opened by `github-actions[bot]` using `GITHUB_TOKEN`,
  and GitHub does not fire `pull_request` workflows for it — so `test-pr.yml` never runs on
  that PR and a required `test` check would block the merge forever. **Confirming this
  behaviour is one of the things this repo is here to verify** (it is the main practical
  difference from the previous GitHub-App-token pipeline, which did trigger CI on the
  release PR).

---

## Validate

### A. Stable release (`latest`)
1. Land a changeset on `master` (there is one committed already — `yarn changeset` to add more).
2. **Release** runs in `version` mode. Confirm it opens **Version Packages**
   (`changeset-release/master`) and that the PR diff contains the bumps **and** a new
   `packages/*/CHANGELOG.md` entry per package.
3. Merge the PR. **Release** runs again, now in `publish` mode. Confirm it:
   - publishes to npm — `npm view @zbynekstara-test/core` and `@zbynekstara-test/dep`;
   - resolves the workspace range — `npm view @zbynekstara-test/dep dependencies` shows `@zbynekstara-test/core: ~1.1.0` (**not** `workspace:~`);
   - cut a git tag **per package** (`@zbynekstara-test/core@1.1.0`, …) and a GitHub Release for each;
   - the `prepublishOnly` guards did **not** block it (they'd have `exit 1` under raw `npm publish`, so a green job proves Yarn was used).

### B. Dependency cascade / core-unchanged case
Add a changeset that bumps only `dep` (`yarn changeset` → pick `dep`, patch) and land it on
`master`. Confirm: `dep` republishes with its own tag + Release, `core` is untouched, and no
`core` tag is cut. Then do the inverse — a `core` **minor** — and confirm `dep` gets an
automatic patch ("Updated dependencies") because its `workspace:~` range had to move.

### C. The changeset gate
Open a PR that edits `packages/core/index.js` with **no** changeset → `test-pr.yml` must
fail on **Check changeset**. Add `yarn changeset add --empty` → it must pass. Then edit only
a `.md` file with no changeset → must also pass (excluded by `changedFilePatterns`).

### D. Prerelease channel
On `master`: `yarn changeset pre enter beta`, commit, land a changeset. Versions become
`x.y.z-beta.N` and `changeset publish` puts them on the **`beta`** npm dist-tag
(`npm view @zbynekstara-test/core dist-tags`) with the GitHub Release marked
**pre-release**. Leave with `yarn changeset pre exit`.

---

## Local smoke test (no npm, no token)
```bash
yarn install
yarn changeset status                    # shows the pending release plan
yarn changeset status --since=origin/master   # exactly what CI's gate runs
yarn changeset version                   # apply bumps + CHANGELOG.md locally, then `git checkout .`
```

## Cleanup when done
```bash
npm unpublish @zbynekstara-test/core --force   # within 72h of publish
npm unpublish @zbynekstara-test/dep --force
gh repo delete <you>/zbynekstara-npm-test --yes
```

Once this all passes, wire the real `@joint`-scoped `NPM_TOKEN` into `clientIO/joint` with confidence.
