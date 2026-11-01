---
name: arm-mobile-overflow
description: |
  Windows Mobile/CE 32 位 ARM 程序出现可控栈溢出，或迁移 ARM 时使用；也包括"手机溢出怎么定位""ARM 返回地址在哪""无源码怎么从启动调试"。先重建 WinCE 内存地图与 ARM 调用约定（LR/R14、PC/R15、R0–R3），再按源码情况选 VS/IDA，最后用"前部填充=缓冲偏移"布置 payload。不适用于：Android/iOS/Linux ARM（android-vulnerability-analysis）；ARM64。trigger：Windows Mobile 溢出 / ARM 栈溢出 / LR / 手机 shellcode。
source_book: 《0day安全：软件漏洞分析技术（第二版）》 王清 等
source_chapter: 第 7 章（7.1–7.5）
tags: [ARM, WindowsMobile, WindowsCE, 栈溢出, LR, shellcode, 移动安全]
related_skills:
  - slug: win-stack-overflow-exploit
    relation: depends-on
  - slug: android-vulnerability-analysis
    relation: contrasts-with
  - slug: win-stack-overflow-exploit
    relation: contrasts-with
  - slug: vuln-debugging-and-breakpoints
    relation: composes-with
---

# 移动平台（ARM）缓冲区溢出

## R — 原文 (Reading)

> “在调用子函数的时候它不是使用的 CALL 指令，而是使用了 BL 指令，这也是 ARM 的一个特点。在执行 BL 指令后，R15 的值（即函数返回地址）会保存在 R14 中，进入函数后程序会将 R14 的值入栈保存。”
>
> — 王清 等，《0day安全：软件漏洞分析技术（第二版）》第 7 章 7.2.5

---

## I — 方法论骨架 (Interpretation)

手机溢出与 PC 溢出在本质上相同：都是往缓冲区复制超长数据，淹没重要结构，最终控制程序流程。**但不能直接把 x86 的模型搬过去**——换架构要先重画三张图，再谈偏移和布局。

**第一张是内存地图。** Windows Mobile 的 4GB 虚拟地址空间由所有进程共享，不像桌面 Windows 那样每进程独立 4GB；高 2GB 归内核，低 2GB 归用户，用户空间再切成 64 个 32MB 的 Slot，Slot 0 是当前进程，Slot 1 常被 XIP DLL 占用，Slot 33–63 由所有进程共享。不知道缓冲区落在哪个 Slot、模块基址在哪，后面的地址计算都是空的。

**第二张是 API 地图。** Windows CE 的 API 是桌面 Win32 API 的子集，还有一批 CE 特有 API；桌面能调的函数在手机上可能根本不存在。写 shellcode 前先核实目标 API 与它在当前 OS build 上的地址。

**第三张是调用约定。** ARM 用 R0–R3 传前 4 个参数，超过 4 个才借助堆栈；返回值放 R0；调用子函数用 `BL`，它把返回地址（R15/PC 的值）存进 LR（R14），函数序言再把 LR 入栈。因此**返回地址不在 x86 的“EBP+4”位置，而是先到 LR、再由序言保存到栈上**——找返回地址必须看序言/尾声，不能套 EBP/EIP 的公式。

有了这三张图，工具选择也就确定了：有源码就用 VS 做源码级调试；没有源码、且需要从程序启动就介入，就用 IDA 远程调试（VS 只能 attach，不能从启动开始）。定位目标函数时，可以把特征字符串（如 “Hello World”）当锚点，找它的 DATA XREF，反查到 `MessageBox` 调用点。最后是布局：`buf[16]` 溢出时，前 16 字节填充正好等于缓冲区到返回地址的偏移，后面接覆盖后的返回地址，再接 shellcode；手机 shellcode 前部的填充不是装饰，它就是缓冲偏移。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: 无源码时用 IDA 远程调试定位 MessageBox（第 7 章 7.4.3）
- **问题**: 手上没有程序源码，VS 只能 attach 到已运行进程，无法从程序启动就开始调试；需要找到 `MessageBox` 的调用过程与溢出路径。
- **方法论的使用**: 按“有无源码、能否从启动调试”选工具，改用 IDA 远程调试从启动介入；把程序里会弹出的字符串 “Hello World” 当锚点，在字符串窗口找到它，再顺着 DATA XREF 反查引用它的函数与 `MessageBox` 调用点。
- **结论**: 无符号、无源码时，字符串锚点可以跳过漫长的调用链回溯，直接落到目标调用；工具选择由约束决定，而不是由习惯决定。
- **结果**: 成功定位 `MessageBox` 调用过程，为后续分析 ARM 调用约定和布置 shellcode 提供了落点。

