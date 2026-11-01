---
name: kernel-driver-fuzzing
description: |
  做驱动 IOCTL Fuzz、判断用 MITM 还是 Driver Fuzz、设计设备名/控制码/参数策略时用；也包括只有 .sys 不知设备名、MITM 覆盖不全、按 Method 位裁剪参数面、Fuzz 项在线增删改。不适用于：网络协议 Fuzz（network-protocol-fuzzing）、通用 Fuzz 选型（fuzzing-strategy）、崩溃后利用（kernel-driver-exploitation）。trigger：内核 Fuzz / IOCTL Fuzz / MITM Fuzz / Driver Fuzz / DeviceIoControl / Method 位。
source_book: 《0day安全：软件漏洞分析技术（第二版）》 王清 等
source_chapter: 第 23 章（23.1–23.4）
tags: [内核Fuzz, IOCTL, MITM-Fuzz, Driver-Fuzz, 设备名, IoControlCode, Method位, Fuzz项]
related_skills:
  - slug: fuzzing-strategy
    relation: depends-on
  - slug: network-protocol-fuzzing
    relation: contrasts-with
  - slug: kernel-driver-exploitation
    relation: contrasts-with
  - slug: kernel-driver-exploitation
    relation: composes-with
---

# 内核驱动 Fuzz

## R — 原文 (Reading)

> “Fuzz 对象——是指被 Fuzz 的对象，或触发 Fuzz 的条件；Fuzz 策略——是指针对 Fuzz 对象，Fuzz 过程中对参数和数据畸形化的方案；Fuzz 项——Fuzz 对象和 Fuzz 策略合起来，称为一个 Fuzz 项。”
>
> — 王清 等，《0day安全：软件漏洞分析技术（第二版）》第 23 章 23.3.1

---

## I — 方法论骨架 (Interpretation)

内核 Fuzz 的第一步不是选工具，而是选入口。Ring3 能触达 Ring0 的面有四类：SSDT / Shadow SSDT 的系统服务、安全软件对内核 API 的 Hook/Inline Hook、网络协议栈的报文处理、以及 IoControl 派遣例程。公布漏洞最多的是 IoControl，尤其第三方驱动，因此它是默认主战场。

IoControl Fuzz 有两种模式。MITM 在核内 Hook `NtDeviceIoControlFile`，识别目标进程/设备/控制码后篡改参数，再转发给原函数——它借的是真实调用链，能进入 GUI 才能触发的深层逻辑，但依赖目标程序主动发起调用，覆盖不全，而且每个控制码往往只被篡改“一次性”。Driver Fuzz 反过来：Fuzzer 自己打开设备，遍历控制码与参数组合，主动、可系统组合，但前提是知道设备名与控制码。

工具配置抽象成三要素：Fuzz 对象是“何时命中”（MITM 用进程/设备/控制码，任一为空表示通配；Driver Fuzz 用设备名 + 控制码，两者都不能为空），Fuzz 策略是“命中后怎么改”（地址/数据/长度分别取不变、随机、非法、合法 Ring3、合法 Ring0 等），Fuzz 项 = 对象 + 策略。工具要支持多项并行与在线增删改，不能每改一次策略就重启引擎。

参数面由 Method 位裁剪：非 NEITHER 时输入输出都被系统缓冲或锁定，改地址没有意义，只 Fuzz 数据与长度；NEITHER 时驱动可能直接解引用用户地址，才增加输入地址、输出地址两个维度。这是“先判机制、再选测试面”，避免全参数组合爆炸。

Fuzz 只负责把系统打蓝；蓝屏转储、栈回溯与驱动偏移才把它变成可归因的漏洞。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: 超级巡警 ASTDriver.sys 的任意地址写 0（第 23 章 23.4.1）
- **问题**: 超级巡警 v4 Build0316 的 ASTDriver.sys 是否会被本地程序打出任意写？设备名与控制码都未知。
- **方法论的使用**: 启动 ast.exe，用 MITM Fuzz 选中该进程，策略设为“随机化输入数据前 64 字节”；触发“恢复选中 Hook”功能，让真实调用链进入驱动；蓝屏后用 `!analyze -v` 看到 `mov dword ptr [eax],0`，eax 来自随机化后的输入；按栈回溯回到派遣函数 `sub_110D0`、控制码 0x50000408，再到 `sub_11690(firstDWORD, secondDWORD)`。
- **结论**: `sub_11690` 在 `MmIsAddressValid` 之前就执行 `*(_DWORD *)firstDWORD = 0`，检查顺序倒置；输入缓冲区第一个 DWORD 就是目标内核地址。
- **结果**: 可向任意内核地址写 0，属本地提权；作者指出函数内还有其他可利用点。

