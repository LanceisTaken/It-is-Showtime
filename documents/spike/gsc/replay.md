# GSC Replay

## Captured Responses

- Movie listing: `documents/spike/gsc/movie-listing.response.json`
- Showtime payload: `documents/spike/gsc/selangor-today.response.json`
- Normalized sample: `documents/spike/gsc/sample-normalized.json`

The `.json` extension is historical from the plan scaffold; the GSC response bodies are XML.

## Replay Steps

1. Fetch movie parents/children:

```text
GET https://epaymentapi.gsc.com.my/showtimews/service.asmx/getEpaymentMovie_ParentChild?includeChild=true&parent=
```

2. Select a parent movie id. This spike used `5116` for `The Odyssey`.

3. Fetch showtimes:

```text
GET https://epaymentapi.gsc.com.my/showtimews/service.asmx/getShowTimesByMovie_ParentChild_V2?parentid=5116&oprndate=2026-07-19
```

4. Parse XML, filter location address/state evidence for Selangor, then flatten `location -> child -> show` into rows.

## Result

The replay produced 40 normalized GSC rows across Selangor-coded locations for `The Odyssey` on 2026-07-19.

Sample extracted rows include:

- `GSC 1 Utama`, `The Odyssey`, `2026-07-19 8:30PM`, `2D - Digital 2D`
- `GSC 1 Utama`, `(Big) The Odyssey`, `2026-07-19 8:15PM`, `BIG - Big`
- `GSC 1 Utama`, `(4DX) The Odyssey`, `2026-07-19 10:00PM`, `4DX - 4DX`

## Validation

Cinema Online's public GSC page for 2026-07-19 lists matching GSC 1 Utama examples, including `The Odyssey` at `08:30PM` and premium/large-format entries at the same venue.

The booking URL in the normalized sample is constructed from IDs observed in the app bundle and XML response:

```text
https://epaymentwebapp.gsc.com.my/seat-selection?parentID=<parent>&oprndate=<date>&locationID=<location>&childCode=<child>&showID=<show>&hallGroup=<hallgroup>
```

This construction should be browser-verified before production use because the XML does not expose a final canonical booking URL field.