### 案例 2: 手机 exploit me：`buf[16]` 溢出（第 7 章 7.5）
- **问题**: `display` 函数里 `char buf[16]`，却用 `fread` 从 `exploit.txt` 读 100 字节；目标是淹没返回地址并执行弹出 failwest 对话框的 shellcode。
- **方法论的使用**: 按 ARM 调用约定重画栈帧，确认 LR 入栈位置与返回地址偏移；构造“16 字节 `0x31` 填充 + 返回地址 + shellcode”的布局，前部填充正好等于缓冲区到返回地址的距离；shellcode 用 R0–R3 传参、`BX R4` 调 `MessageBox`；实验时关闭 GS 编译选项，使用 Windows Mobile 6 Pro。
- **结论**: 手机溢出的布局公式与 PC 同构，但填充长度必须按 ARM 栈帧实测；shellcode 里的 `MessageBox` 地址依赖具体 OS build。
- **结果**: `exploit.txt` 触发溢出，程序弹出 failwest 对话框，验证了移动端返回地址劫持。

### 案例 3: 重画 ARM 调用约定（第 7 章 7.2.5）
- **问题**: 直接把 x86 的“EAX 返回值、栈传参、返回地址在栈顶”模型套到 ARM，会找不到返回地址，也会把参数位置全算错。
- **方法论的使用**: 反汇编 `test(1,2,3,4,5)`：前 4 个参数分别走 R0–R3，第 5 个参数走堆栈；调用用 `BL` 而非 `CALL`，`BL` 把 R15 存入 R14，函数序言再把 R14 入栈；返回值放在 R0，函数尾声从栈中取回返回地址赋给 R15/PC。
- **结论**: ARM 的返回地址先到 LR 再入栈，必须按序言/尾声定位；参数寄存器与 x86 完全不同，迁移前必须先建立本架构的调用约定图。
- **结果**: 建立了 ARM 栈帧模型，后续溢出定位、偏移测量与 shellcode 参数布置都以这套约定为准。

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 一个 Windows Mobile / Windows CE 应用出现可复现崩溃，调试器显示 PC 被输入覆盖，需要把它做成能执行 shellcode 的 exploit。
2. 拿到一台 32 位 ARM 嵌入式设备的固件或程序，想按“找缓冲区 → 量偏移 → 覆盖返回地址”的路径找溢出。
3. 把 PC 上的栈溢出方法原样迁到 ARM，发现偏移对不上、返回地址找不到，或覆盖后 PC 没变。
4. 目标没有源码，或只有设备/模拟器的远程调试通道，需要决定用 VS 还是 IDA、能否从启动调试。
5. 需要为手机写 shellcode，但桌面 API 不存在，或 `MessageBox` 一类函数地址在不同 OS build 上不一样。

### 语言信号 (用户的话里出现这些就应激活)

- “Windows Mobile 溢出 / Windows CE 缓冲区溢出 / 手机 exploit”
- “ARM 栈溢出 / 返回地址在哪 / LR / R14 / PC / R15”
- “R0–R3 传参 / ARM 调用约定 / BL 指令”
- “手机 shellcode / MessageBox 地址 / failwest”
- “IDA 远程调试 / 无源码 / 从启动调试”
- “16 字节缓冲 / 前部填充 / ARM buffer overflow”

### 与相邻 skill 的区分

