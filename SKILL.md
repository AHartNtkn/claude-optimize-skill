# /optimize — Autonomous Performance Optimization

Autonomous performance optimization system for any codebase. Generates optimization candidates, dispatches parallel workers to isolated git worktrees, measures via Mann-Whitney U test, merges winners, discards losers, cleans up code quality, and loops.

Works with any language/build system. Discovers project structure, benchmarks, and conventions at runtime.

## Invocation

- `/optimize` — Run the full autonomous optimization loop (default)
- `/optimize pick` — Generate and rank candidates only (no workers)
- `/optimize measure` — A/B measure current working tree vs baseline
- `/optimize report <slug>` — Write investigation report and update backlog

## Routing

Parse the argument after `/optimize`:

- No argument or `full` → Load and follow `workflows/full-cycle.md`
- `pick` → Load and follow `workflows/pick-target.md`
- `measure` → Load and follow `workflows/measure-compare.md`
- `report` → Load and follow `workflows/write-report.md`

## Reference Material

Before starting any workflow, read the relevant reference docs:

- `references/measurement-protocol.md` — Mann-Whitney U test details
- `references/investigation-template.md` — Report format
- `references/backlog-conventions.md` — Backlog annotation conventions

## Worker Agents and Team System

Workers are managed via the **agent-teams system** (TeamCreate, TaskCreate, SendMessage), NOT via standalone Task subagent calls. The worker agent specification is at:
```
~/.claude/agents/optimizer-worker.md
```

**Team lifecycle per round:**
1. Create a team via `TeamCreate` at the start of the optimization session (reuse across rounds)
2. Create tasks via `TaskCreate` for each optimization candidate
3. Spawn workers as teammates via `Task` with `team_name` parameter
4. Workers claim tasks, execute them, and report results back via `SendMessage`
5. Orchestrator receives compact result messages, writes reports, updates backlog
6. Send `shutdown_request` to workers when the round is complete
7. Clean up the team via `TeamDelete` when the session ends

## Key Principles

1. **Worktree isolation**: Each worker gets `../<project>-opt-<slug>`. Failed attempts are deleted cleanly via `git worktree remove`.
2. **Statistical rigor**: Mann-Whitney U test with N=10 interleaved runs. U >= 73 (p < 0.05) required to KEEP.
3. **Regression checking**: Primary workload improvement AND no regression on secondary workloads.
4. **Ambitious changes over micro-optimizations**: Architectural redesigns, algorithmic changes, and speculative rewrites are ALWAYS preferred over safe data-layout tweaks, struct shrinking, or enum rearranging. **When the profile shows that structural overhead has been eliminated and the remaining time is dominated by actual computation, the next candidates MUST be algorithmic or architectural — not more micro-optimizations.** A risky candidate with 10% estimated ROI is more valuable than a safe candidate with 2% ROI. Worktrees exist precisely to make risky changes cheap to attempt and discard. If you find yourself proposing only data-layout or allocation changes for 2+ consecutive rounds, STOP and force yourself to generate architectural candidates instead.
5. **Self-improvement**: After each round, review worker outcomes for protocol gaps or recurring problems. Update the worker agent spec, workflow docs, or reference files to prevent future issues.
6. **Post-merge cleanup**: After merging a winner, run fmt + lint fix + remove dead code before committing. The merged code must be cleaner than what the worker produced.
7. **Concurrent limits — ALWAYS 2 workers**: You MUST maintain 2 active workers at all times. When a worker finishes (KEEP or DISCARD), spawn its replacement immediately. A DISCARD is not a reason to pause or slow down — it is the expected outcome for most investigations. The only time fewer than 2 workers should be running is during initial startup or the brief gap between a worker finishing and its replacement spawning. **Never let a worker slot sit empty while you "think about" the next candidate. Generate candidates proactively so replacements are ready to dispatch the moment a slot opens.**
8. **Project-agnostic**: Discovers build commands, benchmark commands, test commands, and project conventions at runtime. Never hardcodes project-specific paths or tools.
9. **Simplification as a first-class optimization**: Large refactors that greatly simplify the code are high-priority candidates — not periodic afterthoughts. A refactor that eliminates 200 lines of duplication, unifies parallel code paths, or removes an entire abstraction layer is MORE valuable than a micro-optimization, because it reduces the surface area for future work and makes subsequent optimizations easier to implement and review. **When you identify a large simplifying refactor during survey or candidate generation, prioritize it alongside performance candidates — do not defer it to a "consolidation round."** Additionally, every ~5th round should still include at least one consolidation candidate to catch accumulated cruft. Consolidation candidates are accepted as long as they do not cause a statistically significant regression (U > 27 on all workloads). They do NOT need to demonstrate improvement (U >= 73) — neutral results are fine. The goal is codebase health, not speed. **Simplification workers should be multi-target, not single-focus.** A single simplification worker can tackle multiple unrelated cleanup targets across different files in one investigation — dead code removal here, duplication elimination there, API consolidation elsewhere. The only acceptance criterion is no performance regression (U > 27). There is no reason to limit a simplification worker to a single focused area when multiple targets exist.
10. **Breadth over depth — explore fresh directions, don't iterate on failures**: When generating candidates, prioritize UNINVESTIGATED proposals and approaches over follow-ups to things already tried. If an approach fails on a workload (e.g., "deeper indexing" fails after "root indexing" already partially worked), do NOT try yet another variation of indexing — move to a completely different proposal that attacks the problem from a different angle. **The backlog and Major Proposals list exists precisely for this: each proposal is a distinct tree of investigation. When one tree stops bearing fruit, move to the next tree — don't keep climbing the same one.** Concretely: before generating any follow-up candidate, check how many Major Proposals or backlog categories remain completely uninvestigated. If there are uninvestigated proposals, at least one candidate per round MUST come from a fresh proposal. Never convince yourself that "remaining targets are too hard" or "the design space is exhausted" — those phrases mean you've exhausted one approach, not all approaches. A worker slot should never sit empty because you've run out of ideas on the current approach.
11. **Context window hygiene**: Worker agents produce large transcripts (100K+ tokens). The agent-teams system solves this: workers send structured result messages via `SendMessage`, which are compact and controlled. **Rules:**
    - **Always use the team system** (TeamCreate + Task with `team_name`) for workers. NEVER use standalone `Task` calls with `run_in_background: true` — background task notifications dump full transcripts into the orchestrator's context, persist across compaction, and create a doom loop.
    - **Workers report via `SendMessage`**, sending ONLY the structured `## OPTIMIZATION RESULT` block. The orchestrator never sees the full worker transcript.
    - **Process results immediately**: When a worker message arrives, write the investigation report and update the backlog right away, then move on.
    - **Keep worker prompts compact**: Include only the worker agent spec, project context block, and investigation brief. Do not paste large file contents — tell workers to read files themselves.

