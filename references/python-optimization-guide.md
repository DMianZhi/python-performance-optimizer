# Python 优化指南 — 分层分析参考

此文件在 Step 3（分层分析）时按需加载。AI 根据用户代码涉及的具体场景，查阅对应章节。

---

## Profiling 工具详解

### timeit — 微基准测试（标准库）

```bash
# 命令行：测试单条语句
python -m timeit "'-'.join(str(n) for n in range(100))"

# 命令行：测试函数
python -m timeit -s "from my_module import my_func" "my_func()"

# 指定运行次数
python -m timeit -n 1000 -r 5 "x = [i**2 for i in range(100)]"
```

```python
# 代码内使用
import timeit

# 测试函数
elapsed = timeit.timeit("my_func()", setup="from __main__ import my_func", number=100)
print(f"平均耗时: {elapsed / 100:.6f}s")

# 自动确定重复次数
elapsed = timeit.timeit("my_func()", setup="from __main__ import my_func", number=timeit.auto)
```

### cProfile — 函数级性能分析（标准库，推荐首选）

```bash
# 命令行：直接运行
python -m cProfile -s cumulative your_script.py
python -m cProfile -s time your_script.py        # 按单次耗时排序
python -m cProfile -s calls your_script.py       # 按调用次数排序
python -m cProfile -o output.prof your_script.py # 输出到文件
```

```python
# 代码内使用
import cProfile
import pstats

with cProfile.Profile() as pr:
    main()

# 分析结果
stats = pstats.Stats(pr)
stats.sort_stats("cumulative")   # 累计耗时
stats.print_stats(20)            # Top 20

# 按函数名过滤
stats.sort_stats("time")         # 单次耗时
stats.print_stats("my_module", 10)  # 只看 my_module 中的函数

# 查看调用者/被调用者
stats.print_callers("slow_func")
stats.print_callees("slow_func")
```

### profile — 纯 Python 版 profiler（标准库，cProfile 不可用时备选）

```bash
python -m profile -s cumulative your_script.py
```
用法与 cProfile 完全相同，但因为是纯 Python 实现，比 cProfile 慢约 10 倍。仅在 cProfile 不可用的环境（如部分嵌入式 Python）使用。

### time.perf_counter() — 高精度计时（标准库）

```python
import time

# 测量代码块
t0 = time.perf_counter()
result = do_expensive_work()
elapsed = time.perf_counter() - t0
print(f"耗时: {elapsed:.3f}s")

# 上下文管理器版本
import contextlib
import time

@contextlib.contextmanager
def timer(label: str):
    t0 = time.perf_counter()
    yield
    print(f"[{label}] 耗时: {time.perf_counter() - t0:.3f}s")

with timer("数据处理"):
    process_data()
```

### tracemalloc — 内存追踪（标准库）

```python
import tracemalloc

tracemalloc.start()

# 运行目标代码
main()

# 获取内存快照并对比
snapshot = tracemalloc.take_snapshot()
top_stats = snapshot.statistics("lineno")

print("[ Top 10 内存分配 ]")
for stat in top_stats[:10]:
    print(stat)
```

```bash
# 命令行模式
python -m tracemalloc your_script.py
```

### line_profiler — 行级分析（第三方）

```bash
pip install line_profiler
```

```python
# 在函数上加 @profile 装饰器
@profile
def slow_function():
    # ... 代码 ...
```

```bash
kernprof -l -v your_script.py
```

### py-spy — 实时观察（第三方，无需改代码）

```bash
pip install py-spy

# 实时 top 视图
py-spy top -- python your_script.py

# 生成火焰图
py-spy record -o flame.svg -- python your_script.py

# 附加到正在运行的进程
py-spy top --pid <PID>
```

### memray — 内存火焰图（第三方）

```bash
pip install memray

memray run your_script.py
memray flamegraph memray-your_script.py.*.bin
```

---

## 1. 算法与数据结构

**加载时机：** 代码中有循环嵌套、查找操作、排序、重复计算

### 常见问题

| 问题 | 症状 | 优化方向 |
|---|---|---|
| O(n²) 或更高复杂度 | 嵌套循环处理大量数据 | 哈希表、排序后二分查找、双指针 |
| list 中反复 `in` 查找 | `if x in my_list` 在循环中 | 改用 `set`/`dict`，O(n) → O(1) |
| 重复排序 | 每次循环都 `sorted()` | 排序一次，缓存结果 |
| 重复计算 | 相同表达式在循环中多次出现 | 提取到循环外、缓存、前缀和 |
| 递归无记忆化 | 斐波那契类递归 | `functools.lru_cache` 或迭代 |

### 优化手段

- 用 `set`/`dict` 替代 `list` 做成员检查
- 用 `collections.defaultdict` 替代手动初始化
- 用 `collections.Counter` 替代手动计数
- 用 `heapq` 替代全排序取 top-k
- 用 `deque` 替代 `list.pop(0)`
- 用 `bisect` 在有序列表中快速查找

---

## 2. 文件 I/O

