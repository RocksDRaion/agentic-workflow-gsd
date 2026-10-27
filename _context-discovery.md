---
name: _context-discovery
---
# Project Context Discovery — Shared Include

Zidane agents carry **no hardcoded tech stack**. Every stack fact — languages,
frameworks, directory layout, architecture rules, test commands — is discovered at
runtime from the project itself. This is what makes the pipeline portable.

Read this before acting. Never assume a stack because the role name suggests one.

---

## 1. Discovery order

Read in this order. Later sources refine earlier ones; they never contradict them
silently — if two sources disagree, surface the conflict instead of picking one.

| # | Source | Gives you |
|---|--------|-----------|
| 1 | `.agents/zidane/config.yaml` | Hand-written overrides. **Always wins** (see §3). |
| 2 | `.agents/conventions.md` | **The project's learned rules** — tiered, evidenced, confidence-scored. Built by Iniesta. |
| 3 | `./CLAUDE.md` (and `.claude/CLAUDE.md`) | **Hard constraints.** Treat as non-negotiable. |
| 4 | `.planning/codebase/STACK.md` | Languages, frameworks, versions, package managers |
| 5 | `.planning/codebase/ARCHITECTURE.md` | Layering rules, boundaries, dependency direction |
| 6 | `.planning/codebase/CONVENTIONS.md` | Naming, file placement, idioms |
| 7 | `.planning/codebase/STRUCTURE.md` | Directory map |
| 8 | `.planning/codebase/TESTING.md` | Test frameworks, commands, coverage expectations |
| 9 | `.planning/codebase/INTEGRATIONS.md` | External services |
| 10 | `.planning/codebase/CONCERNS.md` | Known debt and traps |
| 11 | `.planning/PROJECT.md` | Product intent, constraints, key decisions |
| 12 | `.planning/STATE.md` | Current position, accumulated decisions, blockers |
| 13 | `{phase_dir}/{padded}-CONTEXT.md` | Decisions locked for *this* phase |
| 14 | `.claude/skills/` or `.agents/skills/` | Project skills — read `SKILL.md`, load `rules/*.md` on demand |

> **#2 outranks #6 on purpose.** `.agents/conventions.md` and
> `.planning/codebase/CONVENTIONS.md` answer the same question with different
> evidence. Iniesta's carries a tier, a conformance ratio computed over the whole
> repo, an `enforced-by:` marker where a linter already catches the rule, and a
> `file:line` pointer for every claim. `/gsd-map-codebase` output carries none of
> those. The two complement each other; where they disagree, Iniesta's is closer
> to the code.

Load only what your role needs:

| Your role | Load at minimum |
|-----------|-----------------|
| Scout (Iniesta) | everything — producing #2 is its job |
| Analyst (Griezmann) | 1, 2, 3, 4, 5, 7, 11, 12 |
| Planner (Pogba) | 1, 2, 3, 4, 5, 6, 8, 11, 13 |
| Data layer (Messi) | 1, 2, 3, 4, 5, 6, 8 |
| Service layer (Cristiano) | 1, 2, 3, 4, 5, 6, 8 |
| UI layer (Neymar) | 1, 2, 3, 4, 6, 7, 8 |
| Reviewer (Benzema) | 1, 2, 3, 5, 6, 10 |
| Functional tests (Mbappé) | 1, 2, 7, 8, 9 |

### If `.agents/conventions.md` does not exist

The project has not been scouted. The orchestrator spawns Iniesta in Phase 0.6, so
this should not happen mid-run; if it does, say so and continue on `.planning/codebase/*`.

### If `.planning/codebase/` does not exist

The project has not been mapped. Say so and recommend:

```
/gsd-map-codebase
```

Then fall back to direct inspection for this run only: read the manifest files you
actually find (`package.json`, `pom.xml`, `build.gradle.kts`, `pyproject.toml`,
`Cargo.toml`, `go.mod`, `*.csproj`, `Gemfile`, `composer.json`) plus the top two
levels of the source tree. Record what you inferred in your output so the next agent
does not re-derive it.

---

## 2. Layer roles are generic

The football names describe a **layer of responsibility**, not a technology.

