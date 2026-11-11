# 压力测试结果 — `log-integrity-and-evidence`

- **测试时间**：2026-10-19
- **测试方式**：独立 sub-agent 盲测。给 sub-agent 整包 8 个 skill 的 name + description，令其对每条 prompt 做“该激活哪一个 / none”的选择；不提供 type、expected_behavior、notes 与判卷标准。
- **判卷口径**：`should_trigger` 必须选中本 skill；`should_not_trigger` 不得选中本 skill（兄弟诱饵应转给同书另一个 skill，跨书诱饵应为 `none`）；`edge_case` 的判断需与边界理由一致。
- **结果**：**10/10（100%）**，达到 `minimum_pass_rate: 0.8`。

## 逐条结果

| # | 类型 | Prompt 摘要 | 盲测选择 | 判定 |
|---|---|---|---|---|
| 01 | should_trigger | 事件查看器几乎为空 | `log-integrity-and-evidence` | ✓ |
| 02 | should_trigger | IIS 当天日志缺失 + 服务重启 | `log-integrity-and-evidence` | ✓ |
| 03 | should_trigger | 日志导出后换机可读 | `log-integrity-and-evidence` | ✓ |
| 04 | should_trigger | Eventlog 路径被重定位 | `log-integrity-and-evidence` | ✓ |
| 05 | should_not_trigger | 写批处理定时清理日志 | `none` | ✓ 修复后不再触发 |
| 06 | should_not_trigger | 网站 500 报错 | `none` | ✓ |
| 07 | should_not_trigger | 账户锁定设几次（兄弟诱饵） | `host-default-surface-convergence` | ✓ 正确转出 |
| 08 | edge_case | 日志已删还能否证明入侵 | `log-integrity-and-evidence` | ✓ 多源交叉 |
| 09 | edge_case | 已有 Splunk 是否还管本地日志 | `log-integrity-and-evidence` | ✓ 集中收集是补强 |
| 10 | edge_case | 只查某天谁登录过 | `log-integrity-and-evidence` | ✓ 按比例缩小 |

## 失败与修复

首轮 9/10：`should_not_trigger-01`（写批处理定时清理日志）被选中本 skill。根因是 description 只写了“清日志”是检测信号，没有明确“不提供清日志操作”。
修复：description 补入“也不提供清日志/删日志的操作步骤”。修改后 desc 非空白字符 250，重测 10/10。

## 结论

接受，无需回炉重做阶段 2。本 skill 的 description 已按盲测暴露的 trigger 歧义修订，并完成复测。
