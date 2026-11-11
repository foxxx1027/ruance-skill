---
name: windows-access-control-attack
description: |
  当用户要在 Windows 本地从普通用户找配置型提权，枚举服务/注册表/目录/文件/共享内存/命名管道的 ACL，或诊断“拒绝访问”时调用。适用于 SID/令牌/SD/DACL/AccessCheck 分析、受限令牌、NULL DACL、令牌劫持。不适用于：远程域渗透、内核漏洞、仅做合规基线检查，也不适用于没有本地访问权的场景。Trigger: Windows ACL、DACL、ACE、SID、access token、AccessCheck、service permission、privilege escalation、访问控制、配置提权、令牌。
source_book: 《灰帽黑客 第4版》
source_chapter: 第14章
tags: [access-control, windows, dacl, privilege-escalation, local-privilege-escalation]
related_skills:
  - slug: pentest-engagement-and-disclosure
    relation: composes-with
---

# Windows 访问控制模型攻击

## R — 原文 (Reading)

> 如果程序员显式地请求一个 NULL DACL（空自主访问控制列表），那么所有人都会被允许访问。更具体地说，通过 AccessCheck 函数检查的任何期望的访问请求都会被允许。这与创建一个向 Everyone 组授予完全控制权的 DACL 是一样的。
>
> — Daniel Regalado 等，《灰帽黑客（第 4 版）》第 14 章 14.4.2，第 322 页

---

## I — 方法论骨架 (Interpretation)

Windows 访问控制把一次放行或拒绝拆成四个可分别审计的组件：SID 标识主体，访问令牌携带进程或线程实际持有的用户 SID、组 SID、特权、完整性级别与受限 SID，安全描述符（SD）承载对象侧的所有者、DACL 与 SACL，AccessCheck 负责把令牌与 SD 按 desiredAccess 比较。
判定分两阶段：先看调用方是否持有该 API 要求的特权（如 SeLoadDriverPrivilege），再走 DACL 检查——显式禁止 ACE 一票否决，继承禁止可被显式许可覆盖，许可 ACE 必须覆盖 desiredAccess 的每一位，受限 SID 追加检查且只缩权不扩权。
因此“谁能访问”不能靠用户名或单个 ACE 猜测，必须同时固定 DACL、令牌、desiredAccess 三个输入；缺少任一输入，都会把工具过度请求误判成权限不足。
NULL DACL 是没有 DACL，任何请求都放行；Empty DACL 是 DACL 存在但 ACE 数为零，拒绝所有访问；SACL 只负责审核，不参与放行判定。
攻击面来自“可写对象 + 更高权限消费方”：先枚举服务、注册表自启动键、程序目录与文件、共享内存段、命名管道、进程与线程，筛出授予非管理员 SID 的写类权限。
再把写权限按资源类型翻译成“高权限”：SERVICE_CHANGE_CONFIG 可改服务二进制，KEY_SET_VALUE 可改加载路径，FILE_WRITE_DATA 可覆盖可执行文件，SECTION_ALL_ACCESS 可向高权限解析器投毒。
读权限不是终点，而是两阶段攻击的情报；执行权限通常导向拒绝服务或扩大攻击面。
选定路径前必须回答：谁在更高权限下使用这个对象、何时使用、攻击者是否可达；可写但没有消费方，不构成提权。
利用路径按对象分派：服务重配、文件或目录替换与 DLL 重定向、注册表劫持、共享内存注入、管道写入或 squatting/luring 冒充、进程令牌劫持。
结果验证看实际副作用而非命令返回码；执行前先记录原始配置，结束后可恢复。
报告必须区分“可疑 ACL 候选”与“已确认提权”，后者要有可写证据、消费方、利用路径与实际结果。

---

## A1 — 书中的应用 (Past Application)

