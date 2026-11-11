---
name: fuzzing-strategy
description: |
  要在 Fuzz 与静态审计间选路线，或面对复杂文件/协议决定 Blind Fuzz 还是 Smart Fuzz 时使用。不适用于：漏洞已定位只需利用（走 exploit-chain-construction）、崩溃回溯根因（走 crash-to-root-cause）、内核驱动 Fuzz（走 kernel-driver-fuzzing）、只做文件格式逆向（走 file-format-vulnerability-reversing）。trigger：Fuzz 选型 / Blind Fuzz / Smart Fuzz / 变异策略 / 样本变异 / SPIKE / Peach。
source_book: 《0day安全：软件漏洞分析技术（第二版）》 王清 等
source_chapter: 第 16 章（16.1–16.3）+ 第 17 章（17.1–17.3）+ 第 18 章（18.2、18.5）
tags: [Fuzz, SmartFuzz, BlindFuzz, 样本变异, 数据块, SPIKE, Peach, 漏洞挖掘]
related_skills:
  - slug: vuln-debugging-and-breakpoints
    relation: contrasts-with
  - slug: network-protocol-fuzzing
    relation: composes-with
  - slug: file-format-vulnerability-reversing
    relation: composes-with
  - slug: kernel-driver-fuzzing
    relation: composes-with
---

# Fuzz 范式选择与 Smart Fuzz

## R — 原文 (Reading)

> “通常 Smart Fuzz 包括三方面的特征：面向逻辑（Logic Oriented Fuzzing）、面向数据类型（Data Type Oriented Fuzzing）和基于样本（Sample Based Fuzzing）。”
>
> — 王清 等，《0day安全：软件漏洞分析技术（第二版）》第 17 章 17.1.2

---

## I — 方法论骨架 (Interpretation)

Fuzz 不是“随便灌脏数据”，而是一套关于**搜索空间怎么构造**的方法论。先选范式，再选用例生成方式，最后才选工具。

**第一层：动态 Fuzz 还是静态审计。** 动态 Fuzz 是黑盒、面向崩溃：用例是畸形数据，目标是快速找到“能 crash 的输入”；优点是误报少、能直接拿到可复现样本，缺点是永远不能证明“没有漏洞”——找到 100 个，仍可能有第 101 个。静态审计是白盒、面向覆盖：需要源码，追求找出所有缺陷，但误报多、且受可计算性限制，处理不了由动态输入决定的复杂分支与循环。选哪条路取决于目标：拿 0day、没有源码、只求找到一个真漏洞，选 Fuzz；有源码、要系统性减少缺陷，选静态审计。

**第二层：Blind Fuzz 还是 Smart Fuzz。** Blind Fuzz 在随机位置插入随机数据，适合结构简单的输入；面对 PPT、Word、PDF、压缩包这类多层嵌套、带长度/指针/CRC 关系的复杂格式，绝大多数用例会在第一层解析器就被丢掉，测不到深层逻辑。Smart Fuzz 用三个特征同时约束用例生成：**面向逻辑**——先明确要打第几层解析器、改动的深度与粒度，尽量不破坏字段依赖；**面向数据类型**——识别算术型、指针型、字符串型、特殊字符，按类型生成对应畸形值；**基于样本**——以合法文件为模板做局部变异。三者不互斥，好的 Smart Fuzz 方案应同时具备。

**第三层：基于数据块与“先解析、再畸形化”。** 协议/文件的数据单位是块，块可以平行或嵌套，块内常有“指示子块长度”的字段。SPIKE 用 `s_block_start`/`s_block_end` 定义块并自动回填长度，避免“改了数据却忘了改长度”导致用例在上层被拒。更进一步，先用 010 Editor 模板或 Peach DataModel 把文件解析成字段树，确认每个字段的类型与依赖，再只改目标字段——这就是“先解析、再定点畸形化”，比盲目 Fuzz 更有效。

**第四层：工具的角色。** SPIKE 提供块定义 API，Peach 提供 DataModel/StateModel/Monitor 与基于生长/基于变异两种模式，beSTORM 是商业黑盒测试器。工具适合做**初测与批量生成**，但工具跑完不等于结束：出现崩溃要接调试器确认可利用性，工具处理不了登录/加密/状态机时要自己写脚本深入。

