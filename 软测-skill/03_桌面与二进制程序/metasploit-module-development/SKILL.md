---
name: metasploit-module-development
description: |
  要把自用 PoC 改成可复用、可交付的 Metasploit 模块时使用；包括多目标/交付、payload 复用。本 skill 用导弹模型拆运载与弹头，按 initialize/exploit 两方法填空，补齐 Targets、Space、BadChars 与元数据。不适用于：一次性验证、根因或偏移未定（走 crash-to-root-cause / win-stack-overflow-exploit）、需现代 MSF6 语法。trigger：Metasploit 模块 / MSF module / 写 exploit 模块 / POC 交付 / Targets。
source_book: 《0day安全：软件漏洞分析技术（第二版）》 王清 等
source_chapter: 第 4 章（4.1、4.5–4.7）
tags: [Metasploit, 模块化, POC交付, payload, Targets, Ruby]
related_skills:
  - slug: win-stack-overflow-exploit
    relation: depends-on
  - slug: shellcode-engineering
    relation: depends-on
  - slug: shellcode-engineering
    relation: contrasts-with
  - slug: fuzzing-strategy
    relation: composes-with
---

# Metasploit 模块化开发

## R — 原文 (Reading)

> “它对漏洞利用的几个相对独立的过程进行了很好的封装，把一次入侵攻击简化为对若干个模块的选择与组装，就像好像发射导弹一样。”
>
> — 王清 等，《0day安全：软件漏洞分析技术（第二版）》第 4 章 4.1

---

## I — 方法论骨架 (Interpretation)

这套方法解决的不是“怎么发现漏洞”，而是**“一个已经能跑通的利用，怎么变成别人能复用、能换目标、能换载荷的工程件”**。

作者先做**环节分解**：一次漏洞利用可以拆成四个相对独立的过程——触发漏洞（缓冲区多大、第几个字节覆盖返回地址、怎么植入）、选取 shellcode（决定利用性质）、设定重要参数（IP、端口、弹窗内容、跳转地址）、选用编解码算法（适配受限字符与特征）。只有承认这四件事彼此解耦，才谈得上平台化。

然后用**导弹发射模型**做类比：漏洞触发是引爆装置，shellcode 是弹头，IP/端口是目标设定，编码器是躲避拦截的措施。导弹可以“对任意目标、任意航线、任意弹头”分别生产再组装，利用也可以。这个类比的真正价值是**划清运载与弹头的边界**：写 exploit 的人只关心怎么把 EIP 送到 payload 起点，不关心弹头是弹框还是开端口。

落到 MSF 模块，结构被压成**两个方法的填空题**：`initialize` 填平台、各操作系统对应的返回地址 `Targets`、payload 的 `Space`/`BadChars` 约束、描述/作者/引用等元数据；`exploit` 负责组装——把填充物、`[target['Ret']].pack('V')` 和 `payload.encoded` 拼成攻击串并发送，最后 `handler`。开发从“从零搭代码”降级为“按模板填参数”。

最后是**交付标准**：POC 不是一串十六进制，而是知识载体。要补 `Version`、`Author`、`License`、`Description`、`Arch`、`References`、`DefaultOptions`，让完全不了解漏洞的人不用自己调试就能看懂溢出点、缓冲区大小、跳转地址与 shellcode 作用。作者甚至用“框架选型”复盘说明：先立硬指标（可扩展、面向对象），再决软偏好（团队写得顺手），非技术因素也可以是正当依据。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: 用 MSF 远程入侵 Windows（MS06-040，第 4 章 4.2）
- **问题**: 需要一个真实远程漏洞来演示“框架如何把入侵简化为模块选择与组装”，而不是从零写 RPC exploit。
- **方法论的使用**: 在 MSF 中搜索 `netapi32` 相关模块，选中 MS06-040（CVE-2006-3439，`NetpwPathCanonicalize()` 的栈溢出，可经 RPC 远程调用）；选 Target（Windows 2000 SP0–SP4）；选 payload `windows/adduser`（在目标机建立管理员账号）；填 `RHOST`/`LHOST`；执行 exploit。
- **结论**: 一旦漏洞被封装成模块，一次入侵就退化为“选漏洞、选目标、选弹头、填参数”四步；这正是导弹模型所指的“模块选择与组装”。
- **结果**: 目标主机的账户管理中出现新添加的管理员账号，远程入侵成功。

