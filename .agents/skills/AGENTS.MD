# Cinema Showtimes Aggregator

Time-first web app aggregating Malaysian cinema showtimes (TGV + GSC first) into one dataset viewed by time, not location. Flow: state → movie → chronological list of every showing, each tagged with cinema + format, deep-linking to the chain's booking page.

## Stack

- Backend / DB: Supabase (Postgres) — `showings`, `cinemas`, `movies` tables; pgvector, pg_cron, storage bundled
- Frontend / pipeline: TypeScript — frontend (React) + scraper as Vercel functions
- Deploy: Vercel — frontend hosting + Cron for scheduled scrapes
- DNS / edge: Cloudflare — DNS, caching, proxy, DDoS
- Domain: Namecheap
- Analytics: PostHog
- Error tracking: Sentry — pipeline + app errors (a silently-dead scraper is the classic aggregator failure mode)
- Poster storage: Supabase Storage

## Documentation

- Update Sessionlog.md as part of every commit — stage it in the same commit. List what the commit changed as concise bullet points. Keep it terse — no verbose prose.
- Update Troubleshoot.md whenever a bug is fixed. Keep each entry concise: the problem, then the solution. No dates or timestamps.

## Git Workflow

- Divide the project into phases. Create a phase branch off main for each phase.
- For each task in a phase, create a task branch off the phase branch.
- Commit all work for a task to its task branch.
- When a task is complete, open a PR to merge the task branch into the phase branch. One PR per task.
- After the task PR merges, delete the task branch locally.
- When all tasks in a phase are complete, open a PR to merge the phase branch into main.
- After the phase PR merges, delete the phase branch locally.
- Use the gh CLI to open PRs. If gh is unavailable, stop and tell me rather than merging directly.