### 案例 2: 东方微点 mp110013.sys 的可控循环与可控调用（第 23 章 23.4.2）
- **问题**: 对 MPSVC2.exe 做 MITM Fuzz 后蓝屏，停止码 0x7f/8（双重错误），崩溃指令 `push dword ptr [eax]` 的寻址看起来正常，怎么解释？
- **方法论的使用**: 先用 MITM 随机化输入数据；`!analyze -v` 读出双重错误后，检查 eax 指向的 DWORD 确实存在，转而看 esp=f7b25000 落在 KernelStack 页顶，判定为栈上溢；回 IDA 找到 `sub_107B0`：循环次数取自输入缓冲区第二个 DWORD，循环体不断 push，循环结束 `call dword ptr [eax]` 调用第一个 DWORD 指向的地址。
- **结论**: 不是“非法地址”而是“栈耗尽”；用户输入同时被当作循环上界与函数指针，属于设计缺陷，不是典型缓冲区溢出。
- **结果**: 可造成本地 DoS；进一步可在 0 地址布置 Ring0 Shellcode，构造为可控调用与提权。

### 案例 3: 瑞星 HookCont.sys 的 Probe 与 try 双误（第 23 章 23.4.3）
- **问题**: 控制码 0x83003C07 的处理函数明明调用了 `ProbeForWrite`，为什么一个非法 UserBuffer 仍能把它打崩？
- **方法论的使用**: MITM Fuzz 后按蓝屏回溯到处理函数；检查 `ProbeForWrite(UserBuffer, OutputBufferLength, 1)` 发现第二个参数直接来自用户，传 0 时 Probe 不校验任何字节；再检查异常保护范围，发现 try 只包住 Probe，真正写 UserBuffer 的代码在 try 外；最后用 `UserBuffer=0x0、OutputBufferLength=0` 构造 POC。
- **结论**: “调用了 Probe”不等于“地址被验证”；长度参数与异常保护范围都必须以实际解引用点为准。
- **结果**: 触发内核访问违例蓝屏，属本地 DoS，无需提权即可复现。

### 案例 4: 从 IOCTL Fuzzer 到 MyIoControl Fuzzer 的工具设计（第 23 章 23.2–23.4）
- **问题**: eSage Lab 的 IOCTL Fuzzer 用 XML 配置 Fuzz 对象与策略，能快速做 MITM，但每次改配置都要重启程序，策略粒度也不够细。
- **方法论的使用**: 用“对象—策略—项”重画工具：MITM 对象的三字段可空（空 = 通配），Driver Fuzz 对象的设备名与控制码不可空；策略把地址/数据/长度分别给出不变、随机、非法、合法 Ring3、合法 Ring0 等方案；界面支持在线增删改，两种模式放在同一对话框中。
- **结论**: 工具的核心不是变异算法，而是“配置模型 + 热更新 + 两种模式互补”；XML 配置的局限不在表达能力，而在每次修改都要重启、会丢失现场。
- **结果**: MyIoControl Fuzzer 同时支持 MITM 与 Driver Fuzz，成为后续三个实战漏洞的挖掘工具。

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 手上有一个 `.sys`，想系统挖驱动漏洞，但不知道从哪个入口开始、设备名与 IoControlCode 怎么拿。
2. 用 MITM 跑了一轮，发现日志里的控制码种类长期不变，或同一控制码只测到一种参数组合，怀疑覆盖不全。
3. 想改用 Driver Fuzz 主动调用，但不确定 Method 位要不要 Fuzz 地址、参数面怎么裁剪、设备和控制码能否留空。
4. Fuzz 把系统打蓝了，需要判断是哪个控制码/哪组参数触发，以及怎样保留现场继续下一轮。
5. 工具每改一次配置就要重启引擎，想设计支持多 Fuzz 项并行、在线增删改的 IoControl Fuzzer。