### 案例 1: upnphost 服务的 DACL 提权
- **问题**: Windows XP SP1 上，upnphost 服务的 DACL 允许任意用户控制该服务启动哪个二进制文件，并允许停止和启动服务；服务本身以 LocalSystem 运行。
- **方法论的使用**: 先用 AccessChk 枚举服务并筛出授予非管理员 SID 的写权限，确认“可写对象”；再把写权限翻译成高权限——控制服务二进制路径等于控制 LocalSystem 执行的代码；最后按服务对象路径，用 `sc stop upnphost`、`sc start upnphost` 触发被替换的 attack.exe。
- **结论**: 服务对象写权限加上高权限服务账户，直接构成配置型提权；不需要 shellcode，甚至不需要编译器。
- **结果**: attack.exe 以系统最高权限账户 LocalSystem 运行；改回原服务文件也不留痕迹。微软于 2006 年修补该漏洞。

### 案例 2: secret.txt 的“禁止访问”之谜
- **问题**: c:\temp\secret.txt 的 DACL 只给 Everyone 授予 FILE_READ_DATA 与 FILE_WRITE_DATA，但执行 `type secret.txt` 却返回 Access is denied；直觉会误判为读权限被禁止。
- **方法论的使用**: 用三输入法拆解——固定对象 DACL、当前进程令牌与实际 desiredAccess。Process Monitor 显示 type 以 GenericRead（0x80000000）打开文件，该掩码还包含 READ_ATTRIBUTES、READ_EA、SYNCHRONIZE 等位，而 DACL 未授予这些位。随后编写 supertype.c 只请求 FILE_READ_DATA，或用调试器在 kernel32!CreateFileW 断点处把 esp+0x8 的 desiredAccess 改为 MAXIMUM_ALLOWED（0x02000000）。
- **结论**: 拒绝的根因是通用工具请求了过大的访问掩码，不是 FILE_READ_DATA 缺失；测试权限必须只请求最小必要权限。
- **结果**: supertype.exe 成功输出“this is a secret”；调试器改掩码同样读出文件，证明原判断是假阴性。

### 案例 3: 共享内存段 NULL DACL 与命名管道 squatting
- **问题**: 文件与服务之外的特殊对象是否也有可利用的宽松 ACL：共享内存段可能向 Everyone 开放写权限，命名管道可能被低权限用户写入或被高权限方主动连接。
- **方法论的使用**: 用 AccessChk 转储对象管理器命名空间，发现 `\BaseNamedObjects\INOQSIQSYSINFO` 是 Section，Everyone 拥有 SECTION_ALL_ACCESS（等价于 NULL DACL），而消费方是高权限的 eTrust 引擎；按对象名搜索公开漏洞。管道侧则先枚举 ACL，再把某服务的日志路径改为 `\??\Pipe\AttackerPipe`，创建同名管道并触发日志，利用 SeImpersonatePrivilege 冒充连接者。
- **结论**: NULL DACL 或 Everyone 写权限意味着不可信输入能进入高权限消费方，应直接判为漏洞候选；管道 luring 不依赖目标解析缺陷，也能借身份模拟获得高权限。
- **结果**: eTrust 共享段存在公开的栈溢出利用路径（书中未复现实验）；Cerrudo 的管道方法可冒充 LocalSystem 调用者（书中转述）。

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 已拿到 Windows 本地普通用户 shell 或受限令牌，想在补丁齐全的主机上寻找配置型提权，而不是碰运气打内存漏洞。
2. 红队或渗透测试需要枚举服务、注册表自启动键、程序目录、共享内存段、命名管道，判断“可写 ACL”是否真能导向 SYSTEM。
3. 防御方或审计人员要解释某个服务、目录或注册表键为何能被普通用户改写，并需要可复核的 DACL/ACE 证据。
4. 开发或运维遇到 Access Denied，想判断到底缺的是 DACL 权限、令牌能力，还是程序请求的 desiredAccess 过大。
5. 复核漏洞报告时看到 Everyone 完全控制、NULL DACL、SERVICE_CHANGE_CONFIG 等字样，需要确认消费方与实际影响，避免误报。

