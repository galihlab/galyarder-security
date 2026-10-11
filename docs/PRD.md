# Galyarder Security — Product Requirements

**Status**: Draft
**Scope**: Define what Galyarder Security is, who it serves, and the requirements the repository
does and does not yet meet. This is the local-first security and verification layer of Galyarder
Labs' Agentic Company work, built as a fork of KeygraphHQ/shannon.

## Problem

An autonomous agent that can run shell commands, open a browser, and touch a live application is
useful precisely because it can act. The same property makes it hard to trust. When agent setups
stop at chat, the work stays loose: no durable scope, no review gates, and no audit trail of what
ran against what.

Manual security review does not scale to this. A penetration test happens once, and code ships
continuously in between. Teams need a way to run a real security pass on demand, against their own
applications, and to see evidence that a reported issue is actually exploitable rather than a
pattern match. They also need that pass to run locally, under their own credentials, with the model
traffic going where they choose.

The problem this project addresses is: how to run continuous, proof-based security review inside a
local, agent-supervised workspace, without sending source code and credentials to a third-party
scanner, and without leaving generated test scripts and artifacts behind.

## Who it is for

- A solo developer or small team running an "Agentic Company" who need to security-test their own
  web applications and source code, on their own machine.
- Operators who already run a local model proxy and want security tooling to route through it.
- Security-minded engineers who want to read and modify the pipeline rather than treat it as a
  black box.
- Contributors interested in autonomous verification, audit trails, and workspace hygiene.

## Goals

- Run an end-to-end security assessment of a web application and its source code from a single
  command, and produce a report a developer can act on.
- Report only findings that have been demonstrated with a real exploit, so the output favors
  precision over volume.
- Route all model calls through a proxy the operator controls, so no run depends on a hardcoded
  vendor endpoint.
- Leave a durable, inspectable record of every run: prompts, agent turns, timing, and cost.
- Leave the target workspace in a clean state after a run, with generated artifacts removed.

## Non-goals

- Not a hosted or managed scanning service. The tool is local-first and operator-run.
- Not a black-box scanner. It is designed for white-box (source-available) targets.
- Not a replacement for static analysis of dependencies or configuration. Per `COVERAGE.md`,
  findings it cannot actively exploit (vulnerable libraries, weak crypto, insecure configuration)
  are explicitly out of scope for this pipeline.
- Not a proprietary product. It stays under AGPL-3.0.
- Not a compliance-certification tool.

## Requirements

### Pipeline and orchestration

- **R1 — Five-phase assessment pipeline.** The system runs pre-reconnaissance, reconnaissance,
  vulnerability analysis, exploitation, and reporting as ordered phases. `[implemented]` —
  `src/temporal/workflows.ts` (pipeline comment block), `src/session-manager.ts` (agent graph with
  prerequisites), `src/phases/pre-recon.ts`, `src/phases/reporting.ts`.
- **R2 — Durable orchestration with crash recovery.** Workflows resume after a worker restart rather
  than losing progress. `[implemented]` — Temporal-based orchestration in `src/temporal/workflows.ts`,
  `src/temporal/worker.ts`, and the `temporal` service in `docker-compose.yml`.
