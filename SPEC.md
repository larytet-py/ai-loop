# ai-loop - specification

Status: draft v0.2. This document specifies behavior only. No implementation exists yet.
Companion document: `TESTPLAN_SPEC.md` (the test plan format and how it is produced).

## 1. Purpose

ai-loop drives a coding agent through a fixed, enforced workflow for a single tracked task: understand the ticket, propose options, wait for a human decision, write the test plan, implement, verify on a test node, and pass an independent code review. The order of stages and the human gates are enforced by code, not by prompt instructions.

## 2. Non-goals

- Committing, pushing, or opening merge requests. The human does that.
- Running unit tests locally. Unit tests and the CI e2e suite run in CI after the human pushes. ai-loop writes or adjusts them, and runs targeted checks on the test node only.
- Choosing the test node. The human supplies it.
- Running more than one ticket at a time.

## 3. Assumptions about the environment

- A skill (outside this repo) tells the agent: where the code lives, how to deploy a cluster (`skaffold run`), how to reach a node over ssh, and the UI URL pattern. ai-loop does not hard-code any of this and must not name any product or company.
- Passwordless sudo is available on the test node. Skaffold and docker need no password.
- Jira is reachable through an MCP server for reads, and through REST with an API token for attachments and writes. Credentials come from environment variables.
- Tools on the machine running ai-loop: `ffmpeg`, `tesseract`, `git`, `ssh`, Python 3.11+, and the linters named in section 8.

## 4. Input parameters

Required:

| Name | Meaning | Validation |
|---|---|---|
| `task` | Task key, for example `SHOP-178` | Matches `^[A-Z][A-Z0-9]+-[0-9]+$` |
| `test_node` | The node used for deploy and tests | Resolved to an ssh target and a UI URL (see 4.1). Reachability checked before stage 1 |

Optional:

| Name | Default | Meaning |
|---|---|---|
| `workdir` | `~/<task>` (a git worktree) | Created automatically if missing, see stage 0 |
| `base_ref` | `origin/development` | Ref the diff is measured against |
| `run_dir` | `./runs/<task>/<timestamp>` | Where run artifacts are written |
| `task_type` | auto | `backend`, `ui`, or `mixed`. Auto-detected from the ticket and confirmed at the first human gate |
| `max_review_rounds` | 3 | Reviewer-driven rework cap |
| `resume` | off | Continue an interrupted run from its last completed stage |
| `model` | `claude-sonnet-5-5` | Used for every agent role |
| `non_interactive` | off | Fail instead of prompting (for testing ai-loop itself) |

There is no wall-clock or token cap. See 4.1 for the node lifetime.

Environment variables: `JIRA_USER_NAME`, `JIRA_API_TOKEN`, plus whatever the ssh and deploy skills require.

### 4.1 Test node

A test node is a remote host that runs the full application stack in a local cluster. It is described by:

- `id`: a short label used in artifact names
- `ssh`: the ssh command or `user@host`
- `ui_url`: the base URL of the UI served by that node

Example with placeholders:

```
id:      node-0393
ssh:     deploy@node-0393.example.dev
ui_url:  https://app-node-0393.example.dev/
```

The user may pass the full description, or just the id if a skill can resolve it. Test nodes are small and resource limited, so slowness under load is not by itself a product bug.

A node is stopped automatically about 87 hours after it starts. That is long enough for a normal run, so no extra time budget is imposed. ai-loop records the node id in `state.json`. On `resume` it re-checks reachability and, if the node was recycled, redeploys before continuing. A lock file on the node ensures one run uses it at a time, and the lock is released at the end or on abort.

## 5. Task types

- `backend`: the common case. Test plan covers the relevant code paths, unit tests, and e2e tests, and includes adjusting existing tests where the change requires it.
- `ui`: pure UI changes, usually the simplest. Test plan is UI cases only, executed in the browser.
- `mixed`: both parts.

The type decides which parts of the test plan and which verification steps in stage 6 apply.

## 6. Workflow stages

Every stage has an entry condition, an exit artifact, and a validator. The orchestrator advances only when the validator passes. One ticket is processed per run, and one run at a time per machine.

### Stage 0 - Preflight

