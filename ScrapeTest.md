# Cinema Showtimes Aggregator — Project Brief

A web app that aggregates movie showtimes from Malaysian cinemas into one place. Starting with **TGV** and **GSC**; other chains (MBO, LFS, mmCineplexes, etc.) come later.

## The core idea (and the differentiator)

Existing aggregators (Cinema Online, Popcorn) and the chains' own apps are **location-first**: you pick a cinema, then see its showtimes. That assumes you've already decided where to go.

This app is **time-first**. Many people are flexible on location but strict on time. The flow is:

1. Pick a **state** (e.g. Selangor)
2. Pick a **movie** now showing in that state
3. See **all of today's showings for that movie, sorted by time**, each tagged with its cinema and format

The whole product is one dataset viewed by time instead of by location.

## Key UX rules

- **Do not group strictly by exact timestamp.** Cinemas rarely share the same minute (8:00 vs 8:15 vs 8:20). Render a single chronological list of every showing sorted by time; only stack entries together when they genuinely share a time.
- **Show format on every showing** (Standard / IMAX / D-Box / etc.). A 2:30 IMAX and a 2:30 Standard are different products with different prices.
- **Location is a tiebreaker, not a filter** (nice-to-have, not MVP). When multiple cinemas share a time, sort nearest-first. Store each cinema's coordinates so this can be added later without re-architecting.
- Each showing should deep-link out to the chain's own booking page (this app surfaces times; it doesn't handle payment).

## The hard part: data acquisition

Neither TGV nor GSC offers a public API. Two options:

1. **Reverse-engineer their internal API (preferred).** Open the booking/showtime pages in a browser, watch the Network tab (XHR/fetch), and find the JSON endpoints their own frontend calls. Replicate those requests. Cleaner and more stable than HTML scraping.
   - GSC appears to use a separate backend (`epaymentwebapp.gsc.com.my`) worth inspecting.
   - TGV has a mobile app; its API can be observed via the web booking flow or a proxy.
2. **HTML scraping (fallback).** Parse the showtime pages directly. More brittle — breaks on any redesign.

Each chain's raw data gets normalized into one shared schema (see below).

## Suggested data model

A single normalized table/collection of showings:

```
Showing {
  movie_title      string
  movie_id         string        // internal, for grouping across chains
  cinema_name      string
  chain            enum(TGV, GSC, ...)
  state            string
  lat, lng         float         // for nearest-first sorting later
  date             date
  time             time
  format           string        // Standard, IMAX, D-Box, ...
  booking_url      string        // deep link to the chain's booking page
}
```

Supporting reference data: a `cinemas` list (name, chain, state, coordinates) and a `movies` list (title, poster, runtime, rating).

## Rough architecture

```
[TGV fetcher]  [GSC fetcher]      → per-chain modules, each outputs normalized Showings
        \           /
         → normalizer → database (Postgres / SQLite to start)
                          ↓
                     API layer (serves by-state, by-movie, by-time queries)
                          ↓
                     frontend (state → movie → time-sorted list)
```

Run the fetchers on a schedule (e.g. a few times a day, since showtimes are published ahead). Cache results; don't hit the chains on every user request.

## Suggested stack (flexible)

- **Fetchers:** Node/TypeScript or Python. Whatever makes inspecting and replaying HTTP requests easiest.
- **DB:** SQLite for the MVP, Postgres if it grows.
- **Backend:** any lightweight API (Express / FastAPI / Next.js API routes).
- **Frontend:** React/Next.js. Mobile-first — most users check showtimes on a phone.

## Build order

1. **Prove the pipeline for ONE chain, ONE movie.** Manually inspect GSC or TGV, extract a clean list of `{movie, cinema, date, time, format}` for one state. This is the whole ballgame — if it works, everything else is repetition.
2. Add the second chain; normalize both into the shared schema.
3. Persist to DB on a schedule.
4. Build the time-first frontend flow over the DB.
5. Add format badges, then location/nearest-first, then more chains.

## Things to keep in mind

- **Maintenance is the real cost.** Fetchers break when sites change. Design each chain's fetcher as an isolated, easily-repairable module and log failures loudly.
- **Legal is a gray area (not legal advice).** Showtimes are factual data, and other aggregators operate openly, but scraping can conflict with a site's terms of service. Keep it low-key, don't hammer their servers, and be cautious about heavy monetization.
- **Attribute and link out.** Always send users to the chain's own booking page rather than trying to intercept the transaction.
