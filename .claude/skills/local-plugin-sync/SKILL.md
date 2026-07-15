---
name: local-plugin-sync
description: >-
  Rebuilds and copies local understand-anything-plugin changes into Claude
  Code's plugin cache so a fresh session picks them up, per this repo's
  CLAUDE.md "Testing Local Plugin Changes" section. Use when you've edited
  packages/core or the skill source and need to test the change through an
  actual installed plugin session. Do NOT use for changes to the dashboard
  dev server (use `pnpm dev:dashboard` instead) or when no plugin is
  installed in this Claude Code instance yet (install from the marketplace
  first).
user-invocable: true
auto-trigger: false
trigger_keywords:
  - sync local plugin
  - test local plugin changes
  - copy plugin into cache
  - resync plugin cache
  - plugin cache out of date
  - refresh installed plugin
---

# /local-plugin-sync — Local Plugin Cache Sync

## Orientation

**Use when:**
- You changed `understand-anything-plugin/packages/core` or the skill source and want to test it through a real installed session, not just unit tests

**Do NOT use when:**
- No plugin is installed in this Claude Code instance yet (`~/.claude/plugins/cache/understand-anything/understand-anything/` doesn't exist — install via `/plugin marketplace add` / `/plugin install` first, this skill only re-syncs an existing install)
- Only the dashboard dev server is being changed (`pnpm dev:dashboard` doesn't need the cache at all)

**What this skill needs:**
- Nothing beyond repo state. Detects everything else itself.

## Protocol

### Step 1: BUILD — Rebuild before copying anything

Run, in order, and require exit code 0 from each:

```bash
pnpm --filter @understand-anything/core build
pnpm --filter @understand-anything/skill build
```

If either fails, **stop**. Do not proceed to Step 2 or 3 — copying now would ship a stale or broken `dist/` into the cache silently. Report the build error and end.

### Step 2: LOCATE — Find the actual installed cache directory

Run:

```bash
ls ~/.claude/plugins/cache/understand-anything/understand-anything/
```

- **If the path doesn't exist, or lists nothing:** the plugin has never been installed from the marketplace in this Claude Code instance. **Stop.** Tell the user this skill only re-syncs an *existing* install — direct them to install first (`/plugin marketplace add` then `/plugin install`, or the steps in `understand-anything-plugin`'s own INSTALL flow), then re-run this skill.
- **If it lists exactly one version directory:** that is `<VERSION>`. Continue.
- **If it lists more than one:** list them to the user and ask which one is the active install before proceeding — do not guess.

Do not compute `<VERSION>` from `understand-anything-plugin/package.json`. The cache directory name reflects whatever version the marketplace last served, which is frequently *behind* the local repo's in-development version — that mismatch is expected and not an error to fix here (a version bump is a separate, deliberate action — see the `version-bump` skill).

### Step 3: SYNC — Copy the real directory tree, never a symlink

**First sync this session, or after adding/removing files (structural change):**

```bash
rm -rf ~/.claude/plugins/cache/understand-anything/understand-anything/<VERSION>
cp -R ./understand-anything-plugin ~/.claude/plugins/cache/understand-anything/understand-anything/<VERSION>
```

Only run `rm -rf` after Step 2 has confirmed `<VERSION>` is a real, existing directory under the cache path — never `rm -rf` a path whose existence wasn't just verified.

**Subsequent re-syncs in the same session (only file contents changed, no new/removed files):**

```bash
pnpm --filter @understand-anything/core build && \
cp -R ./understand-anything-plugin/* ~/.claude/plugins/cache/understand-anything/understand-anything/<VERSION>/
```

Never use `ln -s` or any symlink here — Claude Code's Search/Glob tools cannot follow symlinks, so a symlinked cache silently fails to expose files to those tools even though `cat`/`Read` would still work.

### Step 4: REMIND — Session and verification

Tell the user, verbatim in substance:
1. "Start a fresh Claude Code session — an already-open session cached the old plugin prompts in its context and won't see this change."
2. "Run `/understand --full` in the target project to verify" (or a more targeted command if the user's change only affects one skill, e.g. `/understand-diff` for a diff-analysis change).

## Quality Gates

- [ ] Both build commands ran and exited 0 before any file copy happened
- [ ] Cache target directory existence was verified via `ls` before any `rm -rf`
- [ ] Copy used `cp -R` on a real directory tree, never `ln -s`
- [ ] `<VERSION>` was read from the actual cache listing, never computed from local `package.json`
- [ ] Any version mismatch between local `package.json` and `<VERSION>` was reported, not silently forced to match
- [ ] User was explicitly told to start a fresh session before testing
- [ ] A verification command was suggested

## Exit Protocol

```
LOCAL PLUGIN SYNC COMPLETE

Build: core OK, skill OK
Cache target: ~/.claude/plugins/cache/understand-anything/understand-anything/<VERSION>
Sync mode: {full (rm -rf + cp -R) | incremental (cp -R contents only)}
Local package.json version: {X.Y.Z}  (cache dir: <VERSION> — {match|mismatch, expected})

Next steps:
  1. Start a fresh Claude Code session.
  2. Run /understand --full (or a targeted skill command) in the target project to verify.
```
