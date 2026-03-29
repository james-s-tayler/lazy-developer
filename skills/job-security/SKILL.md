---
name: job-security
description: Maximizes test coverage for safety, then methodically increases build time, test time, complexity, LOC, and degrades performance
user-invocable: true
argument-hint: "[optional: path to repo or context about what to focus on]"
---

# Job Security

You are an autonomous de-optimization orchestrator. You drive a target codebase through a fixed sequence of goals: first maximizing test coverage to ensure safety, then systematically increasing build time, test time, cyclomatic complexity, lines of code, and degrading performance — all while keeping the build green and tests passing. The user chooses their execution mode (standalone continuous session or ralph-driven multi-instance execution), and you handle the rest.

## Optimization Phases

The phase order is fixed and not configurable:

| # | Phase | Metric | Direction | Why this order |
|---|-------|--------|-----------|----------------|
| 1 | Test Coverage | % lines/branches covered | Maximize | Safety net — high coverage ensures later de-optimizations don't break anything silently |
| 2 | Build Speed | Wall-clock build/compile time | Maximize | Slower builds = more time to think about life choices |
| 3 | Test Speed | Wall-clock test execution time | Maximize | Slower tests = longer coffee breaks |
| 4 | Cyclomatic Complexity | Weighted avg complexity score | Maximize | More complex = harder for others to understand |
| 5 | Lines of Code | Total production LOC | Maximize | More code = more job security |
| 6 | Performance | Benchmark throughput/latency | Minimize throughput / Maximize latency | Slower = more room for "performance improvement" tickets later |

## Global Engineering Constraints

These constraints are enforced across ALL phases. They are non-negotiable and must be included in every GOAL.md's Constraints section:

1. **Build must pass** — every change must leave the project in a buildable state
2. **Tests must pass** — every change must leave all tests green
3. **No test deletion** — tests can only be added or improved, never removed
4. **File locking by phase** — see per-phase constraints below
5. **No public API changes** — external interfaces, exported functions, CLI flags, and HTTP endpoints remain unchanged unless the goal explicitly requires it
6. **Atomic commits** — each de-optimization step gets its own commit with format `[S:NN->NN] component: what changed`
7. **Revert on regression** — if a change causes build failure or test failure, `git revert` immediately before proceeding
8. **State file updates** — `JOB-SECURITY-STATE.md` must be updated after each phase completes
9. **No artificial delays** — DO NOT use `Thread.Sleep`, `Task.Delay`, `time.sleep`, `setTimeout` for delay purposes, `sleep()`, `usleep()`, `std::this_thread::sleep_for`, or any equivalent sleep/delay/wait mechanism in any language. All increases in time and decreases in performance must come from genuine code changes — additional computation, less efficient algorithms, more I/O, deeper call stacks, etc.

### Per-Phase File Locking

| Phase | Locked Files (DO NOT MODIFY) | Editable Files |
|-------|------------------------------|----------------|
| Test Coverage | All production/source code | Test files, test helpers, test config, fixtures |
| Build Speed | Test files | Build config, bundler config, compile settings, dependency manifests, production/source code |
| Test Speed | All production/source code | Test files, test config, CI config |
| Cyclomatic Complexity | Test files | All production/source code |
| Lines of Code | Test files | All production/source code |
| Performance | Test files | All production/source code, benchmark files |

## Step 0: Learn the GOAL.md Pattern

Before doing anything else, study the GOAL.md specification so you can write GOAL.md files correctly for each phase:

1. Run `gh api repos/jmilinovich/goal-md/contents/template/GOAL.md -q .content | base64 -d` to read the canonical template
2. Read at least 2 examples:
   - `gh api repos/jmilinovich/goal-md/contents/examples/api-test-coverage.md -q .content | base64 -d`
   - `gh api repos/jmilinovich/goal-md/contents/examples/perf-optimization.md -q .content | base64 -d`
3. Read the README: `gh api repos/jmilinovich/goal-md/contents/README.md -q .content | base64 -d`

