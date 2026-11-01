---
name: middleware-fingerprint-and-triage
description: |
  当用户只拿到一个 Web 目标、需要先判断它跑的是哪种中间件（IIS/Apache/Nginx/Tomcat/JBoss/WebLogic/GlassFish/WebSphere），再决定查哪些漏洞时调用。适用于按指纹分派检查表、判断版本/组件/认证是否构成可利用条件。不适用于：已明确容器只需修复、纯 Web 应用漏洞（XSS/SQLi）、未授权扫描。Trigger: 中间件指纹、Server 头、X-Powered-By、默认页面、端口 8080/7001/4848/9060、middleware fingerprint、which server。
source_book: 《Web中间件常见漏洞总结》 lyxhh
source_chapter: 全书各中间件节（IIS/Apache/Nginx/Tomcat/JBoss/WebLogic/GlassFish/WebSphere）
tags: [中间件, 指纹识别, 分诊, 检查表, web]
related_skills:
  - slug: upload-parser-bypass-audit
    relation: composes-with
  - slug: java-deserialization-entry-enumeration
    relation: composes-with
  - slug: middleware-misconfig-audit
    relation: composes-with
---

# 中间件指纹识别与检查表分派

## R — 原文 (Reading)

> “默认返回的指纹信息: Server: GlassFish Server Open Source Edition 4.1.2；X-Powered-By: Servlet/3.1 JSP/2.3 (GlassFish Server Open Source Edition 4.1.2 Java/Oracle Corporation/1.8)”
>
> — lyxhh，《Web中间件常见漏洞总结》GlassFish 节

---

## I — 方法论骨架 (Interpretation)

面对一台未知 Web 服务器，不要从漏洞名或端口倒推，而要先确定它属于哪个容器族系，再用该族系的默认组件与解析规则分派检查表。

具体做法分四步：第一，收集指纹——响应头（`Server`、`X-Powered-By`）、默认页面、固定 banner、错误页格式、默认端口（8080 / 7001 / 4848 / 9060 等）。第二，把指纹映射到容器与大致版本区间，并列出该容器的高频漏洞面（IIS 看解析与短文件名，Nginx 看配置，Java 容器看反序列化入口与后台）。第三，过一道「可利用性闸门」：版本是否落在影响范围、所需组件是否开启、是否要求认证、默认配置是否保留。第四，用最小请求验证，并记录响应差异作为证据。

核心是：**指纹决定检查表，闸门决定优先级，响应差异决定结论**。三者缺一，都会退化成“扫一遍再说”。

---

## A1 — 书中的应用 (Past Application)

### 案例 1：从指纹锁定 GlassFish 与目录穿越

- **问题**：目标开放 4848 端口，首页没有明显产品名。
- **方法论的使用**：作者先看响应头，拿到 `Server: GlassFish Server Open Source Edition 4.1.2` 与 `X-Powered-By`，确认是 GlassFish 4.1.2；再由该版本的检查表想到 CVE-2017-1000028，用 `..%c0%af` 目录穿越读取 `windows/win.ini`。
- **结论**：版本指纹直接决定了要验证的漏洞类型与路径。
- **结果**：成功读取 `win.ini`；作者补充“若不能成功读取，请自行添加 `..%c0%af`”，说明验证要按路径深度调整。

### 案例 2：用固定 banner 判断 WebSphere 反序列化入口

- **问题**：目标 8880 端口返回一个未知界面。
- **方法论的使用**：作者用 `Server: WebSphere Application Server/7.0` 确认族系与版本，再由检查表直接访问 8880，把“出现特定界面”作为 CVE-2015-7450 的初判条件。
- **结论**：先确认容器，才能把一个陌生界面解释为“反序列化入口”。
- **结果**：按书中的 SOAP/JMX 请求复现成功。

### 案例 3：默认页面暴露管理面

- **问题**：Tomcat 目标只显示 404。
- **方法论的使用**：作者按 Tomcat 检查表访问 `/manager/html`，发现管理页面存在，再判断是否允许本地访问、是否存在默认用户。
- **结论**：管理面是否存在，比“首页是否好看”更能决定攻击路径。
- **结果**：在弱口令配置下进入后台并部署 war；作者同时注明默认安装没有用户，页面存在不等于可弱口令。

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 拿到一个 IP/域名，只知道开了 Web 端口，不知道该查 IIS 还是 Nginx/Java 容器。
2. 响应头被抹掉或伪造，需要靠默认路径、错误页、固定 banner 反推容器。
3. 已经知道容器类型，但不知道先查解析、配置还是反序列化。
4. 手里有一份扫描结果，需要判断哪些命中具备可利用条件。

