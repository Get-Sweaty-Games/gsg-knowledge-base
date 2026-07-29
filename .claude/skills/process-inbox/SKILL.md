---
name: process-inbox
description: Sorts raw drop-in files from inbox/ into the correct department folder (engineering, game-design, design-marketing) or shared/, checking each one against existing canonical docs for duplicates or contradictions. Stops and asks before integrating anything ambiguous or conflicting — always offering the option to defer to the subject-matter owner. Run manually with /process-inbox whenever inbox/ has new material; nothing here runs automatically.
---

# Process Inbox

## When to run this

Manually, whenever `inbox/` (excluding `pending-owner-review/`) has files sitting in it.
No CI, no git hook triggers this — it only runs when explicitly invoked.

## Before you start

Read `_core/ownership-map.md` in full. Every conflict question must match that table
exactly — never invent an owner or a department that isn't listed there.

## Step by step

1. List files directly in `inbox/` (not `pending-owner-review/`, not `CONTEXT.md`). If
   empty, report that and stop.
2. Read each file fully.
3. Determine candidate department(s): re-read `departments/*/CONTEXT.md`, and use the
   keyword signals from `_core/ownership-map.md` — backend/API/Supabase/infra points to
   Engineering; balance/narrative/systems-design vocabulary points to Game Design; C#/
   scene/prefab/engine-mechanics vocabulary points to Unity (technical, same folder as
   Game Design but a different defer target); brand/landing-page/marketing copy points
   to Design/Marketing. Treat an explicit "For: X" header in the file as a strong signal,
   but still sanity-check it against the actual content.
4. Check for contradiction or duplication (the canonical-sources smell test): search the
   candidate department's `docs/CONTEXT.md` index and `shared/docs/` for matching topics
   or overlapping phrasing.
5. Decide clear vs. conflicted:
   - Clear (one obvious department, no contradiction) → integrate.
   - Ambiguous department, or contradicts/duplicates an existing doc → stop, this is a
     conflict.
6. After the whole batch, print a summary: integrated count and where each landed,
   deferred count and who each is waiting on.

## Integrating a clear item

1. Pick a descriptive lowercase-hyphenated filename.
2. Write the content into `departments/<dept>/docs/<name>.md` (or `shared/docs/<name>.md`
   if it's genuinely cross-department).
3. Add a row to that folder's `docs/CONTEXT.md` index: Doc | Topic | Added | Source.
4. Add a row to the top of that folder's `updates.md`: Date | Doc | Summary | Dropped By.
5. Delete the original file from `inbox/`.

## Handling a conflict

1. Present the content summary and explain why it's a conflict — which departments are
   plausible, or which existing doc (quote the conflicting line) it contradicts.
2. Ask the user. Always include these options:
   - Integrate into department A
   - Integrate into department B (if a second department is genuinely plausible)
   - Integrate into `shared/` (if it's genuinely cross-department)
   - Defer to the subject-matter owner — name them from `_core/ownership-map.md`
   - Something else the user specifies
3. If a department or `shared/` is picked and the conflict was a contradiction, ask
   whether to update the existing canonical doc in place (the default recommendation) or
   keep both as noted variants.
4. If deferred:
   - Move the file, unedited, into `inbox/pending-owner-review/`.
   - Prepend a note block: `Deferred on [date]. Reason: [...]. Owner: [...]`.
   - Do not integrate it anywhere. Do not open a GitHub issue.

## Reference table

| File | What it's for |
|---|---|
| `_core/ownership-map.md` | Owner table — load before any conflict question |
| `departments/*/CONTEXT.md` | What each department covers |
| `departments/*/docs/CONTEXT.md` | Existing docs, for the contradiction check |
| `departments/*/updates.md` | Append a row on every integration |
| `inbox/pending-owner-review/` | Deferred items — never auto-filed as a GitHub issue |

## Notes

- Never invent a department not listed in `_core/ownership-map.md`.
- Never silently pick on ambiguity or contradiction — always ask.
- Non-markdown but readable-as-text files are processed the same way. Binary files: flag
  and ask how to proceed rather than guessing.
