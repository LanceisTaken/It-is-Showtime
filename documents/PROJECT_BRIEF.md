# Cinema Showtimes Aggregator — Project Brief

A web app that aggregates movie showtimes from Malaysian cinemas into one place. Starting with **TGV** and **GSC**; other chains (MBO, LFS, mmCineplexes, etc.) come later.

> **Status note (2026-07-19):** Data-acquisition feasibility has been spiked and confirmed for both launch chains — see *Data acquisition*. The tech stack is now decided — see *Tech stack*.

---

## Core idea: time-first (the differentiator)

Existing aggregators (Cinema Online, Popcorn) and the chains' own apps are **location-first**: you pick a cinema, then see its showtimes. That assumes you've already decided where to go.

This app is **time-first**. Many people are flexible on location but strict on time. The flow is:

1. Pick a **state** (e.g. Selangor)
2. Pick a **movie** now showing in that state
3. See **all of today's showings for that movie, sorted by time**, each tagged with its cinema and format

The whole product is one dataset viewed by time instead of by location. The time-first pivot is the actual reason to use this over what already exists — it works entirely with plain menus and needs no AI.

---

## User flow (MVP)

State → movie → chronological list of every showing for that movie, each row showing time, cinema, and format, deep-linking out to the chain's own booking page.

---

## Key UX rules

- **Do not group strictly by exact timestamp.** Cinemas rarely share the same minute (8:00 vs 8:15 vs 8:20). Render a single chronological list of every showing sorted by time; only stack entries together when they genuinely share a time.
- **Show format on every showing** (Standard / IMAX / D-Box / etc.). A 2:30 IMAX and a 2:30 Standard are different products with different prices.
- **Location is a tiebreaker, not a filter** (nice-to-have, not MVP). When multiple cinemas share a time, sort nearest-first. Store each cinema's coordinates so this can be added later without re-architecting.
- Each showing deep-links to the chain's booking page. This app surfaces times; it does not handle payment.
- **Show a freshness timestamp** ("Showtimes as of 2:15pm"). Costs nothing, buys trust, and covers brief staleness between scrapes.

---

## Tech stack

Decided stack, with rationale and what was deliberately deferred or cut. The guiding principle: Supabase already bundles auth, `pgvector`, `pg_cron`, and object storage, so avoid buying point solutions for things it ships in the box until there's a real reason.

### Core (build on now)

| Component | Choice | Role |
|---|---|---|
| Backend / DB | **Supabase** (Postgres) | Relational data model (`showings`, `cinemas`, `movies`), auth-ready, `pgvector` + `pg_cron` + storage bundled |
| Deploy | **Vercel** | Frontend hosting, Cron for scheduled scrapes |
| Domain | **Namecheap** | Domain registration |
| DNS / edge | **Cloudflare** | DNS, caching, proxy, DDoS |
| Analytics | **PostHog** | Product analytics |
| Error tracking | **Sentry** | Pipeline + app error tracking — high value; a silently-dead scraper is the classic aggregator failure mode |
| Poster/image storage | **Supabase Storage** | Movie posters (no separate service needed) |

### Phase 2 — planner (add when building the AI planner)

| Component | Choice | Role |
|---|---|---|
| LLM API | **Anthropic — Claude Haiku 4.5** | Parse request → intent JSON; solver output → prose. Ideal for cheap structured extraction |
| Routing / directions | **Google Routes API** | Traffic-aware travel time, waypoints for pickups |
| Geocoding | **Google Maps Platform** | "Lives in Subang" → coordinates (same billing bundle as Routes; $200/mo shared credit) |

### Deferred (chosen, not set up yet)

- **Stripe** — payments only ever sit on the *planner*, if a paywall ever exists. No ticket payment (we deep-link out). Back-pocket, not now.
- **Upstash (Redis)** — useful later for caching routing/geocoding responses and rate-limiting. For MVP, Supabase can hold cached routes. Optional, not load-bearing.

### Cut

- **Clerk** — MVP browsing is anonymous; accounts only appear for *saved addresses* (optional/Phase 2). Overlaps Supabase Auth, and running both means syncing Clerk IDs into Supabase RLS for zero MVP benefit. Use Supabase Auth if/when accounts are needed.
- **Pinecone** — nothing here needs a vector DB. Movie selection is a menu; the planner is structured extraction + explanation, not semantic search. Fuzzy title matching is handled by Postgres trigram/full-text search. If vectors are ever needed, `pgvector` is already in Supabase.