- **R3 — Parallel vulnerability and exploitation.** The five vulnerability classes and their
  exploitation steps run concurrently. `[implemented]` — `src/temporal/workflows.ts` ("5 pipelined
  pairs in parallel"); agent definitions in `src/session-manager.ts`.
- **R4 — Exploitation gated on a validated queue.** Exploitation runs only when the vulnerability
  phase produced a valid queue. `[implemented]` — `src/queue-validation.ts`
  (`VulnType = 'injection' | 'xss' | 'auth' | 'ssrf' | 'authz'`, deliverable/queue validation rules).
- **R5 — Retry classification.** Transient errors are retried with backoff; permanent errors are not.
  `[implemented]` — `PRODUCTION_RETRY` / `TESTING_RETRY` with `nonRetryableErrorTypes` in
  `src/temporal/workflows.ts`; error types in `src/error-handling.ts`.

### Proxy routing and model control

- **R6 — Route model calls through a configurable local proxy.** `[partial]` — The `Dockerfile`
  patches `@anthropic-ai` SDK model strings at build time and `docker-compose.yml` passes
  `ANTHROPIC_BASE_URL` / `ANTHROPIC_API_KEY` / `CLAUDE_MODEL`. The proxy itself is not bundled or
  exemplified; the operator must supply it.
- **R7 — No hardcoded vendor endpoints.** `[partial]` — Known model identifiers are rewritten by the
  build step, and the base URL is configurable. The integration still speaks the Anthropic SDK
  contract, so it is proxy-compatible rather than fully provider-neutral.
- **R8 — Configurable model selection, including sub-agents.** `[partial]` — `CLAUDE_MODEL` and
  sub-agent overrides (`CLAUDE_CODE_SUBAGENT_MODEL`, `ANTHROPIC_DEFAULT_*_MODEL`) are present in
  `docker-compose.yml` and `.env.example`. `.env.example` still ships an Anthropic default
  (`claude-sonnet-4-5-20250929`) as the primary example.

### Audit and evidence

- **R9 — Crash-safe audit logging.** Append-only logs with immediate flush and atomic session writes
  survive abrupt termination. `[implemented]` — `src/audit/audit-session.ts`,
  `src/audit/logger.ts`, `src/audit/workflow-logger.ts`, `src/utils/concurrency.ts` (`SessionMutex`).
- **R10 — Per-phase and per-agent metrics.** Timing and cost are tracked and aggregated.
  `[implemented]` — `src/audit/metrics-tracker.ts`, `src/utils/metrics.ts`, and the metrics
  aggregation in `src/temporal/workflows.ts`.
- **R11 — Store the exact prompts used.** Runs are reproducible from the recorded prompts.
  `[implemented]` — `CLAUDE.md` describes `audit-logs/{hostname}_{sessionId}/prompts/`; confirmed by
  the committed run artifacts under `xben-benchmark-results/*/audit-logs/prompts/`.
- **R12 — Queryable progress.** A running workflow can be inspected without reading raw logs.
  `[implemented]` — `src/temporal/query.ts`, `./shannon query ID=<workflow-id>`, Temporal Web UI on
  port 8233.

### Configuration and authentication

- **R13 — YAML configuration with schema validation.** `[implemented]` — `src/config-parser.ts`,
  `configs/config-schema.json` (JSON Schema draft-07), `configs/example-config.yaml`.
- **R14 — Multiple authentication mechanisms.** Form, SSO, API, and basic auth are supported.
  `[implemented]` — `login_type` enum in `configs/config-schema.json`.
- **R15 — TOTP / 2FA support.** Six-digit TOTP codes are generated during login flows.
  `[implemented]` — `mcp-server/src/tools/generate-totp.ts` (RFC 6238/4226), `totp_secret` field in
  the config schema, `prompts/shared/login-instructions.txt`.
- **R16 — Focus and avoid rules.** Operators can steer testing toward or away from specific
  subdomains and paths. `[implemented]` — `rules.avoid` / `rules.focus` in
  `configs/config-schema.json` and `configs/example-config.yaml`.

### Deliverables and reporting

- **R17 — Executive report generation.** A final report is produced as a distinct phase.
  `[implemented]` — `src/phases/reporting.ts`, `prompts/report-executive.txt`,
  `src/ai/output-formatters.ts`.
- **R18 — Machine-readable findings.** `[partial]` — An `outputs/schemas/` directory exists and
  exploitation queues are written as JSON, but there is no stable, documented schema for a finding
  that downstream tools can rely on.
- **R19 — Proof-based findings.** Findings are backed by executed exploit evidence, not just
  indicators. `[implemented]` — Exploitation phase in `src/temporal/workflows.ts`; committed run
  artifacts include files such as `*_exploitation_evidence.md` and `*_exploitation_queue.json` under
  `xben-benchmark-results/`.
- **R20 — Sample reports.** `[implemented]` — `sample-reports/shannon-report-juice-shop.md`,
  `sample-reports/shannon-report-crapi.md`, `sample-reports/shannon-report-capital-api.md`.

### Workspace hygiene and cleanup

- **R21 — Cleanup of generated artifacts.** Generated exploit scripts, result files, and deliverable
  directories can be removed, with a dry run. `[implemented]` — `cleanup_artifacts()` in the
  `shannon` CLI; `DRY_RUN=true` and `KEEP_DELIVERABLES=true` options documented in its help text and
  in `README.md`.
- **R22 — Verified zero-residue guarantee.** `[planned]` — The current cleanup matches fixed filename
  patterns at depth 1 (`find "$repo_path" -maxdepth 1`). It is a best-effort sweep, not a verified
  guarantee that the target is residue-free.
- **R23 — Cleanup scoped to the target repo.** `[planned]` — Cleanup validates that the repo path
  exists but does not otherwise constrain where it runs. A guard that refuses paths outside the
  intended target is not implemented.

### External tooling

- **R24 — Integrated external reconnaissance tools.** `[implemented]` — `Dockerfile` installs
  `nmap`, `subfinder`, `whatweb`, and `schemathesis`; `CLAUDE.md` lists them as external tool
  dependencies.
- **R25 — Tool availability check before execution.** `[implemented]` — `src/tool-checker.ts`
  validates that required tools are present before the pipeline uses them.

### Project and community

- **R26 — Lineage and attribution documentation.** `[planned]` — The source retains "Copyright (C)
  2025 Keygraph, Inc." headers under AGPL-3.0, but there is no `NOTICE` file and no lineage or
  attribution section in `README.md`.
- **R27 — Responsible-use statement.** `[planned]` — The tool performs active exploitation, yet
  there is no explicit statement of authorization expectations and permitted scope in the
  documentation.
- **R28 — Tests and CI.** `[planned]` — No test suite or CI workflow is present in the tree.
- **R29 — Complete rebrand to Galyarder Security.** `[partial]` — `README.md` was rewritten, but the
  CLI file is still `shannon`, the banner still reads "AI Penetration Testing Framework", and
  `package.json` still declares `"name": "shannon"`.
- **R30 — Upstream sync strategy.** `[planned]` — The repository has no shared git history with
  `KeygraphHQ/shannon` (a comparison returns "No common ancestor"), so upstream changes cannot be
  merged by fast-forward. A decision on periodic manual diffs versus a deliberate freeze is not yet
  recorded.

## Success signals

- An operator can go from clone to a completed first assessment using only the documented steps, with
  a proxy they configured themselves.
- A developer who receives a report can reproduce at least one reported exploit from the evidence
  included with it.
- A run leaves the target repository clean, and the operator can verify this before and after rather
  than taking it on trust.
- The audit directory for a run is complete enough that a reviewer can reconstruct what the agents
  did, in what order, and at what cost.
- New contributors can tell what the project is, where it came from, and how to change it safely.
- Coverage claims in `COVERAGE.md` match what the pipeline actually checks.

## Open questions

- **Upstream relationship.** Should Galyarder track `KeygraphHQ/shannon` with periodic manual diffs,
  or freeze at the current fork point and diverge deliberately? There is no shared history to build
  on, and this decision shapes everything else.
- **Proxy scope.** Should the project ship a bundled or example proxy configuration (for instance a
  ready-made CLIProxyAPI or LiteLLM setup), or continue to require the operator to bring their own?
- **Rebrand depth.** How far should the "Shannon" naming be carried? The CLI command, banner, and
  package name are all still upstream; changing them affects existing users' scripts.
- **Deliverable schema.** What should a stable, machine-readable finding look like, and should it
  become a published contract for downstream Galyarder tools?
- **Coverage priorities.** Which unaddressed WSTG categories are worth adding next, given that the
  pipeline only reports what it can exploit?
- **Licensing posture.** The repository is AGPL-3.0. Is the intent to keep any future derivative
  public and copyleft, or to rely on the network-service clause? This should be stated explicitly
  before any hosted use.
- **Benchmark material.** The 4,445 imported XBOW artifacts dominate the tree and describe upstream's
  evaluation, not this repository's. Should they stay, move, or be dropped?

---

## License note

This project is licensed under **AGPL-3.0**, a strong copyleft license. Any derivative must also be
AGPL-3.0 and cannot be relicensed as proprietary. If the software is offered as a network service,
its source code must be made available to the users of that service.
