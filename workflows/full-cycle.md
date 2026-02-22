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

**Source 5 — Simplification and consolidation:** Large refactors that greatly simplify the codebase are **high-priority candidates in every round** — not just periodic cleanup. A refactor that eliminates hundreds of lines of duplication, unifies parallel code paths, or removes an unnecessary abstraction layer is more valuable than a micro-optimization because it reduces the surface area for all future work. **When you identify a large simplifying refactor during survey, include it as a candidate alongside performance candidates and rank it by code-reduction impact.** Additionally, every ~5th round should include at least one consolidation candidate to catch accumulated cruft. **Simplification workers should be multi-target, not single-focus.** A single simplification worker can tackle multiple unrelated cleanup targets across different files in one investigation — dead code removal, duplication elimination, API consolidation, etc. The only acceptance criterion is no performance regression (U > 27). There is no reason to limit a simplification worker to one focused area when multiple targets exist. Consolidation targets include:
- Redundant code paths that can be unified (e.g., multiple functions doing the same thing with slight variations)
- Dead code accumulated from prior optimization rounds
- Abstractions that have become unnecessarily complex through incremental changes
- Duplicated logic that should be factored into shared helpers
- Consolidation candidates use a relaxed acceptance threshold: they are KEPT as long as they do not cause a statistically significant regression (U > 27 on all workloads). They do NOT need U >= 73. Neutral performance (27 < U < 73) is perfectly acceptable for consolidation — the value is in code simplicity, not speed.

For each candidate, provide:
- **Slug**: snake_case identifier (e.g., `lockfree_intern`)
- **Title**: one-line description
- **Hypothesis**: expected mechanism of improvement
- **Estimated ROI**: rough percentage based on profiling data
- **Risk**: low/medium/high (high = architectural rewrite)
- **Files**: which source files will be modified
- **Brief**: 2-3 paragraph investigation brief for the worker agent

**Ranking: Breadth first, then ambition.** Before ranking, check: how many Major Proposals or backlog categories remain completely uninvestigated? If there are uninvestigated proposals, at least one candidate MUST come from a fresh, never-tried proposal — not a follow-up to a previous investigation. **When an approach fails (DISCARD), do NOT generate a variation of the same approach as the next candidate.** Move to a completely different proposal that attacks the problem from a different angle. Each Major Proposal is a distinct tree of investigation; when one tree stops bearing fruit, move to the next tree.

Within that constraint, rank by estimated ROI. Architectural redesigns, algorithmic improvements, and speculative rewrites are ALWAYS preferred over safe micro-optimizations. Failed attempts in worktrees cost nothing. Only deprioritize a high-ROI candidate if it is fundamentally unmeasurable or would take an impractical amount of worker time.

**NEVER leave a worker slot empty because you've "run out of ideas" on the current approach.** If you can't think of a candidate, you are looking too narrowly. Read the full backlog, read uninvestigated Major Proposals, brainstorm from a completely different angle. A speculative long-shot from an untried proposal is always better than an empty slot.

Select the top 2 candidates for dispatch.

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

### Step 5: DISPATCH AND COLLECT (Continuous Pipeline)

Workers are managed via the **agent-teams system**. NEVER use standalone `Task` calls with `run_in_background: true`.

**Team setup (first round only):**
```
TeamCreate(team_name: "optimize", description: "Performance optimization workers")
```

**Continuous pipeline — ALWAYS maintain 2 active workers:**

**THIS IS NON-NEGOTIABLE: You MUST have 2 workers running at all times until the session ends or the user interrupts.** When a worker finishes, immediately spawn a replacement. When a round's candidates are exhausted, generate the next round's candidates and keep spawning. The only acceptable reasons for fewer than 2 active workers are: (a) the session just started and you haven't spawned them yet, (b) you are in the process of generating the next round's candidates (this should take minutes, not idle time), or (c) the user interrupted. **A DISCARD result is not a reason to slow down — it is normal and expected. Most investigations will be discarded. That is the point of worktree isolation.** Never pause, hesitate, or "reflect" before spawning the next worker. Process the result, clean up, spawn replacement, move on.

