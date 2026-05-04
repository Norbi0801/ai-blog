+++
title = "Hosting for indie devs - Hetzner vs Fly.io vs Railway vs Vercel"
date = 2025-09-24
description = "A cost and DX breakdown of four hosting platforms at 100, 1k, and 10k users - with real pricing, free tiers, and vendor lock-in analysis."

[taxonomies]
tags = ["devops", "architecture", "tooling"]
+++

You shipped your side project. It works on localhost. Now comes the question that has derailed more indie projects than scope creep: where do I host this thing?

The answer used to be simple. You'd rent a DigitalOcean droplet, set up nginx, and call it a day. But in 2026, the landscape has fractured into distinct philosophies. You've got traditional VPS providers, edge compute platforms, PaaS with git-push deploys, and serverless-first hosts that don't even give you a server. Each one optimizes for a different part of the developer experience, and each one will cost you very different amounts depending on what you're building and how many people use it.

I've run workloads on all four platforms I'm covering here - Hetzner, Fly.io, Railway, and Vercel. This isn't a theoretical comparison. I'll show you what each one actually costs at different scales, where the hidden charges lurk, and when you should pick one over the others.

<!-- more -->

## The four contenders

Before we get into numbers, here's the mental model:

- **Hetzner** - You get a Linux box. That's it. You install things, you configure things, you own everything. Cheapest raw compute on the market.
- **Fly.io** - Your app runs in micro-VMs on edge locations worldwide. Per-second billing. You get a real server, not a function - but deployment is managed.
- **Railway** - Git push, app deploys. Container-based PaaS with a usage-based pricing model. The modern Heroku.
- **Vercel** - Serverless functions + global CDN. Built around Next.js and the frontend ecosystem. No servers to think about.

These aren't interchangeable. Picking the right one depends on what kind of app you're building, how much ops work you're willing to do, and what your monthly budget looks like.

## Hetzner - the price-performance king

