# CLI 与路径规范

本文件规定所有 Python 脚本的输入输出路径写法。规则适用于新建脚本、改写脚本、批量处理工具。

## 1. argparse 必带默认路径

- 所有输入输出参数通过 `argparse.ArgumentParser` 定义。
- **每个参数都必须指定 `default=`**，不允许无默认值的必填路径（除非用户明确要求必填）。
- 默认值优先取"相对当前工作目录"或"脚本所在目录"的合理路径，并写注释说明口径。

```python
import argparse
from pathlib import Path

def parse_args():
    p = argparse.ArgumentParser(description="批量处理图片")
    p.add_argument("--input-dir", type=Path, default=Path("data/input"),
                   help="输入图片目录（默认: data/input）")
    p.add_argument("--output-dir", type=Path, default=Path("data/output"),
                   help="输出目录（默认: data/output）")
    p.add_argument("--batch-size", type=int, default=64,
                   help="每批处理数量（默认: 64）")
    return p.parse_args()
```

- 参数命名统一：`--input` / `--input-dir` / `--output` / `--output-dir` / `--config`。
- 类型用 `type=Path` 而不是 `type=str`，后续直接用 `Path` 方法。

## 2. 统一用 pathlib

- 禁止手工 `os.path.join` + 字符串拼接；一律 `Path`。
- 相对路径以"用户运行命令时的工作目录"为基准，若脚本需要固定基准，用 `Path(__file__).resolve().parent` 并注释说明。

```python
# 推荐
out = Path(args.output_dir) / "result.csv"

# 禁止
out = args.output_dir + "\\result.csv"
```

## 3. 输出目录先创建再写

写入任何输出文件前，先确保父目录存在：

```python
Path(args.output_dir).mkdir(parents=True, exist_ok=True)
```

## 4. 文件编码

- 文本文件读写显式传 `encoding="utf-8"`，Windows 默认编码不是 UTF-8，不传会乱码。
- 二进制文件（图片/视频/模型权重）用 `"rb"` / `"wb"`，不指定编码。

```python
with open(out_file, "w", encoding="utf-8") as f:
    f.write(text)
```

## 5. Windows 注意事项

- 路径含中文、空格时不要手工转义，直接用 `Path` 对象交给库函数。
- 长路径（>260 字符）用 `Path` 或启用长路径支持，不要截断。
- 打印路径时用 `str(path)`，日志中保留原始路径便于复现。

## 6. 反例清单

| 反例 | 问题 | 修正 |
| --- | --- | --- |
| `--input` 无 `default` | 运行时缺参数即崩 | 补 `default=` |
| `os.path.join(dir, "a" + name)` | 易错、平台不兼容 | `Path(dir) / name` |
| `open(path)` 不指定编码 | Windows 中文乱码 | `encoding="utf-8"` |
| 写入前不建目录 | FileNotFoundError | `mkdir(exist_ok=True)` |