判断一个 Fuzz 方案是否合格，可以问四个问题：目标解析深度是否明确？字段依赖（长度、CRC、指针）是否被保持？数据类型是否被区分？有没有崩溃监控与人工 triage？四问缺一，用例数量再多也可能只是“在浅层制造噪声”。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: PPT 文件的盲测与面向逻辑测试对比（第 17 章 17.1.2）
- **问题**: PPT 是 OLE 复合文档，外面有 OLE Parser，里面还有 PowerPoint 自己的多层解析逻辑；随机改字节的盲测能否打到深层？
- **方法论的使用**: 作者把同一批测试分成两组：Blind Fuzz 在随机位置插随机数据；面向逻辑的测试先明确目标解析层，再只改特定位置、尽量保持其他数据依赖。
- **结论**: 盲测生成的畸形文件绝大多数被第一层 OLE Parser 的异常处理捕获，只有极少数能进入更深层；面向逻辑的用例可以精确到达任意级别的解析器逻辑。
- **结果**: 对比图显示盲测大量用例停在 OLE 层，证明复杂格式必须用 Smart Fuzz，而不是靠增加盲测数量。

### 案例 2: 用 Peach 对 PNG 文件做 Smart Fuzz（第 17 章 17.2.5）
- **问题**: 需要验证 Peach 能否按 PNG 的 Chunk 结构生成畸形文件，并在目标解析时捕获异常。
- **方法论的使用**: 先按 PNG 规范写 DataModel：`Length` 用 `Relation type="size"` 绑定 `Data`，`CRC` 用 `Crc32Fixup` 绑定 `TypeAndData`，外层用 `minOccurs/maxOccurs` 描述多个 Chunk；再用 StateModel 写出畸形文件并调用 `pngcheck`/`explorer` 打开；最后配置 `WindowsDebugEngine` 与 `PageHeap` Monitor，用 WinDbg 捕获异常。
- **结论**: 把长度与 CRC 关系交给框架维护，畸形数据才能通过 PNG 结构校验；工具只负责生成与监视，崩溃分析仍要人工完成。
- **结果**: Peach 能持续生成 PNG 畸形文件并驱动目标打开；书中特别提示 Peach 与 WinDbg 6.12 不兼容，实验应固定使用 WinDbg 6.8。

### 案例 3: 用 010 Editor 解析 PPT 后定点修改字体字段（第 17 章 17.3.3）
- **问题**: 直接改 PPT 二进制容易破坏 OLE 结构，畸形数据到不了 PowerPoint 深层解析器；需要一种“只改目标字段、其他结构保持合法”的方法。
- **方法论的使用**: 先用 010 模板解析 PNG/PPT 结构，定位 `FontEntityAtom` 的 `IfFaceName` 字段；该字段是以 NULL 结尾、长度不超过 32 的字符串。作者把它改成 32 个 `0x1111`、去掉 NULL 终止符，另存为 `test.ppt`。
- **结论**: 修改必须建立在“知道字段类型与边界”的基础上；对字符串型字段，缺少终止符和超长是两类高价值畸形。
- **结果**: PowerPoint 能正常打开文件，但相关文字的字体名显示为乱码，说明畸形数据穿过了 OLE Parser，抵达了 PowerPoint 的字体解析逻辑。

### 案例 4: FTPFuzz 挖出 Quick 'n Easy FTP Server 远程拒绝服务（第 18 章 18.2）
- **问题**: 面对一个 FTP 服务，如何快速判断哪些命令/参数可能有问题？
- **方法论的使用**: 用 FTPFuzz 勾选 `USER`、`PASS`、`LIST` 等命令，配置脏数据列表（如 `..?`），让工具自动替换命令参数并发送；服务端崩溃后，用 OllyDbg 附加进程复现，观察 EAX 被 `2E2E3F2E`（`..?.`）覆盖。
- **结论**: 工具的强项是快速初筛和批量替换；它不负责根因分析，也不擅长处理需要登录/会话保持的场景。
- **结果**: 很快发现 `LIST` 命令参数超过 232 字节且以 `?` 结尾时服务端崩溃；书中明确说明 FTPFuzz 只作初步测试，之后要结合 Python 脚本深入。

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 面对一个二进制程序或文件解析器，纠结“用 Fuzz 还是读源码做静态审计”，需要判断两条路的能力边界与投入产出。
2. 已经决定做 Fuzz，但目标是 PPT/PDF/Office/压缩包这类复杂格式，盲测跑了很多用例却没有打到深层逻辑。
3. 手上有一个合法样本文件，需要设计变异策略：改哪些字段、按什么类型改、如何保持长度/CRC/指针关系。
4. 需要在 SPIKE、Peach、beSTORM 等工具之间选型，或判断某个工具是否支持面向逻辑/面向数据类型/基于样本。
5. Fuzz 跑出大量崩溃或完全没有崩溃，需要判断是“用例无效”“深度不够”“监控没接上”还是“目标确实没有浅层问题”。
6. 需要把工具产出的崩溃样本转成可分析的最小 PoC，或决定何时从工具切换到自写脚本。

