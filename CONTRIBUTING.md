# Contributing

Throwaway repo mirroring the JointJS Changesets release pipeline for end-to-end validation.
Two packages: `@zbynekstara-test/core` and `@zbynekstara-test/dep` (dep → core via `workspace:~`).

## Development Setup

### Prerequisites

- Node.js 22.14.0 (managed via [Volta](https://volta.sh/))
- Yarn 4.18.0

### Installation

```bash
git clone https://github.com/zbynekstara/zbynekstara-npm-test.git
cd zbynekstara-npm-test
yarn install
```

### Building

```bash
yarn dist
```

## Running Tests

```bash
yarn test
```

## Linting

```bash
yarn lint
```

## Project Structure

This is a Yarn workspace monorepo. Packages:

- `packages/core` — `@zbynekstara-test/core`, stands in for `@joint/core`
- `packages/dep` — `@zbynekstara-test/dep`, depends on core via `workspace:~`, so the
  dependency cascade (a core minor pushes `dep` out of range → `dep` gets a patch release)
  is exercised

## Pull Request Guidelines

Before submitting a PR, please verify:

- [ ] Code is up-to-date with the `master` branch
- [ ] You've successfully run `yarn test` locally
- [ ] If the change is releasable, you've added a changeset (`yarn changeset`)

### Commit Message Format

We use conventional commits. Format: `type(scope): description`

Examples:
- `fix(core): correct batch event options`
- `feat(core): add a new option`
- `docs: update contributing guide`

Types: `feat`, `fix`, `style`, `refactor`, `test`, `chore`, `example`

## Changesets

Versions and changelogs are managed by [Changesets](https://changesets.dev). Every package
keeps its own `CHANGELOG.md`, and each package is versioned independently.

If your PR changes releasable code, add a changeset:

```bash
yarn changeset
```

Pick the affected package(s) and the bump type (`patch`, `minor`, `major`), then write a
short summary. The summary is what ends up in the package's `CHANGELOG.md`, so write it for
users of the package - we use the same `scope: description` style as commit messages:

```markdown
---
"@zbynekstara-test/core": minor
---

core: add a new option to the pipeline validation entry point
```

Commit the generated `.changeset/*.md` file with your PR.

CI runs `changeset status --since=origin/master` and fails a PR that changes releasable
code in a public package without a changeset. Test-, docs-, demo- and build-config-only
changes are exempt - the exact list lives in `changedFilePatterns` in
[.changeset/config.json](.changeset/config.json). If a PR touches releasable files but
should not trigger a release, add an empty changeset:

```bash
yarn changeset add --empty
```

## Releasing (maintainers)

Releasing is automated by [.github/workflows/release.yml](.github/workflows/release.yml),
which runs on every push to `master`:

1. **Version** - while there are pending changesets, the workflow keeps a
   `changeset-release/master` PR ("Version Packages") up to date. That PR applies the
   version bumps, writes the per-package `CHANGELOG.md` entries and deletes the consumed
   changesets.
2. **Publish** - merging that PR is the release. The workflow then builds the workspace,
   runs `changeset publish` (which publishes through `yarn npm publish`), and creates a git
   tag plus a GitHub Release for every published package.

Notes:

- Private packages are never versioned or published.
- `@zbynekstara-test/dep` depends on `@zbynekstara-test/core` with a `workspace:~` range, and
  the two are a `linked` group, so a `core` release that pushes `dep` out of its dependency
  range gives `dep` a matching bump instead of a bare patch.
- Prereleases use the standard changesets pre mode: `yarn changeset pre enter beta` on
  `master`, release as usual, then `yarn changeset pre exit`.
- Snapshot releases: `yarn changeset version --snapshot` + `yarn changeset publish --tag`.
- The `prepublishOnly` guards in each package `exit 1` under raw `npm publish`, so a green
  publish job is itself proof that the pipeline went through Yarn.

## Code Style

- Run `yarn lint` before committing

## Questions?

See [README.md](README.md) for the validation plan this repo exists to run.