### 语言信号 (用户的话里出现这些就应激活)

- “驱动怎么 Fuzz / IOCTL Fuzz / DeviceIoControl 怎么测 / kernel driver fuzzing / IOCTL fuzzer”
- “MITM Fuzz / Hook NtDeviceIoControlFile / 中间人 Fuzz”
- “Driver Fuzz / 设备名 / 符号链接 / IoControlCode 怎么构造”
- “Method 位 / METHOD_NEITHER / 要不要 Fuzz 输入输出地址”
- “Fuzz 对象 / Fuzz 策略 / Fuzz 项 / 在线增删改”
- “只有 .sys 没有设备名 / 控制码怎么找 / 设备打不开 / driver fuzz / kernel fuzz”

### 与相邻 skill 的区分

- 与 `fuzzing-strategy` 的区别：那个是跨对象的**范式选型**（Blind vs Smart、样本/模型、变异策略、工具选择），回答“该不该用 Fuzz、用哪一类”；本 skill 是内核 IOCTL 的**具体执行**：MITM/Driver 两种模式、对象—策略—项建模、按 Method 裁剪参数面、设备名与控制码的配置语义。若还没决定 Fuzz 范式，先走前者；已决定打驱动 IOCTL，走本 skill。
- 与 `network-protocol-fuzzing` 的区别：那个打的是用户态网络服务的命令、认证与会话；本 skill 打的是 Ring3→Ring0 的 `DeviceIoControl`，对象是设备与控制码，不是 socket 与协议状态机。
- 与 `kernel-driver-exploitation` 的区别：本 skill 负责“把驱动打蓝、拿到崩溃现场”，那个负责“崩溃之后怎么归约成写原语、走哪条利用路线、落点是否稳定”。先 Fuzz 后利用；不要用利用模板反推该 Fuzz 什么。
- 与 `reverse-driver-and-unknown-structure` 的区别：当设备名、符号链接或控制码完全未知时，那个 skill 从 `IoCreateDevice` / `DeviceName` / `CTL_CODE` 常量与分发例程入手做逆向提取；本 skill 从“入口已知”开始做系统性 Fuzz。若连设备都打不开且无法动态观察，先走逆向 skill。
- 与 `kernel-exploitation` 的区别：那个是归约后的提权链骨架，不负责漏洞挖掘与 Fuzz 工具设计；本 skill 不展开 token/cred 载荷。

---

## E — 可执行步骤 (Execution)

当 skill 被激活后，agent 应按以下步骤执行：

1. **选切入点并确认实验条件**
   - 在四类入口中排序：IoControl（公布漏洞最多，优先）→ 安全软件 Hook → 网络协议栈 → SSDT/Shadow SSDT；准备隔离虚拟机、目标驱动版本、快照与蓝屏转储配置。
   - 完成标准: 明确“打哪个入口、目标驱动版本、出事后如何回滚”。
   - 判停条件: 没有隔离环境/快照或没有授权时停止；内核 Fuzz 会蓝屏整机。

2. **提取 Fuzz 对象：设备名、符号链接、控制码、目标进程**
   - 静态：IDA 找 `IoCreateDevice` / `IoCreateSymbolicLink` / `DeviceName` 与 `CTL_CODE` 常量；动态：设备树、驱动对象、MITM 日志观察真实调用。
   - 完成标准: 至少拿到一个可打开的 `\Device\xxx` 或 `\\.\xxx` 符号链接，以及一个具体 IoControlCode。
   - 判停条件: 设备名或控制码拿不到时，不要硬跑 Driver Fuzz；退回 MITM 借真实调用链，或转 `reverse-driver-and-unknown-structure`。

3. **判 Method 位，裁剪参数面**
   - 拆 IoControlCode 最低两位：非 NEITHER（BUFFERED / IN_DIRECT / OUT_DIRECT）只 Fuzz 输入数据、输入长度、输出长度；NEITHER 才增加输入地址、输出地址。
   - 完成标准: 每个控制码都有一张“要 Fuzz 的字段”清单，并说明为什么排除地址或纳入地址。
   - 判停条件: Method 未知时先拆码或看派遣例程，不要对所有参数盲目随机，避免无效组合爆炸。

