---
name: middleware-misconfig-audit
description: |
  当用户要审计 Web 中间件的配置型漏洞（没有 CVE 的漏洞）时调用。适用于 Nginx alias 目录穿越、autoindex 目录遍历、$uri 解码 CRLF、add_header 继承覆盖、Tomcat readonly、WebDAV 写权限、管理面暴露，也适用于配置漂移、CDN 边缘与源站不一致、无配置文件时的黑盒行为审计。不适用于：代码漏洞、内存破坏、反序列化、没有任何配置或行为证据的猜测。Trigger: 配置错误、alias 穿越、autoindex、CRLF 注入、add_header、readonly、WebDAV、配置漂移、CDN 源站、misconfiguration。
source_book: 《Web中间件常见漏洞总结》 lyxhh
source_chapter: Nginx/Tomcat/IIS 节的“配置错误”与“修复建议”
tags: [配置审计, Nginx, Tomcat, IIS, web]
related_skills:
  - slug: middleware-fingerprint-and-triage
    relation: composes-with
  - slug: upload-parser-bypass-audit
    relation: contrasts-with
  - slug: middleware-remediation-layering
    relation: composes-with
---

# 中间件配置错误审计

## R — 原文 (Reading)

> “Nginx 在配置别名（Alias）的时候，如果忘记加 /，将造成一个目录穿越漏洞。”
>
> — lyxhh，《Web中间件常见漏洞总结》Nginx 节

---

## I — 方法论骨架 (Interpretation)

配置型漏洞的共同点是：**没有 CVE、扫描器覆盖不到、但影响可以很直接**。审计方法不是“看配置写得像不像安全”，而是逐项比较“管理员以为的行为”和“服务器实际的行为”。

高频缝隙有六类：

1. **路径拼接**：`alias` 与 `location` 的尾斜杠不一致，导致 `../` 逃出预期目录。
2. **默认开关**：`autoindex on` 泄露目录结构；`readonly=false` 允许 PUT 写文件；WebDAV 开启后扩大方法面。
3. **变量解码**：把 `$uri`/`$document_uri` 拼进 `Location` 等响应头，解码后引入 `%0d%0a`，形成 CRLF 注入。
4. **继承覆盖**：Nginx 子块一旦写了自己的 `add_header`，父块的安全头不再继承，CSP 等整体失效。
5. **管理面暴露**：manager、console、JMX、测试页默认存在或只做“本地访问”假设。
6. **运行模式**：开发/生产模式切换会改变测试页、上传点与认证要求。

验证时用低影响请求：目录列表看是否返回索引页；别名穿越读一个已知的公共文件；CRLF 只验证响应头是否被拆分，不注入恶意内容。

---

## A1 — 书中的应用 (Past Application)

### 案例 1：alias 少写斜杠导致目录穿越

- **问题**：配置本意是让用户访问 `C:/WWW/home/`，但写成了 `location /files { alias c:/WWW/home/; }`。
- **方法论的使用**：作者对比 `location`（无尾斜杠）与 `alias`（有尾斜杠），说明路径拼接会多出一段，攻击者可用 `../` 跳出。
- **结论**：修复只需保证两边“都有斜杠”或“都没有斜杠”。
- **结果**：复现成功；修复为斜杠一致。

### 案例 2：`$uri` 解码导致 CRLF 注入

- **问题**：站点用 `$uri` 做跳转，能否注入响应头。
- **方法论的使用**：作者指出 `$uri`/`$document_uri` 是解码后的路径，`%0a%0d` 会被还原为换行；`$request_uri` 保留原始编码。
- **结论**：响应头与重定向只能用 `$request_uri`。
- **结果**：构造请求后 `Location` 被拆分并在响应中执行 XSS；修复为 `return 302 https://$host$request_uri;`。

### 案例 3：子块 `add_header` 覆盖父块

- **问题**：全站配置了 CSP，但 `/test2` 仍能触发 XSS。
- **方法论的使用**：作者检查 `/test2` 的 `location`，发现它自己加了 `X-Content-Type-Options`，导致父块 `add_header` 全部不继承。
- **结论**：Nginx 的 `add_header` 是覆盖而非追加；子块必须重复声明所有需要的头。
- **结果**：CSP 实际未下发，XSS 成立；修复需在子块补齐安全头。

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 拿到 Nginx/Apache/Tomcat 配置，要求做安全审计或加固。
2. 黑盒测试中发现目录列表、可疑跳转、XSS 与全站 CSP 声明不一致。
3. 上传或写入类问题怀疑与 `readonly`、WebDAV、脚本执行权限有关。
4. 管理后台暴露，需要判断是“默认配置”还是“业务必需”。

