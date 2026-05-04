+++
title = "Running Rust in AWS Lambda - serverless without the cold start tax"
date = 2025-04-20
description = "How Rust on Lambda gives you 10-50ms cold starts, single-binary deploys, and lower costs than most managed runtimes - with cargo-lambda, lambda_runtime, and real deployment patterns."

[taxonomies]
tags = ["rust", "aws", "lambda", "serverless"]
+++

AWS Lambda charges you for two things: the number of requests and how long your code runs. If your runtime takes 500ms just to initialize, you're paying for 500ms of nothing on every cold start. That's the cold start tax, and it's the reason Java Lambda functions are a meme in serverless circles.

Rust doesn't have this problem. There's no VM to boot, no interpreter to load, no garbage collector to warm up. Your Lambda function is a single Linux binary that starts executing immediately. Cold starts of 10-50ms are normal, not aspirational.

This post covers the full path: project setup with cargo-lambda, the lambda_runtime internals, HTTP APIs with API Gateway, DynamoDB/S3 access, local testing, deployment, and the cost math that makes Rust on Lambda genuinely compelling.

<!-- more -->

## The runtime model

Lambda functions using managed runtimes (Python, Node.js, Java) ship your source code, and AWS provides the language runtime. Rust takes a different path. You compile a static binary targeting Linux, and deploy it on `provided.al2023` - an OS-only runtime based on Amazon Linux 2023 that supports both x86_64 and arm64 architectures.