### 语言信号 (用户的话里出现这些就应激活)

- “该用 Fuzz 还是静态代码审计 / 哪个更容易出 0day”
- “盲测跑了很多用例都没效果 / 复杂文件怎么 Fuzz”
- “Smart Fuzz 三特征是什么 / 面向逻辑、面向数据类型、基于样本怎么落地”
- “样本变异 / mutation / 怎么保持 length 和 CRC”
- “SPIKE 的 block 怎么定义 / Peach Pit 怎么写 / beSTORM 值不值得用”
- “fuzz 工具跑完没崩溃，是不是就没漏洞 / 崩溃了下一步怎么办”
- “fuzzing strategy / blind fuzzing / smart fuzzing / coverage-guided fuzzing”

### 与相邻 skill 的区分

- 与 `logic-and-isolation-bugs` 的区别：那个 skill 是“缺陷类型分析”，处理竞争条件、TOCTOU、类型混淆、沙箱逃逸，回答“这个现象算不算某类逻辑/隔离漏洞、怎么利用时间窗”；本 skill 是“发现策略选择”，回答“用什么方式生成输入、打到哪一层、如何批量找崩溃”。若已经有一个可疑的逻辑缺陷，走前者；若还不知道怎么找，走本 skill。
- 与 `vuln-analysis-strategy` 的区别：那个 skill 是跨范式的总选型（Fuzz、静态审计、补丁比较、逆向、调试分别适合什么目标），本 skill 只负责其中 Fuzz 分支的深化——Blind/Smart、样本/模型、工具与变异策略。若问题停留在“我该用哪种方法找漏洞”，先走 `vuln-analysis-strategy`；若已经决定用 Fuzz，直接用本 skill。
- 与 `file-format-vulnerability-reversing` 的区别：那个 skill 负责把一个具体文件格式解析清楚、定位解析入口与字段；本 skill 负责在解析结果之上设计畸形化策略。两者常组合：先逆向格式，再用本 skill 生成用例。
- 与 `network-protocol-fuzzing` 的区别：那个 skill 处理网络服务的命令清单、登录会话、接收 API 下断与入口转移；本 skill 处理通用的 Fuzz 范式与用例生成。目标是 FTP/SMTP 等服务时，用后者枚举攻击面，用本 skill 选 Blind/Smart 与工具。
- 与 `kernel-driver-fuzzing` 的区别：本 skill 不处理设备名、IOCTL 码、MITM 驱动调用链；内核对象走 `kernel-driver-fuzzing`。
- 与 `crash-to-root-cause` 的区别：本 skill 产出“可疑崩溃样本”，后者负责从崩溃回溯漏洞函数与根因；不要在本 skill 里直接跳到 exploit 构造。

---

## E — 可执行步骤 (Execution)

1. **先做范式选择：Fuzz 还是静态审计**
   - 明确目标（拿 0day/找一类漏洞/系统性减少缺陷）、是否有源码、时间与算力、是否需要可复现样本。
   - 完成标准: 写下一句选型理由：例如“无源码、只求快速找到一个真漏洞，选动态 Fuzz”；或“有源码、要覆盖全部输入路径，选静态审计并接受人工去误报”。
   - 判停条件: 若目标是“证明系统没有漏洞”，停止——Fuzz 与静态审计都给不出这种完备性证明。