**加载时机：** 涉及文件读写、目录遍历、大量小文件

### 常见问题

| 问题 | 症状 | 优化方向 |
|---|---|---|
| 每行 flush | `file.flush()` 在循环中 | 批量 flush 或依赖 OS 缓冲 |
| 循环中频繁 open/close | 循环内 `with open()` | 批量读取、合并文件、流式处理 |
| `os.listdir` + `os.stat` | 遍历目录后逐个 stat | 用 `os.scandir` 或 `pathlib.Path().iterdir()` |
| 重复遍历目录 | 多次 `os.walk` 同一目录 | 遍历一次，缓存路径列表 |
| 一次性加载大文件 | `f.read()` 或 `f.readlines()` | 流式读取 `for line in f` |
| 大量小文件 | 50000 个 1KB 文件 | 合并为单文件、归档、SQLite |

### 优化手段

- `os.scandir()` 替代 `os.listdir()` + `os.stat()` — 减少系统调用
- 大文件用 `for line in file` 流式读取
- CSV 用 `csv.reader` 或 `pandas.read_csv(chunksize=...)`
- JSON Lines (`.jsonl`) 适合逐条写入和追加
- 大量结构化数据考虑 SQLite、DuckDB、Parquet
- 输出文件批量 flush，不要每行 flush
- 用 `pathlib` 提高可读性（性能与 `os.scandir` 相当）
- 文件写入用临时文件 + 原子重命名，避免写入中断损坏

---

## 3. 网络请求

**加载时机：** 涉及 HTTP/API 请求、爬虫、下载

### 常见问题

| 问题 | 症状 | 优化方向 |
|---|---|---|
| 串行请求 | 逐个请求 API，总耗时 = 单次 x N | 并发请求（线程池/asyncio） |
| 每次新建连接 | 每次 `requests.get()` 新建 Session | 复用 `requests.Session()` |
| 无超时设置 | 请求可能永久挂起 | 设置 `timeout=(connect, read)` |
| 无重试机制 | 网络抖动导致失败 | 指数退避重试（tenacity/backoff） |
| 重复请求相同资源 | 相同 URL 多次请求 | 缓存响应（requests-cache/diskcache） |
| 未使用批量接口 | API 支持但未使用 | 合并请求，减少往返次数 |

### 优化手段

- HTTP 请求用 `requests.Session()` 复用 TCP 连接
- I/O 密集型并发用 `ThreadPoolExecutor` 或 `asyncio` + `aiohttp`/`httpx`
- 设置 `timeout=(3.0, 30.0)` 避免永久挂起
- 使用指数退避重试：`tenacity` 库或手动实现
- 缓存可缓存的响应：`requests-cache`、`diskcache`
- 批量接口优于逐条请求
- 分页请求并发时注意 API rate limit
- 失败任务持久化记录，便于重跑

---

## 4. 数据库

**加载时机：** 涉及 SQL 查询、ORM、批量写入

### 常见问题

| 问题 | 症状 | 优化方向 |
|---|---|---|
| N+1 查询 | 循环中每次查一条 | 批量查询、`IN` 子句、JOIN |
| 循环中逐条 INSERT | for 循环 + `execute()` | `executemany()`、批量 INSERT、COPY |
| 每条记录 commit | 事务内逐条提交 | 单事务批量提交 |
| 缺少索引 | 查询扫描全表 | 添加索引（检查 `EXPLAIN`） |
| 查询不需要的字段 | `SELECT *` | 只选必要字段 |
| Python 层做过滤/聚合 | 加载全量数据再 filter | 用 SQL WHERE/GROUP BY/HAVING |
| 无连接池 | 每次新建连接 | `SQLAlchemy.pool`、`psycopg2.pool` |

### 优化手段

- 用 `EXPLAIN` / 慢查询日志定位问题查询
- 循环中逐条查询 → 批量 `WHERE id IN (...)`
- 循环中逐条 INSERT → `executemany()` + 单事务
- ORM N+1 → `selectinload` / `joinedload` (SQLAlchemy) 或 `select_related` / `prefetch_related` (Django)
- 只查询必要字段，不用 `SELECT *`
- 让数据库做过滤、排序、聚合，不要拉到 Python 层处理
- 使用连接池复用连接
- 大批量任务分批提交（每 5000-10000 条），避免长事务和锁问题
- 批量导入用数据库原生工具：`COPY`（PostgreSQL）、`LOAD DATA`（MySQL）

### 安全性警告

- `PRAGMA synchronous = OFF`（SQLite）会牺牲 crash 安全，仅在可重建数据库时使用
- 不要为了速度跳过事务或关闭校验，正确性优先
- 如果建议不安全优化，必须明确标注风险并给出安全的替代方案

---

## 5. 子进程与外部命令

**加载时机：** 涉及 `subprocess`、`os.system`、外部命令调用

### 常见问题

