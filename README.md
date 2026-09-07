# Python Coding Guardrails

> 通用 Agent Skill：AI 写代码时的强制质量护栏
>
> 适用于 **Claude Code · Codex · Doubao（豆包） · Trae** 及一切支持 [Agent Skills 开放规范](https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills) 的 AI 编程工具
>
> [English](README.en.md) | 简体中文

一个可复用的 Agent Skill：当 AI 编写、优化或重构 Python 脚本时，自动应用一套"不改变程序逻辑"的工程规范——路径参数、并发提速、内存管理、faiss 大规模检索、防错检查。

**通用性说明**：本 Skill 本体（`SKILL.md` + `references/`）不绑定任何具体平台，只要是遵循 Agent Skills 开放规范（`SKILL.md` + YAML frontmatter）的工具即可直接使用，**同一份文件夹无需修改**。下方给出各平台的安装位置。

## 核心规则

| # | 规则域 | 强制要求 |
| --- | --- | --- |
| 1 | **CLI 与路径** | 输入输出路径一律 `argparse` 解析并指定 `default=` 默认路径；用 `pathlib`；输出目录自动创建；显式 `utf-8` 编码 |
| 2 | **并发提速** | 保持逻辑不变前提下优先并行化：IO 密集用 `ThreadPoolExecutor`，CPU 密集用 `ProcessPoolExecutor`；结果保序用 `map()`；`max_workers` 动态计算 |
| 3 | **内存管理** | 文件/图片用 `with` 上下文；图片逐张处理完 `del` + `gc.collect()` 释放缓存；大文件流式读取；生成器代替全量列表 |
| 4 | **faiss 检索** | 大规模查找/比对必须用 faiss 建索引（禁暴力循环）；向量 `float32` + 归一化；按规模选索引；批量 `add`/`search`；索引持久化与释放 |
| 5 | **防错** | 输入先校验；精确异常捕获（禁裸 `except:`）；`finally` 释放资源；不硬编码路径/密钥；原子写入防半截文件；交付前最小用例验证 |
| 6 | **测试验证** | 写完代码后从默认路径选取 **10 个样本**实际运行验证正确性；未指定默认路径时**先询问使用者**测试路径，禁止编造路径或跳过测试 |

## 目录结构

```
python-coding-guardrails/
├── SKILL.md                  # 触发描述 + 强制检查清单（AI 写代码前逐条核对）
├── README.md                 # 本文档（中文，仅供展示，非技能组成部分）
├── README.en.md              # 本文档（英文）
└── references/               # 按主题拆分的详细规则，AI 按需加载
    ├── cli-paths.md          # argparse 与路径规范、代码示例
    ├── concurrency.md        # 多线程/多进程提速、保序、线程安全
    ├── memory.md             # 内存管理、图片缓存释放、流式读取
    ├── faiss.md              # faiss 建索引、批量检索、索引持久化
    ├── error-safety.md       # 防错检查清单、异常规范、原子写入
    └── sampling-test.md      # 10 样本冒烟测试：抽样、运行、询问用户路径
```

## 安装与部署

> 通用步骤：把整个 `python-coding-guardrails` 文件夹放到目标工具对应的技能目录即可。部署后**新开会话**生效；各工具会通过 `SKILL.md` 的 `description` 自动发现并触发。

| 平台 | 全局路径（所有项目） | 项目路径（仅当前项目） |
| --- | --- | --- |
| **Claude Code** | `~/.claude/skills/python-coding-guardrails` | `<项目根>/.claude/skills/python-coding-guardrails` |
| **Codex** | `~/.codex/skills/python-coding-guardrails` | `<项目根>/.agents/skills/python-coding-guardrails`（新版路径，可能随版本调整） |
| **Doubao（豆包）** | `<workspace>/.user_skills/python-coding-guardrails` | — |
| **Trae** | `~/.trae/skills/python-coding-guardrails` | `<项目根>/.trae/skills/python-coding-guardrails` |

### Claude Code

```bash
# 方式一：全局安装（所有项目可用）
mkdir -p ~/.claude/skills
git clone https://github.com/1183498834/python-coding-guardrails-skills.git
cp -r python-coding-guardrails-skills ~/.claude/skills/python-coding-guardrails

# 方式二：项目级安装
cp -r python-coding-guardrails-skills <你的项目>/.claude/skills/python-coding-guardrails

# 方式三：直接在 Claude Code 对话中说：
#   "安装这个 skill：https://github.com/1183498834/python-coding-guardrails-skills"
```

### Codex

```bash
# 方式一：全局安装
mkdir -p ~/.codex/skills
cp -r python-coding-guardrails-skills ~/.codex/skills/python-coding-guardrails

# 方式二：项目级安装（新版 Codex 用 .agents/skills）
cp -r python-coding-guardrails-skills <你的项目>/.agents/skills/python-coding-guardrails

# 方式三：用 skills CLI 一键安装
npx skills add https://github.com/1183498834/python-coding-guardrails-skills --skill python-coding-guardrails

# 方式四：直接在 Codex 对话中说：
#   "安装这个 skill：https://github.com/1183498834/python-coding-guardrails-skills"
```

> Codex 的技能目录在不同版本有调整（`.codex/skills` 与 `.agents/skills`），安装后可用 `codex --help` 或官方文档确认当前版本的实际路径。

### Doubao（豆包）

将文件夹复制到豆包工作区的用户技能目录：

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
| `references/sampling-test.md` | 10 样本抽样方法（均匀抽样覆盖头中尾）、运行断言、未指定路径时询问用户、失败修复后重跑全部 |

## 验证

使用 [skill-creator-for-work](https://github.com/anthropics/skills) 的官方校验脚本：

```bash
python scripts/quick_validate.py python-coding-guardrails
```

## 相关链接

- [Anthropic Agent Skills 规范](https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills)
- [Claude Code Skills 文档](https://docs.anthropic.com/en/docs/claude-code/slash-commands)
- [Trae 技能文档](https://docs.trae.cn/ide_skills)
