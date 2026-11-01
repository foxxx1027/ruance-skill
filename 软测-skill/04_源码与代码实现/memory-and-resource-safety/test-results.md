# memory-and-resource-safety — 阶段 4 压力测试结果

> 测试对象：`memory-and-resource-safety/test-prompts.json`（7 条用例，darwin-skill 兼容格式）
> 测试方式：双盲测（两个独立评审 sub-agent，只读 13 个 skill 的 `description`，禁止打开 `test-prompts.json`）
> 测试时间：2026-10-24

## 结果总览

| 指标 | 值 |
|---|---|
| 用例总数 | 7 |
| 评审 A 严格匹配 | 7/7 |
| 评审 B 严格匹配 | 7/7 |
| 两评审一致 | 7/7 |
| 判定通过 | 7/7 = 100% |
| 阈值（`minimum_pass_rate`） | 80% |
| 结论 | **通过** |

## 逐条结果

| # | 类型 | Prompt（截断） | 期望命中 | 评审 A | 评审 B | 判定 |
|---|---|---|---|---|---|---|
| 1 | 正向 | Java 服务跑一天后句柄数一直涨，重启就恢复，是不是哪里没 c… | memory-and-resource-safety | memory-and-resource-safety | memory-and-resource-safety | 通过 |
| 2 | 正向 | 上传的 zip 只有几 KB，解压后把磁盘写满了，怎么加限制？ | memory-and-resource-safety | memory-and-resource-safety | memory-and-resource-safety | 通过 |
| 3 | 正向 | C 里这段 memcpy 的长度是从请求头里读的，会不会写越界？ | memory-and-resource-safety | memory-and-resource-safety | memory-and-resource-safety | 通过 |
| 4 | 正向 | We run on the JVM with GC, but f… | memory-and-resource-safety | memory-and-resource-safety | memory-and-resource-safety | 通过 |
| 5 | 诱饵/负向 | 两个线程同时给同一个计数器加一，压测下来少了好几百，加个锁能解决吗？ | concurrency-safety | concurrency-safety | concurrency-safety | 通过 |
| 6 | 诱饵/负向 | 我们准备把 JVM 堆从 4G 调到 8G，GC 参数怎么配比较稳？ | 不激活 | 不激活 | 不激活 | 通过 |
| 7 | 边界 | Go 里我在循环体里写了 defer file.Close()，… | memory-and-resource-safety | memory-and-resource-safety | memory-and-resource-safety | 通过 |

## 失败与偏差分析

无。7 条用例全部一次通过，两个评审结论完全一致。

## 测试方法说明

1. **隔离**：把 13 个 skill 的 `description` 字段拼成候选清单交给两个独立 sub-agent，不给 `type`、`expected_behavior`、`notes`，也不允许读取 `test-prompts.json`。
2. **任务**：对每条 prompt 只回答「激活哪个 skill，或不激活任何 skill」并给出理由。
3. **判卷**：`should_trigger` 要求命中本 skill；`should_not_trigger` 的诱饵题要求命中指定的兄弟 skill，纯噪声题要求不激活任何安全 skill；`edge_case` 按 expected_behavior 判定。
4. **交叉校验**：两个评审的一致率用于发现「描述本身有歧义」的用例——一致但都判错说明是 skill 缺陷，不一致说明是 description 边界模糊。

## 结论

memory-and-resource-safety 以 **7/7**（100%）通过阶段 4，高于 80% 阈值。skill 本身无需修改。