| 问题 | 症状 | 优化方向 |
|---|---|---|
| 频繁启动子进程 | 循环中调用 subprocess | 合并命令、复用进程、Python 原生替代 |
| `shell=True` | 额外 shell 开销 + 安全风险 | 用列表参数替代字符串 |
| 串行调用 | 多个独立命令逐个执行 | 并行执行 |
| 用外部命令做简单操作 | `grep`/`sed`/`awk` 替代 Python | 用 Python 原生库（`re`、`glob`、`os`） |

### 优化手段

- 合并多个命令为一次调用：`ffmpeg` 多输入、`git` 批量操作
- 用 `subprocess.Popen` 复用进程（stdin 持续输入）
- 简单文本处理用 Python 原生代替外部命令
- 并行执行独立命令：`concurrent.futures` 或 `asyncio.create_subprocess_exec`
- 永远用列表参数：`["cmd", "arg1"]` 而非 `"cmd arg1"` + `shell=True`

---

## 6. 并发与并行

**加载时机：** 用户提到并发、多线程、asyncio，或任务明显可并行

### 判断流程

```
任务是 CPU 密集型还是 I/O 密集型？
├── I/O 密集型（网络、磁盘、数据库等待）
│   ├── 简单场景 → ThreadPoolExecutor
│   └── 复杂场景 → asyncio + aiohttp/httpx
└── CPU 密集型（计算、加密、编解码）
    └── ProcessPoolExecutor（回避 GIL）
```

### 注意事项

- 多线程在 CPU 密集型 Python 中受 GIL 限制，不能提速
- 并发数不是越大越好，结合目标服务承载能力
- 并发可能引入竞态、死锁、重复写入、资源耗尽
- 仅为明确适合的场景引入并发，不要为了"看起来更快"而加并发
- 提供并发方案时，必须说明潜在风险和替代方案

---

## 7. Python 语言层

**加载时机：** profiling 证明热路径在纯 Python 代码（非 I/O、非第三方库）

> 注意：此章节的优化是 P2 级别，仅在 profiling 证明是热点时建议。

### 常见优化

| 优化 | 说明 | 何时用 |
|---|---|---|
| 预编译正则 | `re.compile()` 缓存 | 循环中反复使用同一正则 |
| 局部变量缓存 | 热路径中 `local = module.func` | 减少属性查找 |
| 生成器替代列表 | `yield` 或 `()` 替代 `[]` | 中间结果不需要全部保留 |
| 字符串拼接 | `''.join()` 替代循环 `+=` | 大量字符串拼接 |
| `__slots__` | 减少对象内存 | 大量小对象实例 |
| `lru_cache` | 缓存纯函数结果 | 重复调用相同参数 |

---

## 8. 第三方库使用方式

**加载时机：** 代码使用了 pandas、numpy、requests、SQLAlchemy 等库

### 常见误用

| 库 | 误用 | 优化 |
|---|---|---|
| pandas | `iterrows()` 逐行遍历 | 向量化操作、`apply()`、`itertuples()` |
| pandas | 循环中 `pd.concat()` | 收集到列表，一次性 concat |
| numpy | 用 Python 循环处理数组 | 向量化运算 |
| requests | 每次新建连接 | 使用 `Session()` |
| SQLAlchemy | lazy loading 导致 N+1 | `selectinload` / `joinedload` |
| logging | 热路径中做昂贵字符串格式化 | 用 `%s` 延迟格式化，或用 `logger.isEnabledFor()` |
| datetime | 循环中反复 `strptime` | 预编译格式、缓存结果 |
| json | 循环中反复 `json.dumps/loads` | 批处理、用 `orjson` 替代 |

---

## 9. 长期运行脚本

**加载时机：** 脚本会定期运行、长期运行、或作为后台服务

### 优化方向

- **增量处理：** 只处理新增或变化的数据，用时间戳或 ID 标记
- **状态持久化：** 记录 `last_run_time`、已处理 ID 集合、checkpoint
- **断点续跑：** 失败后能从上次位置继续，不重复处理
- **幂等性：** 重复执行不会造成重复写入或副作用
- **日志与指标：** 记录每次运行的耗时、处理条数、成功数、失败数
- **资源清理：** 确保连接关闭、文件句柄释放、进程回收
- **配置化：** 并发数、batch size、timeout、重试次数可配置
- **监控：** 关注越来越慢、失败率升高、数据积压等趋势

### 断点续跑模式

```python
import json
import os

CHECKPOINT_FILE = "checkpoint.json"

def load_checkpoint():
    if os.path.exists(CHECKPOINT_FILE):
        with open(CHECKPOINT_FILE) as f:
            return set(json.load(f))
    return set()

def save_checkpoint(processed_ids):
    with open(CHECKPOINT_FILE, 'w') as f:
        json.dump(list(processed_ids), f)

def process_items(items):
    processed = load_checkpoint()
    for item in items:
        if item['id'] in processed:
            continue
        try:
            do_work(item)
            processed.add(item['id'])
        except Exception as e:
            logger.error(f"Failed {item['id']}: {e}")
        finally:
            save_checkpoint(processed)  # 每条记录后保存
```