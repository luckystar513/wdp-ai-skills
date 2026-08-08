---
name: wdp-ctx-en
description: Use when the user runs /wdp-ctx-en to save, load, verify, export, or clear the project's context so it survives session reset and can be resumed by any coding agent. English variant of wdp-ctx. Subcommands: sum, init, profile, verify, export, list, clear
disable-model-invocation: true
---

# /wdp-ctx-en — Project context: save anytime · resume anytime · switch anytime

This is the **English variant** of the `/wdp-ctx` skill. It turns the current project's **context** into documents, so any coding agent can reload it with one command after a restart, a context clear/compact, or when switching agents or projects. **Core value = context that is always readable and switchable.**

- **Stable info** (tech stack / architecture / rules / how to run) → the profile (`profile.md`), maintained by the `profile` subcommand;
- **Volatile info** (current focus / done / in progress / next steps / non-obvious learnings) → snapshots, appended by the `sum` subcommand.

**Core principle: stable info goes to the profile, volatile info goes to snapshots.** Snapshots reference the profile via `profile:` in frontmatter instead of duplicating the tech stack.

## Usage

```
/wdp-ctx-en <subcommand> [arguments]
```

| Subcommand | What it does | Arguments |
|------------|--------------|-----------|
| `sum` | Save the current work state as a snapshot (incremental diff vs last; `--deep` = deep analysis) | `<project path>` `--deep` |
| `init` | Load profile + latest snapshot and resume last session | `<project path>` |
| `profile` | Create/update the project's stable profile | `<project path>` |
| `verify` | Check snapshot drift vs current code (read-only) | `<project path>` |
| `export` | Merge profile + snapshot into `AGENTS.md` for agents that don't know this skill | `<project path>` |
| `list` | List all profiles and snapshots (newest first) | — |
| `clear` | Remove snapshots (profile kept by default; `--keep N` keeps newest N) | `<project path>` `--keep N` `--profile` |

**Run with no subcommand** → defaults to `list` and shows this usage.

## Storage model

```
<storage root>/<project slug>/
├── profile.md              # stable layer (maintained by the profile subcommand)
├── <YYYY-MM-DD_HHMMSS>.md  # snapshot layer (appended by the sum subcommand)
└── latest.md               # pointer to the latest snapshot (read by init/verify)
```

## Storage root (shared with wdp-ctx)

`/wdp-ctx-en` and `/wdp-ctx` **share the same storage**, so switching between the two languages never loses context.

Resolution order — use the first one that exists; prefer project-level:

1. `<project path>/.claude/skills/wdp-ctx/summaries/`
2. `<project path>/.claude/skills/wdp-ctx-en/summaries/`
3. `~/.claude/skills/wdp-ctx/summaries/`
4. `~/.claude/skills/wdp-ctx-en/summaries/`

Create the directory if none exists. Per-project subdirectories are keyed by project slug; filenames carry timestamps, so multiple projects never mix.

---

## sum — save a work snapshot

