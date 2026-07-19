# TGV Replay

## Captured Responses

- Cinema listing: `documents/spike/tgv/cinemas.response.json`
- Combined Selangor sessions: `documents/spike/tgv/selangor-today.response.json`
- Normalized sample: `documents/spike/tgv/sample-normalized.json`
- Per-cinema session captures: `documents/spike/tgv/*-sessions.response.json`

## Selangor Cinema IDs

The replay filtered cinemas whose `extdata.cinema.address` contains Selangor. Captured IDs:

```text
SWS, AMP, SW0, STW, JAY, ICT, CYB, CHR, BU0, BR0, BBT
```

## Replay Steps

1. Fetch cinemas with `POST /api/cinemas/v1/getcinemas`.
2. Filter Selangor cinemas from nested `extdata.cinema.address`.
3. For each selected `cinemaid`, call `POST /api/boxoffice/v1/moviesession_get`.
4. Flatten `businessday.cinemas[].movies[].experiences[].sessions[]` into rows.

## Result

The replay produced 119 normalized TGV rows across 11 Selangor cinemas for 2026-07-19.

Sample extracted rows include:

- `TGV 1 UTAMA`, session `320691`, `2026-07-19 20:30`, `IMAX 2D`
- `TGV 1 UTAMA`, session `321710`, `2026-07-19 20:00`, `INDULGE`
- `TGV SUNWAY SQUARE (NEW)`, session `6564`, `2026-07-19 20:15`, `INDULGE`

## Validation

Cinema Online's public TGV page for 2026-07-19 lists matching TGV 1 Utama examples for `The Odyssey`, including `08:30PM` IMAX and `08:00PM` Indulge entries. The TGV API payload confirms the corresponding session times and formats, but the bulk session response only supplies movie IDs, so the title mapping is inferred during validation rather than extracted directly from the session response.

The normalized booking URL is a best-effort details/session URL, not a confirmed checkout URL:

```text
https://www.tgv.com.my/movies/details/<movieid>?cinema=<cinemaid>&session=<sessionid>
```

Production use should resolve the official details route or booking route from TGV's movie metadata/details flow.
