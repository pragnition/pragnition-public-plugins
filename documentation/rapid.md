# RAPID

RAPID (Rapid Agentic Parallelizable and Isolatable Development) is a Claude Code plugin that enables coordinated parallel AI-assisted development. It structures work around independent sets -- each running in an isolated git worktree with strict file ownership -- connected through machine-verifiable interface contracts and validated through a multi-stage adversarial review pipeline. 27 specialized agents handle research, planning, execution, review, and merge so developers focus on decisions, not coordination.

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

- Node.js 20+
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
/rapid:review <set>             # Scope set for review, produces REVIEW-SCOPE.md
/rapid:unit-test <set>          # Run unit tests against scoped set
/rapid:bug-hunt <set>           # Adversarial bug hunt against scoped set
/rapid:uat <set>                # User acceptance testing against scoped set

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
                                                                     |
                                                          unit-test, bug-hunt, uat
```

#### `/rapid:init`

Bootstraps a new RAPID project. Validates prerequisites, runs a 4-batch discovery conversation, spawns 6 parallel research agents (stack, features, architecture, pitfalls, oversights, UX), synthesizes findings, and generates a roadmap with sets for approval. For brownfield projects, a codebase synthesizer analyzes existing code first. No arguments -- interactive discovery gathers all inputs.

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

Claims a set for development by creating an isolated git worktree at `.rapid-worktrees/{set-name}/` on branch `rapid/{set-name}`. Generates a scoped CLAUDE.md with set-specific contracts and deny lists. Spawns a `rapid-set-planner` agent to produce SET-OVERVIEW.md. Accepts string IDs (e.g., `auth-system`) or numeric indices (e.g., `1`).

#### `/rapid:discuss-set <set-id> [--skip]`

Captures developer implementation vision for a set before planning. Identifies exactly 4 gray areas where multiple valid approaches exist, asks batched questions per area, and records decisions in CONTEXT.md. Use `--skip` to auto-generate CONTEXT.md without interaction via a research agent. State transition: `pending` --> `discussed`.

#### `/rapid:plan-set <set-id> [--gaps]`

Plans all waves in a set using a 3-step pipeline:

1. **Research** -- `rapid-research-stack` investigates implementation specifics.
2. **Planning** -- `rapid-planner` decomposes the set into 1-4 waves with per-wave PLAN.md files.
3. **Verification** -- `rapid-plan-verifier` validates coverage, implementability, and consistency.

If verification fails, the planner re-runs once automatically. Contract validation runs after planning completes (advisory during planning, enforced at merge). The `--gaps` flag enables gap-closure mode for addressing post-merge gaps. 3-4 agents spawned. State transition: `discussed` --> `planned`.

Output:

```
.planning/sets/{set-name}/
  wave-1-PLAN.md          -- Tasks for wave 1
  wave-2-PLAN.md          -- Tasks for wave 2 (if multi-wave)
  VERIFICATION-REPORT.md  -- Plan verifier report
