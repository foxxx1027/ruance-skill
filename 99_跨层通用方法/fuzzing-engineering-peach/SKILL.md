---
name: fuzzing-engineering-peach
description: |
  当用户已经决定做模糊测试，需要筛选目标与输入入口、压缩种子集、编写或调试 Peach Pit，或对崩溃做 triage 时调用。适用于 PeachMinset 覆盖评分、!exploitable 复核、会话续跑。不适用于：还没决定是否 fuzz、静态审计/fuzz 选型、Peach/AFL 选型（走 fuzzing-strategy）；AFL 覆盖率停滞排查（走 coverage-guided-fuzzing）；无输入通道、强反 fuzz、纯逻辑漏洞。Trigger: Peach、Pit、PeachMinset、seed corpus、crash triage、!exploitable、模糊测试、种子最小集、崩溃分类。
source_book: 《灰帽黑客 第4版》
source_chapter: 第5章（5.2–5.5）
tags: [fuzzing, peach, seed-minimization, crash-triage, mutation-strategy]
related_skills:
  - slug: source-and-binary-static-audit
    relation: composes-with
  - slug: patch-diff-1day-workflow
    relation: composes-with
---

# 模糊测试工程化与 Peach 配置

## R — 原文 (Reading)

> "可按以下步骤开始进行模糊测试："
>
> "该过程可以分为两个部分：·采集每个数据样本的代码覆盖率的痕迹·计算用于模糊测试的最小数据样本集"
>
> — 第5章 5.4、5.4.3

---

## I — 方法论骨架 (Interpretation)

1. 先判"值不值得测"：目标是否支持多种差异大的输入、能否自动化驱动、复杂度与文档/格式数量是否高，三者共同决定漏洞概率与工程成本。
2. 再判"从哪个入口测"：枚举网络协议、命令行、文件等所有输入通道；效果等价时优先文件，因为网络层增加搭建与复现成本。
3. 决定"怎么造输入"：有可用样本且时间紧走变异式；有公开规范且需要结构化覆盖走生成式；两者可混合。变异内部再决定完全变异还是分片。
4. 决定"改哪些字节、用什么算法"：已知格式跳过魔法数与长度等固定字段；先用精选变异集合覆盖常见触发器，再上大规模随机。
5. 建种子池：先尽量多收集、多来源，不急于筛选；样本同质化会让后续最小集失去意义。
6. 算最小集：对每个目标程序单独跑覆盖跟踪（PeachMinset 两阶段），用绝对覆盖与唯一覆盖两指标评分；工具结果只作参考，独特样本人工保留。
7. 写 Pit：一般配置、数据模型、状态模型、代理/监控、测试配置五部分；未知格式先用单个 Blob 跑黑盒，再按实际实现细化模型，不必照抄规范。
8. 选策略并对齐时间预算：能跑完整轮才用顺序；默认确定性随机；时间不可预估用随机。只有确定性策略下用 skipto 续跑才有意义。
9. 跑会话并管住证据：记录迭代数、Faults 目录、可复现样本、调试日志；崩溃按类型分组，先看可利用/可能可利用。
10. 做 triage 再收尾：!exploitable 只用于排序，人工隔离影响寄存器的字节、逆向调用链、判断可控性，才决定修还是继续做利用。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: 实验5-1/5-2 — 为 VLC 准备并最小化 RM 样本集
- **问题**: 要对 VLC 的 RealMedia 解析做变异 fuzz，但手上没有样本，也不知道该保留哪些样本。
- **方法论的使用**: 用 Pilfer-Archive 从互联网档案馆按 rm/rmvb 抓取，收集约 20 个样本；放入 rm_samples 后运行 PeachMinset，以 `"C:\Program Files\VLC\vlc.exe" %s` 为目标逐一采集覆盖痕迹，再做覆盖分析。
- **结论**: 6 个样本全部保留（6 选 6）；最小集与目标程序绑定，换程序必须重算，且不能盲信工具评分。
- **结果**: 生成 minset 目录（1.rm–6.rm），直接作为实验5-3 的变异模板。

### 案例 2: 实验5-3 — 用 rm_fuzz.xml 跑通 Peach 会话
- **问题**: Pit 配置写错会让整轮 fuzz 无效，且会话中断后如何继续。
- **方法论的使用**: 把 `<Data fileName>` 从 rm_samples\*.rm 改为 minset\*.rm，配置 VLC 的 Executable/CommandLine；先 `Peach.exe -t` 验证，报错时用 `--debug` 定位到 XML 行列；正常后执行 `Peach.exe samples\rm_fuzz.xml Default`，中断后看 status.txt 并用 `--skipto` 续跑。
- **结论**: 配置错误必须在开跑前消灭；续跑只对确定性策略有意义，随机策略下续跑只是重复。
- **结果**: Faults 目录按崩溃类型分组，保存可复现样本、调试报告与原始模板路径。

