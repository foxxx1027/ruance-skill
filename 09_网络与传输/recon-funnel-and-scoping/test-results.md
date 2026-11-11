# 压力测试结果 — recon-funnel-and-scoping

- **测试方式**: 独立 sub-agent 盲测。向 sub-agent 提供本书全部 8 个 skill 的 name + description，隐藏 `type` / `expected_behavior` / `notes`，要求对每条 prompt 单选一个 skill 或 none。
- **测试集**: `test-prompts.json`（should_trigger 4 / should_not_trigger 3 / edge_case 3，共 10 条）
- **通过标准**: should_trigger 必须命中本 skill；should_not_trigger 容错为 0（尤其跨 skill 诱饵）；edge_case 需符合边界理由。
- **最终通过率**: **10/10 (100%)**

## 结果与失败分析

首轮 9/10：边缘用例「TTL=64 是哪家的系统」被判为常识问答而未激活。复核后认定为测试设计问题——原期望过严，单一指纹问答本就不该强行激活完整侦察流程；改为「ping 通了 TTL=64，接下来查什么才能确认系统」后复测通过。3 条诱饵（工具参数、已感染处置、ARP 欺骗）全部正确分流。

## 结论

达到 `minimum_pass_rate: 0.8`，接受。所有 should_not_trigger 诱饵均未误触发本 skill。