### Frontend prototyping tool

**v0** (Vercel). Chosen for fit, not hype: outputs Next.js + Tailwind + shadcn/ui, deploys straight into the Vercel pipeline, generates real components you keep (so the prototype's menu flow becomes the real app's starting components), and doesn't try to own the data layer — which suits deliberately building the Supabase schema separately. Prototype the three-screen menu flow against the **real normalized `Showing` schema** and a slice of the spike's sample JSON, so components are built around the fields actually queried (`time`, `cinema_name`, `format`, `booking_url`) and nothing needs reshaping when real data lands.

*(Lovable and Replit were considered. Lovable auto-provisions a Supabase backend for non-technical builders — the opposite of frontend-only against an existing planned schema. Replit is a full-stack host with unpredictable effort-based pricing — overkill for a bounded frontend mock when Vercel + Supabase are already chosen.)*

---

## Data acquisition — feasibility CONFIRMED

Neither TGV nor GSC offers a public API, but a spike (2026-07-19, Selangor) confirmed **both are reproducible via unauthenticated read-only API replay — no browser automation, no auth required.** This retires the single biggest project risk and simplifies the runtime: plain `fetch` of JSON/XML, no headless browser needed.

### Confirmed method per chain

- **GSC** — public **XML** API (`epaymentapi.gsc.com.my`, `showtimews/service.asmx`). **Movie-keyed** traversal: loop movies, fetch showtimes each. 40 clean Selangor rows extracted in the spike; independently matched against Cinema Online listings.
- **TGV** — public **JSON** API (`api.tgv.com.my`, `/api/cinemas/v1` + `/api/boxoffice/v1`). **Cinema-keyed** traversal: loop cinemas, fetch sessions each. 119 rows across 11 Selangor cinemas; independently matched.

Both graded **YELLOW** pending three fixes before GREEN:

1. **Booking URL (highest priority — only user-facing failure mode).** Both chains expose the parameters but the final deep link is constructed, not observed. A link that 404s or lands on the wrong movie destroys trust instantly. **Browser-verify the resolved URL for both chains, codify the template, and add a test asserting the constructed URL matches the observed shape.**
2. **TGV title enrichment.** Bulk session responses omit movie title. Fetch the movie catalog once, build a `movie_id → title` map, enrich in memory. Confirm the catalog is a single bulk call (even per-movie is cheap — only ~20–30 films — and fully cacheable).
3. **GSC state filtering.** Don't infer state from address on every run. **Seed the `cinemas` reference table once** (cinema ID → state, plus coordinates for nearest-first later). Same fix applies to TGV. This converts a fragile per-scrape string-parse into a maintained lookup.

### Runtime & scheduling (resolved by the spike)

Because no browser is needed and call volume is tiny (~a dozen TGV calls, a few dozen GSC per state/day), the scraper fits in a lightweight scheduled function — **GitHub Actions cron, Supabase Edge Function, or Vercel Cron**, triggered by `pg_cron` or Vercel Cron. No dedicated long-running worker required. Scrape gently and cache aggressively — good citizenship and staying under any rate-limit radar.

### Provider abstraction

Build a **provider abstraction** around the replayed calls: provider-specific normalizers absorb the traversal asymmetry (GSC movie-keyed vs TGV cinema-keyed), but the contract is uniform — each provider returns `Showing[]`, leaking none of its loop shape. A validation job compares a small sample against the provider's own rendered page (treat that as ground truth — Cinema Online is a convenience check but is itself another aggregator that can drift).

### Idempotent upserts

Both chains expose their own IDs (TGV `session_id`, GSC seat-selection IDs). **Use the provider's session/showing ID as the external unique key**, falling back to the composite `chain + cinema + movie + date + time + format`. Provider IDs survive redesigns better than a composite that shifts when a format is relabeled. Re-scrapes must upsert cleanly without duplication.

### Operational monitoring

Beyond Sentry (which catches crashes), add a **dead-pipeline alarm**: if a chain's daily showing count drops to zero or falls off a cliff vs. yesterday, alert. Silent emptiness — a "successful" run returning zero rows because an endpoint quietly changed — is the aggregator killer that crash-tracking misses.

