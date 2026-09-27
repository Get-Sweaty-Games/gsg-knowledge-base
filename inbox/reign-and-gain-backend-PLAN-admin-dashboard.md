# PLAN — Admin dashboard

Status: **Phases 1-4 shipped and merged to `main` 2026-08-26 (PR #151, then #81 in
`gsg-landingpage`, then #155/#156 for Phase 4). Live-route verification (deferred from Phase 1) is
done — the operator ran the UI locally against the live backend and confirmed the routes work
under a real signed-in session.** Written 2026-08-25.

`0083_scheduled_notifications.sql` is confirmed applied on production (`scheduled_notifications`
exists, 0 rows). `ADMIN_USER_IDS` is presumed set in Render (deploy is green) but not curled end to
end — the three admin accounts sign in via Google OAuth, not a Supabase password, so there is no
`grant_type=password` shortcut to mint a token outside the app. **Live-route verification is
deferred to Phase 3**, once the UI can produce a real session itself.

Two corrections Phase 1 forced, both verified against the running code:

- The resolve route is `POST /admin/venues/:venueId/resolve`, not `/admin/gyms/:venueId/resolve`.
  `gyms.ts` already owns `POST /admin/gyms/:id/resolve`, and Fastify matches on path shape rather
  than parameter name, so a second route there throws `FST_ERR_DUPLICATED_ROUTE` at boot. The
  pending queue moved to `/admin/venues/pending` to match.
- `GET /admin/users` reads `profiles` and `v_user_lifecycle` separately instead of joining them.
  The view ends in `where p.is_dev is not true`, so a join would hide exactly the accounts whose
  `is_dev` flag an operator came to flip.

Also settled: `CORS_ORIGIN` is not declared in `render.yaml` at all, so it holds its code default
of `"*"`. Nothing needed adding.

An internal operator dashboard for the friends-and-family launch. It spans two repos: the API
lives here, the UI lives in `gsg-landingpage`.

## Why it exists

A BI tool (Metabase, Grafana, the Supabase SQL editor) already reads every analytics view with
zero code. The dashboard is **not** justified by the read half.

It is justified by the **write** half — three operator actions a BI tool cannot do:

1. Give a gym a manual verdict.
2. Flip a user's `manual_logging_enabled` and `is_dev` flags.
3. Send a push notification to a user.

The analytics panel rides along because it is nearly free once one read route exists.

Keep that ordering in mind if scope is ever cut. The writes are the product; the charts are the
bonus.

## Decisions locked

| Decision | Choice | Why |
|---|---|---|
| Admin auth | Supabase Auth + an allowlist of user ids | Real identity, per-person revocation, an audit trail that names a person |
| Allowlist storage | `ADMIN_USER_IDS` env var, comma-separated | No migration, no RLS surface, revocation is one Render edit |
| UI host | A new `/admin` route tree in `gsg-landingpage` | The Vite/React/Vercel app already exists and deploys |
| Build order | Backend routes first, then UI | Every route is curl-testable, so the UI is pure rendering |
| Sentry / PostHog / Render logs | **Link out. Do not wire.** | All three have a better dashboard than we would build. Sentry is not installed at all — adding it is its own issue, not a panel here |

### Why the allowlist is an env var, not a column

A `profiles.is_admin` column would be a server-trust flag. This repo's convention (see
`CLAUDE.md`, and `0018` for the reference case) requires a column-revoke from `authenticated` in
the same migration that adds any such flag.

An env var needs no migration, adds no RLS surface, and revocation is one Render edit. User ids
are identifiers, not secrets, so the var stays a plain Render var and does **not** go into Secret
Manager (`docs/secrets.md` draws that line).

It fails closed. Unset means no admins, matching `isAdminRequest`'s existing posture.

## The security escalation this represents

`ADMIN_API_KEY` today guards exactly one low-stakes action: a gym verdict. This dashboard puts
behind an admin gate the ability to read every user's analytics, flip server-trust flags, grant
cheats, and **send push notifications to real testers' devices**.