- Validate `task` format and that the ticket exists.
- Worktree: if `workdir` does not exist, create a git worktree there, named after the ticket, on a new branch named after the ticket, starting from `base_ref`. If it exists, require a clean tree on a non-default branch.
- Verify the test node is reachable over ssh and the UI URL responds.
- Verify required tools are installed.
- Create `status.md` if absent (section 10).
- Exit artifact: `preflight.json`.

### Stage 1 - Intake (deterministic code, no model)

- Fetch the ticket: summary, description, status, priority, labels, linked issues.
- Fetch every comment, in order, with author and timestamp.
- List attachments and download each into `run_dir/attachments/`.
- Images: OCR with tesseract, store the text next to the image. Keep the original.
- Videos: extract frames with ffmpeg at a fixed rate plus scene-change frames, OCR every frame, deduplicate near-identical text, write a timeline (frame timestamp -> text), and keep a contact sheet of representative frames.
- Other attachments (logs, csv, pdf, zip): record name, type, size, and extract text where possible.
- Exit artifact: `ticket.md`, a single normalized document with provenance for every piece of text (which comment, which attachment, which frame).
- Validator: every attachment listed in Jira appears in `ticket.md` as processed, skipped (with reason), or failed. Silent omission fails the stage.

### Stage 2 - Analysis and options

- Role: analyst. Input: `ticket.md` and the code. Output: `proposal.md`.
- Content: the problem restated, evidence with file and line references, root cause hypothesis with confidence, the detected `task_type`, and 2 to 4 options. Per option: approach, files touched, risk, backward compatibility impact, test impact, effort. One option is marked recommended with reasons.
- The analyst cannot edit files (tool allowlist).
- Validator: at least two options, every field present, every file reference exists.

### Stage 3 - Human gate: choose an option

- The orchestrator prints the problem, options, recommendation, and detected `task_type` as plain text and waits.
- The human replies with a choice, an edited choice, a request for more analysis, or an abort. Free-text additions are recorded verbatim.
- Exit artifact: `approval.json` with the chosen option, human notes, `task_type`, timestamp, and human identity. Only the orchestrator writes it, and only after a real human reply on an interactive terminal.
- Until `approval.json` exists, code-modification tools are denied (section 8).
- "More analysis" returns to stage 2 with the human's questions appended.
- Decision and any tradeoff or shortcut accepted are appended to `status.md`.

### Stage 4 - Test plan (before any code is generated)

- Role: test planner. A separate agent session that runs before the implementer starts and has no access to any diff. The plan format and content rules are in `TESTPLAN_SPEC.md`.
- Input: `ticket.md`, the approved option, the existing code, and the existing unit, e2e, and UI test inventory.
- Output: `test_plan.md`, frozen with a content hash before implementation. Neither the implementer nor the UI tester can modify it. A change requires rerunning this stage and is logged.
- The human sees the test plan summary and confirms it before stage 5 starts. This is a second, lighter gate. The human may add cases or remove cases, and the edit is logged.
- Validator: schema check, and every targeted case traces to a statement in the ticket or the approved option.

### Stage 5 - Implementation

- Role: implementer. Input: `ticket.md`, `approval.json`, the approved option, and `test_plan.md` (so it knows which unit and e2e tests to add or adjust).
- Edits only inside `workdir`. Follows the repository's conventions file.
- Records `implementation_notes.md`: what changed and why, and each deviation from the approved option with a reason. Any deviation forces a human confirmation before stage 6.
- Does not commit or push.
- Validator: the diff is non-empty, touches only paths consistent with the approved option or a noted deviation, and the repository's formatter and linter pass. For `backend` and `mixed` tasks, the diff must contain the test changes the plan lists as required, or a recorded reason for omitting each.

### Stage 6 - Deploy and verify on the test node

