# 压力测试结果 — `email-trust-boundary`

- **测试时间**：2026-10-19
- **测试方式**：独立 sub-agent 盲测。给 sub-agent 整包 8 个 skill 的 name + description，令其对每条 prompt 做“该激活哪一个 / none”的选择；不提供 type、expected_behavior、notes 与判卷标准。
- **判卷口径**：`should_trigger` 必须选中本 skill；`should_not_trigger` 不得选中本 skill（兄弟诱饵应转给同书另一个 skill，跨书诱饵应为 `none`）；`edge_case` 的判断需与边界理由一致。
- **结果**：**10/10（100%）**，达到 `minimum_pass_rate: 0.8`。

## 逐条结果

| # | 类型 | Prompt 摘要 | 盲测选择 | 判定 |
|---|---|---|---|---|
| 01 | should_trigger | 工资表 zip 能不能打开 | `email-trust-boundary` | ✓ |
| 02 | should_trigger | 没发过信却收到退信 | `email-trust-boundary` | ✓ |
| 03 | should_trigger | 显示名是老板、地址是 gmail | `email-trust-boundary` | ✓ |
| 04 | should_trigger | 重要邮箱与注册邮箱分层 | `email-trust-boundary` | ✓ |
| 05 | should_not_trigger | 邮箱已被顶下线（兄弟诱饵） | `account-compromise-layering` | ✓ 正确转出 |
| 06 | should_not_trigger | 附件已双击、电脑弹窗（跨书诱饵） | `none` | ✓ |
| 07 | should_not_trigger | 配置 SPF/DKIM | `none` | ✓ |
| 08 | edge_case | 熟人账号被盗后发来的 docx | `email-trust-boundary` | ✓ 独立渠道核实 |
| 09 | edge_case | 网关已拦 99% 垃圾邮件 | `email-trust-boundary` | ✓ 仍保留三查 |
| 10 | edge_case | .scr 伪装成 invoice | `email-trust-boundary` | ✓ 视为可执行附件 |

## 失败与修复

无失败。

## 结论

接受，无需回炉重做阶段 2。本 skill 的 description 已按盲测暴露的 trigger 歧义修订，并完成复测。
