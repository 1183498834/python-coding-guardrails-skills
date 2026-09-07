# Python Coding Guardrails

> A universal Agent Skill: mandatory quality guardrails for AI-assisted Python coding
>
> Works with **Claude Code · Codex · Doubao · Trae** and any AI coding tool that follows the [Agent Skills open standard](https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills)
>
> English | [简体中文](README.md)

A reusable Agent Skill that applies a set of engineering conventions — **without changing program behavior** — whenever AI writes, optimizes, or refactors Python scripts: CLI/path handling, concurrency speedups, memory management, faiss for large-scale retrieval, and error-prevention checks.

**Universal**: the Skill itself (`SKILL.md` + `references/`) is not tied to any specific platform. Any tool that follows the Agent Skills open standard (`SKILL.md` + YAML frontmatter) can use it directly — **the same folder, no modifications needed**. Installation locations for each platform are listed below.

## Core Rules

| # | Domain | Mandatory Requirement |
| --- | --- | --- |
| 1 | **CLI & Paths** | All I/O paths parsed via `argparse` with a `default=` value; use `pathlib`; auto-create output directories; explicit `utf-8` encoding |
| 2 | **Concurrency** | Parallelize first without changing logic: `ThreadPoolExecutor` for I/O-bound, `ProcessPoolExecutor` for CPU-bound; `map()` to preserve order; dynamic `max_workers` |
| 3 | **Memory** | Use `with` for files/images; release image caches with `del` + `gc.collect()` after each one; stream large files; generators instead of full lists |
| 4 | **faiss Retrieval** | Use faiss indexes for large-scale lookup/comparison (never brute-force loops); `float32` + L2 normalization; index by scale; batched `add`/`search`; persist and release indexes |
| 5 | **Error Prevention** | Validate inputs first; catch specific exceptions (no bare `except:`); release resources in `finally`; never hardcode paths/secrets; atomic writes; minimal test cases before delivery |
| 6 | **Testing** | Immediately run **10 samples** from the default path after writing code; **pass means done (no full run first)**; if no default path is given, **ask the user** — never fabricate or skip |

## Directory Structure

```
python-coding-guardrails/
├── SKILL.md                  # Trigger description + mandatory checklist (checked before writing code)
├── README.md                 # This document (Chinese, for display only — not part of the Skill)
├── README.en.md              # This document (English)
└── references/               # Topic-specific rules, loaded on demand by AI
    ├── cli-paths.md          # argparse & path conventions, code examples
    ├── concurrency.md        # Thread/process speedups, order preservation, thread safety
    ├── memory.md             # Memory management, image cache release, streaming
    ├── faiss.md              # faiss indexing, batched search, index persistence
    ├── error-safety.md       # Error-prevention checklist, exception rules, atomic writes
    └── sampling-test.md      # 10-sample smoke testing: sampling, running, asking the user for a path
```

## Installation

> Generic steps: copy the whole `python-coding-guardrails` folder into the skill directory of your tool. Take effect in a **new session**; tools auto-discover the Skill via the `description` in `SKILL.md`.

| Platform | Global path (all projects) | Project path (current project only) |
| --- | --- | --- |
| **Claude Code** | `~/.claude/skills/python-coding-guardrails` | `<project-root>/.claude/skills/python-coding-guardrails` |
| **Codex** | `~/.codex/skills/python-coding-guardrails` | `<project-root>/.agents/skills/python-coding-guardrails` (newer versions; may vary) |
| **Doubao** | `<workspace>/.user_skills/python-coding-guardrails` | — |
| **Trae** | `~/.trae/skills/python-coding-guardrails` | `<project-root>/.trae/skills/python-coding-guardrails` |

### Claude Code

```bash
# Option 1: global install (all projects)
mkdir -p ~/.claude/skills
git clone https://github.com/1183498834/python-coding-guardrails-skills.git
cp -r python-coding-guardrails-skills ~/.claude/skills/python-coding-guardrails

# Option 2: project-level install
cp -r python-coding-guardrails-skills <your-project>/.claude/skills/python-coding-guardrails

# Option 3: just ask Claude Code in a conversation:
#   "Install this skill: https://github.com/1183498834/python-coding-guardrails-skills"
```

### Codex

```bash
# Option 1: global install
mkdir -p ~/.codex/skills
cp -r python-coding-guardrails-skills ~/.codex/skills/python-coding-guardrails

# Option 2: project-level install (newer Codex uses .agents/skills)
cp -r python-coding-guardrails-skills <your-project>/.agents/skills/python-coding-guardrails

# Option 3: one-command install with skills CLI
npx skills add https://github.com/1183498834/python-coding-guardrails-skills --skill python-coding-guardrails

# Option 4: just ask Codex in a conversation:
#   "Install this skill: https://github.com/1183498834/python-coding-guardrails-skills"
```

> Codex's skill directory has changed across versions (`.codex/skills` vs `.agents/skills`); after installing, confirm the actual path with `codex --help` or the official docs.

### Doubao

Copy the folder into the user skills directory of your Doubao workspace:

```
<workspace>/.user_skills/python-coding-guardrails
```

### Trae

Copy the folder to any of the following locations:

| Scope | Path | Applies to |
| --- | --- | --- |
| Project | `<project-root>/.trae/skills/python-coding-guardrails` | Current project only |
| User | `~/.trae/skills/python-coding-guardrails` | All projects |
| Trae CLI | `~/.traecli/skills/python-coding-guardrails` | CLI global |

## Usage

No manual invocation needed. The Skill activates automatically when your request involves:

- Writing / optimizing / refactoring Python scripts or batch tools
- Command-line arguments and path handling (argparse)
- Program speedups, multi-threading / multi-processing
- Batch image/large-file processing, memory optimization
- Large-scale data lookup or similarity comparison (faiss)
- Explicit mentions of "robustness / error prevention / resource release"

AI first walks through the mandatory checklist in `SKILL.md`, then reads the matching `references/` rule file for the task.

## Rules Overview

| File | Highlights |
| --- | --- |
| `references/cli-paths.md` | argparse with defaults, pathlib usage, output dir creation, Windows encoding & Chinese paths, anti-patterns |
| `references/concurrency.md` | Bottleneck detection, `map()` vs `as_completed()`, worker limits, thread safety, progress & error visibility |
| `references/memory.md` | Context managers, per-image `del` + `gc.collect()`, streaming reads, generators, concurrency memory caps |
| `references/faiss.md` | When to use, `float32` + normalization, `IndexFlatIP`/`IndexIVFFlat` selection, batched queries, persistence & release |
| `references/error-safety.md` | Input validation, exception rules, no hardcoding, atomic writes, observability, minimal verification |
| `references/sampling-test.md` | Test-immediately (10-sample smoke test, pass means done, no full run), even sampling head/mid/tail, run & assertions, ask user for path, re-run all after fixes |

## Validation

Run the official validator from [skill-creator-for-work](https://github.com/anthropics/skills):

```bash
python scripts/quick_validate.py python-coding-guardrails
```

## Links

- [Anthropic Agent Skills](https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills)
- [Claude Code Skills docs](https://docs.anthropic.com/en/docs/claude-code/slash-commands)
- [Trae Skills Docs](https://docs.trae.cn/ide_skills)
