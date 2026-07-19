# GSC Discovery

Target: Selangor showtimes for 2026-07-19.

## Entry point

- Page: `https://epaymentwebapp.gsc.com.my/showtime-by-movies`
- Captured shell: `documents/spike/gsc/raw.html`
- SPA bundle evidence:
  - `documents/spike/gsc/main.618d213251e99770.js`
  - `documents/spike/gsc/content-layout.211dd97828904139.js`
  - `documents/spike/gsc/runtime.3579b466df8765ec.js`
  - `documents/spike/gsc/vendor.f7b310a8df293c29.js`

The static HTML is an Angular shell. The usable showtime source is exposed in the bundled service code, which points at `https://epaymentapi.gsc.com.my/showtimews/service.asmx`.

## Endpoints Found

Movie listing:

```text
GET https://epaymentapi.gsc.com.my/showtimews/service.asmx/getEpaymentMovie_ParentChild
  includeChild=true
  parent=
```

Showtimes by movie/date:

```text
GET https://epaymentapi.gsc.com.my/showtimews/service.asmx/getShowTimesByMovie_ParentChild_V2
  parentid=<parent movie id>
  oprndate=YYYY-MM-DD
```

The bundle uses `responseType: "text"` for these calls. Responses are XML payloads, not JSON.

## Auth and Headers

No login, cookie, or dynamic token was needed for the discovery calls. Minimal replay headers worked:

```text
User-Agent: Mozilla/5.0
Accept: application/xml,text/xml,*/*
```

## Useful Fields

- Movie title: movie child `title`
- Cinema: location `name`
- Date: show `date`
- Time: show `timestr`
- Format: show `type` and `type_desc`
- IDs for booking URL construction: `parentid`, `location id`, `child code`, `show id`, `hallgroup`

State filtering is not available as an endpoint parameter. The replay filtered Selangor locations by address/state code evidence such as `,SGR,` in the XML location address.