- 与 `android-vulnerability-analysis` 的区别：那个 skill 面向 Android/Linux，主线是 APK/smali、so 库、系统 Java、内核的分层调试与签名校验；本 skill 面向 Windows Mobile/Windows CE 的 32 位 ARM 用户态栈溢出，核心是 WinCE 内存地图与 ARM 调用约定。Android 的 ARM 程序不要套用本 skill 的 Slot 模型与 WinCE API 假设。
- 与 `win-stack-overflow-exploit` 的区别：那个 skill 是 x86 的 EBP/EIP/ESP 模型；本 skill 要先重画 LR/PC 与 R0–R3 传参，返回地址的位置和参数布置都不同。若目标就是 x86 Windows，不要用本 skill。
- 与 `windows-stack-protection-bypass` / `dep-aslr-bypass` 的区别：那两个 skill 处理 x86 Windows 的 GS、SafeSEH、SEHOP、DEP/ASLR；本书的 Windows Mobile 章节没有展开这些保护的绕过，只要求实验时关闭 GS，不能把桌面端的绕过结论直接搬来。
- 与 `exploit-chain-construction` 的区别：那个 skill 是现代通用利用链、信息泄露与 ROP；本 skill 是“换平台先重建内存与调用约定”的基础构造，不涉及现代移动端的 PAC/BTI、沙箱与高熵 ASLR。

---

## E — 可执行步骤 (Execution)

当 skill 被激活后，agent 应按以下步骤执行：

1. **确认平台、架构与调试通道**
   - 记录 Windows Mobile/CE 版本、ARM 32 位、编译器与 SDK、有无源码、是否能从启动调试、是否有设备或模拟器。
   - 完成标准: 输出平台与工具约束表；明确“源码可调试 / 只能远程 attach / 可从启动调试”。
   - 判停条件: 若目标是 Android/iOS/Linux ARM，转到 `android-vulnerability-analysis`，不要继续。

2. **重建 Windows CE 内存地图**
   - 标出共享 4GB、内核 2GB、用户 2GB、64 个 32MB Slot、当前进程所在 Slot、模块基址、栈区与缓冲区位置。
   - 完成标准: 能回答“缓冲区在哪个 Slot、模块加载在哪、栈向哪个方向增长”。
   - 判停条件: 若仍按“每进程独立 4GB”推算地址，先停下纠正模型，否则后续偏移全部不可信。

3. **重建 ARM 调用约定并定位返回地址**
   - 确认前 4 个参数走 R0–R3、返回值走 R0、`BL` 把返回地址存入 LR/R14、序言把 LR 入栈、PC 为 R15；反汇编目标函数的序言与尾声，找保存/恢复 LR 的位置。
   - 完成标准: 给出“缓冲区起始 → 保存的 LR”的精确字节数，而不是套用 x86 的 EBP+4。
   - 判停条件: 若函数尾声直接 `POP {PC}` 或使用帧指针，按实际指令重新计算，不要沿用固定公式。

4. **按约束选调试器并定位漏洞函数**
   - 有源码用 VS 做源码级调试；无源码且需从启动介入用 IDA 远程调试；无符号时用特征字符串找 DATA XREF，反查调用点。
   - 完成标准: 在复制函数与函数返回处各有一个有效断点，能观察到缓冲区写入与 PC/LR 变化。
   - 判停条件: 若调试器只能 attach、无法从启动介入，改用 IDA 或寻找更早的注入点。

5. **用可识别填充实测偏移并关闭 GS**
   - 按实验要求关闭 GS 编译选项；用等长、可辨认的填充串触发溢出，观察保存的 LR/PC 被哪一段覆盖。
   - 完成标准: 偏移经过一次实际崩溃或断点验证，PC 可控。
   - 判停条件: 若偏移随编译选项/优化变化，固定编译配置后重测，不要用静态推算代替实测。

6. **布置 payload 并核对 API/地址**
   - 采用“填充（= 缓冲偏移）+ 覆盖后的返回地址 + shellcode”布局；核实目标 API 在 Windows CE 上存在，`MessageBox` 等地址与具体 OS build 匹配；注意 ARM/Thumb 模式与 4 字节对齐。
   - 完成标准: 在设备或模拟器上 PC 落入 shellcode 并执行预期动作。
   - 判停条件: 若换设备/换 OS build 后地址失效，按目标重新定位，不要照抄公开地址。

