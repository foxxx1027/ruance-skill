# 压力测试结果 — `lan-visibility-and-containment`

- **测试时间**：2026-10-19
- **测试方式**：独立 sub-agent 盲测。给 sub-agent 整包 8 个 skill 的 name + description，令其对每条 prompt 做“该激活哪一个 / none”的选择；不提供 type、expected_behavior、notes 与判卷标准。
- **判卷口径**：`should_trigger` 必须选中本 skill；`should_not_trigger` 不得选中本 skill（兄弟诱饵应转给同书另一个 skill，跨书诱饵应为 `none`）；`edge_case` 的判断需与边界理由一致。
- **结果**：**10/10（100%）**，达到 `minimum_pass_rate: 0.8`。

## 逐条结果

| # | 类型 | Prompt 摘要 | 盲测选择 | 判定 |
|---|---|---|---|---|
| 01 | should_trigger | 网速变慢、怀疑私接路由器 | `lan-visibility-and-containment` | ✓ |
| 02 | should_trigger | 无文档内网的设备与共享盘点 | `lan-visibility-and-containment` | ✓ |
| 03 | should_trigger | IP 冲突，只断一台 | `lan-visibility-and-containment` | ✓ |
| 04 | should_trigger | IP-MAC 绑定与关键主机组 | `lan-visibility-and-containment` | ✓ |
| 05 | should_not_trigger | WPA2 与 WPS（跨书诱饵） | `none` | ✓ |
| 06 | should_not_trigger | RDP 3389 要不要关（兄弟诱饵） | `remote-access-surface-audit` | ✓ 正确转出 |
| 07 | should_not_trigger | 从公网扫描网段（跨书诱饵） | `none` | ✓ |
| 08 | edge_case | 交换机与网管软件重复绑定 | `lan-visibility-and-containment` | ✓ 禁止叠加 |
| 09 | edge_case | 邻居家的网络为什么慢 | `none` | ✓ 未授权 |
| 10 | edge_case | 抓全网包看谁在攻击 | `lan-visibility-and-containment` | ✓ 改用镜像/NetFlow |

## 失败与修复

首轮 8/10：
- `edge-02`（邻居家网络）被选中本 skill，理由是“家庭小网络网速变慢排查”，没有识别出这是未授权网络。
- `edge-03`（抓全网包）被判 `none`，description 缺少该场景的信号词。
修复：description 的“未授权网络”补为“未授权网络（含邻居/他人网络）”，关键信号补入“全网监听”。修改后 desc 非空白字符 297，重测 10/10。

## 结论

接受，无需回炉重做阶段 2。本 skill 的 description 已按盲测暴露的 trigger 歧义修订，并完成复测。
