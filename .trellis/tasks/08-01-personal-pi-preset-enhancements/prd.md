# PRD: Enhance personal Pi preset

## Goal

Extend the public `zidou-kiyn/pi-preset` package with two independently verifiable capabilities:

1. make Matt Pocock's upstream `grill-me` interview workflow available in Pi and keep it refreshable from upstream; and
2. provide a safe interactive flow that adds a predefined Anthropic, OpenAI, or DeepSeek channel with a user-selected subset of models to `~/.pi/agent/models.json`.

The result should make a fresh Pi installation easier to configure without shipping credentials, private endpoints, or stale copies of third-party content.

## Background and Confirmed Facts

- The preset is a standalone public repository at `~/pi-preset`, installed as `git:github.com/zidou-kiyn/pi-preset`.
- The preset currently ships two extensions: `/preset-sync` and `/vibrant-footer`; `/preset-sync` uses diff-first, confirm-before-write, idempotent behavior.
- The preset repository and full history must remain free of API keys and private gateway URLs.
- Pi loads global skills from `~/.pi/agent/skills/` and exposes them as `/skill:<name>` commands.
- Upstream `grill-me` is only a user-invoked wrapper whose complete body is `Run a /grilling session.` The actual interview contract is the separate `grilling` skill, so both skills are required.
- The `skills` CLI directly supports Pi global installs and records source metadata for later update checks:
  - install target: `~/.pi/agent/skills/`
  - global source lock: `~/.agents/.skill-lock.json`
  - source/path/folder hashes are retained for refresh validation
- Pi custom provider configuration belongs in `~/.pi/agent/models.json`, not `model.json`.
- `models.json` is hot-reloaded when `/model` is opened.
- The local `models.json` already contains the requested model metadata and compatibility values. Those templates may be reused after removing local provider identifiers, endpoints, and credentials.

## Child Task Map

| Child | Deliverable | Independent verification |
|---|---|---|
| `08-01-integrate-grill-me` | Install and validate upstream `grill-me` + `grilling`, then integrate the chosen upstream update mechanism into the preset | Pi lists both skills; `/skill:grill-me` performs a multi-turn one-question-at-a-time interview; the documented update path refreshes upstream changes |
| `08-01-interactive-models-config` | Add an interactive predefined-provider wizard with per-family model selection that safely updates `models.json` | Sandbox runs for all three families produce exactly the selected provider models without altering sibling providers or exposing credentials |

## Requirements

- **R1 Staged delivery:** validate `grill-me` in the locally installed Pi before adding it to the preset; validate each preset change in a sandbox before publishing it.
- **R2 Independent children:** each child owns its design, implementation plan, validation, and commit. The parent owns only the source requirements and final integration review.
- **R3 No secret material in the public artifact:** no real API key, private provider URL, or user-specific provider configuration may enter the preset working tree or git history.
- **R4 Upstream ownership remains visible:** Matt Pocock's skill files must remain attributable to their upstream repository and must not be silently rewritten as a local fork.
- **R5 Safe configuration writes:** any `models.json` write must be previewed, confirmed, atomic, permission-preserving, and limited to the selected provider entry.
- **R6 Existing preset behavior remains intact:** `/preset-sync`, the footer, package reconciliation, web-search config, and font behavior must continue to satisfy their existing guarantees.
- **R7 Documentation:** the preset README must explain skill installation/update behavior, the provider wizard flow, credential handling, and any restart or mode limitations.
- **R8 Explicit upstream refresh:** upstream skill updates are user-triggered through a one-command preset flow backed by the official `skills` CLI; they are not coupled to every `pi update --extensions` run.
- **R9 Fault-isolated skill command:** skill installation and refresh use a dedicated `/preset-skills-sync` command rather than extending `/preset-sync`.
- **R10 Existing provider replacement:** when the models wizard finds a conflicting provider identifier, it may replace only that provider after a redacted diff and a second explicit confirmation; the original file backup remains available.
- **R11 Delivery order:** complete and validate the `grill-me` child before starting implementation of the models wizard, matching the requested rollout order.
- **R12 Model choice:** after choosing a provider family, the user selects one or more models from that family's fixed catalog. The checklist starts empty and the wizard never forces or preselects the complete family bundle.

## Acceptance Criteria

- **AC1** Both child tasks meet their own acceptance criteria and are archived independently.
- **AC2** A clean Pi sandbox can install the preset and obtain both the grilling capability and the provider wizard using the documented flow.
- **AC3** A secret scan of the preset working tree and full reachable git history reports no credential or private endpoint.
- **AC4** Existing `/preset-sync` behavior still passes its previous smoke checks after integration.
- **AC5** The final public README contains copy-pasteable install, use, update, and recovery instructions for both new capabilities.

## Out of Scope

- Installing Matt Pocock's entire skill collection.
- Forking or behaviorally modifying `grill-me` / `grilling`.
- A general-purpose model editor, arbitrary model discovery, or custom compatibility authoring.
- Shipping a populated `models.json`, any credential, or any private base URL.
- Verifying paid provider requests against a real third-party endpoint as part of automated tests.

## Key Decision

- **D1 Upstream update contract:** use an explicit one-command refresh backed by the official `skills` CLI. This keeps only `grill-me` and `grilling`, preserves upstream source/hash tracking, and avoids cloning/installing the full Matt Pocock repository during every normal Pi package update.
- **D2 Command boundary:** add `/preset-skills-sync` as a dedicated install/update entry point. Network/npm failures remain isolated from the existing config, package, footer, and font reconciliation command.
- **D3 Provider collision UX:** allow deliberate reconfiguration of an existing provider through a redacted diff plus a second replacement confirmation. Never merge old and new provider internals field-by-field.
- **D4 Command names:** use `/preset-skills-sync` for upstream skill install/refresh and `/preset-models-add` for the provider wizard.
- **D5 Models wizard scope:** model metadata remains fixed, but membership is user-selected per run from an initially empty checklist.
