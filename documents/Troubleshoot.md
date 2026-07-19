## 2026-07-19 data acquisition spike

- Git initially refused commands because the workspace was considered a dubious ownership repository. Use `git -c safe.directory=C:/Users/User/Documents/GitHub/It-is-Showtime ...` for repo commands in this environment.
- Read-only HTTPS fetches from PowerShell failed under the restricted network sandbox. Re-run required `Invoke-WebRequest` calls with approval.
- Both GSC and TGV landing pages are SPA shells; direct HTML scraping does not expose complete showtimes. Inspect the bundled JavaScript first, then replay the public API endpoints.
- GSC returns XML even though the saved spike filename uses `.json`. Parse it as XML.
- TGV endpoint request models are inconsistent by route. `moviesession_getcinemamovies` accepts top-level `cinemaid` plus nested `req.businessday`; `moviesession_get` requires top-level `cinemaid` and `businessdate` plus nested `req.businessdate`.
