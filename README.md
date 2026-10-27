# Zidane — GSD-coupled line

A multi-agent implementation workflow that runs inside a **GSD-initialised**
repository. Nine agents take a phase from analysis to verified, committed code,
with a human gate before anything is written, and close the loop through GSD's
phase directories, state engine and requirement IDs.

This directory is the **GSD-coupled** line. Its twin is `../zidane/`, which runs
the same six stages with no `.planning/`, no state engine and no per-project files.
The two are kept deliberately in step — see *Keeping the lines in step* below.

## The pipeline

| Stage | Agent | Does | Can write |
|---|---|---|---|
| 0.6 | **Iniesta** | Learns the project's real conventions from its code, lint config and docs | `.agents/conventions.md` |
| 1/6 | **Griezmann** | Dissects the phase against the codebase; impact per layer, assumptions | `{padded}-RESEARCH.md`, `{padded}-CONTEXT.md` |
| 2/6 | **Pogba** | One work package per detected layer, with the shared contract written out | `{padded}-{NN}-PLAN.md` |
| 3/6 | **Messi · Cristiano · Neymar** | Persistence · service · UI, by wave, against the plan's contract | source + `{padded}-{NN}-SUMMARY.md` |
| 4/6 | **Benzema** | Read-only quality gate over the diff | `{padded}-REVIEW.md` |
| 5/6 | **Mbappé** | Proves it works end to end | tests + `{padded}-VERIFICATION.md` |
| — | **Modric** (`/zizou`) | Orchestrates, holds the gates, owns every commit and every `gsd_run` write | commits, GSD state |

Everything except `.agents/conventions.md` lands in
`.planning/phases/{padded}-{slug}/` and is frontmatter-validated against GSD's own
schemas. Requirement IDs flow from `phase_req_ids` through each plan to
`requirements.mark-complete`, so the work survives a context reset.

The football names describe a **layer of responsibility, not a technology**. Messi
is "whatever this project uses for persistence" — Liquibase, Alembic, Prisma, raw
SQL, a document store. Nothing in here is bound to a stack.

## Files

| File | What |
|---|---|
|  `zizou-gsd.md` | The orchestrator. Installs as a slash command. |
| `iniesta.md` … `mbappe.md` | The eight agent prompt files. |
| `_gsd-runtime.md` | Shared include: `gsd_run` bootstrap, phase payloads, artifacts, frontmatter, state, commits, worktree safety, layout probe, fix protocol. |
| `_context-discovery.md` | Shared include: how a stack is discovered, the layer vocabulary, CodeGraph, the config schema. |
| `config.example.yaml` | Optional per-project overrides. Inert by default, on purpose. |
| `ARCHITECTURE-COMPARISON.md` | How this line and the standalone line differ, and what a merge would cost. |

## Install

```
zizou-gsd.md  → ~/.claude/commands/zizou-gsd.md
{agent}.md    → {project}/.agents/zidane/{agent}.md
_*.md         → {project}/.agents/zidane/
```

The agents install **inside the project**, not into `~/.claude/agents/`. They are
not registered subagent types: Modric reads `.agents/zidane/{agent}.md` and inlines
it into the prompt it dispatches.

**So tool restrictions in this line are advisory, not enforced.** `benzema.md`
declares `tools: Read, Write, Bash, Grep, Glob, …` and says it is read-only over
the source, but nothing in the harness stops an inlined Benzema from editing the
code it is reviewing — only the prompt does. That is a real weakness of this line,
and the standalone line does not have it. It is the open item in
*Known divergence* below.

 `zizou-gsd.md` installs as a slash command. It was a `modric.md` subagent until
2026-10-22, which was a structural bug rather than a style choice: a subagent can
neither prompt the user nor spawn other agents, and the orchestrator does both at
every gate.

Worktrees go to `~/.claude/zidane/worktrees/`, **outside** the repository, so a
careless `git add` cannot sweep them into a feature branch. Artifacts go to the
GSD phase directory and are committed with the code, separately, and only when
`commit_docs` is true.