2. **判断输入复杂度：Blind 还是 Smart**
   - 看格式是否有嵌套块、长度/指针/CRC 等字段依赖、多层解析器；简单扁平输入可先用 Blind，复杂格式直接上 Smart。
   - 完成标准: 能列出目标的前 2–3 层解析器，以及每层会做哪些结构校验。
   - 判停条件: 若盲测用例绝大多数在第一层就被拒，停止加量，转入 Smart Fuzz。

3. **构造合法样本与结构模型**
   - 准备一个包含尽可能多数据结构/区块的合法样本；用 010 模板、Peach DataModel 或协议规范把字段解析成树，标出类型、长度、偏移、校验和与依赖关系。
   - 完成标准: 模型能完整解释样本；改动任一字段后，框架能自动回填长度/CRC，或明确知道哪些依赖必须手工同步。
   - 判停条件: 若样本缺少大量区块，先补样本；基于样本的变异测不到样本里不存在的结构。

4. **确定测试深度、粒度与变异类型**
   - 明确要打哪一层解析器；每次只改目标字段，尽量不破坏其他依赖；按数据类型选变异：算术型（边界值、整数溢出）、指针型（NULL/非法地址）、字符串型（超长、缺终止符）、特殊字符（`# @ ' < > / \ ../`）。
   - 完成标准: 得到一张“目标层 → 字段 → 变异类型 → 预期畸形”的用例设计表，而不是“随机改几个字节”。
   - 判停条件: 若无法确定字段语义，先回到 `file-format-vulnerability-reversing` 做解析，不要盲目变异。

5. **配置运行器与崩溃监控**
   - 选择工具（SPIKE/Peach/beSTORM 或自写脚本），配置目标启动方式、输入投递方式、调试器/PageHeap 监控与日志；先跑基线，确认合法样本不会误报。
   - 完成标准: 合法样本稳定运行、畸形样本能触发并记录异常（寄存器、栈、崩溃地址），日志可追溯到具体用例。
   - 判停条件: 若监控接不上或工具与调试器版本不兼容（如 Peach 配 WinDbg 6.12），先修工具链，不要在没有监控的情况下批量跑。

6. **批量运行、去重与 triage**
   - 按设计表生成用例；对崩溃按崩溃地址/异常类型/输入特征去重；区分“解析器主动拒绝”“可控崩溃”“不可控崩溃”。
   - 完成标准: 得到一份去重后的崩溃清单，每条都有最小输入和复现步骤；能判断哪些值得进入根因分析。
   - 判停条件: 若连续多轮没有新增覆盖/新增崩溃，停止盲目加量，检查深度、样本覆盖与变异类型是否遗漏。

7. **从工具初测升级到定向深入**
   - 工具只负责初测；对可疑崩溃用调试器确认数据流，必要时写 Python/自定义脚本处理登录、加密、状态机或精确字段变异。
   - 完成标准: 能把“工具发现的崩溃”推进到“明确的处理函数/字段/越界点”，或明确判定为不可利用的拒绝服务。
   - 判停条件: 若需要 exploit 构造，转 `exploit-chain-construction`；若需要从崩溃回溯根因，转 `crash-to-root-cause`。

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- **已经拿到崩溃、需要定位漏洞函数或构造 exploit**：本 skill 只负责“怎么找到可疑崩溃”，不负责根因与利用；应走 `crash-to-root-cause` / `overflow-localization` / `exploit-chain-construction`。
- **目标是内核驱动/IOCTL**：设备名、控制码、MITM 调用链与用户态文件/协议 Fuzz 的约束不同，应走 `kernel-driver-fuzzing`。
- **目标需要登录、加密或复杂会话保持**：通用 Fuzz 工具会在认证/加密处失效；应先走 `network-protocol-fuzzing`，用脚本建立合法会话后再谈变异。
- **只需要解析一个文件格式、定位解析入口**：应走 `file-format-vulnerability-reversing`；本 skill 以“已能构造合法样本/模型”为前提。
- **有源码且目标是系统性消除缺陷**：静态审计/代码审查更合适；Fuzz 只能给出存在性证据，不能证明覆盖完整。
- **生产环境或没有崩溃隔离/快照的环境**：Fuzz 会频繁触发崩溃、挂起、文件锁；必须在虚拟机/隔离环境进行。

### 作者在书中警告的失败模式

