---
name: java-deserialization-entry-enumeration
description: |
  当用户面对 Java 中间件（JBoss/WebLogic/WebSphere）怀疑反序列化，或补丁后旧 payload 失效时调用。适用于枚举 invoker/wls-wsat/_async/T3/JMX/SOAP 入口、读补丁黑名单预测绕过、判断 gadget 与版本/平台绑定、无外连时用回显验证。不适用于：非 Java 目标、从零找 gadget、内存破坏利用、未授权攻击。Trigger: 反序列化、JBoss invoker、WebLogic wls-wsat、T3、XMLDecoder、ysoserial、回显构造、deserialization、patch bypass。
source_book: 《Web中间件常见漏洞总结》 lyxhh
source_chapter: JBoss/WebLogic/WebSphere 三节
tags: [反序列化, Java中间件, JBoss, WebLogic, WebSphere, web]
related_skills:
  - slug: middleware-fingerprint-and-triage
    relation: depends-on
  - slug: middleware-remediation-layering
    relation: composes-with
---

# Java 中间件反序列化入口枚举

## R — 原文 (Reading)

> “漏洞不仅存在于 /wls-wsat/CoordinatorPortType。只要是在 wls-wsat 包中的 Uri 皆受到影响，可以查看 web.xml 得知所有受到影响的 Uri。”
>
> — lyxhh，《Web中间件常见漏洞总结》WebLogic 节

---

## I — 方法论骨架 (Interpretation)

Java 中间件的反序列化利用，顺序应当反过来：**先枚举入口，再选 gadget，最后才是变形 payload**。原因是 gadget 依赖具体版本与 classpath，换环境就失效；而入口是组件的路由与协议面，更稳定、更容易复用。

执行分四层：

1. **组件入口**：JBoss 的 `/invoker/readonly`、`/invoker/JMXInvokerServlet`、`/invoker/EJBInvokerServlet`、`/jbossmq-httpil/HTTPServerILServlet`；WebLogic 的 `wls-wsat` 八个 URI、`_async` 三个 URI；WebSphere 的 8880 SOAP/JMX。
2. **URI 全集**：补丁常常只封一个 servlet 路径；拿到组件 `web.xml` 后，把同一包内的所有 `url-pattern` 列出来，逐个验证。
3. **协议入口**：T3、JRMP、SOAPAction 等协议层入口，可能绕过 HTTP 层的访问控制。
4. **补丁判读**：把补丁当黑名单读——它过滤了哪些标签/类名，哪些等价语法没被过滤。书中 `object → void → class` 的三次演化就是范例。

验证阶段只证明“解析发生了”（例如让目标发起一次可观测的 DNS/HTTP 请求或回显固定字符串），不在生产环境执行破坏性命令。

---

## A1 — 书中的应用 (Past Application)

### 案例 1：JBoss 四个入口复用同一份 ser

- **问题**：`/invoker/readonly` 返回 500，是否说明漏洞不存在。
- **方法论的使用**：作者把 500 解释为“页面存在”的判据，先生成一次 `ReverseShellCommonsCollectionsHashMap.ser`，再分别发送到 `readonly`、`JMXInvokerServlet`、`EJBInvokerServlet` 与 `jbossmq-httpil/HTTPServerILServlet`。
- **结论**：同一 gadget 可以在同一产品的多个入口复用；入口比 payload 更值得枚举。
- **结果**：四个入口均可用同一份 ser；修复分别是删除 `http-invoker.sar`、加访问控制或升级。

### 案例 2：WebLogic 的 URI 全集

- **问题**：`/wls-wsat/CoordinatorPortType` 被补丁封了，是否还有别的入口。
- **方法论的使用**：作者直接查看 `wls-wsat` 组件下的 `web.xml`，列出八个受影响的 `url-pattern`，并指出 `_async` 组件还有三个入口。
- **结论**：补丁只封已知路径时，URI 全集枚举能找到遗漏入口。
- **结果**：`RegistrationPortTypeRPC` 等入口仍可发送 XMLDecoder 请求；后续 CVE-2019-2725 又用 `class` 标签绕过上一版黑名单。

### 案例 3：WebSphere 的协议入口

- **问题**：WebSphere 7.0 的 8880 端口界面是否意味着反序列化。
- **方法论的使用**：作者用 `Server` 头确认版本，访问 8880 看到固定界面后，按 SOAP/JMX 结构替换 `params` 中的序列化对象。
- **结论**：协议入口（SOAPAction/JMXConnectorContext）与 HTTP 入口要分开枚举。
- **结果**：`net user` 命令执行成功；修复为升级或打补丁。

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 目标运行 JBoss/WebLogic/WebSphere，想判断是否存在反序列化入口。
2. 已知 CVE 但目标已打补丁，需要判断补丁是否可绕、还有哪些入口。
3. 拿到一份 `web.xml` 或组件目录，想枚举所有受影响的 URI。
4. 有一个可用的 gadget，但不知道往哪个接口发送。

