---
name: upload-parser-bypass-audit
description: |
  当用户发现上传点、想知道文件会不会被当脚本执行，或要审计 IIS/Apache/Nginx 的解析漏洞时调用。适用于比较存储名与解析名、验证分号/多后缀/AddHandler/fix_pathinfo/空字节/文件名逻辑，以及给出上传目录禁执行等修复。不适用于：反序列化、纯 Content-Type 绕过、没有上传入口、未授权测试。Trigger: 解析漏洞、上传绕过、.php.jpg、AddHandler、cgi.fix_pathinfo、PATH_INFO、parser bypass、upload bypass。
source_book: 《Web中间件常见漏洞总结》 lyxhh
source_chapter: IIS/Apache/Nginx 三节的“解析漏洞”
tags: [解析漏洞, 文件上传, IIS, Apache, Nginx, web]
related_skills:
  - slug: middleware-fingerprint-and-triage
    relation: depends-on
  - slug: middleware-misconfig-audit
    relation: contrasts-with
  - slug: middleware-remediation-layering
    relation: composes-with
---

# 上传解析绕过审计

## R — 原文 (Reading)

> “Apache 的解析漏洞依赖于一个特性：Apache 默认一个文件可以有多个以点分割的后缀，当最右边的后缀无法识别（不在 mime.types 文件内），则继续向左识别，直到识别到合法后缀才进行解析。”
>
> — lyxhh，《Web中间件常见漏洞总结》Apache 节

---

## I — 方法论骨架 (Interpretation)

解析漏洞的统一判据只有一句：**文件“存进去时的名字”和“被执行时的解析名”不是同一个东西**。审计时不要背 payload，而要找出这两者在哪一层分叉。

常见分叉点有四类：

1. **截断类**：IIS 6 把 `;` 之后当截断，`*.asp;.jpg` 按 asp 解析；空字节 `%00` 让后续后缀失效。
2. **目录类**：IIS 6 把 `*.asp/` 目录下的任意文件按 asp 解析。
3. **路径回退类**：PHP 的 `cgi.fix_pathinfo=1` 在完整路径不存在时逐段去掉尾部继续找，造成 `info.jpg/xxx.php` 按 jpg 内容执行。
4. **多后缀类**：Apache 从右向左找可识别后缀；若配置了 `AddHandler ... .php`，则文件名任意位置含 `.php` 即解析。

验证时先用无害标记文件（例如输出固定字符串的脚本）证明“被执行”，不要直接上传 webshell。修复有三层：统一解析规则（关 `fix_pathinfo`、限 `security.limit_extensions`）、禁止上传目录执行、上传后重命名。三层叠加才算收敛。

---

## A1 — 书中的应用 (Past Application)

### 案例 1：IIS 6 分号截断

- **问题**：上传白名单只允许图片，如何让文件按 asp 执行。
- **方法论的使用**：作者指出 IIS 6 默认不解析 `;` 及其后内容，`*.asp;.jpg` 会被当成 asp；另外 `.asa`、`.cdx`、`.cer` 也调用 `asp.dll`。
- **结论**：绕过点在“文件名截断 + 隐藏的可执行后缀映射”。
- **结果**：上传后按 asp 解析；修复建议是限制上传目录执行权限、禁止新建目录、上传后重命名。

### 案例 2：Apache 的多后缀与 AddHandler

- **问题**：`phpinfo.php.jpg` 是否会被解析。
- **方法论的使用**：作者用它说明 Apache 会从右向左识别：认识 `.jpg` 就停止，因此不解析；但若运维加了 `AddHandler application/x-httpd-php .php`，文件名任意位置含 `.php` 就会解析。
- **结论**：同一个服务器，配置不同，结论相反；必须看 `mime.types` 与 `AddHandler`。
- **结果**：`phpinfo.php.xxx` 在未知后缀下解析；`AddHandler` 场景下 `test.jpg.php.xxx` 也可解析。

### 案例 3：Nginx 的 fix_pathinfo 与文件名逻辑

- **问题**：`info.jpg/xxx.php` 为什么会被当 PHP 执行。
- **方法论的使用**：作者沿 `location ~ \.php$` → FastCGI → `cgi.fix_pathinfo` 的链条解释：Nginx 看到 `.php` 结尾就转给 PHP，PHP 找不到 `xxx.php` 后逐段回退，最终执行 `info.jpg` 的内容。
- **结论**：漏洞在“Nginx 的后缀判断”和“PHP 的路径修复”之间，不在 Nginx 版本。
- **结果**：复现成功；修复为 `cgi.fix_pathinfo=0`、`security.limit_extensions=.php` 或禁止上传目录执行。

---

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?

