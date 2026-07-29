# Get Sweaty Games — Knowledge Base

Org-wide single source of truth. Drop raw specs/notes into `inbox/`, run `/process-inbox`
to sort them into the right department.

## Folder Map

```
Interpretable-Context-Methodology/
├── CLAUDE.md                          (you are here)
├── README.md                          (human-facing overview)
├── LICENSE                            (internal-use notice)
├── _core/
│   ├── CONVENTIONS.md                 (canonical rules — source of truth for all patterns)
│   ├── ownership-map.md               (who owns what, defer targets)
│   └── templates/new-department-template.md
├── departments/
│   ├── engineering/                   (backend, infra, API contracts)
│   ├── game-design/                   (Unity + game design content)
│   └── design-marketing/              (landing page, brand, marketing)
├── shared/                            (cross-department docs only)
└── inbox/                             (drop point + pending-owner-review/)
```

## Departments

| Department | Folder | Covers | Owner |
|---|---|---|---|
| Engineering | `departments/engineering/` | Backend, infra, API contracts | see `_core/ownership-map.md` |
| Game Design | `departments/game-design/` | Unity client + game design content | see `_core/ownership-map.md` |
| Design/Marketing | `departments/design-marketing/` | Landing page, brand, marketing | see `_core/ownership-map.md` |

## Routing

| You want to... | Go to |
|---|---|
| Drop in a new raw spec/note | `inbox/CONTEXT.md` |
| Sort what's in the inbox | run `/process-inbox` |
| Browse Engineering knowledge | `departments/engineering/CONTEXT.md` |
| Browse Game Design knowledge | `departments/game-design/CONTEXT.md` |
| Browse Design/Marketing knowledge | `departments/design-marketing/CONTEXT.md` |
| Find a cross-department doc | `shared/CONTEXT.md` |
| See who owns what | `_core/ownership-map.md` |
| Read the full conventions | `_core/CONVENTIONS.md` |
| Add a new department | `_core/templates/new-department-template.md` |

## Triggers

| Keyword | Action |
|---|---|
| `process-inbox` | Sort everything in `inbox/` into the right department (or flag conflicts) |
| `status` | Show inbox backlog, pending-owner-review count, recent additions per department |

### How `status` works

1. Count files directly in `inbox/` (excluding `CONTEXT.md` and `pending-owner-review/`) → backlog count.
2. Count files in `inbox/pending-owner-review/` (excluding `CONTEXT.md`) → pending count, listing which owner each is waiting on (from its note block).
3. For each department and `shared/`, read the top 3 rows of `updates.md` → "Recently added."
4. Render a short report.

## How This Repo Works

Built on the Interpretable Context Methodology (ICM) — reshaped from ICM's linear content
pipelines into a standing, department-organized knowledge base. No stages, no pipelines,
just departments and one manual sorting skill. Full pattern set in `_core/CONVENTIONS.md`.
