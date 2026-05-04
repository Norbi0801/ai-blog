+++
title = "AGPL vs MIT vs Apache - choosing a license for your open source project"
date = 2025-05-24
description = "A practical breakdown of the three most important open source licenses, when to use each, why Google bans AGPL, and how the open-core model actually works."

[taxonomies]
tags = ["open-source", "licensing", "architecture", "rust"]
+++

Most developers pick a license the same way they pick a `.gitignore` template - click the first thing GitHub suggests, never think about it again. MIT, probably. It was there, it was short, it felt right.

Then three years later the project has 500 stars, a company is running it in production, and someone opens an issue asking about patent rights. Or a cloud provider wraps your project in a managed service and sells it back to your users. Or your employer's legal team sends you an email about "copyleft contamination" in the dependency tree.

Licenses aren't bureaucracy. They're the mechanism that determines who gets to do what with your code. Choose wrong and you either give away more than you intended or scare off the users you wanted to attract.

In my post on [open source etiquette](/blog/open-source-etiquette-how-to-contribute-and-maintain-projects), I covered the basics - MIT, Apache 2.0, GPL, and the Rust ecosystem's dual-licensing convention. This post goes deeper into the three licenses that matter most for modern open source projects and the strategic decisions behind each one.

<!-- more -->

## The three licenses, stripped to their core

Before getting into strategy, let's be precise about what each license actually says. Not the vibes - the legal text.

### MIT: do whatever you want

The MIT license is 171 words. The entire thing fits in a tweet thread. Here's the operative part:

> Permission is hereby granted, free of charge, to any person obtaining a copy of this software and associated documentation files, to deal in the Software without restriction, including without limitation the rights to use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of the Software.

That's it. One condition: keep the copyright notice and the license text in copies. No restrictions on commercial use, modification, distribution, sublicensing, or combining with proprietary code. No mention of patents.

