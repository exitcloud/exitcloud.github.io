---
title: "Plausible vs Umami: Self-Hosted Analytics That Respect Privacy"
date: 2026-02-10T09:00:00Z
draft: false
tags:
  - "tutorial"
  - "self-hosted"
  - "analytics"
---

I ripped Google Analytics off all my sites in 2024. Not because of some grand privacy stance — though that's a nice bonus — but because GA4 is genuinely terrible to use. The UI is a maze. Simple questions like "how many people visited my site today" require clicking through five menus. And the tracking script is 45KB and slows down page loads.

So I went looking for something simpler. Ended up trying both Plausible and Umami, running them self-hosted on my own box. I've been using both for over a year now. Here's the real comparison.

## The TL;DR

Both are great. Seriously. You can't go wrong with either one. But they have different strengths:

- **Plausible**: More polished, better out-of-the-box experience, slightly heavier. The Community Edition is free to self-host.
- **Umami**: Lighter, simpler, single binary option, faster to deploy. Also free and open source.

If you want the quick answer: use Plausible if you care about a polished dashboard and don't mind running a few more containers. Use Umami if you want the lightest possible setup and are comfortable with a slightly more minimal UI.

Now let me actually back that up.

## Plausible: the polished one

Plausible started as a paid SaaS and later released a Community Edition (CE) you can self-host for free. The CE has most features of the paid version. It's written in Elixir with a ClickHouse backend for analytics storage.

### Deploying Plausible

Here's a trimmed docker-compose that gets you running:

```yaml
services:
  plausible:
    image: ghcr.io/plausible/community-edition:v2-latest
    ports:
      - "8000:8000"
    env_file:
      - plausible-conf.env
    depends_on:
      plausible_db:
        condition: service_healthy
      plausible_events_db:
        condition: service_healthy

  plausible_db:
    image: postgres:16-alpine
    volumes:
      - db-data:/var/lib/postgresql/data
    environment:
      - POSTGRES_PASSWORD=postgres
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s

  plausible_events_db:
    image: clickhouse/clickhouse-server:24-alpine
    volumes:
      - event-data:/var/lib/clickhouse
    healthcheck:
      test: ["CMD", "wget", "--spider", "-q", "localhost:8123/ping"]
      interval: 5s

volumes:
  db-data:
  event-data:
```

And the env file (`plausible-conf.env`):

```bash
BASE_URL=https://analytics.yourdomain.com
SECRET_KEY_BASE=generate-with-openssl-rand-base64-48
DATABASE_URL=postgres://postgres:postgres@plausible_db:5432/plausible_db
CLICKHOUSE_DATABASE_URL=http://plausible_events_db:8123/plausible_events_db
```

Generate that secret key:

```bash
openssl rand -base64 48
```

Three containers: the Plausible app, Postgres for config/users, and ClickHouse for event data. It's more moving parts than Umami, but ClickHouse is what makes Plausible fast at scale — it can handle millions of events without breaking a sweat.

### What I like about Plausible

**The dashboard is beautiful.** One page, everything you need. Visitors, page views, bounce rate, visit duration, top pages, referrers, countries, devices. No clicking through menus. Load the dashboard, see your data.

**Goal tracking works well.** You can set up custom events (button clicks, form submissions, outbound links) and track conversion rates. The API for sending custom events is clean.

**The tracking script is tiny.** Under 1KB. Compare that to GA4's bloated bundle. Your page load times will thank you.

**Filtering is intuitive.** Click any dimension (a country, a referrer, a page) and the whole dashboard filters to that segment. Click again to remove the filter. Simple.

### What's annoying

**Three containers.** Postgres and ClickHouse both need care and feeding. ClickHouse in particular uses more memory than you'd expect for a small site.

**Resource usage.** On my box, Plausible idles at about 400-500MB of RAM total across all three containers. Not terrible, but noticeable if you're running other stuff on a small VPS.

**CE vs paid feature gaps.** Some features like funnels, custom properties, and revenue tracking are only in the paid cloud version. The CE covers the basics well, but if you want advanced stuff, you're either paying or building it yourself.

## Umami: the lightweight one

Umami takes a different approach. It's a Next.js app backed by either Postgres or MySQL. One app container, one database. That's it.

### Deploying Umami

```yaml
services:
  umami:
    image: ghcr.io/umami-software/umami:postgresql-latest
    ports:
      - "3000:3000"
    environment:
      DATABASE_URL: postgresql://umami:umami@umami_db:5432/umami
      DATABASE_TYPE: postgresql
      APP_SECRET: generate-a-random-string-here
    depends_on:
      umami_db:
        condition: service_healthy

  umami_db:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: umami
      POSTGRES_USER: umami
      POSTGRES_PASSWORD: umami
    volumes:
      - umami-db-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U umami"]
      interval: 5s

volumes:
  umami-db-data:
```

