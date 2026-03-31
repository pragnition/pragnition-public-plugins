# RAPID

RAPID (Rapid Agentic Parallelizable and Isolatable Development) is a Claude Code plugin that enables coordinated parallel AI-assisted development. It structures work around independent sets -- each running in an isolated git worktree with strict file ownership -- connected through machine-verifiable interface contracts and validated through a multi-stage adversarial review pipeline. 28 specialized agents handle research, planning, execution, review, and merge so developers focus on decisions, not coordination.

## Install

```
/plugin install rapid@pragnition-plugins
```

Or clone manually:

```bash
git clone https://github.com/pragnition/RAPID.git
cd RAPID
./setup.sh
```

After either installation method, run `/rapid:install` inside Claude Code to configure the `RAPID_TOOLS` environment variable.

## Requirements

- Node.js 18+
- git 2.30+ (required for worktree support)
- Claude Code (latest version)
- `RAPID_TOOLS` env var must be set (both installation methods handle this)

## Quick Start

A typical RAPID workflow follows these stages:

```
/rapid:install                  # One-time setup: configure RAPID_TOOLS env var
/rapid:init                     # Research, roadmap generation, scaffold .planning/
/rapid:context                  # Analyze existing codebase (skip for greenfield)

# Per set (in parallel across developers):
/rapid:start-set <set>          # Create worktree and branch
/rapid:discuss-set <set>        # Capture implementation vision (or --skip)
/rapid:plan-set <set>           # Research, plan, verify pipeline
/rapid:execute-set <set>        # Run waves sequentially, one agent per wave
/rapid:review <set>             # Unit test + bug hunt + UAT pipeline

# After sets complete:
/rapid:merge                    # 5-level conflict detection, DAG-ordered merge
/rapid:cleanup <set>            # Remove completed worktrees
/rapid:new-version              # Archive, bump version, re-plan
```

## Commands

### Core Lifecycle (7 commands)

These commands form the linear workflow from project initialization to merge:

```
INIT --> START-SET --> DISCUSS-SET --> PLAN-SET --> EXECUTE-SET --> REVIEW --> MERGE
```

#### `/rapid:init`

Bootstraps a new RAPID project. Validates prerequisites, runs an adaptive 4-batch discovery conversation, optionally spawns a `rapid-codebase-synthesizer` for brownfield projects, spawns 6 parallel research agents (stack, features, architecture, pitfalls, oversights, UX), synthesizes findings, and generates a roadmap with sets for approval. Up to 9 agents spawned. No arguments -- interactive discovery gathers all inputs.

Output:

```
.planning/
  PROJECT.md          -- Project description and core value
  REQUIREMENTS.md     -- Requirements extracted from discovery
  ROADMAP.md          -- Sets with dependency ordering
  STATE.json          -- Machine-readable state (set-level)
  CONTRACT.json       -- Interface contracts between sets
  config.json         -- Project configuration
  research/           -- Individual research outputs
```

#### `/rapid:start-set <set-id>`

Claims a set for development by creating an isolated git worktree at `.rapid-worktrees/{set-name}/` on branch `rapid/{set-name}`. Generates a scoped CLAUDE.md with set-specific context and deny lists. Spawns a `rapid-set-planner` agent to produce SET-OVERVIEW.md. Accepts string IDs (e.g., `auth-system`) or numeric indices (e.g., `1`).

#### `/rapid:discuss-set <set-id> [--skip]`

Captures developer implementation vision for a set before planning. Identifies exactly 4 gray areas where multiple valid approaches exist, asks batched questions per area, and records decisions in CONTEXT.md. Use `--skip` to auto-generate CONTEXT.md without interaction via a research agent. State transition: `pending` --> `discussing`.

#### `/rapid:plan-set <set-id>`

Plans all waves in a set using a 3-step pipeline:

1. **Research** -- `rapid-research-stack` investigates implementation specifics.
2. **Planning** -- `rapid-planner` decomposes the set into 1-4 waves with per-wave PLAN.md files.
3. **Verification** -- `rapid-plan-verifier` validates coverage, implementability, and consistency.

If verification fails, the planner re-runs once automatically. Contract validation runs after planning completes (advisory during planning, enforced at merge). 3-4 agents spawned. State transition: `discussing` --> `planning`.

Output:

```
.planning/sets/{set-name}/
  wave-1-PLAN.md          -- Tasks for wave 1
  wave-2-PLAN.md          -- Tasks for wave 2 (if multi-wave)
  VERIFICATION-REPORT.md  -- Plan verifier report
```

#### `/rapid:execute-set <set-id>`

Executes all waves in a set sequentially with one `rapid-executor` agent per wave. Detects completed waves via WAVE-COMPLETE.md markers and git commit verification for crash recovery. After all waves complete, spawns a `rapid-verifier` agent to check objectives. Re-run after a crash to resume from the last checkpoint. State transition: `planning` --> `executing` --> `complete`.

