---
title: "From Vibe to Live: Self‑Hosting AI‑Powered Apps on a $5–20 VPS"
date: 2026-03-15T14:06:15.891Z
draft: false
tags:
  - "tutorial"
  - "devops"
  - "indie-dev"
---
# From Vibe to Live: Self‑Hosting AI‑Powered Apps on a Tiny VPS

Here’s the thing — prototypes don’t ship themselves. Vibes are cheap. Working systems with logs, metrics, and a repeatable deploy take real decisions and some boring glue.

If you’ve been anywhere near the indie scene, you’ve seen the “vibecoded” demo that looks great on video but collapses on contact. There’s even a story titled “100 hour gap between a vibecoded prototype and a working product” that hit Hacker News with a discussion at 30 points — a polite reminder that “works once” isn’t the same as “ships daily” ([HN thread](https://news.ycombinator.com/item?id=47386636), [post](https://kanfa.macbudkowski.com/vibecoding-cryptosaurus)). I learned this the hard way.

Let me show you what actually works.

We’ll build a small, production‑ish stack on a single VPS: a Go ingestor inspired by [Signet](https://signet.watch), a tiny AI service that behaves better than that criticized AI DJ piece ([article](https://www.charlespetzold.com/blog/2026/02/The-Appalling-Stupidity-of-Spotifys-AI-DJ.html), [HN](https://news.ycombinator.com/item?id=47385272)), NAT‑friendly connectivity guided by a well‑liked TCP hole‑punching write‑up ([article](https://robertsdotpm.github.io/cryptography/tcp_hole_punching.html), [HN](https://news.ycombinator.com/item?id=47384032)), and observability inspired by a kernel anti‑cheat explainer that lit up HN ([article](https://s4dbrd.github.io/posts/how-kernel-anti-cheats-work/), [HN](https://news.ycombinator.com/item?id=47382791)). No managed services. No yak shaving. Ship it.

## 1. Plan the Thing You’ll Actually Ship (Not Just Vibecode)

Scope constraints are your friend:
- One deterministic ingestion loop.
- One AI‑assisted decision loop.
- One way to view results (simple web UI or a JSON API).
- One command to boot everything.

Definition of done:
- System starts with `docker compose up`.
- Health endpoint responds. Metrics at `/metrics` or similar.
- Logs readable with one command.
- Backup script exists and runs.

Pick a tiny baseline stack:
- [Docker](https://www.docker.com/) + [Docker Compose](https://docs.docker.com/compose/).
- [PostgreSQL](https://www.postgresql.org/) with [PostGIS](https://postgis.net/).
- A reverse proxy like [Caddy](https://caddyserver.com/) or [NGINX](https://nginx.org/).

Fresh Ubuntu box setup (minimal, and you can adapt to your distro):
```bash
# add a deploy user
sudo adduser deploy
sudo usermod -aG sudo,adm deploy

# add your SSH key and harden SSH (key auth only)
sudo -u deploy mkdir -p /home/deploy/.ssh
sudo -u deploy chmod 700 /home/deploy/.ssh
sudo -u deploy nano /home/deploy/.ssh/authorized_keys
sudo chmod 600 /home/deploy/.ssh/authorized_keys
sudo sed -i 's/^#\?PasswordAuthentication .*/PasswordAuthentication no/' /etc/ssh/sshd_config
sudo systemctl reload ssh

# minimal firewall with ufw
sudo apt-get update
sudo apt-get install -y ufw
sudo ufw allow OpenSSH
sudo ufw allow 80
sudo ufw allow 443
sudo ufw enable

# install Docker (convenience script)
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker deploy
```

Sketch your architecture:
- `ingestor` (Go) fetches data on intervals, normalizes, stores in Postgres.
- `worker-ai` (Python or Node) wraps an LLM with a tiny REST API.
- `web` (your choice) and a reverse proxy in front.

Not big. Not fancy. But it boots in one go.

## 2. Build a Signet‑Style Data Ingestion Loop in Go on a Single VPS

[Signet](https://signet.watch) is a great mental model: built in Go to automate a loop that humans often run by hand — “checking satellite feeds, pulling up weather, looking at terrain and fuels, deciding whether a detection is actually a fire worth tracking.” The data it cites includes NASA FIRMS thermal detections, GOES‑19 imagery, NWS forecasts, LANDFIRE fuel models, USGS elevation, Census population data, and OpenStreetMap. The tough part? Data arriving “from different sources on different cadences in different formats.” And this line hits home: “Most of the system is deterministic plumbing.”

We’ll copy the spirit, not the scale. Two sources is enough to demonstrate:
- NASA FIRMS‑like thermal detections.
- NWS‑style forecasts.

We’ll store them with spatial fields so we can do simple geospatial joins later via [PostGIS](https://postgis.net/).

Docker Compose to run Postgres with PostGIS, optional [pgAdmin](https://www.pgadmin.org/), and a Go ingestor:

```yaml
# docker-compose.yml
services:
  db:
    image: postgis/postgis
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: app
      POSTGRES_DB: app
    volumes:
      - dbdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app -d app"]
      interval: 5s
      timeout: 5s
      retries: 10

  pgadmin:
    image: dpage/pgadmin4
    environment:
      PGADMIN_DEFAULT_EMAIL: admin@example.com
      PGADMIN_DEFAULT_PASSWORD: admin
    depends_on:
      - db
    ports: ["5050:80"]

  ingestor:
    build: ./ingestor
    environment:
      DATABASE_URL: postgres://app:app@db:5432/app?sslmode=disable
      DETECTIONS_URL: ${DETECTIONS_URL:?set in .env}
      WEATHER_URL: ${WEATHER_URL:?set in .env}
      FETCH_INTERVAL_SECS: "120"
    depends_on:
      db:
        condition: service_healthy

volumes:
  dbdata:
```

Initialize PostGIS and tables:
```sql
-- psql -h localhost -U app -d app
CREATE EXTENSION IF NOT EXISTS postgis;

CREATE TABLE IF NOT EXISTS detections (
  id TEXT PRIMARY KEY,
  ts TIMESTAMPTZ NOT NULL,
  geom geometry(Point, 4326) NOT NULL,
  raw jsonb NOT NULL
);

CREATE TABLE IF NOT EXISTS weather (
  id TEXT PRIMARY KEY,
  ts TIMESTAMPTZ NOT NULL,
  geom geometry(Point, 4326),
  raw jsonb NOT NULL
);

CREATE TABLE IF NOT EXISTS alerts (
  id SERIAL PRIMARY KEY,
  detection_id TEXT REFERENCES detections(id),
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  reason TEXT
);

CREATE TABLE IF NOT EXISTS last_fetched (
  source TEXT PRIMARY KEY,
  ts TIMESTAMPTZ NOT NULL
);
```

Basic Go ingestor (skeleton). We’ll use `net/http`, JSON decoding, and idempotent upserts:

```go
// ./ingestor/main.go
package main

import (
  "context"
  "database/sql"
  "encoding/json"
  "log"
  "net/http"
  "os"
  "strconv"
  "time"

  _ "github.com/lib/pq"
)

type Detection struct {
  ID  string    `json:"id"`
  TS  time.Time `json:"ts"`
  Lat float64   `json:"lat"`
  Lng float64   `json:"lng"`
  Raw json.RawMessage `json:"raw"`
}

type Weather struct {
  ID  string    `json:"id"`
  TS  time.Time `json:"ts"`
  Lat float64   `json:"lat"`
  Lng float64   `json:"lng"`
  Raw json.RawMessage `json:"raw"`
}

func main() {
  dbURL := mustEnv("DATABASE_URL")
  detURL := mustEnv("DETECTIONS_URL")
  wURL := mustEnv("WEATHER_URL")
  intervalSecs := mustInt("FETCH_INTERVAL_SECS", 120)

  db, err := sql.Open("postgres", dbURL)
  if err != nil { log.Fatal(err) }
  defer db.Close()

  ticker := time.NewTicker(time.Duration(intervalSecs) * time.Second)
  defer ticker.Stop()

  httpClient := &http.Client{ Timeout: 30 * time.Second }

  for {
    if err := fetchDetections(context.Background(), db, httpClient, detURL); err != nil {
      log.Printf("detections fetch error: %v", err)
    }
    if err := fetchWeather(context.Background(), db, httpClient, wURL); err != nil {
      log.Printf("weather fetch error: %v", err)
    }
    <-ticker.C
  }
}

func fetchDetections(ctx context.Context, db *sql.DB, c *http.Client, url string) error {
  req, _ := http.NewRequestWithContext(ctx, http.MethodGet, url, nil)
  resp, err := c.Do(req)
  if err != nil { return err }
  defer resp.Body.Close()

  var items []Detection
  if err := json.NewDecoder(resp.Body).Decode(&items); err != nil { return err }

  for _, it := range items {
    _, err := db.Exec(`
      INSERT INTO detections (id, ts, geom, raw)
      VALUES ($1, $2, ST_SetSRID(ST_Point($3, $4), 4326), $5)
      ON CONFLICT (id) DO UPDATE
      SET ts = EXCLUDED.ts, geom = EXCLUDED.geom, raw = EXCLUDED.raw
    `, it.ID, it.TS, it.Lng, it.Lat, it.Raw)
    if err != nil { log.Printf("upsert detection %s: %v", it.ID, err) }
  }
  _, _ = db.Exec(`INSERT INTO last_fetched (source, ts) VALUES ($1, now())
                  ON CONFLICT (source) DO UPDATE SET ts = EXCLUDED.ts`, "detections")
  return nil
}

func fetchWeather(ctx context.Context, db *sql.DB, c *http.Client, url string) error {
  req, _ := http.NewRequestWithContext(ctx, http.MethodGet, url, nil)
  resp, err := c.Do(req)
  if err != nil { return err }
  defer resp.Body.Close()

  var items []Weather
  if err := json.NewDecoder(resp.Body).Decode(&items); err != nil { return err }

  for _, it := range items {
    _, err := db.Exec(`
      INSERT INTO weather (id, ts, geom, raw)
      VALUES ($1, $2, ST_SetSRID(ST_Point($3, $4), 4326), $5)
      ON CONFLICT (id) DO UPDATE
      SET ts = EXCLUDED.ts, geom = EXCLUDED.geom, raw = EXCLUDED.raw
    `, it.ID, it.TS, it.Lng, it.Lat, it.Raw)
    if err != nil { log.Printf("upsert weather %s: %v", it.ID, err) }
  }
  _, _ = db.Exec(`INSERT INTO last_fetched (source, ts) VALUES ($1, now())
                  ON CONFLICT (source) DO UPDATE SET ts = EXCLUDED.ts`, "weather")
  return nil
}

func mustEnv(k string) string {
  v := os.Getenv(k)
  if v == "" { log.Fatalf("missing %s", k) }
  return v
}
func mustInt(k string, def int) int {
  v := os.Getenv(k)
  if v == "" { return def }
  n, err := strconv.Atoi(v); if err != nil { return def }
  return n
}
```

Dockerfile for the ingestor:

```dockerfile
# ./ingestor/Dockerfile
FROM golang:alpine AS build
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -o /bin/ingestor ./main.go

FROM alpine
RUN adduser -D -g '' app
USER app
COPY --from=build /bin/ingestor /bin/ingestor
ENTRYPOINT ["/bin/ingestor"]
```

Bring it up and watch logs:
```bash
docker compose up -d --build
docker compose logs -f ingestor
```

Verify data landed:
```bash
docker compose exec -it db psql -U app -d app -c "SELECT count(*) FROM detections;"
docker compose exec -it db psql -U app -d app -c "SELECT count(*) FROM weather;"
```

Throw in a simple health check cron so you know the ingestor hasn’t died. The best monitoring is a cron job that curls your health endpoint.

```bash
# health.sh
#!/usr/bin/env bash
set -euo pipefail
curl -fsS http://localhost:8080/health || echo "ingestor health failed" >&2
```

Add to root’s crontab:
```bash
*/5 * * * * /usr/local/bin/health.sh
```

Contrarian take: you probably don’t need more ML here. You need better deterministic joins and clear thresholds. The HN crowd keeps flocking to deep, implementation‑heavy posts like “How kernel anti‑cheats work” (244 points, 203 comments: [article](https://s4dbrd.github.io/posts/how-kernel-anti-cheats-work/), [HN](https://news.ycombinator.com/item?id=47382791)) and a TCP hole punching algorithm (134 points: [article](https://robertsdotpm.github.io/cryptography/tcp_hole_punching.html), [HN](https://news.ycombinator.com/item?id=47384032)). Not hype. Systems.

## 3. Add an AI Brain That Doesn’t Behave Like Spotify’s DJ

That AI DJ piece got 226 points and 191 comments ([article](https://www.charlespetzold.com/blog/2026/02/The-Appalling-Stupidity-of-Spotifys-AI-DJ.html), [HN](https://news.ycombinator.com/item?id=47385272)). The message: brittle, generic outputs erode trust fast. So our tiny “AI brain” needs to be consistent, explainable, and falsifiable.

Stand up a small AI microservice. You can wrap a local model like [llama.cpp](https://github.com/ggerganov/llama.cpp) or call a hosted API — either way, front it with a minimal web server that returns structured JSON. Here’s a Python example using [FastAPI](https://fastapi.tiangolo.com/) and [Uvicorn](https://www.uvicorn.org/):

```python
# ./ai/app.py
from fastapi import FastAPI
from pydantic import BaseModel
from typing import Any, Dict, Optional
import time

app = FastAPI()

class Context(BaseModel):
    detection: Dict[str, Any]
    weather: Optional[Dict[str, Any]] = None

class Result(BaseModel):
    decision: str
    confidence: float
    explanation: str
    latency_ms: int

@app.post("/classify", response_model=Result)
def classify(ctx: Context) -> Result:
    start = time.time()
    # Replace with your model call. For now: rules + placeholder.
    score = 0.5
    reason = "baseline"
    if ctx.weather and ctx.weather.get("wind_speed", 0) > 20:
        score += 0.2
        reason += " + high wind"
    if ctx.detection.get("sat_source") == "trusted":
        score += 0.2
        reason += " + trusted sensor"
    decision = "track" if score >= 0.7 else "ignore"
    return Result(
        decision=decision,
        confidence=float(round(score, 2)),
        explanation=reason,
        latency_ms=int((time.time() - start) * 1000)
    )
```

Dockerfile:

```dockerfile
# ./ai/Dockerfile
FROM python:slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY app.py .
CMD ["uvicorn", "app:app", "--host", "0.0.0.0", "--port", "9000"]
```

requirements.txt:
```
fastapi
uvicorn
pydantic
```

Compose wire‑up (add to docker-compose.yml):
```yaml
  ai:
    build: ./ai
    ports: ["9000:9000"]
```

Now connect the ingestor to the AI service asynchronously. Add a `jobs` table and an `ai_assessments` table:

```sql
CREATE TABLE IF NOT EXISTS jobs (
  id SERIAL PRIMARY KEY,
  detection_id TEXT NOT NULL,
  status TEXT NOT NULL DEFAULT 'pending',
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE IF NOT EXISTS ai_assessments (
  id SERIAL PRIMARY KEY,
  detection_id TEXT NOT NULL,
  decision TEXT NOT NULL,
  confidence NUMERIC NOT NULL,
  explanation TEXT,
  latency_ms INT,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

Minimal Go worker loop (poll jobs, call AI, store results). This can be a second binary or built into the ingestor as a separate `cmd`:

```go
// pseudo: enqueue on new detection
_, _ = db.Exec(`INSERT INTO jobs (detection_id) VALUES ($1) ON CONFLICT DO NOTHING`, det.ID)

// worker: polls jobs
for {
  var id int
  var detID string
  err := db.QueryRow(`UPDATE jobs
                      SET status='processing', updated_at=now()
                      WHERE id = (
                        SELECT id FROM jobs WHERE status='pending' ORDER BY id FOR UPDATE SKIP LOCKED LIMIT 1
                      )
                      RETURNING id, detection_id`).Scan(&id, &detID)
  if err != nil { time.Sleep(time.Second); continue }

  // load detection + weather context (omitted)
  // call AI service
  reqBody := map[string]any{"detection": detObj, "weather": weatherObj}
  // http POST to http://ai:9000/classify, decode JSON into result

  // store result
  _, _ = db.Exec(`INSERT INTO ai_assessments (detection_id, decision, confidence, explanation, latency_ms)
                  VALUES ($1,$2,$3,$4,$5)`, detID, decision, confidence, explanation, latencyMs)
  _, _ = db.Exec(`UPDATE jobs SET status='done', updated_at=now() WHERE id=$1`, id)
}
```

Guard against one‑shot prompts by sampling a few attempts and scoring them. “Tree Search Distillation for Language Models Using PPO” got interest on HN ([post](https://ayushtambde.com/blog/tree-search-distillation-for-language-models-using-ppo/), [HN](https://news.ycombinator.com/item?id=47383059)). You don’t need a research rig: do 2–3 internal passes and pick the one with the clearest, highest‑confidence structured output.

Design your prompt for structure. Example:

```
You are a classification service. Return ONLY JSON with:
- decision: "track" | "ignore"
- confidence: 0..1
- explanation: short reason

Context:
{{detection_json}}
{{weather_json}}
```

Sample response:
```json
{"decision":"track","confidence":0.78,"explanation":"trusted sensor + high wind"}
```

Evaluation loop. Don’t narrate after the fact — record a prediction now, revisit later, and score it. Inspired by clear ML teaching like “A Visual Introduction to Machine Learning (2015)” ([post](https://r2d3.us/visual-intro-to-machine-learning-part-1/), [HN](https://news.ycombinator.com/item?id=47386116)). Create a simple table to mark correctness:

```sql
ALTER TABLE ai_assessments ADD COLUMN expected_outcome TEXT;
ALTER TABLE ai_assessments ADD COLUMN evaluated_at TIMESTAMPTZ;
ALTER TABLE ai_assessments ADD COLUMN correct BOOLEAN;

-- A tiny job marks rows later based on a heuristic or ground truth input you provide.
```

Expose a minimal `/metrics/ai` with counts, accuracy, and latency buckets from your web service. If it’s not measured, you’re guessing. And guessing is how you ship an AI DJ that people dunk on.

## 4. Make Your Self‑Hosted Stack Reachable with NAT‑Friendly Networking

You might have your VPS in the cloud and a box at home. Or a sensor on a client’s LAN. You want secure connectivity without flinging random ports open. Enter NAT traversal.

Read “A most elegant TCP hole punching algorithm” (134 points, 47 comments: [article](https://robertsdotpm.github.io/cryptography/tcp_hole_punching.html), [HN](https://news.ycombinator.com/item?id=47384032)) for the mental model: two peers, both behind NATs, coordinate through a small rendezvous server, then connect directly.

Practical rendezvous server (WebSocket‑ish handshake). Use [Node.js](https://nodejs.org/) or [Go](https://go.dev/). Sketch (Node + [ws](https://github.com/websockets/ws)):

```javascript
// rendezvous.js
const WebSocket = require('ws');
const wss = new WebSocket.Server({ port: 7777 });
const peers = new Map();

wss.on('connection', (ws) => {
  ws.on('message', (msg) => {
    const m = JSON.parse(msg);
    if (m.type === 'register') {
      peers.set(m.id, ws);
    } else if (m.type === 'offer') {
      const peer = peers.get(m.to);
      if (peer) peer.send(JSON.stringify({type:'offer', from:m.from, data:m.data}));
    } else if (m.type === 'answer') {
      const peer = peers.get(m.to);
      if (peer) peer.send(JSON.stringify({type:'answer', from:m.from, data:m.data}));
    }
  });
});
```

Run that on your VPS, have two peers exchange offers/answers, then attempt direct TCP connections using the negotiated tuples. For many indie setups, though, you’ll be happier with a proven overlay.

Alternative: use [WireGuard](https://www.wireguard.com/) or a managed mesh like [Tailscale](https://tailscale.com/) to create a private network. A minimal WireGuard config (server side):

```
# /etc/wireguard/wg0.conf
[Interface]
PrivateKey = <server-private-key>
Address = 10.6.0.1/24
ListenPort = 51820

[Peer]
PublicKey = <client-public-key>
AllowedIPs = 10.6.0.2/32
```

Client side:

```
[Interface]
PrivateKey = <client-private-key>
Address = 10.6.0.2/32
DNS = 1.1.1.1

[Peer]
PublicKey = <server-public-key>
Endpoint = <server-ip>:51820
AllowedIPs = 10.6.0.0/24
PersistentKeepalive = 25
```

Bring it up with `wg-quick up wg0`. Now your laptop can hit `http://10.6.0.1:9000/metrics` without exposing the port publicly.

Security interlude: key pairs matter; don’t roll your own crypto. If you want to go deep on number theory, “Generating All 32‑Bit Primes (Part I)” has an HN thread ([post](https://hnlyman.github.io/pages/prime32_I.html), [HN](https://news.ycombinator.com/item?id=47386441)). For production, stick to known‑good libraries and defaults.

Tie‑back: with this overlay, your ingestor + AI live on the VPS, and you can pull metrics or dashboards from your laptop or a LAN machine safely. No public dashboards. No footguns.

## 5. Observe Everything Without Becoming a Kernel Rootkit

That “How kernel anti‑cheats work” piece (244 points, 203 comments: [article](https://s4dbrd.github.io/posts/how-kernel-anti-cheats-work/), [HN](https://news.ycombinator.com/item?id=47382791)) shows how deep kernel‑level monitoring goes: system calls, memory inspection, driver hooks. Powerful ideas. We’re not writing anti‑cheat software, but the same primitives inspire excellent observability — ethically, on our own boxes.

Use supported tooling:
- eBPF‑based tracing with [bcc](https://github.com/iovisor/bcc) or [bpftrace](https://github.com/iovisor/bpftrace).
- Container introspection with `docker stats`, `docker top`, and `docker logs`.
- Metrics pipeline with [Prometheus](https://prometheus.io/), [node_exporter](https://github.com/prometheus/node_exporter), and [cAdvisor](https://github.com/google/cadvisor).
- Visuals with [Grafana](https://grafana.com/).

Quick wins:
```bash
# Which container is hungry?
docker stats

# Tail logs for AI and ingestor
docker compose logs -f ai ingestor

# bpftrace: trace read() syscalls longer than 10ms for the ingestor
sudo bpftrace -e 'tracepoint:syscalls:sys_enter_read /comm == "ingestor" / { @count = count(); }'
```

Compose snippet to add exporters:
```yaml
  node_exporter:
    image: prom/node-exporter
    pid: host
    network_mode: host

  cadvisor:
    image: gcr.io/cadvisor/cadvisor
    command:
      - "--docker_only=true"
    ports: ["8081:8080"]
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro

  prometheus:
    image: prom/prometheus
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
    ports: ["9090:9090"]
```

Simple alerting can be as basic as a script that watches latency and fires a webhook or email when thresholds are crossed. Don’t wait for users to tell you it’s on fire.

Clean boundary: anti‑cheat research is great for understanding the guts. For indie ops, use supported tracing and metrics that won’t trash system stability or user trust.

## 6. A Minimal Indie DevOps Toolkit: From Prototype to Repeatable Deploy

Let’s wire it all up. One VPS, one Compose file, one deploy script, one backup script. The “100 hour gap” reminder is hanging over your keyboard ([post](https://kanfa.macbudkowski.com/vibecoding-cryptosaurus), [HN](https://news.ycombinator.com/item?id=47386636)). So we’ll add the boring bits.

Final docker-compose.yml (abridged):
```yaml
services:
  db:
    image: postgis/postgis
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: app
      POSTGRES_DB: app
    volumes: [ "dbdata:/var/lib/postgresql/data" ]

  ai:
    build: ./ai
    ports: ["9000:9000"]

  ingestor:
    build: ./ingestor
    environment:
      DATABASE_URL: postgres://app:app@db:5432/app?sslmode=disable
      DETECTIONS_URL: ${DETECTIONS_URL}
      WEATHER_URL: ${WEATHER_URL}
      FETCH_INTERVAL_SECS: "120"
    depends_on: [ "db" ]

  cadvisor:
    image: gcr.io/cadvisor/cadvisor
    ports: ["8081:8080"]

volumes:
  dbdata:
```

Deploy script:
```bash
# deploy.sh
#!/usr/bin/env bash
set -euo pipefail
git pull
docker compose build
docker compose up -d
docker system prune -f
```

Postgres backup script:
```bash
# backup.sh
#!/usr/bin/env bash
set -euo pipefail
TS=$(date +%Y%m%d-%H%M%S)
docker compose exec -T db pg_dump -U app app | gzip > backups/app-$TS.sql.gz
```

Run your end‑to‑end health check:
- SSH to the box.
- `docker compose ps` shows everything healthy.
- Trigger a fake detection via a small seed file or endpoint.
- Watch it flow: `ingestor` logs show upsert, a `job` is created, AI returns a decision, row lands in `ai_assessments`.
- Hit your metrics endpoint over [WireGuard](https://www.wireguard.com/) or your rendezvous tunnel.
- Restore a backup locally as a rehearsal.

Now you’ve got a stack you understand. You can extend it weekly: add one dataset (e.g., Census or OpenStreetMap‑style context like [OpenStreetMap](https://www.openstreetmap.org/), as mentioned alongside Signet), one metric, or one evaluation. Small wins compound.

## Cheat Sheet

- Signet inspiration: [Signet](https://signet.watch) — built in Go to automate wildfire monitoring; data from NASA FIRMS, GOES‑19, NWS, LANDFIRE, USGS, Census, OpenStreetMap; challenge: different sources/cadences/formats; most of it is deterministic plumbing.
- HN interest, deep systems: Kernel anti‑cheats ([article](https://s4dbrd.github.io/posts/how-kernel-anti-cheats-work/), [HN](https://news.ycombinator.com/item?id=47382791)); TCP hole punching ([article](https://robertsdotpm.github.io/cryptography/tcp_hole_punching.html), [HN](https://news.ycombinator.com/item?id=47384032)).
- HN cautionary AI product: Spotify’s AI DJ critique ([article](https://www.charlespetzold.com/blog/2026/02/The-Appalling-Stupidity-of-Spotifys-AI-DJ.html), [HN](https://news.ycombinator.com/item?id=47385272)).
- Practical reasoning trick: Tree Search Distillation write‑up ([post](https://ayushtambde.com/blog/tree-search-distillation-for-language-models-using-ppo/), [HN](https://news.ycombinator.com/item?id=47383059)).
- ML clarity for evaluation thinking: Visual intro to ML ([post](https://r2d3.us/visual-intro-to-machine-learning-part-1/), [HN](https://news.ycombinator.com/item?id=47386116)).
- Core tools: [Docker](https://www.docker.com/), [Docker Compose](https://docs.docker.com/compose/), [PostgreSQL](https://www.postgresql.org/), [PostGIS](https://postgis.net/), [pgAdmin](https://www.pgadmin.org/), [Caddy](https://caddyserver.com/), [NGINX](https://nginx.org/), [Go](https://go.dev/), [Python](https://www.python.org/), [Node.js](https://nodejs.org/).
- AI microservice scaffold: [FastAPI](https://fastapi.tiangolo.com/), [Uvicorn](https://www.uvicorn.org/), optional local model wrapper [llama.cpp](https://github.com/ggerganov/llama.cpp).
- NAT/overlay: TCP rendezvous (see hole punching article), [WireGuard](https://www.wireguard.com/), [Tailscale](https://tailscale.com/).
- Observability: eBPF with [bcc](https://github.com/iovisor/bcc) or [bpftrace](https://github.com/iovisor/bpftrace), container view with `docker stats`, metrics via [Prometheus](https://prometheus.io/), [node_exporter](https://github.com/prometheus/node_exporter), [cAdvisor](https://github.com/google/cadvisor), dashboards in [Grafana](https://grafana.com/).
- Health/backup glue: `cron` + [curl](https://curl.se/) for health checks, `pg_dump` for backups.

## Wrap

You just stitched together the same patterns that make the HN crowd stop scrolling: a deterministic ingestion loop straight out of the Signet mindset, an AI layer that’s measured (not mystical), NAT‑friendly networking so you can reach your box without opening yourself up, and serious observability that respects the kernel’s boundaries.

Fork your repo, run the checklist, and iterate weekly: one new data source, one new metric, one tiny eval. You don’t need Kubernetes. You need a VPS and a Makefile. Own your stack. Ship it.