## CRITICAL: Skill Files Must Remain Project-Agnostic

**When updating skill files during self-improvement (principle 5), you MUST NEVER write project-specific information into any file under `~/.claude/skills/optimize/` or `~/.claude/agents/optimizer-worker.md`.** These files are shared across ALL projects.

Banned content in skill/agent/reference files:
- Project names, binary names, case IDs, benchmark identifiers
- Specific file paths (e.g., `src/kernel/compose.rs`)
- Project-specific script names (e.g., `pinned_env.sh`)
- Language-specific commands presented as the only option (always provide discovery patterns)
- Any information that would only be true for one project

**What IS allowed in self-improvement updates:**
- Generic protocol improvements (e.g., "always check for merge conflicts before committing cleanup")
- Better phrasing of instructions that workers misunderstood
- Additional decision criteria or edge case handling
- New discovery patterns (e.g., "also check for `benchmarks/` directory")
- Corrections to the Mann-Whitney U protocol or report template

## Project Discovery (Step 0)

Before any workflow begins, discover the project:

1. **Read CLAUDE.md** (project root) — contains project conventions, test commands, build instructions
2. **Detect language/build system** — Cargo.toml (Rust), package.json (JS/TS), Makefile, CMakeLists.txt, go.mod, etc.
3. **Find benchmarks** — look for `benches/`, `benchmarks/`, criterion config, custom benchmark scripts, perf binaries
4. **Find performance documentation** — look for files like `PERFORMANCE_INVESTIGATIONS.md`, `PERFORMANCE.md`, `docs/perf*/`, `docs/performance/`
5. **Find existing investigation reports** — `docs/perf_investigations/`, `docs/benchmarks/`, similar
6. **Identify primary and secondary benchmark workloads** — the heaviest/most representative benchmark case(s) for primary measurement, and 2-3 diverse secondary cases for regression checking
7. **Identify profiling tools available** — `perf`, flamegraph scripts, `cargo flamegraph`, `py-spy`, custom profiling infrastructure

Store all discovered information in a project context block that gets passed to workers.
