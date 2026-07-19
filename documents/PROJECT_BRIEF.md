# Cinema Showtimes Aggregator — Project Brief

A web app that aggregates movie showtimes from Malaysian cinemas into one place. Starting with **TGV** and **GSC**; other chains (MBO, LFS, mmCineplexes, etc.) come later.

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

---

## Data acquisition (the hard part)

Neither TGV nor GSC offers a public API. Two options:

1. **Reverse-engineer their internal API (preferred).** Open the booking/showtime pages in a browser, watch the Network tab (XHR/fetch), and find the JSON endpoints their own frontend calls. Replicate those requests. Cleaner and more stable than HTML scraping.
   - GSC appears to use a separate backend (`epaymentwebapp.gsc.com.my`) worth inspecting.
   - TGV has a mobile app; its API can be observed via the web booking flow or a proxy.
2. **HTML scraping (fallback).** Parse the showtime pages directly. More brittle — breaks on any redesign.

Each chain's raw data gets normalized into one shared schema.

**Maintenance note:** an aggregator is "a thing you keep alive," not "build once." Endpoints and page structures change without warning. With two chains this is manageable; plan for periodic breakage.

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
}
```

Supporting reference data: a `cinemas` list (name, chain, state, coordinates) and a `movies` list (title, poster, runtime, rating).

---

## Build order (phasing)

Build in this order. Each phase is useful on its own; do not start the next until the current one is solid.

1. **Data pipeline** — reliably pull a clean `{movie, cinema, date, time, format}` table from TGV and GSC into a database on a schedule. This is the whole ballgame; everything else is a query over it.
2. **Time-first menus (MVP)** — the state → movie → sorted-showings flow. This is the real differentiator and ships with zero AI.
3. **AI planner (Phase 2, below)** — a natural-language layer on top of the menus. The menus remain the fallback / power-user path.

Keep the location and surrounding-context fields (coordinates, nearby food/transport) in the data model from the start even if nothing uses them yet — that's where any future monetization (affiliate links, promoted placement) will live, and it's cheap to leave the hooks in versus retrofitting.

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
[LLM]  parse to structured intent      ← AI
      │
      ▼
Geocode addresses + fetch candidate showings   ← your code + DB
      │
      ▼
Routing API: travel time per journey (traffic-aware)   ← external API
      │
      ▼
[Solver] feasibility + ranking          ← your code, NO AI. Correctness lives here.
      │
      ▼
[LLM]  explain the result in plain language    ← AI
      │
      ▼
Answer the group can act on
```

### Required components

| Component | What it does | Build or buy |
|---|---|---|
| LLM API | Parse request → intent JSON; turn solver output → prose | Buy (per-token) |
| Intent schema + validator | Fixed JSON shape the model must emit; reject/repair malformed output | Build |
| Geocoding | "Lives in Subang" → coordinates | Buy (or avoid via saved addresses) |
| Routing / directions API | Travel time between points, traffic-aware, by mode (drive/transit/walk), **waypoints** for pickups | Buy (per-request) |
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

Design implication: this per-use cost is why any future paywall, if one ever exists, would sit on the *planner*, not on the free menu browsing.

---

## Testing (the non-AI code)

All of this is deterministic and must be unit-tested independently of the LLM. The LLM is mocked in these tests — you feed the solver a fixed intent object and assert on its output.

### 1. Data normalization

- Raw TGV payload → correct `Showing` objects (all fields mapped).
- Raw GSC payload → correct `Showing` objects.
- Format detection: IMAX / D-Box / Standard correctly tagged from each chain's raw labels.
- Missing/optional fields handled without crashing.
- Date and time parsed into the right timezone (Malaysia, UTC+8).
- Two chains' data merge into one collection without duplication or field collisions.

### 2. Time-first view logic

- Showings returned in strict chronological order.
- Showings at genuinely the same time are stacked; near-but-different times are **not** merged (8:00, 8:15, 8:20 stay as three rows).
- Format badge preserved on every row (a 2:30 IMAX and 2:30 Standard remain distinct).
- Filtering by state returns only that state's showings.
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
- `booking_url` generated correctly per chain and per showing (deep link resolves to the right movie/time).
