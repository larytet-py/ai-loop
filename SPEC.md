# ai-loop - specification

Status: draft v0.4. This document specifies behavior only. No implementation exists yet.
Companion document: `TESTPLAN_SPEC.md` (the test plan format and how it is produced).

## 1. Purpose

ai-loop drives a coding agent through a fixed, enforced workflow for a single tracked task: understand the ticket, propose options, wait for a human decision, write the test plan, implement, verify on a test node, and pass an independent code review. The order of stages and the human gates are enforced by code, not by prompt instructions.

## 2. Non-goals

- Committing, pushing, or opening merge requests. The human does that.
- Running unit tests locally. Unit tests and the CI e2e suite run in CI after the human pushes. ai-loop writes or adjusts them, and runs targeted checks on the test node only.
- Choosing the test node. The human supplies it.
- Running more than one ticket at a time.

## 3. Assumptions about the environment

- Where the code lives is given by the `--repo` parameters, not by a skill. A skill (outside this repo) tells the agent how to deploy a cluster (`skaffold run`), how to reach a node over ssh, and the UI URL pattern. ai-loop does not hard-code any of this and must not name any product or company.
- Passwordless sudo is available on the test node. Skaffold and docker need no password.
- Jira is reachable through an MCP server for reads, and through REST with an API token for attachments and writes. Credentials come from environment variables.
- Tools on the machine running ai-loop: `ffmpeg`, `tesseract`, `git`, `ssh`, Python 3.11+, and the linters named in section 8.

## 4. Input parameters

Required:

| Name | Meaning | Validation |
|---|---|---|
| `task` | Task key, for example `SHOP-178` | Matches `^[A-Z][A-Z0-9]+-[0-9]+$` |
| `test_node` | The node used for deploy and tests | Resolved to an ssh target and a UI URL (see 4.1). Reachability checked before stage 1 |
| `repo` | One or more source folders that may contain the code for the task, for example `~/repo-a` and `~/repo-b` | Repeatable. Each must be a git repository. Syntax in 4.2 |

Optional:

| Name | Default | Meaning |
|---|---|---|
| `worktree_root` | `~` | Directory in which per-repo worktrees named after the ticket are created, see 4.2 and stage 0 |
| `run_dir` | `./runs/<task>/<timestamp>` | Where run artifacts are written |
| `task_type` | auto | `backend`, `ui`, or `mixed`. Auto-detected from the ticket and confirmed at the first human gate |
| `max_review_rounds` | 3 | Reviewer-driven rework cap |
| `resume` | off | Continue an interrupted run from its last completed stage |
| `model` | `claude-sonnet-5-5` | Used for every agent role |
| `non_interactive` | off | Fail instead of prompting (for testing ai-loop itself) |

There is no wall-clock or token cap. See 4.1 for the node lifetime.

Environment variables: `JIRA_USER_NAME`, `JIRA_API_TOKEN`, plus whatever the ssh and deploy skills require.

### 4.2 Source folders

The task may touch more than one component, so the run takes a list of folders to search for the code. Each `--repo` value has the form:

```
<path>[@<base_ref>][:ro]
```

- `path`: a git repository on this machine.
- `@<base_ref>`: the ref diffs are measured against for this repository. Default `origin/development`, or the repository's default remote branch if that ref does not exist.
- `:ro`: the repository is read-only for this run. The analyst and implementer can read it for context, and no worktree is created for it.

Example, with placeholder names:

```
ai-loop SHOP-178 --test-node node-0393 --repo ~/repo-a --repo ~/repo-b@origin/main --repo ~/repo-c:ro
```

Rules:

- The analyst, test planner, and reviewer all search every listed repository. Option files and test cases carry the repository they belong to.
- Every run starts by creating a git worktree (stage 0), before the ticket is read. A worktree is created for each repository that is not `:ro`. It is created under `worktree_root`, from that repository's `base_ref`, on a new branch named after the ticket. It is named after the ticket, with the repository name as a prefix when more than one repository is writable (for example `~/<repo>-<task>`, or `~/<task>` when there is exactly one).
- The primary worktree is the one of the first non-`:ro` repository on the command line. Generated documents (the proposal and the test plan) are written into its `ai-loop/` folder, see 4.3.
- A worktree that ends up with no diff is reported at stage 8 and left for the human to remove.
- Each changed repository has its own diff against its own `base_ref`.
- If a branch or worktree named after the ticket already exists and was not created by this run, the orchestrator does not reuse it and does not let any agent read it, because it may already hold earlier work on the same ticket. It asks the human to choose: use a suffixed name (default, `<task>-ailoop`), or explicitly point the run at the existing one.

### 4.3 The `ai-loop/` folder

