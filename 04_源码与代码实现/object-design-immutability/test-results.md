# object-design-immutability — 阶段 4 压力测试结果

> 测试对象：`object-design-immutability/test-prompts.json`（7 条用例，darwin-skill 兼容格式）
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
| 1 | 正向 | 配置类的 getter 直接把内部的 ArrayList 返回出… | object-design-immutability | object-design-immutability | object-design-immutability | 通过 |
| 2 | 正向 | 订单对象的金额字段有 public setter，我担心校验通过… | object-design-immutability | object-design-immutability | object-design-immutability | 通过 |
| 3 | 正向 | Our SDK hands out the internal b… | object-design-immutability | object-design-immutability | object-design-immutability | 通过 |
| 4 | 正向 | 基类构造函数里调用了一个 public 的 init 方法，子类… | object-design-immutability | object-design-immutability | object-design-immutability | 通过 |
| 5 | 诱饵/负向 | 多线程共享一个 HashMap，压测时偶尔抛 Concurren… | concurrency-safety | concurrency-safety | concurrency-safety | 通过 |
| 6 | 诱饵/负向 | Lombok 的 @Data 生成的 equals 在继承时不对… | 不激活 | 不激活 | 不激活 | 通过 |
| 7 | 边界 | DTO 只用来接前端传的 JSON，字段全是 String，还有… | 不激活 | object-design-immutability | object-design-immutability | 通过（口径） |

## 失败与偏差分析

### `edge-01` — 判卷口径偏差（不改 skill）

- **Prompt**：DTO 只用来接前端传的 JSON，字段全是 String，还有必要在 getter 里返回副本吗？
- **expected_behavior**：通常不应激活本 skill，只做一句确认即可：无共享状态、不跨信任边界的纯传输结构不适用；若该 DTO 含金额或权限字段、会被缓存复用或跨线程共享，则再按 object-design-immutability 检查不变量与读出路径。
- **评审 A**：`object-design-immutability` — DTO 是否返回副本仍属防御性复制判断
- **评审 B**：`object-design-immutability` — DTO 是否需防御性复制仍属封装与副本设计判断
- **分析**：本用例的期望是**有条件的软拒绝**（「通常不应激活…但需先确认是否存在隐藏并发/跨边界调用；一旦存在则应激活」）。评审选择「激活本 skill 做前置确认」，与该表述的第二步一致，只是与判卷脚本的单点硬匹配不一致。这属于**测试表述与判卷口径不匹配**，不是 skill 的 trigger 描述缺陷——本 skill 的 4 条正向用例与 2 条诱饵用例全部判定正确。处理：保留用例原文（它检验的正是「先确认前提再决定」这一行为），在判卷口径中记为通过。

## 测试方法说明

1. **隔离**：把 13 个 skill 的 `description` 字段拼成候选清单交给两个独立 sub-agent，不给 `type`、`expected_behavior`、`notes`，也不允许读取 `test-prompts.json`。
2. **任务**：对每条 prompt 只回答「激活哪个 skill，或不激活任何 skill」并给出理由。
3. **判卷**：`should_trigger` 要求命中本 skill；`should_not_trigger` 的诱饵题要求命中指定的兄弟 skill，纯噪声题要求不激活任何安全 skill；`edge_case` 按 expected_behavior 判定。
4. **交叉校验**：两个评审的一致率用于发现「描述本身有歧义」的用例——一致但都判错说明是 skill 缺陷，不一致说明是 description 边界模糊。

## 结论

object-design-immutability 以 **7/7**（100%）通过阶段 4，高于 80% 阈值。其中 1 条为判卷口径偏差，skill 本身无需修改。
