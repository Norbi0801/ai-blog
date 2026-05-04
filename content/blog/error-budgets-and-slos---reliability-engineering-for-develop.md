+++
title = "Error budgets and SLOs - reliability engineering for developers"
date = 2025-09-02
description = "How to define SLOs for your API, measure error budgets with Prometheus, alert on burn rate, and why targeting 100% availability will slow your team down."

[taxonomies]
tags = ["reliability", "devops", "observability", "architecture"]
+++

Your API went down for 47 seconds last Tuesday. A deploy rolled out a bad database migration, the health check caught it, Kubernetes rolled back. Total impact: a few hundred failed requests. Nobody even noticed.

But your VP of Engineering noticed. "We need 100% uptime." And suddenly half your sprint is spent hardening infrastructure instead of building features.

Here's the problem: 100% availability is not a goal. It's a trap. Google figured this out two decades ago and formalized it into a system that every engineering team should understand, not just SREs. The core idea is simple - decide exactly how much failure is acceptable, measure it, and use that number to make deployment decisions. They call it error budgets.

<!-- more -->

## SLIs, SLOs, and SLAs - three different things

These acronyms get thrown around interchangeably. They shouldn't be.

**SLI (Service Level Indicator)** is the measurement. A concrete metric that tells you how the service is performing from the user's perspective. Examples:

- Proportion of HTTP requests returning a successful response (not 5xx)
- Proportion of requests completing within 200ms
- Proportion of data processing jobs finishing within their deadline

