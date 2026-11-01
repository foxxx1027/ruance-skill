# 压力测试结果 — hidden-account-hunt

- **测试方式**: 独立 sub-agent 盲测。向 sub-agent 提供本书全部 8 个 skill 的 name + description，隐藏 `type` / `expected_behavior` / `notes`，要求对每条 prompt 单选一个 skill 或 none。
- **测试集**: `test-prompts.json`（should_trigger 4 / should_not_trigger 3 / edge_case 3，共 10 条）
- **通过标准**: should_trigger 必须命中本 skill；should_not_trigger 容错为 0（尤其跨 skill 诱饵）；edge_case 需符合边界理由。
- **最终通过率**: **10/10 (100%)**

## 结果与失败分析

首轮 8/10：①「Linux 上怎么查 UID 0 隐藏账号」被判 none，根因是 description 写成「不要把 Windows 路径照搬到 Linux」，被读成完全排除 Linux；已改为「可迁移多视图交叉思路，但换成 passwd/shadow/sudoers/last 等视图」。②边缘用例「离职账号忘了删」被判 none，属测试设计问题（普通账号卫生不在范围内），改为「账号被禁用后还能不能用它登录」。复测 10/10，3 条诱饵全部正确分流。

## 结论

达到 `minimum_pass_rate: 0.8`，接受。所有 should_not_trigger 诱饵均未误触发本 skill。
