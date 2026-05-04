+++
title = "Kubernetes basics for Rust developers"
date = 2025-11-16
description = "Pods, Services, Deployments, ConfigMaps, Helm, and autoscaling - explained by deploying a Rust Axum service, plus when you should just use Fly.io instead."

[taxonomies]
tags = ["rust", "kubernetes", "devops", "deployment"]
+++

You've built a Rust service. It compiles to a tiny static binary, runs in a 8 MB container, and serves traffic on port 3000. Now somebody says "we need to deploy this on Kubernetes" and hands you a 400-line YAML file that mentions `apiVersion: apps/v1` three times and uses words like `selector`, `matchLabels`, and `readinessProbe` interchangeably.

Kubernetes is a platform for running containers across a fleet of machines. It handles scheduling, networking, restarts, rolling updates, config injection, secrets, scaling, and health checks. The API has 40+ core object types and a learning curve that flattens somewhere around month six.

Most Rust projects don't need any of it. But when you do need it - because your team already uses it, because you're running stateful workloads at scale, because you've outgrown a single VM - the core concepts are small enough to fit in one post. This is that post, written through the lens of deploying a Rust Axum service.

<!-- more -->

## Assumptions

I'll assume you have a working Rust binary packaged as a container image. If you haven't built that yet, I covered the whole pipeline in [Docker multi-stage builds for Rust](/blog/docker-multi-stage-builds-for-rust-from-2gb-to-20mb/) - static musl binary on a `scratch` base, 8 MB final image, non-root user, CA certs baked in. That's the starting point for everything here.

The example app is an Axum server with a `/health` endpoint and a `/api/users` route. The image is `ghcr.io/you/myapp:v1.2.3`. We'll deploy it to a Kubernetes cluster - could be `minikube` locally, a managed cluster (EKS, GKE, DigitalOcean Kubernetes), or `k3s` on a single VPS.

## The core objects you actually need

Kubernetes has many object types. For a standard stateless HTTP service, you need five:

| Object | What it does |
|--------|--------------|
| **Pod** | One or more containers sharing a network namespace. The smallest unit K8s schedules. |
| **Deployment** | Manages a set of identical Pods, handles rolling updates, rollbacks, replica count. |
| **Service** | A stable DNS name and virtual IP for a set of Pods. Handles load balancing. |
| **ConfigMap** | Non-secret configuration (env vars, config files) mounted into Pods. |
| **Secret** | Same as ConfigMap but base64-encoded and usually stored encrypted at rest. |

That's it. You can run a production Rust service with just those. Everything else (StatefulSets, DaemonSets, Jobs, CronJobs, Ingresses, NetworkPolicies, ServiceAccounts, PVCs, CRDs) is useful for specific cases but not required for your first deployment.

## Pods - not what you put in YAML

A Pod is a group of containers that share a network namespace, some volumes, and a lifecycle. In practice, 95% of Pods run exactly one container.

You almost never write Pod YAML directly. You write a Deployment, and the Deployment creates Pods. The reason: a bare Pod is ephemeral. If the node it runs on dies, the Pod is gone. A Deployment ensures that if a Pod dies, a replacement appears.

But it helps to understand what's inside. Here's a minimal Pod spec for our Rust service:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  containers:
    - name: myapp
      image: ghcr.io/you/myapp:v1.2.3
      ports:
        - containerPort: 3000
```

`kubectl apply -f pod.yaml` and the Pod starts. Inside the Pod, your Rust binary runs as PID 1 (or PID close to it, depending on the runtime). It has its own network namespace - `localhost` inside the Pod is not the host's `localhost`, it's the Pod's.

If your binary panics, Kubernetes restarts it according to the Pod's restart policy (default: `Always`). But if the node crashes, the Pod is gone forever. That's why you use Deployments.

## Deployments - the replica controller

A Deployment declares "I want N copies of this Pod running at all times, and here's how to roll out updates."

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
        - name: myapp
          image: ghcr.io/you/myapp:v1.2.3
          ports:
            - containerPort: 3000
          env:
            - name: RUST_LOG
              value: "info"
            - name: RUST_BACKTRACE
              value: "1"
          resources:
            requests:
              cpu: "50m"
              memory: "32Mi"
            limits:
              cpu: "500m"
              memory: "128Mi"
          readinessProbe:
            httpGet:
              path: /health
              port: 3000
            initialDelaySeconds: 1
            periodSeconds: 5
          livenessProbe:
            httpGet:
              path: /health
              port: 3000
            initialDelaySeconds: 5
            periodSeconds: 10
```

