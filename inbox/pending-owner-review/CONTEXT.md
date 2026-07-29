# Pending Owner Review

Items deferred during a past `/process-inbox` run because the conflict needed a specific
subject-matter owner's decision. Each file has a note block at the top: when deferred,
why, and which owner (from `_core/ownership-map.md`) needs to decide.

## Resolving One

Once the owner decides, either integrate by hand (removing the note block), or re-run
`/process-inbox` and tell the agent the decision.

## Not a GitHub Issue

Deferring here is a lightweight in-repo flag only. It does not create or update anything
on GitHub.
