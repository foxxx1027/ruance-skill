# sensitive-data-lifecycle — 阶段 4 压力测试结果

> 测试对象：`sensitive-data-lifecycle/test-prompts.json`（7 条用例，darwin-skill 兼容格式）
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
| 1 | 正向 | 用户表要存身份证号和银行卡，哪些字段必须加密，密钥又该放在哪里？ | sensitive-data-lifecycle | sensitive-data-lifecycle | sensitive-data-lifecycle | 通过 |
| 2 | 正向 | 数据库密码是明文写在 application.yml 里的，还提… | sensitive-data-lifecycle | sensitive-data-lifecycle | sensitive-data-lifecycle | 通过 |
| 3 | 正向 | 埋点要上报用户手机号和城市，产品说只用于统计，这样行不行？ | sensitive-data-lifecycle | sensitive-data-lifecycle | sensitive-data-lifecycle | 通过 |
| 4 | 正向 | We generate password reset token… | sensitive-data-lifecycle | sensitive-data-lifecycle | sensitive-data-lifecycle | 通过 |
| 5 | 诱饵/负向 | 登录失败五次就锁定账号，会不会被人故意拿来锁死别人的号？ | authentication-and-passwords | authentication-and-passwords | authentication-and-passwords | 通过 |
| 6 | 诱饵/负向 | App 启动时那个隐私政策同意弹窗，文案和勾选项怎么写才合规？ | 不激活 | 不激活 | 不激活 | 通过 |
| 7 | 边界 | 手机号我们做了 MD5 存库，查询时也算一遍比对，这样算脱敏了吗？ | sensitive-data-lifecycle | sensitive-data-lifecycle | sensitive-data-lifecycle | 通过 |

## 失败与偏差分析

无。7 条用例全部一次通过，两个评审结论完全一致。

## 测试方法说明

1. **隔离**：把 13 个 skill 的 `description` 字段拼成候选清单交给两个独立 sub-agent，不给 `type`、`expected_behavior`、`notes`，也不允许读取 `test-prompts.json`。
2. **任务**：对每条 prompt 只回答「激活哪个 skill，或不激活任何 skill」并给出理由。
3. **判卷**：`should_trigger` 要求命中本 skill；`should_not_trigger` 的诱饵题要求命中指定的兄弟 skill，纯噪声题要求不激活任何安全 skill；`edge_case` 按 expected_behavior 判定。
4. **交叉校验**：两个评审的一致率用于发现「描述本身有歧义」的用例——一致但都判错说明是 skill 缺陷，不一致说明是 description 边界模糊。

## 结论

sensitive-data-lifecycle 以 **7/7**（100%）通过阶段 4，高于 80% 阈值。skill 本身无需修改。
