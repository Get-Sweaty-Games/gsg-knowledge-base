# Client Behavior Spec (Unity, engine-neutral)

> Filed from inbox `SPEC-behavior-derived.md` § Client. Folded in from the `godot-pivot` branch's
> client spec (2026-07-23), stated **engine-neutrally**. The engine is now decided — **Unity,
> 2026-07-28**, built by a partner studio under the Unity-as-base topology (see
> `departments/engineering/docs/unity-plugin-topology-decision.md`) — but this section stays
> engine-neutral on purpose: it describes behavior, and behavior changed neither when the engine got
> picked nor when the topology inverted. Keeping it that way is also what keeps it honest about the
> trust boundary, which is a property of the architecture rather than of the engine — a Kotlin plugin
> `.aar` is still native code outside the IL2CPP runtime, so the host-owned refresh-token guarantee is
> untouched by the flip. Engine-specific plumbing (autoloads, nav/back-stack, the per-request HTTP
> layer, the GDScript/C# port gotchas) is implementation and deliberately **not** specified here.
>
> The backend-side counterpart of this spec (Moat/Game/Edge — trust scoring, XP caps, gyms, guild
> RPCs) is filed as `departments/engineering/docs/backend-behavior-spec.md`.
>
> **Nothing about the client is proven live on device.** The C# that existed in this repo compiled
> clean on 2026-07-28, but that code is deleted and the client is being rebuilt by the partner
> against `docs/unity-client-brief.md`. Unproven: `hostbridge.aar` has not been built, delivered, or
> loaded; the manifest-fragment launcher override has not been exercised; no bridge round trip has
> run on real hardware — see `docs/PLAN-closeout.md` Phase 5 in the backend repo.

## Trust posture (client is untrusted — applies to every screen)

- The client **assembles an evidence bundle of raw signals and displays server-returned numbers**. It never computes an award, a trust score, or a reward. This holds identically for the meta systems (Roster/skill-tree, Game campaign) as for the Home workout loop — a mock returning a number is a **placeholder for a server response, never a client-side formula that later gets "promoted."**
- The one client-side numeric operation that touches rewards is presentational: the **claim count-up** animates from the pre-claim snapshot to the post-claim snapshot. Both numbers come from the server (`GET /state`), so it is a diff of **two server snapshots** — the trust boundary is untouched.
- `GET /state` is the single snapshot feeding the render layer; the client re-derives no progression formula.

## Auth / token split (host-owned refresh)

- The native **host** (Kotlin) does platform-native work only: health read → raw signals, OS permissions, hardware-backed token storage, **token refresh**, push, lifecycle. **App networking (`GET /state`, evidence upload) and the interactive OAuth login live in the client layer**, not the host.
- The one auth exception: **the host owns the token-refresh call.** The client performs interactive login and hands the initial access+refresh pair to the host in a single expression; the host stores both hardware-backed, silently refreshes against Supabase's token endpoint on access-token expiry, and exposes only the *current access token* to the client. **The refresh token is never retained, persisted, logged, or read back in client managed memory** — it transits the login response once and goes straight into host storage. This is the whole point of host-owned refresh: the long-lived crown jewel stays out of the most decompilable layer.
- **401 policy: never auto-relogin.** A 401 latches once (concurrent 401s from a fan-out act once), drops the user to the login/splash gate, and waits for a user gesture — an app-focus re-sync fires with no gesture, and the OS credential sheet throttles after repeated dismissals.

## Screen surface + scope

- **18 screens, tab bar Home / Roster / Game / Guild.** **Portrait-locked, phone-only** in this spec's original framing — **superseded by the B1 orientation decision** (landscape everywhere, including the splash) — see `departments/game-design/docs/unity-integration-owner-decisions.md`. Other stated constraints stand: no tablet support, no light/dark theming, English strings inline.
- **Backend-backed today:** Home (weekly-target hub + claim), sources, profile/name edit, settings, gyms, and the real half of Guild (create/join/invite/members).
- **Zero backend — mocked, quarantined:** **Roster/skill-tree**, **Guild weekly-credits**, and the **Game chapter campaign**. These have no tables, no routes, no `GameConfig` entries. They render against quarantined mock data so UI shape can be validated first; going real is a per-function swap to an API call with the screens unchanged. **The mock shape is not the API contract** — a separate backend design pass derives the schema and the mock adapts to it, never the reverse.
- **No offline queue / cache and no generic HTTP retry.** Every screen is server-backed (no signal ⇒ dead screen, accepted for MVP). Retry is user-visible only, because `POST /activities` is idempotent and register is exactly-once but `POST /guilds` is neither — a blanket retry would create duplicate guilds.

## The weekly loop — four product decisions (OPEN — NEEDS FOUNDER DECISION)

The three mocked meta systems all hang off one question — *"what is the weekly loop?"* — which `profiles.weekly_target` already anchors server-side. **These must be answered before the mocks are written**, since they determine the eventual schema, and each must be answered as *one* coherent loop (everything meta resolves at the weekly boundary), not four unrelated fictions. Recommended shapes below; the **numbers/cadence are the founders' call**. Whichever way they land, the trust boundary is unchanged — the server computes the value, the client displays it.

- **Guild bonus → flat, per-member-who-hit-target, capped** (recommended). Scaled/contribution-weighted bonuses punish being in a guild with a slow member and create a "carry" dynamic — backwards for a product selling accountability. *UI:* Guild is N lit/dim slots + one number. **The number is open.**
- **Builds → earned by completing a week, not bought with gold** (recommended). Routing acquisition through the verified workout (the moat), not a second currency, avoids a farmable economy loop while checkers are dormant. *UI:* a pick-one-of-three choice on week completion. **Cadence (every week vs milestone weeks) is open.**
- **Respec → free, at week boundaries only** (recommended). Irreversible trees make players avoid the tree; a paid respec is monetization this MVP lacks. *Schema note:* respec-able builds need an **owned-instance table, not a template column** — the template-vs-instance split (see the backend's `CLAUDE.md` / Moat docs) governs.
- **Battle → deterministic stat check, server-resolved, no combat resolver** (recommended). Answers "did my workouts make me stronger?" with a stat comparison in one screen; collapses the preview→fight split into a two-column comparison (effective stats vs the level's requirement) + one outcome.

This section is filed as-is — it documents an open product question, not a filing ambiguity. The
four numbers/cadences above are still owed a founder decision; nothing here should be read as
resolved.

## Backend design pass (not yet scoped)

Roster/skill-tree, guild weekly credits, and the chapter campaign require a **data-model + API design pass** (tables, routes, `GameConfig` entries) on the backend that does not exist yet. It is backend work sized separately and **blocked on the four decisions above**, since they determine the schema.