### 语言信号 (用户的话里出现这些就应激活)

- “普通用户怎么在 Windows 上提权”“服务权限配置错了能不能打”
- “AccessChk 显示这个服务对 Users 可写”“SERVICE_CHANGE_CONFIG 能干什么”
- “type 提示拒绝访问，但 ACL 里明明有读权限”
- “Everyone 完全控制 / NULL DACL 算不算漏洞”
- “命名管道能不能冒充高权限进程”“SeImpersonatePrivilege 怎么用”
- “access denied despite ACL”“writable service path”“privilege escalation via ACL”
- “DACL、ACE、SID、访问令牌、AccessCheck 之间是什么关系”

### 与相邻 skill 的区分


- **composes-with** `pentest-engagement-and-disclosure`：本 skill 枚举 ACL、令牌、服务等配置型提权路径；测试是否在授权范围内、发现后如何披露由它门禁。
- 与 `non-memory-vulnerability-analysis` 的区别: 后者处理应用层逻辑与授权缺陷（越权、IDOR、注入等），本 skill 处理 Windows 操作系统对象的 ACL、令牌与 AccessCheck；Web 越权不能套用 AccessChk 流程。
- 与 `vuln-analysis-strategy` / `vulnerability-mechanism-identification` 的区别: 后者负责判定漏洞类型与选择分析路线；本 skill 已把对象类型、枚举顺序与利用路径固定下来。
- 与 `windows-stack-protection-bypass` / `kernel-exploitation` 的区别: 那些处理内存破坏与内核代码缺陷；本 skill 利用的是配置错误，通常不需要 shellcode。
- 与通用漏洞扫描的区别: 扫描器只能报“某对象可写”，不能证明高权限消费方与可达性；本 skill 要求补全确认链。

---

## E — 可执行步骤 (Execution)

当 skill 被激活后，agent 应按以下步骤执行:

1. **确认授权与本地立足点**
   - 完成标准: 有明确授权或靶场范围，且已在目标上获得可执行命令的本地账户（普通用户、受限令牌或服务账户均可）。
   - 判停条件: 无本地执行权、只能远程黑盒测试 → 停止本流程，改走远程攻击面；不要凭端口或版本猜测 ACL。

2. **建立四组件基线**
   - 完成标准: 记录当前用户 SID、组 SID、特权、完整性级别、是否受限令牌；工具可用 `whoami /all`、Process Explorer 的 Security 页、ntsd/cdb 的 `!token -n`，并明确进程实际持有哪枚令牌。
   - 判停条件: 发现受限 SID → 按“只缩权”评估，不得用 Administrators 等正常组 SID 推断能力。

3. **枚举可写对象**
   - 完成标准: 对服务（`accesschk -c * -w`）、注册表自启动键（autorunsc 导出后 `accesschk -w -q -k`）、程序目录与文件（`accesschk -d/-w`）、Section/Pipe/进程（AccessChk、PipeList、WinObj）分别产出候选清单。
   - 判停条件: 清单为空 → 转向第三方软件的自定义安装路径，或结束并说明默认 ACL 可能已足够。

4. **按 SID 过滤与分级**
   - 完成标准: 去掉 Administrators/SYSTEM，把命中项分为不可信用户（Users、Everyone、Guest、Anonymous、INTERACTIVE）与部分可信用户（LocalService、NetworkService、NetworkConfiguration Operators、SERVICE），并标注默认无成员的组。
   - 判停条件: 仅命中部分可信用户且无法先取得该身份 → 降级为两阶段候选，不直接报“可立即提权”。

5. **解码 ACE 与 DACL 结构**
   - 完成标准: 每个候选都有访问掩码逐位含义、继承标志（OI/CI/IO）与所有者；区分 NULL DACL（无 DACL，全放行）与 Empty DACL（有 DACL 但无 ACE，全拒绝）；用 SubInACL 复核 AccessChk 输出。
   - 判停条件: 该条目只出现在审核页 → 判为 SACL，不是 DACL 漏洞；DACL 为空或 Everyone 全权限 → 直接标为漏洞候选并进入第 7 步。