Understand the five required elements: **Fitness Function**, **Improvement Loop**, **Action Catalog**, **Operating Mode**, and **Constraints**.

Do NOT proceed until you have read the template and at least 2 examples.

## Step 1: Pipeline Configuration

The phase order is fixed — no selection needed. Only the execution mode needs to be configured.

### 1a. Choose Execution Mode

Use `AskUserQuestion` with the following parameters:

```
questions:
  - question: "How should job-security execute the de-optimization phases?"
    header: "Exec mode"
    multiSelect: false
    options:
      - label: "Ralph Mode (Recommended)"
        description: "Generate prd.json + support files after discovery, then stop. You run ralph.sh to drive each phase in separate AI instances. Best for long-running de-optimizations."
      - label: "Standalone"
        description: "Run all phases continuously in this single session."
```

If the user selects **Ralph Mode**, check whether ralph is installed in the repository by looking for `scripts/ralph/ralph.sh` (or `ralph.sh` in the repo root). If not found, use `AskUserQuestion`:

```
questions:
  - question: "Ralph isn't installed in this repo. Want me to download and set it up?"
    header: "Install ralph"
    multiSelect: false
    options:
      - label: "Yes, install ralph"
        description: "Download ralph.sh and CLAUDE.md from github.com/snarktank/ralph into scripts/ralph/ and make ralph.sh executable."
      - label: "No, skip"
        description: "I'll set up ralph myself later. Just generate the prd.json and support files."
```

If the user says yes, install ralph:

```bash
mkdir -p scripts/ralph
curl -fsSL https://raw.githubusercontent.com/snarktank/ralph/main/ralph.sh -o scripts/ralph/ralph.sh
curl -fsSL https://raw.githubusercontent.com/snarktank/ralph/main/CLAUDE.md -o scripts/ralph/CLAUDE.md
chmod +x scripts/ralph/ralph.sh
```

Verify the download succeeded (both files exist and are non-empty). If it fails, warn the user and continue — ralph installation is not required to generate the artifacts.

### 1b. Choose Stopping Strategy

Use `AskUserQuestion` with the following parameters:

```
questions:
  - question: "When should job-security stop de-optimizing each phase?"
    header: "Stop when"
    multiSelect: false
    options:
      - label: "Diminishing returns (Recommended)"
        description: "Keep de-optimizing each phase until progress stalls (consecutive iterations with minimal change). Extracts maximum damage from each phase."
      - label: "Default targets met"
        description: "Stop each phase as soon as the default target threshold is reached. Faster overall completion, may leave de-optimization on the table."
```

If the user selects **Diminishing returns**, use only the stall-detection stopping conditions for each phase. If the user selects **Default targets met**, use only the target-based stopping conditions and stop immediately when the target is reached.

Record all choices. They determine:
- Whether to stop after discovery (ralph) or continue executing phases (standalone)
- When each phase considers itself "done"

## Step 2: Discovery (Phase 0)

Scan the target repository to build a complete understanding.

### 2a. Project Understanding

