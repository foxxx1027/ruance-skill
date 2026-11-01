# supply-chain-and-release-hardening — 阶段 4 压力测试结果

> 测试对象：`supply-chain-and-release-hardening/test-prompts.json`（8 条用例，darwin-skill 兼容格式）
> 测试方式：双盲测（两个独立评审 sub-agent，只读 14 个 skill 的 `description`，禁止打开 `test-prompts.json`）
> 测试时间：2026-10-24

## 结果总览

| 指标 | 值 |
|---|---|
| 用例总数 | 8 |
| 评审 A 严格匹配 | 8/8 |
| 评审 B 严格匹配 | 8/8 |
| 两评审一致 | 8/8 |
| 判定通过 | 8/8 = 100% |
| 阈值（`minimum_pass_rate`） | 80% |
| 结论 | **通过** |

## 逐条结果

| # | 类型 | Prompt（截断） | 期望命中 | 评审 A | 评审 B | 判定 |
|---|---|---|---|---|---|---|
| 1 | 正向 | 安全扫描说我们用的 log4j 有高危漏洞，业务死活不升级。我至… | supply-chain-and-release-hardening | supply-chain-and-release-hardening | supply-chain-and-release-hardening | 通过 |
| 2 | 正向 | 发版打包发现 war 里还留着测试接口和示例页面，交付前还要清哪… | supply-chain-and-release-hardening | supply-chain-and-release-hardening | supply-chain-and-release-hardening | 通过 |
| 3 | 正向 | Our npm packages come from an in… | supply-chain-and-release-hardening | supply-chain-and-release-hardening | supply-chain-and-release-hardening | 通过 |
| 4 | 正向 | CI 里同一个工程 Release 和 Debug 配置混着用，… | supply-chain-and-release-hardening | supply-chain-and-release-hardening | supply-chain-and-release-hardening | 通过 |
| 5 | 诱饵/负向 | 打出来的 jar 有 80MB，启动还慢，怎么裁依赖、优化体积？ | 不激活 | 不激活 | 不激活 | 通过 |
| 6 | 诱饵/负向 | 配置文件里的数据库密码是明文，要不要加密，密钥又该放哪？ | sensitive-data-lifecycle | sensitive-data-lifecycle | sensitive-data-lifecycle | 通过 |
| 7 | 诱饵/负向 | 扫描器报了我们两百多个 CVE，怎么按可利用性排序、哪些先修，客… | supply-chain-and-dependency-security | supply-chain-and-dependency-security | supply-chain-and-dependency-security | 通过 |
| 8 | 边界 | 依赖都锁了版本、也全部走内部仓库，供应链这块是不是就齐了？ | supply-chain-and-release-hardening | supply-chain-and-release-hardening | supply-chain-and-release-hardening | 通过 |

## 失败与偏差分析

无。7 条用例全部一次通过，两个评审结论完全一致。

## 测试方法说明

1. **隔离**：把 13 个 skill 的 `description` 字段拼成候选清单交给两个独立 sub-agent，不给 `type`、`expected_behavior`、`notes`，也不允许读取 `test-prompts.json`。
2. **任务**：对每条 prompt 只回答「激活哪个 skill，或不激活任何 skill」并给出理由。
3. **判卷**：`should_trigger` 要求命中本 skill；`should_not_trigger` 的诱饵题要求命中指定的兄弟 skill，纯噪声题要求不激活任何安全 skill；`edge_case` 按 expected_behavior 判定。
4. **交叉校验**：两个评审的一致率用于发现「描述本身有歧义」的用例——一致但都判错说明是 skill 缺陷，不一致说明是 description 边界模糊。

## 结论

supply-chain-and-release-hardening 以 **8/7**（100%）通过阶段 4，高于 80% 阈值。skill 本身无需修改。

## 口径变更记录（2026-10-24 复测）

**变更原因**：安装后发现本机已存在 `supply-chain-and-dependency-security`（负责依赖清单、SCA 比对 CVE、依赖投毒排查、SBOM 与修复优先级）。两者在"第三方组件"上触发重叠，同一个问题可能被任一 skill 抢走。

**变更内容**：

1. `description` 收窄为"交付前核对与收口"，并显式写明 CVE 比对 / 依赖投毒 / SBOM / 修复优先级应转 `supply-chain-and-dependency-security`；触发信号中移除 `依赖投毒`、`组件漏洞`、`SBOM`、`SCA`、`dependency confusion`、`typosquatting`。
2. `A2 — 与相邻 skill 的区分` 增加一条：问"该不该紧急升级、怎么排优先级"归前者，问"能否证明发布的就是审过的那份代码、产物里还残留什么"归后者。
3. `B — 不要在以下情况使用` 增加对应条目。
4. 用例调整：`should-trigger-01` 从"CVE 能否先上线"改写为"内部仓库里的包是否官方原版、构建时是否被替换"，争点从漏洞定级转到交付产物可信性；新增 `should-not-trigger-03`（CVE 排序 + SBOM），用于验证新划定的边界。

**复测结果**：候选集从 13 个扩到 14 个（加入 `supply-chain-and-dependency-security`），重跑双盲测，8/8 全部命中，两评审一致；`should-not-trigger-03` 被两个评审一致路由到 `supply-chain-and-dependency-security`，边界成立。
