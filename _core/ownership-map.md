# Ownership Map

Canonical source for who owns each department and defer target. Nothing else in this
repo restates these names or teams — `CLAUDE.md`, `process-inbox`, and every department
`CONTEXT.md` point here instead of repeating it.

## Departments and defer targets

| Department | Folder | GitHub team | Primary contact | Covers |
|---|---|---|---|---|
| Engineering | `departments/engineering/` | `Get-Sweaty-Games/Engineering` | yarins0 | Backend, infra, data/API contracts |
| Game Design (content) | `departments/game-design/` | `Get-Sweaty-Games/GameDesign` | yarins0 | Balance, narrative, systems design |
| Unity (technical) | `departments/game-design/` | `Get-Sweaty-Games/Unity Devs` | yarins0 | C#, scenes, prefabs, engine mechanics — same folder as Game Design, different defer target |
| Design/Marketing | `departments/design-marketing/` | none yet | yarins0 (direct) | Landing page, brand, marketing copy |

## How `/process-inbox` uses this

- Backend/infra/API-flavored conflict → defer to Engineering.
- Balance/narrative/systems-design-flavored conflict → defer to Game Design.
- C#/scene/engine-mechanics-flavored conflict → defer to Unity Devs, even though the content
  lands in the same `departments/game-design/` folder as Game Design content.
- If a conflict could plausibly be either Game Design or Unity Devs, do not guess — present
  both as separate options when asking the user.
- Design/Marketing has no GitHub team — defer directly to yarins0.

## Team members (reference only, not for routing)

yarins0 (primary operator across all departments), nitai-zweig, nadavhanam12