Three things to unpack: the `selector`, the probes, and the resources.

**The selector is a label match.** The Deployment looks for Pods with the label `app: myapp` and considers those "its" Pods. The Pod template at the bottom creates Pods with exactly that label. This indirection seems pointless for a single Deployment but it's how Services and other objects find Pods too - everything in Kubernetes is glued together by labels, not by direct references.

**Rolling updates happen automatically.** Change the image tag from `v1.2.3` to `v1.2.4`, apply the YAML, and Kubernetes creates a new ReplicaSet with the new image, starts Pods one at a time, waits for each to become ready, and terminates old Pods. If a new Pod fails its readiness probe, the rollout stops. You can then `kubectl rollout undo deployment/myapp` and you're back on the old version in seconds.

**Pod identity doesn't persist.** Each new Pod gets a new random name, a new IP, and a fresh filesystem. Don't store state locally. This is where Rust's statelessness story shines - if your Axum handler doesn't touch local disk and doesn't keep in-memory state that matters, you're already K8s-ready.

## The /health endpoint - what probes actually check

The Kubernetes `readinessProbe` and `livenessProbe` decide two different things:

- **Readiness**: should this Pod receive traffic? If no, K8s removes it from the Service's load balancer but doesn't restart it.
- **Liveness**: is this Pod still alive? If no, K8s kills the container and restarts it.

You can implement both with a simple `/health` route in Axum:

```rust
use axum::{routing::get, Json, Router};
use serde::Serialize;
use std::sync::Arc;

#[derive(Serialize)]
struct Health {
    status: &'static str,
    version: &'static str,
}

async fn health() -> Json<Health> {
    Json(Health {
        status: "ok",
        version: env!("CARGO_PKG_VERSION"),
    })
}

#[tokio::main]
async fn main() {
    let app = Router::new()
        .route("/health", get(health))
        .route("/api/users", get(list_users));

    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000")
        .await
        .unwrap();
    axum::serve(listener, app).await.unwrap();
}

async fn list_users() -> &'static str { "..." }
```

A naive `/health` that always returns 200 is fine for liveness - you just want to know the process is alive and responsive. For readiness, you want to check that the app is actually able to serve requests. If the app depends on a database, check the database connection:

```rust
use sqlx::PgPool;

#[derive(Clone)]
struct AppState {
    db: PgPool,
}

async fn readiness(
    axum::extract::State(state): axum::extract::State<Arc<AppState>>,
) -> Result<Json<Health>, axum::http::StatusCode> {
    match sqlx::query("SELECT 1").execute(&state.db).await {
        Ok(_) => Ok(Json(Health { status: "ok", version: env!("CARGO_PKG_VERSION") })),
        Err(_) => Err(axum::http::StatusCode::SERVICE_UNAVAILABLE),
    }
}
```

Split them: `/health` for liveness (is the process alive?), `/ready` for readiness (can it serve traffic?). Never have `/ready` depend on a downstream that would cause a cascade failure if it blips - for example, if every Pod's readiness depends on a shared Redis and Redis burps for 5 seconds, you've just taken down your entire fleet simultaneously. Degrade gracefully instead.

A subtle point: the first probe fires `initialDelaySeconds` after the container starts. If your Rust app takes 2 seconds to compile schemas at startup and you set `initialDelaySeconds: 0` on a liveness probe with `failureThreshold: 1`, the Pod enters a crash loop before it ever reaches the first request. `startupProbe` exists specifically for this - it runs first, disables the other two until it passes, and gets its own long timeout.