4. **建立“对象—策略—项”配置**
   - 对象：MITM 用进程/设备/控制码，任一为空表示通配；Driver Fuzz 用设备名 + 控制码，两者都不能为空。
   - 策略：地址（不变/随机/非法/合法 Ring3/合法 Ring0）、数据（不变/随机/Byte/Word/DWORD 填充）、长度（不变/随机/增减）。
   - 完成标准: 每个 Fuzz 项都能回答“何时命中、命中后改哪个字段、改成什么”。
   - 判停条件: 工具不支持多 Fuzz 项并行或在线增删改时，先改工具；每轮重启会丢失现场并严重降低效率。

5. **先 MITM 借真实调用链，再 Driver Fuzz 补覆盖**
   - MITM：Hook `NtDeviceIoControlFile`，按对象匹配后篡改参数并转发，重点触发 GUI 才能到达的功能路径，记录出现的控制码。
   - Driver Fuzz：主动打开设备，遍历控制码与参数组合，对单个控制码做系统组合。
   - 完成标准: MITM 日志给出“真实会调用哪些控制码”，Driver Fuzz 对这些控制码完成参数组合覆盖；两者结果能对上。
   - 判停条件: MITM 日志中的控制码种类长期不变，说明覆盖受目标程序功能限制，必须上 Driver Fuzz。

6. **监控崩溃并保全现场**
   - 在蓝屏前记录“进程、设备、控制码、输入/输出地址、数据、长度”，配置完整/内核/小内存转储；蓝屏后回滚快照，而不是重装环境。
   - 完成标准: 每次崩溃都能对应到一条完整的 Fuzz 项与一组具体参数。
   - 判停条件: 崩溃没有参数日志时，先补日志再继续；无法归因的蓝屏只会制造噪声。

7. **崩溃 triage：从停止码回到处理函数**
   - `!analyze -v` 读停止码与参数、出错指令与反汇编、陷阱帧寄存器、STACK_TEXT；锁定“驱动 + 偏移”，回 IDA 找函数与调用者，再对照 Fuzz 记录的原始参数确认哪一字段可控。
   - 完成标准: 能说出“哪条指令、哪个参数、从哪个控制码路径进来”，并复现最小参数集。
   - 判停条件: 若崩溃指令寻址正常，先查 esp 是否落在内核栈顶（栈上溢）；若仍无法归因，转 `crash-to-root-cause`。

8. **移交利用或调整策略继续**
   - 把可控的地址/数据/长度整理成“写原语”与触发参数，移交 `kernel-driver-exploitation`；若不可控，回到步骤 3/4 调整 Method 参数面或策略。
   - 完成标准: 要么产出可复现的最小 PoC 与写原语说明，要么明确记录“仅 DoS/不可控”的结论与已覆盖范围。
   - 判停条件: 只能 DoS 且无内存篡改时，记录为 DoS，不要硬套提权；需要 exploit 构造时转 `kernel-driver-exploitation`。

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- **目标是用户态网络协议、文件格式或 Web 参数**：分别走 `network-protocol-fuzzing`、`file-format-vulnerability-reversing`、`non-memory-vulnerability-analysis`；本 skill 的对象是驱动设备与控制码。
- **连设备名、符号链接、控制码都无法获得，也无法动态观察真实调用**：先走 `reverse-driver-and-unknown-structure`；Driver Fuzz 的设备名与控制码不能为空，缺一不可。
- **生产机、无快照、无授权**：内核 Fuzz 以蓝屏为反馈，会整机不可用并可能损坏数据；必须在隔离虚拟机或明确授权的环境中进行。
- **崩溃已拿到，要归因、利用或判断提权**：走 `kernel-driver-exploitation`（归约/路线/落点）或 `crash-to-root-cause`（通用根因定位）；本 skill 只到“崩溃对应哪组参数”。
- **目标驱动受现代内核保护（HVCI、DSE、PatchGuard、KASLR、SMEP/SMAP）或位于远程内核协议栈**：本书的 Hook 与固定地址假设不成立，需要专门的现代内核 Fuzz 环境与方法，不要照搬。

### 作者在书中警告的失败模式