#### `/rapid:review <set-id>`

Validates a completed set through a multi-stage adversarial review pipeline. User selects which stages to run:

- **Scoping**: Diffs set branch vs main, categorizes files into concern groups via a scoper agent.
- **Unit testing**: Generates test plans and executes tests (one unit-tester agent per concern group, parallel).
- **Adversarial bug hunt** (up to 3 cycles): Bug hunter finds bugs (parallel per concern group), devils advocate challenges findings with counter-evidence, judge rules ACCEPTED/DISMISSED/DEFERRED, bugfix agent fixes accepted bugs. Cycles 2-3 narrow scope to only modified files.
- **UAT**: Generates acceptance test plans with automated (browser) and human-verified steps.

Output: REVIEW-UNIT.md, REVIEW-BUGS.md, REVIEW-UAT.md, REVIEW-SUMMARY.md.

#### `/rapid:merge [set-id]`

Merges completed sets into main with dependency-ordered (DAG) processing. Clean merges skip conflict detection entirely via fast-path `git merge-tree` check.

For conflicting merges, spawns `rapid-set-merger` subagents with 5-level conflict detection:

| Level | Type | Description |
|-------|------|-------------|
| L1 | Textual | Line-level conflicts in the same file |
| L2 | Structural | Incompatible code structure changes |
| L3 | Dependency | Conflicting package or import changes |
| L4 | API | Breaking interface changes between sets |
| L5 | Semantic | Logically incompatible behavior changes |

Resolution uses a 4-tier cascade: T1 auto-resolved (>0.9 confidence), T2 auto-resolved with review flag (0.7-0.9), T3 dispatched to `rapid-conflict-resolver` agent (0.3-0.7), T4 escalated to developer (<0.3). API-signature conflicts always escalate to the developer. State transition: `complete` --> `merged` (terminal).

### Auxiliary (4 commands)

#### `/rapid:status`

Read-only dashboard showing all sets with statuses, last git activity per branch, and actionable next-step suggestions. Supports numeric shorthand for suggested actions.

#### `/rapid:install`

One-time plugin setup. Detects the shell (bash, zsh, fish, POSIX), writes `RAPID_TOOLS` to the shell config, creates `.env` fallback, validates the toolchain, and runs agent file generation. No arguments.

#### `/rapid:new-version`

Completes the current milestone and starts a new planning cycle. Archives the current milestone, gathers new milestone details, handles unfinished sets with carry-forward options, re-runs the full 6-researcher pipeline scoped to new goals, and generates a new roadmap for approval.

#### `/rapid:add-set <set-name>`

Adds a new set to the current milestone mid-stream through a lightweight 2-question discovery flow. Creates DEFINITION.md and CONTRACT.json, updates STATE.json and ROADMAP.md. No subagent spawns.

### Utilities (6 commands)

#### `/rapid:quick <description>`

Ad-hoc changes without set structure. Runs a 3-agent pipeline (planner, plan-verifier, executor) in-place on the current branch. Quick tasks are stored in `.planning/quick/` and excluded from STATE.json.

#### `/rapid:assumptions <set-id>`

Read-only. Surfaces Claude's mental model about a set's scope, file boundaries, contract assumptions, dependency assumptions, and risk factors for developer validation before execution.

#### `/rapid:pause <set-id>`

Saves execution state to HANDOFF.md for later resumption. Records current wave progress, user notes, and checkpoint data. Warns after 3 pause cycles that the set scope may be too large.

#### `/rapid:resume <set-id>`

Resumes a paused set from its last checkpoint. Loads HANDOFF.md context, presents the handoff summary, and transitions the set back to executing.

#### `/rapid:context`

Analyzes existing codebase and generates context files. Spawns a `rapid-context-generator` agent for deep analysis, then writes CLAUDE.md (under 80 lines), CODEBASE.md, ARCHITECTURE.md, CONVENTIONS.md, and STYLE_GUIDE.md in `.planning/context/`. Re-runnable at any time. Skip for greenfield projects.

#### `/rapid:cleanup <set-id>`

Removes a completed set's worktree with safety checks. Blocks removal if uncommitted changes exist. Offers optional branch deletion with double-confirmation for unmerged branches.

### Standalone Pipeline Commands

These commands run individual review pipeline stages independently (outside of `/rapid:review`):

#### `/rapid:bug-fix`

Investigate and fix bugs. User describes a bug, the model investigates and applies a fix.

#### `/rapid:bug-hunt`

Run an adversarial bug hunt on a scoped set. Reads REVIEW-SCOPE.md for file targeting.

#### `/rapid:unit-test`

Run unit test pipeline on a scoped set. Reads REVIEW-SCOPE.md for file targeting.

#### `/rapid:uat`

Run user acceptance testing on a scoped set. Reads REVIEW-SCOPE.md for file targeting.

### Additional Utilities

#### `/rapid:scaffold`