**Maintenance note:** an aggregator is "a thing you keep alive," not "build once." Endpoints and page structures change without warning. Save real captured payloads as **test fixtures** so normalization tests run against reality and a shape change surfaces as a failing test.

---

## Data model

A single normalized collection of showings:

```
Showing {
  movie_title   string
  movie_id      string        // internal, for grouping across chains
  cinema_name   string
  chain         enum(TGV, GSC, ...)
  state         string
  lat, lng      float         // for nearest-first sorting later
  date          date
  time          time
  format        string        // Standard, IMAX, D-Box, ...
  booking_url   string        // deep link to the chain's booking page
  provider_id   string        // chain's own session/showing ID — upsert key
}
```

Supporting reference data: a **`cinemas`** list (name, chain, state, coordinates — **seed this early**, it fixes state filtering and pre-loads nearest-first) and a **`movies`** list (title, poster, runtime, rating — also backs TGV title enrichment).

Keep the location and surrounding-context fields (coordinates, nearby food/transport) in the model from the start even if nothing uses them yet — that's where any future monetization (affiliate links, promoted placement) lives, and it's cheap to leave the hooks in versus retrofitting.

---

## Build order (phasing)

Build in this order. Each phase is useful on its own; do not start the next until the current one is solid.

1. **Data pipeline** — reliably pull a clean `{movie, cinema, date, time, format}` table from TGV and GSC into Supabase on a schedule. Feasibility is confirmed; remaining work is the three YELLOW fixes, the provider abstraction, upserts, and monitoring. This is the whole ballgame; everything else is a query over it.
2. **Time-first menus (MVP)** — the state → movie → sorted-showings flow, prototyped in v0 against real sample data, then wired to Supabase. This is the real differentiator and ships with zero AI. **Get this live and used before touching Phase 3** — if the menu doesn't pull users, the planner won't save it.
3. **AI planner (Phase 2, below)** — a natural-language layer on top of the menus. The menus remain the fallback / power-user path.

---

## Phase 2 — the AI planner

A natural-language layer that answers requests the menus can't, e.g.:

> "Help me pick the best time for a movie for me and two friends. A lives in Subang, free 6–10pm. B lives in Cheras, free after 7. I'm picking up A; B comes on their own. We want to eat first."

It returns the best feasible showtime(s), or an honest "no good option and here's why," plus derived details like how long the group has to eat.

### Architecture: AI at the edges, deterministic core

**The AI must never do the arithmetic.** LLMs are unreliable at math and will confidently produce wrong arrival times. The model is used only at the two edges — reading the messy request into structured data, and explaining the final answer in plain language. Everything between is ordinary deterministic code plus a real routing API.

```
Group's request (plain text)
      │
      ▼
[LLM]  parse to structured intent      ← Claude Haiku 4.5
      │
      ▼
Geocode addresses + fetch candidate showings   ← Google Maps Platform + Supabase
      │
      ▼
Routing API: travel time per journey (traffic-aware)   ← Google Routes API
      │
      ▼
[Solver] feasibility + ranking          ← your code, NO AI. Correctness lives here.
      │
      ▼
[LLM]  explain the result in plain language    ← Claude Haiku 4.5
      │
      ▼
Answer the group can act on
```

### Required components

| Component | What it does | Build or buy |
|---|---|---|
| LLM API (Claude Haiku 4.5) | Parse request → intent JSON; turn solver output → prose | Buy (per-token) |
| Intent schema + validator | Fixed JSON shape the model must emit; reject/repair malformed output | Build |
| Geocoding (Google Maps Platform) | "Lives in Subang" → coordinates | Buy (or avoid via saved addresses) |
| Routing / directions API (Google Routes) | Travel time between points, traffic-aware, by mode, **waypoints** for pickups | Buy (per-request) |
| Solver | Backward-calculates leave-by times, checks every journey against availability, ranks feasible showtimes | Build (this is the core) |
| Buffer constants | Pre-movie overhead (parking, tickets, snacks), walk-to-hall time | Build (config) |
| Saved addresses (optional) | Store home addresses so geocoding is done once, not per request | Build |

**Pickups map cleanly to waypoints.** "I pick up A, then drive to the cinema" is one routing call: home → A's home → cinema. "B comes alone" is a separate call. The group becomes a set of independent journeys that must all reach the cinema by the deadline — no graph theory needed.