## The two ideas that make it work

**Conventions are learned, never assumed.** Iniesta samples the most recently
*added* files of each kind, reads the lint and CI config, and emits rules with a
tier, a confidence computed over the whole repo, and a `file:line` pointer. Rules a
linter already enforces are tagged `enforced-by:` and **Benzema is forbidden from
raising them** — re-reporting what CI catches is the fastest way to make a reviewer
ignored. Where recent and historical conformance diverge, Iniesta records a
migration in progress, so the implementers write the new pattern instead of
faithfully reproducing the one the team is deleting. `.planning/codebase/*` from
`/gsd-map-codebase` is read as a stated-docs source and kept; where the two
disagree, Iniesta wins, because its evidence is closer to the code.

**The contract is written down before the fan-out.** Messi, Cristiano and Neymar
run concurrently and never see each other's code. What makes that safe is that
Pogba pins the shared contract literally inside every plan in the same wave, each
implementer reports what it actually built, and the orchestrator diffs
`## Contract Surface` → `## Surface exposed` → `## Surface consumed` before a single
commit is made. That cross-layer check is the main reason the orchestration earns
its cost over one agent doing everything.

## Keeping the lines in step

`_gsd-runtime.md` and `../zidane/_runtime.md` are **section-for-section identical**:
§3 is artifacts in both, §6 commits, §7 the cwd-drift sentinel, §8 the layout probe,
§9 the fix protocol. Only §1, §2 and §5 differ, because only those touch the state
backend. **When you change a numbered section here, change the same section there.**

The whole divergence between the lines is four slots:

| Slot | standalone | GSD |
|---|---|---|
| Context | `.agents/conventions.md` | same, **plus** `.planning/codebase/*`, `PROJECT.md`, `STATE.md` |
| Artifacts | `~/.claude/zidane/runs/…`, no schema | `.planning/phases/{padded}-{slug}/`, frontmatter-validated |
| State | `RUN.md` ledger | `gsd_run state.* / roadmap.* / requirements.*` |
| Commits | plain git, one per layer | driver from layout; `commit-to-subrepo` for sibling repos |

Everything else — the six stages, the hard pause at [2/6], the coherence check, the
`FIX_REQUEST` protocol and its 2-round cap, the contract table titles, Mbappé's
modes A–D — is the same text in both.

### Known divergence, not yet resolved

The two lines still **dispatch differently**: standalone spawns registered
subagent types (`subagent_type: zidane-messi`, tools enforced by the harness),
while GSD has the orchestrator read `.agents/zidane/messi.md` and inline it (tools
advisory). Registered types are strictly better, but converting this line means
resolving a name collision — both lines want to be called `messi`. That decision is
open.

## Status

**Design-complete, never yet run end to end.** Six blocking defects were found and
fixed by inspection, not by execution; see `ARCHITECTURE-COMPARISON.md` for the
full record.

- **Risk 1 — retired by measurement.** The GSD payload keys were verified against
  `gsd-core/bin/lib/init.cjs` and a live run. `sub_repos`, `branching_strategy`,
  `branch_name` and `parallelization` exist only on `init.execute-phase`; the
  orchestrator was reading them off `init.plan-phase` and getting `undefined`,
  which silently degraded every multi-repo workspace to a single repo.
  `phase_req_ids` is a comma-joined string, not an array. Both are now documented
  in `_gsd-runtime.md` §2.
- **Risk 2 — contained, not retired.** `commit-to-subrepo` is documented for
  sibling directories and has never been observed against git submodules, which
  have their own `.git` and their own index. §6 now routes submodules to plain
  `git -C {submodule}` and reserves `commit-to-subrepo` for the sibling case.
  Retiring this needs one real run.

Treat the first run on a real repo as a test of the workflow, not of your phase —
start on a single-repo GSD project, which exercises everything except the submodule
commit path.