### 语言信号 (用户的话里出现这些就应激活)

- “反序列化 / deserialization / ysoserial”
- “JBoss invoker / WebLogic wls-wsat / T3 / JMXInvokerServlet”
- “XMLDecoder / EventData / readObject”
- “补丁打了还能不能打 / patch bypass”
- “这个 500 是不是说明有漏洞”

### 与相邻 skill 的区分（定稿）

- 与 `middleware-fingerprint-and-triage`：本 skill 依赖后者确认产品与版本（JBoss 4/6、WebLogic 10.3.6/12.1.3、WebSphere 7）；版本决定选用哪份入口清单与哪个 gadget。
- 与 `middleware-remediation-layering`：本 skill 负责枚举入口与无害验证，后者决定删组件、加访问控制还是等补丁。
- 与“从零挖掘 gadget”的区别：本 skill 假设已有 gadget 或公开利用链，重点是入口、版本与平台适配。

---

## E — 可执行步骤 (Execution)

1. **确认产品与版本**
   - 完成标准：拿到 `Server` 头、默认页面或配置目录，给出产品 + 版本区间；标注是否在已知影响范围。

2. **枚举默认组件入口**
   - 完成标准：对每个产品列出 3–8 个默认入口并逐个请求，记录状态码与固定响应特征。
   - 判停条件：若所有入口都 404 且无法读取配置，转为黑盒指纹判断，不继续猜测。

3. **读取组件 `web.xml` 做 URI 全集**
   - 完成标准：能拿到配置时，列出该组件所有 `url-pattern`；不能拿到时，用公开默认清单并标注“未验证”。

4. **判读补丁黑名单**
   - 完成标准：找到补丁过滤的标签/类/方法列表，列出至少 1 个未被过滤的等价语法；不能证明时明确写“未知”。

5. **无害验证**
   - 完成标准：用 DNS/HTTP 外带或固定回显证明解析发生；不使用破坏性命令，不写入生产目录。

6. **判断 gadget 适配**
   - 完成标准：确认 gadget 依赖的类在目标版本存在、平台（Windows/Linux）匹配；不匹配时换链而不是硬打。

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 目标不是 Java 中间件，或没有反序列化组件。
- 需要从零构造 gadget 链；这属于漏洞研究，不是入口枚举。
- 目标是内存破坏类漏洞（缓冲区溢出、UAF），与本方法论无关。
- 未授权环境；反序列化验证可能触发命令执行，必须严格限定在授权靶场。

### 作者在书中警告的失败模式

- 只凭“页面存在/返回 500”判定漏洞会误报（ce12）；必须用可观测的解析证据。
- 黑名单只封标签名，换等价标签即可绕过（ce03）。
- `Content-Type` 不是 `text/xml` 时 XMLDecoder 可能不解析（ce13）。
- 删除 `wls-wsat` 一级应用包可能有未知后果，Oracle 不建议（ce04）。
- 同一 gadget 不跨 WebLogic 版本可用（ce23）；ysoserial payload 与操作系统绑定（ce22）。
- JBoss 6 配 JDK 8 打开 JMX Console 会 500，不是组件不存在（ce17）。

### 作者的盲点 / 时代局限

- 以 JDK 6/7/8、WebLogic 10.3.6/12.1.3、JBoss 4/6 为主；现代 JDK 的反序列化过滤器（JEP 290）与高版本封堵未讨论。
- 未讨论内存马、类加载隔离与云原生 Java 运行时。
- 修复建议偏向“删组件/打补丁”，缺少长期治理（白名单、协议下线、最小化组件）。

### 容易混淆的邻近方法论

- “反序列化利用”通常指 gadget 构造；本 skill 指入口枚举与补丁判读。
- “补丁比较”是另一套方法（见其他书的 patch-diff skill）；本 skill 只读安全补丁的黑名单逻辑。

---

## 相关 skills

- **depends-on** `middleware-fingerprint-and-triage` — 产品与版本决定入口清单、可用 gadget 与平台适配。
- **composes-with** `middleware-remediation-layering` — 入口验证结果直接对应“删组件 / 加访问控制 / 打补丁”的选择。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: 待阶段 4 填写
- **蒸馏时间**: 2026-10-19
