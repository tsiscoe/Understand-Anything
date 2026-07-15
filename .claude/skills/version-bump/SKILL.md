---
name: version-bump
description: >-
  Bumps the version field across all six version-carrying manifests in this
  repo in lockstep, so no platform (Claude/Cursor/Copilot/viewer) drifts out
  of sync before a push to remote. Use when preparing to push after merging
  changes, or when explicitly asked to bump/release a new version. Do NOT use
  for changes that don't touch remote (local experimentation), and do NOT use
  it to edit `.claude-plugin/marketplace.json` — that file has no version field
  by design.
user-invocable: true
auto-trigger: false
trigger_keywords:
  - bump version
  - version bump
  - release version
  - sync versions
  - bump all six files
  - prepare release
  - before pushing to remote
---

# /version-bump — Six-File Version Synchronizer

## Orientation

**Use when:**
- About to push to remote after merging work (per this repo's CLAUDE.md `## Versioning` section)
- Explicitly asked to bump the version or cut a release

**Do NOT use when:**
- The change doesn't touch remote (no push planned)
- The target is `.claude-plugin/marketplace.json` — it intentionally has no `version` field; adding one breaks marketplace schema validation

**What this skill needs:**
- Either an explicit new version (e.g. `2.9.3`) or a bump type (`patch` | `minor` | `major`). If neither is given, default to `patch`.

## Protocol

### Step 1: READ — Establish current state

Read the `version` field from these exact six files:

1. `understand-anything-plugin/package.json`
2. `understand-anything-plugin/.claude-plugin/plugin.json`
3. `understand-anything-plugin/packages/viewer/package.json`
4. `.claude-plugin/plugin.json`
5. `.cursor-plugin/plugin.json`
6. `.copilot-plugin/plugin.json`

Command: `node -e "console.log(require('./<path>').version)"` for each.

If any two files disagree, **stop** — do not guess which is correct. Report the mismatch (file → version, for all six) and ask the user which version is the source of truth before proceeding.

### Step 2: DETERMINE — Compute the new version

- If the invocation supplies an explicit version string, use it as-is. Validate it is valid semver (`X.Y.Z`, digits only per segment).
- If the invocation supplies `patch`, `minor`, or `major`, compute it from the current (agreed) version using standard semver increment rules (patch: Z+1; minor: Y+1, Z=0; major: X+1, Y=0, Z=0).
- If neither is supplied, default to a patch bump and state that assumption explicitly in the Step 5 report.

### Step 3: WRITE — Update each file with a targeted line replace

Each file's version line has the exact form `  "version": "OLD",` (two-space indent, trailing comma) as its own line. Do **not** use `JSON.parse` + `JSON.stringify` to rewrite the whole file — that reorders/reformats keys and produces spurious diffs. Instead, replace only that line, e.g.:

```bash
sed -i 's/"version": "OLD"/"version": "NEW"/' <path>
```

Apply this to all six files. Verify each file's line count is unchanged after the edit (confirms nothing else in the file moved).

### Step 4: VERIFY — Confirm lockstep and confirm the untouched file

1. Re-read the `version` field from all six files (same command as Step 1). All six must equal NEW.
2. Confirm `.claude-plugin/marketplace.json` still has no top-level `version` key: `node -e "console.log('version' in require('./.claude-plugin/marketplace.json'))"` must print `false`.
3. Run `git diff --stat` restricted to the six files — each must show exactly `1 file changed, 1 insertion(+), 1 deletion(-)`. Any file showing a different line-change count means formatting drifted; fix before proceeding.

### Step 5: REPORT — Summarize the bump

Print a table: file path, old version, new version. State the bump type used (explicit / patch / minor / major) and whether it was inferred or supplied. Do not commit or push — that is a separate, explicit user decision.

## Quality Gates

- [ ] All six files listed in Step 1 show the identical new version string
- [ ] `.claude-plugin/marketplace.json` has no `version` key before or after
- [ ] `git diff --stat` shows exactly one changed line per file (no reformatting)
- [ ] New version string is valid semver
- [ ] If files disagreed before the bump, the user was asked rather than the skill guessing
- [ ] No commit or push performed by this skill

## Exit Protocol

```
VERSION BUMP COMPLETE

Bump type: {explicit <version> | patch | minor | major} ({inferred|supplied})
Old version: {old}
New version: {new}

Files updated:
  understand-anything-plugin/package.json                  {old} -> {new}
  understand-anything-plugin/.claude-plugin/plugin.json     {old} -> {new}
  understand-anything-plugin/packages/viewer/package.json   {old} -> {new}
  .claude-plugin/plugin.json                                {old} -> {new}
  .cursor-plugin/plugin.json                                {old} -> {new}
  .copilot-plugin/plugin.json                                {old} -> {new}

marketplace.json: unchanged (no version field, as expected)
git diff --stat: {N} files changed, {N} insertions(+), {N} deletions(-)

Not committed. Review the diff and commit/push when ready.
```