### 案例 2: 用 MSF 生成可直接用的编码 shellcode（第 4 章 4.3）
- **问题**: 第 3 章手写一个跨版本通用 shellcode 需要大量汇编、hash 选型与调试，成本很高；这里只需要一个能弹计算器的 payload。
- **方法论的使用**: 打开 Payloads，选 `Windows Execute Command`；配置 `EXITFUNC=process`、`CMD=calc`、`Restricted Characters`（默认禁止 `0x00`）、`Encoder`（默认 `x86/shikata_ga_nai`）、`Format=C`；点 Generate 导出；把结果放进 `shellcode_loader` 运行。
- **结论**: 编码、自解码头、格式导出都由框架代劳，开发者只需给出“要做什么、不能出现什么字节、导出成什么格式”这三类约束。
- **结果**: 得到带解码指令的通用 shellcode，目标机上成功弹出计算器。

### 案例 3: 用 msfpescan 扫描 kernel32.dll 里的“跳板”（第 4 章 4.4）
- **问题**: 第 3 章找 `jmp esp` 要靠 OllyDbg 插件或自己写搜索程序，重复且易错。
- **方法论的使用**: 执行 `msfpescan -f -j ecx c:/windows/system32/kernel32.dll`，以 PE 文件为输入扫描 `jump/call` 类指令，并直接把地址转成 VA 显示；也可用 `-p` 找 `pop/pop/ret` 组合。
- **结论**: 跳板搜索是可以标准化的工具动作，应交给框架，而不是每个 exploit 重写一遍。
- **结果**: 直接得到 kernel32.dll 中 `jmp ecx` 指令的 VA 列表，替代了手工搜索流程。

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 手上有一个自己写的 PoC（脚本或 C 程序），队友/客户要拿去复现，但对方看不懂那串十六进制，也不知道换系统要改哪里。
2. 同一个漏洞要覆盖多个 Windows 版本/补丁，返回地址各不相同，需要一个 `Targets` 列表而不是复制多份脚本。
3. 想把触发逻辑与 payload 解耦：同一套触发代码要能换弹框、加账号、反弹 shell，不想每次都改攻击串。
4. 团队已经用 MSF，要求新漏洞以模块形式提交，能在 `msfconsole` 里 `use`、`info`、`set`、`exploit`。
5. 在“直接写一次性脚本”与“投入时间做模块化”之间做取舍，需要判断复用频率、交付对象与维护成本。

### 语言信号 (用户的话里出现这些就应激活)

- “把这个 PoC 做成 Metasploit 模块 / 写个 MSF exploit”
- “怎么交付 POC / 怎么让别人复现”
- “一个模块支持多个 Windows 版本 / Targets 怎么写”
- “payload 怎么复用 / 怎么换 shellcode”
- “initialize 和 exploit 两个方法怎么填”
- “Metasploit module / MSF exploit module / POC delivery / payload reuse”

### 与相邻 skill 的区分

- 与 `crash-to-root-cause` 的区别：那个 skill 回答“漏洞根因、溢出点、偏移在哪”，产出的是诊断结论；本 skill 假定这些已经确定，回答“怎么把它封装成可交付模块”。若连 `Ret` 偏移都没测出来，先走 `crash-to-root-cause` / `win-stack-overflow-exploit`。
- 与 `exploit-chain-construction` 的区别：那个 skill 处理“触发→信息泄露→绕过保护→控流”的缺环，重点在 ASLR/DEP 下的稳定性；本 skill 只做工程封装，不解决“模块里的硬编码 Ret 在 ASLR 下失效”这一根本问题。若目标有 ASLR/DEP，先补链，再谈模块化。
- 与 `win-stack-overflow-exploit` 的区别：后者提供“怎么算偏移、选跳板、布置三段式”的具体构造方法，是模块 `exploit` 方法里那几行代码的知识来源；本 skill 提供的是外层骨架、参数化与交付标准。
- 与 `shellcode-engineering` 的区别：后者是手写/改造弹头的工程方法；本 skill 只负责“选弹头、给约束、让框架编码”。若现成 payload 不满足需求，再回到 `shellcode-engineering`。

