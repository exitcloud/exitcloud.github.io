---
title: "Self-Hosted Indie Dev: Tools & Secrets"
date: 2026-02-25T04:44:04.055Z
draft: false
tags:
  - "tutorial"
  - "devops"
  - "indie-dev"
---
# **Self-Hosted Indie Dev: AI-Powered Tools & Deployment Secrets**

> "I just saved $10,000/month by ditching AWS and moving my Rails app to a $20 Hetzner VPS. Here's how I did it (and why Kubernetes is a waste of time)."

I see posts like this all the time. And they're not wrong.

Here's the thing—after six years at FAANG watching simple deployment turn into 47-service Rube Goldberg machines, I had to get out. I learned this the hard way: the cloud isn't cheap, and complexity is a tax on your attention. As an indie dev, your attention is the only currency that matters. That's why more devs are seeking **self-hosted** solutions.

Today, I run profitable apps on a handful of cheap boxes. No Kubernetes. No serverless functions that cost a fortune to debug. Just simple, battle-tested tools that let me ship code instead of wrangling YAML.

Let me show you what actually works.

### Why Go **Self-Hosted**? Escaping the Cloud Bill Black Hole

The cloud providers sell you a dream of infinite scale and convenience. What they don't tell you is that you're paying a massive premium for it, and you're getting locked into their ecosystem. Owning your stack isn't just about saving money, it's about control.

Look at 37signals. They're not a tiny indie shop, but they think like one. They just finished moving a staggering **5 petabytes** of data out of S3. Five. Petabytes.

How'd they do it? Not with some managed cloud migration service that costs six figures. They did it with their own tools, on their own hardware. They bought a Pure Storage FlashBlade box—basically a super-fast, on-prem S3-compatible system—and hooked it up to a 100 gigabit network connection dedicated to just copying data. This wasn't a weekend project; it took 90 days, but they pulled it off with zero downtime.

Performance is another big win. When you control the metal, you control the network. After 37signals moved their Docker registry to an on-premise Harbor instance, they cut deployment times by up to 25 seconds. For apps like HEY and Basecamp, that's a huge deal. Twenty-five seconds might not sound like much, but when you're deploying multiple times a day, it adds up. It's the difference between staying in the flow and getting distracted by a slow progress bar. That's a real, tangible improvement to your workflow, just from bringing one piece of infrastructure in-house.

### DIY Docker Registry: Harbor on a Budget VPS

So you're convinced. You want to run your own stuff. Where do you start? A private Docker registry is a great first step. Every time you `docker push`, you're sending your image to Docker Hub, or ECR, or whatever. That can be slow, especially if your server is on the other side of the world from their registry. A **VPS** can offer a cost-effective solution.

Let's spin up our own.

37signals runs Harbor on-prem. Harbor is an open-source container registry that's become the go-to for this stuff. Now, if you look at their docs, you'll see a lot about Kubernetes. Don't get spooked. This is a classic footgun where the "enterprise-ready" path is shoved in your face.

Harbor's **deployment** is optimized for Kubernetes, sure, but it also ships with a pre-packaged `docker-compose` configuration. That's our entry point. You can get a single-server Harbor instance running on a cheap VPS in under an hour. It just works.

Here's the basic play:

1.  Grab a VPS. A simple box with 4 cores and 8GB of RAM is more than enough to start.
2.  Install Docker and Docker Compose.
3.  Download the Harbor offline installer.
4.  Tweak the `harbor.yml` config file. You'll need to set your hostname and generate some certs (Let's Encrypt is your friend here).
5.  Run `./install.sh`.

That's it. You have a private, secure Docker registry.

For storage, you don't need a massive Pure FlashBlade appliance like 37signals uses for its petabytes of data. You can start with the local filesystem on the VPS. When you outgrow that, you can point Harbor to any S3-compatible object storage. Wasabi, Backblaze B2, or even a **self-hosted** MinIO instance on another box. And if you *are* running your own storage hardware, companies like Pure even provide OpenMetrics exporters (`pure-fb-openmetrics-exporter` for FlashBlade) so you can pipe all your storage metrics right into Prometheus.

You get faster deploys and you own the keys to your images. No more surprise bills from your cloud provider because your CI system went haywire and pushed 10,000 image layers in an hour.

### **Kamal**: Deploy Like a Pro (Without Kubernetes Complexity)

Okay, you have a private registry. Now how do you get your app from that registry onto your servers?

This is where everyone tells you to learn Kubernetes. They'll talk about pods, services, ingresses, and a million other abstractions. You'll spend the next six months shaving that particular yak, and you still won't have shipped your feature.

You don't need Kubernetes. You need **Kamal**. 

Kamal is the deploy tool 37signals built and open-sourced. It's what they use to deploy all their apps. It takes a container-based approach but without the ridiculous complexity. You give it a list of servers, and it uses standard Docker commands over SSH to get your app running. Simple. Fast. Effective.