1. **Resolve the project**: no argument = current directory (`pwd`); `sum <path>` = given path.
2. **Read the previous snapshot** (incremental base): `latest.md` first, else the newest by timestamp.
3. **Gather info** (read-only exploration; don't modify project files):
   - git state (if in a repo): `git status --short`, `git log --oneline -10`, `git diff --stat`.
   - **Incremental diff (core)**: check every "in progress / next / current focus" item from the previous snapshot and mark ✅ done / 🔄 still in progress / ⛔ dropped / 🔀 changed direction; for in-progress items write concrete progress backed by git diff; **if nothing changed since last time, mark "(no change since last)" — never fabricate progress**.
   - This session's TaskList, and this session's non-obvious learnings (pitfalls hit, counterintuitive findings).
4. **Write the snapshot document** (template below); frontmatter `prev` / `created` / `profile` are required.
5. **Update `latest.md`** to the newest snapshot filename.
6. **Report**: path saved + one-line summary; if the profile is missing, suggest `profile` first.

**`--deep` mode** (explicit opt-in, more tokens): split into 2-5 subsystems → fan out read-only sub-agents to analyze each (current state, drift vs snapshot, non-obvious issues) → optionally one cross-cutting agent (TODO/FIXME, git log) → synthesize a more complete snapshot, `tags: [deep]`. Confirm the user is willing to wait before dispatching.

### Snapshot template

```markdown
---
type: snapshot
project: <project slug>
created: <YYYY-MM-DDTHH:MM:SS>
author: <agent name>
commit: <git hash or ~>
prev: <previous snapshot filename or ~>
profile: <profile.md or ~>
tags: []
---

# Work snapshot · <project name>

## Current focus
<one line: why this work is happening now>

## Done since last
- ✅ <point>

## In progress
- 🔄 <point, incl. blockers/risks> (new progress: …; or mark "no change since last")

## Changed / dropped
- ⛔ <dropped item> → reason

## Next steps (3-5)
- <in order of value>

## Non-obvious learnings this session
- <pitfalls, counterintuitive findings, overturned assumptions>

## Decisions & why
- <What + Why, to avoid repeating mistakes>
```

---

## init — load context

1. **Resolve the project**: no argument = `pwd`; else the given path.
2. **Read the profile**: read `<slug>/profile.md`; if missing, report "no profile yet — suggest `/wdp-ctx-en profile`", then continue to the snapshot; if present, read it fully and restate: one-line positioning, tech stack, architecture, **must-not-break rules**, how to run, known gotchas.
3. **Read the latest snapshot**: the one `latest.md` points to, else the newest by timestamp (exclude profile.md/latest.md); read fully and restate: current focus, done, in progress, next steps, learnings, decisions. If the target project directory is empty, fall back to another project's latest snapshot and **say so explicitly**.
4. **Done**: state path + time; suggest 2-3 entry points from "in progress / next steps" and ask which to continue.

---

## profile — maintain the stable profile

1. **Resolve the project**: no argument = `pwd`; else the given path.
2. **Gather stable info** (read-only):
   - Tech stack: manifest files (package.json / requirements.txt / pyproject.toml / go.mod / Cargo.toml) + config (vite/tsconfig/.env.example/docker-compose).
   - Architecture: directory structure, module boundaries, entry points, data flow.
   - Domain: README, project purpose, naming.
   - How to run: package.json scripts, Makefile, build/test/run/deploy in README.
   - Rules/invariants: CLAUDE.md / AGENTS.md / CONTRIBUTING.md / .editorconfig / code comments → extract **mandatory, must-not-break** rules (their own section, strictly separate from temporary notes).
   - Decision records, known gotchas, external dependencies (only where credentials live, never the secrets).
3. **Create or update**: create from the template on first use; if it exists, compare section by section and update only the changed ones. Update frontmatter `updated`.
4. **Report**: path, which sections were added/updated, **and explicitly list the "must-not-break rules" for the user to confirm**.

### Profile template

```markdown
---
type: profile
project: <project slug>
updated: <YYYY-MM-DDTHH:MM:SS>
---

# Project profile · <project name>

## One-line positioning
## Tech stack
## Architecture overview
## Business domain
## Invariants (must not break)
- <mandatory, prohibited to violate>
## How to run
- build/test/run/deploy
## External dependencies
- <services/APIs/where credentials live>
## Decisions & why
## Known gotchas
- <gotcha + how to avoid>
```

---

## verify — check snapshot drift (read-only)

1. Resolve the project; read the latest snapshot (if none, report "no snapshot yet — run sum first" and stop), parse frontmatter (created/commit).
2. **Collect current state**: git (status/branch/log/diff, skip if not a repo); check each key path mentioned in the snapshot/profile for existence; check "in progress / next" items against the code.
3. **Judge item by item**:

   | Label | Meaning |
   |-------|---------|
   | ✅ still valid | matches current state, nothing to do |
   | 🔄 progress not recorded | git has new changes the snapshot doesn't reflect → suggest sum |
   | ⚠️ stale | mentioned files/modules/next steps no longer exist or are done → needs sum to revise |
   | ❌ conflict | snapshot contradicts the code → refresh immediately, don't resume from the stale snapshot |

   - Flag it if the commit differs from current HEAD; mark "profile refresh needed" if the profile layer visibly changed.
4. **Report**: snapshot time vs now, commit comparison, drift table, **staleness verdict** (fresh / slightly stale / badly stale), and the suggested action (sum or profile).

---

## export — export AGENTS.md

Merge the profile + latest snapshot into `AGENTS.md` in the project root, so **agents that don't know this skill** can read it at startup. If either layer is missing, use what exists and mark "this section not maintained yet".

**Merge policy (never lose user content)**:
- Target doesn't exist → create it, fully wdp-generated;
- Exists and contains `<!-- wdp-export-begin -->` / `<!-- wdp-export-end -->` → replace only between the markers;
- Exists without markers → append a marker-wrapped maintenance section at the end, leave existing content untouched.

**Template** (compact — AGENTS.md should stay short):

```markdown
<!-- wdp-export-begin -->
# Project Context (auto-generated by wdp /wdp-ctx export)

## What / Quick summary
## Tech stack
## Architecture (top-level)
## Invariants (must not break)
## How to run
## Current work state (as of <snapshot time>)
- current focus / in progress / next (top 3) / recently done
## Known gotchas / Decisions
<!-- wdp-export-end -->
```

Never write secrets (only where credentials live), never write full snapshot history. Report: path, created/replaced/appended, which sections are missing.

---

## list — list all documents

1. Enumerate `<storage root>/**/*.md`, excluding latest.md (the pointer file). If empty, report "no documents yet" and suggest `sum` / `profile`.
2. Show in two parts:
   - **A. Profiles**: project / updated time / path
   - **B. Snapshots (newest first)**: created time / project / summary (title) / path
   - Sort snapshots by filename timestamp descending (lexicographic order = chronological); use frontmatter `created` and the title as the summary column; end with counts: N snapshots, M profiles, across K projects.
3. Next steps: want details → `init <project path>`; update the profile → `profile <project path>`.

---

## clear — remove documents

**Irreversible destructive operation — confirm twice before deleting.**

**Default behavior**: snapshots (`<timestamp>.md`) are the default targets; profile.md is **kept by default**; latest.md is handled after snapshots are deleted; `--keep N` keeps the newest N per project.

1. **Clarify scope**: all snapshots (default) / all + profile (explicit) / one project (`clear <path>`) / profile only (`clear --profile`) / keep newest N (`clear --keep N`, can combine with a project path).
2. **Compute the delete list**: group by project, sort by filename timestamp **newest first**, first N = "keep", rest = "delete"; show both the delete list and the keep list.
3. **Confirm (required)**: "Delete these N documents (keep M)? This cannot be undone." If refused, report "cancelled, nothing deleted".
4. **Delete**: only files in the list; if latest.md dangles afterward, point it at the newest kept snapshot; if nothing is kept, delete latest.md. Report deleted N, kept M.

---

## Automatic reminders (turn wdp from a manual tool into automatic memory)

Anywhere in this session, proactively remind the user to save/load context when:

- Session context is about to be **compacted / cleared** (/clear, context too long) → suggest `/wdp-ctx-en sum`
- a **milestone / significant change** was completed → suggest `/wdp-ctx-en sum`
- about to **switch agents / end session / hand off** → suggest `/wdp-ctx-en sum`
- after resuming a session (init ran), keep following the above

One line is enough, e.g.: "X is done — run `/wdp-ctx-en sum` to save a snapshot; next time `/wdp-ctx-en init` resumes."

## Common mistakes

| Mistake | Fix |
|---------|-----|
| Putting the tech stack / business into a snapshot | Stable info goes to the profile; snapshots only record state changes |
| Overwriting the previous snapshot | Timestamped filenames are appended, never overwritten |
| Ignoring git diff / the previous snapshot | The incremental diff is what makes snapshots valuable |
| Frontmatter missing `prev`/`created` | Without them incremental resume is impossible |
| init reads the snapshot but not the profile | Read both — rules/tech stack live in the profile |
| Sorting profile/latest.md as snapshots | Show them separately; exclude latest.md |
| clear deletes without confirming | Destructive ops require listing first, then a second confirmation |
| clear deletes the profile by mistake | Profile is kept by default unless explicitly requested |
| clear `--keep N` the wrong direction | Keep newest N, delete the rest |
| verify modifies files | verify is read-only |
| export overwrites an existing AGENTS.md | With markers, replace only the maintenance section; without, append |
| Dispatching sub-agents in normal mode | `--deep` is explicit opt-in |
| Recording secrets, not just locations | No subcommand may write secret contents |