MIT is the most popular license on GitHub. The [2025 OSSRA report from Synopsys](https://www.synopsys.com/software-integrity/resources/analyst-reports/open-source-security-risk-analysis.html) found MIT in the majority of audited open-source codebases. npm defaults to ISC (which is functionally identical to MIT). The Rust ecosystem uses MIT as half of its dual-license convention.

What MIT does not do: it provides zero protection against patent claims. If someone contributes code to your project that's covered by their patent, they can later sue users of that code for patent infringement. The MIT license doesn't address this because it was written in the 1980s, before software patents were a thing.

### Apache 2.0: MIT with patent armor

The Apache License 2.0 is considerably longer - about 4,500 words. Most of it covers the same permissive territory as MIT: use, modify, distribute, sublicense, commercial use, all fine. The critical difference is Section 3:

> Subject to the terms and conditions of this License, each Contributor hereby grants to You a perpetual, worldwide, non-exclusive, no-charge, royalty-free, irrevocable patent license to make, have made, use, offer to sell, sell, import, and otherwise transfer the Work.

Every contributor who submits code to an Apache 2.0 project automatically grants a patent license covering their contribution. If a contributor owns a patent that covers code they've written for your project, users can't be sued for using that patented code. The grant is irrevocable.

There's also a defensive termination clause in Section 3. If a user files a patent lawsuit against the project, their patent license from the project terminates. This is a "patent peace" mechanism - you get patent rights as long as you don't weaponize patents against the project.

This is why corporate-backed projects choose Apache 2.0. Kubernetes, TensorFlow, the Rust compiler, Android (core), Swift - all Apache 2.0. When Google, Microsoft, or Apple employees contribute code, their employer's patents that cover those contributions are licensed to all users. Without this, any corporation contributing code could theoretically sue users later.

For the Rust ecosystem specifically, the convention is dual-licensing under `MIT OR Apache-2.0`. This gives downstream users a choice: MIT's simplicity (no patent clause, but also no patent risk for projects without corporate contributors) or Apache 2.0's patent protection. The Rust compiler, standard library, and most crates on [crates.io](https://crates.io) follow this pattern.

### AGPL 3.0: copyleft that closes the SaaS loophole

The GNU Affero General Public License v3 is a copyleft license. That means if you modify AGPL code and distribute it, you must release your modifications under the AGPL too. But AGPL goes further than regular GPL with one specific clause - Section 13:

> Notwithstanding any other provision of this License, if you modify the Program, your modified version must prominently offer all users interacting with it remotely through a computer network an opportunity to receive the Corresponding Source of your version.

This is the "network use" provision. Regular GPL only triggers its copyleft obligation when you *distribute* the software - ship binaries, publish packages. If you modify GPL code and run it on your server without distributing it, you have no obligation to share your changes. SaaS companies exploited this for years - take GPL code, modify it, run it as a service, share nothing.

AGPL closes that gap. If users interact with your modified AGPL software over a network - through an API, a web interface, a gRPC endpoint - you must make your source code available to them. The copyleft extends from distribution to network access.

## Comparison table

| | MIT | Apache 2.0 | AGPL 3.0 |
|---|---|---|---|
| **Can use commercially** | Yes | Yes | Yes |
| **Can modify** | Yes | Yes | Yes (must share modifications) |
| **Can distribute** | Yes | Yes | Yes (must share source) |
| **Can use in proprietary software** | Yes | Yes | No (derivative work must be AGPL) |
| **Can use as SaaS without sharing source** | Yes | Yes | No |
| **Patent protection** | None | Explicit grant + defensive termination | Same as GPL v3 (grant + termination) |
| **Must include license text** | Yes | Yes | Yes |
| **Must state changes** | No | Yes | Yes |
| **Must disclose source** | No | No | Yes (including network use) |
| **Compatible with GPL v3** | Yes | Yes | Yes (AGPL is GPL-compatible) |
| **Compatible with GPL v2** | Yes | No | No |
| **Typical adopters** | Libraries, small tools | Corporate-backed projects | Projects protecting against SaaS exploitation |

## Why Google bans AGPL

Google maintains one of the most aggressive anti-AGPL policies in the industry. Their [official open source documentation](https://opensource.google/documentation/reference/using/agpl-policy) states it plainly: "Code licensed under the GNU Affero General Public License (AGPL) MUST NOT be used at Google."

The rules go beyond just not using AGPL in products. Google employees cannot install AGPL-licensed programs on their workstations, Google-issued laptops, or Google-issued phones without explicit authorization from the Open Source Programs Office.

The reasoning is structural. Google's core products - Search, Gmail, YouTube, Maps, Cloud - are all network services. The AGPL's Section 13 triggers source code disclosure obligations precisely when software is accessed over a network. Given how deeply interconnected Google's internal codebase is (the infamous monorepo), one AGPL dependency anywhere in the graph could theoretically create obligations to release source code for the service that uses it.

Google's position is that the risks "heavily outweigh the benefits," and the company maintains an "aggressively-broad ban" as a preventive measure. They'd rather lose access to some AGPL-licensed tools entirely than develop nuanced compliance policies for each case.

Google isn't alone. According to [Open Core Ventures](https://www.opencoreventures.com/blog/agpl-license-is-a-non-starter-for-most-companies), many companies implement blanket bans on AGPL rather than invest in compliance frameworks. The logic is straightforward: it's cheaper to ban AGPL than to hire lawyers to evaluate every possible interaction between AGPL code and proprietary systems. Corporate legal teams are risk-averse by design, and AGPL's vague definition of "interacting remotely through a computer network" gives them nightmares.

This fear is partly rational and partly overblown. The AGPL doesn't mean that every line of code touching AGPL software must be open-sourced. It means the modified AGPL program itself must have its source available. If your proprietary application calls an AGPL service over HTTP, your application code doesn't become AGPL - only the AGPL service's code needs to be available. But the boundaries of "derivative work" in copyright law are genuinely fuzzy, and for companies with billions in revenue, "probably fine" isn't good enough.

## The cloud problem and the open-core model

Here's the scenario that changed open source licensing forever: you spend five years building a database. You open-source it under Apache 2.0. Amazon takes your code, offers it as a managed service (AWS SomethingDB), charges customers for it, and contributes nothing back. Your users pay Amazon, not you. Your investors start asking questions.

This isn't hypothetical. It's the story of MongoDB, Elasticsearch, and Redis.

### The timeline

**MongoDB (2018):** Originally AGPL-licensed, MongoDB switched to the Server Side Public License (SSPL) in October 2018. The SSPL requires that anyone offering the software as a service must open-source the entire service stack - not just the database, but the monitoring, backup, orchestration, and management tools around it. The trigger was [AWS launching DocumentDB](https://aws.amazon.com/documentdb/), a MongoDB-compatible managed database. The Open Source Initiative refused to approve SSPL as an open source license. Debian, Red Hat, and Fedora [dropped MongoDB from their repositories](https://www.scylladb.com/2018/10/22/the-dark-side-of-mongodbs-new-license/).

**Elasticsearch (2021 -> 2024):** Elastic changed Elasticsearch and Kibana from Apache 2.0 to dual SSPL/Elastic License in January 2021, directly in response to AWS offering Amazon OpenSearch Service (a fork of Elasticsearch). AWS responded by forking Elasticsearch under the Apache 2.0 license before the switch, creating [OpenSearch](https://opensearch.org/). In September 2024, Elastic added AGPL as a third licensing option, essentially walking back toward an OSI-approved license.

**Redis (2024 -> 2025):** Redis moved from its BSD license to SSPL/RSALv2 in March 2024. The Linux Foundation immediately created [Valkey](https://valkey.io/), a fork based on the last BSD-licensed version. In May 2025, Redis course-corrected and [released Redis 8 under AGPL](https://redis.io/blog/agplv3/), acknowledging that SSPL "hurt our relationship with the Redis community" because it's not truly open source.

### The pattern

There's a repeating cycle here:

1. Company open-sources a database under a permissive license
2. Cloud providers offer it as a managed service
3. Company switches to a restrictive license (usually SSPL)
4. Community forks the permissive version
5. Company realizes SSPL alienated users, considers AGPL

Both Elastic and Redis eventually landed on AGPL as a middle ground. AGPL is copyleft enough to prevent cloud providers from offering the software as a closed-source service without contributing back, but it's still OSI-approved - meaning it's actually open source by the community's accepted definition. SSPL never got that approval.

### How open-core works in practice

The open-core model splits a product into two tiers:

**Core:** The database engine, the query language, the storage layer. Open source (often AGPL or Apache 2.0). Community can use, modify, and contribute.

**Enterprise:** Monitoring dashboards, backup tools, RBAC, SSO integration, clustering, support SLAs. Proprietary. Pay for a license.

MongoDB does this with Community Server (SSPL) vs Enterprise Advanced. GitLab does it with Community Edition (MIT) vs Enterprise Edition (proprietary). Grafana does it with the core (AGPL) and Grafana Enterprise (proprietary).

The AGPL is particularly useful for open-core because it creates a natural boundary: companies that just want to use the software internally can do so freely under AGPL. Companies that want to offer it as a service to their own customers need either a commercial license (which they buy from you) or to open-source their entire service stack (which they won't).

This is also called dual licensing - offering the same code under both AGPL and a commercial license. The copyright holder can do this because they own the code. Users choose: comply with AGPL, or buy a commercial license that removes the copyleft obligations.

## AGPL misconceptions

There's a lot of FUD (Fear, Uncertainty, Doubt) around AGPL. Let's clear up the most common ones.

**"If I use an AGPL library, my entire application becomes AGPL."**

It depends on how you use it. If you link an AGPL library into your application (static or dynamic linking), your application is likely a derivative work and needs to be AGPL. But if your application communicates with an AGPL program through a well-defined interface (HTTP API, Unix socket, CLI invocation), your application is probably not a derivative work. The distinction matters.

Running PostgreSQL (PostgreSQL License, very permissive) doesn't make your app open source. Running an AGPL service that your app talks to over HTTP doesn't either - but the AGPL service itself needs to have its source available to users who interact with it.

**"AGPL means I can't use it commercially."**

Wrong. AGPL explicitly permits commercial use. You can sell AGPL software, offer AGPL software as a service, charge money for support around AGPL software. The only requirement is that users who interact with the modified software over a network can get the source code.

**"AGPL is not real open source."**

The AGPL v3 is approved by the [Open Source Initiative](https://opensource.org/license/agpl-v3) and the [Free Software Foundation](https://www.gnu.org/licenses/agpl-3.0.en.html). It meets all criteria of the Open Source Definition. SSPL is the one that isn't OSI-approved. Don't confuse the two.

**"Nobody uses AGPL."**

[Grafana](https://github.com/grafana/grafana) (62k+ stars), [Nextcloud](https://github.com/nextcloud/server) (29k+ stars), [Mastodon](https://github.com/mastodon/mastodon) (47k+ stars), [Minio](https://github.com/minio/minio) (49k+ stars), and as of 2025, [Redis](https://github.com/redis/redis). AGPL is widely used for infrastructure software where protecting against SaaS exploitation matters.

## Choosing your license: a decision framework

Here's how to actually decide. Answer these questions in order:

### 1. Are you building a library or a standalone application?

**Library** (other code imports and links to it): Copyleft licenses like AGPL make your library difficult to use in proprietary software. If you want maximum adoption, go permissive. MIT for simplicity, Apache 2.0 if patent protection matters.

For the Rust ecosystem, the convention is `MIT OR Apache-2.0`. Use this unless you have a specific reason not to. In your `Cargo.toml`:

```toml
[package]
name = "your-crate"
license = "MIT OR Apache-2.0"
```

And include both license files:

```
your-project/
  LICENSE-MIT
  LICENSE-APACHE
  Cargo.toml
  src/
```

**Application/service** (runs as its own process): Copyleft is viable because users don't link to your code - they run it. AGPL, Apache 2.0, and MIT are all reasonable choices depending on your goals.

### 2. Do you have corporate contributors?

If companies are contributing code to your project, Apache 2.0's patent grant protects everyone. A Google engineer contributing a module that happens to be covered by a Google patent means that patent is licensed to all users of the project. Without Apache 2.0, that patent could theoretically be enforced against users later. This isn't paranoia - [patent trolls](https://www.eff.org/issues/resources-patent-troll-victims) are real, and defensive patent clauses exist for a reason.

### 3. Are you worried about cloud providers offering your software as a service?

If yes, AGPL is your tool. It doesn't prevent cloud providers from offering your software - but it requires them to share any modifications they make. In practice, this means AWS can't take your code, add proprietary improvements, and sell it as a closed-source managed service. They'd have to contribute those improvements back or negotiate a commercial license with you.

If this isn't a concern - maybe your project is a CLI tool, a library, or something that doesn't make sense as a hosted service - AGPL's network clause adds complexity without benefit.

### 4. Do you want to build a business around this?

The open-core model (AGPL core + commercial license for enterprise features) is proven and well-understood. It works because:

- Individual developers and small teams use the AGPL version freely
- Companies that want to embed your software in their proprietary SaaS buy a commercial license
- The AGPL acts as a "forcing function" that pushes commercial users toward your paid tier
- You maintain full copyright, so you can offer the commercial license

If you're a solo developer who just wants people to use your code, MIT or Apache 2.0 removes all friction. If you're building a company, AGPL + commercial dual licensing gives you both community adoption and a revenue path.

### 5. Quick reference

| Scenario | Recommended license |
|---|---|
| Rust crate on crates.io | `MIT OR Apache-2.0` |
| Library in any ecosystem, max adoption | MIT |
| Library with corporate contributors | Apache 2.0 |
| Self-hosted application, community project | AGPL 3.0 or MIT |
| Infrastructure software, SaaS protection | AGPL 3.0 |
| Open-core business model | AGPL 3.0 + commercial |
| Internal tool you're open-sourcing as goodwill | MIT |

## Practical: adding a license to your project

For a Rust project using the ecosystem's dual-license convention:

```bash
# Download the license texts
curl -o LICENSE-MIT https://opensource.org/licenses/MIT
curl -o LICENSE-APACHE https://www.apache.org/licenses/LICENSE-2.0.txt
```

In `Cargo.toml`, use the [SPDX expression](https://spdx.org/licenses/):

```toml
[package]
license = "MIT OR Apache-2.0"
```

For AGPL:

```toml
[package]
license = "AGPL-3.0-only"
```

Or if you want "AGPL v3 or any later version":

```toml
[package]
license = "AGPL-3.0-or-later"
```

Common SPDX identifiers you'll encounter:

| SPDX | License |
|---|---|
| `MIT` | MIT License |
| `Apache-2.0` | Apache License 2.0 |
| `AGPL-3.0-only` | GNU AGPL v3 only |
| `AGPL-3.0-or-later` | GNU AGPL v3 or later |
| `GPL-3.0-only` | GNU GPL v3 only |
| `BSD-2-Clause` | BSD 2-Clause "Simplified" |
| `BSD-3-Clause` | BSD 3-Clause "New" |
| `ISC` | ISC License |
| `MIT OR Apache-2.0` | Dual MIT/Apache (Rust convention) |

One thing to remember, and I mentioned this in the [open source etiquette post](/blog/open-source-etiquette-how-to-contribute-and-maintain-projects): changing a license after the fact requires consent from every contributor. Their code was submitted under the original terms. If you start with MIT and later want to switch to AGPL, you need every contributor to agree - or you rewrite their contributions. This is why MongoDB, Elastic, and Redis could make their license changes: as corporations, they either owned all the code or had Contributor License Agreements (CLAs) that assigned copyright to the company.

If you're starting a project and think you might want to dual-license later, consider requiring a CLA from day one. It's not popular with contributors, but it's the only clean way to retain the ability to relicense.

## The bigger picture

License choice is a product decision, not a legal formality. MIT says "I want maximum adoption and I don't care how people use it." Apache 2.0 says "maximum adoption, but with patent safety nets." AGPL says "use it freely, but if you modify it and serve it to users, share the modifications."

None of these licenses are wrong. They optimize for different things. The mistake isn't choosing the "wrong" license - it's choosing without understanding what you're choosing. A library author who picks AGPL will limit adoption. An infrastructure company that picks MIT will watch cloud providers eat their lunch. A solo developer who picks AGPL for a weekend CLI tool is adding complexity nobody needs.

Read the licenses. Not summaries, not blog posts (including this one) - the [actual text](https://choosealicense.com/licenses/). They're shorter than you'd expect. MIT is 171 words. Apache 2.0 is around 4,500. AGPL 3.0 is the longest, but the key sections (2, 3, and 13) are the ones that matter.

Understand what you're optimizing for. Then choose.
