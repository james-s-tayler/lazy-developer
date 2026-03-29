# Lazy Developer

> **Autonomously optimize every aspect of your codebase with a single command: `/lazy-developer`**

## Introduction

Automate yourself out of a job by using Claude Code to perform multiple rounds of [autoresearch](https://github.com/jmilinovich/goal-md) in a [Ralph Wiggum Loop](https://github.com/snarktank/ralph) using the [GOAL.md](https://github.com/jmilinovich/goal-md) pattern against a prioritized set of goals. First make the code safe to work with by increasing test coverage, then make it fast to work with by reducing build and test times, then make it nice to work with by improving code quality, then make it performant. Because that's just where we're at on the timeline...

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
   - **Performance** — optimize benchmark results (locks test files)
   - **Discovered metrics** — any additional project-specific metrics found during discovery
4. **Per-goal orchestration** — for each goal, automatically writes a GOAL.md with fitness function, runs the autoresearch improvement loop until stopping conditions are met, records results, and advances to the next goal
5. **State persistence** — tracks progress in `LAZY-DEV-STATE.md` so it can resume if interrupted

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
/lazy-developer path/to/specific/repo
/lazy-developer focus on API performance
```