- **只做 MITM Fuzz 就认为覆盖完整**：MITM 依赖目标程序主动发起调用，难以覆盖所有控制码；即便覆盖到，每个控制码也只被篡改“一次性”，不够全面（ce-g09-09）。
- **把 MITM 的空字段当成“禁用”**：进程/设备/控制码任一为空表示匹配所有，不是不匹配；配置时容易把范围放大到意外对象（p-g09-18）。
- **Driver Fuzz 的设备名或控制码留空**：主动调用必须明确“打哪个设备、打哪个控制码”，缺一个都无法构造有效请求（p-g09-19）。
- **不看 Method 位就 Fuzz 地址**：非 NEITHER 时地址由系统缓冲或锁定，改地址没有意义；反过来，NEITHER 时漏掉地址维度会丢掉主要攻击面（f-g09-11、ce-g09-16）。
- **每次改策略都重启 Fuzz 引擎**：内核 Fuzz 常在蓝屏后重启环境，配置不能热更新会把时间耗在恢复与重配，而不是分析（p-g09-16）。
- **不建快照**：蓝屏后从零重装目标软件与环境，恢复成本远高于回滚（ce-g09-17）。
- **崩溃只看指令寻址、不看 esp**：`push dword ptr [eax]` 的源地址合法仍可能因栈上溢崩溃，停止码可能是 0x7f/8（ce-g09-12）。
- **把“调用了 Probe”当成“地址已验证”**：Probe 的长度可能来自用户且为 0，异常保护也可能只包住 Probe 而没包住真正的写（ce-g09-03、ce-g09-04）。

### 作者的盲点 / 时代局限

- **2011 年的 32 位 XP/2003 语境**：MITM 依赖在核内 Hook `NtDeviceIoControlFile`，书中默认可以自由加载驱动、修改内核代码；现代 x64 的 PatchGuard、DSE、HVCI 会让这类 Hook 与测试驱动加载变得困难甚至不可行。
- **没有 KASLR / SMEP / SMAP / VBS**：书中的设备对象、函数地址与内核调试假设都建立在未随机化、无内核页权限隔离的环境；现代地址每次启动变化，固定地址与直接 Hook 需要重新设计。
- **以“蓝屏”为主要反馈信号**：只能发现导致崩溃的内存破坏，漏掉静默越界读、信息泄露、权限绕过与逻辑缺陷；也没有覆盖反馈、语料调度、崩溃去重与自动 triage。
- **工具形态是手工配置 + 单机 GUI**：对象/策略/项靠人填，缺少现代内核 Fuzz 的覆盖率引导（如 syzkaller 式 feedback）、结构化描述与分布式执行；书中工具更适合教学与小规模定向测试。
- **只覆盖 IoControl 主路径**：Fast I/O、WDF/KMDF 框架、文件系统过滤驱动、网络协议栈等入口没有展开；把“IoControl 最多”当成“只需测 IoControl”会漏掉其他内核攻击面。
- **对第三方驱动生态的估计偏乐观**：书中把第三方驱动视为主要目标，但没有讨论现代驱动签名、白名单、云查杀与内核隔离对“加载/调用测试驱动”的限制。

### 容易混淆的邻近方法论

- `fuzzing-strategy`：跨对象的 Fuzz 范式选型（Blind/Smart、样本/模型、变异策略）；本 skill 是内核 IOCTL 专用执行与工具建模。
- `network-protocol-fuzzing`：用户态网络服务的命令/认证/会话；本 skill 是 Ring3→Ring0 的 DeviceIoControl。
- `reverse-driver-and-unknown-structure`：从 `.sys` 与 PoC 片段逆出设备名、控制码与分发例程；本 skill 从入口已知开始系统 Fuzz。
- `kernel-driver-exploitation`：把崩溃归约为写原语、选利用路线、判落点稳定性；本 skill 只负责找到崩溃并保全现场。
- `crash-to-root-cause`：通用崩溃根因定位；本 skill 的 triage 只借用 `!analyze -v` 与栈回溯这一内核路径。

---

## 相关 skills

- **depends-on**:
  - `fuzzing-strategy` — 先选 MITM 还是主动构造

- **contrasts-with**:
  - `network-protocol-fuzzing` — 对象与传输层不同
  - `kernel-driver-exploitation` — 一个找崩溃，一个把崩溃变成提权

- **composes-with**:
  - `kernel-driver-exploitation` — 发现后转入利用

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: 待阶段 4
- **蒸馏时间**: 2026-10-16
