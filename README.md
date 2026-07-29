# Get Sweaty Games — Knowledge Base

The org's single source of truth for knowledge that doesn't live inside a single code repo.
Organized by department: Engineering, Game Design, Design/Marketing. Not a build pipeline —
structured knowledge plus one manual sorting tool.

## How to add something

1. Write what you know into a markdown file and drop it in `inbox/`. Zero formatting
   requirements — don't tag it, don't pick a department.
2. Commit and push.
3. The next `/process-inbox` run sorts it, or asks a clarifying question if it's ambiguous.

## What `/process-inbox` does

- Reads everything in `inbox/`, works out the right department (or `shared/` for
  cross-department content), and checks it against existing docs for contradictions or
  duplicates.
- Files clean items into the right department, updates that department's doc index and
  `updates.md`, and clears the inbox.
- On ambiguity or contradiction: stops and asks — always offering the option to defer to
  the subject-matter owner.
- On defer: moves the item to `inbox/pending-owner-review/` with a note on who needs to
  decide. This is an in-repo flag only — it does **not** open a GitHub issue.

## Departments

| Department | Folder | GitHub team |
|---|---|---|
| Engineering | `departments/engineering/` | `Get-Sweaty-Games/Engineering` |
| Game Design | `departments/game-design/` | `Get-Sweaty-Games/GameDesign` + `Unity Devs` |
| Design/Marketing | `departments/design-marketing/` | none yet |

Full owner/defer-target detail: `_core/ownership-map.md`.

## Cross-department knowledge

Some content genuinely belongs to more than one department — a backend/Unity wire contract,
a shared brand asset, an org-wide decision. That lives in `shared/`, not duplicated into
each department. Departments may reference `shared/docs/`; `shared/docs/` never references
back into a specific department.

## Keeping up without reading everything

Run the `status` trigger, or open a department's `updates.md` directly for its most recent
additions.

## Methodology credit

Adapted from the [Interpretable Context Methodology](https://github.com/RinDig/Interpretable-Context-Methodology)
(ICM) by Jake Van Clief — folder structure as agent architecture. ICM's original pipeline
model (numbered stages producing per-run output) has been reshaped here into a standing,
department-based knowledge base. See `_core/CONVENTIONS.md` for exactly which patterns
carried over and which were retired.

## Internal use

See `LICENSE`.
