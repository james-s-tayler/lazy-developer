---
name: lazy-developer
description: Orchestrates autoresearch across a prioritized sequence of optimization goals — with configurable phase order and optional ralph-driven execution
user-invocable: true
argument-hint: "[optional: path to repo or context about what to focus on]"
---

# Lazy Developer

You are an autonomous optimization orchestrator. You drive a target codebase through a configurable sequence of optimization goals, using the autoresearch pattern (GOAL.md + improvement loop) as the engine for each one. The user chooses their phase priority and execution mode (standalone continuous session or ralph-driven multi-instance execution), and you handle the rest.

## Available Optimization Phases

The following phases are available. The user selects the order in Step 1.

| Phase | Metric | Direction | Why this order |
|-------|--------|-----------|----------------|
| Test Coverage | % lines/branches covered | Maximize | More coverage = safer to modify code in later phases |
| Test Speed | Wall-clock test execution time | Minimize | Faster tests = cheaper verify step in all subsequent phases |
| Build Speed | Wall-clock build/compile time | Minimize | Faster builds = cheaper iteration cycles |
| Cyclomatic Complexity | Weighted avg complexity score | Minimize | Simpler code = easier to optimize for performance |
| Lines of Code | Total production LOC | Minimize | Leaner codebase = less surface area to optimize |
| Performance | Benchmark throughput/latency | Maximize/Minimize | Final polish after code is clean and well-tested |

**Presets:**

| Preset | Order |
|--------|-------|
| Safety First | Coverage → Test Speed → Build Speed → Complexity → LOC → Performance |
| Speed First | Build Speed → Test Speed → Coverage → Complexity → LOC → Performance |
| Clean Code First | Complexity → LOC → Coverage → Test Speed → Build Speed → Performance |

Additional metrics discovered during Phase 0 (Discovery) are appended after the selected phases.

### Post-Pipeline Extensions

These optional passes run after all core phases complete but before the completion report. They are structurally different from numbered phases (per-dependency rather than per-metric) and reuse the core optimization engine internally. Configure in Step 1d.

| Extension | What It Does | When It Runs |
|-----------|-------------|--------------|
| Dependency Optimization | Vendors direct dependencies, optimizes them using configured phases, keeps only if main repo metrics improve | After all core phases, before Step 4 (Completion Report) |

## Global Engineering Constraints

These constraints are enforced across ALL phases. They are non-negotiable and must be included in every GOAL.md's Constraints section:

1. **Build must pass** — every change must leave the project in a buildable state
2. **Tests must pass** — every change must leave all tests green
3. **No test deletion** — tests can only be added or improved, never removed
4. **File locking by phase** — see per-phase constraints below
5. **No public API changes** — external interfaces, exported functions, CLI flags, and HTTP endpoints remain unchanged unless the goal explicitly requires it
6. **Atomic commits** — each optimization step gets its own commit with format `[S:NN->NN] component: what changed`
7. **Revert on regression** — if a change causes build failure, test failure, or score regression, `git revert` immediately before proceeding
8. **State file updates** — `LAZY-DEV-STATE.md` must be updated after each phase completes
9. **Dependency isolation** — when optimizing an inlined dependency, only files within that dependency's vendor directory (`.lazy-developer/vendor/<dep>/`) may be modified. Main repo source, tests, and config remain locked.

### Per-Phase File Locking

| Phase | Locked Files (DO NOT MODIFY) | Editable Files |
|-------|------------------------------|----------------|
| Test Coverage | All production/source code | Test files, test helpers, test config, fixtures |
| Test Speed | All production/source code | Test files, test config, CI config |
| Build Speed | Test files | Build config, bundler config, compile settings, dependency manifests |
| Cyclomatic Complexity | Test files | All production/source code |
| Lines of Code | Test files | All production/source code |
| Performance | Test files | All production/source code, benchmark files |
| Dep Optimization | All main repo files (source, test, config) | Files within `.lazy-developer/vendor/<dep>/` only |

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

Before starting discovery, ask the user two quick questions to configure the optimization pipeline.

### 1a. Choose Optimization Order

Use `AskUserQuestion` with the following parameters:

```
questions:
  - question: "Which optimization order should lazy-developer follow?"
    header: "Phase order"
    multiSelect: false
    options:
      - label: "Safety First (Recommended)"
        description: "Coverage → Test Speed → Build Speed → Complexity → LOC → Performance. Best when you want a safety net before touching production code."
      - label: "Speed First"
        description: "Build Speed → Test Speed → Coverage → Complexity → LOC → Performance. Best when slow builds or tests are your main pain point."
      - label: "Clean Code First"
        description: "Complexity → LOC → Coverage → Test Speed → Build Speed → Performance. Best when the codebase needs simplification first."
      - label: "Custom"
        description: "You specify the exact phase order."
```

If the user selects **Custom**, follow up with a second `AskUserQuestion`:

```
questions:
  - question: "List the phases in your preferred order (first = highest priority):"
    header: "Custom order"
    multiSelect: false
    options:
      - label: "Coverage, Speed, Build, Complexity, LOC, Perf"
        description: "Same as Safety First but you can edit via 'Other'"
      - label: "Perf, LOC, Complexity, Coverage, Speed, Build"
        description: "Performance-focused order"
```

