---
title: "Ship Your Own Stack: A Practical Guide to Self‑Hosted, AI‑Assisted Dev on One Small Server"
date: 2026-03-03T14:06:37.128Z
draft: false
tags:
  - "tutorial"
  - "devops"
  - "indie-dev"
---
# Ship Your Own Stack: A Practical Guide to Self‑Hosted, AI‑Assisted Dev on One Small Server

Somebody leaked a Gemini API key and woke up to an $82,000 bill 48 hours later. That really happened, and it’s documented on [llmhorrors.com](https://llmhorrors.com/all/gemini-stolen-api-key-82k/). Here’s the thing — you don’t need to be the next headline because your side project quietly blasted an LLM API all weekend. We’re going to put the guardrails on a single box: firewall, rate limits, and basic alerts. Cheap. Fast. Understandable.

Meanwhile, AI is also busy breaking trust in public. [Futurism](https://futurism.com/artificial-intelligence/ars-technica-fires-reporter-ai-quotes) covered a reporter getting fired at Ars Technica after an AI‑related mess with fabricated quotes. [BBC News](https://www.bbc.com/news/articles/c178zzw780xo) wrote about India’s top court getting angry when a junior judge cited fake AI‑generated court orders. That’s the reputational and legal blast radius. Financial, reputational, legal — all three are avoidable if you keep your stack simple and observable.

Let me show you what actually works.

We’ll build a single‑node stack that covers: your own DNS, container‑level egress control, indie dev tools via a catalog, dead‑simple docs, and local AI with no cloud dependency. On one small server. No Kubernetes. You need a VPS and a Makefile. Ship it.

---

## What You’re Actually Building (and Why It Fits on One Box)

Self‑hosting is simple: run your own applications and keep your own data so you remove the “unknown” in who’s handling it, while keeping the same functionality you’re used to. That’s how /r/selfhosted’s welcome post puts it. Example: you can replace Dropbox with a self‑hosted option if you don’t want sensitive data on a third‑party provider. Clean premise. Clear goal.

The internet’s discoverability is weirder than it’s been in years. A [Growtika post](https://growtika.com/blog/tech-media-collapse) claims top tech publications have lost 58% of their Google traffic since 2024. And an OSS maintainer said they’re “[losing the SEO battle](https://twitter.com/Gavriel_Cohen/status/2028821432759717930)” for their own project. Translation: don’t bet your docs or your product onboarding on Search saving you. Own your docs. Own your infra.

Here’s what we’ll ship on a modest server:  
- A container host with Docker, a few Compose files, and a private DNS resolver (AdGuard + Unbound) so your services stop talking to strangers for lookups.  
- A label‑driven egress firewall (iptables on the host) so compromised containers can’t call command‑and‑control or LLM APIs by default.  
- Indie‑dev tools you can spin up quickly using The AltStack catalog.  
- A dead‑simple [MkDocs](https://www.mkdocs.org/) docs pipeline you can own and deploy in seconds.  
- A local AI helper with [Ollama](https://ollama.com/) so your “assist” doesn’t require an API key at all.  
- Minimal observability and backups that don’t slow you down.

Also — resist the homelab itch. Plenty of folks in /r/selfhosted show off clusters that idle most of the time and ask what to run on all that hardware. Not necessary for the workloads we’re talking about. Earn upgrades by saturating a single node first.

---

## Step 1 – Bring Up a Lean, Self‑Hosted Base: Docker Host + DIY DNS

Start with one Ubuntu or Debian server. Can be a small VPS or a NUC‑ish machine. Keep the base clean: a non‑root user, SSH keys, and a firewall that allows only SSH and whatever web ports you plan to use later.

- Install Docker and Compose:
  - Docker: https://docs.docker.com/engine/install/
  - Compose plugin: https://docs.docker.com/compose/install/linux/

We’ll run a simplified version of a battle‑tested DNS flow modeled on a setup described in /r/selfhosted: router (dnsmasq) → AdGuard Home → Unbound → root servers. Their full design includes a GL‑MT6000 router, two [AdGuard Home](https://github.com/AdguardTeam/AdGuardHome) instances behind a VIP, [Unbound](https://nlnetlabs.nl/projects/unbound/about/) as the recursive resolver with DNSSEC, and [Redis](https://redis.io/) as a cold cache so Unbound doesn’t start empty after restarts, plus a Raspberry Pi as backup. We’ll condense that to one box.

Why do this? Privacy and control. Also, for local traffic, a nearby resolver is snappy. But the point isn’t chasing microseconds — it’s bringing your DNS in‑house so you know what’s happening.

Compose file for a single‑node DNS stack (one AdGuard, one Unbound, optional Redis):

```yaml
version: "3.8"
services:
  adguard:
    image: adguard/adguardhome:latest
    container_name: adguard
    restart: unless-stopped
    network_mode: host
    volumes:
      - ./adguard/work:/opt/adguardhome/work
      - ./adguard/conf:/opt/adguardhome/conf
    # Web UI defaults to http://server:3000 on first run
    # Configure upstream DNS to point at unbound below (e.g., 127.0.0.1:5353)

  unbound:
    image: klutchell/unbound:latest
    container_name: unbound
    restart: unless-stopped
    ports:
      - "127.0.0.1:5353:53/udp"
      - "127.0.0.1:5353:53/tcp"
    volumes:
      - ./unbound/unbound.conf:/etc/unbound/unbound.conf:ro

  redis:
    image: redis:alpine
    container_name: redis
    restart: unless-stopped
    ports:
      - "127.0.0.1:6379:6379"
    command: ["redis-server", "--save", "60", "1000"]
```

An example `unbound.conf` tuned for validating recursion and talking to root servers directly (DNSSEC on, no forwarders):

```
server:
  username: "_unbound"
  directory: "/etc/unbound"
  auto-trust-anchor-file: "var/root.key"

  interface: 0.0.0.0
  port: 53
  do-ip4: yes
  do-udp: yes
  do-tcp: yes

  hide-identity: yes
  hide-version: yes

  harden-dnssec-stripped: yes
  qname-minimisation: yes
  prefetch: yes
  rrset-roundrobin: yes
  aggressive-nsec: yes

  # Cache tuning is up to you; if you integrate Redis cold cache,
  # use the redis plugin as documented by your image/vendor.

forward-zone:
  name: "."
  forward-first: no
  forward-no-cache: no
  # No forwarders; resolve from root by default
```

Bring it up:

```
docker compose up -d
```

Then, point [AdGuard Home](https://github.com/AdguardTeam/AdGuardHome) upstream to Unbound at 127.0.0.1:5353 via the AdGuard web UI. On your home router (if you’re doing this on‑prem with something like [dnsmasq](https://thekelleys.org.uk/dnsmasq/doc.html)), set its DNS to your AdGuard’s IP. Query chain looks like this: Router → AdGuard → Unbound → Root servers. The reddit post describes exactly that flow and notes Unbound validating DNSSEC and supporting zone transfers for local zones.

Could you just keep using public DNS? Sure. But if you’re self‑hosting because you want control over your data (like replacing Dropbox, per the /r/selfhosted welcome post), then DNS belongs in that circle too.

By the way, people happily run decent stacks — media servers like [Jellyfin](https://jellyfin.org/) with remote access via [Tailscale](https://tailscale.com/) and mobile apps like [Symfonium](https://symfonium.app/) for [Android Auto](https://www.android.com/auto/) — on old small‑form‑factor desktops. That’s straight from a user story in /r/selfhosted. You don’t need a data center to start.

---

## Step 2 – Lock Down Containers and AI Keys: Label‑Driven Egress Control

Most people harden ingress. Fewer people harden egress. That’s a footgun.

A user in /r/selfhosted described a sane container hardening baseline: run containers `read_only`, drop Linux capabilities, run rootless where possible, and enforce memory/CPU/PIDs limits. They worry — correctly — that an exploited container could contact a command‑and‑control server. So they only grant WAN/LAN to containers that need it and use Docker labels like `no-internal` and `no-public` to drive host firewall rules.

Concrete steps you can copy:

1) Harden container specs in Compose:

```yaml
services:
  myapp:
    image: your/image:tag
    read_only: true
    cap_drop: ["ALL"]
    security_opt:
      - no-new-privileges:true
    mem_limit: "512m"
    cpus: "0.50"
    pids_limit: 256
    user: "1000:1000"
    labels:
      - "egress.no-internal=true"
      - "egress.no-public=true"
      # Or permit specific egress by declaring destinations:
      # - "egress.allow.tcp=api.example.com:443"
      # - "egress.allow.ip=1.2.3.4:1234"
    networks:
      - default
```

2) Put egress policy in the host firewall with [iptables](https://netfilter.org/projects/iptables/) (DOCKER‑USER chain). A tiny script to read labels and punch holes only where allowed. This is the vibe from that reddit post — simple, host‑level, observable rules on a bare Ubuntu + Docker machine.

Minimal example (as root):

```bash
#!/usr/bin/env bash
set -euo pipefail

# Default deny for containers: block egress unless allowed by labels
# Apply in DOCKER-USER so it sees container traffic
iptables -N EXITCLOUD_EGRESS 2>/dev/null || true
iptables -C DOCKER-USER -j EXITCLOUD_EGRESS 2>/dev/null || iptables -I DOCKER-USER -j EXITCLOUD_EGRESS

# Clear existing rules in our chain
iptables -F EXITCLOUD_EGRESS

# Block RFC1918 by default
for cidr in 10.0.0.0/8 172.16.0.0/12 192.168.0.0/16; do
  iptables -A EXITCLOUD_EGRESS -d $cidr -j REJECT
done

# Block public internet by default
iptables -A EXITCLOUD_EGRESS -d 0.0.0.0/0 -j REJECT

# Allow DNS to our local Unbound
iptables -I EXITCLOUD_EGRESS -p udp --dport 53 -d 127.0.0.1 -j ACCEPT
iptables -I EXITCLOUD_EGRESS -p tcp --dport 53 -d 127.0.0.1 -j ACCEPT

# Punch specific holes from container labels like:
#   egress.allow.tcp=api.example.com:443
containers=$(docker ps -q)
for c in $containers; do
  id=$(docker inspect --format '{{.Id}}' "$c")
  name=$(docker inspect --format '{{.Name}}' "$c" | sed 's#^/##')

  # Resolve per-container IP
  ip=$(docker inspect --format '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' "$c")
  [ -z "$ip" ] && continue

  # Check label flags
  allow_rules=$(docker inspect --format '{{range $k,$v := .Config.Labels}}{{println $k "=" $v}}{{end}}' "$c" | grep '^egress.allow\.' || true)

  # If container explicitly allows endpoints, add them
  while IFS= read -r rule; do
    key=${rule%%=*}
    val=${rule#*=}
    proto=${key##*.}     # tcp or ip
    host=${val%%:*}
    port=${val##*:}

    # Resolve host to IP if needed
    if [[ "$host" =~ [a-zA-Z] ]]; then
      dest_ip=$(getent ahostsv4 "$host" | awk '{print $1; exit}')
    else
      dest_ip=$host
    fi
    [ -z "$dest_ip" ] && continue

    if [ "$proto" = "tcp" ]; then
      iptables -I EXITCLOUD_EGRESS -s "$ip" -p tcp -d "$dest_ip" --dport "$port" -j ACCEPT
    else
      iptables -I EXITCLOUD_EGRESS -s "$ip" -d "$dest_ip" -j ACCEPT
    fi
  done <<< "$allow_rules"
done
```

This isn’t a full product. It’s a pattern. Labels describe intent; the host enforces it. You should also block traffic to your LAN ranges by default, as that reddit user does, and only allow what a container truly needs.

Now tie it to AI keys. The financial blast radius is real: the [Gemini API key incident](https://llmhorrors.com/all/gemini-stolen-api-key-82k/) is the nightmare scenario. Keep your AI keys in env files or a secrets manager, let only one service read them, and set strict egress on that service so it can call only the one domain you expect. Add a basic counter in logs for outbound calls (grep the container logs, increment a daily number), and alert if it spikes. A bash one‑liner in cron is enough to catch “why did we call api.vendor.com 10,000 times today?” before billing does.

And about automation: the post “[We Automated Everything Except Knowing What’s Going On](https://eversole.dev/blog/we-automated-everything/)” is the cautionary tale. Don’t build a Rube Goldberg egress pipeline you can’t explain at 2 AM. Simple rules. Text logs you can read. Clear labels.

We also live in a world where AI‑assisted content can nuke credibility fast — [fabricated quotes](https://futurism.com/artificial-intelligence/ars-technica-fires-reporter-ai-quotes), [fake court orders](https://www.bbc.com/news/articles/c178zzw780xo). Treat outbound API calls and generated content as untrusted until it passes human review or automated sanity checks. You’re the operator.

---

## Step 3 – Self‑Hosted Dev & Docs: AltStack Services + Dead‑Simple MkDocs Pipeline

You don’t need to spend a weekend yak shaving every service. The AltStack is an open‑source directory of 450+ self‑hostable alternatives across 28 categories — with 56 tools offering copy‑paste Docker Compose configs you can bring up with `docker compose up`. It also provides side‑by‑side comparisons (e.g., [Supabase](https://supabase.com/) vs [Appwrite](https://appwrite.io/) vs [PocketBase](https://pocketbase.io/)), a SaaS savings calculator, and “best‑of” rankings based on GitHub stars and community health. That’s the short path to an indie‑friendly stack: a BaaS, analytics, project management, CRM, DevOps, monitoring — pick what you need, not everything.

For docs, resist the temptation to over‑automate. One /r/selfhosted user ran a beautiful little pipeline for a static site: a bare Git repo on a home server, a post‑receive hook that runs `mkdocs build`, and serve the built `/site` via an [Nginx](https://nginx.org/) container. Later they moved to [Gitea](https://about.gitea.com/) in Docker and felt it got more complicated than the Git‑hook‑based approach. I’ve been there. Simple usually wins.

Here’s the “classic” pattern I still use for small projects:

- On your server:
  ```
  sudo apt-get update
  sudo apt-get install -y git python3-pip
  pip3 install mkdocs
  sudo mkdir -p /srv/docs.git
  sudo chown -R $USER:$USER /srv/docs.git
  cd /srv/docs.git
  git init --bare
  ```

- Create `/srv/docs.git/hooks/post-receive`:
  ```bash
  #!/usr/bin/env bash
  set -e
  GIT_WORK_TREE=/srv/docs-src git checkout -f
  cd /srv/docs-src
  mkdocs build --clean
  ```

  Make it executable:
  ```
  chmod +x /srv/docs.git/hooks/post-receive
  ```

- Serve the built site with Nginx in Docker:

  ```yaml
  version: "3.8"
  services:
    docs:
      image: nginx:alpine
      container_name: nginx_docs
      restart: unless-stopped
      ports:
        - "80:80"
      volumes:
        - /srv/docs-src/site:/usr/share/nginx/html:ro
  ```

- On your laptop, add the remote and push:
  ```
  git remote add prod ssh://user@server/srv/docs.git
  git push prod main
  ```

That’s it. Push. Build. Served.

As discoverability shifts — with [tech media traffic collapsing](https://growtika.com/blog/tech-media-collapse) and even OSS authors “[losing the SEO battle](https://twitter.com/Gavriel_Cohen/status/2028821432759717930)” — owning your docs matters more. Keep them up, fast to deploy, and under your control.

If you want basic office tooling on this same box, there are self‑hosted options. A /r/selfhosted post notes that LibreOffice Online is restarting development after a 2022 pause, and mentions other online office suites like [OnlyOffice](https://www.onlyoffice.com/), [Collabora Online](https://www.collaboraoffice.com/collabora-online/), and [NeoOffice](https://www.neooffice.org/). Pick one and keep your docs local.

---

## Step 4 – Add Local AI to Your Workflow Without Becoming a Horror Story

Run a local model with [Ollama](https://ollama.com/) for coding help, config review, and log triage — no API key, no bill. Keep it boring and it’ll keep you out of trouble.

Compose snippet:

```yaml
version: "3.8"
services:
  ollama:
    image: ollama/ollama:latest
    container_name: ollama
    restart: unless-stopped
    volumes:
      - ./ollama:/root/.ollama
    ports:
      - "11434:11434"
    labels:
      - "egress.no-public=true"
      - "egress.no-internal=true"
```

After it’s up, you can pull a model and chat via CLI on the server or through a thin UI of your choice. The point is: your prompts never leave the box.

If you want to get fancy, there’s a local smart‑home voice architecture described in /r/selfhosted: an Android tablet as a dashboard and mic, a home server running speech‑to‑text/ text‑to‑speech models, a Wyoming‑OpenAI proxy for the Wyoming protocol, [Ollama](https://ollama.com/) as the conversation agent, and [Home Assistant](https://www.home-assistant.io/) to control devices. Reported outcome: it works well as a replacement for [Google Nest](https://store.google.com/us/category/nest_speakers_displays) or Amazon Alexa, fully local and private. You can adapt that idea for dev ops too — a voice command to restart a service or summarize logs for the last hour.

But — AI is a co‑pilot, not an autopilot. If you let it draft a Dockerfile or an iptables rule, validate it. The reputational and legal risks are real, as seen in the [Ars Technica firing story](https://futurism.com/artificial-intelligence/ars-technica-fires-reporter-ai-quotes) and [India’s fake orders incident](https://www.bbc.com/news/articles/c178zzw780xo). And the financial risk is obvious from the [Gemini API key bill](https://llmhorrors.com/all/gemini-stolen-api-key-82k/). Local models plus egress control buys you peace of mind.

Culture note: the /r/selfhosted mods say AI made it easier to spin up weekend “vibe‑coded” projects, which led to an influx of AI‑generated posts — and more negative community reactions to low‑effort stuff. Aim higher. Tests, docs, hardening. Move past the demo.

---

## Step 5 – Observability, Backups, and Surviving Your First Outage

The best monitoring is a cron job that curls your health endpoint. Start there.

Observability on one server doesn’t have to mean a massive stack. Do these:

- Central logs: pipe Docker logs to files you can grep.  
- Health checks: a tiny `/healthz` in each service and a cron job that curls them every minute and emails you on failure.  
- Key metrics: disk usage, container restarts, network errors. A weekly email report is often enough early on.

Backups: copy your databases, Docker volumes, DNS configs, and MkDocs repo to another disk or an off‑site target on a schedule. Encrypt if it leaves the box. Keep restore steps in a README next to the backup script. Boring? Yes. Essential.

Upgrades: write a two‑line “upgrade playbook” so you don’t wing it. The wrong update at the wrong time can take you down. One /r/selfhosted user reported running services for a few months, including an [Audiobookshelf](https://www.audiobookshelf.org/) instance with roughly 20 users, and had their first outage when a TrueNAS OS update (from version 25.04.1 to 25.04.2.6) broke [Portainer](https://www.portainer.io/) and multiple Docker containers. The server was down for less than 10 minutes before messages started coming in from users who relied on it. That’s the moment you realize your “small” stack matters to people.

Automation is great until it hides what’s going on. The “[We Automated Everything Except Knowing What’s Going On](https://eversole.dev/blog/we-automated-everything/)” post nails this. Don’t wire auto‑updates to prod without visibility. Stage changes. Log what changed.

The hardware cautionary tale extends beyond servers. A user tried cheap Tuya WiFi breakers that dropped WiFi after idle, didn’t reconnect, depended on cloud firmware, and lost security updates when the cloud was blocked — so they reverse‑engineered the fallback to BLE and built a [BLE → WiFi → MQTT](https://mqtt.org/) proxy on a coin‑sized ESP with a small web UI, making the breakers fully local with no cloud dependency and no WiFi drop issues (project). Why mention this? Because awareness and control are the whole game. If a $20 breaker can strand you, so can opaque automation on your server.

Same vibe with media: someone described using a GoPro Hero 8 with a 256GB microSD and a $5/month cloud storage subscription to record lots of life — commutes, family outings, conversations — then wondered how to self‑host that going forward (story). If you’re going to add media‑heavy workloads, decide retention and backup first. Otherwise you’re just pushing the “unknowns” into your own garage.

Keep it small. Keep it visible. Keep it recoverable.

---

## Quick Reference

- [Docker](https://www.docker.com/) — container runtime for all services on one box
- [Docker Compose](https://docs.docker.com/compose/) — define multi‑container apps
- [AdGuard Home](https://github.com/AdguardTeam/AdGuardHome) — DNS sinkhole and filtering; point upstream to Unbound
- [Unbound](https://nlnetlabs.nl/projects/unbound/about/) — validating recursive DNS resolver (DNSSEC)
- [Redis](https://redis.io/) — optional “cold cache” backing for Unbound as described in a reddit setup
- [dnsmasq](https://thekelleys.org.uk/dnsmasq/doc.html) — lightweight DNS/DHCP on many routers; forward to AdGuard
- Raspberry Pi — can serve as a backup Unbound node in a split setup
- /r/selfhosted DNS thread — model architecture: Router → AdGuard (VIP) → Unbound → Root servers; Redis cold‑cache; backup Pi
- [iptables](https://netfilter.org/projects/iptables/) — host firewall; use DOCKER‑USER to enforce egress policy
- The AltStack — 450+ self‑hostable alternatives, 28 categories, 56 ready Compose files; comparisons and rankings
- [Supabase](https://supabase.com/), [Appwrite](https://appwrite.io/), [PocketBase](https://pocketbase.io/) — example BaaS options compared in AltStack
- [MkDocs](https://www.mkdocs.org/) — static docs generator; simple Git hook deploys
- [Nginx](https://nginx.org/) — serve your static `/site` directory
- [Gitea](https://about.gitea.com/) — self‑hosted Git; can replace the bare‑repo workflow if you want
- LibreOffice Online (dev restart post) — development restarting; alternatives: [OnlyOffice](https://www.onlyoffice.com/), [Collabora Online](https://www.collaboraoffice.com/collabora-online/), [NeoOffice](https://www.neooffice.org/)
- [Ollama](https://ollama.com/) — run local LLMs; no external API key
- Wyoming‑OpenAI proxy — Wyoming protocol bridge used in a local voice assistant stack
- [Home Assistant](https://www.home-assistant.io/) — home automation; local voice assistant control target
- [Tailscale](https://tailscale.com/) — WireGuard‑based overlay; used in a self‑hosted music server story
- [Jellyfin](https://jellyfin.org/) — media server used for self‑hosted music
- [Symfonium](https://symfonium.app/) — paid mobile client; works with [Android Auto](https://www.android.com/auto/)
- [Audiobookshelf](https://www.audiobookshelf.org/) — audiobook server in the first‑outage story
- [Portainer](https://www.portainer.io/) — Docker management GUI that broke after a TrueNAS update in that story
- [MQTT](https://mqtt.org/) — lightweight messaging protocol; used in Tuya breaker proxy
- Tuya breaker proxy project — BLE → WiFi → MQTT proxy to make breakers local
- /r/selfhosted welcome post — definition of self‑hosting and replacing Dropbox
- Homelab over‑provisioning thread — cautionary tale about idle clusters
- [Tech media traffic collapse](https://growtika.com/blog/tech-media-collapse) — claim about 58% Google traffic drop since 2024
- [“Losing the SEO battle” tweet](https://twitter.com/Gavriel_Cohen/status/2028821432759717930) — discoverability challenge even for project authors
- [We Automated Everything Except Knowing What’s Going On](https://eversole.dev/blog/we-automated-everything/) — caution on automation outrunning understanding
- [Fired reporter over AI quotes](https://futurism.com/artificial-intelligence/ars-technica-fires-reporter-ai-quotes) — reputational risk
- [Fake AI court orders story](https://www.bbc.com/news/articles/c178zzw780xo) — legal risk
- [Stolen Gemini API key: $82k in 48h](https://llmhorrors.com/all/gemini-stolen-api-key-82k/) — financial risk
- /r/selfhosted “Vibe Code Friday” mod note — influx of AI‑generated posts; community reaction
- First outage story — TrueNAS upgrade broke Portainer and containers; users noticed quickly
- GoPro life‑logging to self‑host — motivation to plan storage/backup
- [Dawarich](https://github.com/Freika/dawarich) — self‑hosted alternative to Google Timeline; started simple, initially shipped without auth; ingests OwnTracks iOS data

---

You just went from “random VPS” to a coherent, self‑hosted stack: your own DNS, a label‑driven firewall, an indie‑friendly toolchain via The AltStack, a stupid‑simple [MkDocs](https://www.mkdocs.org/) docs pipeline, and a local AI helper via [Ollama](https://ollama.com/). It fits on one small server instead of a 48‑core monument to idle CPU, and you’ve got just enough observability and backup to avoid surprises.

What’s next? Two weekends, real outcomes:
- Add [Dawarich](https://github.com/Freika/dawarich) as your self‑hosted Google Timeline alternative.
- Bring your office into LibreOffice Online or [OnlyOffice](https://www.onlyoffice.com/), keep control of files.
- De‑cloud a device like those Tuya breakers using MQTT and [Home Assistant](https://www.home-assistant.io/).

AI‑assisted code is welcome here — just make it production‑grade, not a “vibe‑coded” demo. The day your friends yell when it goes down? That’s when you know your tiny stack is doing real work. Ship it.
