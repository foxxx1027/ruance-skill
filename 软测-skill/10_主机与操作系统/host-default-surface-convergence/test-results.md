# 压力测试结果 — `host-default-surface-convergence`

- **测试时间**：2026-10-19
- **测试方式**：独立 sub-agent 盲测。给 sub-agent 整包 8 个 skill 的 name + description，令其对每条 prompt 做“该激活哪一个 / none”的选择；不提供 type、expected_behavior、notes 与判卷标准。
- **判卷口径**：`should_trigger` 必须选中本 skill；`should_not_trigger` 不得选中本 skill（兄弟诱饵应转给同书另一个 skill，跨书诱饵应为 `none`）；`edge_case` 的判断需与边界理由一致。
- **结果**：**10/10（100%）**，达到 `minimum_pass_rate: 0.8`。

## 逐条结果

| # | 类型 | Prompt 摘要 | 盲测选择 | 判定 |
|---|---|---|---|---|
| 01 | should_trigger | 新装 Windows 办公机的默认设置 | `host-default-surface-convergence` | ✓ |
| 02 | should_trigger | Administrator 改名、Guest 禁用 | `host-default-surface-convergence` | ✓ |
| 03 | should_trigger | Everyone 读取共享盘 | `host-default-surface-convergence` | ✓ |
| 04 | should_trigger | 登录界面显示用户名、账户锁定 | `host-default-surface-convergence` | ✓ |
| 05 | should_not_trigger | 公网 RDP 只改端口够不够（兄弟诱饵） | `remote-access-surface-audit` | ✓ 正确转出 |
| 06 | should_not_trigger | Web 根目录与数据库文件（兄弟诱饵） | `web-minimal-exposure` | ✓ 正确转出 |
| 07 | should_not_trigger | 陌生进程连外网（跨书诱饵） | `none` | ✓ |
| 08 | edge_case | 域策略覆盖本地账户锁定 | `host-default-surface-convergence` | ✓ 只诊断覆盖 |
| 09 | edge_case | 已被入侵后再补加固 | `none` | ✓ |
| 10 | edge_case | macOS 是否套用同一设置 | `none` | ✓ 平台不符 |

## 失败与修复

首轮 9/10：`edge-01`（域策略覆盖本地账户锁定）被判为 `none`，理由是 description 中的“域集中策略设计”把“覆盖诊断”一起排除了。这是真实的 trigger 歧义，不是测试过严。
修复：把 description 的“域集中策略设计”改为“域策略设计（覆盖诊断除外）”，明确本 skill 只诊断覆盖、不设计域策略。修改后 desc 非空白字符 295，重测 10/10。

## 结论

接受，无需回炉重做阶段 2。本 skill 的 description 已按盲测暴露的 trigger 歧义修订，并完成复测。
