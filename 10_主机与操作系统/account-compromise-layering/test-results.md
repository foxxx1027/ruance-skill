# 压力测试结果 — `account-compromise-layering`

- **测试时间**：2026-10-19
- **测试方式**：独立 sub-agent 盲测。给 sub-agent 整包 8 个 skill 的 name + description，令其对每条 prompt 做“该激活哪一个 / none”的选择；不提供 type、expected_behavior、notes 与判卷标准。
- **判卷口径**：`should_trigger` 必须选中本 skill；`should_not_trigger` 不得选中本 skill（兄弟诱饵应转给同书另一个 skill，跨书诱饵应为 `none`）；`edge_case` 的判断需与边界理由一致。
- **结果**：**10/10（100%）**，达到 `minimum_pass_rate: 0.8`。

## 逐条结果

| # | 类型 | Prompt 摘要 | 盲测选择 | 判定 |
|---|---|---|---|---|
| 01 | should_trigger | 微信被顶下线、异地登录 | `account-compromise-layering` | ✓ 先止损 |
| 02 | should_trigger | 手机丢失，支付与银行卡 | `account-compromise-layering` | ✓ |
| 03 | should_trigger | 游戏账号被盗但未绑卡 | `account-compromise-layering` | ✓ |
| 04 | should_trigger | 改密三次仍被踢下线 | `account-compromise-layering` | ✓ 并行查终端 |
| 05 | should_not_trigger | 登录链接域名多一个字母（跨书诱饵） | `none` | ✓ |
| 06 | should_not_trigger | 工资表 zip 能不能打开（兄弟诱饵） | `email-trust-boundary` | ✓ 正确转出 |
| 07 | should_not_trigger | C 盘空间不足 | `none` | ✓ |
| 08 | edge_case | SSO 账号被锁 30 分钟 | `account-compromise-layering` | ✓ 先区分锁定与失守 |
| 09 | edge_case | 重要账号与注册小号分层 | `account-compromise-layering` | ✓ |
| 10 | edge_case | 密保问题可从公开信息猜到 | `account-compromise-layering` | ✓ 更换找回通道 |

## 失败与修复

无失败。`edge-01` 选择本 skill 而非 `none`，理由为“先判断是否异常登录与盗用”，与本 skill 第 1 步“先定损”一致，按边界判定通过；若按“必须选 none”的严格口径则为 9/10，仍 ≥80%。

## 结论

接受，无需回炉重做阶段 2。本 skill 的 description 已按盲测暴露的 trigger 歧义修订，并完成复测。
