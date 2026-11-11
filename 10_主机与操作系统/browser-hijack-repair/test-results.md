# 压力测试结果 — `browser-hijack-repair`

- **测试时间**：2026-10-19
- **测试方式**：独立 sub-agent 盲测。给 sub-agent 整包 8 个 skill 的 name + description，令其对每条 prompt 做“该激活哪一个 / none”的选择；不提供 type、expected_behavior、notes 与判卷标准。
- **判卷口径**：`should_trigger` 必须选中本 skill；`should_not_trigger` 不得选中本 skill（兄弟诱饵应转给同书另一个 skill，跨书诱饵应为 `none`）；`edge_case` 的判断需与边界理由一致。
- **结果**：**10/10（100%）**，达到 `minimum_pass_rate: 0.8`。

## 逐条结果

| # | 类型 | Prompt 摘要 | 盲测选择 | 判定 |
|---|---|---|---|---|
| 01 | should_trigger | 主页改回后又变回去 | `browser-hijack-repair` | ✓ 先找写入源 |
| 02 | should_trigger | IE 右键菜单多出陌生链接 | `browser-hijack-repair` | ✓ |
| 03 | should_trigger | 标题栏广告 + 查看源文件消失 | `browser-hijack-repair` | ✓ |
| 04 | should_trigger | 公用电脑清理表单与密码 | `browser-hijack-repair` | ✓ |
| 05 | should_not_trigger | 证书警告是不是钓鱼站（跨书诱饵） | `none` | ✓ |
| 06 | should_not_trigger | 弹广告 + 陌生进程（跨书诱饵） | `none` | ✓ |
| 07 | should_not_trigger | 登录界面显示用户名（兄弟诱饵） | `host-default-surface-convergence` | ✓ 正确转出 |
| 08 | edge_case | 公司电脑主页被域策略锁定 | `none` | ✓ 不按劫持处理 |
| 09 | edge_case | 默认搜索引擎被换成 Bing | `browser-hijack-repair` | ✓ 先判断是否默认设置变更 |
| 10 | edge_case | 浏览器重置后仍跳转 | `browser-hijack-repair` | ✓ 重置不清写入源 |

## 失败与修复

无失败。`edge-02` 选择本 skill，理由为“需判断是否浏览器劫持”，与本 skill 第 1 步“认症状并记录复发行为”一致，按边界判定通过。

## 结论

接受，无需回炉重做阶段 2。本 skill 的 description 已按盲测暴露的 trigger 歧义修订，并完成复测。