---

## E — 可执行步骤 (Execution)

1. **判断是否值得模块化**
   - 看三个信号：是否会重复使用、是否要交付给不熟悉该漏洞的人、是否需要多目标/多 payload。
   - 完成标准: 能明确说出至少一个复用或交付对象；若答案是“只跑一次看看”，停止本流程，直接写一次性脚本。
   - 判停条件: 若漏洞触发条件本身还没稳定复现，先回去做定位与构造，不要用模块骨架掩盖不确定性。

2. **把利用拆成四个相对独立的过程**
   - 分别写清：触发（缓冲区大小、覆盖偏移、植入方式）、payload（要做什么）、参数（IP/端口/返回地址/弹窗内容）、编码（禁用字节、编码器）。
   - 完成标准: 四项各有明确取值或来源，且能指出哪些是“每个目标不同”、哪些是“全目标相同”。

3. **搭最小模块骨架**
   - 继承 `Msf::Exploit::Remote`（或 `Local`），按协议 `include` 对应 mixin；只实现 `initialize` 与 `exploit` 两个方法，先让模块能被加载。
   - 完成标准: `msfconsole` 能加载模块，`info` 能显示名称、平台、Targets、Payload 约束，且无 Ruby 语法错误。

4. **填 `initialize`：把差异参数化**
   - 填 `Name`、`Platform`、`Targets`（每个 OS/补丁一组 `Ret`）、`Payload` 的 `Space` 与 `BadChars`、`Arch`；再补 `Author`、`References`。
   - 完成标准: 切换 Target 时返回地址随之变化；MSF 能按 `Platform` 自动筛选可用 payload；`BadChars` 与真实入口限制一致。
   - 判停条件: 若某个 Target 的 `Ret` 依赖 ASLR 且模块没有 leak 能力，停止硬编码，转 `exploit-chain-construction`。

5. **填 `exploit`：组装并发送**
   - `connect` → 按“填充物 + `[target['Ret']].pack('V')` + `payload.encoded`”拼 `attack_buf` → 发送 → `handler` → `disconnect`；把协议细节交给 mixin。
   - 完成标准: 至少在一个 Target 上稳定触发并拿到 session/执行预期命令；换 Target 只需改配置，不改代码。
   - 判停条件: 若攻击串长度超过 `Space` 或含 `BadChars`，先调布局/换编码器，而不是把约束改宽。

6. **按“陌生读者可复现”标准发布**
   - 补 `Version`、`License`、`Description`（溢出点、缓冲区大小、布局、平台差异）、`References`（URL/CVE）、`DefaultOptions`；用没参与调试的人做一次盲测。
   - 完成标准: 对方仅凭模块元数据与说明，不读原始调试记录也能复现；能说清换 OS/补丁要改哪一项。

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- **只做一次性验证或单机复现**：模块化的收益来自复用与交付；一次性脚本更快、更直接，强行上框架只会增加维护面。
- **漏洞根因、偏移或触发条件尚未确定**：模块骨架不能替代调试；`Ret` 填错只会得到一个“看起来专业但打不通”的模块。先走 `crash-to-root-cause` / `win-stack-overflow-exploit`。
- **目标启用 ASLR/DEP，模块没有信息泄露或保护绕过能力**：`Targets` 里硬编码的返回地址会随机化，`payload.encoded` 也可能落在不可执行页；应先走 `exploit-chain-construction` / `dep-aslr-bypass`。
- **团队/目标平台不使用 Ruby 或 MSF**：书中方案绑定 MSF 3.x 与 Ruby 生态；若团队用 Python 或自研框架，应迁移“四过程拆分 + 两方法填空 + 元数据交付”的结构，而不是照搬代码。
- **需要现代 MSF6 模块语义**：书中是 MSF 3.4 的写法，现代模块还涉及 `register_options`、`check`、`FailWith`、`AutoCheck`、`Notes` 等；直接照抄会过时。

