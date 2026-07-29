# Knowledge Base Conventions

Canonical rules for this repo. Every department follows these.

## What changed from upstream ICM

This repo started as a clone of the Interpretable Context Methodology (ICM) — a demo of
linear content-production pipelines (numbered "stages" writing to `output/` folders inside
per-project "workspaces"). We're not running pipelines: this is a standing, repeatedly
refilled inbox sorted into permanent department folders. "Workspaces" became "departments,"
stages/questionnaires/onboarding are gone, and the leftover "MWP" / "model-workspace-protocol"
naming from the source repo has been purged.

## Routing model

```
Root CLAUDE.md              "Where am I, what departments exist"   always loaded
Department CONTEXT.md       "What's here, what do I load"           read on entry
docs/ + shared/ + ownership-map.md   "What do we already know"      loaded selectively, stable
inbox/                       "What's new, not yet sorted"            loaded selectively, changes per drop
```

There is no per-stage contract layer — the one process this repo runs has its contract
inside `.claude/skills/process-inbox/SKILL.md` instead.

## Patterns

**One-Way Cross-References.** Departments may reference `shared/docs/`. `shared/docs/`
never references back into a specific department — department-specific detail stays in
that department's own doc with a pointer to the shared contract.

**Canonical Sources.** Every fact/rule has exactly one authoritative home; everything else
points to it, never duplicates it. Smell test: search the repo for a specific phrase — if
it appears in more than one file and both are meant to be authoritative, one needs to
become a pointer. Example: `_core/ownership-map.md` is the only place owner/team names live.

**CONTEXT.md = Routing, Not Content.** CONTEXT.md files only answer "what is this folder /
what do I load / what's the process" — never actual reference material. Keep under 80 lines.

**Conflict Checkpoints.** `/process-inbox` runs straight through every clear item and only
stops for ones that need a human decision — ambiguous department, or contradicts/duplicates
an existing canonical doc.

| Conflict | Agent presents | Human decides |
|---|---|---|
| Ambiguous department | The content + candidate departments | Which department, or defer to owner |
| Contradicts existing doc | The new content + the quoted conflicting line | Update in place, keep both, or defer to owner |

The option to defer to the subject-matter owner (named in `_core/ownership-map.md`) is
always on the menu.

**Docs Over Inbox Scraps.** Once a drop is integrated into a department's `docs/`, the
inbox copy is deleted, not kept as a second reference.

## Naming conventions

Lowercase-with-hyphens, no spaces, plain English, no em dashes anywhere in the repo.

## Quality guardrails

- CONTEXT.md files: under 80 lines
- Canonical docs in `docs/`: under 200 lines each (split by topic if longer)
- No em dashes anywhere in the repo
- Every folder that should persist gets a CONTEXT.md — the substitute for `.gitkeep`
- Plain English, no unexplained jargon

## Retired patterns

Kept for institutional memory only — do not follow these, they were pipeline-specific:

| Old pattern | Why retired |
|---|---|
| Stage Contracts | No stages; `process-inbox` has its own contract instead |
| Stage Handoffs via `output/` | No pipeline; `docs/` is permanent, not per-run |
| Selective Section Routing | Docs capped at 200 lines, read in full |
| Tool Prerequisites | Pure markdown KB, no external tools |
| Questionnaire Design / `setup` trigger | No per-workspace onboarding step |
| Bundled Skills (workspace-local `skills/`) | Skills live at the repo-standard `.claude/skills/` |
| Specs Are Contracts | Pipeline-specific spec-vs-build-stage distinction |
| Stage Audits | Folded into `process-inbox`'s conflict-handling step |
| Value Validation | Content-value framework, not applicable to a reference KB |
| Shared Constants | Code-value pattern, not applicable to markdown docs |