Two containers. Spin it up, visit port 3000, log in with `admin` / `umami`, add your site, drop the script tag on your pages. Done in five minutes.

### What I like about Umami

**Dead simple to deploy.** Two containers, one env var for the database, one for the secret. No ClickHouse, no extra moving parts. If you already run Postgres for your app, you could even share the database (use a separate schema).

**Low resource usage.** Umami idles at about 150-200MB total. On a small VPS where every megabyte counts, this matters.

**Clean API.** Umami has a solid REST API for pulling analytics data programmatically. I use this to show visitor counts on my site's footer — a small vanity thing, but the API made it easy.

**Active development.** The team ships updates regularly. The v2 rewrite was a big improvement in both UI and performance.

### What's annoying

**The dashboard is more minimal.** It shows the essentials — pageviews, visitors, referrers, pages, countries, browsers — but it doesn't feel as polished as Plausible. The charts are simpler. The filtering is there but less intuitive.

**No built-in goal tracking out of the box.** You can track custom events, but it takes a bit more setup compared to Plausible's guided flow.

**Postgres at scale.** For small-to-medium sites (under a few million pageviews/month), Postgres is fine. At higher volumes, you might want ClickHouse's columnar storage — which is exactly what Plausible uses.

## Head-to-head comparison

| Feature | Plausible CE | Umami |
|---------|-------------|-------|
| Containers needed | 3 (app + Postgres + ClickHouse) | 2 (app + Postgres) |
| RAM usage (idle) | ~400-500MB | ~150-200MB |
| Tracking script size | <1KB | <2KB |
| Dashboard polish | Very polished, single-page | Clean but more minimal |
| Custom events | Yes, built-in | Yes, via API |
| Goal/conversion tracking | Yes | Manual setup |
| API | Yes | Yes, well-documented |
| Filtering | Click-to-filter, very intuitive | Present but less slick |
| Scale ceiling | High (ClickHouse backend) | Medium (Postgres) |
| License | AGPL-3.0 | MIT |
| Cookies | None | None |
| GDPR consent needed | No | No |

## The privacy angle

Both tools are privacy-first by design. Neither uses cookies. Neither collects personal data. Neither requires a GDPR consent banner. This alone is worth the switch from Google Analytics.

How they count unique visitors without cookies: they hash the visitor's IP + User-Agent + a daily rotating salt. The hash is never stored — it's used in-memory to deduplicate, then discarded. Next day, new salt, no way to track anyone across days. It's a genuinely good approach.

For your visitors, nothing changes. No cookie banners. No "accept all" buttons. No third-party scripts phoning home to Google. Your analytics stay on your server, under your control.

## Which one should you pick?

**Pick Plausible if:**
- You want the nicest looking dashboard with zero customization
- You plan to track goals and conversions
- You're running on a box with 1GB+ RAM to spare
- You might scale to millions of pageviews

**Pick Umami if:**
- You want the lightest possible setup
- You're already running Postgres and want to minimize new containers
- You care about a permissive MIT license
- You're on a tight-resource VPS (512MB-1GB)

For my personal sites, I run Umami. It's lighter and I don't need conversion tracking. For client projects where I want to show them a pretty dashboard, I use Plausible. Both have been rock solid for over a year.

Either way — you'll never want to go back to Google Analytics. Trust me on that.

## Cheat Sheet

| Tool / Concept | Description | Link |
|---------------|-------------|------|
| [Plausible CE](https://github.com/plausible/community-edition) | Polished self-hosted analytics, Elixir + ClickHouse, AGPL-3.0 | [plausible.io](https://plausible.io/) |
| [Umami](https://github.com/umami-software/umami) | Lightweight self-hosted analytics, Next.js + Postgres, MIT | [umami.is](https://umami.is/) |
| ClickHouse | Columnar database Plausible uses for fast analytics queries at scale | [clickhouse.com](https://clickhouse.com/) |
| `openssl rand -base64 48` | Generate a secret key for Plausible's `SECRET_KEY_BASE` | — |
| No cookies, no consent banner | Both tools hash IP+UA with a daily rotating salt, never store PII | — |
| Plausible tracking script | <1KB, async, won't slow your page | Add via `<script>` tag |
| Umami tracking script | <2KB, similarly lightweight | Add via `<script>` tag |
| Umami REST API | Pull analytics data programmatically (visitor counts, etc.) | [umami.is/docs/api](https://umami.is/docs/api) |
| **Plausible idle RAM** | ~400-500MB across 3 containers | — |
| **Umami idle RAM** | ~150-200MB across 2 containers | — |
| **Rule of thumb** | Umami for lightweight/personal, Plausible for polished/client-facing. Both beat GA4. | — |
