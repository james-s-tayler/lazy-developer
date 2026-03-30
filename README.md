# Lazy Developer

**Autonomously optimize every aspect of your codebase with a single command `/lazy-developer` or completely trash it and protect yourself from the coming AI job losses with `/job-security`**

## Introduction

Automate yourself out of a job by using Claude Code to perform multiple rounds of [autoresearch](https://github.com/alvinunreal/awesome-autoresearch) in a [Ralph Wiggum Loop](https://github.com/snarktank/ralph) using the [GOAL.md](https://github.com/jmilinovich/goal-md) pattern against a prioritized set of goals. First make the code safe to work with by increasing test coverage, then make it fast to work with by reducing build and test times, then make it nice to work with by improving code quality, then make it performant. All while you sleep. Because that's just where we're at on the timeline...

## Too lazy to install this yourself?

It's hard work being a code monkey. Have Claude do it:

```
Claude, what's happening?
If I could get you to go ahead and take a look at https://github.com/james-s-tayler/lazy-developer
and use it to automate myself out of a job? That'd be great. 
```

## Installation

Add this marketplace to Claude Code:

```
/plugin marketplace add james-s-tayler/skills
```

Then install individual skills:

```
/plugin install <skill-name>@james-s-tayler-skills
```

## Available Skills

### autoresearch

Set up autonomous goal-driven optimization on any repo. Based on the [GOAL.md](https://github.com/jmilinovich/goal-md) pattern.

```
/plugin install autoresearch@james-s-tayler-skills
```

**What it does:**

1. Reads the GOAL.md spec and examples from [jmilinovich/goal-md](https://github.com/jmilinovich/goal-md)
2. Scans your project to understand what "better" means
3. Asks you what metric to optimize, the direction (minimize/maximize/target), and your constraints
4. Writes a complete GOAL.md with a runnable fitness function and scoring script
5. Asks clarifying questions about unstated constraints
6. Starts an autonomous improvement loop — measure, diagnose, act, verify, commit or revert, log, repeat

**Usage:**

```
/autoresearch
```

### lazy-developer

Orchestrates autoresearch across a configurable sequence of optimization goals. You choose the phase priority and execution mode, and it handles the rest — writing GOAL.md files, running improvement loops, and tracking state across phases.

```
/plugin install lazy-developer@james-s-tayler-skills
```

**What it does:**

1. **Configuration** — Asks you two questions before starting:
   - **Phase order** — Safety First (coverage first), Speed First (build/test speed first), Clean Code First (complexity first), or Custom
   - **Execution mode** — Standalone (runs everything in one session) or Ralph Mode (generates artifacts for [ralph](https://github.com/snarktank/ralph) to drive each phase as a separate AI instance)
2. **Discovery** — Scans the target codebase to understand its structure, tools, test framework, build system, and all measurable metrics
3. **Sequential optimization** — Works through goals in the chosen order:
   - **Test Coverage** — maximize coverage to make the codebase safer to modify (locks production code)
   - **Test Speed** — minimize test execution time for cheaper iteration cycles (locks production code)
   - **Build Speed** — minimize build time for faster feedback loops (locks test files)
   - **Cyclomatic Complexity** — reduce complexity for maintainability (locks test files)
   - **Lines of Code** — reduce total LOC by removing dead, duplicated, and verbose code (locks test files)
   - **Performance** — optimize benchmark results (locks test files)
   - **Discovered metrics** — any additional project-specific metrics found during discovery
4. **Per-goal orchestration** — for each goal, automatically writes a GOAL.md with fitness function, runs the autoresearch improvement loop until stopping conditions are met, records results, and advances to the next goal
5. **State persistence** — tracks progress in `LAZY-DEV-STATE.md` so it can resume if interrupted
6. **Recursively improve dependencies (experimental)** - after optimization completes, optionally proceed to vendor in direct dependencies and run autoresearch against them to see if doing so improves the current repo. I'm too lazy to actually test this mode myself. YMMV, wildly.

**Engineering constraints enforced across all phases:**
- Build must always pass
- All tests must always pass
- Tests can never be deleted
- File locking by phase (test phases lock prod code, prod phases lock test files)
- Each change committed individually with clear messages
- Regressions are reverted immediately

**Execution modes:**

| Mode | How it works |
|------|-------------|
| **Ralph Mode** (recommended) | After discovery, generates `prd.json`, `.lazy-developer/CLAUDE.md`, and `progress.txt`, then stops. You run [`ralph.sh`](https://github.com/snarktank/ralph) to execute each phase as a fresh AI instance. If ralph isn't installed, lazy-developer offers to download and set it up for you. Best for long-running optimizations or when you want to review between phases. |
| **Standalone** | Runs all phases continuously in a single session. |

**Usage:**

```
/lazy-developer
```

**Example run — [reactiveui/refit](https://github.com/reactiveui/refit):**

Refit is a mature .NET REST library with Roslyn source generators, 547 xUnit tests, and multi-target builds (net462, netstandard2.0, net8.0, net9.0, net10.0). Here's what lazy-developer achieved running in Ralph Mode overnight:

```
Discovery (Phase 0)
├── Languages: C# (.NET 8/9/10, .NET Standard 2.0, .NET 4.6.2)
├── Build: dotnet build Refit.sln --no-incremental -c Release (~14.6s)
├── Test: 501 tests + 46 generator tests (~7.7s full suite)
├── Coverage: 65.14% line, 59.56% branch
├── Complexity: 2.98 avg CC, 7 functions with CC>15, max CC=37
└── Baselines recorded, 5 phases planned

Test Speed (Phase 2)
├── Baseline: 9.85s
├── Final: 2.0s (79.7% improvement)
├── Parallel execution of both test projects (9.85s → 6.50s)
├── Direct vstest invocation on pre-built assemblies (6.50s → 2.0s)
└── Key insight: dotnet test has ~4s startup overhead — dotnet vstest skips it

Build Speed (Phase 3)
├── Baseline: 12.50s
├── Final: 6.50s (48.0% improvement)
├── Disabled analyzers during Release builds (~40% win alone)
├── Reduced test TFMs to net9.0 for non-CI builds (~22% additional)
└── Key insight: AllEnabledByDefault analyzer mode is extremely expensive

Cyclomatic Complexity (Phase 4)
├── Baseline: 3.0 avg CC, 7 functions with CC>15
├── Final: 2.8 avg CC, 0 functions with CC>15
├── RestMethodInfoInternal ctor: CC 37→14
├── BuildRequestFactoryForMethod: CC 33→8
├── BuildCancellableTaskFuncForMethod: CC 21→10
├── FormValueMultimap: CC 20→5
└── All hotspots >CC 15 eliminated via method extraction

Performance (Phase 5)
├── Baseline: 3666.17 ns
├── Final: 2279.93 ns (37.8% improvement)
├── Eliminated DynamicInvoke with strongly-typed delegates
├── Cached reflection in BuildQueryMap (ConcurrentDictionary)
├── Skipped UriBuilder for routes without hardcoded query strings
└── Replaced LINQ operations with direct loops and index lookups
```

### job-security

The opposite of lazy-developer. First maximizes test coverage to create a safety net, then methodically makes everything worse — slower builds, slower tests, higher complexity, more lines of code, and degraded performance. All changes must be genuine (no `Thread.Sleep` or `Task.Delay` allowed) and the build and tests must keep passing throughout.

```
/plugin install job-security@james-s-tayler-skills
```

**What it does:**

1. **Configuration** — Asks you two questions:
   - **Execution mode** — Standalone or Ralph Mode (same as lazy-developer)
   - **Stopping strategy** — Diminishing returns (keep going until progress stalls) or Default targets met
2. **Discovery** — Same as lazy-developer — scans the codebase to understand structure and establish baselines
3. **Fixed-order de-optimization** — Works through goals in a fixed sequence:
   - **Test Coverage** — maximize coverage first so the remaining phases are safe (locks production code)
   - **Build Speed** — increase build time through genuine complexity (locks test files)
   - **Test Speed** — increase test execution time without artificial delays (locks production code)
   - **Cyclomatic Complexity** — increase complexity by inlining helpers, expanding conditionals, etc. (locks test files)
   - **Lines of Code** — increase LOC by expanding concise patterns, duplicating logic, etc. (locks test files)
   - **Performance** — degrade performance through less efficient algorithms and data structures (locks test files)
4. **Same orchestration engine** — uses GOAL.md, fitness functions, improvement loops, and state tracking just like lazy-developer

**Additional constraint beyond lazy-developer:**
- **No artificial delays** — `Thread.Sleep`, `Task.Delay`, `time.sleep`, `setTimeout`, `sleep()`, and all equivalents are forbidden. All time increases and performance degradation must come from genuine code changes.

**Usage:**

```
/job-security
```

**Example run — [reactiveui/refit](https://github.com/reactiveui/refit):**

Same codebase as the lazy-developer example above, but this time running job-security in Ralph Mode to make everything worse:

```
Discovery (Phase 0)
├── Languages: C# (.NET 8/9/10, .NET Standard 2.0, .NET 4.6.2)
├── Build: dotnet build Refit.sln --no-incremental -c Release (~13s)
├── Test: 602 tests (~1.6s)
├── Coverage: 66.9% line
├── Complexity: 3.0 avg CCN (338 functions)
├── LOC: 10,266 production lines
└── Baselines recorded, 6 phases planned

Build Speed (Phase 2)
├── Baseline: 16.64s
├── Final: 24.77s (+48.9% slower)
├── Added 6 Roslyn analyzers (StyleCop, SonarAnalyzer, Meziantou, Roslynator, BannedApi, PublicApi)
├── Added net6.0/net7.0 TFMs across all production projects
├── Disabled build parallelism, switched to full PDBs
└── Key insight: extra TFMs are highest-impact — each adds ~15% more compilation units

Test Speed (Phase 3)
├── Baseline: 1.64s (602 tests)
├── Final: 409.73s / 190,567 tests (+24,882% slower)
├── Disabled test parallelism (maxParallelThreads=1) for +58% alone
├── Added ~190k parameterized tests via 7-way combinatorial MemberData
├── XML serializer tests are ~4x slower per-test than JSON — high-value additions
└── Key insight: 7×4×4×3×4×4×4 = 21,504 test cases per method from compact MemberData

Cyclomatic Complexity (Phase 4)
├── Baseline: 3.0 avg CCN (338 functions)
├── Final: 3.5 avg CCN (325 functions) (+16.7% increase)
├── Replaced ~80 LINQ usages with explicit foreach+if loops
├── Expanded ~40 ternary expressions to full if/else blocks
├── Added parameter validation (null checks, range checks) to ~30 methods
└── Key insight: LINQ→foreach+if is highest-impact — each adds 1-2 CCN from loop+condition

Lines of Code (Phase 5)
├── Baseline: 11,091 production LOC
├── Final: 13,857 production LOC (+24.9% increase)
├── 20 iterations of code expansion across all production files
├── Highest-yield: accessor expansion (+302), multi-arg formatting (+268), if-condition extraction (+185)
└── Key insight: method arg expansion to one-per-line is completely safe and very high yield

Performance (Phase 6)
├── Baseline: 3.489 μs average mean execution time
├── Final: 194.268 μs (+5,467% slower, 55.7x degradation)
├── Replaced efficient patterns with Regex(RegexOptions.Compiled) — ~30 μs each
├── Added Expression.Lambda.Compile() in per-parameter hot paths — ~30 μs each
├── Inserted JSON serialization round-trips and reflection (GetCustomAttributes, GetProperties)
└── Key insight: Regex.Compiled and Expression.Compile in per-parameter paths multiply cost by parameter count
```
