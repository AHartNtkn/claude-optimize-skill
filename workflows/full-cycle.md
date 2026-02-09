# Full Optimization Cycle

You are the orchestrator for an autonomous performance optimization loop. You generate candidates, dispatch parallel workers to isolated worktrees, collect results, merge winners, clean up code quality, discard losers, and loop.

## Prerequisites

- Working directory is a git repository root
- Current branch is clean (no uncommitted changes to source files)
- Project discovery (SKILL.md Step 0) has been completed

## The Loop

Repeat until the user interrupts (Ctrl+C):

### Step 1: SURVEY

Read the current state of the project:

1. Read performance investigation backlog (if one exists — e.g., `PERFORMANCE_INVESTIGATIONS.md`)
2. Read recent investigation reports (if a reports directory exists) — note "Remaining opportunities" sections
3. Check for existing flamegraph SVGs or profiling data
4. Read source files in known hotspot areas (identified via profiling or past reports)

If this is the first round and no profiling data exists, generate a profile:
- For Rust: use `perf record` + `inferno-flamegraph`, or `cargo flamegraph`, or whatever profiling infrastructure the project has
- For other languages: use the appropriate profiling tool
- If no profiling tool is available, rely on code structure analysis and speculation

### Step 2: GENERATE CANDIDATES

Generate a ranked list of 3-5 optimization candidates from four sources:

**Source 1 — Profiling hotspots:** Read profiling data and identify the top functions by inclusive time. Bottom-up analysis: "function X is Y% of runtime, what can we do about it?"

**Source 2 — Code structure analysis:** Read source files and identify:
- Allocation-heavy patterns (allocations in hot loops, unnecessary clones/copies)
- Cache-unfriendly data access patterns (pointer chasing, poor locality)
- Lock contention or synchronization overhead
- Redundant computation (same work done multiple times)
- Data structure inefficiencies (wrong container type, oversized enums, etc.)

**Source 3 — Backlog + past reports:** Pull from the performance backlog's open items and "Remaining opportunities" in recent investigation reports.

**Source 4 — Speculative redesigns:** Brainstorm architectural changes that could yield large improvements:
- Data structure replacements
- Algorithm redesigns
- Elimination of entire subsystems or passes
- These are explicitly encouraged because worktree isolation makes them safe to attempt

For each candidate, provide:
- **Slug**: snake_case identifier (e.g., `lockfree_intern`)
- **Title**: one-line description
- **Hypothesis**: expected mechanism of improvement
- **Estimated ROI**: rough percentage based on profiling data
- **Risk**: low/medium/high (high = architectural rewrite)
- **Files**: which source files will be modified
- **Brief**: 2-3 paragraph investigation brief for the worker agent

Rank by estimated ROI × probability of success. Select the top 2-3 candidates for dispatch.

### Step 3: BUILD BASELINE

Build a release/optimized binary from the current state and copy it to `/tmp/` for A/B comparison:

```bash
# Discover and run the project's release build command
# Copy the benchmark-relevant binary to /tmp/<project>_baseline_<binary-name>
```

The specific build command depends on the project (discovered in Step 0). Ensure deterministic build flags where possible (e.g., `CARGO_INCREMENTAL=0` for Rust).

Verify the baseline binary produces expected benchmark output.

### Step 4: CREATE WORKTREES

For each selected candidate:
```bash
git worktree add ../<project-dir>-opt-<slug> -b opt/<slug>
```

### Step 5: DISPATCH WORKERS

Spawn 2-3 worker agents in parallel using the Task tool:

```
Task(
  subagent_type: "general-purpose",
  mode: "bypassPermissions",
  prompt: <worker prompt — see below>,
  run_in_background: true
)
```

**Worker Prompt Construction:**

1. Read `~/.claude/agents/optimizer-worker.md` — include its FULL contents in the prompt
2. Add the project context block (from Step 0 discovery)
3. Add the assignment:

```
## Your Assignment

WORKTREE_PATH: <absolute path to worktree>
BASELINE_BIN: <absolute path to baseline binary in /tmp/>
PROJECT_ROOT: <absolute path to main repo>
BUILD_CMD: <the release build command for this project>
BENCH_CMD: <command to run the primary benchmark, producing timing output>
SECONDARY_BENCH_CMDS: <commands for secondary workload regression checks>
TEST_CMD: <command to run the test suite>
LINT_CMD: <command to run linting/clippy>
FMT_CMD: <command to run formatting>

## Investigation Brief

<paste the candidate brief>
```

### Step 6: COLLECT RESULTS

Wait for all workers to complete. Read their output files.