The binary itself implements the [Lambda Runtime API](https://docs.aws.amazon.com/lambda/latest/dg/runtimes-api.html) - a simple HTTP protocol. Your process starts, polls `GET /2018-06-01/runtime/invocation/next` for events, processes them, and posts the response back. That's it. No sidecar, no agent, no shim.

The [aws-lambda-rust-runtime](https://github.com/awslabs/aws-lambda-rust-runtime) project (3.6k stars, actively maintained by AWS Labs) provides the crates that handle this protocol for you:

- **lambda_runtime** (v1.1.2) - the core event loop, Tower service integration
- **lambda_http** - converts API Gateway/ALB events into standard `http::Request` types
- **lambda_events** - typed structs for SQS, S3, Cognito, CloudWatch, and dozens of other event sources
- **lambda_extension** - for building Lambda Extensions in Rust

## Why cold starts are fast

A Python Lambda cold start involves: start the OS, load the Python interpreter, import your dependencies, run module-level code, then call your handler. A Java one is worse - JVM boot, class loading, JIT warmup.

A Rust Lambda cold start is: start the OS, exec the binary. That's it. The binary is already compiled, already optimized, already linked. There's no interpretation step. If you covered how zero-cost abstractions compile away at build time in [the earlier post](/blog/zero-cost-abstractions-in-rust-what-it-actually-means), this is where that matters in practice. All the iterator chains, all the generic monomorphization - it happened at compile time, not at Lambda init time.

Real-world numbers from the [lambda-perf](https://github.com/maxday/lambda-perf) benchmark project (runs daily across 40+ runtimes):

| Runtime | Cold Start (128MB) | Cold Start (512MB) |
|---------|-------------------|-------------------|
| Rust (provided.al2023) | ~10-16ms | ~10-12ms |
| Go (provided.al2023) | ~10-14ms | ~8-11ms |
| Node.js 22.x | ~150-250ms | ~100-170ms |
| Python 3.13 | ~120-200ms | ~80-150ms |
| Java 21 | ~800-3500ms | ~500-2000ms |
| .NET 8 | ~250-400ms | ~180-300ms |

The Rust numbers barely change with memory allocation because the binary is small and there's nothing to JIT. Java's range is enormous because JVM init is proportional to the resources available - it eagerly uses whatever memory you give it.

## Project setup with cargo-lambda

[cargo-lambda](https://www.cargo-lambda.info/) is the standard tool for Rust Lambda development. It handles cross-compilation (you're probably not developing on Amazon Linux), local testing, and deployment. Install it:

```bash
# With cargo
cargo install cargo-lambda

# Or with Homebrew
brew tap cargo-lambda/cargo-lambda
brew install cargo-lambda
```

Create a new function:

```bash
cargo lambda new my-api --http-feature apigw_rest
```

The `--http-feature` flag tells it to set up an HTTP-triggered function (API Gateway). Without it, you get a raw event handler. Here's what the generated project looks like:

```toml
# Cargo.toml
[package]
name = "my-api"
version = "0.1.0"
edition = "2024"

[dependencies]
lambda_http = "0.14"
lambda_runtime = "0.14"
tokio = { version = "1", features = ["macros"] }
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter", "fmt"] }
```

And the handler:

```rust
use lambda_http::{run, service_fn, tracing, Body, Error, Request, Response};

async fn handler(event: Request) -> Result<Response<Body>, Error> {
    let who = event
        .query_string_parameters_ref()
        .and_then(|params| params.first("name"))
        .unwrap_or("world");

    let resp = Response::builder()
        .status(200)
        .header("content-type", "application/json")
        .body(format!(r#"{{"message": "hello {who}"}}"#).into())
        .map_err(Box::new)?;

    Ok(resp)
}

#[tokio::main]
async fn main() -> Result<(), Error> {
    tracing_subscriber::fmt()
        .with_max_level(tracing::Level::INFO)
        .with_target(false)
        .without_time() // Lambda adds timestamps
        .init();

    run(service_fn(handler)).await
}
```

Notice there's no Lambda-specific boilerplate in the handler itself. It receives a standard `http::Request` and returns a standard `http::Response`. The `lambda_http` crate's adapter layer converts API Gateway events into these types transparently.

## How the runtime actually works

If you've read the [tokio deep dive](/blog/understanding-tokio---the-rust-async-runtime-under-the-hood/), you know the async runtime spins up a thread pool and an I/O driver. Lambda's runtime sits on top of that. Here's the simplified event loop from `lambda-runtime/src/runtime.rs`:

```rust
// Simplified from the actual source
fn incoming(client: &ApiClient) -> impl Stream<Item = Result<...>> + Send {
    async_stream::stream! {
        loop {
            trace!("Waiting for next event (incoming loop)");
            let req = NextEventRequest.into_req().expect("valid request");
            let res = client.call(req).await;
            yield res;
        }
    }
}
```

The runtime creates an infinite async stream that long-polls the Lambda Runtime API. Each iteration hits `/2018-06-01/runtime/invocation/next` - this blocks until Lambda has an event for you. When an event arrives, `run_with_incoming()` processes it:

```rust
while let Some(next_event_response) = incoming.next().await {
    let event = next_event_response?;
    process_invocation(&mut service, &config, event, true).await?;
}
```

Sequential, one event at a time. The runtime also supports concurrent processing via `run_concurrent()` for Lambda Managed Instances - when `AWS_LAMBDA_MAX_CONCURRENCY` is set above 1, it spawns multiple worker tasks each running their own polling loop.

The key design choice: the handler is a `tower::Service`. This means you can compose middleware using the entire Tower ecosystem - tracing, timeouts, rate limiting, retries - the same middleware you'd use in an Axum server. If you read the [web frameworks comparison](/blog/axum-vs-actix-web-vs-warp-rust-web-frameworks-in-2026), Axum's Tower foundation is what makes it a natural fit for Lambda.

## HTTP APIs with Axum on Lambda

You don't have to use `service_fn`. The `lambda_http::run()` function accepts any `tower::Service<Request>`, and Axum's `Router` implements exactly that trait. This means you can run your full Axum application on Lambda:

```rust
use axum::{
    extract::{Path, Query, State},
    http::StatusCode,
    response::Json,
    routing::{get, post, delete},
    Router,
};
use lambda_http::{run, tracing, Error};
use serde::{Deserialize, Serialize};
use std::sync::Arc;

#[derive(Clone)]
struct AppState {
    db: aws_sdk_dynamodb::Client,
}

#[derive(Serialize)]
struct Product {
    id: String,
    name: String,
    price_cents: i64,
}

#[derive(Deserialize)]
struct CreateProductRequest {
    name: String,
    price_cents: i64,
}

async fn list_products(
    State(state): State<Arc<AppState>>,
) -> Result<Json<Vec<Product>>, StatusCode> {
    // DynamoDB scan, covered in the next section
    todo!()
}

async fn get_product(
    State(state): State<Arc<AppState>>,
    Path(id): Path<String>,
) -> Result<Json<Product>, StatusCode> {
    todo!()
}

async fn create_product(
    State(state): State<Arc<AppState>>,
    Json(payload): Json<CreateProductRequest>,
) -> Result<(StatusCode, Json<Product>), StatusCode> {
    todo!()
}

#[tokio::main]
async fn main() -> Result<(), Error> {
    tracing_subscriber::fmt()
        .with_max_level(tracing::Level::INFO)
        .without_time()
        .init();

    let config = aws_config::load_from_env().await;
    let db = aws_sdk_dynamodb::Client::new(&config);
    let state = Arc::new(AppState { db });

    let app = Router::new()
        .route("/products", get(list_products).post(create_product))
        .route("/products/:id", get(get_product))
        .with_state(state);

    // This line is the only Lambda-specific part
    run(app).await
}
```

One line - `run(app).await` - is the only thing that makes this a Lambda function instead of a regular Axum server. You could swap that for `axum::serve()` and run the same code on EC2. That's not theoretical - it's a practical deployment strategy. Develop locally as a normal HTTP server, deploy to Lambda in production.

One important detail: set the environment variable `AWS_LAMBDA_HTTP_IGNORE_STAGE_IN_PATH=true` in your Lambda config. API Gateway prepends a stage name (`/prod`, `/staging`) to all paths, and without this flag your routes won't match.

## Accessing DynamoDB and S3

The [AWS SDK for Rust](https://github.com/awslabs/aws-sdk-rust) works exactly as you'd expect in Lambda. The key pattern: create SDK clients in `main()` (outside the handler), pass them through state. This way the client is initialized once during the cold start and reused across all warm invocations.

```rust
use aws_sdk_dynamodb::Client as DynamoClient;
use aws_sdk_dynamodb::types::AttributeValue;
use serde_dynamo::{from_items, to_item};

async fn list_products(
    State(state): State<Arc<AppState>>,
) -> Result<Json<Vec<Product>>, StatusCode> {
    let result = state.db
        .scan()
        .table_name("products")
        .send()
        .await
        .map_err(|_| StatusCode::INTERNAL_SERVER_ERROR)?;

    let items = result.items();
    let products: Vec<Product> = from_items(items.to_vec())
        .map_err(|_| StatusCode::INTERNAL_SERVER_ERROR)?;

    Ok(Json(products))
}

async fn create_product(
    State(state): State<Arc<AppState>>,
    Json(payload): Json<CreateProductRequest>,
) -> Result<(StatusCode, Json<Product>), StatusCode> {
    let product = Product {
        id: uuid::Uuid::new_v4().to_string(),
        name: payload.name,
        price_cents: payload.price_cents,
    };

    let item = to_item(&product)
        .map_err(|_| StatusCode::INTERNAL_SERVER_ERROR)?;

    state.db
        .put_item()
        .table_name("products")
        .set_item(Some(item))
        .send()
        .await
        .map_err(|_| StatusCode::INTERNAL_SERVER_ERROR)?;

    Ok((StatusCode::CREATED, Json(product)))
}
```

The [serde_dynamo](https://crates.io/crates/serde_dynamo) crate handles conversion between your Rust types and DynamoDB's `AttributeValue` types. You already have serde derives on your structs (if you haven't read the [serde deep dive](/blog/serde-deep-dive-beyond-derive), now's a good time) - serde_dynamo leverages those same derives.

S3 follows the same pattern:

```rust
use aws_sdk_s3::Client as S3Client;

async fn upload_image(
    state: &AppState,
    key: &str,
    body: Vec<u8>,
) -> Result<(), aws_sdk_s3::Error> {
    state.s3
        .put_object()
        .bucket("my-bucket")
        .key(key)
        .body(body.into())
        .content_type("image/png")
        .send()
        .await?;
    Ok(())
}
```

No special Lambda magic. The SDK reads `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, and `AWS_SESSION_TOKEN` from the environment - Lambda sets these automatically from your function's IAM role.

## Processing non-HTTP events

Lambda isn't just for HTTP APIs. You can process SQS messages, S3 events, CloudWatch scheduled events, and more. The `lambda_events` crate provides typed structs for all of them:

```rust
use lambda_runtime::{run, service_fn, tracing, Error, LambdaEvent};
use aws_lambda_events::event::sqs::SqsEvent;

async fn handle_sqs(event: LambdaEvent<SqsEvent>) -> Result<(), Error> {
    for record in &event.payload.records {
        let body = record.body.as_deref().unwrap_or_default();
        tracing::info!(message_id = ?record.message_id, "Processing: {body}");

        // Your business logic here
        process_message(body).await?;
    }
    Ok(())
}

#[tokio::main]
async fn main() -> Result<(), Error> {
    tracing::init_default_subscriber();
    run(service_fn(handle_sqs)).await
}
```

For S3 notifications:

```rust
use aws_lambda_events::event::s3::S3Event;

async fn handle_s3(event: LambdaEvent<S3Event>) -> Result<(), Error> {
    for record in &event.payload.records {
        let bucket = &record.s3.bucket.name;
        let key = &record.s3.object.key;
        tracing::info!("New object: s3://{}/{}", 
            bucket.as_deref().unwrap_or("?"), 
            key.as_deref().unwrap_or("?"));
    }
    Ok(())
}
```

The `LambdaEvent<T>` wrapper gives you both the deserialized payload and a `Context` struct with the request ID, function ARN, memory limit, and remaining time. That last one is useful - you can check `context.deadline` to bail out gracefully before Lambda kills your process.

## Local testing

cargo-lambda provides a local emulator. No Docker, no SAM CLI, no LocalStack:

```bash
# Terminal 1: start the watch server
cargo lambda watch

# Terminal 2: invoke with a test event
cargo lambda invoke --data-ascii '{"name": "test"}'

# For HTTP functions, just use curl
curl http://localhost:9000/products
```

`cargo lambda watch` compiles your function and starts a local HTTP server that mimics the Lambda Runtime API. It watches for file changes and recompiles automatically. This is significantly faster than the SAM CLI workflow, which builds a Docker container for every invocation.

For more structured testing, you can invoke with event files:

```bash
# Generate a sample API Gateway event
cargo lambda invoke --data-file events/api-gateway.json
```

And of course, unit tests work normally:

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use lambda_http::{Request, Body};

    #[tokio::test]
    async fn test_handler_returns_200() {
        let request = Request::builder()
            .uri("/?name=test")
            .body(Body::Empty)
            .unwrap();

        let response = handler(request).await.unwrap();
        assert_eq!(response.status(), 200);

        let body = std::str::from_utf8(response.body().as_ref()).unwrap();
        assert!(body.contains("test"));
    }
}
```

No mocking framework needed. The handler takes a standard `Request` and returns a standard `Response`. Test it like any other function.

## Building and deploying

### Building

```bash
# For ARM64 (recommended - better price/performance)
cargo lambda build --release --arm64

# For x86_64
cargo lambda build --release
```

cargo-lambda uses [Zig](https://ziglang.org/) as a cross-compilation linker. This means you can build ARM64 Linux binaries from macOS or Windows without Docker. The output goes to `target/lambda/my-api/bootstrap` - a single binary named `bootstrap`, which is what Lambda expects.

Check the binary size:

```bash
ls -lh target/lambda/my-api/bootstrap
# Typically 5-15MB for a simple function
# Compare: a Node.js function with node_modules can easily be 50-100MB
```

You can shrink it further with standard Rust tricks in your `Cargo.toml`:

```toml
[profile.release]
strip = true        # Strip debug symbols
lto = true          # Link-time optimization
codegen-units = 1   # Better optimization, slower compile
opt-level = "z"     # Optimize for size
```

This can get a simple function down to 2-4MB. Smaller binary means faster cold starts since Lambda needs to decompress and load it into memory.

### Deploy with cargo-lambda

The simplest path:

```bash
cargo lambda deploy my-api \
    --iam-role arn:aws:iam::123456789:role/lambda-role
```

This creates (or updates) the Lambda function, uploads the binary as a ZIP, and configures the `provided.al2023` runtime. For HTTP functions, you'll still need to set up API Gateway separately.

### Deploy with AWS SAM

For infrastructure-as-code, AWS SAM works well:

```yaml
# template.yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31

Globals:
  Function:
    Timeout: 10
    MemorySize: 128
    Architectures:
      - arm64

Resources:
  MyApi:
    Type: AWS::Serverless::Function
    Properties:
      FunctionName: my-api
      Handler: bootstrap
      Runtime: provided.al2023
      CodeUri: target/lambda/my-api/
      Events:
        ApiGateway:
          Type: HttpApi
          Properties:
            Path: /{proxy+}
            Method: ANY
      Policies:
        - DynamoDBCrudPolicy:
            TableName: !Ref ProductsTable

  ProductsTable:
    Type: AWS::DynamoDB::Table
    Properties:
      TableName: products
      AttributeDefinitions:
        - AttributeName: id
          AttributeType: S
      KeySchema:
        - AttributeName: id
          KeyType: HASH
      BillingMode: PAY_PER_REQUEST
```

```bash
cargo lambda build --release --arm64
sam deploy --guided
```

SAM bundles the API Gateway, Lambda function, DynamoDB table, and IAM permissions into a single CloudFormation stack.

### Deploy with AWS CDK

If you prefer CDK (TypeScript):

```typescript
import * as cdk from 'aws-cdk-lib';
import * as lambda from 'aws-cdk-lib/aws-lambda';
import * as apigw from 'aws-cdk-lib/aws-apigatewayv2';
import { HttpLambdaIntegration } from 'aws-cdk-lib/aws-apigatewayv2-integrations';

const fn = new lambda.Function(this, 'MyApi', {
  runtime: lambda.Runtime.PROVIDED_AL2023,
  handler: 'bootstrap',
  code: lambda.Code.fromAsset('target/lambda/my-api'),
  architecture: lambda.Architecture.ARM_64,
  memorySize: 128,
  timeout: cdk.Duration.seconds(10),
});

const api = new apigw.HttpApi(this, 'HttpApi');
api.addRoutes({
  path: '/{proxy+}',
  methods: [apigw.HttpMethod.ANY],
  integration: new HttpLambdaIntegration('Integration', fn),
});
```

## The cost math

Lambda pricing (us-east-1, ARM64/Graviton2):

- **Requests**: $0.20 per 1M requests
- **Duration**: $0.0000133334 per GB-second
- **Free tier**: 1M requests + 400,000 GB-seconds per month

Let's compare a Rust Lambda API handling 10M requests/month at 128MB memory with an average execution time of 5ms:

**Lambda cost:**
- Requests: 10M * $0.20/1M = **$2.00**
- Compute: 10M * 0.005s * 0.125GB * $0.0000133334 = **$0.83**
- Total: **$2.83/month**

The same workload on an always-on EC2 instance:

**EC2 t4g.micro (2 vCPU, 1GB RAM):**
- On-demand: **$6.12/month**
- And you still need to handle scaling, patching, health checks, load balancing

**EC2 t4g.small (2 vCPU, 2GB RAM):**
- On-demand: **$12.26/month**

At 10M requests/month, Lambda with Rust is cheaper than the smallest practical EC2 instance. And that's before you account for the operational overhead of running a server - no ALB cost, no ECS/EKS cluster, no auto-scaling configuration.

The break-even point shifts around 50-100M requests/month depending on your execution time. Past that, reserved EC2 instances or Fargate start winning on raw compute cost. But Lambda's advantage isn't just price - it's that you pay exactly zero when nobody's calling your API.

Where the math changes: if your function runs for seconds (not milliseconds), or needs more than 10GB of memory, or makes heavy use of provisioned concurrency. Rust helps even there - a function that takes 200ms in Python might take 5ms in Rust, directly reducing your duration cost by 40x.

## Shared resources across invocations

Lambda reuses execution environments for warm invocations. Anything you initialize in `main()` before the handler loop persists across invocations. This is important for:

- **SDK clients**: HTTP connection pools survive between invocations
- **Database connections**: If you're using RDS with IAM auth, the connection persists
- **Loaded configuration**: Environment variables, secrets from SSM/Secrets Manager
- **In-memory caches**: Small lookup tables, compiled regexes

```rust
use std::sync::OnceLock;
use regex::Regex;

static EMAIL_REGEX: OnceLock<Regex> = OnceLock::new();

fn validate_email(email: &str) -> bool {
    let re = EMAIL_REGEX.get_or_init(|| {
        Regex::new(r"^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$").unwrap()
    });
    re.is_match(email)
}
```

The regex compiles once on the first invocation, then every subsequent warm invocation gets the compiled version for free. This pattern matters more in Rust than in Python or Node because Rust's initialization costs are already low - but it still adds up at high throughput.

## Error handling and observability

Lambda captures anything written to stdout/stderr as CloudWatch Logs. The `tracing` crate with `tracing-subscriber` gives you structured logging out of the box:

```rust
use lambda_runtime::diagnostic::Diagnostic;
use std::fmt;

#[derive(Debug)]
enum ApiError {
    NotFound(String),
    BadRequest(String),
    Internal(String),
}

impl fmt::Display for ApiError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            ApiError::NotFound(msg) => write!(f, "not found: {msg}"),
            ApiError::BadRequest(msg) => write!(f, "bad request: {msg}"),
            ApiError::Internal(msg) => write!(f, "internal: {msg}"),
        }
    }
}

impl std::error::Error for ApiError {}

impl From<ApiError> for Diagnostic {
    fn from(err: ApiError) -> Self {
        let error_type = match &err {
            ApiError::NotFound(_) => "NotFound",
            ApiError::BadRequest(_) => "BadRequest",
            ApiError::Internal(_) => "Internal",
        };
        Diagnostic {
            error_type: error_type.to_string().into(),
            error_message: err.to_string().into(),
        }
    }
}
```

The `Diagnostic` struct is Lambda-specific - it maps to the error response format that shows up in CloudWatch and X-Ray. For deeper tracing, the runtime supports OpenTelemetry through the `opentelemetry-tracing` example in the repo.

For production error handling, you can also enable the `anyhow` or `eyre` feature flags on `lambda_runtime` to use those crates directly as error types:

```toml
lambda_runtime = { version = "0.14", features = ["anyhow"] }
```

## Graceful shutdown

When Lambda recycles your execution environment, it sends SIGTERM. The runtime supports catching this:

```rust
use lambda_runtime::spawn_graceful_shutdown_handler;

#[tokio::main]
async fn main() -> Result<(), Error> {
    // Register shutdown handler before starting the runtime
    spawn_graceful_shutdown_handler(|| async {
        tracing::info!("Shutting down, flushing buffers...");
        // Close database connections, flush metrics, etc.
    });

    run(service_fn(handler)).await
}
```

This requires the `graceful-shutdown` feature flag. The function registers a no-op Lambda Extension internally to receive the shutdown event from the Lambda platform.

## When Lambda makes sense for Rust

**Good fits:**
- API backends with variable traffic (pay-per-request beats pay-per-hour)
- Event processing pipelines (SQS, S3 triggers, EventBridge)
- Scheduled tasks that run briefly (cron-like jobs via CloudWatch Events)
- Webhook receivers (GitHub, Stripe, Slack integrations)
- Low-traffic microservices where running a full server is overkill

**Bad fits:**
- WebSocket connections (Lambda has a 15-minute max execution time)
- Long-running data processing (use Fargate or EC2)
- Anything that needs persistent local state (Lambda is ephemeral)
- Ultra-low-latency requirements where even 10ms cold starts matter (use a long-running server)
- High-throughput services above ~100M requests/month (EC2/Fargate is cheaper)

The sweet spot: you have a Rust API that handles anywhere from 0 to 50M requests/month, traffic is spiky or unpredictable, and you don't want to manage infrastructure. Lambda with Rust gives you single-digit millisecond cold starts, sub-5ms execution times for typical CRUD operations, and a bill that scales linearly with actual usage.

## The development workflow

Putting it all together, here's the workflow that works in practice:

```bash
# 1. Create the project
cargo lambda new my-service --http-feature apigw_rest

# 2. Develop locally (hot reload)
cargo lambda watch
# In another terminal: curl http://localhost:9000/products

# 3. Run tests
cargo test

# 4. Build for production
cargo lambda build --release --arm64

# 5. Deploy
cargo lambda deploy my-service --iam-role $ROLE_ARN
# Or: sam deploy / cdk deploy
```

No Docker in the loop. No emulator containers. The compile-test-deploy cycle is fast enough that serverless doesn't feel like a compromise - it feels like less infrastructure to worry about.

The Rust Lambda ecosystem has matured to the point where cold starts aren't a conversation anymore. They're just fast. The real question is whether serverless fits your architecture, and with Rust's execution speed and memory efficiency, the cost threshold where it stops making sense is much higher than with any other runtime.
