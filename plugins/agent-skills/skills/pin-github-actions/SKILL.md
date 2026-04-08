---
name: pin-github-actions
description: >
  Audit and pin GitHub Actions to commit SHAs with version comments, and update outdated actions to their latest releases.
  Use this skill whenever the user asks about GitHub Actions versions, wants to pin actions to SHAs, check for outdated
  actions, update CI workflows, harden supply chain security, or audit workflow dependencies. Also trigger when the user
  mentions "pin actions", "update actions", "latest action versions", "SHA pinning", or anything related to GitHub Actions
  version management -- even if they don't use those exact words.
---

# Pin GitHub Actions

Audit GitHub Actions workflows, check for outdated versions, and pin all external actions to commit SHAs with version comments for supply chain security.

## Why this matters

Mutable version tags (like `@v4`) can be force-pushed by action maintainers (or attackers who compromise the repo). Pinning to a commit SHA ensures you run exactly the code you audited. The version comment preserves human readability.

## Workflow

### Step 1: Discover workflow files

Find all GitHub Actions workflow and composite action files:

```
.github/workflows/*.yml
.github/workflows/*.yaml
.github/actions/**/*.yml
.github/actions/**/*.yaml
```

Use the Glob tool for this.

### Step 2: Extract external action references

Read each file and extract all `uses:` lines. Categorize them:

- **External actions**: `owner/repo@ref` -- these need pinning
- **Local actions**: `./.github/actions/...` -- skip these
- **Reusable workflows**: `./.github/workflows/...` -- skip these

For actions already pinned to a SHA (40-char hex string), extract the version from the trailing comment (e.g., `# v4.0.0`) if present. These still need checking for newer versions.

### Step 3: Check latest versions via `gh` CLI

For each unique external action, fetch the latest release tag:

```bash
gh api repos/{owner}/{repo}/releases/latest --jq '.tag_name'
```

If the action has no releases (404), try fetching the latest tag instead:

```bash
gh api repos/{owner}/{repo}/tags --jq '.[0].name'
```

### Step 4: Compare versions

Compare the current version (from the tag or SHA comment) against the latest release. Categorize each action as:

- **Up to date**: Major version matches latest (e.g., `v6` matches `v6.3.0`)
- **Outdated**: Major version is behind (e.g., `v3` vs `v4.0.0`)
- **Already pinned**: SHA-pinned and on the latest major version

Present a summary table to the user showing all actions, their current and latest versions, and status. For example:

```
| Action | Current | Latest | Status |
|--------|---------|--------|--------|
| actions/checkout | v6 | v6.0.2 | OK |
| docker/setup-buildx-action | v3 | v4.0.0 | OUTDATED |
```

### Step 5: Handle outdated actions

If any actions have outdated major versions, use the AskUserQuestion tool to confirm whether the user wants to bump them to the latest major version before pinning. Present the specific actions and version changes clearly.

Some major version bumps may have breaking changes -- mention this when asking for confirmation. Group all outdated actions into a single confirmation prompt rather than asking one at a time.

### Step 6: Resolve commit SHAs

For each action (at its target version), resolve the commit SHA:

1. Get the ref SHA:
   ```bash
   gh api repos/{owner}/{repo}/git/ref/tags/{tag} --jq '.object.sha'
   ```

2. Check if it's an annotated tag (which points to a tag object, not a commit) by attempting to dereference:
   ```bash
   gh api repos/{owner}/{repo}/git/tags/{sha} --jq '.object.sha'
   ```
   - If this returns a SHA: use the dereferenced SHA (it's the actual commit)
   - If this returns 404: the original SHA is already the commit SHA

This distinction matters because GitHub Actions resolves annotated tags differently, and pinning to the tag object SHA instead of the commit SHA can cause failures.

### Step 7: Apply pins

Replace each `uses:` reference with the pinned format:

```yaml
# Before
uses: actions/checkout@v6

# After
uses: actions/checkout@de0fac2e4500dabe0009e67214ff5f5447ce83dd # v6.0.2
```

Use the Edit tool with `replace_all: true` when the same action appears multiple times in a single file.

For actions that were already pinned to a SHA but had an outdated version, update both the SHA and the version comment.

### Step 8: Report

Present a final summary of all changes made:
- How many actions were pinned
- How many were updated to newer versions
- Any actions that couldn't be resolved (and why)

### Step 9: Offer a conventional commit

After presenting the report, offer to create a single, clean [Conventional Commits](https://www.conventionalcommits.org) commit for the changes. Propose a commit message based on what was done. The commit type should reflect the nature of the change:

- If only pinning (no version bumps): `ci: pin github actions to commit SHAs`
- If versions were bumped: `ci: update and pin github actions to commit SHAs`
- If a mix, lean toward the more descriptive message

Include a body that lists the actions that were changed. For example:

```
ci: pin github actions to commit SHAs

Pin external GitHub Actions to commit SHAs for supply chain security.

Actions pinned:
- actions/checkout@v6.0.2
- docker/setup-buildx-action@v4.0.0
```

Use the AskUserQuestion tool to confirm with the user before committing. Show them the proposed message and let them adjust it. If they decline, that's fine -- the file changes are already applied.

## Edge cases

- **Actions with no GitHub releases**: Fall back to tags. If no tags either, warn the user and skip.
- **Pre-release versions**: Ignore pre-releases; only use stable releases. Check the `prerelease` field if unsure.
- **Monorepo actions** (e.g., `aws-actions/amazon-ecr-login`): These work fine with the standard flow since each action repo has its own releases.
- **Actions pinned to a branch** (e.g., `@main`): Flag these to the user as needing attention -- they should be pinned to a specific version first.
- **Docker container actions** (e.g., `uses: docker://image:tag`): Skip these entirely.