```

#### `/rapid:execute-set <set-id> [--gaps]`

Executes all waves in a set sequentially with one `rapid-executor` agent per wave. Detects completed waves via WAVE-COMPLETE.md markers and git commit verification for crash recovery. After all waves complete, spawns a `rapid-verifier` agent to check objectives. The `--gaps` flag enables gap-closure mode, executing plans generated from post-merge gap analysis (GAPS.md). Re-run after a crash to resume from the last checkpoint. State transition: `planned` --> `executed` --> `complete`.

#### `/rapid:review <set-id>`

Scopes a completed set for review by diffing the set branch against main, identifying changed files and their dependents, and producing `REVIEW-SCOPE.md`. This artifact is consumed by the downstream review skills (`/rapid:unit-test`, `/rapid:bug-hunt`, `/rapid:uat`). Does not run tests or hunt bugs itself.

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

Read-only dashboard showing all sets with statuses, last git activity per branch, and actionable next-step suggestions. Displays sets in DAG wave order. Supports numeric shorthand for suggested actions.

#### `/rapid:install`

One-time plugin setup. Detects the shell (bash, zsh, fish, POSIX), writes `RAPID_TOOLS` to the shell config, creates `.env` fallback, validates the toolchain, and runs agent file generation. No arguments.

#### `/rapid:new-version [--spec <path>]`

Completes the current milestone and starts a new planning cycle. Archives the current milestone, gathers new milestone details (or reads them from a spec file), handles unfinished sets with carry-forward options, auto-discovers DEFERRED.md files, re-runs the full 6-researcher pipeline scoped to new goals, and generates a new roadmap for approval.

#### `/rapid:add-set <set-name>`

Adds a new set to the current milestone mid-stream through a lightweight 2-question discovery flow. Creates DEFINITION.md and CONTRACT.json, updates STATE.json and ROADMAP.md. No subagent spawns.

### Utilities (7 commands)

#### `/rapid:quick <description>`

Ad-hoc changes without set structure. Runs a 3-agent pipeline (planner, plan-verifier, executor) in-place on the current branch. Fully autonomous after the initial task description. Quick tasks are stored in `.planning/quick/` and excluded from STATE.json.

#### `/rapid:assumptions <set-id>`

Read-only. Surfaces Claude's mental model about a set's scope, file boundaries, contract assumptions, dependency assumptions, and risk factors for developer validation before execution.

#### `/rapid:pause <set-id>`

Saves execution state to HANDOFF.md for later resumption. Records current wave/job progress, user notes, and checkpoint data. Warns after 3 pause cycles that the set scope may be too large.

#### `/rapid:resume <set-id>`

Resumes a paused set from its last checkpoint. Loads HANDOFF.md context, presents the handoff summary, and transitions the set back to executing.

#### `/rapid:context`

Analyzes existing codebase and generates context files. Spawns a `rapid-context-generator` agent for deep analysis, then writes CLAUDE.md (under 80 lines), CODEBASE.md, ARCHITECTURE.md, CONVENTIONS.md, and STYLE_GUIDE.md in `.planning/context/`. Re-runnable at any time. Skip for greenfield projects.

#### `/rapid:bug-fix <description> [--uat <set-id>]`

Investigates and fixes bugs. User describes a bug, the model investigates and applies a fix with atomic commits. Works from any branch -- no set association required. With `--uat <set-id>`, reads `UAT-FAILURES.md` from the set's planning directory and fixes reported failures automatically without manual investigation.

#### `/rapid:cleanup <set-id>`

Removes a completed set's worktree with safety checks. Blocks removal if uncommitted changes exist. Offers optional branch deletion with double-confirmation for unmerged branches.

### Standalone Pipeline Commands

These commands run individual review pipeline stages independently (outside of `/rapid:review`):

#### `/rapid:unit-test <set-id>`

Run unit test pipeline on a scoped set. Reads REVIEW-SCOPE.md for file targeting. Generates test plans and executes tests using the configured test framework. Results written to REVIEW-UNIT.md. Spawns `rapid-unit-tester` agents per concern group.

#### `/rapid:bug-hunt <set-id>`

Run adversarial bug hunt on a scoped set. Uses a hunter-advocate-judge pattern with up to 3 iterative fix-and-rehunt cycles. Reads REVIEW-SCOPE.md for file targeting. Accepted bugs dispatched to `rapid-bugfix` for targeted fixes. Results written to REVIEW-BUGS.md.

#### `/rapid:uat <set-id>`

Run user acceptance testing on a scoped set. Reads REVIEW-SCOPE.md for file targeting. Generates acceptance test plans with automated (browser automation) and human-verified steps. Results written to REVIEW-UAT.md.

### Additional Utilities

#### `/rapid:scaffold`

Generate project-type-aware foundation files for the target codebase. Detects the project type and scaffolds appropriate directory structure, config files, and boilerplate. Additive-only -- existing files are never overwritten.

#### `/rapid:documentation [--scope <full|changelog|api|architecture>] [--diff-only]`

Generate, update, and maintain project documentation from git history and RAPID artifacts. Supports scoped generation and diff-only mode for previewing changes without writing files.

#### `/rapid:branding`

Conduct a structured branding interview with codebase-aware visual/UX brand guidelines. Generates BRANDING.md that shapes how RAPID agents communicate and style output. Optional -- use before frontend-heavy sets.

#### `/rapid:audit-version [version]`

Audits a completed milestone by cross-referencing planned requirements against actual delivery. Produces a structured gap report at `.planning/v{version}-AUDIT.md`. Read-only -- never mutates state. Offers remediation through `/rapid:add-set` or deferral for the next version.

#### `/rapid:migrate [--dry-run]`

Migrate `.planning/` state from older RAPID versions to the current version. Handles schema changes, status renames, and structural updates. Supports dry-run mode to preview changes.

#### `/rapid:register-web`

Registers the current project with the RAPID Mission Control web dashboard. Only needed for projects initialized before v4.1.0. New projects auto-register during `/rapid:init` when `RAPID_WEB=true` is set.

#### `/rapid:help`

Static command reference and workflow guide. Displays all 28 commands organized by category with usage guidance.

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

**Sets** -- Independent workstreams that run in isolated git worktrees. Each set has strict file ownership and connects to other sets through interface contracts. Sets are the sole stateful entity in STATE.json. Lifecycle: `pending -> discussed -> planned -> executed -> complete -> merged`.

**Waves** -- Execution groups within sets. Within a set, waves execute sequentially with one executor agent per wave. Crash recovery detects completed waves via WAVE-COMPLETE.md markers and git commits.

**Interface Contracts** -- Machine-verifiable JSON schemas (CONTRACT.json) defining exports, imports, and file ownership between sets. Validated after planning, during execution, and before merge.

**File Ownership** -- Every file belongs to exactly one set. Cross-set file access is tracked. Ownership violations are detected during reconciliation and merge. If two sets need the same file, restructure the decomposition in `/rapid:plan-set`.

**State Machine** -- Set-level JSON state (STATE.json) with lock-protected atomic writes. Crash recovery via `detectCorruption` and `recoverFromGit`. No wave or task state in STATE.json.

## Agents

27 agents organized into 7 categories:

- **Core (4):** planner, executor, merger, reviewer
- **Research (7):** 6 domain researchers (stack, features, architecture, pitfalls, oversights, UX) + synthesizer
- **Review (7):** scoper, unit-tester, bug-hunter, devils-advocate, judge, bugfix, uat
- **Merge (2):** set-merger, conflict-resolver
- **Utility (6):** roadmapper, set-planner, plan-verifier, verifier, codebase-synthesizer, auditor
- **Context (1):** context-generator

## File Structure

```
RAPID/
  .claude-plugin/plugin.json       Plugin manifest
  skills/                          Skill definitions (one per command)
  agents/                          27 generated agent definitions
  src/
    bin/rapid-tools.cjs            CLI tool library
    commands/                      CLI command handlers
    hooks/                         Post-task verification hooks
    lib/                           Core libraries (state, merge, worktree, etc.)
    modules/                       Role definitions and core modules
    schemas/                       Zod validation schemas
  docs/                            Detailed documentation (11 files)
  web/                             Mission Control dashboard source
  branding/                        Branding assets
  test/                            Test suites
  config.json                      Project configuration
  package.json
  setup.sh                         Installation script
  README.md                        User-facing overview
  DOCS.md                          Full technical reference
  technical_documentation.md       Architectural deep-dive narrative