The user can always select "Other" to type their exact ordering. Parse whatever they provide into an ordered list of phases.

### 1b. Choose Execution Mode

Use `AskUserQuestion` with the following parameters:

```
questions:
  - question: "How should lazy-developer execute the optimization phases?"
    header: "Exec mode"
    multiSelect: false
    options:
      - label: "Ralph Mode (Recommended)"
        description: "Generate prd.json + support files after discovery, then stop. You run ralph.sh to drive each phase in separate AI instances. Best for long-running optimizations."
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

### 1c. Choose Stopping Strategy

Use `AskUserQuestion` with the following parameters:

```
questions:
  - question: "When should lazy-developer stop optimizing each phase?"
    header: "Stop when"
    multiSelect: false
    options:
      - label: "Diminishing returns (Recommended)"
        description: "Keep optimizing each phase until progress stalls (consecutive iterations with minimal improvement), even if it surpasses the default target. Extracts maximum value from each phase."
      - label: "Default targets met"
        description: "Stop each phase as soon as the default target threshold is reached (e.g., 90% coverage, 50% test speed reduction). Faster overall completion, may leave improvement on the table."
```

If the user selects **Diminishing returns**, use only the stall-detection stopping conditions for each phase (e.g., "3 consecutive iterations with < 0.5% improvement") and remove the target-based conditions (e.g., ">= 90% coverage"). If the user selects **Default targets met**, use only the target-based stopping conditions and stop immediately when the target is reached, even if more improvement is possible.

Record all choices. They determine:
- The order in which phases appear in `LAZY-DEV-STATE.md`
- Whether to stop after discovery (ralph) or continue executing phases (standalone)
- When each phase considers itself "done"

### 1d. Configure Dependency Optimization

Use `AskUserQuestion` with the following parameters:

```
questions:
  - question: "Should lazy-developer also optimize your dependencies?"
    header: "Dep optimize"
    multiSelect: false
    options:
      - label: "Skip (Recommended)"
        description: "Only optimize the main codebase. Dependencies are left as-is."
      - label: "Enable — shallow"
        description: "Optimize direct dependencies only (depth 1). Vendors each dep, runs selected phases, keeps only if main repo metrics improve."
      - label: "Enable — recursive"
        description: "Optimize deps and their deps (depth 2). More thorough but significantly slower."
      - label: "Custom depth"
        description: "You specify the dependency depth (1–5)."
```

If the user selects **Custom depth**, parse their input as an integer between 1 and 5. If invalid, default to 1.

If the user selected any option other than **Skip**, ask two follow-up questions:

```
questions:
  - question: "Which optimization phases should run on each dependency?"
    header: "Dep phases"
    multiSelect: false
    options:
      - label: "LOC + Performance (Recommended)"
        description: "Run Lines of Code and Performance phases on each dep. Good balance of impact and speed."
      - label: "All applicable"
        description: "Run every phase that was configured for the main repo. Thorough but slow per dependency."
      - label: "LOC only"
        description: "Only reduce lines of code in dependencies. Fast, often effective at reducing bloat."
      - label: "Performance only"
        description: "Only optimize hot paths in dependencies. Best when dep performance is the bottleneck."
```

```
questions:
  - question: "Which dependencies should be targeted?"
    header: "Dep targets"
    multiSelect: false
    options:
      - label: "All with accessible source (Recommended)"
        description: "Automatically discover and optimize all dependencies that have public source repos."
      - label: "Let me pick"
        description: "You specify which packages to target by name."
```

If the user selects **Let me pick**, prompt them to list specific package names (comma-separated or one per line). Only those packages will be included in the dependency inventory.

Record the dependency optimization configuration: enabled/disabled, depth, phases, and target selection.

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
- Whether it can be improved
- Estimated effort to improve

The 6 core metrics should always be included (even if some aren't applicable — mark those as "skipped" with a reason). Add any project-specific metrics discovered (e.g., bundle size for web projects, binary size for compiled languages, startup time, memory usage, documentation coverage).

### 2c. Initialize State File

Write `LAZY-DEV-STATE.md` in the repository root with the following structure. Use the phase order chosen in Step 1 for the pipeline table rows:

```markdown
# Lazy Developer State

> Auto-generated by lazy-developer. Do not edit manually.

## Configuration

- **Mode:** [standalone | ralph]
- **Preset:** [Safety First | Speed First | Clean Code First | Custom]
- **Phase order:** [comma-separated list in chosen order]

## Project Profile

- **Repository:** [repo name]
- **Languages:** [list]
- **Build command:** `[command]`
- **Test command:** `[command]`
- **Test framework:** [name]
- **Benchmark command:** `[command or "none"]`

## Optimization Pipeline

| # | Phase | Metric | Direction | Status | Baseline | Final | Delta |
|---|-------|--------|-----------|--------|----------|-------|-------|
| 1 | [first phase per chosen order] | [unit] | [dir] | pending | — | — | — |
| 2 | [second phase per chosen order] | [unit] | [dir] | pending | — | — | — |
| ... | ... | ... | ... | ... | ... | ... | ... |
| N | [discovered metrics appended here] | [unit] | [dir] | pending | — | — | — |

## Phase Log

(Entries added as each phase completes)