Generate project-type-aware foundation files for the target codebase.

#### `/rapid:documentation`

Generate, update, and maintain project documentation from git history and RAPID artifacts.

#### `/rapid:branding`

Conduct a structured branding interview with codebase-aware visual/UX brand guidelines.

#### `/rapid:migrate`

Migrate `.planning/` state from older RAPID versions to the current version.

#### `/rapid:help`

Static command reference and workflow guide.

## Configuration

### .planning/ Directory

Created by `/rapid:init`:

| File/Directory | Purpose |
|----------------|---------|
| `PROJECT.md` | Project description and core value |
| `REQUIREMENTS.md` | Requirements extracted from discovery |
| `ROADMAP.md` | Sets with dependency ordering |
| `STATE.json` | Machine-readable state (set-level only) |
| `CONTRACT.json` | Interface contracts between sets |
| `config.json` | Project configuration |
| `research/` | Individual research agent outputs |
| `context/` | Generated context files: CODEBASE.md, ARCHITECTURE.md, CONVENTIONS.md, STYLE_GUIDE.md |
| `sets/{set-name}/` | Per-set artifacts: DEFINITION.md, CONTRACT.json, CONTEXT.md, PLAN.md files |
| `quick/` | Quick task artifacts (excluded from STATE.json) |

### Environment Variables

| Variable | Purpose | Default |
|----------|---------|---------|
| `RAPID_TOOLS` | Absolute path to `rapid-tools.cjs` CLI | Must be set via `/rapid:install` or `setup.sh` |

## Key Concepts

**Sets** -- Independent workstreams that run in isolated git worktrees. Each set has strict file ownership and connects to other sets through interface contracts. Sets are the sole stateful entity in STATE.json. Lifecycle: `pending -> discussing -> planning -> executing -> complete -> merged`.

**Waves** -- Execution groups within sets. Within a set, waves execute sequentially with one executor agent per wave. Crash recovery detects completed waves via WAVE-COMPLETE.md markers and git commits.

**Interface Contracts** -- Machine-verifiable JSON schemas (CONTRACT.json) defining exports, imports, and file ownership between sets. Validated after planning, during execution, and before merge.

**File Ownership** -- Every file belongs to exactly one set. Cross-set file access is tracked. Ownership violations are detected during reconciliation and merge. If two sets need the same file, restructure the decomposition in `/rapid:plan-set`.

**State Machine** -- Set-level JSON state (STATE.json) with lock-protected atomic writes. Crash recovery via `detectCorruption` and `recoverFromGit`. No wave or task state in STATE.json.

## Agents

28 agents organized into 6 categories:

- **Core (4):** planner, executor, merger, reviewer
- **Research (7):** 6 domain researchers (stack, features, architecture, pitfalls, oversights, UX) + synthesizer
- **Review (7):** scoper, unit-tester, bug-hunter, devils-advocate, judge, bugfix, uat
- **Merge (2):** set-merger, conflict-resolver
- **Utility (5):** roadmapper, set-planner, plan-verifier, verifier, codebase-synthesizer
- **Context (1):** context-generator

## File Structure

```
RAPID/
  .claude-plugin/plugin.json       Plugin manifest
  skills/                          Skill definitions (one per command)
  agents/                          Specialized agent definitions
  src/
    bin/rapid-tools.cjs            CLI tool library
  setup.sh                         Installation script
  README.md                        User-facing overview
  DOCS.md                          Full technical reference
```

## Tips

- Run `/rapid:install` first after marketplace installation to configure the `RAPID_TOOLS` environment variable.
- Run `/rapid:context` before planning on existing codebases to ensure generated code matches project conventions.
- Use `/rapid:assumptions` to catch misunderstandings about a set's scope before committing to execution.
- Use `--skip` on `/rapid:discuss-set` to save an interactive session when you trust Claude's defaults.
- If execution is interrupted, re-run `/rapid:execute-set` -- it detects completed waves and resumes from the last checkpoint.
- If planning crashes, re-run `/rapid:plan-set` -- it is idempotent and will overwrite incomplete plan files.
- If merge crashes, re-run `/rapid:merge` -- MERGE-STATE.json tracks which sets have been processed for idempotent re-entry.
- The adversarial bug hunt runs up to 3 cycles, narrowing scope each time.
- File ownership is strict by design. If two sets need the same file, restructure the decomposition.
- The merge command respects dependency order automatically via topological sort of the DAG. Clean merges skip conflict detection entirely.
- Use `/rapid:pause` and `/rapid:resume` for long-running sets that span multiple sessions.
- Commands accepting `<set-id>` support both string IDs (e.g., `auth-system`) and numeric indices (e.g., `1`).
- `/rapid:quick` uses only 3 agent spawns for ad-hoc changes, avoiding the full set lifecycle overhead.
- The adversarial bug hunt is the most expensive review stage. Use stage selection in `/rapid:review` to run only the stages you need.