That is why the shared-key path was rejected. A shared string has no identity, no expiry, no
per-person revocation, and its audit row can only say that *someone* acted.

`POST /admin/push` sends a real notification to a real device. There is no preview and no undo.
The route requires an explicit `userId` and must **reject any wildcard or all-users shape**, so a
broadcast is impossible by construction until someone deliberately builds one.

## Verified facts

Everything below was checked against the live database or the code on 2026-08-25. Do not
re-derive it.

### Analytics — 22 views exist

`information_schema.views` and `supabase/migrations/` agree exactly. No drift.

```
v_activation_funnel      v_permission_outcomes    v_source_mix
v_boss_progress          v_retention_cohort       v_unrecognized_origins
v_currency_flow          v_roguelike_loop         v_user_active_days
v_dau_wau_mau            v_run_outcomes           v_user_lifecycle
v_engagement             v_screen_flow            v_verification_quality
v_growth_accounting      v_session_shape
v_l28                    v_signal_discrimination
v_onboarding_funnel      v_skill_economy
v_pending_award_backlog
```

Three of these are created early and later **dropped and recreated** — `v_engagement`,
`v_roguelike_loop`, `v_activation_funnel` all take their final form in `0080`, and `0050` replaced
`v_roguelike_loop` before that. Reading only the first `create view` gives a superseded
definition. Prefer `information_schema` over the migration files for "what exists".

**No view exposes PII.** Confirmed structurally: a query across all 22 views' columns for
`email|lat|lng|address|name|token|phone|ip` returns zero rows. Re-run that query before adding a
view to the allowlist.

**Allowlist is 21 of the 22.** Exclude `v_user_active_days`: it is the raw per-user-per-day spine
with unbounded growth, and every other view already derives from it.

Four views have a per-user grain (`v_user_lifecycle`, `v_currency_flow`, `v_l28`,
`v_pending_award_backlog`). They stay in. `v_user_lifecycle` is *documented* as a worklist — "an
`at_risk` row is a person to message" — which is exactly what an operator dashboard is for.
Rather than classify 22 views by expected size, the route carries **one `.limit()`**. That handles
growth for all of them at once.

### The analytics route needs no SQL

The backend's Supabase client holds the service role, and PostgREST reads a view by name:
`supabase.from(viewName).select("*")`. With an allowlisted name there is **no SQL string to
build**, so the injection surface is zero rather than merely guarded. Never interpolate the path
parameter into SQL; never accept a name that is not in the array.

### Gym approvals — the queue, and one trap

The queue is `gym_venues.validation_status = 'review_requested'`. Production currently holds
3 waiting, 5 `validated`, 1 `unverified`.

Status vocabulary (`0032`, `0039`): `pending`, `validated`, `unverified`, `review_requested`,
`rejected`.
- Automation sets `pending`, `validated`, `unverified`.
- A **user** sets `review_requested` via `POST /gyms/:id/request-verification`.
- Only an admin sets `rejected`.
- `unverified` is *eligible* for a user to request review. It is **not** itself in the queue.

A verdict is **venue-scoped and shared**. `gym_venues` is one row per physical place; `user_gyms`
is a thin per-user link to it. One verdict affects every user linked to that venue.

**THE TRAP.** `resolveValidation` takes a **`user_gyms.id` link id**, not a venue id
(`GymService.ts:641`, `venueIdForLink` at `:667`). The pending queue lives on `gym_venues`. A
naive `GET /admin/gyms/pending` returns **venue ids that `POST /gyms/:id/resolve` cannot accept**,
and every approval 404s.

The link-id indirection exists only because the *user's* client speaks in link ids. An admin does
not. So the admin resolve route takes a **venue id** directly.

`lat`/`lng` on `gym_venues` are precise coordinates. The schema models them as a gym, but
registration is self-reported — the code's own comment says "a cheater can register their couch."
No column distinguishes a real gym from a home address. An operator panel showing coordinates may
therefore be showing someone's home. Worth a note in the UI, not a blocker.

### Landing page (`gsg-landingpage/app`)