### 案例 3: msvcrt!strlen 崩溃 — Unknown 不等于不可利用
- **问题**: fuzz 产出一个访问违例，!exploitable 分类为 Unknown，是否应丢弃。
- **方法论的使用**: 读 WinDbg 日志：c0000005、eip 在 msvcrt!strlen+0x20、`mov eax,[ecx]`、ecx 指向无效内存；确认 strlen 参数被破坏后，隔离样本中影响 ecx 的字节、确定 ecx 可取值、逆向调用 strlen 的函数。
- **结论**: 工具分类只是线索；该崩溃"可能无法利用"，必须完成三项人工核查才能定性。
- **结果**: 书中未给出最终利用结论，明确反对把 Unknown 当结论。

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 手上有一批文件样本或协议抓包，想把 fuzz 跑成一轮可复现、可续跑、可解释的会话。
2. 样本太多或会话太慢，想知道怎么筛种子、怎么给模板评分、换目标后要不要重算最小集。
3. 要写或改 Peach Pit，不确定数据模型、状态模型、代理监控、测试配置各放什么，或者 Pit 一跑就报错。
4. fuzz 出了崩溃，想知道先看哪些、!exploitable 的 Exploitable/Unknown 该怎么用、哪些必须人工复核。
5. 时间预算有限（一周/一夜），需要在顺序、确定性随机、随机三种策略间做取舍。
6. 还没决定要不要 fuzz，或在静态审计与 fuzz、Peach 与 AFL 之间选型——此时不激活本 skill，先走 `fuzzing-strategy`。

### 语言信号 (用户的话里出现这些就应激活)

- "跑一轮 fuzz 总跑不完 / 时间不够"
- "种子太多了，怎么选最小集 / 样本怎么评分"
- "PeachMinset 结果能直接信吗"
- "Pit / rm_fuzz.xml 怎么写，报错了"
- "这个崩溃 !exploitable 说是 Unknown，还要不要看"
- "该用变异还是生成 / 完全变异还是分片"

### 与相邻 skill 的区分


- **composes-with** `source-and-binary-static-audit`：本 skill 负责种子、Pit、会话与崩溃 triage；目标内部结构、危险点与格式线索由它先给出。
- **composes-with** `patch-diff-1day-workflow`：本 skill 产出崩溃样本与可达路径；已有补丁时由它正推根因，避免重复 fuzz。
- 与 `fuzzing-strategy` 的区别: 后者负责在 Fuzz 与静态审计、Blind 与 Smart 之间选路线；本 skill 是在已经决定 fuzz 之后，把目标、种子、Pit、策略、triage 落成工程参数。
- 与 `coverage-guided-fuzzing` 的区别: 后者讲覆盖率反馈驱动的 fuzz 闭环（AFL 类、种子库、崩溃去重与可利用性判定）；本 skill 用 Peach 的覆盖跟踪做模板评分与最小集，重点是配置与会话管理，不重讲反馈算法。
- 与 `file-format-vulnerability-reversing` 的区别: 后者从畸形样本反推格式、构造最小 POC；本 skill 假设样本或规范已可用，直接建模并跑会话，不承担格式逆向。

---

## E — 可执行步骤 (Execution)

当 skill 被激活后，agent 应按以下步骤执行:

1. **确认授权与边界**
   - 完成标准: 写下目标、授权范围、输入通道、可用样本、时间预算与停止条件。
   - 判停条件: 无授权或目标不可执行 → 停止，不进入技术步骤。

2. **枚举输入通道，选最先测的入口**
   - 完成标准: 列出至少一个可脚本化入口，并记录驱动命令与参数。
   - 判停条件: 网络协议不可达或难以稳定复现时，改文件输入或本地重放；连本地重放都做不到 → 停。

3. **给候选目标排序**
   - 完成标准: 按"多输入类型 / 易自动化 / 高复杂性"三项给出排序与理由。
   - 判停条件: 目标已被委托指定 → 跳过排序，直接进入入口选择。

4. **选变异式或生成式，并定变异片段与算法**
   - 完成标准: 明确数据来源（样本或规范）、变异片段（完全/分片）、算法组合（集合+随机）。
   - 判停条件: 时间预算不足时改用确定性、基于集合的策略，不做大规模随机；私有格式无规范 → 先用单 Blob 黑盒，不先逆向建模。

5. **采集多样本**
   - 完成标准: 样本来自至少两个独立来源，覆盖不同子格式或版本，并记录来源。
   - 判停条件: 完全拿不到样本 → 转生成式（需规范）；规范也没有 → 停，先走格式逆向。

6. **计算最小集**
   - 完成标准: 对当前目标程序跑完覆盖跟踪，得到 minset，并记录被剔除样本及理由。
   - 判停条件: 无法采集覆盖（不可插桩或无调试信息）→ 退回按结构、来源、版本人工分层抽样。

