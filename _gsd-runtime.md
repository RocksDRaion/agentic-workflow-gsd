---
name: _gsd-runtime
---
# GSD Runtime — Shared Include

Every Zidane agent reads this file before doing anything else. It defines how to
talk to GSD's state engine and where artifacts live. Nothing here is project-specific.

---

## 1. Bootstrap `gsd_run`

Run this once, in your first Bash call. It resolves `gsd-tools.cjs` across every
supported runtime layout and defines the `gsd_run` shell function.

```bash
_GSD_SHIM_NAME="gsd-tools.cjs"; _GSD_RUNTIME_ROOT="${RUNTIME_DIR:-$(git rev-parse --show-toplevel 2>/dev/null || pwd)}"; GSD_TOOLS="${_GSD_RUNTIME_ROOT}/gsd-core/bin/${_GSD_SHIM_NAME}"; if [ -f "$GSD_TOOLS" ]; then gsd_run() { node "$GSD_TOOLS" "$@"; }; elif [ -f "${_GSD_RUNTIME_ROOT}/.claude/gsd-core/bin/${_GSD_SHIM_NAME}" ]; then GSD_TOOLS="${_GSD_RUNTIME_ROOT}/.claude/gsd-core/bin/${_GSD_SHIM_NAME}"; gsd_run() { node "$GSD_TOOLS" "$@"; }; elif command -v gsd-tools >/dev/null 2>&1; then GSD_TOOLS="$(command -v gsd-tools)"; gsd_run() { "$GSD_TOOLS" "$@"; }; elif [ -f "$HOME/.claude/gsd-core/bin/${_GSD_SHIM_NAME}" ]; then GSD_TOOLS="$HOME/.claude/gsd-core/bin/${_GSD_SHIM_NAME}"; gsd_run() { node "$GSD_TOOLS" "$@"; }; else echo "ERROR: gsd-tools.cjs not found. Run: npx -y @opengsd/gsd-core@latest --claude --local" >&2; exit 1; fi
```

> If this fails, GSD is not installed. Report it and stop — do **not** fall back to
> ad-hoc file writing.

---

## 2. Load phase context

Two workflows, and **they do not return the same keys.** This caught us once; the
table below is verified against `gsd-core/bin/lib/init.cjs` and a real run, not
inferred from reading the GSD agents.

```bash
# Planner-side context — Pogba, and Griezmann
INIT=$(gsd_run query init.plan-phase "${PHASE}")
if [[ "$INIT" == @file:* ]]; then INIT=$(cat "${INIT#@file:}"); fi

# Execution-side context — implementers, Benzema, Mbappé, AND the orchestrator
EXEC=$(gsd_run query init.execute-phase "${PHASE}")
if [[ "$EXEC" == @file:* ]]; then EXEC=$(cat "${EXEC#@file:}"); fi
```

| Key | `plan-phase` | `execute-phase` | Type / note |
|-----|:---:|:---:|---|
| `phase_dir` | ✅ | ✅ | string or `null` when the phase dir does not exist yet |
| `phase_number`, `phase_name`, `phase_slug` | ✅ | ✅ | string |
| `padded_phase` | ✅ | ❌ | **already zero-padded — use it, do not recompute** |
| `phase_found` | ✅ | ✅ | boolean — check it before trusting `phase_dir` |
| `phase_req_ids` | ✅ | ✅ | **comma-joined string** (`"R1, R2"`), *not* an array, `null` when TBD |
| `commit_docs` | ✅ | ✅ | boolean |
| `plan_count`, `has_plans` | ✅ | ✅ | |
| `plans`, `incomplete_plans`, `summaries` | ❌ | ✅ | arrays |
| **`sub_repos`** | ❌ | ✅ | **array — `[]` by default** |
| **`branching_strategy`** | ❌ | ✅ | `"none"` \| `"phase"` \| `"milestone"` |
| **`branch_name`** | ❌ | ✅ | **`null` whenever `branching_strategy` is `"none"`** |
| `parallelization` | ❌ | ✅ | boolean |
| `response_language` | only when set | only when set | key is **absent**, not null, when unconfigured |

> **The orchestrator must call `init.execute-phase`, not just `init.plan-phase`.**
> Everything that drives worktree branching and commit routing — `sub_repos`,
> `branching_strategy`, `branch_name` — exists only on the execute-phase payload.
> Reading them off `init.plan-phase` yields `undefined` and the orchestrator will
> silently treat every multi-repo workspace as a single repo.

### `sub_repos` is a configuration, not a detection

`sub_repos` comes straight from `.planning/config.json` and defaults to `[]`. A
project with real git submodules reports `[]` until someone configures it —
verified on a submodule project that returns `sub_repos: []`.

So: **`sub_repos` empty does not mean single-repo.** It means *unconfigured*.
Cross-check it against the layout probe (§8). When the probe finds submodules or a
workspace manifest and `sub_repos` is empty, say so and ask — do not assume either way.