## Services - stable endpoints for ephemeral Pods

Pods come and go. IPs change. You need a stable name other apps can call. That's a Service:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp
spec:
  selector:
    app: myapp
  ports:
    - port: 80
      targetPort: 3000
  type: ClusterIP
```

The Service watches for Pods with label `app: myapp` and maintains a list of their IPs. It exposes a virtual IP (the cluster IP) and a DNS name (`myapp.default.svc.cluster.local`) that load-balances across those Pods.

There are three Service types you'll encounter:

**ClusterIP** - internal only. Other Pods can reach it via the DNS name but it's not exposed outside the cluster. Default and most common for service-to-service calls.

**NodePort** - opens a port (30000-32767) on every node. External clients hit any node IP on that port and get load-balanced to the Pods. Rarely used directly in production.

**LoadBalancer** - on cloud providers, this provisions an actual load balancer (AWS ELB, GCP Load Balancer, etc). The cloud LB terminates external traffic and forwards to NodePorts behind the scenes. This is how external traffic usually enters a cluster.

For HTTP, most teams use an Ingress (or a newer Gateway API object) instead of `type: LoadBalancer` on every Service. An Ingress is a shared HTTP reverse proxy (usually nginx-ingress or traefik) that routes based on hostname and path. You get one LoadBalancer for the whole cluster instead of one per service.

## ConfigMaps and Secrets - feeding config to Rust

Your Rust app probably reads config from environment variables or a config file. ConfigMaps and Secrets inject both.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: myapp-config
data:
  RUST_LOG: "info,myapp=debug"
  FEATURE_NEW_API: "true"
  MAX_CONNECTIONS: "100"
---
apiVersion: v1
kind: Secret
metadata:
  name: myapp-secrets
type: Opaque
stringData:
  DATABASE_URL: "postgres://user:pass@db.internal:5432/myapp"
  JWT_SECRET: "s3cret-key-change-me"
```

Then reference them in the Deployment:

```yaml
spec:
  containers:
    - name: myapp
      envFrom:
        - configMapRef:
            name: myapp-config
        - secretRef:
            name: myapp-secrets
```

`envFrom` pulls every key as an environment variable. Your Rust code reads them with `std::env::var`:

```rust
let db_url = std::env::var("DATABASE_URL")
    .expect("DATABASE_URL must be set");
let pool = PgPool::connect(&db_url).await.unwrap();
```

A few things people get wrong:

**Secrets aren't encrypted by default.** They're base64-encoded, which is not encryption. Anyone with `get secret` permission on the cluster can read them. For real encryption, use sealed-secrets, external-secrets with a vault backend, or the cloud provider's secret manager. On a managed cluster, enable encryption at rest at the etcd layer (most managed K8s providers do this by default now).

**Updating a ConfigMap does not restart Pods.** If you change `RUST_LOG` in the ConfigMap, existing Pods keep the old value until they restart. You either roll the Deployment (`kubectl rollout restart deployment/myapp`) or mount the ConfigMap as a file and have your app reload it on a SIGHUP. The env-var approach is simpler but less dynamic.

**Don't ship secrets in the image.** Build-time ARGs end up in image layers. `docker history your-image` will show them. Always inject at runtime via Secret.

## Resource limits and the OOM killer

Every container has two resource numbers per dimension: `requests` (what the scheduler guarantees) and `limits` (the hard ceiling).

```yaml
resources:
  requests:
    cpu: "50m"          # 0.05 CPU cores guaranteed
    memory: "32Mi"      # 32 MiB guaranteed
  limits:
    cpu: "500m"         # burst up to 0.5 cores
    memory: "128Mi"     # hard cap, exceed = OOM kill
```

CPU and memory behave very differently under pressure:

**CPU throttling is soft.** If your Pod exceeds its CPU limit, the kernel CFS scheduler throttles it. Your code runs slower. Nothing dies. You'll see latency spikes.

