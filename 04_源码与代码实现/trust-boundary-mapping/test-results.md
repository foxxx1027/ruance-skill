# trust-boundary-mapping — 阶段 4 压力测试结果

> 测试对象：`trust-boundary-mapping/test-prompts.json`（7 条用例，darwin-skill 兼容格式）
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
| 1 | 正向 | 服务 A 用 HTTP 调 B，B 再消费 Kafka 写 Re… | trust-boundary-mapping | trust-boundary-mapping | trust-boundary-mapping | 通过 |
| 2 | 正向 | 网关说 token 已经验过了，内网服务之间互调时这个 user… | trust-boundary-mapping | trust-boundary-mapping | trust-boundary-mapping | 通过 |
| 3 | 正向 | 表单里那个隐藏的价格字段用户改不了吧，我直接拿它算总价可以吗？ | trust-boundary-mapping | trust-boundary-mapping | trust-boundary-mapping | 通过 |
| 4 | 正向 | Our service consumes messages fr… | trust-boundary-mapping | trust-boundary-mapping | trust-boundary-mapping | 通过 |
| 5 | 诱饵/负向 | JSP 里直接把 request.getParameter('n… | input-validation-output-encoding | input-validation-output-encoding | input-validation-output-encoding | 通过 |
| 6 | 诱饵/负向 | 我写了个单机离线脚本，只读本地 CSV 汇总后生成报表，这种也要… | 不激活 | 不激活 | 不激活 | 通过 |
| 7 | 边界 | Redis 里的数据都是我们自己服务写进去的，读出来还要不要再校… | trust-boundary-mapping | trust-boundary-mapping | trust-boundary-mapping | 通过 |

## 失败与偏差分析

无。7 条用例全部一次通过，两个评审结论完全一致。

## 测试方法说明

1. **隔离**：把 13 个 skill 的 `description` 字段拼成候选清单交给两个独立 sub-agent，不给 `type`、`expected_behavior`、`notes`，也不允许读取 `test-prompts.json`。
2. **任务**：对每条 prompt 只回答「激活哪个 skill，或不激活任何 skill」并给出理由。
3. **判卷**：`should_trigger` 要求命中本 skill；`should_not_trigger` 的诱饵题要求命中指定的兄弟 skill，纯噪声题要求不激活任何安全 skill；`edge_case` 按 expected_behavior 判定。
4. **交叉校验**：两个评审的一致率用于发现「描述本身有歧义」的用例——一致但都判错说明是 skill 缺陷，不一致说明是 description 边界模糊。

## 结论

trust-boundary-mapping 以 **7/7**（100%）通过阶段 4，高于 80% 阈值。skill 本身无需修改。