### `phase_req_ids` is a string

```bash
# WRONG — passes "R1," and "R2" as separate mangled arguments
gsd_run query requirements.mark-complete ${PHASE_REQ_IDS}

# RIGHT — split on the ", " separator first
IFS=',' read -ra REQ_ARR <<< "${PHASE_REQ_IDS}"
for r in "${REQ_ARR[@]}"; do gsd_run query requirements.mark-complete "$(echo "$r" | xargs)"; done
```

Current position and accumulated context:

```bash
gsd_run query state.load
```

> `.planning/config.json` **rejects unknown keys** with a warning. Zidane's own
> settings never go there — they live in `.agents/zidane/config.yaml`.

---

## 3. Artifact paths

All per-phase artifacts live in `{phase_dir}` = `.planning/phases/{padded_phase}-{slug}/`,
where `padded_phase` is zero-padded (`01`, `07`, `02.1`).

| Artifact | Filename | Written by |
|----------|----------|------------|
| Analysis / research | `{padded}-RESEARCH.md` | Griezmann |
| Locked context | `{padded}-CONTEXT.md` | Griezmann (decisions confirmed by the user) |
| Plan (one per layer) | `{padded}-{NN}-PLAN.md` | Pogba |
| Execution summary | `{padded}-{NN}-SUMMARY.md` | Messi / Cristiano / Neymar |
| Code review | `{padded}-REVIEW.md` | Benzema |
| Functional verification | `{padded}-VERIFICATION.md` | Mbappé |

Naming is strict. The plan suffix is always `-PLAN.md` — never `PLAN-NN.md` or
`NN-PLAN.md`. Summaries mirror their plan's `{NN}`.

Workspace-level files, never phase-scoped:
`.planning/PROJECT.md`, `.planning/ROADMAP.md`, `.planning/STATE.md`,
`.planning/REQUIREMENTS.md`, `.planning/config.json`, `.planning/codebase/`.

---

## 4. Required frontmatter

### `{padded}-{NN}-PLAN.md`

```yaml
---
phase: XX-name
plan: NN
type: execute          # or: tdd
wave: N                # execution wave
depends_on: []         # e.g. ["01-01"]
files_modified: []
autonomous: true       # false if the plan contains checkpoints
requirements: []       # MUST NOT be empty — IDs from ROADMAP/REQUIREMENTS
user_setup: []         # omit when empty
must_haves:
  truths: []           # observable behaviours
  artifacts: []        # files that must exist
  key_links: []        # critical connections
---
```

Validate before handing off:

```bash
gsd_run query frontmatter.validate "$PLAN_PATH" --schema plan
gsd_run query verify.plan-structure "$PLAN_PATH"
```

### `{padded}-{NN}-SUMMARY.md`

```yaml
---
phase: XX-name
plan: NN
subsystem: [data | api | ui | infra | testing | ...]
tags: []
provides:
  - [what this plan delivered]
affects: []
tech-stack:
  added: []
  patterns: []
key-files:
  created: []
  modified: []
key-decisions:
  - "..."
requirements-completed: []   # copy ALL IDs from the plan's `requirements`
duration: Xmin
completed: YYYY-MM-DD
status: complete
---
```

`status: complete` is required — GSD's open-work scanner keys off it.

---

## 5. State updates after finishing work

Implementation agents run these after their SUMMARY.md is written:

```bash
gsd_run query state.advance-plan
gsd_run query state.update-progress
gsd_run query state.record-metric "${PHASE}" "${PLAN}" "${DURATION}" "${TASK_COUNT}" "${FILE_COUNT}"
gsd_run query state.add-decision "${decision}"        # once per key decision
gsd_run query state.record-session "" "Completed ${PHASE}-${PLAN}-PLAN.md" "None"
gsd_run query roadmap.update-plan-progress "${PHASE_NUMBER}"

# REQ_IDS is a comma-joined string — split it, never splat it (see §2)
IFS=',' read -ra REQ_ARR <<< "${REQ_IDS}"
for r in "${REQ_ARR[@]}"; do gsd_run query requirements.mark-complete "$(echo "$r" | xargs)"; done
```

Blockers go to `gsd_run query state.add-blocker "..."` — never silently to prose.

---

## 6. Commits

**Agents never commit.** Modric commits after coherence checks pass.

The driver is chosen from the **layout** (§8), cross-checked against `sub_repos`
from `init.execute-phase` — never from `sub_repos` alone, which is `[]` by default
even on projects that do have sub-repos.

| Layout (§8) | `sub_repos` | Driver |
|---|---|---|
| single repo | `[]` | `gsd_run query commit` — one commit per layer |
| workspace monorepo | `[]` | `gsd_run query commit` — one commit per layer, **never per package** |
| sibling sub-repos | non-empty | `gsd_run query commit-to-subrepo` — one commit per repo |
| submodules | non-empty | raw git per submodule — **see the caveat below** |
| submodules | `[]` | **stop and ask.** The layout and the config disagree. |