### 作者在书中警告的失败模式

- **裸十六进制 POC 无法传递知识**：很多 C 语言 POC 只向目标发送一堆十六进制数据；不自己调试就看不出溢出点、缓冲区大小、shellcode 作用、缓冲区组织方式，以及换操作系统该改哪里。交付物应是“含完整元数据的模块”，不是数据串。
- **返回地址硬编码且不随目标变化**：书中明确提醒“测试脚本中的跳转指令地址可能需要根据实验平台重新确定”；把某台机器的 `Ret` 当成通用值，换 Target 即失效。
- **把某个 MSF 版本的 payload 当成永久可用**：作者提醒 MSF 3.0 以后部分 shellcode 经过调整，实验中可能失效，遇到就换其他 payload；模块应对 payload 变化保持可替换性。
- **只做“选择与组装”、不理解底层机制**：框架降低了门槛，但不理解偏移、布局与编码约束的人无法判断模块为何失败，也无法为新漏洞写出正确的 `Targets` 与 `BadChars`。

### 作者的盲点 / 时代局限

- **绑定 MSF 3.4 / Ruby 1.8 时代的 API**：现代 MSF6 的模块生命周期、选项注册、检查逻辑、数据库与会话管理都已演进；书中的 `def exploit` + 直接 `connect` 只是最小骨架，不是现代模块规范。
- **默认 32 位 Windows 2000/XP、无 ASLR/DEP 的实验语境**：`Targets` 里放几个固定 `Ret` 就能工作；现代目标需要 leak、ROP、`AutoCheck` 与版本指纹，模块结构必须为此扩展。
- **“框架优先”带有明显的工具乐观主义**：作者用“傻瓜相机”比喻强调门槛降低，但没有充分讨论依赖锁定、供应链可信度、模块可审计性与“框架失败时如何手工复核”；这些在现代红队工程里是硬问题。
- **发布元数据标准偏薄**：书中要求 `Version/Author/License/Description/References` 已属超前，但现代交付还需要影响版本范围、披露时间线、复现环境、检测建议与回滚/清理说明。

### 容易混淆的邻近方法论

- **`win-stack-overflow-exploit`**：它教“怎么算出偏移、选跳板、布置三段式”，是模块 `exploit` 方法里的具体施工知识；本 skill 教“怎么把施工封装成可复用件”。
- **`shellcode-engineering`**：它教“怎么手写/改造弹头”；本 skill 教“怎么选弹头、给约束、让框架编码”。两者是“造弹头”与“装弹头”的分工。
- **`exploit-chain-construction`**：它处理保护绕过与稳定性，是模块能否在现代目标上工作的前提；本 skill 不解决“Ret 被 ASLR 随机化”这类链缺口。
- **`crash-to-root-cause`**：它产出根因与偏移，是模块参数的来源；不要用“模块能加载”替代“漏洞已定位”。

---

## 相关 skills

- **depends-on**:
  - `win-stack-overflow-exploit` — 模块的 payload 偏移来自已量出的栈布局
  - `shellcode-engineering` — stager/loader 的弹头来自 shellcode 工程

- **contrasts-with**:
  - `shellcode-engineering` — 一个关注模块封装与交付，一个关注机器码本身

- **composes-with**:
  - `fuzzing-strategy` — Fuzz 出的崩溃可快速落成可复现模块

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: 待阶段 4 (详见 test-prompts.json)
- **蒸馏时间**: 2026-10-16