6. **按资源类型判“高权限”**
   - 完成标准: 把权限分为写类（提权）、读类（信息泄露或两阶段情报）、执行类（拒绝服务或扩大攻击面），并写出该对象特有的权限名，如 SERVICE_CHANGE_CONFIG、KEY_SET_VALUE、FILE_WRITE_DATA、SECTION_ALL_ACCESS。
   - 判停条件: 只有读权限 → 不按提权漏洞报告，转为情报收集并继续枚举。

7. **确认消费方与可达性**
   - 完成标准: 说明“谁以更高权限、在什么时机、从哪个路径使用该对象”，以及攻击者是否本地可达、跨会话可达或只能等待重启。
   - 判停条件: 可写但没有任何高权限消费方，或该共享本就是设计功能 → 判为不可利用，回到第 3 步继续枚举。

8. **验证真实访问（三输入法）**
   - 完成标准: 遇到 Access Denied 时，分别固定 DACL、令牌与 desiredAccess；用 Process Monitor 读取实际 DesiredAccess，用最小请求的自写工具或调试器改掩码验证；不以 type/notepad 的结果下结论。
   - 判停条件: 通用工具被拒但最小请求成功 → 修正“权限不足”判断，按真实权限继续评估，而不是放弃该对象。

9. **选择利用路径**
   - 完成标准: 依对象类型选定路径——服务改 binPath 与启动账户；文件或目录覆盖内容、用 .manifest 做 DLL 重定向、利用 %path% 或自启动位置；注册表劫持加载路径、COM 或解析器输入；Section 注入数据；管道写入或 squatting/luring 冒充；进程尝试令牌劫持。同时列出前置条件。
   - 判停条件: 关键前置条件不满足（无法取得服务账户、无 SeImpersonatePrivilege、目标路径不可达）→ 回到第 3 步寻找其他对象。

10. **先快照后执行并验证效果**
    - 完成标准: 执行前记录原始服务配置、文件内容或 ACL；执行后以实际副作用（新账户、新进程、文件变化）验证，而不只看命令返回码。
    - 判停条件: `net start` 报“服务未响应控制函数”但目标副作用已出现 → 判为成功，不要按失败处理。

11. **区分候选与确认并写报告**
    - 完成标准: 每条发现都包含可写证据、消费方、利用路径、实际结果与修复建议（显式授予所需权限的 DACL、最小权限、替换 NULL DACL）；未验证的只标候选。
    - 判停条件: 仅有 Everyone 写权限而无消费方证据 → 不得写成确认漏洞。

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 没有本地执行权，只能远程黑盒测试: ACL 与令牌枚举依赖本地工具和进程上下文，远程扫描无法替代，否则会把端口或版本猜测当成权限结论。
- 无第三方软件的新版 Windows 纯默认环境: Windows 10/11 的默认 ACL 已明显收紧，书中 XP/2003/7 时代大量可写服务、目录与注册表示例不再默认成立；不能拿旧默认配置直接报漏洞。
- 纯域环境或 AD 对象权限（ACL 委派、组策略、BloodHound 类路径）: 这是另一套域权限模型，应使用 AD 专用方法。
- 已确认是内存破坏漏洞: 应走内存利用或保护绕过类 skill，不要用 ACL 枚举替代根因分析。
- 对象共享属于设计功能且没有高权限消费方: 不能仅凭 Everyone 可写或 NULL DACL 就定性为漏洞。

### 作者在书中警告的失败模式