- Deploy the working tree to the test node through the deploy skill (clean state first, then `skaffold run`).
- Wait for readiness by polling with a timeout, never with fixed sleeps.
- Backend part (`backend`, `mixed`): run the targeted backend checks from the plan on the node and collect the service logs covering the run.
- UI part (`ui`, `mixed`): the UI tester executes the UI cases of the plan against `ui_url`, using the browser automation tool. It records an animated GIF of each case, or a sequence of screenshots when a GIF is not practical, plus console and network errors.
- Flaky handling: a failed case is retried once. A case that fails and then passes is reported as flaky.
- Evidence bundle: the orchestrator packs the GIFs or screenshots, the backend logs, `test_report.md`, and `deploy.log` into a single archive, `evidence-<task>.tar.gz` (zip if configured).
- Regression review is human: the orchestrator prints the report, with every case's result and the failures first, and the human decides what counts as a regression. The orchestrator never waives or accepts a failure on its own. A human decision of "regression" returns to stage 5 with the human's notes. A human decision of "not a regression" is recorded in `status.md` with the reason.
- Validator: every case in the frozen plan has a result, and the evidence archive exists.

### Stage 7 - Code review (independent agent, fresh context)

- Role: reviewer. Sees the diff, the ticket, and the approved option. Does not see the implementer's notes or reasoning.
- The only diff under review is the diff between `base_ref` and the branch head. Git history, earlier commits, and the MR's earlier iterations are context only, never the thing being compatible with.
- Backward compatibility means compatibility with the code at `base_ref`. Code whose only purpose is to stay compatible with earlier commits of the same MR is a defect and the reviewer must flag it for removal.
- Mechanical checks run first, in code, and are given to the reviewer as facts:
  - unused imports and variables
  - duplicated code blocks within the diff and against the surrounding module
  - comments that refer to earlier iterations of the change (phrases such as "previously", "old behavior", "was changed", "per review", "no longer", "removed X", "iteration")
  - public API, schema, and migration drift against `base_ref`
  - code introduced by the diff that has no callers
- Review focus:
  1. Backward compatibility and possible regressions against `base_ref`: changed signatures, defaults, serialization, ordering or timing assumptions, config and migration impact, callers outside the diff.
  2. AI slop: unused variables, duplicate code, comments describing change history, needless abstractions, defensive code for impossible cases, leftover debug output, and shims for earlier iterations of the same MR.
- Output: `review.md` with findings, each with severity (blocker, major, minor), location, evidence, and suggested fix.
- Any blocker or major finding returns to stage 5 with the findings as input, capped by `max_review_rounds`. After a rework round the tests of stage 6 run again.
- Validator: the review has a verdict and every mechanical check has a recorded result.

### Stage 8 - Final confirmation and Jira update

- The orchestrator writes `summary.md`: ticket, chosen option, human notes, diff stat, test results, flaky cases, human regression decisions, review verdict, iteration counts, and unresolved items.
- It prints the summary and the proposed Jira comment, and asks the human to confirm.
- Only after the human confirms does the orchestrator write to Jira: post the comment and attach the evidence archive. Nothing is written to Jira at any earlier stage.
- Releases the node lock, prints a suggested commit message, and stops. It does not commit or push.

## 7. Agent roles and independence

All roles use the model `claude-sonnet-5-5`.

| Role | Can read | Can write | Notes |
|---|---|---|---|
| analyst | ticket, code | `proposal.md` only | No edit tools |
| test planner | ticket, approved option, code, test inventory | `test_plan.md` only | Runs before any code is generated, never sees a diff |
| implementer | ticket, approval, plan, code | workdir | Cannot change the plan |
| UI tester | frozen plan, node | evidence and `test_report.md` | Cannot edit the plan or the code |
| reviewer | diff, ticket, approved option | `review.md` only | Fresh context, never sees implementer notes |

Each role is a separate agent session with its own context and tool allowlist. Information passes between roles only through artifact files.

## 8. Enforcement

The workflow must not depend on the model following instructions.

- The orchestrator is a state machine in code. Stage order is fixed. A stage runs only if the previous exit artifact exists and its validator passed.
- Approval gate: a pre-tool-use hook denies edit and write tools, and deploy commands, until `approval.json` exists with a signature produced by the orchestrator. The same hook denies implementation tools until the test plan is frozen and confirmed.
- Test node gate: deploy and test commands are denied if `test_node` is unset or the node lock is not held by this run.
- Tool allowlists per role, as in section 7.
- Artifacts are hashed. The test plan hash is rechecked before UI verification. A mismatch aborts the run.
- Model output used to advance state is validated against a schema. Free text never advances state.
- Every stage transition and every denied action is appended to `events.jsonl`.