### 语言信号 (用户的话里出现这些就应激活)

- “这是什么中间件 / 什么服务器”
- “Server 头是空的，怎么判断”
- “8080/7001/4848/9060 端口能做什么”
- “middleware fingerprint / which server / identify the container”
- “先查哪个漏洞”

### 与相邻 skill 的区分（定稿）

- 与 `upload-parser-bypass-audit`：本 skill 只输出“容器 + 版本区间 + 优先级”，分号截断、多后缀、`fix_pathinfo` 等具体验证交给后者；容器未确认时不要先试解析 payload。
- 与 `java-deserialization-entry-enumeration`：本 skill 判断“是不是 Java 容器、管理面是否暴露”，后者枚举具体反序列化入口并判读补丁黑名单。
- 与 `middleware-misconfig-audit`：本 skill 用默认路径、默认页面与响应头发现配置线索，后者把线索扩展成逐项配置审计。
- 与通用漏洞扫描的区别：本 skill 不追求覆盖率，只做族系判定与闸门筛选。

---

## E — 可执行步骤 (Execution)

1. **收集指纹**
   - 完成标准：至少拿到 2 类证据（响应头、默认页面/错误页、固定 banner、默认端口）。
   - 判停条件：若响应头被 WAF 统一改写，跳到步骤 2 的路径探测。

2. **路径与 banner 探测**
   - 完成标准：对候选容器各试 2–3 个默认路径（如 `/manager/html`、`/invoker/readonly`、`/console`、`/wls-wsat/CoordinatorPortType`、`/theme/META-INF/`），记录状态码与响应特征。
   - 注意：只做低影响 GET/HEAD；需要认证或写入的路径不在此步验证。

3. **映射容器与版本区间**
   - 完成标准：给出“容器 + 版本区间 + 依据”的一行结论；依据不足时标注“低置信度”。

4. **分派检查表并过闸门**
   - 完成标准：列出 3–5 个候选漏洞面，每个都回答“版本是否命中、组件是否开启、是否需要认证”。
   - 判停条件：若全部候选都被闸门排除，停止，不进入利用验证。

5. **最小验证并记录证据**
   - 完成标准：每条结论都有可复现的请求与响应差异（状态码、错误文本、回显），并注明测试时间与授权范围。

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 目标是纯 Web 应用层漏洞（SQL 注入、XSS、越权），容器指纹不改变检查路径。
- 已有明确资产清单与版本，只需要修复，不需要分诊。
- 没有授权，或目标不在约定范围；本 skill 的所有探测都必须先过授权门禁。

### 作者在书中警告的失败模式

- 只凭“页面存在/返回 500”就判定漏洞存在（ce12）；页面存在可能只是默认组件，不等于可利用。
- 改完 IIS 配置未重启或未回收 `w3wp`，测试结论无效（ce09）。
- Tomcat manager 默认没有用户且只允许本地访问，页面存在不等于可弱口令（ce16）。

### 作者的盲点 / 时代局限

- 内容截至 2019 年，默认端口、默认页面与版本区间已变化；指纹只能作为线索，必须实测。
- 未讨论容器化与云原生网关（Ingress、Envoy、内嵌 Tomcat），这些场景的指纹面不同。
- 书中默认测试者在自有靶场操作，缺少授权与边界的显式声明。

### 容易混淆的邻近方法论

- “端口扫描”只给开放服务，不给容器族系；不要把端口等同于产品。
- “版本比对”只回答“是否在影响范围”，不回答“组件是否开启、是否需要认证”。

---

## 相关 skills

- **composes-with** `upload-parser-bypass-audit` — 指纹给出容器与版本，解析绕过审计据此选择 IIS/Apache/Nginx 的验证路径。
- **composes-with** `java-deserialization-entry-enumeration` — 指纹先判断是否 Java 容器，入口枚举再展开具体 URI 与 gadget。
- **composes-with** `middleware-misconfig-audit` — 默认页面与错误页既是指纹证据，也是配置问题的第一手线索。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: 待阶段 4 填写
- **蒸馏时间**: 2026-10-19
