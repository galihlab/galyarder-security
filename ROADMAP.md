# Galyarder Security — Roadmap

Galyarder Security is the security and verification layer for Galyarder Labs' "Agentic Company"
work. It runs an autonomous penetration-testing pipeline against a web application and its source
code, and returns a report of findings that were actually exploited rather than a list of alerts.
It is meant for one operator or a small team who want to run security review continuously and
locally, against targets they are authorized to test, without handing credentials or traffic to a
third-party scanner.

## Where this comes from

This repository is a fork of [KeygraphHQ/shannon](https://github.com/KeygraphHQ/shannon)
(Shannon Lite), an autonomous AI pentester for web applications and APIs. The GitHub API reports
`fork: true` with `parent: KeygraphHQ/shannon`, and the repository keeps that project's AGPL-3.0
license. The upstream work — the five-phase pipeline, the Temporal orchestration, the prompt set,
and the audit system — is Keygraph's, and copyright headers in the source still read
"Copyright (C) 2025 Keygraph, Inc."

The direction that belongs to Galyarder is narrower and local-first:

- Treat the pentester as one component of a local agentic workspace, not a standalone product.
- Route every model call through a proxy the operator controls, so no run depends on a hardcoded
  vendor endpoint.
- Keep the audit trail and the cleanup behavior in the operator's hands.

This document describes that direction. It does not claim the upstream engine as original work.

## Where it is now

Verified against the repository on the `main` branch (GitHub API and file reads, at the time of
writing):

- The repository is public, default branch `main`, licensed AGPL-3.0, and reported by the API as a
  fork of `KeygraphHQ/shannon`. It has no shared git history with the upstream default branch
  (a comparison returns "No common ancestor"), so upstream updates do not arrive by fast-forward.
- The tree holds 4,595 entries. 4,445 of those are imported XBOW benchmark artifacts under
  `xben-benchmark-results/` (turn-by-turn agent logs, prompts, and per-challenge reports). The
  working code is much smaller: 43 TypeScript files under `src/`, 33 files under `prompts/`, and
  18 files under `mcp-server/`.
- The pipeline is real and wired: `src/temporal/workflows.ts` orchestrates pre-recon, recon, five
  parallel vulnerability/exploitation pairs, and reporting; `src/session-manager.ts` defines the
  agent graph and prerequisites; `src/queue-validation.ts` gates exploitation on the vulnerability
  queue. `docker-compose.yml` runs Temporal and a worker container.
- Proxy routing exists as configuration, not as a bundled proxy. The `Dockerfile` patches the
  `@anthropic-ai` SDK model strings at build time and `docker-compose.yml` passes
  `ANTHROPIC_BASE_URL`, `ANTHROPIC_API_KEY`, and `CLAUDE_MODEL`. The operator must supply the proxy
  itself (CLIProxyAPI, LiteLLM, OpenRouter, or similar).
- The audit system is present and crash-safe by design: `src/audit/audit-session.ts`,
  `src/audit/logger.ts`, and `src/audit/metrics-tracker.ts` write append-only session logs, metrics,
  and the exact prompts used.
- The `./shannon` CLI implements `start`, `logs`, `query`, `cleanup`, `stop`, and `help`. `cleanup`
  supports `DRY_RUN=true` and removes generated Python/JSON/Markdown artifacts and deliverable
  directories by pattern.
- The documentation is thin in the places that matter for adoption. There is no `NOTICE` file and
  no lineage/attribution section in `README.md`, even though the source retains Keygraph copyright
  headers. `README-original.md` and `SHANNON-PRO.md` are upstream material that still reference
  Keygraph's website, a commercial "Shannon Pro" edition, and a contact address.
- The rename to "Galyarder Security" is partial. `README.md` was rewritten, but the CLI file is
  still named `shannon`, the banner still prints "AI Penetration Testing Framework", and
  `package.json` still declares `"name": "shannon"`.

What is thin: a bundled or example proxy configuration, an attribution/lineage document, a
responsible-use/scope statement, a verified zero-residue cleanup guarantee, and any test or CI
suite.

## Now

- **Finish the rename without breaking the command.** Align the CLI name, banner, and
  `package.json` name with Galyarder Security while keeping `./shannon` working as an alias.
  The rename is already half-done, and the mismatch is the first thing a new reader notices.
- **Add a lineage and attribution section, and a `NOTICE` file.** The source carries Keygraph
  copyright headers under AGPL-3.0; attribution should be explicit and in one place rather than
  scattered across upstream files.
- **Ship one worked proxy example.** Document a single end-to-end setup (for example CLIProxyAPI)
  so an operator can go from clone to a first run without guessing the proxy contract. Today the
  README assumes a proxy that is not provided.
- **Make cleanup verifiable.** The current `cleanup` command matches patterns at depth 1. Turn it
  into a routine that reports exactly what it will remove, refuses to run outside the target repo,
  and can be checked before it is trusted.
- **Write a responsible-use statement.** The tool performs active exploitation. A short, clear
  statement of scope, authorization expectations, and white-box-only intent belongs in the README.
- **Decide the upstream sync strategy.** With no shared history, the choice is a periodic manual
  diff against `KeygraphHQ/shannon` or a deliberate freeze. Either is fine; leaving it undecided is
  not.

## Next

- **Broaden vulnerability coverage.** `COVERAGE.md` honestly lists a large number of WSTG items the
  pipeline does not yet check. Add classes that fit the proof-based model, and mark each addition in
  `COVERAGE.md` when it is reliable.
- **Extend the configuration surface.** The YAML schema currently covers authentication and
  `focus`/`avoid` rules. Per-phase limits and clearer run metadata would make repeated runs easier to
  reason about.
- **Give deliverables a machine-readable shape.** The `outputs/schemas/` directory exists; a stable
  schema for findings would let a control plane or dashboard consume results instead of parsing
  prose.
- **Add tests and CI.** There is no test or CI suite in the tree. Even a small set of unit tests
  around the config parser and queue validation would make contributions safer.
- **Support batch or repeated targets.** Running the same pipeline across several applications in one
  session is a common operator need and is not currently first-class.

## Later

- **Deeper code-level analysis, as original work.** Upstream reserves its LLM-powered data-flow
  engine for the commercial edition. If Galyarder wants source-to-sink analysis, it should be
  designed and built here, not copied from a proprietary edition.
- **A policy layer for agent tool execution.** The README names "agent tool execution drift" as a
  threat vector. A supervisor that inspects tool calls and shell commands before they run would
  address that directly, and is a natural extension of the audit system.
- **Integration with the wider Galyarder stack.** Folding security runs into `galyarder-hq` as a
  scheduled, inspectable activity would make verification a standing part of the workspace rather
  than a separate tool.

## Not planned

- **No hosted scanning service.** The project is local-first and proxy-controlled by design. A
  cloud scanner would move credentials and traffic off the operator's machine, which is the thing
  this setup exists to avoid.
- **No re-licensing to a proprietary model.** AGPL-3.0 is a strong copyleft license; derivatives
  must stay AGPL-3.0. See the note below.
- **No reimplementation of commercial-edition features by copying them.** Rebuilding "Shannon Pro"
  capabilities from proprietary source is not on the table. Original work is.
- **No black-box-only mode presented as equivalent.** The pipeline is white-box and source-aware.
  It will not pretend to match black-box tools on their terms.
- **No performance or benchmark numbers invented for this repository.** The only benchmark material
  here is upstream's XBOW run, which is imported unchanged and carries its own caveats.

## How to follow along

- Open an issue for bugs, gaps in coverage, or documentation problems on
  [galihlab/galyarder-security](https://github.com/galihlab/galyarder-security).
- Send pull requests for documentation, cleanup behavior, and tests first; they are the lowest-risk
  way to contribute while the engine is still stabilizing.
- Read `COVERAGE.md` before assuming a vulnerability class is checked, and `CLAUDE.md` before
  changing the pipeline.

---

## License note

This repository is licensed under **AGPL-3.0**, a strong copyleft license. Any derivative work must
also be released under AGPL-3.0; it cannot be relicensed as proprietary. If the software is run as a
network service and users interact with it over a network, the source code must be made available to
those users. Keep this in mind before building a hosted offering on top of it.