```bash
# Sibling sub-repos — commit-to-subrepo groups files by repo prefix
gsd_run query commit-to-subrepo "{message}" --files <paths…>

# Single repo or workspace monorepo
gsd_run query commit "{message}" --files <paths…>
```

`commit-to-subrepo` groups files by their sub-repo prefix, rewrites paths relative
to each repo root, and commits each independently. Files matching no sub-repo are
reported as unmatched — treat that as an error, not a warning.

> **`commit-to-subrepo` is documented for sibling directories, not git submodules.**
> A submodule has its own `.git` and its own index; path-prefix grouping is not the
> same operation. Until this is proven on a real submodule project, commit
> submodules with plain git, one per touched submodule, and **never on the
> superproject**:
>
> ```bash
> git -C {WORKTREE_PATH}/{submodule} add {explicit paths from the agent's report}
> git -C {WORKTREE_PATH}/{submodule} commit -m "{message}"
> ```

**Stage explicit paths only — never `git add -A`.** Use each agent's *Files
Created / Modified* list verbatim. A blanket add sweeps `.planning/` artifacts,
local scratch and `.agents/` caches into the feature branch.

The commit **message format** — type prefix, scope, trailers — is derived from the
project's own history and tooling (`git log -n 50 --format='%s%n%b'`, any commitlint
config, `git config user.name`), not from a template hardcoded here. Planning
artifacts are committed separately and only when `commit_docs` is true.

---

## 7. Worktree safety

When running inside a worktree, guard against cwd drift before any staging:

```bash
if [ -f .git ]; then
  WT_GIT_DIR=$(git rev-parse --git-dir 2>/dev/null)
  case "$WT_GIT_DIR" in
    *.git/worktrees/*)
      SENTINEL="$WT_GIT_DIR/gsd-spawn-toplevel"
      [ ! -f "$SENTINEL" ] && git rev-parse --show-toplevel > "$SENTINEL" 2>/dev/null
      EXPECTED_TL=$(cat "$SENTINEL" 2>/dev/null)
      ACTUAL_TL=$(git rev-parse --show-toplevel 2>/dev/null)
      if [ -n "$EXPECTED_TL" ] && [ "$ACTUAL_TL" != "$EXPECTED_TL" ]; then
        echo "FATAL: cwd drifted from spawn-time worktree root" >&2
        echo "RECOVERY: cd \"$EXPECTED_TL\" before staging." >&2
        exit 1
      fi
      ;;
  esac
fi
```

Prefer relative paths for Edit/Write. When an absolute path is unavoidable, derive
it from `git rev-parse --show-toplevel` **run inside the worktree** — never from a
`pwd` captured in the orchestrator's context.

---

## 8. Layout probe

GSD's `sub_repos` tells you what was *configured*. This tells you what is actually
*there*. Run it in the orchestrator, before the worktree, and reconcile the two (§6).

| Probe | Layout | Commit unit |
|---|---|---|
| `.gitmodules` exists | **submodules** | each touched submodule; **never the superproject** |
| `pnpm-workspace.yaml`, `turbo.json`, `nx.json`, `go.work`, `[workspace]` in `Cargo.toml`, `<modules>` in `pom.xml` | **workspace monorepo** | ONE repo, ONE branch — **never branch or commit per package** |
| `sub_repos` non-empty and no `.gitmodules` | **sibling sub-repos** | each repo |
| otherwise | **single repo** | the repo |

Missing this distinction is how a workspace monorepo gets branched per package.
A monorepo is one git repository that happens to contain many manifests; it is not
a multi-repo workspace, and nothing about it is per-package at the git level.

---

## 9. Fix hand-back protocol

Benzema and Mbappé are subagents: they cannot call other agents and cannot prompt
the user. They hand findings back as machine-readable blocks the orchestrator
parses and re-dispatches.

```
FIX_REQUEST
  id: B1
  target: messi | cristiano | neymar | pogba
  files: path/one, path/two
  problem: {one line — the root cause, not the symptom}
  evidence: {the failing assertion, the two sides that disagree, or the log excerpt}
  acceptance: {the observable condition that proves it is fixed}
END
```

One block per blocking finding. None for advisory or informational findings.
`target: pogba` means the **plan** was wrong — the orchestrator returns to planning
with the user's agreement rather than patching around it.

**The orchestrator caps every fix loop at 2 rounds.** A third round means the plan
or the analysis is wrong, not the implementation; stop and surface it to the user
instead of looping. Findings the user then chooses to accept are recorded, never
dropped silently:

```bash
gsd_run query state.add-blocker "{accepted finding}"
```