```

## Tips

- Run `/rapid:install` first after marketplace installation to configure the `RAPID_TOOLS` environment variable.
- Run `/rapid:context` before planning on existing codebases to ensure generated code matches project conventions.
- Use `/rapid:assumptions` to catch misunderstandings about a set's scope before committing to execution.
- Use `--skip` on `/rapid:discuss-set` to save an interactive session when you trust Claude's defaults.
- If execution is interrupted, re-run `/rapid:execute-set` -- it detects completed waves and resumes from the last checkpoint.
- If planning crashes, re-run `/rapid:plan-set` -- it is idempotent and will overwrite incomplete plan files.
- If merge crashes, re-run `/rapid:merge` -- MERGE-STATE.json tracks which sets have been processed for idempotent re-entry.
- The review pipeline is split: first run `/rapid:review` to scope, then run `/rapid:unit-test`, `/rapid:bug-hunt`, `/rapid:uat` independently against the scope.
- The adversarial bug hunt runs up to 3 cycles, narrowing scope each time. It is the most expensive review stage -- skip it for low-risk changes.
- File ownership is strict by design. If two sets need the same file, restructure the decomposition.
- The merge command respects dependency order automatically via topological sort of the DAG. Clean merges skip conflict detection entirely.
- Use `/rapid:pause` and `/rapid:resume` for long-running sets that span multiple sessions.
- Commands accepting `<set-id>` support both string IDs (e.g., `auth-system`) and numeric indices (e.g., `1`).
- `/rapid:quick` uses only 3 agent spawns for ad-hoc changes, avoiding the full set lifecycle overhead.
- Use `/rapid:audit-version` after merging to verify planned requirements were delivered.