**Memory limits are hard.** If your Pod exceeds its memory limit, the kernel OOM killer terminates the container. Kubernetes restarts it. If it OOMs repeatedly, you get `CrashLoopBackOff` and the Pod sits in a failure state.

For Rust services, memory usage is usually flat and predictable - unless you're buffering large requests, building big in-memory indexes, or leaking (it happens even in Rust - `Box::leak`, cyclic `Arc`s, caches that never evict). A good starting point for a small Axum service:

- `requests: 32Mi, limits: 128Mi`
- Watch actual usage in `kubectl top pod myapp` or your metrics system for a few days
- Set requests to the p95 and limits to 2-3x that

If you use a custom allocator like `mimalloc` (as discussed in the [Docker post](/blog/docker-multi-stage-builds-for-rust-from-2gb-to-20mb/)), RSS can look different from glibc or musl default allocator behavior. Always measure under real load. `cgroups` accounting (what K8s reads) includes page cache for some kernels - your "memory usage" graph may show more than your allocator reports.

The CPU side matters for the next topic.

## Horizontal Pod Autoscaler

HPA scales the number of replicas based on metrics. The classic one is CPU:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  minReplicas: 2
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```

"Scale up when average CPU across Pods exceeds 70% of the *request* (not limit)." With `request: 50m`, 70% means ~35m. When the average exceeds that, HPA adds replicas.

For Rust services, CPU-based autoscaling works reasonably well for request-bound workloads. But async runtimes can saturate CPU in subtle ways - one blocking call in a handler and your tokio runtime is starved, but top-level CPU graphs look fine. Custom metrics (requests per second, queue depth, p99 latency) often make more sense:

```yaml
metrics:
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: "100"
```

That requires the metrics to be exposed (Prometheus adapter or similar). The autoscaling/v2 docs on the official [HPA walkthrough](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale-walkthrough/) cover the setup.

Two gotchas:

**HPA only scales Pods, not nodes.** If your cluster doesn't have capacity, new Pods go into `Pending` status. You need the cluster autoscaler (separate component) to add nodes.

**Scale-down is slow by default.** Kubernetes waits 5 minutes of stable low metrics before scaling down to avoid flapping. Scale-up is fast. This is usually what you want but it means you pay for peak capacity for ~5 minutes after each spike.

## Helm - templated YAML at scale

You now have Deployment, Service, ConfigMap, Secret, HPA - five YAML files. Different environments (dev, staging, production) have different replica counts, different image tags, different secrets. You end up with `myapp-dev.yaml`, `myapp-prod.yaml`, or you write a bash script to `sed` values in.

Helm is the standard solution. A Helm chart is a directory with YAML templates and a `values.yaml`:

```
myapp/
├── Chart.yaml
├── values.yaml
└── templates/
    ├── deployment.yaml
    ├── service.yaml
    └── configmap.yaml
```

`templates/deployment.yaml` uses Go template syntax:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}
spec:
  replicas: {{ .Values.replicaCount }}
  template:
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
```

`values.yaml` provides defaults:

```yaml
replicaCount: 3
image:
  repository: ghcr.io/you/myapp
  tag: v1.2.3
resources:
  requests:
    cpu: "50m"
    memory: "32Mi"
  limits:
    cpu: "500m"
    memory: "128Mi"
```

Install it:

```bash
helm install myapp ./myapp
# Override values
helm install myapp-prod ./myapp -f values-prod.yaml
# Upgrade
helm upgrade myapp ./myapp --set image.tag=v1.2.4
# Rollback
helm rollback myapp 1
```

Helm tracks releases and makes rollback one command. Chart repositories (`helm repo add bitnami https://charts.bitnami.com/bitnami`) let you install common services (postgres, redis, etc) in one line.

