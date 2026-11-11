# 压力测试结果 — buffer-overflow-defense-layering

- **测试方式**: 独立 sub-agent 盲测。向 sub-agent 提供本书全部 8 个 skill 的 name + description，隐藏 `type` / `expected_behavior` / `notes`，要求对每条 prompt 单选一个 skill 或 none。
- **测试集**: `test-prompts.json`（should_trigger 4 / should_not_trigger 3 / edge_case 3，共 10 条）
- **通过标准**: should_trigger 必须命中本 skill；should_not_trigger 容错为 0（尤其跨 skill 诱饵）；edge_case 需符合边界理由。
- **最终通过率**: **10/10 (100%)**

## 结果与失败分析

首轮 7/10：①「怎么防止攻击者拿到 shell 后继续操作」被判 none；②「加密 CMD 设了密码就安全吧」被判 none；③边缘用例「新服务器默认配置很安全，还要做溢出防御吗」被判 none。前两条暴露 description 未覆盖 post-exploitation 与「加固有效性验证」，已回填 description、A2 场景 6 与语言信号；第三条属测试设计问题，改为「补丁打全后是否还要限制 shell 后的操作」。复测 10/10，诱饵全部正确分流。

## 结论

达到 `minimum_pass_rate: 0.8`，接受。所有 should_not_trigger 诱饵均未误触发本 skill。