An SLI is always a ratio: good events divided by total events. This is important. Not "average latency" (which hides outliers), not "uptime percentage" (which doesn't reflect user experience), but a ratio between 0 and 1.

**SLO (Service Level Objective)** is the target. You pick an SLI and say "this should be above X% over a rolling Y-day window." For example:

- 99.9% of HTTP requests succeed over a 30-day window
- 99.5% of requests complete within 200ms over a 30-day window

The SLO defines what "good enough" looks like. Not perfect. Good enough.

**SLA (Service Level Agreement)** is the contract. It's a business document with financial consequences - refunds, credits, contract termination. SLAs are always less strict than SLOs. If your SLO is 99.9%, your SLA might be 99.5%, giving you a buffer before you start owing customers money.

Developers should care about SLOs. SLAs are for legal and sales teams.

## Why 100% is the wrong number

Let's do the math on what different availability targets actually mean:

| Target | Allowed downtime/month | Allowed downtime/year |
|--------|----------------------|---------------------|
| 99% (two nines) | 7.2 hours | 3.65 days |
| 99.5% | 3.6 hours | 1.83 days |
| 99.9% (three nines) | 43.2 minutes | 8.76 hours |
| 99.95% | 21.6 minutes | 4.38 hours |
| 99.99% (four nines) | 4.32 minutes | 52.6 minutes |
| 99.999% (five nines) | 25.9 seconds | 5.26 minutes |

Look at that jump from 99.9% to 99.99%. You go from 43 minutes of allowed downtime per month to 4 minutes. That single extra nine means you can't do a rolling deploy that takes longer than 4 minutes total across the entire month. Can't do database migrations that cause any interruption. Can't restart your service during a traffic spike without it counting against you.

And 100%? Zero seconds of allowed downtime. Ever. No deploys, no maintenance, no experiments. Your network must never drop a packet, your disks must never fail, your cloud provider must never have an incident. It's physically impossible for any distributed system.

Google's Site Reliability Engineering book puts it directly: *"100% is the wrong reliability target for basically everything."* Your users don't need 100%. Their ISP doesn't give them 100%. Their browser crashes. Their WiFi drops. A service that's 99.99% available is indistinguishable from 100% for almost all users - their own infrastructure fails more often than yours.

The real question is: what's the lowest reliability your users will tolerate? Start there.

## The error budget

This is where it gets interesting. Once you have an SLO, you can derive the error budget:

```
error_budget = 1 - SLO_target
```

For a 99.9% SLO, your error budget is 0.1%. Over a 30-day window, that means 0.1% of your requests are allowed to fail. If you serve 10 million requests per month, you can afford 10,000 failures before you're in trouble.

The error budget reframes reliability from a vague aspiration ("be more reliable") into a concrete, spendable resource. You're not trying to prevent all failures. You're managing a budget.

And like any budget, you can spend it. A risky deploy that might cause a few errors? That's fine - you have budget for it. A major refactor that requires a brief service disruption? Check your remaining budget. A/B testing a new feature that increases latency for 0.01% of users? Go ahead, you can afford it.

The math for how much budget an incident consumes:

```
budget_consumed = (bad_requests_during_incident / total_requests_in_window) / error_budget
```

Say your service handles 10 million requests over a 30-day window with a 99.9% SLO. That gives you an error budget of 10,000 bad requests. An incident causes 2,000 errors. You've consumed 20% of your monthly error budget. You still have 80% left to spend on deploys, experiments, and the occasional hiccup.

## Defining SLOs for your API - a practical walkthrough

Enough theory. Here's how to actually define SLOs for a Rust API.

### Step 1: Pick your SLIs

Start with two SLIs. Not five, not ten. Two:

**Availability SLI** - the proportion of non-5xx responses:

```
availability = successful_requests / total_requests
```

Where "successful" means any response that isn't a server error (5xx). Note that 4xx responses count as successful from an SLI perspective - a 404 means the server correctly told the user the resource doesn't exist. The server did its job.

**Latency SLI** - the proportion of requests faster than a threshold:

```
latency = requests_faster_than_threshold / total_requests
```

The threshold should be your p99 from normal operation. If your [load testing](/blog/load-testing-your-rust-api---tools-and-methodology/) shows a p99 of 180ms, set the latency threshold at 200ms.

### Step 2: Set targets based on real data

Don't pick numbers out of thin air. Look at your actual performance over the last 30-90 days. If your API currently succeeds 99.95% of the time, setting a 99.99% SLO means you're already in violation. Setting 99.9% gives you headroom.

Some guidelines by service type:

- **User-facing API** (web/mobile frontend): 99.9% availability, 99.5% latency under 200ms
- **Internal API** (service-to-service): 99.95% availability, 99.9% latency under 100ms
- **Batch processing / async jobs**: 99.9% completion rate, 99% within deadline
- **Data pipeline**: 99.5% freshness (data arrives within SLA window)

These are starting points. Adjust based on what your actual users notice. If nobody complains until availability drops below 99.5%, maybe 99.9% is already overengineering.

### Step 3: Choose your window

The standard is a **30-day rolling window**. Not calendar months (which have different lengths and create end-of-month cliff effects), not weekly (too volatile), not quarterly (too slow to react). A rolling 30-day window means every day you drop the oldest day and add the newest.

Some teams use shorter windows for fast-moving services. A 7-day rolling window for an API that deploys 10 times a day catches regressions faster but is also more volatile - a single bad hour has a much bigger proportional impact.

### Step 4: Write it down

An SLO isn't real until it's documented and agreed upon. Here's a minimal SLO document:

```yaml
service: order-api
owner: backend-team
slos:
  - name: availability
    description: "Proportion of non-5xx responses"
    target: 99.9%
    window: 30 days (rolling)
    measurement: |
      sum(rate(http_requests_total{job="order-api",status!~"5.."}[30d]))
      / sum(rate(http_requests_total{job="order-api"}[30d]))

  - name: latency
    description: "Proportion of requests completing within 200ms"
    target: 99.5%
    window: 30 days (rolling)
    measurement: |
      sum(rate(http_request_duration_seconds_bucket{job="order-api",le="0.2"}[30d]))
      / sum(rate(http_request_duration_seconds_count{job="order-api"}[30d]))
```

## Measuring SLIs with Prometheus

If you've read the [monitoring post](/blog/monitoring-rust-applications-in-production), you already have `http_requests_total` and `http_request_duration_seconds` exposed from your Rust service. Those two metrics are enough to compute both SLIs.

The availability SLI in PromQL:

```promql
sum(rate(http_requests_total{job="order-api", status!~"5.."}[30d]))
/
sum(rate(http_requests_total{job="order-api"}[30d]))
```

The latency SLI (using histogram buckets):

```promql
sum(rate(http_request_duration_seconds_bucket{job="order-api", le="0.2"}[30d]))
/
sum(rate(http_request_duration_seconds_count{job="order-api"}[30d]))
```

The `le="0.2"` selects the histogram bucket for requests completing in 200ms or less. This is why you want histograms, not summaries - histograms let you query arbitrary thresholds after the fact.

### Recording rules for multiple time windows

You don't want to compute these over 30 days on every dashboard refresh. Prometheus recording rules pre-compute them at shorter intervals so you can compose them efficiently. You'll need error ratios at multiple window sizes for burn rate alerting (more on that below).

```yaml
groups:
  - name: slo-availability-recording-rules
    rules:
      - record: job:slo_errors_per_request:ratio_rate5m
        expr: |
          sum(rate(http_requests_total{status=~"5.."}[5m])) by (job)
          /
          sum(rate(http_requests_total[5m])) by (job)

      - record: job:slo_errors_per_request:ratio_rate30m
        expr: |
          sum(rate(http_requests_total{status=~"5.."}[30m])) by (job)
          /
          sum(rate(http_requests_total[30m])) by (job)

      - record: job:slo_errors_per_request:ratio_rate1h
        expr: |
          sum(rate(http_requests_total{status=~"5.."}[1h])) by (job)
          /
          sum(rate(http_requests_total[1h])) by (job)

      - record: job:slo_errors_per_request:ratio_rate6h
        expr: |
          sum(rate(http_requests_total{status=~"5.."}[6h])) by (job)
          /
          sum(rate(http_requests_total[6h])) by (job)

      - record: job:slo_errors_per_request:ratio_rate1d
        expr: |
          sum(rate(http_requests_total{status=~"5.."}[1d])) by (job)
          /
          sum(rate(http_requests_total[1d])) by (job)

      - record: job:slo_errors_per_request:ratio_rate3d
        expr: |
          sum(rate(http_requests_total{status=~"5.."}[3d])) by (job)
          /
          sum(rate(http_requests_total[3d])) by (job)
```

The naming convention follows Prometheus best practices: `level:metric:operations`. Each rule gives you the error ratio (proportion of 5xx responses) over a specific time window, grouped by job.

### Dashboard query for remaining error budget

To visualize how much budget you've consumed over the current 30-day window:

```promql
1 - (
  avg_over_time(job:slo_errors_per_request:ratio_rate5m{job="order-api"}[30d])
  / (1 - 0.999)
)
```

This returns a number between 0 and 1. At 1.0, you've consumed none of your budget. At 0, it's gone. Below 0, you're in SLO violation.

A Grafana panel showing this as a percentage with thresholds at 25%, 50%, and 75% consumed gives your team a clear visual: green, yellow, orange, red.

## Burn rate alerting

Here's where most teams get SLO monitoring wrong. The naive approach is to alert when the SLI drops below the target:

```yaml
# DON'T DO THIS
- alert: SLOViolation
  expr: |
    sum(rate(http_requests_total{job="order-api",status=~"5.."}[30d]))
    / sum(rate(http_requests_total{job="order-api"}[30d]))
    > 0.001
```

This fires after you've already burned your entire budget. That's useless - you need to know while the incident is happening, not after the damage is done.

The other naive approach is to alert on raw error rate:

```yaml
# ALSO DON'T DO THIS
- alert: HighErrorRate
  expr: |
    sum(rate(http_requests_total{job="order-api",status=~"5.."}[5m]))
    / sum(rate(http_requests_total{job="order-api"}[5m]))
    > 0.001
  for: 5m
```

This fires on every brief spike, even if the spike is well within your error budget. You'll get paged for a 2-minute blip that consumed 0.01% of your budget. Alert fatigue sets in fast.

The solution comes from Google's SRE Workbook: **multi-window, multi-burn-rate alerting**.

### What's a burn rate?

Burn rate measures how fast you're consuming your error budget relative to the SLO window. A burn rate of 1 means you'll exactly exhaust your budget at the end of the 30-day window. A burn rate of 2 means you'll burn through it in 15 days. A burn rate of 14.4 means you'll burn through it in about 2 days.

```
burn_rate = observed_error_rate / (1 - SLO_target)
```

For a 99.9% SLO, if your current error rate is 1.44%, your burn rate is:

```
1.44% / 0.1% = 14.4
```

### Why two windows?

A single burn rate threshold has a problem. If you alert when the 1-hour burn rate exceeds 14.4, you'll catch active incidents. But you'll also get alerted about an incident that lasted 10 minutes, was already resolved 50 minutes ago, but still shows in the 1-hour window. You get paged for something that's already over.

The fix: require both a long window and a short window to be in violation. The long window ensures enough budget has been consumed to matter. The short window confirms the problem is still actively happening.

Google's recommended parameters for a 30-day SLO window:

| Severity | Long Window | Short Window | Burn Rate | Budget Consumed | Time to Exhaustion |
|----------|-------------|--------------|-----------|-----------------|-------------------|
| Page (critical) | 1 hour | 5 minutes | 14.4 | 2% | ~2 days |
| Page (urgent) | 6 hours | 30 minutes | 6 | 5% | ~5 days |
| Ticket (warning) | 1 day | 2 hours | 3 | 10% | ~10 days |
| Ticket (info) | 3 days | 6 hours | 1 | 10% | ~30 days |

The short window is always 1/12th of the long window. The critical alert fires when you're burning budget fast enough to exhaust it in 2 days - that deserves a page. The ticket-level alerts catch slower burns that will eventually exhaust the budget but don't need someone woken up at 3 AM.

### The actual alerting rules

For a 99.9% SLO (error budget = 0.001):

```yaml
groups:
  - name: slo-burn-rate-alerts
    rules:
      # Page: fast burn - will exhaust budget in ~2 days
      - alert: ErrorBudgetFastBurn
        expr: |
          (
              job:slo_errors_per_request:ratio_rate1h{job="order-api"} > (14.4 * 0.001)
            and
              job:slo_errors_per_request:ratio_rate5m{job="order-api"} > (14.4 * 0.001)
          )
          or
          (
              job:slo_errors_per_request:ratio_rate6h{job="order-api"} > (6 * 0.001)
            and
              job:slo_errors_per_request:ratio_rate30m{job="order-api"} > (6 * 0.001)
          )
        labels:
          severity: page
        annotations:
          summary: "order-api is burning error budget at a dangerous rate"
          description: >
            Error budget burn rate exceeds safe threshold.
            At current rate, the 30-day error budget will be exhausted
            within 2-5 days.

      # Ticket: slow burn - will exhaust budget within the window
      - alert: ErrorBudgetSlowBurn
        expr: |
          (
              job:slo_errors_per_request:ratio_rate1d{job="order-api"} > (3 * 0.001)
            and
              job:slo_errors_per_request:ratio_rate2h{job="order-api"} > (3 * 0.001)
          )
          or
          (
              job:slo_errors_per_request:ratio_rate3d{job="order-api"} > (1 * 0.001)
            and
              job:slo_errors_per_request:ratio_rate6h{job="order-api"} > (1 * 0.001)
          )
        labels:
          severity: ticket
        annotations:
          summary: "order-api is slowly burning through its error budget"
          description: >
            Error budget is being consumed at a rate that will
            exhaust it within the 30-day window if the trend continues.
```

Read the page alert expression carefully. It fires when EITHER: (a) both the 1-hour and 5-minute error ratios exceed 14.4x the budget, OR (b) both the 6-hour and 30-minute error ratios exceed 6x the budget. The `and` within each pair is the multi-window check. The `or` between the pairs gives you two sensitivity levels - fast burns and moderate burns both page, but through different detection windows.

The thresholds are just `burn_rate * error_budget`. For the critical alert: `14.4 * 0.001 = 0.0144`, meaning a 1.44% error rate. For the ticket alert at burn rate 3: `3 * 0.001 = 0.003`, meaning a 0.3% error rate sustained over a day.

## What happens when the budget runs out

This is the part that separates SLOs-as-dashboard-decoration from SLOs-as-engineering-tool. You need an **error budget policy** - a written agreement about what happens when the budget is exhausted.

Google's approach, simplified for a typical development team:

**Budget remaining > 50%:** Business as usual. Ship features, experiment, deploy as often as you want.

**Budget remaining 25-50%:** Increased caution. Review deploy practices. Ensure every deploy has a rollback plan. No "YOLO Friday deploys."

**Budget remaining < 25%:** Feature freeze for the affected service. All engineering effort goes to reliability - fixing the causes of budget consumption. Only ship bug fixes, performance improvements, and reliability work.

**Budget exhausted (0% remaining):** Hard freeze. No changes except critical security patches and fixes that directly restore reliability. This continues until the rolling window recovers enough budget.

**Single incident consuming > 20% of the budget:** Mandatory postmortem. At least one concrete action item must come out of it. Not "be more careful" - something measurable. "Add circuit breaker to the payment service dependency" or "reduce deployment batch size from 100% to 25% canary."

### The incentive structure

This is the clever part. Error budgets create a natural tension between velocity and reliability, and give both sides a shared metric to negotiate with.

Product managers want to ship features. If the error budget is healthy, they can push for faster releases - the data says the service can handle more risk. Engineers want reliability. If the error budget is depleted, they have a concrete, data-backed reason to push back on feature work.

Without error budgets, the conversation is "we need to slow down and fix things" vs. "we need to ship this feature." That's a political argument with no resolution criteria. With error budgets, it's "we have 60% budget remaining, we can afford the risk of this deploy" or "we have 5% budget remaining, the data says we need to focus on reliability." The argument resolves itself.

## Automating SLO rule generation with Sloth

Writing all those recording rules and alerting rules by hand is tedious and error-prone. [Sloth](https://github.com/slok/sloth) (v0.15.0, 2,400+ GitHub stars) takes a simple YAML spec and generates the full Prometheus rule set - all the multi-window recording rules and the multi-burn-rate alerts.

Here's a complete Sloth spec for our order-api:

```yaml
version: "prometheus/v1"
service: "order-api"
labels:
  owner: "backend-team"
  repo: "myorg/order-api"
slos:
  - name: "requests-availability"
    objective: 99.9
    description: "HTTP request availability"
    sli:
      events:
        error_query: >
          sum(rate(http_requests_total{job="order-api",status=~"(5..|429)"}[{{.window}}]))
        total_query: >
          sum(rate(http_requests_total{job="order-api"}[{{.window}}]))
    alerting:
      name: OrderApiHighErrorRate
      labels:
        category: "availability"
      annotations:
        summary: "High error rate on order-api"
      page_alert:
        labels:
          severity: page
          routing_key: backend-team
      ticket_alert:
        labels:
          severity: ticket
          slack_channel: "#alerts-backend"
```

The `{{.window}}` template is what makes Sloth work. It generates one version of each query for every time window (5m, 30m, 1h, 2h, 6h, 1d, 3d) and wires them into the correct multi-window alert expressions. One input spec, dozens of generated rules.

Generate the rules:

```bash
sloth generate -i order-api-slo.yaml -o order-api-rules.yaml
```

The output is a standard Prometheus rules file you can drop into your Prometheus config or load through a Kubernetes PrometheusRule CRD.

Sloth also generates metadata metrics: `slo:objective:ratio`, `slo:current_burn_rate:ratio`, `slo:period_error_budget_remaining:ratio`. These feed directly into Grafana dashboards with no additional PromQL needed.

### Alternatives to Sloth

[Pyrra](https://github.com/pyrra-dev/pyrra) (v0.9.5) takes a different approach - it's a single binary with a built-in web UI that shows SLO status, error budget burn-down charts, and auto-generated alert rules. It can run as a Kubernetes operator or in filesystem mode. Pyrra v0.10.0 (currently in release candidate) adds a "Performance Mode" using subquery-based calculations that reduce query load on Prometheus clusters significantly.

[OpenSLO](https://openslo.com/) is a vendor-neutral spec for defining SLOs as YAML. The current stable version is `openslo/v1`. It defines object kinds for `SLO`, `SLI`, `AlertPolicy`, `AlertCondition`, and `Service`. Sloth can consume OpenSLO specs as input, so you get portability across tools.

If you're on Grafana Cloud, their built-in SLO feature (now GA) generates recording rules and alert rules natively - no extra tooling needed.

## Common mistakes

**Setting SLOs too high.** A 99.99% SLO on a service backed by a 99.9% database means you've set a target your infrastructure physically can't meet. Your SLO can't be higher than your weakest dependency's reliability.

**Not including all error types.** If your API returns errors as 200 responses with `{"error": "something broke"}` (looking at you, GraphQL), your HTTP-status-based SLI won't catch them. You need a custom SLI metric that understands your error format, or fix your API to use proper status codes.

**Measuring from the server, not the client.** Your server thinks latency is 50ms. The client sees 200ms because of TLS handshakes, DNS, and network transit. Measure SLIs as close to the user as possible - load balancer metrics, CDN edge metrics, or client-side telemetry.

**Different SLOs for the same user journey.** If placing an order requires hitting `/api/cart`, `/api/checkout`, and `/api/payment`, having separate SLOs for each endpoint misses the point. The user doesn't care which endpoint failed - their order didn't go through. Consider a journey-level SLO that tracks the end-to-end success rate.

**No error budget policy.** SLOs without consequences are just dashboards. If nothing changes when the budget runs out, your SLOs are decoration. The policy is what makes the system work.

## Putting it together - a real setup

Here's what a complete SLO monitoring stack looks like for a Rust service:

**1. Instrumentation** - Your Rust API exposes `http_requests_total` and `http_request_duration_seconds` via the `metrics` crate with a Prometheus exporter. I covered this setup in the [monitoring post](/blog/monitoring-rust-applications-in-production), including the exact code for middleware-based metric collection.

**2. Recording rules** - Prometheus evaluates recording rules every 15 seconds, pre-computing error ratios at 5m, 30m, 1h, 6h, 1d, and 3d windows. Generated by Sloth from a single YAML spec.

**3. Alerting rules** - Multi-window, multi-burn-rate alerts at two severity levels. Page for fast burns (exhaust budget in 2-5 days). Ticket for slow burns (exhaust budget within the 30-day window). Also generated by Sloth.

**4. Dashboard** - A Grafana panel showing:
- Current SLI value (e.g., 99.94%)
- Error budget remaining (e.g., 62%)
- Burn rate over the last hour
- A timeline of error budget consumption over 30 days

**5. Policy** - A document your team agrees to, checked into your repo, that defines what happens at each budget threshold. Reviewed quarterly.

## The latency SLO - worth the extra effort

I've focused mostly on availability because it's simpler. But latency SLOs catch a different class of problem. A service can return 100% successful responses and still be broken if every response takes 10 seconds.

The latency SLI uses histogram buckets:

```yaml
- record: job:slo_latency_good_per_request:ratio_rate5m
  expr: |
    sum(rate(http_request_duration_seconds_bucket{le="0.2"}[5m])) by (job)
    /
    sum(rate(http_request_duration_seconds_count[5m])) by (job)
```

This gives you the proportion of requests completing within 200ms. The burn rate math and alerting structure are identical to availability - just swap the error ratio for the latency ratio.

One subtlety: latency SLOs are more sensitive to load. Your availability SLI might stay at 99.99% while your latency SLI drops to 98% during a traffic spike because your database connection pool is saturated. If you read the [load testing post](/blog/load-testing-your-rust-api---tools-and-methodology/), you've seen how quickly latency percentiles degrade under load. The latency SLO catches this degradation before it becomes an outage.

## Starting small

If you've never used SLOs before, don't try to instrument everything at once. Start with one service - your most critical user-facing API. Define two SLOs (availability and latency). Set conservative targets based on your current performance. Deploy the recording rules and dashboard. Live with it for a month before adding alerting.

The first month will teach you things no design document can. You'll discover that your error rate spikes every day at 2 AM (a cron job that hammers the API). You'll find that your p99 latency is worse than you thought because you never measured it continuously. You'll realize that certain endpoints should have different SLOs than others.

After that first month, add burn rate alerts. After the second month, write your error budget policy. After three months, you'll have enough historical data to adjust your targets and extend to more services.

SLOs are a practice, not a tool. The Prometheus rules and Grafana dashboards are just the implementation. The real value comes from the conversations they enable: "we have budget to take this risk" or "we need to pay down reliability debt before shipping more features." Those conversations, grounded in data instead of opinions, are what make engineering teams ship faster and break less.
