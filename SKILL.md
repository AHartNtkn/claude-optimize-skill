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

## Worker Agent

Workers are spawned as `general-purpose` Task agents. Their specification is at:
```
~/.claude/agents/optimizer-worker.md
```

Read this file and include its contents in each worker's prompt (workers cannot access global config).

## Key Principles

1. **Worktree isolation**: Each worker gets `../<project>-opt-<slug>`. Failed attempts are deleted cleanly via `git worktree remove`.
2. **Statistical rigor**: Mann-Whitney U test with N=10 interleaved runs. U >= 73 (p < 0.05) required to KEEP.
3. **Regression checking**: Primary workload improvement AND no regression on secondary workloads.
4. **Ambitious changes over micro-optimizations**: Architectural redesigns, algorithmic changes, and speculative rewrites are ALWAYS preferred over safe data-layout tweaks, struct shrinking, or enum rearranging. **When the profile shows that structural overhead has been eliminated and the remaining time is dominated by actual computation, the next candidates MUST be algorithmic or architectural — not more micro-optimizations.** A risky candidate with 10% estimated ROI is more valuable than a safe candidate with 2% ROI. Worktrees exist precisely to make risky changes cheap to attempt and discard. If you find yourself proposing only data-layout or allocation changes for 2+ consecutive rounds, STOP and force yourself to generate architectural candidates instead.
5. **Self-improvement**: After each round, review worker outcomes for protocol gaps or recurring problems. Update the worker agent spec, workflow docs, or reference files to prevent future issues.
6. **Post-merge cleanup**: After merging a winner, run fmt + lint fix + remove dead code before committing. The merged code must be cleaner than what the worker produced.
7. **Concurrent limits**: Maximum 3 workers at a time (respects 6-agent system limit).
8. **Project-agnostic**: Discovers build commands, benchmark commands, test commands, and project conventions at runtime. Never hardcodes project-specific paths or tools.

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
