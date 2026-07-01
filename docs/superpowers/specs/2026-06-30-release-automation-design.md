---
name: release-automation
description: Automated GitHub releases on merge to main, driven by conventional commits, with semver tagging and floating major tags
metadata:
  type: project
---

# Release Automation Design

## Goal

Generate a GitHub Release automatically whenever a conventional commit lands on `main`, enabling consumers of `cicd-tooling` workflows and actions to pin to a specific version (e.g. `@v1.2.3` or `@v1`) rather than `@main`.

## Trigger

A new workflow `.github/workflows/release.yml` fires on `push` to `main`.

## Commit Parsing

The HEAD commit message is inspected to determine the bump type:

| Commit prefix | Bump type |
|---|---|
| `fix:` / `fix(…):` | patch |
| `feat:` / `feat(…):` | minor |
| `feat!:` / `fix!:` / `BREAKING CHANGE:` in footer | major |
| anything else | skip — exit 0, no release |

Classification is performed with `grep -E` against `${{ github.event.head_commit.message }}`. Non-conventional commits produce no release and no error.

## Version Computation

1. Fetch the latest existing semver tag: `git tag --list 'v*.*.*' --sort=-version:refname | head -1`
2. If no tag exists: the new version is exactly `v1.0.0` (no bump applied). This covers the initial bootstrap.
3. If a tag exists: split `vMAJOR.MINOR.PATCH` and apply the bump:
   - `patch` → increment PATCH
   - `minor` → increment MINOR, reset PATCH to 0
   - `major` → increment MAJOR, reset MINOR and PATCH to 0

## Release Creation

```
gh release create <new-tag> --generate-notes --title "<new-tag>"
```

This creates a GitHub Release, tags the HEAD commit, and populates the release body with GitHub's auto-generated notes (PRs and commits since the previous release).

## Floating Major Tag

After the release is created, the floating major tag is force-updated:

```bash
git tag -f v<MAJOR>
git push origin v<MAJOR> --force
```

This allows consumers to pin to `@v1` and automatically receive all non-breaking updates in that major line.

## Permissions & Concurrency

```yaml
permissions:
  contents: write  # create releases, push tags

concurrency:
  group: release
  cancel-in-progress: false  # never interrupt an in-flight release
```

`cancel-in-progress: false` prevents a race condition where a second push could cancel a release mid-flight, leaving a tag without its floating major pointer.

## Files Changed

- **New:** `.github/workflows/release.yml`

No other files are added or modified.

## Out of Scope

- Changelog file committed to the repo (release notes live on the GitHub Release object only)
- NPM / package versioning (no `package.json` in this repo)
- Pre-release or draft releases