1. Spawn workers for the first 2 candidates (or fewer if fewer candidates exist)
2. When ANY worker completes and sends its result via `SendMessage`:
   a. **Parse the `## OPTIMIZATION RESULT` block** from the message for:
      - Verdict (KEEP/DISCARD)
      - Primary workload U statistic and improvement percentage
      - Secondary workload results (if applicable)
      - Files changed
      - Insights discovered
      - Failure details (if DISCARD)
   b. **Process immediately**: write the investigation report and update the backlog
   c. **Merge or discard** the worktree (Steps 6-8)
   d. **Spawn a new worker** for the next unstarted candidate (if any remain). If no candidates remain for the current round, **immediately start Step 1 of the next round** (survey + generate candidates) and spawn workers for the new candidates.
   e. Send `shutdown_request` to the finished worker
3. Repeat until all candidates for this round have been processed, then **immediately begin the next round**

The orchestrator should never be idle waiting for all workers to finish before acting. Each result is processed the moment it arrives, and the freed slot is immediately filled. **There is no "pause between rounds" — when the last candidate of round N finishes, round N+1 candidates should already be dispatched or dispatching.**

**Spawning a worker:**

For each candidate, create a task and spawn a teammate:

```
TaskCreate(subject: "<slug>: <title>", description: "<investigation brief + assignment details>")
```

```
Task(
  subagent_type: "general-purpose",
  mode: "bypassPermissions",
  team_name: "optimize",
  name: "worker-<slug>",
  prompt: <worker prompt — see below>
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

Workers report results via `SendMessage` back to the orchestrator. The orchestrator receives only the compact structured result, not the full worker transcript.

### Step 6: REVIEW AND LEARN

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

### Step 7: MERGE, CLEAN, OR DISCARD

For each worker:

**If KEEP (U >= 73, tests pass, no regression) OR consolidation candidate (U > 27, tests pass, no regression):**

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

### Step 8: REPORT

For ALL results (kept and discarded):

1. Write investigation report following the template in `~/.claude/skills/optimize/references/investigation-template.md`
   - Place in the project's investigation reports directory (discovered in Step 0)
   - If no such directory exists, create one (e.g., `docs/perf_investigations/`)
2. Update the performance backlog (if one exists) following conventions in `~/.claude/skills/optimize/references/backlog-conventions.md`
3. Commit the reports

### Step 9: LOOP

Return to Step 1 with the updated baseline. The survey will see the new reports and updated backlog, informing the next round of candidate generation.

## Concurrency Limits

- **ALWAYS 2 worker teammates running** (leaves headroom within the agent limit). The only time you should have fewer than 2 is during the brief window between a worker finishing and its replacement spawning, or during initial startup.
- If a worker goes idle without sending a result message, send it a check-in message before spawning more
- Sequential merge: merge one winning worktree at a time, running cleanup and rebuilding baseline between merges
- Reuse the team across rounds — only create it once per session. Clean up with `TeamDelete` when the session ends.

## Error Recovery

- **Worker crashes**: Read whatever output exists, report as DISCARD, clean up worktree, **spawn a replacement immediately**
- **Merge conflict**: Attempt resolution. If complex, discard and note in report.
- **Baseline build failure**: Fix before continuing. Do not dispatch workers against a broken baseline.
- **All candidates in a round discarded**: **This is completely normal and expected.** Most optimization attempts fail — that is the entire point of worktree isolation. A round where all candidates are discarded is not a signal to slow down, pause, or reconsider. It means you generate the next round's candidates and keep going. **Generate new candidates immediately and continue.**
- **Multiple consecutive rounds with no winners**: Still normal. Keep going. Performance optimization is a numbers game — you try many things and most fail. The successes compound.

## Stopping Conditions

**The ONLY conditions that stop the loop:**
- User interrupts (Ctrl+C)
- **10 consecutive ROUNDS** (not individual candidates) with zero winners across ALL candidates in those rounds. This means roughly 20+ consecutive DISCARD results before you even consider pausing. Until you hit that threshold, you keep generating candidates and dispatching workers without hesitation.
- A critical error that cannot be auto-recovered (build system broken, git corruption, etc.)

**To be absolutely clear:** 5 consecutive DISCARDs is not a reason to stop. 10 consecutive DISCARDs is not a reason to stop. 20 consecutive DISCARDs is not a reason to stop. You stop at 10 consecutive ROUNDS (each containing 2 candidates) with no winners — that's 20+ consecutive DISCARDs. Until then, the loop continues unconditionally. Every DISCARD teaches you something that informs the next round's candidates.
