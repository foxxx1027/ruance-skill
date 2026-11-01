# 压力测试结果 — usb-autorun-hardening

- **测试方式**: 独立 sub-agent 盲测。向 sub-agent 提供本书全部 8 个 skill 的 name + description，隐藏 `type` / `expected_behavior` / `notes`，要求对每条 prompt 单选一个 skill 或 none。
- **测试集**: `test-prompts.json`（should_trigger 4 / should_not_trigger 3 / edge_case 3，共 10 条）
- **通过标准**: should_trigger 必须命中本 skill；should_not_trigger 容错为 0（尤其跨 skill 诱饵）；edge_case 需符合边界理由。
- **最终通过率**: **10/10 (100%)**

## 结果与失败分析

首轮 9/10：边缘用例「U 盘里的文件都变成快捷方式」被判给 malware-triage-order。复核后认定测试设计问题——该现象说明已经感染，处置确实应走 malware-triage-order，本 skill 的 B 段也如此界定；改为「根目录出现隐藏 autorun.inf 但还没打开过」后复测通过。3 条诱饵全部正确分流。

## 结论

达到 `minimum_pass_rate: 0.8`，接受。所有 should_not_trigger 诱饵均未误触发本 skill。