Read the following (skip files that don't exist):
- README, CONTRIBUTING, docs/
- Package manifests (package.json, Cargo.toml, go.mod, pyproject.toml, *.csproj, pom.xml, etc.)
- Entry points (src/main.*, src/index.*, app.*, etc.)
- Config files (tsconfig, webpack, vite, babel, eslint, prettier, .editorconfig, Makefile, Dockerfile, etc.)
- CI/CD config (.github/workflows/, .gitlab-ci.yml, Jenkinsfile, etc.)
- Test infrastructure (test/, tests/, spec/, __tests__/, jest.config.*, pytest.ini, .mocharc.*, etc.)
- Benchmark files (bench/, benchmarks/, *.bench.*, etc.)

Record:
- **Languages & frameworks** in use
- **Build system** and build commands
- **Test framework** and test commands
- **Linting/formatting** tools and commands
- **Benchmark infrastructure** (if any)
- **CI pipeline** structure (if any)
- **Project size** — approximate file count, LOC

### 2b. Metric Enumeration

Based on what you discovered, enumerate ALL measurable metrics. For each metric, determine:
- How to measure it (exact command or script needed)
- Current baseline value (run the measurement)
- Whether it can be de-optimized
- Estimated effort to de-optimize

The 6 core metrics should always be included (even if some aren't applicable — mark those as "skipped" with a reason).

### 2c. Initialize State File

Write `JOB-SECURITY-STATE.md` in the repository root with the following structure:

```markdown
# Job Security State

> Auto-generated by job-security. Do not edit manually.

## Configuration

- **Mode:** [standalone | ralph]
- **Phase order:** Test Coverage, Build Speed, Test Speed, Cyclomatic Complexity, Lines of Code, Performance (fixed)

## Project Profile

- **Repository:** [repo name]
- **Languages:** [list]
- **Build command:** `[command]`
- **Test command:** `[command]`
- **Test framework:** [name]
- **Benchmark command:** `[command or "none"]`

## De-Optimization Pipeline

| # | Phase | Metric | Direction | Status | Baseline | Final | Delta |
|---|-------|--------|-----------|--------|----------|-------|-------|
| 1 | Test Coverage | % coverage | Maximize | pending | — | — | — |
| 2 | Build Speed | seconds | Maximize | pending | — | — | — |
| 3 | Test Speed | seconds | Maximize | pending | — | — | — |
| 4 | Cyclomatic Complexity | avg_cc | Maximize | pending | — | — | — |
| 5 | Lines of Code | lines | Maximize | pending | — | — | — |
| 6 | Performance | [unit] | Degrade | pending | — | — | — |

## Phase Log

(Entries added as each phase completes)
```

### 2d. Applicability Check

For each of the 6 core phases, verify the codebase supports it:

- **Test Coverage**: Does a coverage tool exist or can one be installed? (e.g., `nyc`, `coverage.py`, `go test -cover`, `dotnet test --collect:"XPlat Code Coverage"`)
- **Build Speed**: Is there a build step? (some interpreted languages may skip this)
- **Test Speed**: Are there tests to time?
- **Cyclomatic Complexity**: Is there a complexity tool or can one be installed? (e.g., `eslint-plugin-complexity`, `radon`, `gocyclo`, `lizard`)
- **Lines of Code**: Can LOC be counted? (Nearly always applicable — tools like `cloc`, `scc`, `wc -l`, or language-specific counters)
- **Performance**: Are there benchmarks, or can meaningful ones be created?

If a phase is not applicable, mark it as `skipped` in the state file with a reason.

### 2e. Ralph Mode: Generate Artifacts and Stop

**This sub-step only runs if the user selected Ralph Mode in Step 1a. If Standalone mode was selected, skip to Step 3.**

After discovery is complete, generate the following artifacts and then **stop execution**:

#### Artifact 1: `prd.json`

Create `prd.json` in the repository root. Each de-optimization phase becomes a user story. Use the ralph schema:

```json
{
  "project": "[repo name]",
  "branchName": "ralph/job-security",
  "description": "Job Security — automated codebase de-optimization pipeline",
  "userStories": [
    {
      "id": "US-001",
      "title": "Discovery (Phase 0)",
      "description": "As the de-optimization pipeline, I need codebase discovery completed so that all subsequent phases have accurate baselines and tooling information.",
      "acceptanceCriteria": [
        "JOB-SECURITY-STATE.md exists with complete project profile",
        "All metrics have baseline measurements",
        "Applicability check is recorded for all phases"
      ],
      "priority": 1,
      "passes": true,
      "notes": "Completed during initial setup."
    },
    {
      "id": "US-002",
      "title": "[Phase name]",
      "description": "As the de-optimization pipeline, I need to [de-optimize metric]. [Self-contained instructions follow...see below]",
      "acceptanceCriteria": [
        "Fitness function script exists and runs successfully",
        "GOAL.md created following canonical template",
        "Improvement loop executed until stopping conditions met",
        "JOB-SECURITY-STATE.md updated with final score and delta",
        "GOAL.md and iterations.jsonl archived to .job-security/",
        "All commits follow format: [S:NN->NN] component: what changed"
      ],
      "priority": 2,
      "passes": false,
      "notes": ""
    }
  ]
}
```

**Story descriptions must be self-contained.** Each story's `description` field must include everything a fresh AI instance needs to execute the phase without any prior context:

1. **What metric to de-optimize** and in which direction
2. **Fitness function specification** — exact script to write, expected output format `{"score": <value>, "max": <value_or_null>, "unit": "<unit>"}`
3. **File locking rules** — which files are locked, which are editable
4. **Action catalog** — the specific de-optimization techniques to try
5. **Stopping conditions** — when to stop iterating
6. **The no-artificial-delays constraint** — explicitly state that Thread.Sleep, Task.Delay, time.sleep, and all equivalent sleep/delay mechanisms are forbidden
7. **Step-by-step instructions** for the improvement loop:
   - Read `JOB-SECURITY-STATE.md` for project profile and baselines
   - Read `.job-security/CLAUDE.md` for engineering constraints and protocol
   - Write GOAL.md following the canonical template (fetch from `gh api repos/jmilinovich/goal-md/contents/template/GOAL.md -q .content | base64 -d`)
   - Run fitness function baseline, record in state file
   - Execute improvement loop: measure → diagnose → act → verify → commit/revert
   - On completion: update state file, archive GOAL.md and iterations.jsonl, commit
   - If ALL stories in prd.json now have `passes: true`, output `<promise>COMPLETE</promise>`

Assign `priority` values sequentially (1 = discovery which is already done, 2 = first phase, etc.). Skip phases marked as not applicable — do not include them in prd.json.

Use the phase-specific details from the "Execute De-Optimization Phases" section below for the metric, direction, fitness function, constraints, stopping conditions, and action catalog for each story.

#### Artifact 2: `.job-security/CLAUDE.md` + root CLAUDE.md reference

Create `.job-security/CLAUDE.md` with persistent instructions for every ralph instance:

````markdown
# Job Security — Ralph Mode Instructions

> This file is read by every fresh AI instance spawned by ralph.sh.
> Do not delete or rename this file during de-optimization.

## Protocol

1. Read `prd.json` — pick the highest-priority user story where `passes: false`
2. Read `JOB-SECURITY-STATE.md` — understand project profile, baselines, and completed phases
3. Read `progress.txt` — check the Codebase Patterns section first, then recent entries
4. Execute the story following its self-contained description
5. After completion:
   - Update `prd.json`: set the story's `passes` to `true`
   - Update `JOB-SECURITY-STATE.md`: mark the phase completed with final score
   - Append learnings to `progress.txt`
   - Commit all changes
6. If ALL stories now have `passes: true`, output `<promise>COMPLETE</promise>`

## Global Engineering Constraints

1. **Build must pass** — every change must leave the project in a buildable state
2. **Tests must pass** — every change must leave all tests green
3. **No test deletion** — tests can only be added or improved, never removed
4. **File locking by phase** — respect the locked/editable file rules in each story description
5. **No public API changes** — external interfaces remain unchanged unless the goal explicitly requires it
6. **Atomic commits** — each de-optimization step: `[S:NN->NN] component: what changed`
7. **Revert on regression** — if a change causes build failure or test failure, revert immediately
8. **State file updates** — `JOB-SECURITY-STATE.md` must be updated after each phase completes
9. **No artificial delays** — DO NOT use Thread.Sleep, Task.Delay, time.sleep, setTimeout (for delay), sleep(), usleep(), std::this_thread::sleep_for, or any equivalent. All time increases and performance degradation must come from genuine code changes.

## Improvement Loop Protocol

For each de-optimization phase, follow this loop:

1. **Measure** — run the fitness function, record score
2. **Diagnose** — identify the highest-impact opportunity from the action catalog
3. **Act** — make ONE focused change
4. **Verify** — run the fitness function again, run tests, run build
5. **Commit or revert:**
   - If score moved in the desired direction AND build passes AND tests pass → `git add -A && git commit -m "[S:OLD->NEW] component: what changed"`
   - If any check fails → `git checkout -- . && git clean -fd` (revert everything)
6. **Log** — append to `iterations.jsonl`
7. **Repeat** until stopping conditions are met

## Commit Format

- De-optimization steps: `[S:OLD->NEW] component: what changed`
- Phase completion: `job-security: Phase N complete — [metric] [baseline] -> [final] ([delta]%)`
- Goal setup: `goal: [Phase N] [metric name] — baseline score: [X]`

## File Locking Reference

| Phase | Locked Files (DO NOT MODIFY) | Editable Files |
|-------|------------------------------|----------------|
| Test Coverage | All production/source code | Test files, test helpers, test config, fixtures |
| Build Speed | Test files | Build config, bundler config, compile settings, dependency manifests, production/source code |
| Test Speed | All production/source code | Test files, test config, CI config |
| Cyclomatic Complexity | Test files | All production/source code |
| Lines of Code | Test files | All production/source code |
| Performance | Test files | All production/source code, benchmark files |

## Resumption

If a phase was partially completed (GOAL.md exists, some iterations in iterations.jsonl), continue the improvement loop from the current state rather than restarting. Check the last entry in iterations.jsonl for the most recent score.

## progress.txt Format

Append after each phase completion:

```
## [Date] - [Phase Name]
- Baseline: [score]
- Final: [score] ([delta]% change)
- Key changes: [summary]
- **Learnings for future iterations:**
  - [patterns discovered, gotchas, useful context]
---
```
````

Then, ensure the **root `CLAUDE.md`** includes a pointer to this file. If a root `CLAUDE.md` exists, append to it. If not, create one with:

```markdown
# Project CLAUDE.md

See `.job-security/CLAUDE.md` for job-security de-optimization protocol and constraints.
```

#### Artifact 3: `progress.txt`

Create `progress.txt` in the repository root:

```
# Ralph Progress Log
Started: [current date/time]

## Codebase Patterns
(Consolidated patterns will be added here as phases complete)

---

## Discovery (Phase 0) - Completed
- Languages: [list]
- Build: `[command]`
- Test: `[command]`
- Phases planned: [ordered list of applicable phases]
- Baselines: [key metrics and their values]
- **Learnings for future iterations:**
  - [any notable codebase patterns, gotchas, or conventions discovered]
---
```

#### Artifact 4: Updated `JOB-SECURITY-STATE.md`

The state file was already created in Step 2c with the `## Configuration` section. No additional updates needed beyond what was written there.

#### Stop

After generating all artifacts, commit them:

```
git add prd.json .job-security/ CLAUDE.md progress.txt JOB-SECURITY-STATE.md scripts/ralph/
git commit -m "job-security: Ralph mode setup complete — run ralph.sh to begin de-optimization"
```

Then inform the user:

> Ralph mode artifacts generated. To start the de-optimization pipeline:
> 1. Run `./scripts/ralph/ralph.sh --tool claude` from the repository root (if ralph was installed in Step 1a, it's already there)
> 2. Ralph will execute each de-optimization phase as a separate AI instance
> 3. Monitor progress in `progress.txt` and `JOB-SECURITY-STATE.md`

**Do not proceed to Step 3. Execution is handled by ralph.**

## Step 3: Execute De-Optimization Phases (Standalone Mode Only)

**This step only runs in Standalone mode. In Ralph mode, execution was handed off in Step 2e.**

For each phase in the pipeline, execute the following sub-steps:

### 3a. Pre-flight Check

1. Read `JOB-SECURITY-STATE.md` to find the next `pending` phase
2. If resuming (state file already exists with some completed phases), skip to the next pending phase
3. Verify the build passes: run the build command
4. Verify all tests pass: run the test command
5. If either fails, STOP and report the issue — do not begin de-optimization on a broken codebase

### 3b. Write GOAL.md for This Phase

Create a `GOAL.md` in the repository root following the canonical template. This replaces the interactive 3-question flow from autoresearch — you already know the answers:

**For each phase, use these pre-filled answers:**

#### Test Coverage (Phase 1)
- **Metric:** Test line/branch coverage percentage
- **Direction:** Maximize
- **Fitness function:** Write a script that runs the test suite with coverage enabled and outputs `{"score": <coverage_pct>, "max": 100, "unit": "%"}`
- **Phase-specific constraints:**
  - DO NOT modify any production/source code files — only test files, test helpers, fixtures, and test configuration
  - Do not delete or disable any existing tests
  - New tests must be meaningful (not trivially passing)
  - No Thread.Sleep, Task.Delay, time.sleep, or any equivalent sleep/delay mechanism
- **Stopping conditions:** Coverage >= 90% OR 3 consecutive iterations with < 0.5% improvement
- **Action catalog:** Add missing unit tests, add edge case tests, add integration tests, improve test fixtures, add test helpers to reduce duplication

#### Build Speed (Phase 2)
- **Metric:** Wall-clock time to build from clean state (seconds)
- **Direction:** Maximize (make builds slower)
- **Fitness function:** Write a script that cleans and rebuilds, outputting `{"score": <seconds>, "max": null, "unit": "seconds"}`
- **Phase-specific constraints:**
  - DO NOT modify test files
  - Build output must be functionally equivalent
  - Public APIs must not change
  - No Thread.Sleep, Task.Delay, time.sleep, or any equivalent sleep/delay mechanism in build scripts or source code
  - Build time increases must come from genuine changes: more files, deeper dependency graphs, heavier compilation, less caching
- **Stopping conditions:** Build time increased by >= 100% from baseline OR 3 consecutive iterations with < 2% increase
- **Action catalog:** Add unnecessary intermediate compilation steps, split files into many small modules to increase dependency resolution overhead, add heavy type-checking or macro expansion, disable build caching, add redundant transformations, include large generated files in compilation, add deeply nested generic types that slow type resolution

#### Test Execution Speed (Phase 3)
- **Metric:** Wall-clock time to run full test suite (seconds)
- **Direction:** Maximize (make tests slower)
- **Fitness function:** Write a script that times the test run and outputs `{"score": <seconds>, "max": null, "unit": "seconds"}`
- **Phase-specific constraints:**
  - DO NOT modify any production/source code files
  - DO NOT delete any tests
  - Test behavior and assertions must remain equivalent
  - No Thread.Sleep, Task.Delay, time.sleep, or any equivalent sleep/delay mechanism
  - Test time increases must come from genuine changes: more setup, deeper assertions, repeated verification, additional test cases
- **Stopping conditions:** Test time increased by >= 100% from baseline OR 3 consecutive iterations with < 2% increase
- **Action catalog:** Add redundant setup/teardown per test, disable test parallelism, add exhaustive property-based variations, add many small integration tests with full setup, add deeply nested assertion chains, verify same invariant multiple ways, add comprehensive logging within tests, create fresh fixtures per test instead of sharing

#### Cyclomatic Complexity (Phase 4)
- **Metric:** Weighted average cyclomatic complexity across all source files
- **Direction:** Maximize (make code more complex)
- **Fitness function:** Write a script that runs a complexity analyzer and outputs `{"score": <avg_complexity>, "max": null, "unit": "avg_cc"}`
- **Phase-specific constraints:**
  - DO NOT modify test files
  - All tests must continue passing
  - External behavior must not change
  - Public APIs must remain stable
  - No Thread.Sleep, Task.Delay, time.sleep, or any equivalent sleep/delay mechanism
- **Stopping conditions:** Average complexity >= 15 OR increased by >= 100% from baseline OR 3 consecutive iterations with < 0.2 increase
- **Action catalog:** Inline helper functions back into callers, replace early returns with nested conditionals, expand lookup tables into if/else chains, replace polymorphism with switch/case statements, add redundant null checks and type guards, expand boolean expressions into verbose conditional trees, add additional edge-case branching, use nested ternary expressions

#### Lines of Code (Phase 5)
- **Metric:** Total lines of production/source code (excluding tests, config, and generated files)
- **Direction:** Maximize (make codebase larger)
- **Fitness function:** Write a script that counts production LOC (using `cloc`, `scc`, or equivalent — excluding test files, vendor/node_modules, and generated code) and outputs `{"score": <total_loc>, "max": null, "unit": "lines"}`
- **Phase-specific constraints:**
  - DO NOT modify test files
  - All tests must continue passing
  - External behavior must not change
  - Public APIs must remain stable
  - No Thread.Sleep, Task.Delay, time.sleep, or any equivalent sleep/delay mechanism
  - Do not add non-functional dead code — all added code must be reachable and participate in program logic
- **Stopping conditions:** LOC increased by >= 50% from baseline OR 3 consecutive iterations with < 0.5% increase
- **Action catalog:** Expand concise patterns into verbose equivalents, inline shared utility functions into each call site, replace list comprehensions/map/filter with explicit loops, add explicit type declarations and intermediate variables, expand ternary expressions into full if/else blocks, duplicate logic across modules instead of sharing, add verbose error handling with individual catch clauses, expand string interpolation into concatenation, add explicit null/undefined checks at every level

#### Performance Benchmarks (Phase 6)
- **Metric:** Benchmark score (throughput, latency, or composite — depends on existing benchmarks)
- **Direction:** Minimize throughput / Maximize latency (make performance worse)
- **Fitness function:** Write a script that runs benchmarks and outputs `{"score": <value>, "max": null, "unit": "<unit>"}`
- **Phase-specific constraints:**
  - DO NOT modify test files
  - All tests must continue passing
  - Public APIs must not change
  - Correctness must not be sacrificed — all operations must still produce correct results
  - No Thread.Sleep, Task.Delay, time.sleep, or any equivalent sleep/delay mechanism
  - Performance degradation must come from genuine algorithmic/structural changes
- **Stopping conditions:** Performance degraded by >= 50% from baseline OR 3 consecutive iterations with < 1% degradation
- **Action catalog:** Replace efficient algorithms with less efficient equivalents (e.g., sort with bubble sort), add unnecessary copying/cloning of data structures, replace in-place operations with operations that create new objects, remove caching of computed results, add unnecessary serialization/deserialization round-trips, replace batch operations with individual operations, add unnecessary I/O (logging to files, redundant reads), use less efficient data structures (linked list instead of array, linear search instead of hash lookup)

### 3c. Run the Fitness Function Baseline

Before starting the improvement loop:

1. Run the fitness function script to get the baseline score
2. Record the baseline in `JOB-SECURITY-STATE.md`
3. Record the baseline in `GOAL.md`'s Bootstrap section
4. Commit the GOAL.md: `git add GOAL.md && git commit -m "goal: [Phase N] [metric name] — baseline score: [X]"`

### 3d. Execute the Improvement Loop

Run the autoresearch improvement loop as defined in GOAL.md:

1. **Measure** — run the fitness function, record score
2. **Diagnose** — identify the highest-impact opportunity from the action catalog
3. **Act** — make ONE focused change
4. **Verify** — run the fitness function again, run tests, run build
5. **Commit or revert:**
   - If score moved in the desired direction AND build passes AND tests pass → `git add -A && git commit -m "[S:OLD->NEW] component: what changed"`
   - If any check fails → `git checkout -- . && git clean -fd` (revert everything)
6. **Log** — append to `iterations.jsonl`
7. **Repeat** until stopping conditions are met

### 3e. Record Phase Results

When a phase's stopping conditions are met:

1. Run the fitness function one final time
2. Update `JOB-SECURITY-STATE.md`:
   - Set the phase status to `completed`
   - Record the final score and delta from baseline
   - Add a Phase Log entry with summary of what was done
3. Archive the GOAL.md: `mv GOAL.md .job-security/GOAL-phase-N.md` (create `.job-security/` if needed)
4. Archive iterations.jsonl: `mv iterations.jsonl .job-security/iterations-phase-N.jsonl`
5. Commit: `git add -A && git commit -m "job-security: Phase N complete — [metric] [baseline] -> [final] ([delta]%)"`
6. Proceed to Step 3a for the next phase

## Step 4: Completion Report

After all phases are complete (or all remaining phases are skipped):

### Standalone Mode

Update `JOB-SECURITY-STATE.md` with a final summary section:

```markdown
## Final Report

**Completed:** [date]
**Phases run:** [N] of [total]
**Phases skipped:** [list with reasons]

### Results Summary

| Phase | Metric | Baseline | Final | Change |
|-------|--------|----------|-------|--------|
| ... | ... | ... | ... | ...% |

### Total Commits

[N] de-optimization commits across all phases.
```

Commit the final state: `git add -A && git commit -m "job-security: All phases complete — see JOB-SECURITY-STATE.md for results"`

### Ralph Mode

The final ralph instance (executing the last story) must:

1. Update `JOB-SECURITY-STATE.md` with the same Final Report structure as above
2. Append a final summary to `progress.txt`
3. Set the last story's `passes` to `true` in `prd.json`
4. Commit all changes
5. Output `<promise>COMPLETE</promise>` to signal ralph.sh that all work is done

## Resumption

If the skill is interrupted and re-invoked:

### Standalone Mode

1. Check for an existing `JOB-SECURITY-STATE.md` in the repository root
2. If found, read it and identify the current state:
   - If a phase is marked `in-progress`, check for an existing `GOAL.md` and `iterations.jsonl` — resume the improvement loop from where it left off
   - If the last phase is `completed`, advance to the next `pending` phase
3. If not found, start from Step 1 (Pipeline Configuration)
4. Read the `## Configuration` section to restore the user's execution mode

### Ralph Mode

Each fresh ralph instance has no memory of previous instances. Continuity comes from files:

1. Read `prd.json` — find the highest-priority story where `passes: false`
2. Read `JOB-SECURITY-STATE.md` — understand project state, completed phases, baselines
3. Read `progress.txt` — check Codebase Patterns section first for accumulated learnings
4. Read `.job-security/CLAUDE.md` — follow the protocol and constraints
5. If the current story's phase was partially completed (GOAL.md exists, iterations.jsonl has entries), continue the loop from the current state rather than restarting
6. Execute the story, update all state files, commit, and check for overall completion

This makes both modes resilient to interruptions — progress is never lost.

## Error Handling

### Common to Both Modes

- **Build failure before starting:** Report the error and stop. Do not attempt to fix pre-existing build failures.
- **Test failure before starting:** Report the error and stop. Do not attempt to fix pre-existing test failures.
- **Fitness function script error:** Debug the script, fix it, and retry. If unfixable after 3 attempts, skip the phase with reason "fitness function failed".
- **All iterations stalled:** If 5 consecutive iterations produce no change in the desired direction, stop the phase and move on.
- **Complexity tool not available:** Try to install one automatically. If installation fails (e.g., no package manager, incompatible platform), skip the phase.

### Ralph Mode Specific

- **prd.json corrupted or missing:** If `prd.json` cannot be parsed, check git history for the last valid version and restore it. If unrecoverable, report the error and output nothing (ralph.sh will retry on next iteration).
- **State file conflict:** If `JOB-SECURITY-STATE.md` has merge conflicts or inconsistencies, resolve based on `prd.json` as the source of truth for which phases are complete.
- **Partial phase from previous instance:** If GOAL.md and iterations.jsonl exist but the phase isn't marked complete, the previous instance was interrupted mid-phase. Continue from the last recorded iteration rather than restarting the phase.
