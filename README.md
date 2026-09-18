# Python Performance Optimizer

一个用于 [Kscc](https://kscc.ai/code) / Claude Code 类 AI 编码助手的 **Python 性能优化 Skill**。让 AI 在优化你的 Python 代码时遵循严格的工程化工作流：先理解、先定位瓶颈、再用数据验证优化效果，而不是凭感觉乱给"Quick wins"。

## 特性

- **Hard Gate**：信息不足时禁止给出任何优化建议，必须先问 3-5 个关键问题
- **Profiling 优先**：强制使用 `cProfile`（标准库）等工具定位真实瓶颈，拒绝凭空猜测
- **8 层分析框架**：算法 / 文件 I/O / 网络 / 数据库 / 子进程 / 并发 / 语言层 / 第三方库
- **P0-P2 方案排序**：按收益和风险排序优化方案，避免微观优化喧宾夺主
- **优化前后对比验证**：用 cProfile 基线 + 输出一致性检查证明优化有效

## 工作流（6 步）

```
Step 1 理解背景 → Step 2 定位瓶颈 → Step 3 分层分析
→ Step 4 排序方案 → Step 5 代码修改 → Step 6 cProfile 验证
```

| 步骤 | 说明 |
|---|---|
| 1. 理解背景 | 信息不足时先问问题：目标、耗时、数据规模、环境、依赖限制等 |
| 2. 定位瓶颈 | `cProfile` / `timeit` / `line_profiler` / `py-spy` / `tracemalloc` / 慢查询分析 |
| 3. 分层分析 | 从 8 个层次系统分析，不只盯着局部代码 |
| 4. 排序方案 | P0 高收益 → P1 中等 → P2 微观（仅 profiling 证明时建议） |
| 5. 代码修改 | 最小 diff、可运行代码、分步骤降低风险 |
| 6. 验证方法 | `before.prof` vs `after.prof` 对比 + 输出一致性检查 |

## 安装

将本仓库克隆到 kscc 的 skills 目录：

```bash
git clone https://github.com/DMianZhi/python-performance-optimizer.git \
  ~/.kscc/skills/python-performance-optimizer
```

## 使用

在 kscc 会话中直接描述你的性能问题即可触发，例如：

- "这个脚本跑得太慢了，帮我优化"
- "我的爬虫请求太多导致超时，怎么提速？"
- "pandas 处理 500 万行数据内存爆了"

也可以显式调用：`/python-performance-optimizer`

## 文件结构

```
python-performance-optimizer/
├── SKILL.md                              # Skill 主文件（工作流与规则）
├── references/
│   └── python-optimization-guide.md      # 8 层优化详细清单（按需加载）
└── README.md
```

## 防退化规则

Skill 内置以下禁令，保证 AI 输出质量：

- ❌ 无上下文时给优化建议（包括 "Quick wins"）
- ❌ 把微观优化（局部变量缓存、`__slots__` 等）作为主要方案
- ❌ 为性能随意引入复杂并发（竞态 / 死锁风险）
- ❌ 推荐不安全优化（忽略事务、跳过校验、吞异常）
- ❌ 优化完不做前后对比验证

## License

MIT