- **`@supabase/supabase-js` is NOT installed.** Admin login adds a dependency. The `/admin` route
  must be **lazy-loaded** so the marketing bundle does not carry it.
- **No shared fetch helper.** Three components (`ContactForm`, `WaitlistForm`,
  `DeleteAccountForm`) hand-roll the same inline `fetch` plus a duplicated error dictionary. Add a
  small `apiFetch` in `src/shared/` rather than a fourth copy.
- **No auth-guard pattern exists.** The admin tree is the first one.
- Routing is plain `react-router-dom` v7 in `src/App.tsx:44-64`. One nested-layout precedent
  exists at `/reignandgain` via `ReignAndGainLayout` + `<Outlet>`.
- Base URL is `import.meta.env.VITE_API_BASE_URL`. Any **new** `VITE_*` var needs an entry in
  `vite-plugin-assert-env.ts`'s `REQUIREMENTS` record, or the production build hard-fails. There is
  no bypass flag, by design.
- `vercel.json` has no headers block. `/admin` needs `X-Robots-Tag: noindex` added.
- Tailwind 4, CSS-first config in `src/index.css:1-24`. **No shared component primitives** — no
  `Button`, no `Input`. Every form hand-writes its own.
- `tsconfig.app.json` does **not** set `strict`. The admin UI gets weaker type checking than the
  API it calls. Not a blocker; know it.
- CORS is already configured from `env.CORS_ORIGIN` (`backend/src/app.ts:159`). Adding the origin
  is one value, not new code.

## Phase 1 — backend admin surface

| # | File | Change |
|---|---|---|
| 1 | `backend/src/config/env.ts` | Add `ADMIN_USER_IDS` to `EnvSchema`. Optional, comma-separated. |
| 2 | `backend/src/services/account/auth/adminUser.ts` | New. `requireAdminUser(req)` calls `authGateway.requireUser`, then checks the allowlist. 403 otherwise. |
| 3 | `backend/src/routes/admin.ts` | New. The routes below, under an `/admin` prefix. |
| 4 | `backend/src/docs/openapi.ts` | Document them under bearer auth. |
| 5 | `backend/src/routes/test/admin.test.ts` | New. Include a non-admin 403 and an off-allowlist view name. |
| 6 | `render.yaml` | Declare `ADMIN_USER_IDS`. Confirm `CORS_ORIGIN` carries the landing page origin. |

Routes:

1. `GET /admin/analytics/:view` — allowlisted name, read via PostgREST, with a `.limit()`.
2. `GET /admin/gyms/pending` — `gym_venues` where `validation_status = 'review_requested'`.
3. `POST /admin/gyms/:venueId/resolve` — **venue id**, verdict `validated` or `rejected`.
4. `GET /admin/users` — roster with both flags and lifecycle state.
5. `PATCH /admin/users/:id/flags` — set `manual_logging_enabled` and `is_dev`.
6. `POST /admin/push` — send now, over `notificationService.enqueue`. Explicit `userId` only.

Note on `is_dev`: every view in `0080` filters `is_dev is not true`. Flipping it also removes that
account from all analytics. That is correct, and it means the flag is one switch, not two.

Step 1 is files 1 and 2 only. They are testable alone and every route depends on them.

## Phase 2 — scheduled notifications

Shipped and merged 2026-08-26 (migration `0083_scheduled_notifications.sql`, confirmed applied on
production).

`notificationService.enqueue` was a misnomer: it sends immediately and synchronously, and no queue
table existed, so `POST /admin/push` had no way to express "send this tomorrow morning".

1. `scheduled_notifications` — user id, title, body, route, `send_after`, `send_after_zone`,
   `sent_at`. RLS on, no policies; service-role only, same posture as `contact_messages` (0041).
2. The tick drains it as **sweep #5**, above the telemetry prune. The prune's own comment says it
   runs last so housekeeping can never be the reason a push did not go out; appending the drain
   after it would have inverted exactly that reasoning.
