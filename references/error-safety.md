# 防错检查清单

写任何代码（尤其是批处理、文件 IO、并发、检索类脚本）前，逐条过以下防线。每条都是强制项，不是建议。

## 1. 输入校验（先校验，后使用）

- 路径参数：检查存在性与类型（文件还是目录），不存在时给出明确错误而不是 `FileNotFoundError` 裸崩。
- 空输入：空目录、空列表、空文件都要有明确行为（跳过并提示，而不是静默成功或下标越界）。
- 类型与数量：函数入口校验关键参数类型（可用类型注解 + `isinstance`），批处理校验数量上限。

```python
from pathlib import Path

def load_input(input_dir: Path) -> list:
    input_dir = Path(input_dir)
    if not input_dir.exists():
        raise ValueError(f"输入目录不存在: {input_dir}")
    files = sorted(input_dir.glob("*.jpg"))
    if not files:
        print("警告: 输入目录中没有 .jpg 文件，直接返回空结果")
    return files
```

## 2. 异常处理规范

- **禁止裸 `except:`**，禁止 `except Exception: pass` 吞异常。
- 精确捕获可预期的异常（`FileNotFoundError` / `ValueError` / `KeyError`），其余允许冒泡。
- 错误信息必须带上下文：文件名、批次号、出错步骤。
- 需要释放资源时用 `finally` 或 `with`，保证异常路径也能释放。

```python
try:
    with Image.open(p) as img:
        img.convert("RGB").save(out)
except FileNotFoundError:
    print(f"文件不存在，跳过: {p}")
except Exception as e:
    print(f"处理失败 {p}: {type(e).__name__}: {e}")
    raise          # 未知错误向上抛，不静默
```

## 3. 不硬编码

- 路径、密钥、端口、超参数一律通过 argparse / 配置文件 / 环境变量传入。
- 代码中出现 `/Users/xxx/...`、`password=...`、魔法数字即视为缺陷。
- 环境变量读取：`os.environ.get("KEY", default)`，缺失时给出提示。

## 4. 原子写入与幂等

- 输出文件先写临时文件再 `os.replace()` 重命名，避免程序中断留下半截文件：

```python
import os, tempfile

def atomic_write(path, text):
    path = Path(path)
    path.parent.mkdir(parents=True, exist_ok=True)
    fd, tmp = tempfile.mkstemp(dir=path.parent, suffix=".tmp")
    try:
        with os.fdopen(fd, "w", encoding="utf-8") as f:
            f.write(text)
        os.replace(tmp, path)
    except Exception:
        os.unlink(tmp)
        raise
```

- 输出文件已存在时明确策略（覆盖/跳过/报错），由参数控制，不静默覆盖用户数据。

## 5. 编码与换行

- 所有文本读写显式 `encoding="utf-8"`。
- 追加写注意换行；CSV 用 `newline=""` 避免空行（Windows）。

## 6. 日志与可观测

- 长任务记录进度与里程碑：开始、每 N 条、结束、失败数。
- 使用 `logging` 或 `print` + `tqdm`；错误统一汇总到结尾输出，便于排查。

## 7. 并发与资源的边界

- 线程/进程数、批量大小、队列深度都设上限（见 `concurrency.md`）。
- 显式关闭/释放：文件句柄、faiss 索引、模型、连接（见 `memory.md`）。

## 8. 改动最小化

- 优化/重构时不改原有函数签名、输出格式、返回结构，除非用户明确要求。
- 新增行为（如跳过坏文件）用注释说明，并让用户可见（日志输出）。

## 9. 提交前最小验证

- 必测：空输入、单条输入、正常输入、异常输入（坏文件/缺失路径）。
- 并发版本额外测：重复运行结果一致（幂等）、无资源泄漏（跑一遍观察内存回落）。
