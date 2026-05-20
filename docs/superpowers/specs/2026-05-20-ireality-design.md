# ireality CLI Design

## Goal

Build `ireality`, a command-line tool that helps operators choose Xray REALITY-compatible TLS target domains. The tool evaluates whether candidate domains are suitable for use as REALITY `dest` / `serverNames` targets by measuring reachability, TLS/SNI behavior, certificate properties, ALPN support, HTTP responsiveness, and latency.

The implementation intentionally avoids offensive behavior. It does not steal certificates, bypass defenses, evade detection, or perform high-volume internet scanning. It performs low-rate compatibility checks against explicit candidate domains and built-in public candidates.

## Non-goals and safety boundaries

- No arbitrary wide internet scanning.
- No stealth, evasion, credential access, exploitation, or denial-of-service behavior.
- No automated probing of random CIDR ranges by default.
- No claim that a recommended domain is authorized for all uses; the output is a technical compatibility recommendation.
- Network probing must use conservative defaults: low concurrency, timeouts, retry limits, and clear user-controlled input.

## User-facing commands

### `ireality check DOMAIN`

Evaluate a single domain and print a recommendation.

Checks:

- TCP connection to port 443.
- TLS handshake with SNI set to the domain.
- Certificate validity window, subject, issuer, and SAN coverage.
- Negotiated ALPN protocol from `h2` / `http/1.1`.
- Lightweight HTTP `HEAD /` request when TLS succeeds.
- Latency samples and jitter.

Output:

- Human-readable table by default.
- JSON with `--json`.
- Final verdict: `excellent`, `good`, `weak`, or `reject`.
- Explanation of major positive and negative factors.

### `ireality scan --input domains.txt`

Evaluate a user-provided list of candidate domains.

Behavior:

- Deduplicate and normalize domains.
- Skip invalid hostnames.
- Probe candidates with bounded concurrency.
- Rank successful candidates by score.
- Emit table or JSON output.

### `ireality recommend --server HOST_OR_IP`

Recommend REALITY target domains for a server location.

Behavior:

- Use built-in candidate domains plus optional user-provided `--input` candidates.
- Probe candidate domains from the local machine running the CLI.
- Use `--server` as metadata in reports and future extension point; the first implementation does not perform remote probing from that server.
- Sort by REALITY suitability score, latency, and stability.

### `ireality explain DOMAIN`

Run the same checks as `check`, but emphasize the explanation and scoring details.

## Architecture

Package layout:

- `ireality/cli.py`: Typer command definitions and output formatting.
- `ireality/models.py`: dataclasses for probe results, candidate scores, verdicts, and explanations.
- `ireality/candidates.py`: built-in candidate list, file loading, hostname normalization, and deduplication.
- `ireality/tls_probe.py`: TCP/TLS/SNI/ALPN/certificate probing.
- `ireality/http_probe.py`: minimal HTTP `HEAD` probing over TLS.
- `ireality/scoring.py`: REALITY suitability scoring and verdict assignment.
- `tests/`: unit tests for parsing, scoring, candidate handling, and probe result interpretation.

The code should keep network probing separate from scoring so tests can validate recommendations without making network calls.

## Data flow

1. CLI parses command options.
2. Candidate loader produces normalized domain names.
3. Probe runner performs bounded concurrent checks.
4. TLS and HTTP probe modules return structured results, not formatted text.
5. Scoring module converts probe results into scores, verdicts, and reasons.
6. CLI renders table or JSON output.

## Scoring model

Start at 0 and add or subtract weighted factors:

- TLS handshake succeeds: required for non-reject verdict.
- Certificate currently valid: strong positive.
- SAN includes the requested SNI domain: strong positive.
- ALPN negotiates `h2`: positive.
- ALPN negotiates `http/1.1`: small positive.
- HTTP `HEAD` returns any sane HTTP response below 500: positive.
- Low average latency: positive.
- Low jitter: positive.
- Expired certificate, hostname mismatch, handshake failure, DNS failure, or TCP timeout: reject or heavy penalty.

Suggested verdict thresholds:

- `excellent`: score >= 85.
- `good`: score >= 70.
- `weak`: score >= 50.
- `reject`: score < 50 or required TLS checks fail.

## Error handling

- DNS failures, timeouts, refused connections, TLS errors, and HTTP errors are captured per candidate.
- One failed candidate must not stop a batch scan.
- CLI exits non-zero only for invalid invocation, unreadable input files, or complete internal failures.
- JSON output includes machine-readable error codes.

## Testing strategy

- Unit-test domain normalization and invalid hostname rejection.
- Unit-test scoring with synthetic probe results for excellent/good/weak/reject cases.
- Unit-test JSON serialization shape.
- Keep live network tests optional and disabled by default.
- Provide a small manual smoke-test path using `ireality check example.com`.

## Implementation choices

- Use Python for portability and fast development.
- Use Typer for CLI ergonomics.
- Use the Python standard library for TLS and HTTP probing where practical.
- Use pytest for tests.
- Keep dependencies minimal: `typer`, `rich`, and `pytest`.

## Safety defaults

- Default concurrency: 5.
- Default timeout: 5 seconds.
- Default latency samples: 3.
- No CIDR scanning in the first implementation.
- Built-in candidates are curated domains only, not generated targets.