7. **跨设备、跨重启验证**
   - 记录 OS build、设备型号、模块基址与所有硬编码地址；在全新进程、重启后各跑一次。
   - 完成标准: 两次以上独立运行成功，失败模式可解释。
   - 判停条件: 仅单次成功时，按地址差异、栈布局或 Thumb 位复查，不要把它当作通用 exploit。

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- **Android、iOS 或 Linux ARM 目标**：操作系统、ABI、调试工具与漏洞面都不同；Android 走 `android-vulnerability-analysis`，不能套用 WinCE 的 Slot 地图与 API 子集假设。
- **ARM64/AArch64、现代移动端**：本书只讨论 32 位 ARM；ARM64 的寄存器、调用约定、PAC/BTI/MTE、现代 ASLR 与沙箱都不在覆盖范围内。
- **x86 Windows 桌面程序**：返回地址、参数传递与保护机制完全不同，走 `win-stack-overflow-exploit`。
- **没有设备、模拟器或任何调试通道**：本 skill 的偏移与地址都必须实测，纯静态推算不足以支撑可靠 exploit。
- **纯托管应用或没有原生缓冲区的场景**：本 skill 针对原生 C/C++ 缓冲区溢出。

### 作者在书中警告的失败模式

- **实验时不关闭 GS**：作者在手机栈溢出实验的编译选项里明确要求“禁用 GS 选项”，否则栈保护会拦截返回地址覆盖。
- **混用桌面与移动的内存模型**：桌面 Windows 每进程独立 4GB，Windows Mobile 所有进程共享一个 4GB 并按 Slot 划分；按独立地址空间推算会得到完全错误的地址。
- **驱动中用带缓存的映射直接写设备**：缓存会改变写操作顺序，直接访问设备 I/O 或寄存器时必须使用无缓存映射，否则可能造成灾难性后果。
- **假设桌面 API 在手机上都有**：Windows CE 的 API 是桌面 Win32 的子集；迁移检测/利用代码前先核实 API 是否存在。
- **照抄公开地址**：`MessageBox` 等函数地址随 OS build 变化，书中给出的地址只对特定 Windows Mobile 版本成立。

### 作者的盲点 / 时代局限

- **2011 年的 Windows Mobile 6 / Windows CE 语境**：实验基线是 32 位 ARM、Windows Mobile 6 Pro、VS 2008；没有 Android/iOS，没有 ARM64，也没有现代移动端的 PAC/BTI/MTE、沙箱与高熵 ASLR。
- **只讨论“关闭 GS”，没有讨论绕过**：本章为复现经典溢出直接禁用 GS，没有给出 WinCE 上 GS、DEP、ASLR 的绕过方法；不能据此认为移动端没有这些保护。
- **调用约定表述有笔误**：书中写“参数不超过 4 个时使用 R0~R4 寄存器”，按 ARM AAPCS 实际是 R0–R3 四个寄存器传前 4 个参数；分析时应以 ABI 与目标反汇编为准。
- **未覆盖 Thumb/ARM 切换与高级利用**：书中示例以 ARM 指令为主，没有系统讨论 Thumb 模式、指令对齐、ROP/JOP、堆利用与信息泄露。
- **地址与实验高度依赖具体环境**：Slot 布局、模块基址、`MessageBox` 地址都随设备与 OS build 变化；书中结论不能直接迁移到其他手机或固件。

### 容易混淆的邻近方法论

- `android-vulnerability-analysis`：Android 分层调试与签名机制，不是 Windows CE/ARM 栈溢出构造。
- `win-stack-overflow-exploit`：x86 的 EBP/EIP/ESP 模型，不是 LR/PC 与 R0–R3 传参。
- `windows-stack-protection-bypass` / `dep-aslr-bypass`：x86 Windows 的保护绕过，不适用于本书的 WinCE 实验前提。
- `exploit-chain-construction`：现代通用利用链与信息泄露，不是“换平台先重建内存地图”的基础方法。
- `heap-overflow-exploitation`：堆块结构与堆中函数指针；本章的手机溢出是栈缓冲区溢出，不涉及 WinCE 堆管理。

---

## 相关 skills

- **depends-on**:
  - `win-stack-overflow-exploit` — 方法骨架沿用 PC 栈溢出，但需重建栈帧

- **contrasts-with**:
  - `android-vulnerability-analysis` — 同为移动平台，但 Android 走 APK/so/内核分层，Windows Mobile 走 Win32 子集
  - `win-stack-overflow-exploit` — 架构与调用约定不同，不能照搬偏移

- **composes-with**:
  - `vuln-debugging-and-breakpoints` — IDA 远程调试与断点观察

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: 待阶段 4
- **蒸馏时间**: 2026-10-16
