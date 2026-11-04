# Test plan - specification

Status: draft v0.1. Companion to `SPEC.md`, stage 4.

## 1. Purpose

The test plan defines how a change is verified, and it is written before any code for the change exists. It is produced by a dedicated agent session that never sees a diff, so the plan cannot be fitted to the implementation. It is frozen by content hash before implementation starts.

## 2. Inputs and outputs

Input: the normalized ticket, the approved option, the current code at the base ref, and an inventory of existing unit, e2e, and UI tests.

Output: `ai-loop/test_plan.md` and `ai-loop/test_plan.sha256` in the primary worktree of the run. The `ai-loop/` folder is created at stage 0 and excluded from git, so the plan is never part of the ticket's diff. The human reviews and may add or remove cases before the freeze.

## 3. Sections by task type

### 3.1 Backend (and the backend part of mixed)

- Code paths: the modules, functions, and endpoints that the approved option touches or depends on, each with a short reason.
- Unit tests: tests to add, and existing tests to adjust, each with file, test name, and what it asserts. Include the boundary cases, error paths, and any date or time edge relevant to the change.
- E2e tests: scenarios to add or adjust in the project's e2e suite, with preconditions and expected outcomes.
- Targeted checks on the test node: commands or API calls to run after deploy, with expected results, and which service logs to collect and what to look for in them.
- Existing tests expected to change, with the reason each change is legitimate. A test that is changed only to make it pass is not allowed.
- Unit and CI e2e tests are run in CI after the human pushes. The plan marks which cases run on the node and which run only in CI.

### 3.2 UI (and the UI part of mixed)

- Targeted cases that exercise the change.
- Regression cases that guard neighbouring behavior: pages and components that share code, data, or routes with the change.
- A smoke set covering core navigation and the main data views.
- Per case: preconditions (login, data), steps, expected result, and the artifact to capture (animated GIF, or a sequence of screenshots).

## 4. Case format

Each case has:

- `id`: stable, for example `T-01`
- `kind`: `unit`, `e2e`, `node-check`, `ui-targeted`, `ui-regression`, or `ui-smoke`
- `runs_on`: `ci`, `node`, or `browser`
- `title`
- `preconditions`
- `steps`
- `expected`
- `trace`: the ticket statement or approved-option item the case covers
- `capture`: what evidence to keep (log excerpt, GIF, screenshots)

## 5. Rules

- Every targeted case has a `trace`. A case with no trace is rejected by the validator.
- The plan must not reference any file added or changed by the implementation, because none exists yet. It references the existing code and states new tests by intended name and location.
- The plan states explicitly what it does not cover and why.
- After the freeze the plan is read-only. Adding a case later means rerunning the planner stage and recording the reason in `status.md`.
- The UI tester executes every UI case and reports a result for each. A case with no result fails the stage.
- The plan never defines a pass or fail threshold for regressions. Failures are shown to the human, who decides.

## 6. Evidence

After execution the orchestrator packs the GIFs or screenshots, backend log excerpts, the results table, and the deploy log into one archive per run, so it can be attached to the ticket as a single file once the human confirms.
