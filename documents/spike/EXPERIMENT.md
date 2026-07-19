# Data Acquisition Feasibility - Findings

Target state: Selangor. Run date: 2026-07-19.

## Verdict summary
| Chain | Grade (GREEN/YELLOW/RED) | Primary method (API/scrape) | Auth needed? | Missing fields | Notes |
|-------|--------------------------|-----------------------------|--------------|----------------|-------|
| GSC   | YELLOW | Public XML API replay | No | Direct state filter; canonical booking URL | Reliable structured payloads, but state is inferred from address/state code and booking URL is constructed. |
| TGV   | YELLOW | Public JSON API replay | No | Movie title in bulk sessions; canonical booking URL | Session replay is clean JSON, but movie metadata requires a companion lookup/details flow. |

## GSC
### Discovery
GSC's `showtime-by-movies` page is an Angular SPA shell. Bundle inspection found the API base `https://epaymentapi.gsc.com.my/` and showtime service calls under `showtimews/service.asmx`.

Captured evidence is in `documents/spike/gsc/`.

### Replay
Two unauthenticated GET calls reproduced the data:

- `getEpaymentMovie_ParentChild?includeChild=true&parent=`
- `getShowTimesByMovie_ParentChild_V2?parentid=5116&oprndate=2026-07-19`

The replay used parent movie id `5116` for `The Odyssey`. The endpoint returned XML text.

### Extraction & validation
The XML flattened into 40 Selangor-coded rows in `documents/spike/gsc/sample-normalized.json`. Fields extracted cleanly: movie title, cinema name, date, time, format, and IDs needed to construct a seat-selection URL.

Cinema Online's public GSC listings for 2026-07-19 independently matched sampled GSC 1 Utama showtimes such as `The Odyssey` at `08:30PM`. Remaining production risk is the booking URL: the app bundle exposes the parameter names, but the final URL should still be browser-verified before relying on it.

## TGV
### Discovery
TGV's `showtimes` page is also an Angular/Ionic SPA shell. Bundle inspection found `baseApiUrl: "https://api.tgv.com.my"` and JSON APIs under `/api/cinemas/v1` and `/api/boxoffice/v1`.

Captured evidence is in `documents/spike/tgv/`.

### Replay
The cinema listing replay worked with:

- `POST /api/cinemas/v1/getcinemas`
- body `{"startrowindex":0,"maxrows":100}`

Selangor cinemas were selected from nested `extdata.cinema.address`. The full session replay worked per cinema with:

- `POST /api/boxoffice/v1/moviesession_get`
- top-level `cinemaid` and `businessdate`
- nested `req.businessdate`, `req.movieid`, `req.experience`, and `req.cinemaid`

### Extraction & validation
The combined replay produced 119 normalized rows in `documents/spike/tgv/sample-normalized.json` across 11 Selangor cinemas. It extracts cinema, cinema id, date, time, hall, format, session id, and movie id.

Cinema Online's public TGV listings for 2026-07-19 independently matched sampled TGV 1 Utama sessions, including `08:30PM` IMAX and `08:00PM` Indulge. The bulk TGV session response does not include movie title, so title enrichment must come from a separate movie metadata/details lookup.

## Reliability assessment & recommendation
Both providers are feasible without browser automation for the data acquisition layer. Neither required authentication for the read-only endpoints tested.

Recommended next step: build a provider abstraction around replayed API calls, with explicit provider-specific normalizers and a validation job that compares a small sample against Cinema Online or the provider-rendered page. Treat both as `YELLOW` until the remaining enrichment/URL gaps are resolved:

- GSC: confirm state filtering rules and seat-selection URL behavior in a browser.
- TGV: add movie metadata lookup and confirm the canonical booking/details route.
