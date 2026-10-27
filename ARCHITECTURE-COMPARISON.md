# Zidane: two architectures, side by side

As of 2026-10-22 there are two live Zidane lines plus the original. They are not
compatible as-is. This file records how they differ so the experiment can be
resolved later instead of re-derived.

| | **A — `~/.claude/agents/zidane-*` + `commands/zizou.md`** | **B — `~/IdeaProjects/zidane-gsd/`** |
|---|---|---|
| Form | Registered subagent types + a slash command | Markdown prompt files + a slash command |
| Agents present | iniesta, griezmann, pogba, messi, cristiano, neymar, benzema, mbappe (8) | the same 8 — iniesta added 2026-10-22 |
| Orchestrator | `/zizou` (command) | `/zizou-gsd` (command, was `modric.md` subagent until 2026-10-22) |
| Dispatch | `Agent(subagent_type: "zidane-pogba")` | Modric reads `.agents/zidane/pogba.md` and inlines it |
| Tool limits | Declared in frontmatter, enforced by harness | Stated in prose, enforced by obedience |
| Convention source | `.agents/conventions.md`, built by Iniesta | `.agents/conventions.md` (tier 0) + `.planning/codebase/*.md` |
| Artifacts | `~/.claude/zidane/runs/{repo}/{ticket}/` | `.planning/phases/{padded}-{slug}/`, frontmatter-validated |
| State / requirements | none | `gsd_run query state.* / roadmap.* / requirements.*` |
| Commits | Orchestrator, explicit paths, message from CONVENTIONS | Orchestrator, driver from layout (§6) |
| Parallelism | all active layers in one message | waves from the plan frontmatter |
| Solo entry points | `/griezmann`, `/messi`, … (8 commands) | none |
| GSD coupling | zero | total |

> The 2026-10-22 edit to this table corrects an earlier version that described A as
> having four agents including "deschamps". A has eight; deschamps was renamed
> Iniesta. Re-check this table against the filesystem before trusting it.

`~/IdeaProjects/zidane/` **was** the pre-GSD, SOPHI-hardcoded original. As of
2026-10-22 it is the generalised **standalone line** — A's twin, with its own
README. The SOPHI version survives only as the deployed copy at
`sophi-main-module/.agents/zidane/`, which no longer matches either line.

---

## Where A is better

**Tool enforcement is structural, not advisory.** `zidane-griezmann` declares
`tools: Read, Glob, Grep, Bash, mcp__codegraph__codegraph_explore` — it *cannot*
write. B's Griezmann is read-only because the prompt says so. One of those survives
a confused model; the other doesn't.

**Iniesta** (called Deschamps in the first version of this document) **is a
genuinely better convention source than `/gsd-map-codebase`** for review purposes,
because of two ideas B lacked until it was ported across on 2026-10-22:

- *Tier 2 — machine-enforced config.* Lint, formatter, ArchUnit, `tsconfig` strict
  flags, pre-commit hooks. A rule carries `enforced-by: <tool>`, **and Benzema is
  forbidden from raising it.** Re-reporting what CI already catches is noise, and
  B has no mechanism to suppress it.
- *Recency sampling.* `git log --diff-filter=A` to read the most recently *added*
  files rather than a random sample — random teaches you the codebase's history,
  recency teaches you its present. A tier-4 inference contradicting tier 3 is
  recorded as a `CONFLICT` (usually a live migration) rather than silently lost.

**Correct handling of the subagent boundary.** A's Griezmann states plainly that it
cannot prompt the user and that the orchestrator confirms naming. B shipped with
the opposite instruction inherited from the original; it was a real bug, fixed
2026-10-22 — but A got it right by construction.

## Where B is better

**It closes the loop with GSD.** Phase directories, PLAN/SUMMARY frontmatter that
validates against `gsd_run query frontmatter.validate --schema plan`, requirement
IDs flowing from `phase_req_ids` through to `requirements.mark-complete`, progress
and decisions landing in `STATE.md`. A now writes its artifacts too, but to
`~/.claude/zidane/runs/` — they survive a context reset, yet they are tied to no
roadmap and close no requirement.

**Commit discipline.** Agents never commit; Modric commits after coherence checks.
An incoherent tree never reaches history. A has since adopted the same discipline
(`_runtime.md` §6), so this is no longer a differentiator — only the driver differs.

**The contract-literal gate.** When the service and client plans share a wave, both
plans contain the same written-out API contract, and Benzema diffs what was built
against it. This is the highest-value check in a parallel pipeline — the two agents
never saw each other's code — and it is the main reason the orchestration earns its
cost over a single agent.

**It was complete first.** B covered analysis → verification before A did. Both
now carry the same eight agents.

---

## The merge — mostly done, 2026-10-22

The decision taken was **not** to collapse the two into one. They stay as two
complete, synchronised lines: `zidane/` (standalone) and `zidane-gsd/` (GSD). The
cross-pollination that made them equivalent:

