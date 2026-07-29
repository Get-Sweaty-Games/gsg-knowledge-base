# Inbox

Drop point for raw, untagged material. Zero friction: write what you know into a
markdown file (plain text is fine) and put it here. Don't tag it, don't decide the
department, don't format it — that's what `/process-inbox` is for.

## What Happens Next

Nothing, automatically. Processing is manual (`.claude/skills/process-inbox/SKILL.md`).
When run, the agent sorts each item into the right department (or `shared/`) and clears
it from here once integrated.

## If Something Can't Be Sorted Cleanly

Ambiguous or contradictory items get flagged for a human decision, never guessed. If
deferred to a subject-matter owner, the item moves to `pending-owner-review/` instead.

## Files Here Are Committed, Not Ignored

This folder is shared team state — one person drops a file, another may process it later
from a different clone. Don't add `inbox/*` to `.gitignore`.
