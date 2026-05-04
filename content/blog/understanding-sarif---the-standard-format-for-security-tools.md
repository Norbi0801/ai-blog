+++
title = "Understanding SARIF - the standard format for security tools"
date = 2026-03-31
description = "SARIF is the JSON standard that lets security scanners talk to GitHub, VS Code, and each other - here is how it works and how to generate it from your own tools."

[taxonomies]
tags = ["security", "rust", "devops", "tooling"]
+++

Every security scanner has its own output format. Clippy gives you compiler diagnostics. Semgrep dumps JSON. Trivy has its own table format, or JSON, or a template system. Bandit outputs yet another JSON shape. If you run three tools in your CI pipeline, you get three different formats, three different parsers, and three different ways of saying "there's a SQL injection on line 42."

This is the problem SARIF solves. One format. Every tool writes it, every consumer reads it. GitHub Code Scanning, VS Code, Azure DevOps, SonarQube - they all understand SARIF natively.

SARIF (Static Analysis Results Interchange Format) is an [OASIS standard](https://docs.oasis-open.org/sarif/sarif/v2.1.0/sarif-v2.1.0.html) - the same standards body behind MQTT, OData, and STIX. Version 2.1.0 was approved in 2020 and is the current version used everywhere. It's a JSON schema, not a binary protocol, so you can read and write it with any language, any JSON library, no special SDK required.

<!-- more -->

## Why you should care

If you're building developer tools - linters, scanners, audit scripts, anything that finds problems in code - SARIF is how you get your results into the places developers actually look. Without SARIF, your tool lives in a terminal window. With SARIF, your findings show up as:

- **GitHub Security alerts** in the Security tab of any repository
- **Inline PR annotations** that point to the exact line of code
- **VS Code squiggles** through the [SARIF Viewer extension](https://github.com/microsoft/sarif-vscode-extension)
- **Dashboard aggregations** across teams in GitHub Advanced Security or Azure DevOps

The format is also useful for non-security tools. Any static analysis - style checks, performance warnings, deprecation notices - fits the SARIF model. The name says "static analysis," not "security."

## Anatomy of a SARIF file

A SARIF file is a JSON object with a fixed top-level shape. Here's the smallest valid SARIF file:

```json
{
  "$schema": "https://raw.githubusercontent.com/oasis-tcs/sarif-spec/main/sarif-2.1/schema/sarif-schema-2.1.0.json",
  "version": "2.1.0",
  "runs": []
}
```

Three required fields. `$schema` points to the JSON Schema definition (tools use this for validation). `version` must be `"2.1.0"`. `runs` is an array of analysis runs - each run represents one invocation of one tool.

An empty `runs` array is valid but useless. Here's what a real run looks like, with one result:

```json
{
  "$schema": "https://raw.githubusercontent.com/oasis-tcs/sarif-spec/main/sarif-2.1/schema/sarif-schema-2.1.0.json",
  "version": "2.1.0",
  "runs": [
    {
      "tool": {
        "driver": {
          "name": "my-secret-scanner",
          "version": "0.1.0",
          "informationUri": "https://github.com/you/my-secret-scanner",
          "rules": [
            {
              "id": "SECRET001",
              "shortDescription": {
                "text": "Hardcoded API key detected"
              },
              "fullDescription": {
                "text": "A string matching common API key patterns was found in source code. Hardcoded secrets should be moved to environment variables or a secrets manager."
              },
              "helpUri": "https://owasp.org/Top10/A07_2021-Identification_and_Authentication_Failures/",
              "defaultConfiguration": {
                "level": "error"
              },
              "properties": {
                "security-severity": "8.5"
              }
            }
          ]
        }
      },
      "results": [
        {
          "ruleId": "SECRET001",
          "ruleIndex": 0,
          "level": "error",
          "message": {
            "text": "Possible AWS access key found: AKIA... (20 characters matching [A-Z0-9]{20} after 'AKIA' prefix)"
          },
          "locations": [
            {
              "physicalLocation": {
                "artifactLocation": {
                  "uri": "src/config.rs",
                  "uriBaseId": "%SRCROOT%"
                },
                "region": {
                  "startLine": 14,
                  "startColumn": 28,
                  "endLine": 14,
                  "endColumn": 48
                }
              }
            }
          ],
          "partialFingerprints": {
            "primaryLocationLineHash": "a1b2c3d4e5f6:1"
          }
        }
      ]
    }
  ]
}
```

Let me walk through the key pieces.

### The `tool` object

Every run has exactly one tool. The `driver` is the main analysis engine. It declares:

- `name` - what GitHub shows in the Security tab as the source of alerts
- `version` - your tool's version
- `rules` - the full catalog of checks your tool implements

The `rules` array is where GitHub (and VS Code) pulls the description, severity, and help text for each finding. Even if a rule didn't fire in this run, you can include it in the array to advertise all your tool's capabilities.

There's also a `tool.extensions` array for plugins that add rules on top of the driver. CodeQL uses this to separate its core engine from language-specific query packs. For most tools, `driver` alone is enough.

### The `results` array

Each result is one finding - one problem at one location. The critical fields:

**`ruleId`** links the result back to a rule in `tool.driver.rules`. GitHub uses this to group duplicate findings across runs. If your tool finds the same issue in two different commits, `ruleId` (combined with fingerprinting) is how GitHub knows it's the same alert, not two new ones.

**`level`** is the severity: `"error"`, `"warning"`, or `"note"`. GitHub maps these to High, Medium, and Low severity. For security findings, you can also set `properties.security-severity` on the rule (a float from 0.0 to 10.0, following CVSS scoring) - GitHub uses this for finer-grained severity sorting.

**`message.text`** is the human-readable description shown in the alert. Be specific. Don't write "security issue found" - write "Possible AWS access key found: AKIA... (20 characters matching pattern)." The message is the first thing a developer reads when triaging your finding.

**`locations`** is where SARIF gets interesting. Each location has a `physicalLocation` with:

- `artifactLocation.uri` - the file path, relative to the repository root
- `region` - the exact lines and columns

The `region` object supports `startLine`, `startColumn`, `endLine`, `endColumn`. Lines are 1-indexed. Columns are 1-indexed. `endColumn` is exclusive (points one past the last character). If you only know the line, you can omit columns - GitHub will highlight the entire line.

`uriBaseId` with value `"%SRCROOT%"` tells consumers the path is relative to the source root. This matters when your tool runs in a subdirectory or inside a container where absolute paths don't match the repo structure.

### Fingerprinting

This is the field most custom tools get wrong, leading to duplicate alerts on every push.

`partialFingerprints.primaryLocationLineHash` should be a stable hash of the finding's identity. GitHub uses it to deduplicate results across runs. If you omit it and upload via the `upload-sarif` GitHub Action, GitHub calculates it from the file content around the finding. But if you upload via the REST API without fingerprints, you'll see duplicate alerts pile up.

A good fingerprint strategy: hash the rule ID + file path + the line content (not the line number, because line numbers shift when code above the finding changes). This way, if someone adds a blank line at the top of the file, the fingerprint stays the same and GitHub matches it to the existing alert.

```
SHA256(ruleId + ":" + filePath + ":" + lineContent) -> take first 16 hex chars
```

### Optional but useful fields

**`codeFlows`** describes the execution path that leads to a finding. For taint analysis (data flows from source to sink), this shows the developer exactly how user input reaches a dangerous function. GitHub renders these as collapsible step-by-step traces in the alert view.

**`relatedLocations`** points to other relevant code. If a function is called unsafely at line 42, but the function definition on line 100 is also relevant, you'd add line 100 as a related location. GitHub hyperlinks these.

**`fixes`** contains suggested code changes. Each fix has a `description` and an array of `artifactChanges` with `replacements`. Tools like CodeQL use this to offer one-click fixes in the GitHub UI.

**`artifacts`** lists all files that were scanned (not just files with findings). This lets consumers distinguish "we scanned this file and it's clean" from "we didn't scan this file at all."

## GitHub Code Scanning integration

GitHub is the biggest consumer of SARIF. There are two ways to get your SARIF into a repository's Security tab.

### Method 1: GitHub Actions with upload-sarif

The `github/codeql-action/upload-sarif` action takes a SARIF file and uploads it:

```yaml
name: Security Scan
on:
  push:
    branches: [main]
  pull_request:

jobs:
  scan:
    runs-on: ubuntu-latest
    permissions:
      security-events: write
      contents: read
    steps:
      - uses: actions/checkout@v5

      - name: Run custom scanner
        run: ./my-scanner --output results.sarif

      - name: Upload SARIF
        uses: github/codeql-action/upload-sarif@v4
        with:
          sarif_file: results.sarif
          category: my-scanner
```

The `category` parameter matters when you run multiple tools. Each category gets its own set of alerts. Without it, uploading results from tool B would replace results from tool A on the same commit.

The `permissions` block is required. `security-events: write` grants access to the code scanning API. On private repositories, you also need `actions: read`.

The action handles gzip compression (SARIF uploads are limited to 10 MB compressed) and calculates `partialFingerprints` if your SARIF doesn't include them.

### Method 2: REST API

For CI systems that aren't GitHub Actions:

```bash
# Compress the SARIF file
gzip -c results.sarif > results.sarif.gz

# Base64 encode it
SARIF_CONTENT=$(base64 -w0 results.sarif.gz)

# Upload via the API
curl -X POST \
  -H "Authorization: Bearer $GITHUB_TOKEN" \
  -H "Accept: application/vnd.github+json" \
  "https://api.github.com/repos/OWNER/REPO/code-scanning/sarifs" \
  -d "{
    \"commit_sha\": \"$(git rev-parse HEAD)\",
    \"ref\": \"refs/heads/main\",
    \"sarif\": \"$SARIF_CONTENT\"
  }"
```

The API returns a `202 Accepted` with a URL to check processing status. SARIF processing is async - it can take a few seconds for alerts to appear.

### Limits to know

GitHub enforces hard limits on SARIF uploads:

| Limit | Value |
|---|---|
| File size (gzip compressed) | 10 MB |
| Runs per file | 20 |
| Results per run | 25,000 (top 5,000 shown) |
| Rules per run | 25,000 |
| Thread flow locations per result | 10,000 (top 1,000 shown) |
| Locations per result | 1,000 (top 100 shown) |

If your tool produces more than 25,000 results per run, GitHub processes all of them for deduplication but only displays the top 5,000, prioritized by severity. In practice, if your scanner is producing 25,000 findings, the signal-to-noise ratio needs work before the output format does.

## Building a secret scanner that outputs SARIF

Theory is fine. Let's build something. Here's a minimal secret scanner in Rust that checks for hardcoded API keys and outputs valid SARIF. No special SARIF crate needed - just `serde_json`.

```rust
use serde_json::{json, Value};
use std::fs;
use std::path::{Path, PathBuf};

#[derive(Debug)]
struct Finding {
    rule_id: String,
    message: String,
    file: PathBuf,
    line: usize,
    col_start: usize,
    col_end: usize,
    level: String,
    line_content: String,
}

struct Rule {
    id: &'static str,
    pattern: &'static str,
    short_desc: &'static str,
    full_desc: &'static str,
    severity: f64,
}

const RULES: &[Rule] = &[
    Rule {
        id: "SECRET001",
        pattern: "AKIA[0-9A-Z]{16}",
        short_desc: "AWS access key ID detected",
        full_desc: "A string matching the AWS access key ID pattern (AKIA followed by 16 alphanumeric characters) was found in source code.",
        severity: 8.5,
    },
    Rule {
        id: "SECRET002",
        pattern: "ghp_[0-9a-zA-Z]{36}",
        short_desc: "GitHub personal access token detected",
        full_desc: "A string matching the GitHub personal access token pattern (ghp_ followed by 36 alphanumeric characters) was found in source code.",
        severity: 9.0,
    },
    Rule {
        id: "SECRET003",
        pattern: "sk-[0-9a-zA-Z]{48}",
        short_desc: "OpenAI API key detected",
        full_desc: "A string matching the OpenAI API key pattern (sk- followed by 48 alphanumeric characters) was found in source code.",
        severity: 7.5,
    },
];

fn scan_file(path: &Path, rules: &[Rule]) -> Vec<Finding> {
    let content = match fs::read_to_string(path) {
        Ok(c) => c,
        Err(_) => return vec![], // skip binary files
    };

    let mut findings = Vec::new();

    for (line_num, line) in content.lines().enumerate() {
        for rule in rules {
            let re = regex::Regex::new(rule.pattern).unwrap();
            for mat in re.find_iter(line) {
                findings.push(Finding {
                    rule_id: rule.id.to_string(),
                    message: format!(
                        "{}: found '{}...' at {}:{}",
                        rule.short_desc,
                        &line[mat.start()..mat.start() + 4.min(mat.len())],
                        path.display(),
                        line_num + 1
                    ),
                    file: path.to_path_buf(),
                    line: line_num + 1,
                    col_start: mat.start() + 1, // SARIF columns are 1-indexed
                    col_end: mat.end() + 1,     // exclusive
                    level: "error".to_string(),
                    line_content: line.to_string(),
                });
            }
        }
    }

    findings
}

fn to_sarif(findings: &[Finding], rules: &[Rule]) -> Value {
    let sarif_rules: Vec<Value> = rules
        .iter()
        .map(|r| {
            json!({
                "id": r.id,
                "shortDescription": { "text": r.short_desc },
                "fullDescription": { "text": r.full_desc },
                "defaultConfiguration": { "level": "error" },
                "properties": {
                    "security-severity": format!("{:.1}", r.severity)
                }
            })
        })
        .collect();

    let sarif_results: Vec<Value> = findings
        .iter()
        .map(|f| {
            // Fingerprint from rule + path + content (not line number)
            let fingerprint_input =
                format!("{}:{}:{}", f.rule_id, f.file.display(), f.line_content.trim());
            let hash = format!("{:x}", md5_hash(fingerprint_input.as_bytes()));

            json!({
                "ruleId": f.rule_id,
                "ruleIndex": rules.iter().position(|r| r.id == f.rule_id).unwrap_or(0),
                "level": f.level,
                "message": { "text": f.message },
                "locations": [{
                    "physicalLocation": {
                        "artifactLocation": {
                            "uri": f.file.display().to_string(),
                            "uriBaseId": "%SRCROOT%"
                        },
                        "region": {
                            "startLine": f.line,
                            "startColumn": f.col_start,
                            "endLine": f.line,
                            "endColumn": f.col_end
                        }
                    }
                }],
                "partialFingerprints": {
                    "primaryLocationLineHash": &hash[..16]
                }
            })
        })
        .collect();

    json!({
        "$schema": "https://raw.githubusercontent.com/oasis-tcs/sarif-spec/main/sarif-2.1/schema/sarif-schema-2.1.0.json",
        "version": "2.1.0",
        "runs": [{
            "tool": {
                "driver": {
                    "name": "secret-scanner",
                    "version": env!("CARGO_PKG_VERSION"),
                    "rules": sarif_rules
                }
            },
            "results": sarif_results
        }]
    })
}

// Minimal hash for fingerprinting - in production, use a proper crate
fn md5_hash(input: &[u8]) -> u128 {
    // Simplified: in a real tool, use the `md5` or `sha2` crate
    let mut hash: u128 = 0;
    for (i, &byte) in input.iter().enumerate() {
        hash = hash.wrapping_mul(31).wrapping_add(byte as u128);
        hash ^= (i as u128).wrapping_mul(0x517cc1b727220a95);
    }
    hash
}

fn main() {
    let args: Vec<String> = std::env::args().collect();
    let target_dir = args.get(1).map(|s| s.as_str()).unwrap_or(".");

    let mut findings = Vec::new();

    // Walk directory, scan each file
    fn walk(dir: &Path, findings: &mut Vec<Finding>, rules: &[Rule]) {
        let entries = match fs::read_dir(dir) {
            Ok(e) => e,
            Err(_) => return,
        };
        for entry in entries.flatten() {
            let path = entry.path();
            if path.is_dir() {
                // Skip hidden dirs and common non-source dirs
                let name = path.file_name().unwrap_or_default().to_string_lossy();
                if name.starts_with('.') || name == "target" || name == "node_modules" {
                    continue;
                }
                walk(&path, findings, rules);
            } else {
                findings.extend(scan_file(&path, rules));
            }
        }
    }

    walk(Path::new(target_dir), &mut findings, RULES);

    let sarif = to_sarif(&findings, RULES);
    println!("{}", serde_json::to_string_pretty(&sarif).unwrap());
}
```

The `Cargo.toml`:

```toml
[package]
name = "secret-scanner"
version = "0.1.0"
edition = "2021"

[dependencies]
serde_json = "1"
regex = "1"
```

Run it:

```bash
cargo run -- ./src > results.sarif
```

That's a working scanner. It walks a directory, matches regex patterns against known secret formats, and outputs a valid SARIF file that GitHub Code Scanning will accept. The fingerprinting uses content-based hashing so findings survive line number shifts.

Is this production-quality secret detection? No - real tools like [gitleaks](https://github.com/gitleaks/gitleaks) or [truffleHog](https://github.com/trufflesecurity/trufflehog) have entropy analysis, allowlists, and hundreds of patterns. But the SARIF output structure is exactly the same. The format doesn't care whether your detection logic is 10 lines or 10,000.

## Using serde-sarif for type safety

If you're building a more serious tool, raw `serde_json::json!()` gets tedious. The [`serde-sarif`](https://crates.io/crates/serde-sarif) crate provides typed Rust structs generated from the official SARIF JSON Schema. Every field is a proper type, every enum is a Rust enum, and the builder pattern catches mistakes at compile time.

```toml
[dependencies]
serde-sarif = "0.8"
serde_json = "1"
```

```rust
use serde_sarif::sarif::{
    ArtifactLocation, Location, Message, PhysicalLocation,
    Region, ReportingDescriptor, Result as SarifResult, Run,
    Sarif, Tool, ToolComponent,
};

fn build_sarif() -> Sarif {
    let rule = ReportingDescriptor::builder()
        .id("SECRET001")
        .short_description(
            serde_sarif::sarif::MultiformatMessageString::builder()
                .text("Hardcoded API key detected")
                .build(),
        )
        .build();

    let location = Location::builder()
        .physical_location(
            PhysicalLocation::builder()
                .artifact_location(
                    ArtifactLocation::builder()
                        .uri("src/config.rs")
                        .build(),
                )
                .region(
                    Region::builder()
                        .start_line(14)
                        .start_column(28)
                        .end_column(48)
                        .build(),
                )
                .build(),
        )
        .build();

    let result = SarifResult::builder()
        .rule_id("SECRET001")
        .level("error")
        .message(Message::builder().text("AWS key found in source").build())
        .locations(vec![location])
        .build();

    let run = Run::builder()
        .tool(
            Tool::builder()
                .driver(
                    ToolComponent::builder()
                        .name("secret-scanner")
                        .version("0.1.0")
                        .rules(vec![rule])
                        .build(),
                )
                .build(),
        )
        .results(vec![result])
        .build();

    Sarif::builder()
        .version("2.1.0")
        .runs(vec![run])
        .build()
}

fn main() {
    let sarif = build_sarif();
    let json = serde_json::to_string_pretty(&sarif).unwrap();
    println!("{json}");
}
```

The builder pattern here means you can't forget required fields (the code won't compile). And you get autocomplete in your editor for every SARIF property instead of guessing JSON key names.

## The Rust SARIF ecosystem

The [`sarif-rs`](https://github.com/psastras/sarif-rs) project provides a suite of tools that demonstrate SARIF's power as a universal format:

**`clippy-sarif`** converts Clippy's JSON output to SARIF:

```bash
cargo clippy --message-format=json 2>&1 | clippy-sarif
```

**`sarif-fmt`** renders any SARIF file as pretty terminal output:

```bash
cargo clippy --message-format=json 2>&1 | clippy-sarif | sarif-fmt
```

This pipeline is the SARIF value proposition in one line. Clippy produces its native format. `clippy-sarif` normalizes it. `sarif-fmt` consumes the normalized version. You could swap `clippy-sarif` for `shellcheck-sarif` or `hadolint-sarif` and `sarif-fmt` wouldn't know the difference.

The same pipeline works for CI:

```yaml
- name: Run Clippy
  run: |
    cargo clippy --message-format=json 2>&1 | clippy-sarif > clippy.sarif

- name: Upload Clippy results
  uses: github/codeql-action/upload-sarif@v4
  with:
    sarif_file: clippy.sarif
    category: clippy
```

Now your Clippy warnings show up in GitHub's Security tab alongside findings from CodeQL, Semgrep, or any other SARIF-producing tool. Same UI. Same triage workflow. Same alert lifecycle (open, dismissed, fixed).

## Who speaks SARIF

The ecosystem is larger than you might expect:

**Security scanners:**
- [CodeQL](https://codeql.github.com/) - GitHub's own semantic analysis engine. Native SARIF output.
- [Semgrep](https://semgrep.dev/) - pattern-based scanner. `semgrep --sarif` flag.
- [Trivy](https://trivy.dev/) - container and IaC scanner. `trivy fs --format sarif` flag.
- [Checkov](https://www.checkov.io/) - infrastructure-as-code scanner. `--output sarif` flag.
- [ESLint](https://eslint.org/) - via `@microsoft/eslint-formatter-sarif`.
- [gitleaks](https://github.com/gitleaks/gitleaks) - secret scanner. `--report-format sarif` flag.

**Rust-specific:**
- `clippy-sarif` - Clippy diagnostics
- `miri-sarif` - Miri (undefined behavior detector) diagnostics
- `cargo-audit` can output JSON that you could convert to SARIF with a small wrapper

**Consumers:**
- GitHub Code Scanning
- Azure DevOps Advanced Security
- VS Code via the [SARIF Viewer extension](https://github.com/microsoft/sarif-vscode-extension)
- [SARIF Explorer](https://blog.trailofbits.com/2024/03/20/streamline-the-static-analysis-triage-process-with-sarif-explorer/) by Trail of Bits (VS Code extension for triage workflows)
- SonarQube (import via plugin)

The [SARIF web validator](https://sarifweb.azurewebsites.net/) is useful for checking your output before wiring up CI. Paste in your JSON, it tells you what's wrong.

## Common mistakes

After seeing several custom SARIF integrations, these are the patterns that cause the most pain:

**1. Absolute file paths.** Your SARIF says `"/home/runner/work/repo/src/main.rs"`. GitHub expects `"src/main.rs"`. Use paths relative to the repository root, or set `invocations[0].workingDirectory.uri` to the checkout path so consumers can strip the prefix.

**2. Missing fingerprints on API uploads.** The `upload-sarif` Action calculates fingerprints for you. The REST API does not. If you use the API without `partialFingerprints`, every push creates a new set of alerts, even if nothing changed. You end up with 500 "open" alerts that are all the same finding.

**3. Inconsistent `ruleId` across runs.** If your rule ID changes between versions (maybe you renamed `SEC001` to `SECRET001`), GitHub treats them as different rules. All alerts from the old ID stay open, and new alerts from the new ID get created alongside them. Pick stable rule IDs and keep them forever.

**4. Line numbers without columns.** GitHub accepts this, but your PR annotations will highlight the entire line instead of the specific token. Precise column ranges make the difference between "something is wrong on line 14" and "this specific string on line 14 is an API key."

**5. Security severity as a string.** The `properties.security-severity` field must be a string representation of a number, not a number. `"8.5"`, not `8.5`. GitHub silently ignores it if you get the type wrong, and your findings lose their CVSS-based severity ordering.

## Validating your SARIF

Before uploading to GitHub, validate locally. The SARIF schema is public:

```bash
# Install ajv-cli for JSON Schema validation
npm install -g ajv-cli ajv-formats

# Download the schema
curl -sL https://raw.githubusercontent.com/oasis-tcs/sarif-spec/main/sarif-2.1/schema/sarif-schema-2.1.0.json -o sarif-schema.json

# Validate your file
ajv validate -s sarif-schema.json -d results.sarif --spec=draft2020 -c ajv-formats
```

Or use the [Microsoft SARIF Validator](https://sarifweb.azurewebsites.net/Validation) web tool, which checks both schema validity and GitHub-specific requirements.

In a Rust project, you can validate programmatically with `jsonschema`:

```rust
use jsonschema::JSONSchema;
use serde_json::Value;

fn validate_sarif(sarif: &Value) -> Result<(), Vec<String>> {
    let schema_str = include_str!("sarif-schema-2.1.0.json");
    let schema: Value = serde_json::from_str(schema_str).unwrap();
    let compiled = JSONSchema::compile(&schema).unwrap();

    let errors: Vec<String> = compiled
        .validate(sarif)
        .err()
        .into_iter()
        .flatten()
        .map(|e| format!("{} at {}", e, e.instance_path))
        .collect();

    if errors.is_empty() {
        Ok(())
    } else {
        Err(errors)
    }
}
```

## A complete CI pipeline

Putting it all together - a GitHub Actions workflow that runs multiple scanners, collects SARIF from each, and uploads everything:

```yaml
name: Security Analysis
on:
  push:
    branches: [main]
  pull_request:
  schedule:
    - cron: '0 6 * * 1' # weekly Monday 6am

jobs:
  analyze:
    runs-on: ubuntu-latest
    permissions:
      security-events: write
      contents: read

    steps:
      - uses: actions/checkout@v5

      - name: Install Rust toolchain
        uses: dtolnay/rust-toolchain@stable
        with:
          components: clippy

      - name: Install SARIF tools
        run: cargo install clippy-sarif sarif-fmt

      - name: Clippy analysis
        run: |
          cargo clippy --all-targets --message-format=json 2>&1 \
            | clippy-sarif > clippy.sarif
        continue-on-error: true

      - name: Custom secret scan
        run: cargo run -p secret-scanner -- ./src > secrets.sarif
        continue-on-error: true

      - name: Trivy filesystem scan
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: 'fs'
          format: 'sarif'
          output: 'trivy.sarif'

      - name: Upload Clippy SARIF
        uses: github/codeql-action/upload-sarif@v4
        with:
          sarif_file: clippy.sarif
          category: clippy

      - name: Upload Secret Scanner SARIF
        uses: github/codeql-action/upload-sarif@v4
        with:
          sarif_file: secrets.sarif
          category: secret-scanner

      - name: Upload Trivy SARIF
        uses: github/codeql-action/upload-sarif@v4
        with:
          sarif_file: trivy.sarif
          category: trivy
```

Three tools, three categories, all landing in the same Security tab. A developer reviewing a PR sees annotations from Clippy, secret detection, and Trivy vulnerability scanning - all inline, all with the same triage workflow.

The `continue-on-error: true` on scanner steps is important. If your scanner exits with a non-zero code when it finds issues (which is the convention), you don't want the workflow to stop before uploading the SARIF. The findings are the whole point.

## Beyond GitHub

SARIF isn't GitHub-specific, even though GitHub is its biggest consumer. The format works anywhere you need machine-readable static analysis results:

- **Custom dashboards** - parse SARIF with any JSON library, aggregate findings across repos, track trends over time
- **IDE integration** - the VS Code SARIF Viewer shows findings with inline squiggles, and the SARIF Explorer extension from Trail of Bits adds triage workflow (mark as false positive, add notes, track status)
- **Compliance reporting** - security teams can collect SARIF from all repos and generate audit reports showing which rules passed, which failed, and what was triaged
- **Tool comparison** - run two scanners on the same codebase, produce two SARIF files, diff the results to see coverage gaps

The format is verbose - a single finding with all its metadata can be 30-40 lines of JSON. But verbosity in a machine-readable format is a feature, not a bug. You're not meant to read SARIF files by hand. You're meant to write tools that produce them and consume them.

## The spec is worth reading

The [full SARIF 2.1.0 specification](https://docs.oasis-open.org/sarif/sarif/v2.1.0/sarif-v2.1.0.html) is long (over 200 pages) but well-written for a standards document. If you're going beyond basic results and locations, it covers:

- **Taxonomies** - mapping your rules to standard classifications like CWE or OWASP
- **Graphs and graph traversals** - for representing call graphs and data flow
- **Suppressions** - marking results as false positives or won't-fix
- **Notifications** - tool configuration errors and warnings (distinct from analysis results)
- **Conversion metadata** - tracking when SARIF was generated from a non-SARIF source

Microsoft also maintains a [SARIF tutorials repo](https://github.com/microsoft/sarif-tutorials) with worked examples for each feature.

SARIF solves a real problem in a boring, reliable way. It's not exciting technology. It's plumbing. But good plumbing is what makes it possible to build multi-tool security pipelines that actually work in practice, instead of a collection of scripts parsing bespoke output formats. If you're building any kind of code analysis tool, outputting SARIF is the single best thing you can do for your users' integration story.
