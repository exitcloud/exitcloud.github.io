---
title: "GitHub Actions for Indie Devs: CI/CD Without the Bloat"
date: 2025-12-18T09:00:00Z
draft: false
tags:
  - "tutorial"
  - "devops"
  - "CI/CD"
---

I used to deploy by SSH-ing into my VPS and running `git pull && docker compose up -d`. It worked. Until I fat-fingered a command, brought the site down for 20 minutes, and realized I'd been running the wrong branch. Classic.

[GitHub Actions](https://github.com/features/actions) fixed this for me, and the free tier is absurdly generous for indie devs: 2,000 minutes/month on Linux runners. I burn maybe 200-300 minutes across all my projects. It's basically free CI/CD.

But here's the thing -- most GitHub Actions tutorials are written for teams with 15 microservices, a staging environment, and a QA department. You don't need that. You need: run tests, build, deploy to your VPS. Done. Let's set that up.

## The Simplest Useful Workflow

This is what I use for a Node.js app that deploys to a VPS via SSH:

```yaml
# .github/workflows/deploy.yml
name: Deploy

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - run: npm ci
      - run: npm test
      - run: npm run build

      - name: Deploy to VPS
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.SSH_HOST }}
          username: ${{ secrets.SSH_USER }}
          key: ${{ secrets.SSH_KEY }}
          script: |
            cd /opt/apps/myapp
            git pull origin main
            npm ci --production
            npm run build
            pm2 restart myapp
```

That's 30 lines. Push to main, tests run, if they pass, it SSH-es into your box and deploys. No Kubernetes. No Terraform. No 800-line YAML. Just ship it.

## Docker Build + Push

If your app runs in [Docker](https://docs.docker.com/compose/) (it probably should), here's the pattern I use most:

```yaml
name: Build and Deploy

on:
  push:
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write

    steps:
      - uses: actions/checkout@v4

      - name: Log in to GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Build and push image
        uses: docker/build-push-action@v6
        with:
          context: .
          push: true
          tags: |
            ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:latest
            ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}

      - name: Deploy to VPS
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.SSH_HOST }}
          username: ${{ secrets.SSH_USER }}
          key: ${{ secrets.SSH_KEY }}
          script: |
            docker pull ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:latest
            cd /opt/apps/myapp
            docker compose up -d --pull always
```

This builds your Docker image on GitHub's runners (free compute!), pushes it to [GitHub Container Registry](https://ghcr.io/) (free for public repos, generous limits for private), and then tells your VPS to pull the new image. Your VPS doesn't need Node.js, Go, Rust, or whatever installed. Just Docker.

## Secrets Management

Don't put passwords in your YAML. Obviously. Go to your repo's Settings > Secrets and variables > Actions, and add:

- `SSH_HOST` -- your VPS IP or domain
- `SSH_USER` -- usually `deploy` or `root` (use a deploy user, please)
- `SSH_KEY` -- your private SSH key (the whole thing, including the `-----BEGIN` lines)

For the SSH key, I create a dedicated deploy key:

```bash
ssh-keygen -t ed25519 -C "github-actions-deploy" -f ~/.ssh/deploy_key
# Add the public key to your VPS:
cat ~/.ssh/deploy_key.pub >> ~/.ssh/authorized_keys
# Paste the private key content into the GitHub secret
```

If your app needs env vars (API keys, database URLs), add those as secrets too and pass them during the build or deploy step:

```yaml
- name: Deploy
  uses: appleboy/ssh-action@v1
  with:
    host: ${{ secrets.SSH_HOST }}
    username: ${{ secrets.SSH_USER }}
    key: ${{ secrets.SSH_KEY }}
    envs: DATABASE_URL,API_KEY
    script: |
      export DATABASE_URL="${DATABASE_URL}"
      export API_KEY="${API_KEY}"
      cd /opt/apps/myapp && docker compose up -d
  env:
    DATABASE_URL: ${{ secrets.DATABASE_URL }}
    API_KEY: ${{ secrets.API_KEY }}
```

## Caching Dependencies (Save Minutes)

Build times add up. Caching cuts them in half or more. The `setup-node` action has built-in caching, which is why I use `cache: 'npm'` above. For other ecosystems:

```yaml
# Python
- uses: actions/setup-python@v5
  with:
    python-version: '3.12'
    cache: 'pip'

# Go
- uses: actions/setup-go@v5
  with:
    go-version: '1.22'
    cache: true

# Rust (use a dedicated cache action)
- uses: Swatinem/rust-cache@v2
```

For Docker builds, layer caching makes a massive difference:

```yaml
- name: Build and push
  uses: docker/build-push-action@v6
  with:
    context: .
    push: true
    tags: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:latest
    cache-from: type=gha
    cache-to: type=gha,mode=max
```

That `cache-from: type=gha` uses GitHub Actions' built-in cache for Docker layers. A build that takes 4 minutes cold might take 30 seconds cached. It's free and there's no reason not to use it.

## Self-Hosted Runners (The Power Move)

Here's something most indie devs don't realize: you can run GitHub Actions runners on your own hardware. Why would you?

1. **Unlimited minutes.** Self-hosted runners don't count against your free tier.
2. **Faster deploys.** The runner is already on your VPS, so there's no SSH step. It just... runs the commands locally.
3. **Access to local resources.** The runner can talk to your database, your Docker daemon, everything.

Setting one up:

```bash
# On your VPS
mkdir actions-runner && cd actions-runner

# Download (check GitHub for latest version)
curl -o actions-runner-linux-x64-2.321.0.tar.gz -L \
  https://github.com/actions/runner/releases/download/v2.321.0/actions-runner-linux-x64-2.321.0.tar.gz
tar xzf actions-runner-linux-x64-2.321.0.tar.gz

# Configure (get the token from your repo's Settings > Actions > Runners)
./config.sh --url https://github.com/yourusername/yourrepo --token YOUR_TOKEN

# Install as a service
sudo ./svc.sh install
sudo ./svc.sh start
```

Then in your workflow:

```yaml
jobs:
  deploy:
    runs-on: self-hosted  # <-- this is the only change
    steps:
      - uses: actions/checkout@v4
      - run: npm ci && npm test && npm run build
      - run: pm2 restart myapp
```

No SSH step needed. The runner IS the VPS. Push, build, deploy. Whole thing takes maybe 30 seconds instead of 2-3 minutes.

**Security note:** only use self-hosted runners on private repos. On public repos, anyone who opens a PR can run arbitrary code on your machine. That's a footgun you don't want.

## Workflows I Actually Use

### Run Tests on PRs

```yaml
name: Tests
on:
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      - run: npm ci
      - run: npm test
```

Keep this separate from deploy. Tests run on every PR. Deploys only happen on main.

### Scheduled Tasks

```yaml
name: Daily Digest
on:
  schedule:
    - cron: '0 8 * * *'  # 8am UTC daily

jobs:
  digest:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: node scripts/send-digest.js
        env:
          API_KEY: ${{ secrets.API_KEY }}
```

Free cron jobs. I use this for daily email digests, database cleanup scripts, and checking if my domains are about to expire. Way easier than setting up cron on the VPS itself.

### Deploy on Release Tag

```yaml
name: Release Deploy
on:
  push:
    tags:
      - 'v*'

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      # ... build and deploy steps
```

For apps where I want more control over when deploys happen. Tag a release, it ships. Good for SaaS apps where you want to batch changes.

## Cost Reality Check

| What | Cost |
|---|---|
| GitHub Actions free tier | 2,000 min/month (Linux) |
| Typical indie usage | 200-400 min/month |
| GitHub Container Registry (public) | Free |
| GitHub Container Registry (private) | 500MB free, then $0.008/GB/day |
| Self-hosted runner | Free (uses your VPS compute) |

I have 6 repos with CI/CD workflows and I've never come close to the free tier limit. If you somehow do, self-hosted runners are the escape hatch.

## Common Footguns

**1. Running on every push to every branch.** Be specific with your triggers. `on: push: branches: [main]` only. Otherwise you're burning minutes on feature branches that don't need CI.

**2. Not caching.** A 4-minute build that could be 30 seconds, running 10 times a day. Do the math. Add caching.

**3. Storing secrets in the workflow file.** I've seen this in public repos. People hardcode API keys in YAML and push to GitHub. Use repository secrets. Always.

**4. Giant Docker images.** Use multi-stage builds. Your final image should be as small as possible. Faster pushes, faster pulls, less bandwidth.

**5. Not pinning action versions.** Use `actions/checkout@v4`, not `actions/checkout@main`. A compromised action at `main` could steal your secrets.

## Cheat Sheet

| Tool/Technique | What It Does | Link |
|---|---|---|
| **GitHub Actions** | CI/CD built into GitHub, 2000 free min/month | [docs.github.com/actions](https://docs.github.com/en/actions) |
| `appleboy/ssh-action@v1` | SSH into your VPS from a workflow step | [github.com/appleboy/ssh-action](https://github.com/appleboy/ssh-action) |
| `docker/build-push-action@v6` | Build + push Docker images to any registry | [github.com/docker/build-push-action](https://github.com/docker/build-push-action) |
| `docker/login-action@v3` | Authenticate to GHCR, Docker Hub, ECR, etc. | [github.com/docker/login-action](https://github.com/docker/login-action) |
| **GHCR** (GitHub Container Registry) | Free image hosting for public repos | [docs.github.com](https://docs.github.com/en/packages) |
| `cache-from: type=gha` | Docker layer caching via GitHub Actions cache | -- |
| **Self-hosted runners** | Run Actions on your own VPS for free unlimited minutes | [docs.github.com](https://docs.github.com/en/actions/hosting-your-own-runners) |
| `actions/setup-node@v4` with `cache: 'npm'` | Auto-cache npm dependencies between runs | -- |
| `Swatinem/rust-cache@v2` | Cache Rust build artifacts | [github.com/Swatinem/rust-cache](https://github.com/Swatinem/rust-cache) |
| **Dedicated deploy SSH key** | `ssh-keygen -t ed25519` -- create a key just for CI deploys | -- |
| **Pin action versions** | Use `@v4` not `@main` to avoid supply chain attacks | -- |
| `on: schedule: cron` | Free scheduled tasks (daily digests, cleanup scripts) | -- |
| **Private repos only for self-hosted runners** | Public repo PRs can run arbitrary code on your box | -- |
