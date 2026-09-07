# 内存管理规范

处理文件、图片、视频、大数组时的内存纪律。核心目标：**处理完立即释放，不在循环中累积引用，内存峰值有界。**

## 1. 用上下文管理器

文件、图片、连接一律 `with`，作用域结束自动释放句柄：

```python
# 推荐：with 自动关闭
with Image.open(p) as img:
    data = img.convert("RGB")

# 禁止：忘记 close 导致句柄泄漏
img = Image.open(p)
data = img.convert("RGB")
```

## 2. 图片逐张处理，用完即释放

批处理图片的标准模式：**循环内开 → 处理 → 存盘 → `del` → `gc.collect()`**。

```python
import gc
from pathlib import Path
from PIL import Image

def process_images(input_dir, output_dir):
    input_dir, output_dir = Path(input_dir), Path(output_dir)
    output_dir.mkdir(parents=True, exist_ok=True)

    for p in sorted(input_dir.glob("*.jpg")):
        with Image.open(p) as img:
            out = img.convert("RGB").resize((512, 512))
            out.save(output_dir / p.name)
        del out          # 释放图片对象引用
        gc.collect()     # 回收不可达对象，控制峰值
```

- 每张图处理完必须 `del` + `gc.collect()`，尤其是 PIL/OpenCV/numpy 大数组。
- 不要在循环外保存所有处理结果再统一写盘——除非结果总量可预期且小。

## 3. 大文件流式读取，不一次载入

- 文本大文件逐行/分块处理，禁止 `data = f.read()` 全量载入。

```python
with open(big_file, encoding="utf-8") as f:
    for line in f:          # 逐行，内存恒定
        process_line(line)
```

- 二进制大文件用固定大小分块：

```python
with open(big_file, "rb") as f:
    while chunk := f.read(1 << 20):   # 1MB 一块
        process_chunk(chunk)
```

## 4. 用生成器代替全量列表

- 需要"逐个产出再处理"时用生成器，避免把全部路径/数据装入内存：

```python
def iter_images(d):
    for p in sorted(Path(d).glob("*.jpg")):
        with Image.open(p) as img:
            yield img.convert("RGB")   # 逐个产出，不驻留

for img in iter_images("data/input"):
    img.save(...)
```

## 5. 并发时的内存峰值控制

- 多线程批处理图片时，worker 数 × 单图内存必须小于可用内存；用信号量或限制 worker 数兜底。

```python
import threading

sem = threading.Semaphore(4)   # 最多同时 4 张图在内存

def worker(p):
    with sem:
        with Image.open(p) as img:
            ...  # 处理
```

## 6. 其他资源释放

- faiss 索引、模型、GPU tensor 用完释放：`del index` / `index.reset()` / `del model`，必要时 `gc.collect()`。
- 不再使用的 DataFrame/数组用 `del` 缩短生命周期，避免长函数内堆积。

## 7. 反例清单

| 反例 | 问题 |
| --- | --- |
| 循环外累积所有图片再处理 | 内存随图片数线性增长直至 OOM |
| `open()` 不用 with | 句柄泄漏，Windows 上文件被占用无法删除 |
| `f.read()` 读整个大文件 | 峰值内存 = 文件大小 |
| 列表推导收集所有路径再 for | 可改为生成器 |