They even use it to run their new synthetic monitoring system, Upright. Upright is a Rails engine they open-sourced that lets you run checks from all over the world to make sure your app is, well, upright. It supports Playwright for full browser testing, simple HTTP pings, and even SMTP and Traceroute probes.

And the best part? They deploy Upright's probe nodes using Kamal to cheap VPS instances around the globe. A minimal setup with two monitoring locations on Hetzner costs under $20 a month. That's a worldwide monitoring system for the price of a few fancy coffees.

This is the ExitCloud philosophy in action. Use simple, powerful tools to solve problems. The big S3 migration? They didn't use some complex data pipeline orchestrator. They used custom Rails tooling, Rclone (a command-line monster for cloud storage), and DuckDB for partitioning the data. Focused tools for a focused job. That's how you move 5 petabytes with no downtime. Not by building a distributed system that requires a team of 10 to maintain.

### Rails Multi-Tenancy: Because Sharing is *Not* Always Caring

If you're building a SaaS app, you're going to hit the multi-tenancy problem. How do you keep one customer's data completely separate from another's? This is a classic footgun. One bad `WHERE` clause and you've got a massive data leak on your hands.

I learned this the hard way. Early on, I messed this up and spent a very stressful weekend emailing customers to apologize. Never again.

The team at 37signals has been doing this for 20 years, so they've got some scar tissue. They solved this problem for themselves and then open-sourced the solution: a gem called `Active Record Tenanted`.

It's a clean way to facilitate multi-tenancy in your Rails app. You define what a "tenant" is in your system (usually an `Account` or `Organization` model), and the gem handles the rest. The most important feature? It includes built-in safety checks to prevent you from accidentally leaking data between tenants. If you try to run a query that isn't scoped to a specific tenant, it will raise an error by default. It forces you to be explicit, which is exactly what you want when dealing with other people's data.

### AI-Assisted Dev: Coding Smarter, Not Harder

Let's talk about AI. It's not magic, and it's not going to take your job. Think of it as a force multiplier. It's like having an infinitely patient junior **AI coding** dev sitting next to you who's really, really good at boilerplate.

But you're still the senior dev. You still have to understand what the code does. You're still responsible for the final product.

AI coding tools are fantastic for getting rid of the tedious stuff. Writing unit tests. Generating database migrations. Translating a blob of JSON into typed objects. Use them for that. Let them accelerate your workflow so you can focus on the hard parts: the business logic, the user experience, the architecture. You know, the stuff that actually matters.

The key is to keep the AI on a leash. You're the one in control. This is why projects are popping up to create a control layer for AI agents that take real actions. You need guardrails.

### Observability for the Indie Dev: Seeing What's Happening (Without Breaking the Bank)

When you're self-hosting, you're responsible for uptime. The best monitoring is often a cron job that curls your health endpoint. But sometimes you need more.

You don't need to pay for Datadog. You can build a powerful observability stack with open-source tools. The 37signals Upright monitoring tool is a perfect example of this. Under the hood, it's a Rails engine that uses a stack any indie dev can manage: SQLite for the database, Solid Queue for background jobs, and a trio of observability powerhouses: Prometheus, AlertManager, and OpenTelemetry.

OpenTelemetry (or OTel) is the new standard for instrumenting your code. You add its libraries to your app, and it gives you traces, metrics, and logs in a standardized format. The ecosystem is moving fast. They just had the first alpha release of eBPF instrumentation, which is a super-efficient way to get data out of the Linux kernel itself. They're also deprecating the old Zipkin exporter spec because Zipkin can now ingest OTLP directly. It's maturing.

Once your app is emitting OTel data, you need somewhere to send it. That's where Prometheus comes in. It's a time-series database that scrapes metrics from your app. You pair it with AlertManager to fire off alerts when things go wrong and Grafana to build pretty dashboards.

You can run this entire stack on a single VPS. You own the data. You define the alerts. You build the dashboards that make sense for *your* application.

It's more work upfront than plugging in a credit card for a managed service. But you'll understand your system on a much deeper level. And when something breaks at 3 AM, that understanding is priceless.

So, that's the blueprint. Ditch the expensive cloud services that lock you in. Embrace simple, powerful tools like Kamal and Harbor. Own your stack. Use AI as a co-pilot, not an autopilot. Build your own observability. It's the **indie dev** way. It's how you build a sustainable, profitable business without a team of SREs and a VC check. You can even start a **homelab** to experiment!

**Self-hosting** isn't just about saving money; it's about gaining control and understanding. It's a path towards true **devops** mastery and **indie dev** independence.

Ship it.

### FAQ

**Q: Is self-hosting really cheaper than using cloud services?**
A: Often, yes, especially for predictable workloads. Cloud services charge a premium for scalability and convenience, which you may not always need.

**Q: What are the biggest challenges of self-hosting?**
A: You're responsible for everything: security, maintenance, and updates. It requires more technical expertise and time investment.