Is Helm great? No, it's fine. Go templates on YAML is a fundamentally awkward way to generate structured data - you'll hit whitespace bugs, nested conditional hell, and the "why is this field empty" dance. The alternatives (kustomize, jsonnet, cdk8s, timoni) each have their own trade-offs. Helm's saving grace is ubiquity: every production K8s cluster uses it, every vendor ships a chart, and everyone on your team already knows the basics.

## When Kubernetes is overkill

Kubernetes is designed for the problems Google had in 2014: thousands of machines, hundreds of services, independent teams shipping constantly, strict isolation requirements. Your situation is almost certainly not that.

You probably don't need Kubernetes if:

- You have fewer than 5 services total
- Your traffic fits on one VM with room to spare
- You don't need multi-region failover
- Your team has fewer than 10 engineers
- You're not running stateful databases at scale
- Nobody on your team has deep K8s operational experience

The cost of running K8s isn't the cluster bill (though a managed control plane is $70-150/month minimum). The cost is the ongoing mental overhead: upgrades, networking issues, resource tuning, debugging CrashLoopBackOffs at 3am, understanding why a rolling deploy stalled because of a misconfigured PodDisruptionBudget. For a small team with a single Rust service, that time is better spent on the product.

I've watched teams of three engineers run three services on EKS because "everyone uses Kubernetes." The service would fit happily on a $20 VPS with systemd. They spent months on infrastructure instead of features. Don't be them.

## Simpler alternatives

For most Rust services I've shipped, the deployment story is: `git push`, a CI job builds the container, and a platform picks it up and runs it. No YAML, no kubelet, no etcd.

**Fly.io** - runs your Docker image as a Firecracker microVM on their edge network. You write a `fly.toml`, run `fly deploy`, and your app is live in multiple regions in under a minute. Built-in health checks, horizontal autoscaling, secrets management, and Postgres. Pricing starts at ~$2/month per small instance. The `fly.toml` for our Axum app is maybe 15 lines.

**Railway** - similar model, simpler UI, more forgiving defaults. Point it at a GitHub repo, it detects the Dockerfile, builds and deploys. Good for early-stage projects where the team doesn't have an infra person. Pricing is usage-based.

**Render** - like Railway but slightly more enterprise-y. Zero-config deploys, preview environments on PRs, native Postgres/Redis.

**Single VPS with Docker Compose or systemd** - still the right answer for most side projects and early-stage startups. A $10 VPS runs a Rust service handling tens of millions of requests per month with capacity to spare. `docker compose up -d` or a systemd unit file, nginx or Caddy for TLS, done. If you need resilience, two VPSes behind a cheap load balancer. Upgrade to K8s when the pain of not having it exceeds the pain of having it, not before.

**Managed Nomad clusters** - if you want container orchestration without the K8s complexity. Single binary control plane, straightforward config language, works great for batch jobs and services mixed together.

## If you do use Kubernetes, keep it boring

If you've worked through all of this and decided you genuinely need K8s (or you inherited it), a few pieces of advice from people who've operated it in production:

- Use a managed control plane. EKS, GKE, AKS, DigitalOcean Kubernetes. Do not run your own control plane unless that's your full-time job.
- Start with the 5 objects in this post. Resist adding CRDs, operators, service meshes, and GitOps until you actually hit a pain point.
- Pin Kubernetes versions. Upgrades break things. Test them.
- Resource limits on every container, not "we'll add them later."
- Single namespace per app or environment, not one giant `default` namespace.
- Readiness probes that actually check readiness. Liveness probes that are conservative - a failing liveness probe restarts containers, which can turn a slow-dependency blip into an outage.
- Observability before scale. Logs, metrics, and traces from day one. You cannot debug a Pod that's been restarted 80 times without them.

Kubernetes is a powerful, complex, occasionally beautiful, frequently infuriating platform. It solves real problems. It also creates many problems you didn't have before. The trick is to use it only when you need it and to keep your usage as boring as possible when you do.

For a Rust service, the story usually ends at "8 MB static binary, one container, one healthcheck, one platform, done." If yours doesn't, the YAML above is the shortest path from zero to a production K8s deployment I know how to write.