1. 渗透测试中发现上传点，白名单只允许 jpg/png，想知道能否解析执行。
2. 代码审计看到 `move_uploaded_file`、`basename`、`pathinfo`，需要判断服务器层是否会二次解释。
3. 运维/开发要确认“上传目录禁执行”是否足够，或需要给 Nginx/PHP 出加固配置。
4. 复现某个解析 CVE，需要判断它属于截断、目录、回退还是多后缀。

### 语言信号 (用户的话里出现这些就应激活)

- “上传点能不能 getshell”
- “`.php.jpg` 会不会解析”
- “解析漏洞 / parser bypass / upload bypass”
- “`cgi.fix_pathinfo` / `PATH_INFO` / `AddHandler`”
- “为什么 `info.jpg/xxx.php` 能执行”

### 与相邻 skill 的区分（定稿）

- 与 `middleware-fingerprint-and-triage`：本 skill 依赖后者给出的容器族系与版本区间；容器未确认时先回去做指纹，不要盲试各类 payload。
- 与 `middleware-misconfig-audit`：两者都在看“服务器如何解释路径”，但本 skill 的判据是“存储名 vs 解析名”，后者是“配置意图 vs 实际行为”（alias 斜杠、autoindex、安全头继承）；一条上传链的最终结论常需要两者合看。
- 与 `middleware-remediation-layering`：本 skill 只回答“能不能执行、在哪一层分叉”，修复怎么排优先级由后者决定。
- 与纯“文件上传漏洞”常识的区别：本 skill 不满足于扩展名黑名单，而是定位“存储名 vs 解析名”的分叉层。

---

## E — 可执行步骤 (Execution)

1. **确认上传入口与存储规则**
   - 完成标准：知道文件存到哪、是否重命名、是否允许控制目录名、返回的访问 URL。
   - 判停条件：若服务端完全重命名为固定扩展名且不可控目录，跳到步骤 5 只做修复审计。

2. **画出解析链**
   - 完成标准：列出“Web 服务器 → 动态处理器 → 运行时配置”三层，并标注每层的后缀/路径判断规则（含 `mime.types`、`AddHandler`、`location ~ \.php$`、`fix_pathinfo`、`security.limit_extensions`）。

3. **用无害标记验证**
   - 完成标准：上传一个只输出固定字符串的测试文件，证明“被执行”而非“被下载”。
   - 判停条件：若只能证明“被下载”，停止，记录为不可执行。

4. **逐个验证分叉类型**
   - 完成标准：对截断、目录、路径回退、多后缀四类各给一个“成立/不成立 + 证据”的结论；不成立的也要写原因。

5. **给出三层修复**
   - 完成标准：同时给出“统一解析规则、禁止上传目录执行、上传后重命名”的具体配置或代码位置，并说明哪一层是主修复。

---

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill

- 目标没有上传入口，或上传后文件不落在 Web 可访问路径。
- 问题属于反序列化、SQL 注入、XSS 等非解析类漏洞。
- 只讨论 `Content-Type` 伪造而服务器根本不看该字段；这属于另一类校验问题。
- 未授权测试：所有上传验证必须先确认范围与允许的测试方式。

### 作者在书中警告的失败模式

- 只看最后一个后缀，会漏掉 `AddHandler` 的“文件名任意位置含 `.php`”（ce24）。
- 复现换行解析时用 `$_FILES['file']['name']` 会丢掉换行，导致结论错误（ce19）。
- CVE-2013-4547 在 Windows 上因不允许尾随空格无法复现，需 Linux/Docker（ce25）。
- 空字节与文件名逻辑漏洞不受 `cgi.fix_pathinfo` 影响，关掉它仍可能解析（p23）。

### 作者的盲点 / 时代局限

- 以 IIS 6/7、PHP 5.x、Nginx 0.x–1.4 为主；现代 PHP 默认 `security.limit_extensions`、IIS 10 与 Nginx 新版本已改变部分前提。
- 未讨论对象存储、CDN 回源、Serverless 等“文件不在本机”的场景。
- 未给出“如何证明无害”的测试规范，读者容易直接上传 webshell。

### 容易混淆的邻近方法论

- “扩展名黑名单绕过”只是本方法论的一个子集；真正的判据是解析层分叉。
- “任意文件上传”强调写入位置，本 skill 强调写入后的解析行为。

---

## 相关 skills

- **depends-on** `middleware-fingerprint-and-triage` — 先确认 IIS/Apache/Nginx 与版本，才能选对截断、目录、路径回退、多后缀四类判据。
- **contrasts-with** `middleware-misconfig-audit` — 同看路径解释，但一个追“存储名 ≠ 解析名”，一个追“配置意图 ≠ 实际行为”。
- **composes-with** `middleware-remediation-layering` — 验证出的分叉层直接决定修复该落在统一解析规则、禁止执行还是重命名。

---

## 审计信息

- **验证通过**: V1 ✓ / V2 ✓ / V3 ✓
- **测试通过率**: 待阶段 4 填写
- **蒸馏时间**: 2026-10-19