| # | Item | Status |
|---|---|---|
| 1 | A's form — registered subagent types, tools enforced in frontmatter | **standalone only.** See the open question below. |
| 2 | Iniesta as the convention layer, with `.agents/conventions.md` outranking `.planning/codebase/CONVENTIONS.md` | done — `_context-discovery.md` §1 in both |
| 3 | B's GSD layer kept wholesale — artifacts, state, requirements, commit routing | unchanged |
| 4 | Benzema taught the `enforced-by:` rule and given a closed block-list | done in both |
| 5 | B's missing agents ported into the standalone line | done — both lines carry all eight |
| 6 | A's stack detection, repo preflight, worktree bootstrap, layout taxonomy | done in both |
| 7 | A's `FIX_REQUEST` protocol with a 2-round cap | done in both, `§9` of each runtime include |
| 8 | A's Mbappé modes A–D and process discipline; B's `HARNESS PROPOSAL` gate | both, in both lines |
| 9 | Contract table titles unified: `## Contract Surface` → `## Surface exposed` → `## Surface consumed` | done in both |

**Artifacts written or returned?** Resolved in favour of *written*: every agent on
both lines writes its artifact and returns only a short summary plus a completion
marker. Writing is what survives a context reset. The lines differ only in the
sink — a GSD phase directory, or `~/.claude/zidane/runs/`.

### Still open

**Dispatch mechanism.** The standalone line spawns registered subagent types
(`subagent_type: zidane-messi`), so the harness enforces each agent's tool list —
Benzema physically cannot write. The GSD line still has its orchestrator read
`.agents/zidane/messi.md` and inline it, where the tool list is advisory prose.
A's way is strictly better. Converting the GSD line means resolving a name
collision: both lines want an agent called `messi`. Either namespace them
(`zidane-gsd-messi`) or accept that one set of registered agents serves both
orchestrators. Not yet decided.

---

## Status of B — updated 2026-10-22

Still **execution-unverified end to end**, but six blocking defects found by
inspection have been fixed, and the headline risk has been measured rather than
guessed.

### Risk 1 — RETIRED, and it was real

Verified against `gsd-core/bin/lib/init.cjs` plus a live run on a real GSD project.
All 14 documented keys exist. But:

- **`init.plan-phase` returns neither `sub_repos`, nor `branching_strategy`, nor
  `branch_name`, nor `parallelization`.** Only `init.execute-phase` does. Modric
  read all of them off `plan-phase` — every one was `undefined`, which silently
  degraded every multi-repo workspace to "single repo".
- **`sub_repos` defaults to `[]`** and is pure configuration. A project with real
  git submodules reports `[]`. "Empty means single-repo" was wrong; empty means
  *unconfigured*. It must be cross-checked against a layout probe.
- **`branching_strategy` defaults to `"none"`**, which makes `branch_name` `null`.
- **`phase_req_ids` is a comma-joined string**, not an array. `requirements.mark-complete
  ${PHASE_REQ_IDS}` mangles it.
- `plan-phase` already returns `padded_phase`; B was recomputing it.
- `.planning/config.json` warns on unknown keys, so Zidane settings stay in
  `.agents/zidane/config.yaml`.

All of this is now in `_gsd-runtime.md` §2, split per workflow.

### Risk 2 — CONTAINED, not retired

`commit-to-subrepo` is still documented for sibling directories, and B still has no
observed run against git submodules. `_gsd-runtime.md` §6 now routes submodules to
plain `git -C {submodule}` instead, and reserves `commit-to-subrepo` for the sibling
case it is documented for. Retiring this properly needs one real run.

### Risk 3 — REDUCED

The worktree setup is no longer the SOPHI original generalised by reasoning. It now
has a repo preflight (not-a-repo / detached HEAD / dirty tree), creates the worktree
**outside** the repo, bootstraps dependencies before the first build, and pre-checks
branch existence.

### Also fixed

- **Modric was declared as a subagent** while needing `AskUserQuestion` and the
  ability to spawn agents — structurally impossible. It is now the `zizou-gsd.md` slash
  command. `modric.md` is deleted.
- **Mbappé had no process protocol.** A foreground dev server would have hung the
  pipeline on the first project with one. It now has modes A–D, background launch,
  readiness polling, unconditional teardown, and a Residue section.
- **No layout taxonomy.** A pnpm/turbo/nx/go.work/Cargo workspace would have been
  branched per package. `_gsd-runtime.md` §8 now distinguishes submodules /
  workspace monorepo / sibling sub-repos / single repo, and §6 derives the commit
  driver from it.
- **The fix loop was unbounded.** `FIX_REQUEST` blocks (§9) are now emitted by
  Benzema and Mbappé and capped at 2 rounds by the orchestrator.
- **Benzema had no noise suppression.** It now carries the closed block-list,
  `enforced-by:` silence, and a "Not Raised" section.

### Next

The remaining gap is a real run. Cheapest order: a single-repo project (exercises
0.1–0.7, the wave dispatch and `gsd_run commit`), then a submodule project
(exercises §6's submodule path and the `sub_repos: []` reconciliation).
