# authorization-and-least-privilege — 阶段 4 压力测试结果

> 测试对象：`authorization-and-least-privilege/test-prompts.json`（7 条用例，darwin-skill 兼容格式）
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
| 1 | 正向 | 订单详情接口只按 id 查库，用户改成别人的 id 也能看到内容… | authorization-and-least-privilege | authorization-and-least-privilege | authorization-and-least-privilege | 通过 |
| 2 | 正向 | 后台给运营配了管理员角色，现在他能看到所有租户的数据，权限该怎么拆？ | authorization-and-least-privilege | authorization-and-least-privilege | authorization-and-least-privilege | 通过 |
| 3 | 正向 | 应用连数据库用的是 root 账号，图省事，这样有问题吗？ | authorization-and-least-privilege | authorization-and-least-privilege | authorization-and-least-privilege | 通过 |
| 4 | 正向 | A user changed the order id in t… | authorization-and-least-privilege | authorization-and-least-privilege | authorization-and-least-privilege | 通过 |
| 5 | 诱饵/负向 | 登录验证码最近被刷得很凶，而且用户不存在时提示语也不一样，该怎么… | authentication-and-passwords | authentication-and-passwords | authentication-and-passwords | 通过 |
| 6 | 诱饵/负向 | 这段权限判断的 if-else 写得太长了，帮我重构成策略模式，… | 不激活 | 不激活 | 不激活 | 通过 |
| 7 | 边界 | 查询已经用了 PreparedStatement，参数也绑定好了… | authorization-and-least-privilege | authorization-and-least-privilege | authorization-and-least-privilege | 通过 |

## 失败与偏差分析

无。7 条用例全部一次通过，两个评审结论完全一致。

## 测试方法说明

1. **隔离**：把 13 个 skill 的 `description` 字段拼成候选清单交给两个独立 sub-agent，不给 `type`、`expected_behavior`、`notes`，也不允许读取 `test-prompts.json`。
2. **任务**：对每条 prompt 只回答「激活哪个 skill，或不激活任何 skill」并给出理由。
3. **判卷**：`should_trigger` 要求命中本 skill；`should_not_trigger` 的诱饵题要求命中指定的兄弟 skill，纯噪声题要求不激活任何安全 skill；`edge_case` 按 expected_behavior 判定。
4. **交叉校验**：两个评审的一致率用于发现「描述本身有歧义」的用例——一致但都判错说明是 skill 缺陷，不一致说明是 description 边界模糊。

## 结论

authorization-and-least-privilege 以 **7/7**（100%）通过阶段 4，高于 80% 阈值。skill 本身无需修改。