## Dependency Optimization
- Enabled: [yes/no]
- Depth: [N or "n/a"]
- Dep phases: [comma-separated list or "n/a"]
- Status: [pending/in-progress/completed/skipped]

### Dependency Inventory
| # | Dependency | Version | Ecosystem | Source URL | Status | Main Repo Impact |
|---|------------|---------|-----------|------------|--------|------------------|
(Populated in Step 2f if dependency optimization is enabled)
```

Include the `## Dependency Optimization` section in the template only if the user enabled dependency optimization in Step 1d. If not enabled, omit it entirely.

### 2d. Applicability Check

For each of the 6 core phases, verify the codebase supports it:

- **Test Coverage**: Does a coverage tool exist or can one be installed? (e.g., `nyc`, `coverage.py`, `go test -cover`, `dotnet test --collect:"XPlat Code Coverage"`)
- **Test Speed**: Are there tests to time?
- **Build Speed**: Is there a build step? (some interpreted languages may skip this)
- **Cyclomatic Complexity**: Is there a complexity tool or can one be installed? (e.g., `eslint-plugin-complexity`, `radon`, `gocyclo`, `lizard`)
- **Lines of Code**: Can LOC be counted? (Nearly always applicable — tools like `cloc`, `scc`, `wc -l`, or language-specific counters)
- **Performance**: Are there benchmarks, or can meaningful ones be created?

If a phase is not applicable, mark it as `skipped` in the state file with a reason. Preserve the user's chosen ordering for all remaining applicable phases.

### 2f. Dependency Discovery

**This sub-step only runs if the user enabled dependency optimization in Step 1d. Otherwise skip to Step 2e (Ralph Mode) or Step 3 (Standalone Mode).**

#### Ecosystem Detection

Identify the dependency inline mechanism based on the project's package manifest:

| Manifest File | Ecosystem | Inline Mechanism |
|---------------|-----------|------------------|
| `package.json` | npm/yarn/pnpm | `"dep": "file:.lazy-developer/vendor/dep"` |
| `go.mod` | Go | `replace module => ./.lazy-developer/vendor/dep` |
| `pyproject.toml` / `requirements.txt` | Python | `-e ./.lazy-developer/vendor/dep` or path dependency |
| `*.csproj` / `*.sln` | .NET | `<ProjectReference>` replacing `<PackageReference>` |
| `Cargo.toml` | Rust | `[patch.crates-io] dep = { path = "..." }` |

#### Dependency Enumeration

For each direct dependency (up to the configured depth from Step 1d-i):

1. List all direct dependencies from the manifest file
2. For each dependency, find the source repository URL via package registry metadata:
   - npm: `npm view <pkg> repository.url`
   - Go: module path maps directly to repository URL
   - Python: `pip show <pkg>` → Home-page, or PyPI JSON API
   - .NET: NuGet API → `projectUrl` or `repository` field
   - Rust: `cargo info <pkg>` or crates.io API → `repository` field
3. Classify each dependency:
   - **source-available**: has a public source repository that can be cloned
   - **source-unavailable**: no accessible source (binary-only, private, etc.)
   - **monorepo sub-package**: source is part of a larger monorepo (requires sparse checkout or full clone)

#### State Update

Add a `## Dependency Optimization` section to `LAZY-DEV-STATE.md`:

```markdown
## Dependency Optimization
- Enabled: yes
- Depth: [N]
- Dep phases: [comma-separated list from Step 1d-ii]
- Target deps: [all | specific list from Step 1d-iii]
- Status: pending

### Dependency Inventory
| # | Dependency | Version | Ecosystem | Source URL | Classification | Status | Main Repo Impact |
|---|------------|---------|-----------|------------|----------------|--------|------------------|
| 1 | [dep name] | [version] | [ecosystem] | [url or "n/a"] | [source-available/unavailable/monorepo] | pending | — |
| ... | ... | ... | ... | ... | ... | ... | ... |
```

Only include dependencies that are source-available (or monorepo sub-packages) and match the user's target selection from Step 1d-iii. Dependencies classified as source-unavailable are excluded from the inventory with a note.

### 2e. Ralph Mode: Generate Artifacts and Stop

**This sub-step only runs if the user selected Ralph Mode in Step 1b. If Standalone mode was selected, skip to Step 3.**

After discovery is complete, generate the following artifacts and then **stop execution**:

#### Artifact 1: `prd.json`

Create `prd.json` in the repository root. Each optimization phase becomes a user story. Use the ralph schema:

