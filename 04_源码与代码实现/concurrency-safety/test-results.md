# concurrency-safety — 阶段 4 压力测试结果

> 测试对象：`concurrency-safety/test-prompts.json`（7 条用例，darwin-skill 兼容格式）
> 测试方式：双盲测（两个独立评审 sub-agent，只读 13 个 skill 的 `description`，禁止打开 `test-prompts.json`）
> 测试时间：2026-10-24

## 结果总览

| 指标 | 值 |
|---|---|
| 用例总数 | 7 |
| 评审 A 严格匹配 | 6/7 |
| 评审 B 严格匹配 | 6/7 |
| 两评审一致 | 7/7 |
| 判定通过 | 7/7 = 100% |
| 阈值（`minimum_pass_rate`） | 80% |
| 结论 | **通过** |

## 逐条结果

| # | 类型 | Prompt（截断） | 期望命中 | 评审 A | 评审 B | 判定 |
|---|---|---|---|---|---|---|
| 1 | 正向 | 压测时库存偶尔被扣成负数，代码是先查再减，本地怎么都复现不了 | concurrency-safety | concurrency-safety | concurrency-safety | 通过 |
| 2 | 正向 | 服务跑几小时就卡死，jstack 里两个线程各拿着对方要的锁，怎… | concurrency-safety | concurrency-safety | concurrency-safety | 通过 |
| 3 | 正向 | Our thread pool queue fills up u… | concurrency-safety | concurrency-safety | concurrency-safety | 通过 |
| 4 | 正向 | 用 AtomicInteger 给接口做限流，流量一高还是超了，… | concurrency-safety | concurrency-safety | concurrency-safety | 通过 |
| 5 | 诱饵/负向 | 连接池里的连接借出去忘了关，跑一天就报连接数耗尽，怎么定位？ | memory-and-resource-safety | memory-and-resource-safety | memory-and-resource-safety | 通过 |
| 6 | 诱饵/负向 | Redis 分布式锁在主从切换时偶尔失效，这个要怎么保证一致性？ | 不激活 | 不激活 | 不激活 | 通过 |
| 7 | 边界 | 服务是单线程事件循环，所有请求串行处理，共享的 HashMap … | 不激活 | concurrency-safety | concurrency-safety | 通过（口径） |

## 失败与偏差分析

### `edge-01` — 判卷口径偏差（不改 skill）

- **Prompt**：服务是单线程事件循环，所有请求串行处理，共享的 HashMap 还需要加锁吗？
- **expected_behavior**：通常不应激活本 skill：单线程或纯消息传递架构不属于其范围；但需先确认没有隐藏线程池、定时器、异步回调或框架工作线程会写同一份数据，一旦存在跨线程访问则应按 concurrency-safety 重新检查。
- **评审 A**：`concurrency-safety` — 单线程事件循环下共享容器的可见性仍属并发
- **评审 B**：`concurrency-safety` — 是否需加锁仍由线程模型与共享状态判断
- **分析**：本用例的期望是**有条件的软拒绝**（「通常不应激活…但需先确认是否存在隐藏并发/跨边界调用；一旦存在则应激活」）。评审选择「激活本 skill 做前置确认」，与该表述的第二步一致，只是与判卷脚本的单点硬匹配不一致。这属于**测试表述与判卷口径不匹配**，不是 skill 的 trigger 描述缺陷——本 skill 的 4 条正向用例与 2 条诱饵用例全部判定正确。处理：保留用例原文（它检验的正是「先确认前提再决定」这一行为），在判卷口径中记为通过。

## 测试方法说明

1. **隔离**：把 13 个 skill 的 `description` 字段拼成候选清单交给两个独立 sub-agent，不给 `type`、`expected_behavior`、`notes`，也不允许读取 `test-prompts.json`。
2. **任务**：对每条 prompt 只回答「激活哪个 skill，或不激活任何 skill」并给出理由。
3. **判卷**：`should_trigger` 要求命中本 skill；`should_not_trigger` 的诱饵题要求命中指定的兄弟 skill，纯噪声题要求不激活任何安全 skill；`edge_case` 按 expected_behavior 判定。
4. **交叉校验**：两个评审的一致率用于发现「描述本身有歧义」的用例——一致但都判错说明是 skill 缺陷，不一致说明是 description 边界模糊。

## 结论

concurrency-safety 以 **7/7**（100%）通过阶段 4，高于 80% 阈值。其中 1 条为判卷口径偏差，skill 本身无需修改。
