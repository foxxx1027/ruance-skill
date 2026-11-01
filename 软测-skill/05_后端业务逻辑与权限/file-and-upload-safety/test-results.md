# file-and-upload-safety — 阶段 4 压力测试结果

> 测试对象：`file-and-upload-safety/test-prompts.json`（7 条用例，darwin-skill 兼容格式）
> 测试方式：双盲测（两个独立评审 sub-agent，只读 13 个 skill 的 `description`，禁止打开 `test-prompts.json`）
> 测试时间：2026-10-24

## 结果总览

| 指标 | 值 |
|---|---|
| 用例总数 | 7 |
| 评审 A 严格匹配 | 7/7 |
| 评审 B 严格匹配 | 7/7 |
| 两评审一致 | 7/7 |
| 判定通过 | 7/7 = 100% |
| 阈值（`minimum_pass_rate`） | 80% |
| 结论 | **通过** |

## 逐条结果

| # | 类型 | Prompt（截断） | 期望命中 | 评审 A | 评审 B | 判定 |
|---|---|---|---|---|---|---|
| 1 | 正向 | 下载接口把 filename 直接拼进 new File(bas… | file-and-upload-safety | file-and-upload-safety | file-and-upload-safety | 通过 |
| 2 | 正向 | 头像上传只判断了扩展名是不是 jpg/png，评审说这样不行，还… | file-and-upload-safety | file-and-upload-safety | file-and-upload-safety | 通过 |
| 3 | 正向 | We unzip user-uploaded archives … | file-and-upload-safety | file-and-upload-safety | file-and-upload-safety | 通过 |
| 4 | 正向 | 导入功能要在 /tmp 下生成临时 Excel，用随机文件名，还… | file-and-upload-safety | file-and-upload-safety | file-and-upload-safety | 通过 |
| 5 | 诱饵/负向 | 对象存储的预签名 URL 换个 key 就能下别人的文件，权限该… | authorization-and-least-privilege | authorization-and-least-privilege | authorization-and-least-privilege | 通过 |
| 6 | 诱饵/负向 | 大文件直传对象存储经常超时，要不要改成分片上传？ | 不激活 | 不激活 | 不激活 | 通过 |
| 7 | 边界 | 图片走前端直传对象存储，服务端不落盘，是不是就不用校验文件类型了？ | file-and-upload-safety | file-and-upload-safety | file-and-upload-safety | 通过 |

## 失败与偏差分析

无。7 条用例全部一次通过，两个评审结论完全一致。

## 测试方法说明

1. **隔离**：把 13 个 skill 的 `description` 字段拼成候选清单交给两个独立 sub-agent，不给 `type`、`expected_behavior`、`notes`，也不允许读取 `test-prompts.json`。
2. **任务**：对每条 prompt 只回答「激活哪个 skill，或不激活任何 skill」并给出理由。
3. **判卷**：`should_trigger` 要求命中本 skill；`should_not_trigger` 的诱饵题要求命中指定的兄弟 skill，纯噪声题要求不激活任何安全 skill；`edge_case` 按 expected_behavior 判定。
4. **交叉校验**：两个评审的一致率用于发现「描述本身有歧义」的用例——一致但都判错说明是 skill 缺陷，不一致说明是 description 边界模糊。

## 结论

file-and-upload-safety 以 **7/7**（100%）通过阶段 4，高于 80% 阈值。skill 本身无需修改。