[Hetzner](https://www.hetzner.com/cloud/) is a German hosting company that has been around since 1997. Their cloud VPS lineup is the cheapest serious hosting you'll find anywhere, and "serious" is the key word here - this isn't some bargain-bin reseller. Hetzner data centers are in Germany, Finland, Singapore, and the US (Ashburn and Hillsboro).

### What you get

The CX (cost-optimized) line starts at the CX22: 2 shared vCPUs, 4 GB RAM, 40 GB NVMe, 20 TB of traffic, for **EUR 3.79/month** (roughly $4.10 USD). That's not a typo. Four dollars a month for a usable server with 20 terabytes of included bandwidth.

The lineup scales predictably:

| Plan  | vCPU | RAM   | Disk   | Traffic | Price/mo |
|-------|------|-------|--------|---------|----------|
| CX22  | 2    | 4 GB  | 40 GB  | 20 TB   | ~$4.10   |
| CX32  | 4    | 8 GB  | 80 GB  | 20 TB   | ~$7.40   |
| CX42  | 8    | 16 GB | 160 GB | 20 TB   | ~$17.80  |
| CX52  | 16   | 32 GB | 320 GB | 20 TB   | ~$35.20  |

If you want ARM (Ampere Altra), the CAX line has similar pricing and is available in the EU regions. For dedicated vCPUs, the CCX line starts around $15/month for 2 cores.

All plans use hourly billing with a monthly cap - you never pay more than the listed monthly price, but you can delete a server mid-month and only pay for the hours used.

### The catch

Hetzner gives you a blank Linux server. There's no deployment pipeline, no git integration, no managed database, no automatic TLS certificates. You set up everything yourself. That means:

- Installing your runtime (Docker, your language's toolchain, whatever)
- Configuring a reverse proxy (nginx, Caddy, Traefik)
- Setting up TLS (Caddy does this automatically, or use certbot)
- Configuring firewall rules
- Setting up deployment (rsync scripts, Docker Compose, or a tool like [Coolify](https://coolify.io/) or [Kamal](https://kamal-deploy.org/))
- Managing backups, updates, and monitoring

If you read my post on [monitoring Rust applications in production](/blog/monitoring-rust-applications-in-production/), you know this stuff isn't optional. It's work. Real, ongoing, unglamorous work.

The payoff is complete control and the lowest cost. Nobody's going to surprise you with a $300 bill because your function invocations spiked. Your server costs $4.10/month whether it serves 10 requests or 10 million.

### Who this is for

Solo devs who are comfortable with Linux and want to spend $4-15/month instead of $40-200/month. Backend-heavy apps (APIs, databases, background workers) where you need predictable costs. SaaS products past the early validation phase where the savings justify the ops overhead.

## Fly.io - edge compute with real servers

[Fly.io](https://fly.io/) takes a different approach. You write a Dockerfile (or use a buildpack), run `fly deploy`, and your app starts running on micro-VMs powered by [Firecracker](https://firecracker-microvm.github.io/) - the same virtualization technology AWS Lambda uses under the hood. The difference from Lambda is that your VM stays running. It's a real process, not a function invocation.

### What you get

Fly.io runs your app in 30+ regions worldwide. You can deploy to one region or many. The pricing is per-second while machines are running:

| Machine Type        | vCPU     | RAM   | Price/hr | ~Price/mo (24/7) |
|---------------------|----------|-------|----------|-------------------|
| shared-cpu-1x       | 1 shared | 256MB | $0.0028  | $2.02             |
| shared-cpu-2x       | 2 shared | 512MB | $0.0056  | $4.04             |
| shared-cpu-4x       | 4 shared | 1 GB  | $0.0112  | $8.08             |
| performance-1x      | 1 dedicated | 2 GB  | $0.0447  | $32.19         |
| performance-2x      | 2 dedicated | 4 GB  | $0.0894  | $64.39         |

Additional RAM costs about $5/month per GB. Volumes (persistent storage) are $0.15/GB/month. Egress is $0.02/GB in North America and Europe.

One thing people miss: **dedicated IPv4 addresses cost $2/month per app**. If you're running three small apps, that's $6/month just for IPs. Shared IPv4 (via Fly Proxy) is free, but dedicated addresses add up.

### The catch

Fly.io killed its free tier for new customers in 2024. New signups get a trial of 2 VM hours or 7 days, whichever comes first. After that, you need a credit card and you pay for everything.

The billing model can also surprise you. Volumes are billed even when machines are stopped. If you provision a 10 GB volume and forget about it, that's $1.50/month doing nothing. Stopped machines with custom root filesystems also incur storage charges at $0.15/GB/month.

The developer experience is good but has friction. `flyctl` is powerful but the config (fly.toml) has a learning curve. Networking between machines, volume attachments, and multi-region setups require reading the docs carefully. It's easier than raw Hetzner, but it's not "git push and done."

### Where Fly.io shines

Fly.io is the best option when you need your app in multiple geographic regions without managing multiple VPS instances yourself. If you're building a real-time app (WebSockets, multiplayer games, collaboration tools) and cold starts are unacceptable, Fly.io gives you persistent processes at edge locations.

The scale-to-zero feature is also genuinely useful for side projects - machines can stop when idle and start on the first request. For a low-traffic side project, you might pay less than $1/month if the machine only runs a few hours a day.

## Railway - the modern Heroku

[Railway](https://railway.com/) is the platform I'd recommend to someone who just wants to deploy something without thinking about infrastructure. Connect your GitHub repo, pick a branch, and Railway builds and deploys your app. Environment variables, databases, cron jobs, logs - all in a clean web UI.

### What you get

Railway uses a hybrid model: a base subscription that includes usage credits, plus pay-as-you-go for anything beyond the credits.

| Plan       | Base Cost | Included Credits | vCPU Limit  | RAM Limit |
|------------|-----------|------------------|-------------|-----------|
| Trial      | Free      | $5 one-time      | -           | -         |
| Hobby      | $5/mo     | $5/mo            | 48 vCPU     | 48 GB     |
| Pro        | $20/mo    | $20/mo           | 1,000 vCPU  | 1 TB      |
| Enterprise | Custom    | Custom           | 2,400 vCPU  | 2.4 TB    |

The usage rates:
- **vCPU**: $0.000463/vCPU-minute (~$20/vCPU-month)
- **Memory**: $0.000232/GB-minute (~$10/GB-month)
- **Storage**: $0.0000036/GB-minute (~$0.16/GB-month)
- **Egress**: $0.05/GB

A small app running on 0.5 vCPU and 512 MB RAM 24/7 costs roughly **$10-15/month** in raw resources. With the $5 Hobby credit, you'd pay around $5-10/month total.

### The DX advantage

This is where Railway wins, and it wins hard. Your deployment workflow looks like:

```bash
git push origin main
# That's it. Railway builds, deploys, runs health checks.
```

Need a PostgreSQL database? Click "New Service", pick Postgres, done. It's provisioned in seconds, the connection string is auto-injected into your app's environment variables. Same for Redis, MySQL, MongoDB.

Need to see what's happening? `railway logs` or open the dashboard. Need to SSH into the container? `railway shell`. Need a preview environment for a PR? Railway creates them automatically.

I covered [SQLite as a database choice](/blog/why-sqlite-with-wal-mode-is-good-enough-for-most-web-apps/) before, and if you're running on Railway, SQLite with WAL mode on a volume is a perfectly valid option for early-stage apps. But Railway also makes managed Postgres trivial, so you have options.

### The catch

The trial is a one-time $5 credit that expires after 30 days. Not a recurring free tier - a trial. After that, the Hobby plan's $5/month base fee kicks in whether you run anything or not.

Hobby also limits you to single-developer workspaces. If you're working with someone, you need Pro at $20/month.

The biggest hidden cost is that Railway's per-vCPU and per-GB prices are significantly higher than a raw VPS. You're paying for the convenience. A CX22 on Hetzner gives you 2 vCPU + 4 GB RAM for ~$4.10/month. The equivalent compute on Railway would cost roughly $40/month. That's a 10x premium for the managed experience.

## Vercel - serverless for the frontend

[Vercel](https://vercel.com/) is a different animal entirely. It's not a general-purpose hosting platform - it's a deployment and edge platform optimized for frontend frameworks, especially Next.js (which Vercel created). Your static assets go on a global CDN. Your API routes become serverless functions. There's no server to manage because there's no server.

### What you get

| Plan       | Cost          | Bandwidth      | Function Invocations | Edge Requests |
|------------|---------------|----------------|----------------------|---------------|
| Hobby      | Free          | 100 GB/mo      | 1M/mo               | 1M/mo         |
| Pro        | $20/user/mo   | 1 TB/mo        | Included + overage   | 10M/mo        |
| Enterprise | Custom        | Custom         | Custom               | Custom         |

The Hobby tier is generous for personal projects: 100 GB bandwidth, 1 million function invocations, and 4 hours of active CPU time per month. That handles a personal blog or portfolio comfortably.

Pro's usage-based charges:
- **Active CPU**: $0.128/hour
- **Memory**: $0.0106/GB-hour
- **Invocations**: $0.60/million
- **Bandwidth overage**: $0.15/GB
- **Additional team members**: $20/user/month

### The catch - and it's a big one

Vercel's **Hobby plan is non-commercial**. From their terms of service: the free tier is for personal, non-commercial use only. If you're building a SaaS product, even a tiny one, you need Pro at $20/month minimum. And that's per user - add a co-founder and you're at $40/month before any usage charges.

The per-seat pricing is the thing that catches indie devs off guard. Most other platforms charge per resource. Vercel charges per human.

Then there's the bill shock problem. Vercel's pricing page shows clean tiers, but the actual bill depends on usage patterns that are hard to predict. A function that runs for 2 seconds per invocation at 1000 invocations/day is 2000 seconds/day = ~16.7 hours/month of active CPU = $2.13/month. Sounds fine. But if that same function starts getting bot traffic and jumps to 50,000 invocations/day, you're at 833 CPU-hours/month = $106.67/month just for that one endpoint. On a VPS, you'd pay the same $4 regardless.

There's a well-known [post](https://deploywise.dev/blog/vercel-pricing-explained) about someone whose $20/month Pro plan actually cost $286 in a single month after accounting for overages.

### Vendor lock-in

This is where I need to be blunt. Vercel has the highest vendor lock-in of all four platforms.

If you use Vercel-specific features - ISR (Incremental Static Regeneration), image optimization, edge middleware, `next/image` with Vercel's loader - you're building on proprietary infrastructure. Migrating a mature Next.js app off Vercel to, say, a Docker container on Railway requires rewriting or replacing every Vercel-specific integration.

Vercel argues they're "anti-lock-in" because Next.js is open source and projects like [OpenNext](https://opennext.js.org/) let you self-host. That's true in theory. In practice, ISR behaves differently on every platform, and edge functions don't have a universal standard yet. Moving off Vercel is a project, not a config change.

Railway, Fly.io, and Hetzner all run standard containers or VMs. If you package your app as a Docker image, you can move between them (and to AWS, GCP, or any other provider) by changing a deployment config. Your app doesn't know or care where it runs.

## The real costs - 100, 1,000, and 10,000 users

Theory is nice. Let's model actual costs for a typical indie SaaS: a web app with a backend API, a database, authentication, and some background jobs. I'm assuming a standard web app - not video streaming, not ML inference, just a normal CRUD SaaS.

### Assumptions

- Each active user generates ~50 API requests/day
- Average response size: 5 KB
- Database: PostgreSQL (except Hetzner where you'd self-host)
- The app needs 24/7 uptime (no scale-to-zero)
- Prices in USD, monthly

### 100 users (early stage, pre-revenue)

| Platform | Compute | Database | Bandwidth | Other | Total |
|----------|---------|----------|-----------|-------|-------|
| Hetzner CX22 | $4.10 | included (self-hosted) | included (20 TB) | - | **~$4** |
| Fly.io shared-cpu-2x | $4.04 | $0 (SQLite on volume) | ~$0.10 | IPv4: $2 | **~$7** |
| Railway Hobby | $5 base | Postgres add-on ~$3 | ~$0.05 | - | **~$8** |
| Vercel Pro | $20/seat | Vercel Postgres ~$5 | included | - | **~$25** |

At 100 users, Hetzner is absurdly cheap. You're paying less for a month of hosting than a single coffee. Fly.io and Railway are reasonable. Vercel's per-seat minimum makes it the most expensive option for what is essentially zero load.

### 1,000 users (growing, some revenue)

Now the app actually needs resources. 50,000 requests/day = ~35 requests/minute average, with peaks of maybe 100-200/min. You need 2-4 vCPUs and 4-8 GB RAM to handle this comfortably with a database.

| Platform | Compute | Database | Bandwidth | Other | Total |
|----------|---------|----------|-----------|-------|-------|
| Hetzner CX32 | $7.40 | included | included | - | **~$7** |
| Fly.io shared-cpu-4x + 2GB extra RAM | $8.08 + $10 | Fly Postgres ~$15 | ~$1 | IPv4: $4 | **~$38** |
| Railway Pro | $20 base | Postgres ~$10 | ~$0.50 | overages ~$15 | **~$45** |
| Vercel Pro (2 seats) | $40 | Vercel Postgres ~$20 | ~$5 | CPU overages ~$15 | **~$80** |

At 1,000 users, the gap widens significantly. Hetzner is still under $10/month because compute and bandwidth are bundled. The managed platforms cost 5-10x more. The question is whether that 5-10x buys you enough saved time to be worth it.

### 10,000 users (real traction)

Now you need proper infrastructure. 500,000 requests/day = ~350/minute average, peaks of 1,000+/min. You want 4-8 vCPUs, 16+ GB RAM, and a production database setup.

| Platform | Compute | Database | Bandwidth | Other | Total |
|----------|---------|----------|-----------|-------|-------|
| Hetzner CX42 | $17.80 | included | included | LB: $6 | **~$24** |
| Fly.io perf-2x + extra RAM | $64.39 + $20 | Fly Postgres ~$65 | ~$10 | IPv4: $4 | **~$163** |
| Railway Pro | $20 base | Postgres ~$40 | ~$5 | overages ~$80 | **~$145** |
| Vercel Pro (2 seats) | $40 | External DB ~$50 | ~$30 | CPU overages ~$100 | **~$220** |

At 10,000 users, Hetzner is costing you roughly $24/month. Vercel is approaching $220/month. That's a 9x difference. Over a year, that's $2,640 vs $288 - a delta of $2,352. For an indie developer, that's real money.

The Hetzner numbers assume you're doing your own ops. If you value your time at $50/hour and spend 2 extra hours per month on server maintenance, that adds $100 in effective cost. Even then, Hetzner at $124 effective cost is still cheaper than every managed option.

## Free tiers - the honest breakdown

| Platform | Free Tier | Limits | Commercial Use |
|----------|-----------|--------|----------------|
| Hetzner | None | - | N/A |
| Fly.io | Trial only (2 VM-hours or 7 days) | Very limited | N/A after trial |
| Railway | $5 one-time credit, 30 days | Expires | Yes, during trial |
| Vercel | Hobby plan (forever free) | 100 GB BW, 4 CPU-hrs, 1M invocations | **No** - personal only |

The only real free tier is Vercel's Hobby plan, and you can't use it for anything commercial. If you're building a SaaS, none of these platforms are free. The cheapest path to production is Hetzner at ~$4/month.

Railway's "free trial" is misleading if you're comparing it to Heroku's old free tier (which also no longer exists). It's $5 of credits that expire in 30 days. Think of it as a test drive, not a free tier.

## Vendor lock-in spectrum

From least to most lock-in:

**Hetzner (minimal)** - It's a Linux box. Your Docker Compose file, your systemd services, your nginx config - all of it works on any VPS provider, any cloud, any bare metal server. Moving to a different provider means copying files and updating DNS. You could do it in an afternoon.

**Railway (low)** - Railway runs Docker containers. If you have a Dockerfile (and you should), your app can run on any container platform. Railway-specific features are mostly in the deployment pipeline, not in your application code. The main friction in migrating is recreating managed databases and environment variable injection.

**Fly.io (low-medium)** - Fly.io also runs containers, but you might be using Fly-specific features like multi-region Postgres, Fly Proxy headers, internal DNS (`*.internal`), or the Machines API for orchestration. The more you lean on Fly's networking and orchestration, the more work it takes to leave. But at the application level, it's still just a Docker container.

**Vercel (high)** - If you're using plain Next.js with basic API routes, you can move. If you're using ISR, edge middleware, `next/image` optimization, Vercel KV, Vercel Postgres, Vercel Blob, Vercel Analytics, or any of the Vercel-specific integrations - each one is a migration task. The more of Vercel's ecosystem you adopt, the more expensive (in engineering time) it becomes to leave.

## The decision framework

Here's how I'd think about it:

**Choose Hetzner if:**
- You're comfortable with Linux administration
- You want the absolute lowest cost
- Your app is backend-heavy (API, database, workers)
- You're past early validation and want predictable, flat costs
- You want zero vendor lock-in

**Choose Fly.io if:**
- You need multi-region deployment without managing multiple servers
- Your app uses WebSockets or real-time features where cold starts are unacceptable
- You want scale-to-zero for low-traffic side projects
- You're comfortable with CLIs and config files but don't want to manage a full server

**Choose Railway if:**
- You want the fastest path from code to production
- You're in the early stages and speed of iteration matters more than cost optimization
- You need managed databases without the ops overhead
- Your team is small and you value DX above everything else

**Choose Vercel if:**
- You're building a Next.js frontend or a Jamstack site
- Your backend is an external API (separate service, not hosted on Vercel)
- You want the best possible frontend delivery performance (edge CDN, image optimization)
- You're fine with the per-seat pricing and have budget for it

### The "start here, move there" pattern

A pattern I see working well for indie devs:

1. **Validate on Railway** - Ship fast, iterate fast, don't think about infrastructure. The $5-20/month is worth it when you're figuring out if anyone wants your product.

2. **Scale on Fly.io** - When you have real users and need multi-region or more control, Fly.io gives you that without going full VPS.

3. **Optimize on Hetzner** - When your product is stable, your architecture is settled, and the managed platform tax is eating into margins, move to Hetzner. Set up Coolify or Kamal for deployments, and enjoy 80-90% cost reduction.

Not everyone needs step 3. If your SaaS makes $10k/month and Railway costs you $45/month, the $38 you'd save by moving to Hetzner isn't worth the ops overhead. But if you're running 5 services across multiple projects, the savings compound.

## What I'd actually do

If I were starting a new SaaS today, I'd put the frontend on Vercel's free tier (it's a static Next.js export, no serverless functions, no lock-in features) and the backend on a Hetzner CX22 with Caddy as the reverse proxy. Total cost: $4.10/month. Caddy handles TLS automatically, Docker Compose handles service orchestration, and a simple GitHub Actions workflow handles deployment.

If you told me "I don't want to touch servers ever," I'd say Railway on the Pro plan. $20/month with credits, managed Postgres, zero config deploys, and you can focus entirely on your product.

The worst choice is picking a platform because it's trendy. Pick the one that matches your skills, your budget, and your app's actual requirements. The best hosting platform is the one that gets out of your way so you can build.