- ce83: 把 SACL 当成 DACL 报漏洞；SACL 只决定审核，不参与放行或拒绝。
- ce84: 假设 Everyone 组总在令牌中；匿名、空会话或特定配置下并非如此。
- ce85、ce88: 用 type/notepad 的拒绝结果推断文件权限；它们以 GenericRead 打开，属于过度请求造成的假阴性。
- ce86、ce87: 混淆 NULL DACL 与 Empty DACL；前者全放行（还等于给出 WriteOwner/WriteDAC），后者全拒绝。
- ce89、ce91: 只看正常组 SID 判断受限令牌能力；受限 SID 会追加检查且只缩权，但同桌面以正常令牌运行的高权限进程仍可能是突破口。
- ce90: 只盯显式写出的权限位；目录的 FILE_LIST_DIRECTORY 等权限会带来子对象的关联授权。
- ce92: 以为服务改成 LocalService/NetworkService 就阻断了提权；SERVICE_CHANGE_CONFIG、令牌劫持等路径仍可到 SYSTEM。
- ce93: 只检查 DELETE 权限；只要拥有 FILE_WRITE_DATA，就能直接覆盖可执行文件内容。
- ce94: 在 XP/Vista 上继续用 .local 重定向 KnownDLLs；该机制对 kernel32、user32 等无效，应改用 .manifest。
- ce95: 把 `net start` 的报错当作攻击失败；服务协议未应答但子进程可能已经执行。
- ce96、ce101: 把“注册表键可写”或“可疑 DACL”直接当确认漏洞；必须证明高权限消费方与实际影响。
- ce97: 以为所有解析器都经过充分加固；注册表解析与私有格式常缺少同等强度的模糊测试。
- ce98: 以为 %ProgramFiles% 的 ACE 继承能保证所有目录安全；自定义安装路径与自行设置 DACL 的软件不受保护。
- ce99、ce100: 以为共享内存段与命名管道中的数据天然可信私有；其 ACL 可能允许低权限写入，消费方若信任内容就会中招。

### 作者的盲点 / 时代局限

- 书中的默认配置、工具与语义属于 XP/2003/7 时代：AccessChk、SubInACL、旧版 ntsd、AutoRuns、PipeList 的可用性与输出格式已变化，.local 融合加载和 KnownDLLs 行为也与现代 Windows 不同。
- 案例多依赖未打补丁系统或可写默认路径；现代安装器与默认 ACL 提高了门槛，但第三方软件、企业自定义部署与遗留系统仍可能复现同类问题。
- 全书以“不留痕迹”的本地提权叙事为主，没有系统讨论 SACL 审核策略、日志与 EDR 可见性，不能据此认为此类攻击在现代环境中难以检测。
- 权限判断以手工推演为主，未覆盖竞态（TOCTOU）、对象重建、ACL 继承动态变化等情形；结论需要以实际访问测试为准。

### 容易混淆的邻近方法论

- 应用层越权（`non-memory-vulnerability-analysis`）: 关注业务逻辑与功能级授权缺失，本 skill 关注操作系统对象 ACL 与令牌，两者证据链完全不同。
- 漏洞分类与路线选择（`vuln-analysis-strategy`、`vulnerability-mechanism-identification`）: 那些回答“这是什么类型的漏洞、下一步走哪条分析路线”，本 skill 已给定 Windows ACL 的枚举与利用顺序。
- 内存保护绕过与内核利用（`windows-stack-protection-bypass`、`kernel-exploitation`）: 那些针对代码缺陷与保护机制，本 skill 针对配置错误，通常无需 shellcode。
- 通用漏洞扫描与基线检查: 扫描器只能给出“可写”告警，不能替代消费方确认、可达性判断与最小请求验证。
- 域内 ACL 攻击: AD 对象权限、委派与组策略是另一套模型，不能直接套用本地 DACL 的五步判定。

---

## 相关 skills

- **composes-with** `pentest-engagement-and-disclosure` — 本地提权测试同样受 SOW 与范围约束

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: 待阶段 4（详见 test-prompts.json）
- **蒸馏时间**: 2026-10-19