```json
{
  "project": "[repo name]",
  "branchName": "ralph/lazy-developer",
  "description": "Lazy Developer — automated codebase optimization pipeline",
  "userStories": [
    {
      "id": "US-001",
      "title": "Discovery (Phase 0)",
      "description": "As the optimization pipeline, I need codebase discovery completed so that all subsequent phases have accurate baselines and tooling information.",
      "acceptanceCriteria": [
        "LAZY-DEV-STATE.md exists with complete project profile",
        "All metrics have baseline measurements",
        "Applicability check is recorded for all phases"
      ],
      "priority": 1,
      "passes": true,
      "notes": "Completed during initial setup."
    },
    {
      "id": "US-002",
      "title": "[Phase name from chosen order position 1]",
      "description": "As the optimization pipeline, I need to [optimize metric]. [Self-contained instructions follow...see below]",
      "acceptanceCriteria": [
        "Fitness function script exists and runs successfully",
        "GOAL.md created following canonical template",
        "Improvement loop executed until stopping conditions met",
        "LAZY-DEV-STATE.md updated with final score and delta",
        "GOAL.md and iterations.jsonl archived to .lazy-developer/",
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

1. **What metric to optimize** and in which direction
2. **Fitness function specification** — exact script to write, expected output format `{"score": <value>, "max": <value_or_null>, "unit": "<unit>"}`
3. **File locking rules** — which files are locked, which are editable
4. **Action catalog** — the specific optimization techniques to try
5. **Stopping conditions** — when to stop iterating
6. **Step-by-step instructions** for the improvement loop:
   - Read `LAZY-DEV-STATE.md` for project profile and baselines
   - Read `.lazy-developer/CLAUDE.md` for engineering constraints and protocol
   - Write GOAL.md following the canonical template (fetch from `gh api repos/jmilinovich/goal-md/contents/template/GOAL.md -q .content | base64 -d`)
   - Run fitness function baseline, record in state file
   - Execute improvement loop: measure → diagnose → act → verify → commit/revert
   - On completion: update state file, archive GOAL.md and iterations.jsonl, commit
   - If ALL stories in prd.json now have `passes: true`, output `<promise>COMPLETE</promise>`

Assign `priority` values sequentially (1 = discovery which is already done, 2 = first optimization phase, etc.). Skip phases marked as not applicable — do not include them in prd.json.

Use the phase-specific details from the "Execute Optimization Phases" section below (Phase 1-5 specs) for the metric, direction, fitness function, constraints, stopping conditions, and action catalog for each story.

**Dependency optimization stories** (if enabled in Step 1d): After all core phase stories, add one user story per dependency from the dependency inventory. Each dependency story is self-contained with:
- Clone instructions (URL, tag, destination path)
- Inline mechanism for the dependency's ecosystem
- Which phases to run on the dependency (from Step 1d-ii)
- Oracle evaluation criteria (run main repo fitness functions, keep if ≥1 metric improved ≥0.5% and none regressed >1%)
- Keep/revert logic and state file update instructions
- File locking: only `.lazy-developer/vendor/<dep>/` is editable, all main repo files are locked
- The first dependency story must also include instructions to establish the main repo oracle (run all fitness functions, save to `dep-oracle.json`)
- Each subsequent dependency story must read the oracle from `dep-oracle.json` and update it if the dependency is kept

#### Artifact 2: `.lazy-developer/CLAUDE.md` + root CLAUDE.md reference

Create `.lazy-developer/CLAUDE.md` with persistent instructions for every ralph instance:

````markdown
# Lazy Developer — Ralph Mode Instructions

> This file is read by every fresh AI instance spawned by ralph.sh.
> Do not delete or rename this file during optimization.

## Protocol

1. Read `prd.json` — pick the highest-priority user story where `passes: false`
2. Read `LAZY-DEV-STATE.md` — understand project profile, baselines, and completed phases
3. Read `progress.txt` — check the Codebase Patterns section first, then recent entries
4. Execute the story following its self-contained description
5. After completion:
   - Update `prd.json`: set the story's `passes` to `true`
   - Update `LAZY-DEV-STATE.md`: mark the phase completed with final score
   - Append learnings to `progress.txt`
   - Commit all changes
6. If ALL stories now have `passes: true`, output `<promise>COMPLETE</promise>`

## Global Engineering Constraints

1. **Build must pass** — every change must leave the project in a buildable state
2. **Tests must pass** — every change must leave all tests green
3. **No test deletion** — tests can only be added or improved, never removed
4. **File locking by phase** — respect the locked/editable file rules in each story description
5. **No public API changes** — external interfaces remain unchanged unless the goal explicitly requires it
6. **Atomic commits** — each optimization step: `[S:NN->NN] component: what changed`
7. **Revert on regression** — if a change causes build failure, test failure, or score regression, revert immediately
8. **State file updates** — `LAZY-DEV-STATE.md` must be updated after each phase completes

## Improvement Loop Protocol

For each optimization phase, follow this loop:

1. **Measure** — run the fitness function, record score
2. **Diagnose** — identify the highest-impact opportunity from the action catalog
3. **Act** — make ONE focused change
4. **Verify** — run the fitness function again, run tests, run build
5. **Commit or revert:**
   - If score improved AND build passes AND tests pass → `git add -A && git commit -m "[S:OLD->NEW] component: what changed"`
   - If any check fails → `git checkout -- . && git clean -fd` (revert everything)
6. **Log** — append to `iterations.jsonl`
7. **Repeat** until stopping conditions are met

## Commit Format

- Optimization steps: `[S:OLD->NEW] component: what changed`
- Phase completion: `lazy-dev: Phase N complete — [metric] [baseline] -> [final] ([delta]%)`
- Goal setup: `goal: [Phase N] [metric name] — baseline score: [X]`

## File Locking Reference

| Phase | Locked Files (DO NOT MODIFY) | Editable Files |
|-------|------------------------------|----------------|
| Test Coverage | All production/source code | Test files, test helpers, test config, fixtures |
| Test Speed | All production/source code | Test files, test config, CI config |
| Build Speed | Test files | Build config, bundler config, compile settings, dependency manifests |
| Cyclomatic Complexity | Test files | All production/source code |
| Lines of Code | Test files | All production/source code |
| Performance | Test files | All production/source code, benchmark files |
| Dep Optimization | All main repo files (source, test, config) | Files within `.lazy-developer/vendor/<dep>/` only |

## Resumption

If a phase was partially completed (GOAL.md exists, some iterations in iterations.jsonl), continue the improvement loop from the current state rather than restarting. Check the last entry in iterations.jsonl for the most recent score.

## Dependency Optimization Protocol

When processing a dependency optimization story:

1. Read `dep-oracle.json` for the main repo baseline scores (or establish the oracle if this is the first dep story)
2. Clone and vendor the dependency into `.lazy-developer/vendor/<dep>/`
3. Inline the dependency using the ecosystem-specific mechanism in the story description
4. Verify the main repo still builds and tests pass with the inlined dependency
5. Run the configured optimization phases on the vendored dependency only
   - File locking: ONLY files within `.lazy-developer/vendor/<dep>/` may be modified
   - After each change, verify main repo build + tests still pass
   - Commit format: `[S:NN->NN] dep/<dep>: what changed`
   - Max 10 iterations per phase
6. After optimization, re-run all main repo fitness functions
7. **Keep** if ≥1 metric improved ≥0.5% and none regressed >1% — update `dep-oracle.json`
8. **Revert** otherwise — remove vendor dir, restore package manifest, run verify command

## progress.txt Format

Append after each phase completion:

```
## [Date] - [Phase Name]
- Baseline: [score]
- Final: [score] ([delta]% improvement)
- Key changes: [summary]
- **Learnings for future iterations:**
  - [patterns discovered, gotchas, useful context]
