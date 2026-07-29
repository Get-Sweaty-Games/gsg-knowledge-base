# New Department Template

How to add a department later (e.g. if Design/Marketing gets its own GitHub team, or a
new team like QA appears).

1. Create `departments/<name>/` with `CONTEXT.md`, `updates.md`, and `docs/CONTEXT.md`
   (skeletons below).
2. Add a row to `_core/ownership-map.md`.
3. Add the department to root `CLAUDE.md`'s Departments and Routing tables.
4. Update `.claude/skills/process-inbox/SKILL.md`'s candidate-department signal list to
   mention the new department.

## `departments/<name>/CONTEXT.md`

```markdown
# <Department Name>
<One-sentence description of what this department covers.>

## What's Here
| Location | Contains |
|---|---|
| docs/CONTEXT.md | Index of every canonical doc here, and what each covers |
| updates.md | Append-only recent-additions log — read for a quick catch-up |

## Owner
See `_core/ownership-map.md` for the current owner/team. Not restated here.

## Related
- `shared/CONTEXT.md` — cross-department material lives there instead
- `_core/CONVENTIONS.md` — the rules this folder follows
```

## `departments/<name>/docs/CONTEXT.md`

```markdown
# <Department Name> Docs Index
Canonical <department> documents. Maintained by /process-inbox — when it integrates a
new file, it adds a row below. Don't hand-edit the table without also editing/renaming
the underlying file.

| Doc | Topic | Added | Source |
|---|---|---|---|
```

## `departments/<name>/updates.md`

```markdown
# <Department Name> — Recent Updates
Append-only, newest first. A quick way for other departments to see what's new here
without reading every doc.

| Date | Doc | Summary | Dropped By |
|---|---|---|---|
```
