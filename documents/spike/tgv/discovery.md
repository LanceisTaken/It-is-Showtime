# TGV Discovery

Target: Selangor showtimes for 2026-07-19.

## Entry Point

- Page: `https://www.tgv.com.my/showtimes`
- Captured shell: `documents/spike/tgv/raw.html`
- SPA bundle evidence:
  - `documents/spike/tgv/main.eede635e5be43ed7.js`
  - `documents/spike/tgv/runtime.ef7a03c511dbfda5.js`

The page is an Angular/Ionic SPA. The bundle exposes `baseApiUrl: "https://api.tgv.com.my"` and calls JSON endpoints under `/api/cinemas/v1` and `/api/boxoffice/v1`.

## Endpoints Found

Cinema listing:

```text
POST https://api.tgv.com.my/api/cinemas/v1/getcinemas
Body: {"startrowindex":0,"maxrows":100}
```

Cinema movies by date:

```text
POST https://api.tgv.com.my/api/boxoffice/v1/moviesession_getcinemamovies
Body: {
  "cinemaid": "BU0",
  "req": {
    "businessday": "2026-07-19",
    "cinemaid": "BU0",
    "experience": "",
    "experienceGroup": ""
  }
}
```

Full cinema sessions by date:

```text
POST https://api.tgv.com.my/api/boxoffice/v1/moviesession_get
Body: {
  "cinemaid": "BU0",
  "businessdate": "2026-07-19",
  "req": {
    "businessdate": "2026-07-19",
    "movieid": "",
    "experience": "",
    "cinemaid": "BU0"
  }
}
```

## Auth and Headers

No login, cookie, or non-empty session token was needed for cinema and session discovery. Minimal headers worked:

```text
User-Agent: Mozilla/5.0
Accept: application/json
Content-Type: application/json
```

Some bundle service methods add `x-mvcsession` from user state for account-specific flows. The showtime endpoints used here did not require it.

## Useful Fields

- Cinema id/name/address: `results.businessday.cinemas[].cinemaid`, `name`, `extdata.cinema.address`
- Movie id: `movies[].movieid`
- Session id: `sessions[].sessionid`
- Date/time: `sessions[].businessdate`, `sessions[].showtimemy`
- Hall/format: `sessions[].screenname`, `sessions[].experience`, `sessions[].extdata.conceptattributenames`, `sessions[].extdata.sessionattributenames`

The full session endpoint does not include movie title in the same bulk response. A production pipeline needs a companion movie metadata lookup or a controlled title map.
