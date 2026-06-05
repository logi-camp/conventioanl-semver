# conventional-semver

A GitHub Action that calculates the next [semantic version](https://semver.org) from your git history using [Conventional Commits](https://www.conventionalcommits.org). No Node.js, no Docker — pure bash.

## Usage

```yaml
- uses: actions/checkout@v4
  with:
    fetch-depth: 0  # required: full history + tags

- uses: logi-camp/conventioanl-semver@v1
  id: semver

- run: echo "Next version is ${{ steps.semver.outputs.version_tag }}"
```

> `fetch-depth: 0` is required. The default shallow clone has no tags or history, so the action cannot determine the previous version.

## Inputs

| Input | Description | Default |
|---|---|---|
| `prefix` | Tag prefix | `v` |
| `default_bump` | Bump type when no conventional commits are found (`major` / `minor` / `patch` / `none`) | `patch` |
| `initial_version` | Starting version when the repo has no tags yet | `0.0.0` |

## Outputs

| Output | Description | Example |
|---|---|---|
| `version` | Next semantic version | `1.4.0` |
| `version_tag` | Next version with prefix | `v1.4.0` |
| `previous_version` | Previous version without prefix | `1.3.0` |
| `previous_version_tag` | Previous version tag, empty if none existed | `v1.3.0` |
| `changelog` | Markdown changelog grouped by commit type | _(see below)_ |
| `bump` | Bump type applied | `minor` |
| `last_commit` | SHA of HEAD at calculation time | `49d6690...` |

## Version bump rules

| Commit pattern | Bump |
|---|---|
| `type!: …` or `type(scope)!: …` | **major** |
| `BREAKING CHANGE:` in commit body | **major** |
| `feat: …` / `feat(scope): …` | **minor** |
| `fix: …` / `fix(scope): …` | **patch** |
| `perf: …` / `perf(scope): …` | **patch** |
| anything else | value of `default_bump` input |

The highest bump found across all commits since the last tag wins (major > minor > patch).

## Changelog format

The `changelog` output is a markdown string grouped by commit type:

```markdown
## Breaking Changes
- feat!: redesign API response format

## Features
- feat: add dark mode toggle
- feat(api): add pagination support

## Bug Fixes
- fix: resolve null pointer on login

## Performance Improvements
- perf: cache database queries
```

## Full workflow example

```yaml
name: Release

on:
  push:
    branches: [main]

jobs:
  release:
    runs-on: ubuntu-latest
    permissions:
      contents: write

    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - uses: logi-camp/conventioanl-semver@v1
        id: semver

      - name: Create tag
        if: steps.semver.outputs.bump != 'none'
        run: |
          git tag ${{ steps.semver.outputs.version_tag }}
          git push origin ${{ steps.semver.outputs.version_tag }}

      - name: Create GitHub Release
        if: steps.semver.outputs.bump != 'none'
        uses: softprops/action-gh-release@v2
        with:
          tag_name: ${{ steps.semver.outputs.version_tag }}
          body: ${{ steps.semver.outputs.changelog }}
```
