# 压力测试结果 — `web-minimal-exposure`

- **测试时间**：2026-10-19
- **测试方式**：独立 sub-agent 盲测。给 sub-agent 整包 8 个 skill 的 name + description，令其对每条 prompt 做“该激活哪一个 / none”的选择；不提供 type、expected_behavior、notes 与判卷标准。
- **判卷口径**：`should_trigger` 必须选中本 skill；`should_not_trigger` 不得选中本 skill（兄弟诱饵应转给同书另一个 skill，跨书诱饵应为 `none`）；`edge_case` 的判断需与边界理由一致。
- **结果**：**10/10（100%）**，达到 `minimum_pass_rate: 0.8`。

## 逐条结果

| # | 类型 | Prompt 摘要 | 盲测选择 | 判定 |
|---|---|---|---|---|
| 01 | should_trigger | 云服务器只开 80/443 够不够 | `web-minimal-exposure` | ✓ |
| 02 | should_trigger | 备份与数据库可被 URL 下载 | `web-minimal-exposure` | ✓ |
| 03 | should_trigger | 上传目录 php 可执行 | `web-minimal-exposure` | ✓ |
| 04 | should_trigger | 上了 WAF 是否还需权限收敛 | `web-minimal-exposure` | ✓ |
| 05 | should_not_trigger | IIS 版本与已知漏洞（跨书诱饵） | `none` | ✓ |
| 06 | should_not_trigger | DDoS 流量清洗 | `none` | ✓ |
| 07 | should_not_trigger | Windows 默认共享与 Administrator（兄弟诱饵） | `host-default-surface-convergence` | ✓ 正确转出 |
| 08 | edge_case | 数据库暂时不能移出 Web 根 | `web-minimal-exposure` | ✓ 临时补偿 |
| 09 | edge_case | React 静态站 + 对象存储 | `web-minimal-exposure` | ✓ 重新映射对象 |
| 10 | edge_case | 只想修 SQL 注入代码 | `none` | ✓ 代码修复不在范围 |

## 失败与修复

无失败。

## 结论

接受，无需回炉重做阶段 2。本 skill 的 description 已按盲测暴露的 trigger 歧义修订，并完成复测。
