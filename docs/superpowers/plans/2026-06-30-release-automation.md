# Release Automation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add `.github/workflows/release.yml` that creates a versioned GitHub Release on every conventional commit merged to `main`, and keeps a floating major tag (`v1`, `v2`, …) up to date.

**Architecture:** A single self-contained workflow file with two bash steps — one that classifies the commit and computes the next semver, and one that creates the release and updates the floating tag. No external Actions or npm tooling; everything runs with `git` and `gh` (both pre-installed on `ubuntu-latest`).

**Tech Stack:** GitHub Actions, bash, `gh` CLI, `git`

## Global Constraints

- Runner: `ubuntu-latest`
- `gh` and `git` are the only tools used — no Node, no npm, no third-party Actions beyond `actions/checkout`
- Permissions: `contents: write` (minimum required)
- Concurrency group: `release`, `cancel-in-progress: false`
- Bootstrap: if no `v*.*.*` tag exists, create `v1.0.0` exactly (no bump applied)
- Conventional commit prefixes recognised: `fix:`, `fix(…):` (patch); `feat:`, `feat(…):` (minor); any type followed by `!` OR `BREAKING CHANGE:` in footer (major)
- Non-conventional commits: exit `0` silently, no release created
- Release body: GitHub auto-generated notes (`--generate-notes`)
- Floating major tag: `v{MAJOR}` force-pushed after every release

---

### Task 1: Create the release workflow

**Files:**
- Create: `.github/workflows/release.yml`

**Interfaces:**
- Consumes: `${{ github.event.head_commit.message }}` (the full commit message including footer)
- Produces: a GitHub Release tagged `v{MAJOR}.{MINOR}.{PATCH}` and a floating tag `v{MAJOR}` on the HEAD of `main`

- [ ] **Step 1: Create `.github/workflows/release.yml`** with the following exact content:

```yaml
name: Release

on:
  push:
    branches:
      - main

permissions:
  contents: write

concurrency:
  group: release
  cancel-in-progress: false

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          fetch-depth: 0  # full history needed to list all tags

      - name: Compute next version
        id: version
        env:
          COMMIT_MSG: ${{ github.event.head_commit.message }}
        run: |
          set -euo pipefail

          # Classify commit
          if echo "$COMMIT_MSG" | grep -qE '^[a-z]+(\([^)]+\))?!:' || \
             echo "$COMMIT_MSG" | grep -qE '^BREAKING CHANGE:'; then
            BUMP=major
          elif echo "$COMMIT_MSG" | grep -qE '^feat(\([^)]+\))?:'; then
            BUMP=minor
          elif echo "$COMMIT_MSG" | grep -qE '^fix(\([^)]+\))?:'; then
            BUMP=patch
          else
            echo "Non-conventional commit — skipping release."
            echo "skip=true" >> "$GITHUB_OUTPUT"
            exit 0
          fi

          echo "Bump type: $BUMP"

          # Find latest semver tag
          LATEST=$(git tag --list 'v*.*.*' --sort=-version:refname | head -1)

          if [ -z "$LATEST" ]; then
            echo "No existing tag — bootstrapping at v1.0.0"
            NEW_TAG="v1.0.0"
          else
            echo "Latest tag: $LATEST"
            # Strip leading 'v'
            VERSION="${LATEST#v}"
            MAJOR=$(echo "$VERSION" | cut -d. -f1)
            MINOR=$(echo "$VERSION" | cut -d. -f2)
            PATCH=$(echo "$VERSION" | cut -d. -f3)

            case "$BUMP" in
              major) MAJOR=$((MAJOR + 1)); MINOR=0; PATCH=0 ;;
              minor) MINOR=$((MINOR + 1)); PATCH=0 ;;
              patch) PATCH=$((PATCH + 1)) ;;
            esac

            NEW_TAG="v${MAJOR}.${MINOR}.${PATCH}"
          fi

          echo "New tag: $NEW_TAG"
          echo "skip=false"        >> "$GITHUB_OUTPUT"
          echo "new_tag=$NEW_TAG"  >> "$GITHUB_OUTPUT"
          echo "major=v$(echo "$NEW_TAG" | cut -d. -f1 | tr -d 'v')" >> "$GITHUB_OUTPUT"

      - name: Create GitHub Release
        if: steps.version.outputs.skip != 'true'
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          NEW_TAG: ${{ steps.version.outputs.new_tag }}
        run: |
          set -euo pipefail
          gh release create "$NEW_TAG" \
            --generate-notes \
            --title "$NEW_TAG"
          echo "Created release $NEW_TAG"

      - name: Update floating major tag
        if: steps.version.outputs.skip != 'true'
        env:
          MAJOR_TAG: ${{ steps.version.outputs.major }}
        run: |
          set -euo pipefail
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
          git tag -f "$MAJOR_TAG"
          git push origin "$MAJOR_TAG" --force
          echo "Updated floating tag $MAJOR_TAG"
```

- [ ] **Step 2: Verify the file is syntactically valid YAML**

```bash
python3 -c "import yaml, sys; yaml.safe_load(open('.github/workflows/release.yml'))" && echo "YAML OK"
```

Expected: `YAML OK`

- [ ] **Step 3: Smoke-test the commit classification logic locally**

Paste and run this in bash to confirm each branch fires correctly:

```bash
classify() {
  local COMMIT_MSG="$1"
  if echo "$COMMIT_MSG" | grep -qE '^[a-z]+(\([^)]+\))?!:' || \
     echo "$COMMIT_MSG" | grep -qE '^BREAKING CHANGE:'; then
    echo major
  elif echo "$COMMIT_MSG" | grep -qE '^feat(\([^)]+\))?:'; then
    echo minor
  elif echo "$COMMIT_MSG" | grep -qE '^fix(\([^)]+\))?:'; then
    echo patch
  else
    echo skip
  fi
}

assert() { [ "$1" = "$2" ] && echo "PASS: $3" || echo "FAIL: $3 (got $1, want $2)"; }

assert "$(classify 'fix: correct typo')"              patch "plain fix"
assert "$(classify 'fix(auth): handle null')"         patch "scoped fix"
assert "$(classify 'feat: add retry logic')"          minor "plain feat"
assert "$(classify 'feat(api): new endpoint')"        minor "scoped feat"
assert "$(classify 'feat!: drop Node 14 support')"    major "feat breaking !"
assert "$(classify 'fix!: remove deprecated flag')"   major "fix breaking !"
assert "$(classify 'BREAKING CHANGE: remove v1 API')" major "BREAKING CHANGE footer"
assert "$(classify 'chore: update deps')"             skip  "non-conventional chore"
assert "$(classify 'Merge pull request #12')"         skip  "merge commit"
assert "$(classify 'docs: fix readme')"               skip  "docs — no release"
```

Expected output: 9× `PASS:`

- [ ] **Step 4: Commit**

```bash
git add .github/workflows/release.yml
git commit -m "feat: add automated release workflow on push to main"
```

- [ ] **Step 5: Push and verify the workflow appears in GitHub Actions**

```bash
git push origin main
```

Then open `https://github.com/dougis-org/cicd-tooling/actions` and confirm the `Release` workflow run appears. Since this commit message starts with `feat:`, it should:
1. Classify as `minor`
2. Find no existing tag → create `v1.0.0` (bootstrap)
3. Create a GitHub Release at `v1.0.0`
4. Push a floating `v1` tag

Verify at `https://github.com/dougis-org/cicd-tooling/releases` that `v1.0.0` exists with auto-generated release notes, and that the `v1` tag exists under `https://github.com/dougis-org/cicd-tooling/tags`.
