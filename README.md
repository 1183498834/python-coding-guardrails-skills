# Python Coding Guardrails

> AI 写代码时的强制质量护栏 Skill · 适用于豆包（Doubao）与 Trae
>
> [English](README.en.md) | 简体中文

一个可复用的 Agent Skill：当 AI 编写、优化或重构 Python 脚本时，自动应用一套"不改变程序逻辑"的工程规范——路径参数、并发提速、内存管理、faiss 大规模检索、防错检查。

豆包与 Trae 均遵循 [Agent Skills 开放规范](https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills)（`SKILL.md` + YAML frontmatter），因此**同一份 Skill 文件夹两端通用**，无需为每个平台各写一份。

## 核心规则

| # | 规则域 | 强制要求 |
| --- | --- | --- |
| 1 | **CLI 与路径** | 输入输出路径一律 `argparse` 解析并指定 `default=` 默认路径；用 `pathlib`；输出目录自动创建；显式 `utf-8` 编码 |
| 2 | **并发提速** | 保持逻辑不变前提下优先并行化：IO 密集用 `ThreadPoolExecutor`，CPU 密集用 `ProcessPoolExecutor`；结果保序用 `map()`；`max_workers` 动态计算 |
| 3 | **内存管理** | 文件/图片用 `with` 上下文；图片逐张处理完 `del` + `gc.collect()` 释放缓存；大文件流式读取；生成器代替全量列表 |
| 4 | **faiss 检索** | 大规模查找/比对必须用 faiss 建索引（禁暴力循环）；向量 `float32` + 归一化；按规模选索引；批量 `add`/`search`；索引持久化与释放 |
| 5 | **防错** | 输入先校验；精确异常捕获（禁裸 `except:`）；`finally` 释放资源；不硬编码路径/密钥；原子写入防半截文件；交付前最小用例验证 |

## 目录结构

```
python-coding-guardrails/
├── SKILL.md                  # 触发描述 + 强制检查清单（AI 写代码前逐条核对）
├── README.md                 # 本文档（中文，仅供 GitHub 展示，非技能组成部分）
├── README.en.md              # 本文档（英文）
└── references/               # 按主题拆分的详细规则，AI 按需加载
    ├── cli-paths.md          # argparse 与路径规范、代码示例
    ├── concurrency.md        # 多线程/多进程提速、保序、线程安全
    ├── memory.md             # 内存管理、图片缓存释放、流式读取
    ├── faiss.md              # faiss 建索引、批量检索、索引持久化
    └── error-safety.md       # 防错检查清单、异常规范、原子写入
```

## 安装与部署

> 部署后**新开会话**生效；两端 AI 会通过 `SKILL.md` 的 `description` 自动发现并触发。

### 豆包（Doubao）

将 `python-coding-guardrails` 文件夹复制到豆包工作区的用户技能目录：

```
<workspace>/.user_skills/python-coding-guardrails
```

### Trae

将文件夹复制到以下任一位置：

| 范围 | 路径 | 生效范围 |
| --- | --- | --- |
| 项目级 | `<项目根>/.trae/skills/python-coding-guardrails` | 仅当前项目 |
| 用户级 | `~/.trae/skills/python-coding-guardrails` | 所有项目 |
| Trae CLI | `~/.traecli/skills/python-coding-guardrails` | CLI 全局 |

## 使用方式

无需手动调用。当你的指令涉及以下任一场景时，Skill 自动上场：

- 编写/优化/重构 Python 脚本、批处理工具
- 命令行参数与路径处理（argparse）
- 程序提速、多线程/多进程并行
- 图片/大文件批处理、内存占用优化
- 大规模数据查找、相似度比对（faiss）
- 明确提到"防止出错 / 健壮性 / 资源释放"

AI 会先逐条核对 `SKILL.md` 中的强制检查清单，再按任务主题读取对应的 `references/` 规则文件。

## 规则详览

| 文件 | 内容要点 |
| --- | --- |
| `references/cli-paths.md` | argparse 必带默认路径、pathlib 用法、输出目录创建、Windows 编码与中文路径、反例清单 |
| `references/concurrency.md` | 瓶颈类型判断、`map()` 保序 vs `as_completed()`、线程数与资源上限、线程安全、进度与错误可见 |
| `references/memory.md` | 上下文管理器、逐张处理 + `del` + `gc.collect()`、流式读取、生成器、并发内存峰值控制 |
| `references/faiss.md` | 使用阈值、`float32` 与归一化、`IndexFlatIP`/`IndexIVFFlat` 选型、批量查询、索引持久化与释放 |
| `references/error-safety.md` | 输入校验、异常处理规范、不硬编码、原子写入、日志可观测、最小验证清单 |

## 验证

使用 [skill-creator-for-work](https://github.com/anthropics/skills) 的官方校验脚本：

```bash
python scripts/quick_validate.py python-coding-guardrails
```

## 相关链接

- [Anthropic Agent Skills 规范](https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills)
- [Trae 技能文档](https://docs.trae.cn/ide_skills)
