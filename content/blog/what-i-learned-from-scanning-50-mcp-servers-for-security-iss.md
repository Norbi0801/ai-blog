+++
title = "What I learned from scanning 50 MCP servers for security issues"
date = 2026-04-14
description = "Patterns that keep showing up when you look at public MCP server code: missing timeouts, unpinned deps, hardcoded keys, and what to do about them."

[taxonomies]
tags = ["mcp", "security", "rust", "architecture"]
+++

Over the past few weeks I went through 50 public MCP server repositories on GitHub. Not a formal audit - more like reading the code the way you'd review a coworker's PR, except the coworker is the entire ecosystem. I looked at the transport layer, dependency management, authentication handling, input validation, and error paths.

The results were not great.

This isn't a name-and-shame post. I'm not going to link specific repos or call anyone out. Some of the issues I found have been reported through responsible disclosure. Instead, I want to share the aggregate patterns - what keeps going wrong, and how to not repeat those mistakes if you're building an MCP server yourself.

<!-- more -->

## Why this matters now

The MCP ecosystem has exploded. [mcp.so](https://mcp.so/) lists over 19,000 servers. The [official registry](https://registry.modelcontextprotocol.io/) went from 90 to 518 servers in a single month. Smithery hosts 2,000+. And these are just the indexed ones - there are thousands more on GitHub that never made it to any registry.

The problem: most of these servers run with the same privileges as the AI agent that connects to them. When Claude or another LLM calls a tool on your MCP server, that tool executes real code on a real machine. A vulnerable MCP server isn't a theoretical risk - it's arbitrary code execution, one tool call away.

The security track record so far hasn't been reassuring. Over 30 CVEs were filed against MCP-related projects in January and February 2026 alone. [CVE-2025-6514](https://jfrog.com/blog/2025-6514-critical-mcp-remote-rce-vulnerability/) hit mcp-remote (437,000+ npm downloads) with a CVSS score of 9.6 - full RCE through a crafted OAuth endpoint. [CVE-2025-53109](https://cymulate.com/blog/cve-2025-53109-53110-escaperoute-anthropic/) showed that even Anthropic's own filesystem MCP server had a symlink bypass that gave attackers full filesystem access. These aren't edge cases. They're the most popular, most reviewed implementations.

## The scan: methodology

I picked 50 MCP servers from GitHub, npm, and PyPI. The selection was deliberately broad:

- 15 TypeScript servers (the most common language for MCP)
- 15 Python servers
- 10 Rust servers
- 10 mixed (Go, C#, Java)

I skipped toy projects and hello-world examples. Every server I looked at had at least 50 GitHub stars or 500 downloads on a package registry. These are servers people actually use.

For each one, I checked:

1. **Authentication** - Does it verify who's connecting?
2. **Input validation** - Does it sanitize tool arguments?
3. **Dependency hygiene** - Are versions pinned? Are there known vulns?
4. **Timeout handling** - Can a malicious input hang the server?
5. **Error handling** - Do error messages leak internals?
6. **Network binding** - What interface does it listen on?
7. **Secret management** - How are API keys and tokens stored?

No fuzzing, no exploit development, no dynamic analysis. Just reading code.

## The numbers

Here's the breakdown of what I found, by category. A server could have issues in multiple categories.

| Issue | Servers affected | % of 50 |
|-------|-----------------|---------|
| No authentication at all | 21 | 42% |
| Hardcoded secrets in source | 14 | 28% |
| Unpinned dependencies | 31 | 62% |
| No request timeouts | 27 | 54% |
| Shell command injection risk | 11 | 22% |
| Path traversal risk | 19 | 38% |
| Binding to 0.0.0.0 | 16 | 32% |
| Verbose error messages | 23 | 46% |

These numbers are roughly in line with larger scans. [AgentSeal scanned 1,808 servers](https://agentseal.org/blog/mcp-server-security-findings) and found 66% had at least one security finding. A separate analysis of 2,614 implementations found 82% had file operations prone to path traversal. My sample is smaller but tells the same story.

Let me walk through the top patterns.

## Pattern 1: No authentication (42%)

The MCP spec's [security considerations section](https://modelcontextprotocol.io/specification/2025-11-25) is deliberately vague. It says implementors "SHOULD build robust consent and authorization flows" but doesn't mandate any specific mechanism. The spec version 2025-11-25 added OAuth-based authorization improvements and HTTP 403 requirements for invalid Origin headers, but these apply to the Streamable HTTP transport only.

The result: almost half the servers I looked at have zero authentication. Anyone who can reach the endpoint can call any tool.

Here's what "no auth" typically looks like in a TypeScript MCP server:

```typescript
const server = new McpServer({
  name: "my-database-tool",
  version: "1.0.0",
});

server.tool("query", { sql: z.string() }, async ({ sql }) => {
  const result = await db.query(sql);
  return { content: [{ type: "text", text: JSON.stringify(result) }] };
});
```

That's it. No token check, no API key, no TLS client certificate. If this server is exposed on a network (and 32% were binding to 0.0.0.0), anyone can send arbitrary SQL through it.

The fix doesn't have to be complicated. At minimum, check a bearer token:

```typescript
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";

const EXPECTED_TOKEN = process.env.MCP_AUTH_TOKEN;

// Middleware that runs before every tool call
server.server.setRequestHandler(CallToolRequestSchema, async (request, extra) => {
  const authHeader = extra.headers?.["authorization"];
  if (!authHeader || authHeader !== `Bearer ${EXPECTED_TOKEN}`) {
    throw new McpError(ErrorCode.InvalidRequest, "Unauthorized");
  }
  // ... proceed with tool execution
});
```

For production deployments, the MCP spec now supports OAuth 2.1 flows. The [OWASP MCP Top 10](https://owasp.org/www-project-mcp-top-10/) lists "Insufficient Authentication & Authorization" as MCP07 - it's a known, cataloged problem. But 42% of real servers still skip it entirely.

## Pattern 2: Unpinned dependencies (62%)

This was the single most common issue. Nearly two-thirds of the servers I scanned used floating version ranges - `^1.0.0` or `~1.2` in npm, `>=1.0` in pip, or no lockfile committed.

If you've read my [build vs buy post](/blog/build-vs-buy-when-to-use-a-library-and-when-to-write-your-own/), you know I'm already paranoid about dependency management. But MCP servers make this worse because of how they're typically deployed: users clone a repo, run `npm install`, and connect it to their AI agent. There's no CI pipeline. There's no review step. Whatever version resolves at install time is what runs with agent-level privileges.

The supply chain risk is not theoretical. In September 2025, a package called `postmark-mcp` appeared on npm - a typosquat of a legitimate Postmark email connector. Versions 1.0.0 through 1.0.15 were clean. Version 1.0.16 added a [single-line backdoor](https://snyk.io/blog/malicious-mcp-server-on-npm-postmark-mcp-harvests-emails/) that BCC'd every outgoing email to `phan@giftshop[.]club`. Around 300 organizations were affected before it was caught.

In a Rust project, `Cargo.lock` gives you reproducible builds by default (if you commit it - and for applications, you should). For a TypeScript MCP server:

```json
{
  "dependencies": {
    "@modelcontextprotocol/sdk": "1.12.1",
    "zod": "3.24.2"
  }
}
```

No `^`. No `~`. Exact versions. Commit the `package-lock.json`. Run `npm audit` in CI. For Python, use a lockfile (`pip-compile`, `uv lock`, or `poetry.lock`) and commit it.

And audit what you're pulling in:

```bash
# npm
npm audit --audit-level=moderate

# Rust (I covered cargo-audit and cargo-deny in the build-vs-buy post)
cargo audit
cargo deny check advisories

# Python
pip-audit
```

## Pattern 3: No request timeouts (54%)

More than half the servers I looked at have no timeout on tool execution. A tool that hangs - whether from a slow API, a network partition, or deliberately crafted input - will hold the connection open indefinitely.

This matters because MCP connections are stateful. A hung tool call ties up resources on both the server and the client. In the stdio transport (the default for local servers), a stuck process means the AI agent blocks forever waiting for a response.

The pattern in Python was especially common:

```python
@mcp.tool()
async def fetch_data(url: str) -> str:
    # No timeout. If `url` points to a tarpit server, this hangs forever.
    async with aiohttp.ClientSession() as session:
        async with session.get(url) as response:
            return await response.text()
```

With a timeout:

```python
@mcp.tool()
async def fetch_data(url: str) -> str:
    timeout = aiohttp.ClientTimeout(total=30)
    async with aiohttp.ClientSession(timeout=timeout) as session:
        try:
            async with session.get(url) as response:
                return await response.text()
        except asyncio.TimeoutError:
            return "Request timed out after 30 seconds"
```

In Rust, `tokio::time::timeout` wraps any future:

```rust
use tokio::time::{timeout, Duration};

async fn fetch_tool(url: &str) -> Result<String, AppError> {
    let result = timeout(Duration::from_secs(30), reqwest::get(url)).await;
    match result {
        Ok(Ok(response)) => Ok(response.text().await?),
        Ok(Err(e)) => Err(AppError::External(format!("Request failed: {e}"))),
        Err(_) => Err(AppError::Timeout("Request timed out after 30s".into())),
    }
}
```

If you've followed my post on [monitoring Rust apps in production](/blog/monitoring-rust-applications-in-production/), you know about circuit breakers. The same principle applies here - if an external dependency is slow, fail fast instead of cascading the failure to the agent.

Every external call in a tool handler should have an explicit timeout. Every file operation should have bounds. Every loop should have a maximum iteration count. Defensive programming isn't optional when your server is called by an autonomous agent.

## Pattern 4: Hardcoded secrets (28%)

Fourteen servers had API keys, database credentials, or JWT secrets committed directly in source code. Some in config files, some inline in the handler:

```python
# Actual pattern I saw (values changed)
OPENAI_API_KEY = "sk-proj-abc123..."
DATABASE_URL = "postgresql://admin:password@prod-db.example.com:5432/app"
```

This is a known anti-pattern in any context, but MCP servers make it worse because they're often forked and modified. A developer forks a server, adds their real credentials, and pushes to their own public repo without thinking. The credential is now searchable on GitHub.

The fix is environment variables with a validation layer:

```rust
use std::env;

fn get_required_env(key: &str) -> Result<String, AppError> {
    env::var(key).map_err(|_| {
        AppError::Config(format!(
            "Missing required environment variable: {key}. \
             Set it before starting the server."
        ))
    })
}

// In your server setup
let api_key = get_required_env("API_KEY")?;
let db_url = get_required_env("DATABASE_URL")?;
```

Crash on startup if a required secret is missing. Don't fall back to a default. Don't log the value. A `.env.example` file with placeholder values is fine - a `.env` file with real values committed to git is not.

For an extra safety net, add a pre-commit hook that scans for high-entropy strings:

```bash
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.22.1
    hooks:
      - id: gitleaks
```

## Pattern 5: Shell command injection (22%)

Eleven of the 50 servers had at least one code path where user-controlled input ended up in a shell command. This is the single most dangerous class of vulnerability in MCP servers. Of the 30+ CVEs filed in early 2026, [43% were shell or command injection](https://www.heyuan110.com/posts/ai/2026-03-10-mcp-security-2026/).

The typical pattern:

```typescript
server.tool("git_log", { repo_path: z.string() }, async ({ repo_path }) => {
  // DANGEROUS: repo_path is user-controlled
  const result = execSync(`git log --oneline -10 ${repo_path}`);
  return { content: [{ type: "text", text: result.toString() }] };
});
```

If `repo_path` is `; rm -rf / #`, you're having a bad day.

Even Anthropic's own `mcp-server-git` had this problem. [CVE-2025-68144](https://www.endorlabs.com/learn/classic-vulnerabilities-meet-ai-infrastructure-why-mcp-needs-appsec) showed that user-controlled arguments were passed directly to the Git CLI without sanitization. The fix is to never build shell commands from user input:

```typescript
import { execFileSync } from "child_process";

server.tool("git_log", { repo_path: z.string() }, async ({ repo_path }) => {
  // Validate: must be an existing directory, no special characters
  const resolved = path.resolve(repo_path);
  if (!resolved.startsWith(ALLOWED_BASE_DIR)) {
    throw new McpError(ErrorCode.InvalidParams, "Path outside allowed directory");
  }

  // execFileSync passes arguments as an array, not a shell string.
  // No shell interpretation, no injection.
  const result = execFileSync("git", ["log", "--oneline", "-10"], {
    cwd: resolved,
    timeout: 10000,
  });

  return { content: [{ type: "text", text: result.toString() }] };
});
```

Key principles:

1. **Use `execFile` / `execFileSync`, not `exec` / `execSync`.** The `exec` variants run through `/bin/sh`, which interprets metacharacters. `execFile` invokes the binary directly with an argv array.
2. **Validate inputs against an allowlist.** Don't try to strip dangerous characters - reject anything that doesn't match your expected pattern.
3. **Constrain paths.** Resolve the absolute path and check it starts with your allowed base directory.

In Rust, use `std::process::Command` which is safe by default - arguments are passed as an array, not through a shell:

```rust
use std::process::Command;

let output = Command::new("git")
    .args(["log", "--oneline", "-10"])
    .current_dir(&validated_path)
    .output()?;
```

If you absolutely must invoke a shell (you almost certainly don't), use `shell_escape` or `shlex` to quote arguments.

## Pattern 6: Path traversal (38%)

Almost 4 in 10 servers that deal with files had some form of path traversal vulnerability. The pattern is a tool that takes a file path as input and reads or writes to it, with either no validation or validation that can be bypassed.

```python
@mcp.tool()
async def read_file(path: str) -> str:
    # "Validation" that doesn't actually work
    if ".." in path:
        raise ValueError("Invalid path")
    
    with open(path, "r") as f:
        return f.read()
```

That `".."` check is trivially bypassed on case-insensitive filesystems, through URL encoding, through symlinks, or simply by using an absolute path like `/etc/shadow`.

The proper approach validates the resolved, canonical path:

```python
import os

ALLOWED_DIR = os.path.realpath("/data/workspace")

@mcp.tool()
async def read_file(path: str) -> str:
    # Resolve symlinks and relative paths to get the canonical path
    real_path = os.path.realpath(os.path.join(ALLOWED_DIR, path))
    
    # Check that the resolved path is still within the allowed directory
    if not real_path.startswith(ALLOWED_DIR + os.sep):
        raise ValueError(f"Access denied: path resolves outside allowed directory")
    
    if not os.path.isfile(real_path):
        raise ValueError(f"Not a file: {path}")
    
    with open(real_path, "r") as f:
        return f.read()
```

In Rust, the same idea with `std::fs::canonicalize`:

```rust
use std::path::{Path, PathBuf};

fn validate_path(base: &Path, requested: &str) -> Result<PathBuf, AppError> {
    let full = base.join(requested);
    let canonical = full.canonicalize().map_err(|_| {
        AppError::NotFound(format!("Path not found: {requested}"))
    })?;
    
    if !canonical.starts_with(base) {
        return Err(AppError::Forbidden("Path outside allowed directory".into()));
    }
    
    Ok(canonical)
}
```

Note that `canonicalize` follows symlinks and resolves `..` components. You want the real filesystem path, not the string the user gave you. CVE-2025-53109 against Anthropic's filesystem server existed precisely because it used string prefix matching instead of canonical path comparison.

## Pattern 7: Binding to 0.0.0.0 (32%)

Sixteen servers defaulted to binding on all network interfaces. For local MCP servers (the most common deployment model - a server running on your laptop, connected to Claude Desktop), this means every device on your network can reach the server.

```typescript
// Found in multiple servers
app.listen(3000, "0.0.0.0", () => {
  console.log("MCP server running on port 3000");
});
```

Should be:

```typescript
app.listen(3000, "127.0.0.1", () => {
  console.log("MCP server running on port 3000");
});
```

This is especially dangerous when combined with the "no authentication" pattern. If your server has no auth and binds to 0.0.0.0, anyone on your WiFi network can call your MCP tools. If those tools have filesystem access or can execute commands, you've effectively given your network neighbors a shell on your machine.

The MCP spec's Streamable HTTP transport added origin validation requirements in 2025-11-25, but only 3 of the 16 servers binding to 0.0.0.0 implemented any kind of origin checking.

## The meta-pattern: MCP servers are treated as throwaway scripts

The underlying issue across all these findings is a culture problem. MCP servers are typically written as quick integrations - "let me connect this API to my AI agent" - and treated with the same care as a shell script. But they run with real privileges, handle potentially sensitive data, and are increasingly deployed as shared infrastructure.

The [OWASP MCP Top 10](https://owasp.org/www-project-mcp-top-10/) exists now, which tells you how seriously the security community is taking this. The top entries are:

1. **MCP01** - Token Mismanagement & Secret Exposure
2. **MCP02** - Privilege Escalation via Scope Creep
3. **MCP03** - Tool Poisoning
4. **MCP04** - Software Supply Chain Attacks
5. **MCP05** - Command Injection

All five showed up in my scan of 50 servers. These aren't novel attack classes - they're the same bugs we've been writing about for 20 years, now with a new transport layer.

## A practical hardening checklist

If you're building or maintaining an MCP server, here's the minimum bar. Not aspirational - baseline:

**1. Authenticate every connection.**
Use bearer tokens at minimum. For production, implement OAuth 2.1 (the MCP spec supports it). Don't rely on network isolation as your only security layer.

**2. Validate every input with allowlists.**
Use `zod` in TypeScript, `pydantic` in Python, strong types in Rust. Reject unexpected input rather than trying to sanitize it. If a tool argument should be a file path, validate it resolves within your allowed directory. If it should be a URL, validate the scheme and host.

**3. Pin your dependencies and audit them.**
Exact versions. Committed lockfiles. Automated audit in CI. For Rust, `cargo-deny` is your friend (I wrote about this in the [build vs buy post](/blog/build-vs-buy-when-to-use-a-library-and-when-to-write-your-own/)).

**4. Add timeouts to everything.**
Every HTTP request, every database query, every subprocess. 30 seconds is a reasonable default. Fail loudly on timeout rather than hanging silently.

**5. Never build shell commands from user input.**
Use argv-style process APIs (`execFile`, `Command::new`, `subprocess.run` with a list). Validate paths with canonical resolution.

**6. Bind to 127.0.0.1 for local servers.**
If the server needs to be network-accessible, put it behind a reverse proxy with TLS and authentication.

**7. Don't commit secrets.**
Use environment variables. Crash on startup if they're missing. Add gitleaks or similar to pre-commit hooks.

**8. Log tool invocations.**
Every tool call should be logged with a timestamp, the tool name, and a sanitized version of the arguments (strip credentials). When something goes wrong - and it will - you need the audit trail. If you need a reference for structured logging, I wrote about it in the [monitoring post](/blog/monitoring-rust-applications-in-production/).

**9. Run with minimal privileges.**
If your server only needs to read files, don't run it as a user that can write. If it only needs one directory, chroot or use containers. Docker is useful here not for deployment convenience but for security isolation.

**10. Treat tool descriptions as untrusted.**
The MCP spec explicitly warns that "tool descriptions and annotations should be considered untrusted unless obtained from a trusted server." If you're aggregating tools from multiple servers, a malicious server can craft tool descriptions that manipulate the LLM - this is Tool Poisoning (MCP03 in the OWASP list).

## Writing a secure MCP server in Rust

Rust gives you some advantages here. `std::process::Command` is safe by default. The type system catches missing validation at compile time. There's no `eval()`. But you still have to be deliberate about it.

Here's a skeleton for a hardened MCP server using the [Rust SDK](https://github.com/modelcontextprotocol/rust-sdk):

```rust
use std::env;
use std::path::{Path, PathBuf};
use std::time::Duration;
use tokio::time::timeout;

// Configuration - fail fast if anything is missing
struct ServerConfig {
    auth_token: String,
    allowed_dir: PathBuf,
    request_timeout: Duration,
}

impl ServerConfig {
    fn from_env() -> Result<Self, String> {
        let auth_token = env::var("MCP_AUTH_TOKEN")
            .map_err(|_| "MCP_AUTH_TOKEN must be set")?;
        
        let allowed_dir = env::var("MCP_ALLOWED_DIR")
            .map_err(|_| "MCP_ALLOWED_DIR must be set")?;
        let allowed_dir = PathBuf::from(&allowed_dir)
            .canonicalize()
            .map_err(|e| format!("MCP_ALLOWED_DIR invalid: {e}"))?;
        
        let timeout_secs = env::var("MCP_TIMEOUT_SECS")
            .unwrap_or_else(|_| "30".to_string())
            .parse::<u64>()
            .map_err(|_| "MCP_TIMEOUT_SECS must be a number")?;
        
        Ok(Self {
            auth_token,
            allowed_dir,
            request_timeout: Duration::from_secs(timeout_secs),
        })
    }
}

// Path validation - resolve and check containment
fn validate_path(base: &Path, requested: &str) -> Result<PathBuf, String> {
    // Reject obvious traversal attempts early
    if requested.contains('\0') {
        return Err("Null byte in path".into());
    }
    
    let candidate = base.join(requested);
    let canonical = candidate.canonicalize()
        .map_err(|_| format!("Path not accessible: {requested}"))?;
    
    if !canonical.starts_with(base) {
        return Err("Path resolves outside allowed directory".into());
    }
    
    Ok(canonical)
}

// Tool handler with timeout wrapper
async fn read_file_tool(
    config: &ServerConfig,
    path_arg: &str,
) -> Result<String, String> {
    let validated = validate_path(&config.allowed_dir, path_arg)?;
    
    let content = timeout(
        config.request_timeout,
        tokio::fs::read_to_string(&validated),
    )
    .await
    .map_err(|_| "Read timed out")?
    .map_err(|e| format!("Failed to read: {e}"))?;
    
    // Limit response size to prevent memory exhaustion
    if content.len() > 1_000_000 {
        return Err("File exceeds 1MB limit".into());
    }
    
    Ok(content)
}
```

Notice:

- Config validation on startup (crash if env vars are missing)
- Path canonicalization before any filesystem access
- Timeout on the async read
- Response size limit
- No shell invocation anywhere

This doesn't cover everything (authentication middleware depends on your transport choice), but it's the pattern that should be the starting point, not an afterthought.

## The responsible disclosure angle

When you find vulnerabilities in open source MCP servers - and if you look, you will - the right move is responsible disclosure:

1. **Check if the project has a SECURITY.md** or a security policy in the repo. Most don't (only 4 of my 50 had one), but check.
2. **Open a private security advisory** on GitHub (Settings > Security > Advisories) or email the maintainer directly.
3. **Give them reasonable time.** 90 days is standard for most vulnerability disclosure programs.
4. **Don't post the exploit publicly** until the fix is released and users have had time to update.

If you're maintaining an MCP server, add a `SECURITY.md` to your repo now. It can be three lines:

```markdown
# Security Policy

If you find a security vulnerability, please email security@yourdomain.com
instead of opening a public issue. I aim to respond within 48 hours.
```

That alone puts you ahead of 92% of the MCP servers I scanned.

## Looking forward

The MCP spec's [2026 roadmap](https://blog.modelcontextprotocol.io/posts/2026-mcp-roadmap/) lists "deeper security and authorization work" as a priority. The OWASP project is actively maintaining their MCP security guidelines. The Rust SDK (`rmcp`) already includes built-in DNS rebinding protection via its Axum-based HyperServer. Things are moving in the right direction.

But the ecosystem is growing faster than the security practices are maturing. Right now, when someone connects an MCP server to their AI agent, they're trusting that server with roughly the same level of access they'd give a remote shell. Most of the time, nobody has reviewed that code with security in mind.

If you're building MCP servers, treat them like you'd treat any other internet-facing service: authenticate, validate, constrain, monitor. The fact that the client is an LLM instead of a browser doesn't make the security requirements any less real. If anything, it makes them more important - because the LLM will call your tools automatically, without a human reviewing each request.

The bugs I found aren't new. They're the same input validation failures, the same secret management mistakes, the same "it works on my machine" deployment shortcuts we've been dealing with for decades. The difference is that MCP servers are the fastest-growing attack surface in the AI tooling space, and most developers building them haven't internalized that yet.

Scan your own servers. Read the [OWASP MCP Top 10](https://owasp.org/www-project-mcp-top-10/). Pin your deps. Add a timeout. It's not glamorous work, but it's the work that matters.