Parse each worker's report for:
- Verdict (KEEP/DISCARD)
- Primary workload U statistic and improvement percentage
- Secondary workload results
- Files changed
- Tests added/modified
- Insights discovered
- Failure details (if DISCARD)

### Step 7: REVIEW AND LEARN

For each worker result, evaluate:

1. **Did the worker follow the protocol correctly?** (tests, lint, measurement, report format)
2. **Were there recurring problems?** (workers struggling with a specific API, test framework issues, measurement problems)
3. **Did any worker discover something surprising?** (unexpected hotspot, architectural insight)
4. **Code quality issues in KEEP results?** (dead code introduced, redundant paths, missing error handling)

**Self-improvement**: If you identify a pattern of worker failures or protocol gaps:
- Update `~/.claude/agents/optimizer-worker.md` with clarifications or additional guidance
- Update workflow files if the measurement or dispatch process needs refinement
- Update reference docs if conventions need correction
- Log what you changed and why in the investigation report

**CRITICAL: When updating skill/agent/reference files, NEVER write project-specific information.** These files are shared across all projects. No project names, binary names, case IDs, specific file paths, or project-specific commands. Only generic protocol improvements, better phrasing, and additional discovery patterns. See SKILL.md "Skill Files Must Remain Project-Agnostic" section.

### Step 8: MERGE, CLEAN, OR DISCARD

For each worker:

**If KEEP (U >= 73, tests pass, no regression):**

1. **Merge the worktree branch:**
```bash
cd <main-repo>
git merge opt/<slug> --no-edit
```
If merge conflicts: attempt resolution. If non-trivial, discard instead.

2. **Post-merge cleanup** — the merged code must be cleaner than what the worker produced:
```bash
# Run formatter (project-specific: cargo fmt, prettier, black, gofmt, etc.)
<FMT_CMD>

# Run linter with auto-fix where possible
<LINT_FIX_CMD>  # e.g., cargo clippy --fix --allow-dirty --all-targets

# Fix ALL remaining warnings manually — including pre-existing ones
<LINT_CMD>  # Review output, fix everything

# Remove dead code introduced by the optimization
# Review git diff for unused imports, unreachable branches, commented-out code
```

3. **Review the diff** — read `git diff HEAD~1` and fix:
   - Dead code (unused functions, variables, imports)
   - Redundant code paths
   - Missing or incorrect comments
   - Style inconsistencies with the rest of the codebase
   - Any code that wouldn't belong in an ideal version of these files

4. **Run full test suite again** after cleanup to confirm nothing broke:
```bash
<TEST_CMD>
```
If tests fail after cleanup, the cleanup introduced a bug — fix it.

5. **Commit cleanup as a separate commit:**
```bash
git add -A
git commit -m "Post-merge cleanup: fmt, lint, dead code removal"
```

6. **Rebuild baseline** for next round:
```bash
<BUILD_CMD>
cp <binary> /tmp/<project>_baseline_<binary>
```

7. **Clean up worktree:**
```bash
git worktree remove ../<project>-opt-<slug>
git branch -d opt/<slug>
```

**If DISCARD:**
```bash
git worktree remove ../<project>-opt-<slug>
git branch -D opt/<slug>
```

### Step 9: REPORT

For ALL results (kept and discarded):

1. Write investigation report following the template in `~/.claude/skills/optimize/references/investigation-template.md`
   - Place in the project's investigation reports directory (discovered in Step 0)
   - If no such directory exists, create one (e.g., `docs/perf_investigations/`)
2. Update the performance backlog (if one exists) following conventions in `~/.claude/skills/optimize/references/backlog-conventions.md`
3. Commit the reports

### Step 10: LOOP

Return to Step 1 with the updated baseline. The survey will see the new reports and updated backlog, informing the next round of candidate generation.

## Concurrency Limits

- Maximum 3 worker agents at a time (leaves headroom for sub-agents within the 6-agent limit)
- If a worker seems stuck (no output after 10 minutes), check on it before spawning more
- Sequential merge: merge one winning worktree at a time, running cleanup and rebuilding baseline between merges

## Error Recovery

- **Worker crashes**: Read whatever output exists, report as DISCARD, clean up worktree
- **Merge conflict**: Attempt resolution. If complex, discard and note in report.
- **Baseline build failure**: Fix before continuing. Do not dispatch workers against a broken baseline.
- **All candidates discarded**: This is normal. Generate new candidates and continue. If 3 consecutive rounds produce no winners, pause and report to the user — the low-hanging fruit may be exhausted.

## Stopping Conditions

The loop runs until:
- User interrupts (Ctrl+C)
- 3 consecutive rounds with no winners (pause and report to user)
- A critical error that cannot be auto-recovered