### The intent schema

The LLM is prompted to output **only** JSON in this shape (and to ask a clarifying question instead of guessing when the request is ambiguous):

```json
{
  "movie": "Dune: Part Two",        // or null = flexible
  "date": "2026-07-19",
  "participants": [
    { "name": "me", "origin": "Petaling Jaya", "available": ["17:30", "23:00"], "transport": "car" },
    { "name": "A",  "origin": "Subang",        "available": ["18:00", "22:00"], "transport": "picked_up" },
    { "name": "B",  "origin": "Cheras",        "available": ["19:00", "23:30"], "transport": "car" }
  ],
  "pickups": [ { "driver": "me", "passenger": "A" } ],
  "wants_to_eat": true,
  "eat_minutes": 60
}
```

### Solver logic (deterministic — no AI)

For each candidate showtime `T`:

1. `deadline = T − pre_movie_buffer` (parking, ticket collection, snacks — e.g. 15–20 min).
2. Build the journeys from `participants` + `pickups`. A pickup journey has a waypoint.
3. For each journey, get its total travel time from the routing API (traffic-aware for `deadline`).
4. `leave_by = deadline − journey_travel_time`. The journey is OK if `leave_by` falls inside every participant-in-that-journey's availability window (and, for a pickup, the passenger is available at pickup time).
5. `T` is **feasible only if every journey is OK.**
6. Among feasible `T`, rank by preference (earliest, most eating slack, or preferred format).
7. If none are feasible, return the **binding constraint** ("B isn't free until 19:00; the earliest showing still needs them leaving by 18:40") so the LLM can explain why.

**Eating slack** = `deadline − group_convergence_time − walk_to_hall`. Negative slack means "no time to eat" at that showtime.

**Efficiency:** travel time from a fixed origin to a fixed cinema barely changes across an evening, so compute each journey's duration per candidate cinema once, then run all feasibility checks as free arithmetic. Only the routing calls cost money. Traffic is time-dependent (6pm ≠ 9pm) — pass an arrival time to the routing API so a 6pm estimate isn't applied to a 9pm showing.

### Cost estimate (mid-2026 figures — verify current pricing)

Per-unit:

- **LLM (Claude Haiku 4.5, ideal for extraction):** ~$1 per million input tokens, ~$5 per million output. A plan makes one parse call + one explain call, each small (~1–2K in, ~300 out). **Well under 1 cent per plan** on Haiku; a few cents if using a larger model. Prompt caching (up to 90% off cached input) and batch processing (50% off) reduce this further.
- **Routing (Google Routes API):** on the order of ~$5 per 1,000 calls (~half a cent each). A plan makes one call per journey (typically 2–4). ~1–2 cents per plan.
- **Geocoding:** ~$5 per 1,000 (~half a cent each), but near-zero in practice if home addresses are saved and cached.

**All-in: roughly 1–3 cents per plan**, dominated by routing, not the AI. Google Maps includes a **$200/month shared credit** across Maps/Routes/Places, so early usage is effectively free. LLM APIs are pay-as-you-go with prepaid credits.

Design implication: this per-use cost is why any future paywall, if one ever exists, would sit on the *planner* (Stripe, deferred), not on the free menu browsing.

---

## Testing (the non-AI code)

All of this is deterministic and must be unit-tested independently of the LLM. The LLM is mocked in these tests — you feed the solver a fixed intent object and assert on its output.

### 1. Data normalization

- Raw TGV JSON payload → correct `Showing` objects (all fields mapped) — run against **saved real fixtures**, not hand-written mocks.
- Raw GSC XML payload → correct `Showing` objects — same.
- Format detection: IMAX / D-Box / Standard correctly tagged from each chain's raw labels.
- Missing/optional fields handled without crashing (e.g. TGV bulk session missing title → enriched from catalog map).
- Date and time parsed into the right timezone (Malaysia, UTC+8).
- Two chains' data merge into one collection without duplication or field collisions — assert `provider_id`-based upsert dedupes re-scrapes.

### 2. Time-first view logic

- Showings returned in strict chronological order.
- Showings at genuinely the same time are stacked; near-but-different times are **not** merged (8:00, 8:15, 8:20 stay as three rows).
- Format badge preserved on every row (a 2:30 IMAX and 2:30 Standard remain distinct).
- Filtering by state returns only that state's showings (via seeded `cinemas` lookup).
- Empty case: movie with no showings today returns an empty list, not an error.