7. **编写或修改 Pit**
   - 完成标准: 五要素齐全；`Peach.exe -t` 输出 No Errors Found；能生成第一条测试用例。
   - 判停条件: 报错 → 用 --debug 或 PeachValidator 定位到行列；未通过验证不得开跑。

8. **按时间预算选策略并记录续跑方式**
   - 完成标准: 策略与预算匹配；若选随机策略，明确不依赖 --skipto。
   - 判停条件: 时间不可预估 → 禁用顺序策略；只有确定性策略才允许续跑。

9. **运行会话并保存证据**
   - 完成标准: status.txt、Faults 目录、可复现样本、调试日志齐全；崩溃已按类型分组。
   - 判停条件: 长时间零崩溃且迭代数虚高 → 回查魔法数或固定字段是否被破坏，回到步骤 4。

10. **崩溃 triage 与移交**
    - 完成标准: 每个崩溃都有"工具分类 + 人工结论 + 是否可控/是否可复现"记录。
    - 判停条件: 需要构造利用 → 移交 `exploit-chain-construction`；需要定位根因 → 移交 `crash-to-root-cause`；Unknown 不得直接丢弃。

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 目标没有可达、可脚本化的输入通道: 纯 GUI、依赖人工交互或物理设备时 fuzzer 无法驱动；先解决驱动问题，否则种子、Pit、triage 都无从落地。
- 目标有强反 fuzz 且尚未处理: 检测调试器或虚拟机、异常即退出、输入被加密或校验；应先处理反调试或脱壳，再回到本流程。
- 只排查逻辑、权限或业务漏洞: 越权、IDOR、支付逻辑等不会以崩溃形式出现，本 skill 的覆盖与 !exploitable 体系帮不上。
- 漏洞已定位、只需构造利用链: 直接走 `exploit-chain-construction`，不必再搭 fuzz 工程。
- 还没决定是否 fuzz，或在静态审计与 fuzz、Peach 与 AFL 之间选工具: 这是技术选型，走 `fuzzing-strategy`；本 skill 只承接已决定 fuzz 之后的工程参数。
- AFL 覆盖率停滞、反馈闭环或种子质量排查: 走 `coverage-guided-fuzzing`；本 skill 的 PeachMinset 只做模板评分与最小集。

### 作者在书中警告的失败模式

- ce28 模板越多越好: 不做覆盖去重，相似样本反复触发同一路径，测试周期拉长而覆盖不增。
- ce29 纯随机变异的组合爆炸: 不先跑基于集合的触发器，长时间零产出。
- ce30 变异破坏魔法数: 完全变异把文件头也改掉，程序在入口拒绝，覆盖为零。
- ce31/ce32 策略与续跑错配: 时间不可控仍用顺序策略；随机策略下用 --skipto 续跑。
- ce33/ce34 盲信工具: 把 PeachMinset 当裁决删掉独特样本；把 !exploitable 的 Unknown 或 Exploitable 当最终结论。
- ce35/ce36 建模失误: 严格照规范而忽略实现扩展；私有格式直接建模，预算耗尽仍未开跑。

### 作者的盲点 / 时代局限

- Peach 2.x/3.x 语境: 书中命令（`Peach.exe -t`、`--debug`、`--skipto`、Default 发布器、PeachMinset、LocalAgent）与目录结构属于该版本；后续版本或社区分支参数已变，不能照抄。
- Windows 工具链依赖: WinDbg、!exploitable、PageHeap、VLC 2.x、Python 2.7、Pilfer-Archive 均为 2015 年前后环境；今天常用 AFL++、libFuzzer、honggfuzz、ASAN 与集群化 triage 替代或补充。
- 只讲单机、手工会话: 没有 CI/CD、并行 fuzz、云端种子库与持续 triage；这套流程适合学习参数含义，不宜直接当现代生产流水线。
- 覆盖率只按基本块或函数执行量计: 未讨论边覆盖、路径爆炸与状态化协议 fuzz 的现代反馈机制。

### 容易混淆的邻近方法论

- `fuzzing-strategy`: 选"要不要 fuzz、用 Blind 还是 Smart"；本 skill 选"fuzz 内部的工程参数"。
- `coverage-guided-fuzzing`: 以覆盖率反馈驱动变异与去重；本 skill 用覆盖只做种子评分与最小化。
- `file-format-vulnerability-reversing`: 先逆向格式再构造 POC；本 skill 假设输入模型已可描述。
- `crash-to-root-cause` 与 `exploit-chain-construction`: 一个管崩溃根因，一个管利用链；本 skill 只做到 triage 与移交。

---

## 相关 skills

- **composes-with** `source-and-binary-static-audit` — 静态审计给出攻击面与输入入口，fuzz 用来验证可达性
- **composes-with** `patch-diff-1day-workflow` — 补丁差异可作为 fuzz 的定向输入

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: 待阶段 4（详见 test-prompts.json）
- **蒸馏时间**: 2026-10-19