| Agent | Layer | Typical artifacts (whatever the stack calls them) |
|-------|-------|---------------------------------------------------|
| Messi | Persistence / data | schema migrations, repositories, DAOs, ORM models, seed data, data-layer integration tests |
| Cristiano | Service / API | domain logic, use cases, HTTP or RPC handlers, serializers, unit tests |
| Neymar | Presentation / client | components, views, pages, client-side state, data-fetching, styling, component tests |
| Mbappé | End-to-end | black-box functional tests against the running system |
| Benzema | Cross-cutting review | architecture conformance, contract coherence, security, tests |

A phase may not need every layer. If `.planning/codebase/STACK.md` shows no client
application, Neymar is simply not dispatched — do not invent frontend work.

In a single-repo backend service, Messi and Cristiano both operate inside that one
repo; the split is by concern, not by directory.

---

## 3. Optional overrides — `.agents/zidane/config.yaml`

Projects may pin anything discovery would otherwise guess. Every key is optional,
and the file itself is optional. Schema:

```yaml
layers:                      # maps the 3 implementation agents onto this project
  data:                      # → Messi
    repo: <sub-repo name>    # must match a GSD `sub_repos` entry; OMIT for single-repo
    paths: [<glob>, …]       # where this layer's code lives
  service:                   # → Cristiano
    repo: <sub-repo name>
    paths: [<glob>, …]
  client:                    # → Neymar
    repo: <sub-repo name>
    paths: [<glob>, …]
  # Omit a layer the project does not have — its agent is then never dispatched.

commands:
  data:    { test: <cmd> }
  service: { build: <cmd>, test: <cmd> }
  client:  { build: <cmd>, lint: <cmd>, test: <cmd> }
  e2e:     { test: <cmd> }

conventions:
  migration_author: <string>

preview:                     # omit the whole block to skip the preview step
  command:  <cmd>
  url:      <url>
  teardown: <cmd>
```

Worked examples for several stacks live in `config.example.yaml`.

Resolution rule: **config.yaml > `.planning/codebase/*` > direct inspection.**
If a command is not configured and not discoverable, **escalate to the orchestrator
rather than guess** — you cannot prompt the user yourself, so return with the
question instead of inventing an answer. A wrong
build command burns a whole cycle.

---

## 4. CodeGraph

When a `.codegraph/` directory exists at the repo root, prefer it over Glob/Grep/Read
for understanding code. It is stack-agnostic.

- `codegraph explore "<symbols or question>"` — verbatim source, call paths, blast radius
- `codegraph impact <symbol>` — what breaks if this changes
- `codegraph callers <symbol>` / `codegraph callees <symbol>` — precise dependency tracing
- `codegraph affected <changed files…>` — tests and modules touched by a change

The `mcp__codegraph__codegraph_explore` MCP tool is equivalent and preferred when available.

If there is no `.codegraph/` directory, skip CodeGraph entirely — indexing is the
user's decision, never yours.

---

## 5. Library documentation

For version-sensitive framework APIs, do not rely on training knowledge:

1. Context7 MCP when available — `mcp__context7__resolve-library-id`, then
   `mcp__context7__get-library-docs`.
2. CLI fallback — `command -v ctx7 && ctx7 library <name> "<query>"`.
3. `WebSearch` / `WebFetch` against official docs as a last resort.

---

## 6. Reporting unknowns

Discovery that comes up empty is a finding, not a blocker to paper over. Any agent
that could not determine something it needed must say so explicitly in its artifact,
under an `## Unknowns` heading, with the impact of guessing wrong.

**No agent except the orchestrator can talk to the user.** You are a subagent: you
run once and return. When you need a human decision, return it as a question for
the orchestrator to put to the user — do not wait, and do not substitute your own
answer for theirs. Prefer recording an assumption over blocking, and reserve
blocking for information you genuinely cannot derive from the repository.

---

## 7. The `rtk` history trap

Some shells here run `git` behind the `rtk` token-reducing hook, which silently
caps `git log` at 50 entries. A repo with 380 commits then looks like a two-week-old
project, and any recency-based sampling is wrong.

```bash
# if these disagree, your history is truncated
git log --format=%H | wc -l
git rev-list --count HEAD
```

When they disagree, use `rtk proxy git log …` for all history sampling and say so
in your output so the next agent does not fall into the same trap.

**Submodules are separate repositories.** Run every sampling command with
`git -C {submodule path}`; at the superproject you will see no file history at all.
