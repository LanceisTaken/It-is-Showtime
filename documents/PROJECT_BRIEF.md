# Cinema Showtimes Aggregator — Project Brief

Aggregates Malaysian cinema showtimes into one **time-first** view. Launch chains: **TGV + GSC**. Others (MBO, LFS, mmCineplexes) later.

**Differentiator:** existing aggregators (Cinema Online, Popcorn, xinemas) are location-first (pick cinema → see its times). This is time-first: pick a movie, pick your cinemas, see all their times together sorted by time, click a time → chain's booking page. Works with plain menus, no AI. AI planner is Phase 2 only, never paywalled-core.

---

## MVP flow

**Homepage** → movie list (own now-showing set, derived from movies with active showings; sorted by `release_date` desc). Horizontally scrollable strip, not a paginated table. No trending section, no editorial/blog, no admin reviews, no upcoming-movies strip, no location-first Cinemas page. (All cut from xinemas reference.)

**Movie page** → details (poster, cast, director, runtime, genre, synopsis, trailer embed) + external review **scores** (IMDb, Rotten Tomatoes, Letterboxd — display their numbers, never write our own). Then the showtimes section:

1. Pick **state**.
2. Pick **day** — interactive day strip (GSC-style), defaults to today.
3. Pick **cinemas** — multi-select, required (users have a shortlist; defaulting to all-in-state floods the view). Prompted right after state.
4. **Showtimes render grouped by format**, each format its own section by its **raw label** (no normalization/aliasing — TGV "Regular" and GSC "Standard" stay separate sections; chain-exclusive halls like "Beanie" just show that chain's times). Within each section: time boxes left→right ascending, **cinema name under the time**. No ticket price.
5. Click a box → chain's **movie-level booking page** (user re-picks date/cinema/time on the chain site; pre-selecting is out of scope). Movie-level URL is far more stable than session-level — de-risks the booking-URL problem.

**UX rules:** don't merge near-but-different times (8:00/8:15/8:20 = three boxes); only stack genuinely identical times. Show freshness timestamp ("Showtimes as of 2:15pm"). Store cinema coordinates now for future nearest-first sort (not MVP).

---

## Metadata & reviews

- **Metadata** (poster, cast, director, runtime, genre, synopsis, trailer YouTube key): plan to use **TMDB** (clean free API, one source for all of it). Trailer embed can alternatively reuse whatever link TGV/GSC use. *(Not finalized — Lance evaluating TMDB.)*
- **Review scores** (non-negotiable sources): **IMDb, Rotten Tomatoes, Letterboxd**. None have clean free APIs — all effectively scrape-only (Letterboxd has an approval-gated API). Scrape on slow cadence (daily+), cache against movie record, failed scrape → "score unavailable", never breaks page.
- **Title matching** across TMDB + 3 review sites = same normalize-then-fuzzy-match problem as cross-chain. Keep a manual ID/alias table per source (store IMDb `tt` id, RT slug, Letterboxd slug once, reuse).

---

## Tech stack

**Core (now):**
- **Supabase** (Postgres) — data (`showings`, `cinemas`, `movies`), plus bundled `pg_cron`, `pgvector`, Storage (posters), Auth-ready. Principle: use what Supabase bundles before buying point solutions.
- **Vercel** — frontend host + Cron for scrapes.
- **Cloudflare** — DNS/edge. **Namecheap** — domain.
- **PostHog** — analytics. **Sentry** — error tracking.
- **v0** — frontend prototyping (outputs Next.js + Tailwind + shadcn/ui, keeps real components). Prototype against the real `Showing` schema + spike sample data.

**Phase 2 (AI planner):** Claude Haiku 4.5 (parse/explain), Google Routes API (traffic-aware travel + pickup waypoints), Google Maps Platform (geocoding; $200/mo shared credit).

**Deferred:** Stripe (only ever on planner paywall, if ever), Upstash/Redis (Supabase caches routes for now).
**Cut:** Clerk (Supabase Auth covers it), Pinecone (no vector need; pgvector if ever).

---

## Data acquisition — feasibility CONFIRMED (2026-07-19, Selangor)

Both chains reproducible via **unauthenticated read-only API replay** — plain `fetch`, no browser, no auth.

- **GSC** — XML API (`epaymentapi.gsc.com.my`, `showtimews/service.asmx`). **Movie-keyed**: loop movies → fetch showtimes.
- **TGV** — JSON API (`api.tgv.com.my`, `/api/cinemas/v1` + `/api/boxoffice/v1`). **Cinema-keyed**: loop cinemas → fetch sessions.

**Three fixes before GREEN:**
1. **Booking URL (highest priority — only user-facing failure mode).** Deep link is constructed, not observed. Browser-verify resolved URL for both chains, codify template, add a test asserting constructed URL matches. (Now movie-level target, so lower risk.)
2. **TGV title enrichment.** Bulk sessions omit title. Fetch catalog once → `movie_id → title` map → enrich in memory.
3. **State filtering.** Seed `cinemas` table once (cinema id → state + coordinates) instead of parsing address per run. Applies to both chains.

**Pipeline design:**
- **Provider abstraction** — per-chain normalizers absorb traversal asymmetry (GSC movie-keyed vs TGV cinema-keyed); uniform contract: each returns `Showing[]`.
- **Idempotent upserts** — key on provider's own session/showing id; fallback composite `chain+cinema+movie+date+time+format`. Re-scrapes must not duplicate.
- **Format sections** — group by raw format label from the chain, no normalization (see MVP flow step 4).
- **Runtime** — tiny call volume; lightweight scheduled function (Vercel Cron / Supabase Edge Fn / GitHub Actions, triggered by pg_cron or Vercel Cron). Scrape gently, cache.
- **Monitoring** — Sentry catches crashes; add **dead-pipeline alarm**: alert if a chain's daily showing count hits zero / drops off a cliff vs yesterday (silent empty runs are the classic aggregator killer).
- **Test fixtures** — save real captured payloads; normalization tests run against reality so a shape change fails a test.

---

## Data model

```
Showing {
  movie_title  string
  movie_id     string        // internal, groups across chains
  cinema_name  string
  chain        enum(TGV, GSC, ...)
  state        string
  lat, lng     float         // future nearest-first sort
  date         date
  time         time
  format       string        // raw chain label
  booking_url  string        // movie-level deep link
  provider_id  string        // chain session/showing id — upsert key
}
```

Reference tables: **`cinemas`** (name, chain, state, coordinates — seed early) and **`movies`** (title, poster, runtime, rating, external ids/slugs). Keep coordinate/context fields even if unused (future affiliate/monetization hooks are cheap to leave in, costly to retrofit).

**Cross-chain movie identity:** only join key is title string → normalize-then-fuzzy-match + small manual alias table. Deterministic, not ML.

---

## Data lifecycle — filtering ≠ pruning

Two separate jobs; freshness must never depend on a delete having run. Both in Malaysia time (UTC+8).

**Read-time filter (hides elapsed showings, deletes nothing):**
```sql
SELECT * FROM showings
WHERE (date + time) >= (now() AT TIME ZONE 'Asia/Kuala_Lumpur')
ORDER BY (date + time);
```

**Weekly prune (housekeeping only — storage is not a constraint):**
```sql
DELETE FROM showings WHERE date < CURRENT_DATE;  -- pg_cron, Asia/Kuala_Lumpur tz
```
Prune whole past **days**, never per-elapsed-showtime — deleting live-day rows would make today's count shrink through the day and break the dead-pipeline alarm.

---

## Build order

1. **Data pipeline** — clean `{movie, cinema, date, time, format}` from TGV+GSC into Supabase on schedule. The whole ballgame; everything else queries it. Remaining: 3 YELLOW fixes, provider abstraction, upserts, monitoring.
2. **Time-first menus (MVP)** — the flow above, prototyped in v0, wired to Supabase. Ship and get used before Phase 3.
3. **AI planner (Phase 2)** — natural-language layer over the menus; menus stay as fallback.

---

## Phase 2 — AI planner (summary)

Natural-language group scheduling, e.g. "pick best time for me + 2 friends, A in Subang free 6–10, B in Cheras free after 7, I pick up A, we want to eat first." Returns best feasible showtime(s) or an honest "no option + why."

**Architecture: AI at the edges, deterministic core.** LLM never does arithmetic.
```
request text → [LLM parse → intent JSON] → geocode + fetch showings
→ [Routing API travel times] → [SOLVER: feasibility + ranking, NO AI]
→ [LLM explain] → answer
```

- **Intent JSON:** `movie` (or null), `date`, `participants[]` (name, origin, available window, transport), `pickups[]` (driver→passenger), `wants_to_eat`, `eat_minutes`. LLM asks a clarifying Q rather than guessing.
- **Solver (deterministic, the core):** per candidate showtime T: `deadline = T − buffer`; build journeys (pickups = waypoint home→passenger→cinema); `leave_by = deadline − travel_time`; journey OK if leave_by inside every participant's window (+ passenger available at pickup); T feasible only if all journeys OK; rank feasible by earliest / eating-slack / format; if none, return the **binding constraint** for the LLM to explain. Eating slack = `deadline − convergence − walk_to_hall`.
- **Efficiency:** travel time per origin→cinema barely varies across an evening → compute once per cinema, reuse as free arithmetic; only routing calls cost. Pass arrival time so 9pm showing uses 9pm traffic.
- **Cost:** ~1–3¢/plan, dominated by routing not LLM. Google $200/mo credit covers early usage.

---

## Testing (deterministic code; LLM mocked)

**Normalization:** raw TGV JSON / GSC XML → correct `Showing` (real fixtures); format tagged from raw labels; missing fields don't crash; timezone = UTC+8; provider_id upsert dedupes re-scrapes.

**Time-first view:** strict chronological order; identical times stack, near times don't; format section preserved; state filter via cinemas lookup; empty movie → empty list not error.

**Solver:** single journey feasible/infeasible/boundary; pickup waypoint + passenger-availability; multi-journey (all must clear); no-option returns binding constraint; ranking + deterministic tiebreak; eating slack incl. negative→"no time to eat" and `wants_to_eat:false`; traffic uses per-showtime arrival time; edge cases (same origin, impossibly short window, barely-feasible).

**Deep links:** constructed `booking_url` matches browser-verified template (highest-priority fix); nearest-first sort correct (when added).