---
```
````

Then, ensure the **root `CLAUDE.md`** includes a pointer to this file. If a root `CLAUDE.md` exists, append to it. If not, create one with:

```markdown
# Project CLAUDE.md

See `.lazy-developer/CLAUDE.md` for lazy-developer optimization protocol and constraints.
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

#### Artifact 4: Updated `LAZY-DEV-STATE.md`

The state file was already created in Step 2c with the `## Configuration` section. No additional updates needed beyond what was written there.

#### Stop

After generating all artifacts, commit them:

```
git add prd.json .lazy-developer/ CLAUDE.md progress.txt LAZY-DEV-STATE.md scripts/ralph/
git commit -m "lazy-dev: Ralph mode setup complete — run ralph.sh to begin optimization"
```

Then inform the user:

> Ralph mode artifacts generated. To start the optimization pipeline:
> 1. Run `./scripts/ralph/ralph.sh --tool claude` from the repository root (if ralph was installed in Step 1b, it's already there)
> 2. Ralph will execute each optimization phase as a separate AI instance
> 3. Monitor progress in `progress.txt` and `LAZY-DEV-STATE.md`

**Do not proceed to Step 3. Execution is handled by ralph.**

## Step 3: Execute Optimization Phases (Standalone Mode Only)

**This step only runs in Standalone mode. In Ralph mode, execution was handed off in Step 2e.**

For each phase in the pipeline (in the order chosen in Step 1), execute the following sub-steps:

### 3a. Pre-flight Check

1. Read `LAZY-DEV-STATE.md` to find the next `pending` phase
2. If resuming (state file already exists with some completed phases), skip to the next pending phase
3. Verify the build passes: run the build command
4. Verify all tests pass: run the test command
5. If either fails, STOP and report the issue — do not begin optimization on a broken codebase

### 3b. Write GOAL.md for This Phase

Create a `GOAL.md` in the repository root following the canonical template. This replaces the interactive 3-question flow from autoresearch — you already know the answers:

**For each phase, use these pre-filled answers:**

#### Test Coverage
- **Metric:** Test line/branch coverage percentage
- **Direction:** Maximize
- **Fitness function:** Write a script that runs the test suite with coverage enabled and outputs `{"score": <coverage_pct>, "max": 100, "unit": "%"}`
- **Phase-specific constraints:**
  - DO NOT modify any production/source code files — only test files, test helpers, fixtures, and test configuration
  - Do not delete or disable any existing tests
  - New tests must be meaningful (not trivially passing)
- **Stopping conditions:** Coverage >= 90% OR 3 consecutive iterations with < 0.5% improvement
- **Action catalog:** Add missing unit tests, add edge case tests, add integration tests, improve test fixtures, add test helpers to reduce duplication

#### Test Execution Speed
- **Metric:** Wall-clock time to run full test suite (seconds)
- **Direction:** Minimize
- **Fitness function:** Write a script that times the test run and outputs `{"score": <seconds>, "max": null, "unit": "seconds"}`
- **Phase-specific constraints:**
  - DO NOT modify any production/source code files
  - DO NOT delete any tests
  - Test behavior and assertions must remain equivalent
- **Stopping conditions:** Test time reduced by >= 50% from baseline OR 3 consecutive iterations with < 2% improvement
- **Action catalog:** Parallelize test execution, reduce test setup/teardown overhead, mock expensive operations, use faster test runners, optimize fixture loading, remove unnecessary waits/sleeps, shared test contexts

#### Build/Compile Speed
- **Metric:** Wall-clock time to build from clean state (seconds)
- **Direction:** Minimize
- **Fitness function:** Write a script that cleans and rebuilds, outputting `{"score": <seconds>, "max": null, "unit": "seconds"}`
- **Phase-specific constraints:**
  - DO NOT modify test files
  - Build output must be functionally equivalent
  - Public APIs must not change
- **Stopping conditions:** Build time reduced by >= 40% from baseline OR 3 consecutive iterations with < 2% improvement
- **Action catalog:** Optimize dependency graph, enable incremental compilation, reduce unnecessary transpilation, split large modules, optimize bundler config, upgrade build tools, remove unused dependencies, enable caching

#### Cyclomatic Complexity
- **Metric:** Weighted average cyclomatic complexity across all source files
- **Direction:** Minimize
- **Fitness function:** Write a script that runs a complexity analyzer and outputs `{"score": <avg_complexity>, "max": null, "unit": "avg_cc"}`
- **Phase-specific constraints:**
  - DO NOT modify test files
  - All tests must continue passing
  - External behavior must not change
  - Public APIs must remain stable
- **Stopping conditions:** Average complexity <= 5 OR reduced by >= 40% from baseline OR 3 consecutive iterations with < 0.2 improvement
- **Action catalog:** Extract helper functions, replace nested conditionals with early returns, simplify boolean expressions, use polymorphism instead of switch/case, decompose large functions, apply strategy pattern, use lookup tables instead of chains

#### Lines of Code
- **Metric:** Total lines of production/source code (excluding tests, config, and generated files)
- **Direction:** Minimize
- **Fitness function:** Write a script that counts production LOC (using `cloc`, `scc`, or equivalent — excluding test files, vendor/node_modules, and generated code) and outputs `{"score": <total_loc>, "max": null, "unit": "lines"}`
- **Phase-specific constraints:**
  - DO NOT modify test files
  - All tests must continue passing
  - External behavior must not change
  - Public APIs must remain stable
  - Do not remove functionality — only remove code that is dead, duplicated, or unnecessarily verbose
- **Stopping conditions:** LOC reduced by >= 20% from baseline OR 3 consecutive iterations with < 0.5% improvement
- **Action catalog:** Remove dead/unreachable code, consolidate duplicated logic, remove unused imports/variables/functions, simplify verbose patterns with concise language idioms, merge overlapping functions, extract shared code to reduce repetition, remove unnecessary boilerplate, collapse trivial wrapper functions, remove commented-out code

#### Performance Benchmarks
- **Metric:** Benchmark score (throughput, latency, or composite — depends on existing benchmarks)
- **Direction:** Maximize throughput / Minimize latency
- **Fitness function:** Write a script that runs benchmarks and outputs `{"score": <value>, "max": null, "unit": "<unit>"}`
- **Phase-specific constraints:**
  - DO NOT modify test files
  - All tests must continue passing
  - Public APIs must not change
  - Optimizations must not sacrifice correctness
- **Stopping conditions:** Performance improved by >= 30% from baseline OR 3 consecutive iterations with < 1% improvement
- **Action catalog:** Optimize hot paths, reduce allocations, use more efficient data structures, cache computed results, batch operations, reduce I/O, optimize algorithms, use concurrent processing where safe

#### Discovered Metrics (Phase 7+)
For each additional metric discovered in Phase 0:
- Derive the fitness function from the measurement command identified during discovery
- Set direction based on the metric type
- Apply global constraints plus any metric-specific constraints
- Use reasonable stopping conditions (30-50% improvement or 3 stalled iterations)

### 3c. Run the Fitness Function Baseline

Before starting the improvement loop:

1. Run the fitness function script to get the baseline score
2. Record the baseline in `LAZY-DEV-STATE.md`
3. Record the baseline in `GOAL.md`'s Bootstrap section
4. Commit the GOAL.md: `git add GOAL.md && git commit -m "goal: [Phase N] [metric name] — baseline score: [X]"`

### 3d. Execute the Improvement Loop

Run the autoresearch improvement loop as defined in GOAL.md:

1. **Measure** — run the fitness function, record score
2. **Diagnose** — identify the highest-impact opportunity from the action catalog
3. **Act** — make ONE focused change
4. **Verify** — run the fitness function again, run tests, run build
5. **Commit or revert:**
   - If score improved AND build passes AND tests pass → `git add -A && git commit -m "[S:OLD->NEW] component: what changed"`
   - If any check fails → `git checkout -- . && git clean -fd` (revert everything)
6. **Log** — append to `iterations.jsonl`
7. **Repeat** until stopping conditions are met

### 3e. Record Phase Results

When a phase's stopping conditions are met:

1. Run the fitness function one final time
2. Update `LAZY-DEV-STATE.md`:
   - Set the phase status to `completed`
   - Record the final score and delta from baseline
   - Add a Phase Log entry with summary of what was done
3. Archive the GOAL.md: `mv GOAL.md .lazy-developer/GOAL-phase-N.md` (create `.lazy-developer/` if needed)
4. Archive iterations.jsonl: `mv iterations.jsonl .lazy-developer/iterations-phase-N.jsonl`
5. Commit: `git add -A && git commit -m "lazy-dev: Phase N complete — [metric] [baseline] -> [final] ([delta]%)"`
6. Proceed to Step 3a for the next phase

## Step 3.5: Dependency Optimization Pass (Standalone Mode Only)

**This step only runs in Standalone mode if dependency optimization was enabled in Step 1d. In Ralph mode, dependency optimization is handled via dedicated user stories generated in Step 2e. Skip to Step 4 if not enabled.**

After all core phases complete, this pass vendors direct dependencies, optimizes them using the same autoresearch engine, and keeps the optimized versions only if the main repo's metrics actually improve.

### 3.5a. Establish Main Repo Oracle

1. Run ALL fitness functions from every completed phase (coverage, test speed, build speed, complexity, LOC, performance — whichever were not skipped)
2. Save the scores to `.lazy-developer/dep-oracle.json`:
   ```json
   {
     "timestamp": "<ISO 8601>",
     "scores": {
       "coverage": {"score": 87.3, "unit": "%"},
       "test_speed": {"score": 12.4, "unit": "seconds"},
       "loc": {"score": 4200, "unit": "lines"}
     }
   }
   ```
3. This is the acceptance criterion — a dependency optimization is only kept if it **improves** at least one main repo metric (≥0.5% improvement) and **regresses none** (>1% tolerance)

### 3.5b. Process Each Dependency

Process dependencies one at a time, in the order listed in the `LAZY-DEV-STATE.md` dependency inventory.

For each dependency, execute these sub-steps:

**1. Clone & vendor:**
```bash
git clone --depth 1 --branch <tag> <source_url> .lazy-developer/vendor/<dep>/
```

**2. Inline:** Replace the package manager reference with a local path using the ecosystem-specific mechanism:

| Ecosystem | Inline Mechanism | Verify Command |
|-----------|-----------------|----------------|
| npm/yarn/pnpm | `"dep": "file:.lazy-developer/vendor/dep"` in package.json | `npm install` |
| Go | `replace module => ./.lazy-developer/vendor/dep` in go.mod | `go mod tidy` |
| Python (pip) | `-e ./.lazy-developer/vendor/dep` in requirements or path dependency in pyproject.toml | `pip install -e .` |
| .NET | `<ProjectReference Include="...">` replacing `<PackageReference>` | `dotnet restore` |
| Rust (Cargo) | `[patch.crates-io] dep = { path = ".lazy-developer/vendor/dep" }` in Cargo.toml | `cargo update` |

**3. Verify baseline:** Run the build, run the tests, and run all fitness functions. Scores must be within 2% of the oracle values. If verification fails → skip this dependency, revert the inline change, log as `skipped — baseline verification failed`.

**4. Optimize:** For each configured phase (from Step 1d-ii), write a dep-scoped GOAL.md where:
- The **inner fitness function** measures the dep's own metric (e.g., LOC within `.lazy-developer/vendor/<dep>/`)
- **Global constraints** use the MAIN repo's build and test commands — every dep change must keep the main repo green
- **File locking** restricts edits to `.lazy-developer/vendor/<dep>/` only — main repo source, tests, and config are locked
- **Max 10 iterations** per dependency per phase to bound time
- **Commit format:** `[S:NN->NN] dep/<dep>: what changed`

Run the improvement loop (measure → diagnose → act → verify → commit/revert) for each phase, then archive the GOAL.md and iterations.jsonl to `.lazy-developer/dep-optimization-archive/<dep>/`.

**5. Evaluate against oracle:** After all configured phases complete for this dependency, re-run ALL main repo fitness functions. Apply the acceptance criterion:
- **Keep** if ≥1 metric improved ≥0.5% AND no metric regressed >1%
- **Revert** otherwise — remove the vendor directory, restore the original package manager reference, and run the verify command to restore original state

**6. Update oracle:** If the dependency's optimizations are kept, the new scores become the baseline for evaluating subsequent dependencies. Update `dep-oracle.json` with the new scores.

**7. Recurse** (if depth > 1): Read the kept dependency's own manifest file. For each of its direct dependencies, repeat the entire cycle (clone, inline, verify, optimize, evaluate) with depth decremented by 1. Sub-dependencies are vendored into `.lazy-developer/vendor/<dep>/vendor/<sub-dep>/`. The main repo oracle is still the acceptance judge — sub-dep changes must improve (or not regress) the main repo.

**8. Update state:** Mark the dependency as `kept`, `reverted`, or `skipped` in `LAZY-DEV-STATE.md` with the main repo impact deltas.

### 3.5c. Completion

1. Run all main repo fitness functions one final time
2. Update `LAZY-DEV-STATE.md` with the dependency results summary:
   - Total dependencies processed, kept, reverted, skipped
   - Per-dependency status and main repo impact deltas
3. Archive all vendor artifacts to `.lazy-developer/dep-optimization-archive/`
4. Commit: `git add -A && git commit -m "lazy-dev: Dependency optimization complete — [N] deps processed, [N] kept"`

## Step 4: Completion Report

After all phases are complete (or all remaining phases are skipped):

### Standalone Mode

Update `LAZY-DEV-STATE.md` with a final summary section:

```markdown
## Final Report

**Completed:** [date]
**Phases run:** [N] of [total]
**Phases skipped:** [list with reasons]

### Results Summary

| Phase | Metric | Baseline | Final | Improvement |
|-------|--------|----------|-------|-------------|
| ... | ... | ... | ... | ...% |

### Total Commits

[N] optimization commits across all phases.

### Dependency Optimization Results

(Include this section only if dependency optimization was enabled)

| Dependency | Ecosystem | Status | Phases Run | Main Repo Impact |
|------------|-----------|--------|------------|------------------|
| [dep name] | [npm/go/...] | kept/reverted/skipped | [list] | [metric deltas] |
| ... | ... | ... | ... | ... |

**Summary:** [N] dependencies processed, [N] kept, [N] reverted, [N] skipped.
**Net main repo impact:** [aggregate metric changes from all kept dependencies]
```

Commit the final state: `git add -A && git commit -m "lazy-dev: All phases complete — see LAZY-DEV-STATE.md for results"`

### Ralph Mode

The final ralph instance (executing the last story) must:

1. Update `LAZY-DEV-STATE.md` with the same Final Report structure as above
2. Append a final summary to `progress.txt`
3. Set the last story's `passes` to `true` in `prd.json`
4. Commit all changes
5. Output `<promise>COMPLETE</promise>` to signal ralph.sh that all work is done

## Resumption

If the skill is interrupted and re-invoked:

### Standalone Mode

1. Check for an existing `LAZY-DEV-STATE.md` in the repository root
2. If found, read it and identify the current state:
   - If a phase is marked `in-progress`, check for an existing `GOAL.md` and `iterations.jsonl` — resume the improvement loop from where it left off
   - If the last phase is `completed`, advance to the next `pending` phase
3. If not found, start from Step 1 (Pipeline Configuration)
4. Read the `## Configuration` section to restore the user's chosen phase order and mode

### Ralph Mode

Each fresh ralph instance has no memory of previous instances. Continuity comes from files:

1. Read `prd.json` — find the highest-priority story where `passes: false`
2. Read `LAZY-DEV-STATE.md` — understand project state, completed phases, baselines
3. Read `progress.txt` — check Codebase Patterns section first for accumulated learnings
4. Read `.lazy-developer/CLAUDE.md` — follow the protocol and constraints
5. If the current story's phase was partially completed (GOAL.md exists, iterations.jsonl has entries), continue the loop from the current state rather than restarting
6. Execute the story, update all state files, commit, and check for overall completion

### Dependency Optimization

If the dependency optimization pass was interrupted:

1. Read `LAZY-DEV-STATE.md` — check the `## Dependency Optimization` section for per-dependency status
2. If a dependency is marked `in-progress`, check for its vendor directory (`.lazy-developer/vendor/<dep>/`) and any existing GOAL.md or iterations.jsonl within it — resume the optimization loop from the current state
3. If a dependency is marked `completed` or `reverted`, skip it
4. Re-establish the main repo oracle from `dep-oracle.json` if it exists, or re-run all fitness functions to regenerate it
5. Continue processing the next unprocessed dependency in the inventory

This makes both modes resilient to interruptions — progress is never lost.

## Error Handling

### Common to Both Modes

- **Build failure before starting:** Report the error and stop. Do not attempt to fix pre-existing build failures.
- **Test failure before starting:** Report the error and stop. Do not attempt to fix pre-existing test failures.
- **Fitness function script error:** Debug the script, fix it, and retry. If unfixable after 3 attempts, skip the phase with reason "fitness function failed".
- **All iterations stalled:** If 5 consecutive iterations produce no improvement, stop the phase and move on.
- **Complexity tool not available:** Try to install one automatically. If installation fails (e.g., no package manager, incompatible platform), skip the phase.

### Ralph Mode Specific

- **prd.json corrupted or missing:** If `prd.json` cannot be parsed, check git history for the last valid version and restore it. If unrecoverable, report the error and output nothing (ralph.sh will retry on next iteration).
- **State file conflict:** If `LAZY-DEV-STATE.md` has merge conflicts or inconsistencies, resolve based on `prd.json` as the source of truth for which phases are complete.
- **Partial phase from previous instance:** If GOAL.md and iterations.jsonl exist but the phase isn't marked complete, the previous instance was interrupted mid-phase. Continue from the last recorded iteration rather than restarting the phase.

### Dependency Optimization Specific

- **Clone failure:** If `git clone` fails for a dependency (private repo, invalid URL, network error), skip that dependency, log it as `skipped — clone failed` in the state file, and continue to the next dependency.
- **Inline failure:** If the ecosystem-specific inlining mechanism fails (e.g., `npm install` errors after modifying `package.json`), revert the manifest change, skip the dependency with reason `inline failed`, and continue.
- **Dep optimization stall:** If a dependency's optimization loop stalls (5 consecutive iterations with no improvement across all configured phases), evaluate the current state against the main repo oracle. Keep changes if they pass the oracle threshold, revert otherwise.
- **Recursive sub-dep failure:** If a sub-dependency (depth > 1) fails to clone, inline, or optimize, skip only that sub-dependency. The parent dependency's optimizations are unaffected.
- **Disk space >500MB:** If `.lazy-developer/vendor/` exceeds 500MB total, warn the user and stop processing additional dependencies. Evaluate and finalize any in-progress dependency before stopping.