3. **No `plpgsql`, so no integration test.** Exactly-once comes from one conditional
   `update ... where sent_at is null` that returns the row it touched — a single atomic Postgres
   statement, not logic anyone wrote. That is the letter *and* the spirit of the repo rule.

### Two operator-time modes

The push route takes two mutually exclusive fields, never a time plus a mode enum, so the two
meanings cannot disagree and no third mode can be expressed:

| Field | Meaning | Stored `send_after_zone` |
|---|---|---|
| `sendAfter` | an absolute instant, offset required — "09:00 **my** time" | `null` |
| `sendAfterLocal` | the recipient's wall clock — "09:00 **their** time" | the IANA zone |

Neither field means send now. **The Phase 3 UI renders this as a two-option radio**, and each
option maps to one field — the UI needs no third value and no mode string.

`sendAfterLocal` is resolved to an instant at schedule time, not at drain time: one row has exactly
one recipient, so nothing is left to resolve later and the drain stays free of all timezone math.
It reuses `utcInstantOfLocalWallClock` in `weekWindow.ts` (generalized from the existing
`utcInstantOfLocalMidnight`), so DST is handled by code the tick's tests already prove.

Two accepted consequences:

- **A later timezone change does not move an already-scheduled push.** `profiles.timezone` is
  rate-latched to one change per 24h (0053), so re-scheduling covers it.
- **A recipient whose `timezone` fails `Intl` gets a 400**, never a silent fall back to UTC.
  Sending an operator's push at the wrong hour without saying so is worse than refusing it.

### Precision is hourly, not to the minute

The drain runs in the tick cron (`0 * * * *`), so a push set for 09:30 sends at 10:00. Minutes are
still accepted — rounding up to the next hour is predictable, and an operator scheduling a nudge
does not care about the half hour. The delay is documented in `openapi.ts` and in the schema's
docstring; leaving it undocumented was the actual defect.

### The queue is editable, because a scheduled push is a loaded gun

A push scheduled for tomorrow morning with a typo reaches a real device in a live launch. Four
routes, not one:

```
POST   /admin/push                  202 sent now, or 201 with the created queue row
GET    /admin/push/scheduled        pending rows, soonest first
PATCH  /admin/push/scheduled/:id    edit title, body, route, or time
DELETE /admin/push/scheduled/:id    cancel
```

- **`userId` is not editable.** Re-aiming a message at a different person makes it a different
  push. Cancel and re-post.
- **`route` is nullable on the edit schema only**, so a deeplink set by mistake can be cleared.
  `optional` alone can only ever add one.
- **Both writes are conditional on `sent_at is null`** and answer **409** when the drain won the
  race. Each reads the row first purely to tell 404 from 409 — an unconditional write would edit a
  row that had already gone out and report success for a message nobody saw.
- **`sent_at` means a send was started, never delivered.** `enqueue` swallows every per-device
  failure, so no delivery signal exists to record. Same honesty limit as the 202.

## Phase 3 — the UI

A lazy-loaded `/admin` tree in `gsg-landingpage/app`, one page per panel, guarded by a Supabase
Auth session. Constraints are in "Landing page" above.

## Phase 4 — cleanup

**Shipped 2026-08-26.** `isAdminRequest`, `ADMIN_API_KEY`, and `POST /admin/gyms/:id/resolve` are
deleted from the codebase — `resolveValidation`/`venueIdForLink` in `GymService.ts` went with them,
since the deleted route was their only caller. **One admin auth scheme instead of two running side
by side.**

`docs/secrets.md` has a "Retired secrets" note. The GCP secret container (`rg-backend-admin-api-key`)
and both `.env` copies of the var are deleted too — nothing left referencing `ADMIN_API_KEY`
anywhere.

## Explicitly out of scope

- Wiring Sentry, PostHog, or Render logs into this dashboard. Link out instead. Installing Sentry
  in the backend is worthwhile and deserves its own issue; embedding it here does not.
- Any C# or Unity client work. That is the partner studio's, always.
- A broadcast / all-users push. Deliberately impossible in Phase 1.
