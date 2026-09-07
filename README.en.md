# Python Coding Guardrails

> A mandatory quality-guardrail Skill for AI-assisted Python coding · Works with Doubao and Trae
>
> English | [简体中文](README.md)

A reusable Agent Skill that applies a set of engineering conventions — **without changing program behavior** — whenever AI writes, optimizes, or refactors Python scripts: CLI/path handling, concurrency speedups, memory management, faiss for large-scale retrieval, and error-prevention checks.

Both Doubao and Trae follow the [Agent Skills open standard](https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills) (`SKILL.md` + YAML frontmatter), so **the same Skill folder works on both platforms** — no need to write separate versions.

## Core Rules

| # | Domain | Mandatory Requirement |
| --- | --- | --- |
| 1 | **CLI & Paths** | All I/O paths parsed via `argparse` with a `default=` value; use `pathlib`; auto-create output directories; explicit `utf-8` encoding |
| 2 | **Concurrency** | Parallelize first without changing logic: `ThreadPoolExecutor` for I/O-bound, `ProcessPoolExecutor` for CPU-bound; `map()` to preserve order; dynamic `max_workers` |
| 3 | **Memory** | Use `with` for files/images; release image caches with `del` + `gc.collect()` after each one; stream large files; generators instead of full lists |
| 4 | **faiss Retrieval** | Use faiss indexes for large-scale lookup/comparison (never brute-force loops); `float32` + L2 normalization; index by scale; batched `add`/`search`; persist and release indexes |
| 5 | **Error Prevention** | Validate inputs first; catch specific exceptions (no bare `except:`); release resources in `finally`; never hardcode paths/secrets; atomic writes; minimal test cases before delivery |

## Directory Structure

```
python-coding-guardrails/
├── SKILL.md                  # Trigger description + mandatory checklist (checked before writing code)
├── README.md                 # This document (Chinese, for GitHub display only — not part of the Skill)
├── README.en.md              # This document (English)
└── references/               # Topic-specific rules, loaded on demand by AI
    ├── cli-paths.md          # argparse & path conventions, code examples
    ├── concurrency.md        # Thread/process speedups, order preservation, thread safety
    ├── memory.md             # Memory management, image cache release, streaming
    ├── faiss.md              # faiss indexing, batched search, index persistence
    └── error-safety.md       # Error-prevention checklist, exception rules, atomic writes
```

## Installation

> Deploy the folder, then **start a new session** for it to take effect. Both platforms auto-discover the Skill via the `description` in `SKILL.md`.

### Doubao

Copy the `python-coding-guardrails` folder into the user skills directory of your Doubao workspace:

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

## Validation

Run the official validator from [skill-creator-for-work](https://github.com/anthropics/skills):

```bash
python scripts/quick_validate.py python-coding-guardrails
```

## Links

- [Anthropic Agent Skills](https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills)
- [Trae Skills Docs](https://docs.trae.cn/ide_skills)