Inside the primary worktree, `ai-loop/` holds the documents the run generates for that ticket: `proposal.md`, `test_plan.md`, and `test_plan.sha256`. The orchestrator adds `ai-loop/` to the worktree's `info/exclude`, so it never shows in `git status` or in any diff, and it can never be committed by accident. Agents may write there before approval. They may not write anywhere else in the worktree until the gates in section 8 are passed.
- The deploy skill decides how changes in several repositories reach the test node. ai-loop only passes it the list of changed worktrees.

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
- Repositories: verify every `--repo` path is a git repository and that its `base_ref` resolves. Fetch remotes so the base is current.
- Worktrees (always the first thing that is created): create a worktree and ticket branch for every non-`:ro` repository, from its `base_ref`, as described in 4.2. Create the `ai-loop/` folder in the primary worktree and exclude it from git (4.3). If a worktree or branch named after the ticket already exists, apply the collision rule in 4.2 before going on.
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

- Role: analyst. Input: `ticket.md` and the code. Output: `proposal.md`, written to `ai-loop/proposal.md` in the primary worktree.
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
- Output: `ai-loop/test_plan.md` in the primary worktree, frozen with a content hash (`ai-loop/test_plan.sha256`) before implementation. Neither the implementer nor the UI tester can modify it. A change requires rerunning this stage and is logged.
- The human sees the test plan summary and confirms it before stage 5 starts. This is a second, lighter gate. The human may add cases or remove cases, and the edit is logged.
- Validator: schema check, and every targeted case traces to a statement in the ticket or the approved option.

### Stage 5 - Implementation

- Role: implementer. Input: `ticket.md`, `approval.json`, the approved option, and `test_plan.md` (so it knows which unit and e2e tests to add or adjust).
- Edits only inside the worktrees created at stage 0, and only in the repositories the approved option changes. Follows each repository's own conventions file.
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
- For each changed repository, the only diff under review is the diff between that repository's `base_ref` and its branch head. Git history, earlier commits, and the MR's earlier iterations are context only, never the thing being compatible with. Cross-repository effects (for example a changed interface used by another listed repository) are in scope for the review.
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
| analyst | ticket, code | `ai-loop/proposal.md` in the primary worktree, nothing else | Cannot edit code |
| test planner | ticket, approved option, code, test inventory | `ai-loop/test_plan.md` in the primary worktree, nothing else | Runs before any code is generated, never sees a diff |
| implementer | ticket, approval, plan, code | workdir | Cannot change the plan |
| UI tester | frozen plan, node | evidence and `test_report.md` | Cannot edit the plan or the code |
| reviewer | diff, ticket, approved option | `review.md` only | Fresh context, never sees implementer notes |

Each role is a separate agent session with its own context and tool allowlist. Information passes between roles only through artifact files.

## 8. Enforcement

The workflow must not depend on the model following instructions.

- The orchestrator is a state machine in code. Stage order is fixed. A stage runs only if the previous exit artifact exists and its validator passed.
- Path gate: before approval, the hook allows writes only to the `ai-loop/` folder of the primary worktree. Every other path in every worktree is denied.
- Approval gate: the hook denies writes to code paths, and deploy commands, until `approval.json` exists with a signature produced by the orchestrator. The same hook keeps code paths denied until the test plan is frozen and confirmed.
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
  approval.json
  implementation_notes.md
  diff-<repo>.patch       one per changed repository, base_ref..HEAD
  deploy.log
  test_report.md
  evidence-<task>.tar.gz  GIFs or screenshots, backend logs, report
  review.md
  summary.md
  events.jsonl
  state.json
tasks/<task>/status.md
```

In the primary worktree (excluded from git, see 4.3):

```
<worktree>/ai-loop/
  proposal.md
  test_plan.md
  test_plan.sha256
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
- The tool writes only inside the ticket worktrees it created, never in the original `--repo` checkouts, never in a `:ro` repository, and touches only the given test node and the one Jira ticket.

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
5. The first action of every run, after the task key is validated and before intake starts, is creating a worktree and ticket branch from `base_ref` for each non-`:ro` repository. `proposal.md` and `test_plan.md` appear in the primary worktree's `ai-loop/` folder and are absent from `git status` and from the diff. A `:ro` repository is never written to.
5a. If a branch or worktree named after the ticket already exists and was not created by the run, the run stops and asks, uses a suffixed name by default, and no agent reads the existing one.
5b. Before approval, an agent attempt to write to any path other than `ai-loop/` is denied and logged.
6. A deliberately injected UI regression is shown in the report and the orchestrator waits for a human decision instead of passing or failing it.
7. The evidence archive contains GIFs or screenshots, backend logs, and the report, and is attached to the ticket only after the human confirms at stage 8.
8. An injected unused variable, a duplicated block, a comment mentioning a previous iteration, and a shim for an earlier commit of the same MR are each reported in stage 7.
9. A run interrupted after stage 4 resumes at stage 5 without repeating intake or re-asking for approval, including after the test node was recycled.
10. `status.md` reflects every decision and tradeoff made during the run.
11. No file in the repository contains a product or company name.
