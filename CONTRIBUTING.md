# Contributing

Throwaway repo mirroring the JointJS Changesets release pipeline for end-to-end validation.
Two packages: `@zbynekstara-test/core` and `@zbynekstara-test/dep` (dep → core via `workspace:~`).

## Changesets

Versioning and the changelog are driven by [Changesets](https://github.com/changesets/changesets).
If your PR changes releasable code, add a changeset:

```bash
yarn changeset
```

Pick the affected package(s) and bump type, then write the changelog line(s) in the
**same `type(scope): description` form as commits** — one row per line:

```markdown
---
"@zbynekstara-test/core": minor
---
feat(core): add a thing
fix(core): allow a thing
```

Commit the generated `.changeset/*.md` with your PR. CI (`changeset status`) fails a PR that
changes releasable code without a changeset; docs-, test-, and demo-only PRs are exempt. Don't
add PR/commit links by hand — the GitHub Release notes link each row to its `master` commit
automatically. `feat` rows render verbatim; `fix` rows are prefixed with `fix to `. Non-`feat`/`fix`
lines are ignored by the changelog renderer, and `!` before the `:` marks a breaking change. Only
`@zbynekstara-test/core` and `@zbynekstara-test/dep` appear in the root `CHANGELOG`; any other
publishable packages are still versioned and published, just not listed there.

## Releasing (maintainers)

Releases are automated in two phases; no manual npm/tag/GitHub-Release steps.

1. **Prepare** — dispatch the **Release (prepare)** workflow (`release.yml`), choosing the `dist_tag`
   (`latest` | `alpha` | `beta`). It renders the root `CHANGELOG` + `RELEASE_NOTES.md` from the pending
   changesets, applies the bumps (`changeset version`), and opens an auto-merging `release/pending` PR
   to `master`.
2. **Publish** — merging that PR triggers **Release (publish)** (`publish.yml`), which publishes the
   changed packages to npm under the chosen `dist_tag` (via Yarn), tags `vX.Y.Z` + cuts one GitHub
   Release when `@zbynekstara-test/core` changed, and force-updates `prod` to the merged commit.

The `dist_tag` input is the release channel: `latest` publishes a stable Release; `alpha`/`beta`
publish a prerelease under that npm tag. To also get prerelease *version numbers* (`x.y.z-beta.N`),
run `yarn changeset pre enter <tag>` on `master` before step 1 (and `yarn changeset pre exit` to
return to stable) — that affects the numbering only, not the channel.
