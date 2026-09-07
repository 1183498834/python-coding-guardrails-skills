# 并发提速规范

**前提：不改变程序逻辑与结果。** 只替换执行方式，不修改算法本身。提速前先确认瓶颈，别盲目加线程。

## 1. 判断瓶颈类型

| 场景 | 密集类型 | 推荐方式 |
| --- | --- | --- |
| 读写文件、下载、图片解码、网络请求 | IO 密集 | `ThreadPoolExecutor` |
| 数值计算、向量比对、纯 CPU 循环 | CPU 密集 | `ProcessPoolExecutor`（绕过 GIL） |

- Python 有 GIL，纯计算用线程池不会提速；IO 用线程池效果明显。
- 不确定时先小规模计时对比，再决定。

## 2. 保序 vs 不保序

- **结果需要保持输入顺序**：`executor.map()`（按提交顺序返回，懒加载）。

```python
from concurrent.futures import ThreadPoolExecutor

with ThreadPoolExecutor(max_workers=8) as ex:
    results = list(ex.map(process_one, image_paths))
```

- **结果顺序无所谓、想尽快出结果**：`submit()` + `as_completed()`。

```python
from concurrent.futures import ThreadPoolExecutor, as_completed

with ThreadPoolExecutor(max_workers=8) as ex:
    futures = {ex.submit(process_one, p): p for p in image_paths}
    for fut in as_completed(futures):
        try:
            done = fut.result()
        except Exception as e:
            print(f"任务失败: {futures[fut]} -> {e}")
```

## 3. 线程数与资源上限

- 不硬编码线程数。默认 `min(32, os.cpu_count() + 4)`；IO 密集可适当放大，CPU 密集不超过核心数。
- 并发任务总量有界：任务列表过大时分批提交，避免一次塞入百万个 future 导致内存暴涨。

```python
import os

MAX_WORKERS = min(32, os.cpu_count() + 4)
```

## 4. 线程安全

- 各 worker 尽量只依赖入参、返回结果，不共享可变状态。
- 必须共享（如写日志、写同一个输出文件）时：
  - 用 `threading.Lock` 保护临界区；
  - 或让每个 worker 写独立文件，最后合并；
  - 或只让主线程做汇总/写盘。

```python
import threading

write_lock = threading.Lock()

def save(item):
    with write_lock:
        with open(out, "a", encoding="utf-8") as f:
            f.write(str(item) + "\n")
```

## 5. 进度与错误可见

- 长任务用 `tqdm` 显示进度（`tqdm(executor.map(...), total=n)`）。
- 每个 worker 内捕获异常并返回 `(路径, 错误)` 结构，汇总时统一报告，避免一个坏文件中断整批。

## 6. 完整模板

```python
import os
from concurrent.futures import ThreadPoolExecutor, as_completed
from pathlib import Path
from tqdm import tqdm

def process_batch(image_paths, out_dir, max_workers=None):
    out_dir = Path(out_dir)
    out_dir.mkdir(parents=True, exist_ok=True)
    workers = max_workers or min(32, os.cpu_count() + 4)
    errors = []

    with ThreadPoolExecutor(max_workers=workers) as ex:
        futures = {ex.submit(process_one, p, out_dir): p for p in image_paths}
        for fut in tqdm(as_completed(futures), total=len(futures), desc="处理中"):
            p = futures[fut]
            try:
                fut.result()
            except Exception as e:
                errors.append((str(p), repr(e)))

    if errors:
        print(f"共 {len(errors)} 个失败:")
        for path, err in errors[:10]:
            print(f"  {path}: {err}")
    return errors
```