### 语言信号 (用户的话里出现这些就应激活)

- “帮我看看这段 Nginx 配置有没有问题”
- “alias 目录穿越 / autoindex / CRLF 注入”
- “为什么配了 CSP 还是能 XSS”
- “readonly / WebDAV / PUT 写入”
- “线上配置和 Git 里不一致 / CDN 边缘和源站不一致”
- “只有黑盒、没有配置文件，怎么审配置”
- “misconfiguration audit / harden the middleware”

### 与相邻 skill 的区分（定稿）

- 与 `middleware-fingerprint-and-triage`：当问题已经变成“配置是否有问题”（包括只有黑盒、需要按行为反推配置）时用本 skill；若问题还停留在“这是什么服务器”，才用指纹 skill。
- 与 `upload-parser-bypass-audit`：本 skill 关注路径拼接、开关与响应头；解析绕过关注文件是否被执行。上传目录禁执行属于两者交集，需要分别确认。
- 与 `middleware-remediation-layering`：本 skill 负责发现问题与证据，后者负责排修复顺序与评估副作用。
- 与代码审计的区别：本 skill 不需要源码，只需要配置或可观测行为。

---

## E — 可执行步骤 (Execution)

1. **建立配置/行为清单**
   - 完成标准：列出所有 `server`/`location`（或 IIS 站点、Tomcat Context），标注静态目录、代理、重定向、上传目录。

2. **查路径拼接**
   - 完成标准：逐条比对 `location` 与 `alias`/`root` 的尾斜杠；对可疑项用 `../` 读一个已知公共文件验证。

3. **查默认开关**
   - 完成标准：确认 `autoindex`、`readonly`、WebDAV、CGI、测试页、管理页的实际状态；区分“默认值”与“当前值”。

4. **查响应头与变量**
   - 完成标准：确认重定向/响应头是否使用 `$request_uri`；确认子块是否覆盖了父块安全头；用无害换行验证是否可拆分响应头。

5. **查管理面**
   - 完成标准：列出管理路径、绑定地址、认证方式与默认账号状态；判断是否仅靠“内网”假设。

6. **输出问题与证据**
   - 完成标准：每条问题都有“配置位置/请求 → 实际行为 → 影响 → 修复建议”，并标注是否需要停机或重启。

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 问题是代码层漏洞（SQL 注入、命令注入）或内存破坏，配置审计解释不了。
- 没有任何配置访问权，也无法做黑盒验证；此时只能列假设，不能下结论。
- 目标使用 CDN/WAF 托管配置，边缘与源站行为不一致；需要先确认实际生效层。

### 作者在书中警告的失败模式

- 子块 `add_header` 会整体覆盖父块安全头（ce15）；不要以为“父块配了就等于全站生效”。
- 使用解码后的 `$uri` 拼响应会导致 CRLF 注入（ce14）。
- `alias` 与 `location` 斜杠不一致会穿越（ce20）。
- 改完 IIS 配置未重启，测试结论无效（ce09）。

### 作者的盲点 / 时代局限

- 只覆盖 Nginx/Apache/Tomcat/IIS 的经典配置，未涉及容器化网关、Ingress 注解与云 WAF。
- 未讨论配置漂移与 IaC 审计；手工检查难以持续。
- 默认“测试者能改配置并重启”，生产环境往往不允许。

### 容易混淆的邻近方法论

- 配置审计 ≠ 漏洞扫描：扫描器只能发现已知模式，配置审计要找“意图与行为不一致”。
- 配置审计 ≠ 代码审计：前者看部署层，后者看实现层。

---

## 相关 skills

- **composes-with** `middleware-fingerprint-and-triage` — 指纹阶段看到的默认页面、错误页与默认端口，是配置审计的入口线索。
- **contrasts-with** `upload-parser-bypass-audit` — 一个追配置意图与实际行为的偏差，一个追存储名与解析名的偏差。
- **composes-with** `middleware-remediation-layering` — 每条配置问题都要转成“删组件 / 改配置 / 打补丁”的修复选项。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: 待阶段 4 填写
- **蒸馏时间**: 2026-10-19
