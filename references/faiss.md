# faiss 大规模检索规范

当任务涉及大规模数据查找、相似度比对、最近邻检索时，必须用 faiss 建索引提速，禁止暴力双层循环。

## 1. 何时必须用 faiss

满足任一条件即使用：
- 比对对象数量大（经验阈值：**超过 10 万条向量**，或暴力循环耗时不可接受）；
- 需要反复查询同一批数据（先建索引，查询 O(1) 级别）；
- 特征向量维度固定、可批量组织成矩阵（如 `N x D` 的 `float32` 数组）。

```python
# 禁止：双层循环逐个算距离
for q in queries:
    for v in vectors:
        d = np.linalg.norm(q - v)

# 推荐：一次建索引，批量搜索
import faiss
import numpy as np
```

## 2. 向量预处理

- 向量统一为 `numpy.float32` 二维数组（`N x D`），faiss 不接受 float64。
- 需要"余弦相似度"语义时，先对每行 L2 归一化，再配 `IndexFlatIP`（内积）。

```python
vectors = vectors.astype("float32")
faiss.normalize_L2(vectors)          # 就地归一化，之后用内积当余弦
queries  = queries.astype("float32")
faiss.normalize_L2(queries)
```

## 3. 索引选择

| 场景 | 索引 |
| --- | --- |
| 精确检索、数据量适中（< 100 万） | `faiss.IndexFlatIP` / `faiss.IndexFlatL2` |
| 数据量大、可接受近似结果 | `faiss.IndexIVFFlat`（配合 `nlist ≈ sqrt(N)`） |
| 数据量极大、内存紧张 | `faiss.IndexIVFPQ`（压缩） |

```python
d = vectors.shape[1]

# 精确
index = faiss.IndexFlatIP(d)
index.add(vectors)

# 近似（大库）
nlist = int(np.sqrt(len(vectors)))
quantizer = faiss.IndexFlatL2(d)
index = faiss.IndexIVFFlat(quantizer, d, nlist, faiss.METRIC_L2)
index.train(vectors)      # IVFFlat 必须先 train
index.add(vectors)
index.nprobe = min(nlist, 32)   # 查询时探测的簇数，越大越准越慢
```

## 4. 批量查询

查询也批量传入，返回 `(distances, indices)`：

```python
k = 10
distances, indices = index.search(queries, k)   # queries: M x D
# indices[i] 是第 i 个查询的 top-k 索引；-1 表示未命中
```

- 禁止对每条查询单独调用 `search`；一次传入所有查询。
- 结果 `indices == -1` 表示该位置无结果（近似索引常见），调用方需处理。

## 5. 索引持久化

大规模数据建索引很贵，建好后写入磁盘复用：

```python
faiss.write_index(index, "vectors.index")
# 下次直接加载
index = faiss.read_index("vectors.index")
```

- 写索引前确保目录存在（见 `cli-paths.md`）。
- 索引文件路径同样通过 argparse 参数传入，不硬编码。

## 6. 内存释放

- 索引用完 `del index`（或 `index.reset()`）释放，再 `gc.collect()`。
- 与 `memory.md` 一致：大规模任务里索引对象驻留会占用大量内存。

## 7. 完整模板

```python
import faiss
import numpy as np
from pathlib import Path

def build_index(vectors_path, index_path):
    vectors = np.load(vectors_path).astype("float32")
    faiss.normalize_L2(vectors)
    d = vectors.shape[1]

    index = faiss.IndexFlatIP(d)
    index.add(vectors)

    Path(index_path).parent.mkdir(parents=True, exist_ok=True)
    faiss.write_index(index, index_path)
    return len(vectors)

def search(queries, index_path, k=10):
    index = faiss.read_index(index_path)
    queries = np.asarray(queries, dtype="float32")
    faiss.normalize_L2(queries)
    try:
        distances, indices = index.search(queries, k)
        return distances, indices
    finally:
        del index
```
