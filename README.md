# CinemaScout

**Bandsintown for cinema.** Follow the actors, directors and films you care
about, or import your Letterboxd watchlist, and get told when they're
**playing at an indie or repertory theater near you**.

**Live at [cinemascout.app](https://cinemascout.app)**

<!-- screenshot: dashboard "Playing near you" -->
<!-- screenshot: digest email -->

> The source is private. This repo describes the project. I'm happy to walk
> through the code or grant read access on request.

## Why

New releases are easy to find. A 35mm print of a 1970s film playing one night
at a 200-seat arthouse is not. Repertory programming is spread across dozens of
small theater sites, each with its own format. CinemaScout collects that
programming, maps every listing to a canonical film, and matches it against
your watchlist. You get **one email** listing your films that are playing soon.
The same matches are available on the dashboard and as a subscribable
calendar feed.

Coverage: independent and repertory theaters in **Los Angeles, the SF Bay
Area, New York and Seattle**.

## Features

- **One watchlist.** Follow people (actors, directors) or individual films.
  Import a Letterboxd watchlist from its data export; the file is parsed in the
  browser.
- **"Playing near you" digest.** All of a user's matches from a run are batched
  into one email that mirrors the dashboard: one card per film, then venues,
  then showtimes, with ticket links.
- **Calendar feeds.** Signed per-user ICS feeds put matched screenings straight
  into Google or Apple Calendar.
- **Passwordless auth.** Magic-link sign-in issues a JWT session.

## Architecture

```
 browser ──────► ┌───────────────────────────┐  reads/writes  ┌──────────┐
 calendar apps ─►│  Next.js on Vercel        │ ◄────────────► │ Postgres │
                 │  App Router · GraphQL API │                │  (Neon)  │
                 └────────────┬──────────────┘                └────▲─────┘
                    enqueue / │ serve feeds                        │
                    poll jobs ▼                                    │
                 ┌───────────────────────────┐                     │
                 │  Cloudflare D1            │                     │
                 │  job queue · ICS feeds    │                     │
                 └────────────▲──────────────┘                     │
                 claim jobs / │ write feeds                        │
                              │                                    │
 theater sites ─►┌────────────┴──────────────┐  ingest/match       │
 TMDB ──────────►│  Worker (Raspberry Pi)    │ ────────────────────┘
                 │  async only: cron + jobs  │
                 └────────────┬──────────────┘
                              ▼
                        Resend (email)
```

The web app and API on Vercel handle all user-facing traffic against Neon. The
Pi runs **only asynchronous work**: the twice-daily ingest, match and digest
pipeline, plus queued jobs such as Letterboxd imports.

TypeScript monorepo (pnpm + Turborepo):

| Package | Role |
| --- | --- |
| `apps/web` | Next.js App Router, Tailwind, shadcn/ui. Also hosts the GraphQL endpoint. |
| `apps/api` | GraphQL schema as a library (Yoga + Pothos, code-first). |
| `apps/worker` | Cron pipeline (ingest, match, digest, calendar render) and job-queue poller. |
| `packages/ingest` | Pluggable `ShowtimeAdapter` interface with one adapter per theater platform. |
| `packages/catalog` | TMDB client, film/credit sync, title resolution. |
| `packages/db` | Drizzle schema, drizzle-kit migrations, HTTP fetch cache. |
| `packages/letterboxd` | Watchlist import. |

**Infra:** Vercel (web + API), Neon Postgres, Cloudflare D1 (job queue and
calendar feeds), Resend (email), a self-hosted Raspberry Pi worker run with
mise and systemd, and Sentry across all three.

## Engineering highlights

### Title resolution at ingest

Theater listings aren't clean titles. They look like "Aug 31 I SHOT ANDY
WARHOL", "Dead Calm on 35mm" or "Basement Fest presents: Phenomena on VHS". Each
listing is resolved to a TMDB film **when it's ingested**, not at query time:

1. A **cleanup ladder** produces progressively cleaner search queries (strip
   dates, formats, presenter prefixes and so on). It moves to the next rung only
   when a search returns zero results. Zero results is a reliable "bad query"
   signal, so a title that already resolves costs exactly one search.
2. Results are cached in a resolution table. The same table is the
   **unmatched-title queue**: anything that can't be mapped is excluded from
   matching until it's handled. A fix re-links every affected screening in one
   set-based update.
3. A **nightly LLM pass** (Claude) works through that queue. It derives a
   better search query from film knowledge. It then chooses a match **only from
   candidates our own TMDB search returned**, so it widens where we search but
   never what can be written. It can also mark a title as ignorable (marathons,
   multi-film programs) or leave it for a human. Each title gets one pass. A
   malformed response falls back to "human".

### One job interface, two runtimes

`dispatch(job)` writes a job row (pending, running, succeeded or failed) and
clients poll it. In development, jobs run in-process. In production the
serverless API only enqueues, because fire-and-forget work doesn't survive a
serverless function. The Pi worker then claims and runs the job. Slow work such
as a large Letterboxd import uses this path.

### Keeping Postgres asleep

Neon scales to zero after 5 minutes idle, but any query resets that timer. A
worker polling for jobs every few seconds, or calendar apps refreshing feeds
every few minutes, would keep the database awake around the clock just to
answer "nothing new". So both of those polling workloads live in Cloudflare D1:

- **Job queue.** D1 has no `FOR UPDATE SKIP LOCKED`, but it serializes writes
  per database. One conditional `UPDATE … RETURNING` is therefore an atomic
  claim. Finished rows are swept, so polls scan only in-flight work.
- **Calendar feeds.** The pipeline renders each user's ICS feed into D1 after
  every run. Serving a feed is a single D1 read with an ETag and never touches
  Postgres.

Postgres handles only real traffic, and the polling fits within D1's free tier.

### Polite, failure-aware fetching

Every outbound fetch goes through a cache with per-source TTLs. Re-fetches are
gated on the **last attempt**, not the last success, with a 24-hour floor. A
source returning errors therefore can't be hammered by re-runs. Failures where
the server answered (4xx/5xx) back off. Network errors (timeouts, DNS) stay
retryable. Any failure serves the last good response when one exists.

### Deliberately simple location model

Users choose metro areas and theaters belong to a metro. I tried PostGIS
early on and dropped it. At a few dozen theaters per metro it added
hand-written SQL for no benefit. Plain Postgres keeps migrations generated by
drizzle-kit and hosting cheap.

### Correctness details

- Showtimes are stored as UTC and converted from each theater's local wall time.
  DST is handled explicitly.
- Notification ledgers make the digest idempotent, so a re-run never
  double-emails.
- ICS output follows RFC 5545 line folding (never splitting inside a URL) and
  uses signed feed tokens.

## Stack

TypeScript · Next.js · React · Tailwind · shadcn/ui · GraphQL (Yoga, Pothos) ·
PostgreSQL · Drizzle · Turborepo · pnpm · node-cron · TMDB API · Claude API ·
Resend · Vercel · Neon · Cloudflare D1 · Sentry · Raspberry Pi (systemd)