### 3. Routing / solver algorithm (the big one)

Mock the routing API so travel times are fixed and known, then assert on feasibility and ranking.

**Single person, single journey**
- Leave-by inside availability window → **feasible**.
- Leave-by before the window opens → **infeasible**.
- Leave-by exactly on the window boundary → define and test the boundary rule (inclusive vs exclusive).
- `deadline = showtime − buffer` computed correctly.

**Pickup journey (waypoint)**
- Travel time = home → passenger → cinema (waypoint included, not a direct route).
- Driver's leave-by respects the **driver's** window.
- Passenger must be available at pickup time; if not → infeasible even if the driver is free.

**Multiple journeys converging**
- Showtime is feasible only if **all** journeys clear; one failing journey makes it infeasible.
- Two journeys that individually work but need different showtimes → correct per-showtime result.

**No feasible option**
- When nothing works, the solver returns the **binding constraint** (which person/window blocked it), not just "no."
- The reason is specific enough for the LLM to explain ("B free from 19:00; earliest showing needs leave-by 18:40").

**Ranking**
- Among multiple feasible showtimes, correct ordering under each strategy (earliest / most eating slack / preferred format).
- Tie between two feasible showtimes broken deterministically.

**Eating slack**
- Correct slack given convergence time and deadline.
- Negative slack → reported as "no time to eat," not a negative number leaking through.
- `wants_to_eat: false` → eating is not considered in feasibility.

**Traffic / time dependency**
- Solver requests routing with the correct arrival time per showtime (a 9pm showing uses 9pm traffic, not 6pm).

**Edge cases**
- Everyone at the same origin.
- One person with an availability window shorter than the movie's travel + buffer (never feasible).
- Overlapping windows that only just barely permit one showtime.

### 4. Tiebreaker + deep links

- When multiple cinemas share a time, they sort nearest-first by the stored coordinates.
- Distance sort is correct for a few hand-computed cases.
- `booking_url` generated correctly per chain and per showing — **assert the constructed URL matches the browser-verified template** (the highest-priority YELLOW fix).


### Data lifecycle — filtering vs. pruning

Two separate jobs that must never be conflated: **what users see** (a read-time filter) and **what the table holds** (a scheduled prune). The freshness of what users see must never depend on a delete having run.

#### Filtering past showings (read time)

Showings are hidden the moment their start time passes, but the rows are **not** deleted to achieve this. Browsing at 9pm should surface only 9pm-onward showings — a 5pm showing the same day is filtered out, not removed. Because `date` and `time` are stored separately, compare their combination against current Malaysia time:

```sql
SELECT *
FROM showings
WHERE (date + time) >= (now() AT TIME ZONE 'Asia/Kuala_Lumpur')
ORDER BY (date + time);
```

`(date + time)` composes the two columns into a timestamp; comparing against Malaysia-local `now()` (UTC+8) drops everything already started while keeping all future showings. This is the only mechanism for hiding elapsed showings — deletion plays no part in it.

#### Pruning old data (weekly)

Past-date rows are genuinely redundant — nothing in the app reads them — so they're cleared on a **weekly** `pg_cron` job for tidiness. This is housekeeping, not a survival requirement: at national coverage a full year is still well under a gigabyte, so storage is not a near-term constraint. The prune keeps the table to roughly today plus any future days, with up to a week of already-past days lingering between runs (harmless).

```sql
-- runs weekly
DELETE FROM showings
WHERE date < CURRENT_DATE;
```

Prune by whole past **days** (`date < CURRENT_DATE`), never by individual elapsed showtime. Deleting rows the instant each showtime passes would make today's row count shrink through the afternoon, breaking the dead-pipeline alarm (which compares today's count against yesterday's) — it couldn't tell a real scraper failure from normal time passing. Whole-day pruning keeps each day's count stable while the day is live.

**Timezone caveat (applies to both jobs).** Run the prune on the same clock the dates are stored in. Dates are Malaysia time (UTC+8); if the cron fires in UTC, `CURRENT_DATE` rolls over at 8am local rather than midnight, so a few hours of "yesterday" would survive longer than expected. Set the job's timezone to `Asia/Kuala_Lumpur` (or compare against a Malaysia-time date) so both the filter and the prune behave as intended.