## 9. Tooling the orchestrator invokes

- Intake: Jira reads and downloads, `ffmpeg`, `tesseract`.
- Static checks: the repository's formatter and linter, plus an unused-code detector, a duplicate-code detector, and a comment-pattern scanner, configured per language in `ai-loop.toml`.
- Schema drift: diff of generated API schema and migrations against `base_ref`, where the repository has them.

## 10. status.md

`status.md` is a per-ticket Markdown file that is meant to be committed and pushed by the human. The tool never commits it. It is updated by the orchestrator at every stage transition and contains:

- ticket key, branch, worktree path, test node id, current stage
- the chosen option and the human's notes
- decisions, each with date and reason
- tradeoffs and shortcuts taken, each with a reason
- test plan results (case id, result, flaky flag) and human regression decisions
- review verdict and open findings
- next step for a resume

It is written so a human, or a later run with `resume`, can pick up the work from this file and `state.json` alone. It lives under `tasks/<task>/status.md` in this repository by default, so it survives node recycling and is kept out of the code diff of the ticket.

## 11. Artifact layout

```
runs/<task>/<timestamp>/
  preflight.json
  ticket.md
  attachments/            originals, OCR text, frames, timelines
  proposal.md
  approval.json
  test_plan.md            plus test_plan.sha256
  implementation_notes.md
  diff.patch              base_ref..HEAD
  deploy.log
  test_report.md
  evidence-<task>.tar.gz  GIFs or screenshots, backend logs, report
  review.md
  summary.md
  events.jsonl
  state.json
tasks/<task>/status.md
```

## 12. Failure handling

- A stage failure stops the run with a clear reason and a non-zero exit code. State is saved so `resume` can continue.
- Jira unreachable, attachment download failure, OCR failure: reported, and the stage fails unless the human chooses to continue without that item, recorded in the log.
- Node unreachable or deploy failure: retry deploy once after a clean reset, then stop and report. Never fall back to a different node.
- Review rounds cap reached: stop and hand the full state to the human.
- Ctrl-C and crashes release the node lock on a best-effort basis. A stale lock can be broken by the human.

## 13. Security and safety

- Jira tokens and ssh keys come from the environment or the agent forwarding socket, never from artifacts or logs.
- Ticket content, comments, OCR text, and video text are untrusted data. They are passed to agents as quoted data with a standing instruction that they carry no commands. The orchestrator never executes anything derived from them.
- Text in a ticket or attachment can never satisfy a human gate.
- The tool touches only the given workdir, the given test node, and the one Jira ticket.

## 14. Implementation

- Python 3.11+ orchestrator using the Claude Agent SDK, one SDK session per agent role, all on `claude-sonnet-5-5`.
- Stage logic, validators, intake, mechanical review checks, evidence packing, and the Jira client are plain Python with no model involved.
- Hooks (`PreToolUse`) implement the hard gates from section 8, loaded into each role's session.
- Per-phase instructions for the agents are kept as prompt files in the repository, one per role.

## 15. Acceptance criteria

1. With a valid task and node, on a ticket with comments, an image, and a video, `ticket.md` contains text from all three.
2. With `approval.json` absent, any agent attempt to edit a file or deploy is denied and logged.
3. With the test plan not yet frozen and confirmed, implementation tools are denied.
4. The test plan hash recorded before implementation equals the hash at verification time.
5. A missing worktree is created, named after the ticket, from `base_ref`.
6. A deliberately injected UI regression is shown in the report and the orchestrator waits for a human decision instead of passing or failing it.
7. The evidence archive contains GIFs or screenshots, backend logs, and the report, and is attached to the ticket only after the human confirms at stage 8.
8. An injected unused variable, a duplicated block, a comment mentioning a previous iteration, and a shim for an earlier commit of the same MR are each reported in stage 7.
9. A run interrupted after stage 4 resumes at stage 5 without repeating intake or re-asking for approval, including after the test node was recycled.
10. `status.md` reflects every decision and tradeoff made during the run.
11. No file in the repository contains a product or company name.