- **对复杂格式做 Blind Fuzz**：绝大多数畸形文件被第一层 OLE Parser 拦截，投入大量用例却打不到深层逻辑（第 17 章 17.1.2）。
- **基于样本变异测不到样本外结构**：文件格式有 18 种 Chunk，样本只含 10 种，另外 8 种解析逻辑永远不会被触发；样本必须尽量包含全部结构（第 17 章 17.1.2）。
- **静态分析误报泛滥**：一次产生上千条警告，没人愿意逐条排查，真正的漏洞被淹没；必须有排序、去噪与人工复核（第 16 章 16.1）。
- **指望静态分析彻底读懂动态分支**：由运行时输入决定的复杂分支/循环会让静态分析漏报或近似，编译理论目前只到上下文无关文法（第 16 章 16.3）。
- **只改数据不改长度/CRC**：畸形数据在上层结构校验就被拒，根本到不了目标解析器；块式 Fuzz 与 DataModel 的意义就是自动维护这些依赖。
- **把工具输出当结论**：工具跑完只说明“这批用例没触发可监控异常”，不代表目标安全；还要看监控是否接上、深度是否足够、用例是否有效。
- **忽视工具链版本绑定**：Peach 与 WinDbg 6.12 不兼容会导致实验失败，书中明确要求固定 WinDbg 6.8；现代工具同样有版本/依赖约束。
- **在需要登录/加密的协议上硬套通用工具**：工具在认证环节就会失败，登录后的命令根本没被测到（第 18 章 18.2）。

### 作者的盲点 / 时代局限

- **2011 年的工具生态**：SPIKE 运行在 Linux、无完善文档、需读源码；Peach 使用 `phed.org/2008/Peach` 旧 XML 架构与 Python 2.5/2.6；beSTORM 是商业软件。书中没有覆盖现代覆盖率引导 Fuzz（AFL/libFuzzer/honggfuzz）、Sanitizer、符号执行/混合执行等范式。
- **Smart Fuzz 仍是人工启发式**：面向逻辑/数据类型/样本三特征依赖测试者手工建模，没有现代 coverage feedback、变异调度、语料最小化与自动 triage；用例质量高度依赖人的格式理解。
- **目标格式与平台偏旧**：案例集中在 PPT/OLE、PNG、PDF、旧版 Office/IE 与 32 位 Windows XP；现代文件格式、内存安全语言解析器、容器化服务与 64 位环境需要重新建模。
- **默认“崩溃=漏洞候选”**：书中主要靠调试器看异常，缺少可利用性评分、去重与根因自动归类；大量拒绝服务崩溃可能与真正可提权/可 RCE 的漏洞混在一起。
- **对静态审计的讨论偏简**：只给出“误报多、动态分支难”的边界，没有展开现代静态分析（抽象解释、符号执行、污点分析）的能力；读者容易把“静态审计”等同于旧式 grep/语法树扫描。

### 容易混淆的邻近方法论

- **`vuln-analysis-strategy`**：跨范式总选型；本 skill 是 Fuzz 分支的深化。先选“用不用 Fuzz”，再选“怎么 Fuzz”。
- **`file-format-vulnerability-reversing`**：解析格式、定位入口；本 skill 在解析结果上设计畸形用例。两者是“看懂结构”与“按结构打”的分工。
- **`network-protocol-fuzzing`**：协议命令枚举、登录会话、接收 API 下断；本 skill 提供通用 Fuzz 范式与工具选择。
- **`kernel-driver-fuzzing`**：内核对象/IOCTL/MITM；本 skill 不涉及 ring0。
- **`crash-to-root-cause`**：从崩溃回溯根因；本 skill 只产出崩溃候选。
- **`logic-and-isolation-bugs`**：竞争/类型混淆/沙箱逃逸的缺陷分析；本 skill 不判断具体缺陷类型，只负责搜索策略。

---

## 相关 skills

- **depends-on**: 无

- **contrasts-with**:
  - `vuln-debugging-and-breakpoints` — 一个主动生成输入找崩溃，一个被动观察已触发的崩溃

- **composes-with**:
  - `network-protocol-fuzzing` — 协议场景的落地
  - `file-format-vulnerability-reversing` — 文件格式场景的落地
  - `kernel-driver-fuzzing` — 内核 IOCTL 场景的落地

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: 待阶段 4 (详见 test-prompts.json)
- **蒸馏时间**: 2026-10-16
