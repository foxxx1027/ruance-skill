---
name: api-and-interface-security
description: |
  当用户要审 REST、GraphQL、WebSocket、RPC 接口，或发现 Swagger/OpenAPI 暴露、接口无限流时使用。先枚举接口与协议，再逐接口查认证、授权与对象级校验，再查批量赋值与过度数据暴露，最后查限流、分页、复杂度与文档调试面。关键信号：GraphQL introspection 开放、响应含他人字段、接口可枚举。不适用于通用注入（non-memory-vulnerability-analysis）与会话 JWT 细节（auth-session-and-federation-security）。
source_book: 工程清单型：用户提供的《漏洞筛查覆盖框架》（2026-10-19）；参考 OWASP Top 10 2021 / CWE / NIST
source_chapter: 第 8 类「API 与接口安全」
tags: [API, GraphQL, WebSocket, RPC, 批量赋值, 限流, OpenAPI]
related_skills:
  - slug: access-control-and-idor
    relation: composes-with
  - slug: auth-session-and-federation-security
    relation: composes-with
  - slug: business-logic-and-race-security
    relation: contrasts-with
  - slug: non-memory-vulnerability-analysis
    relation: contrasts-with
  - slug: ssrf-xxe-and-template-injection
    relation: contrasts-with
  - slug: client-side-and-frontend-security
    relation: composes-with
---

# API 与接口安全（工程检查清单）

## R — 原文 (Reading)

> REST、GraphQL、WebSocket、RPC 接口
>
> 鉴权、速率限制、参数校验
>
> 批量赋值、过度数据暴露
>
> Swagger/OpenAPI 文档暴露、GraphQL introspection
>
> — 用户，《漏洞筛查覆盖框架》第 8 类「API 与接口安全」，2026-10-19

## I — 方法论骨架 (Interpretation)

1. **先枚举接口与协议，建立可核对的接口清单。** 从网关配置、前端与移动端代码、流量、OpenAPI/Swagger、GraphQL schema、WebSocket 握手与消息、RPC 方法列表、BFF 与内部版本中收集端点；标注协议、方法、认证方式、调用方、版本、是否对外。只扫页面或只看文档会漏掉移动端、旧版本、内部接口和 WebSocket 消息级操作。

2. **再逐接口判认证，确认“每个入口都要求身份”。** 检查匿名可访问的端点，尤其是 OPTIONS/HEAD、健康检查、指标、内部版本、文件下载、Webhook、GraphQL 单端点内的公开字段；确认令牌在网关与应用两层都被验证，不能只依赖前端携带。认证缺失是实现或配置问题；把公开端点误判为漏洞则需要业务确认其设计意图。

3. **判功能级与对象级授权，这是 API 风险最集中的一段。** 功能级看角色能否调用管理或高权操作；对象级看接口是否用当前主体校验目标对象归属、租户、状态。REST 要覆盖路径、查询、请求体、批量列表、嵌套资源、导出；GraphQL 要覆盖 query、mutation、字段 resolver 与节点 ID；WebSocket 要覆盖每条消息与订阅频道；RPC 要覆盖每个方法。不要因为“同一控制器已有权限注解”就假设所有方法都受保护。

4. **判输入：参数校验、类型与批量赋值。** 确认服务端有 schema 或等价校验，拒绝未知字段或使用允许列表；重点检查注册、资料更新、邀请、导入、GraphQL mutation 是否接受 role、isAdmin、tenantId、ownerId、status 等敏感字段。还要看 Content-Type 处理、参数重复与污染、数组/对象类型混淆、超长与深层嵌套。前端校验不算数。

5. **判输出：过度数据暴露与错误信息。** 对照不同角色与不同对象，检查响应是否返回了不该返回的字段（手机号、身份证、内部备注、成本价、其他租户数据）、是否把数据库对象直接序列化、分页与导出是否越界、错误是否泄露堆栈或内部路径。过度暴露可能是设计缺陷（响应模型未分级）或实现缺陷（字段过滤遗漏）。

6. **判资源控制：限流、配额、分页与复杂度。** 检查按 IP、账号、令牌、租户、操作维度的速率限制与配额；分页是否有最大页大小与总量上限；GraphQL 是否限制查询深度、复杂度与别名数量；WebSocket 是否限制连接数、订阅数与消息频率；RPC 是否有超时与并发上限。只在前端做节流不构成服务端资源控制。

7. **判文档与调试面。** Swagger/OpenAPI、GraphiQL、GraphQL introspection、调试端点、指标、健康检查详情、测试接口、旧版本文档是否对非授权方开放；文档是否暴露内部主机、密钥、示例账号或未上线接口。文档暴露本身通常是配置问题，但会显著扩大攻击面，应单独记录。

8. **验证顺序：匿名基线 → 双账号对照 → 字段与批量 → 资源边界 → 文档面。** 先用匿名请求建立基线，再用两个自己控制的账号做对象级 A/B，再用低权账号调用高权操作，随后测试未知字段、敏感字段、超长/深层输入与分页边界，最后检查文档与调试面。所有主动请求仅在授权范围内，写操作只对测试数据；记录请求、响应、状态码、耗时与限流头。最后把问题归为设计缺陷、实现缺陷或配置问题。

## A1 — 书中的应用 (Past Application)

> 本 skill 为工程清单型，没有单一书籍案例；以下为公开标准条款映射，不虚构事件细节。

### 标准映射 1: OWASP A05:2021 Security Misconfiguration
- **条款**: OWASP Top 10 2021 A05:2021 Security Misconfiguration。
- **对应检查**: 真实系统里通常表现为：Swagger/OpenAPI 或 GraphiQL 在生产对公网开放；GraphQL introspection 未关闭；调试、指标、健康检查返回内部细节；默认错误页暴露堆栈；不必要的 HTTP 方法与跨域规则被放开。
- **落到本 skill**: 用 I 段第 7 模块和 E 段文档面步骤，逐项记录文档/调试端点的可达性与返回内容，产出“端点—认证要求—暴露内容—应关闭/应鉴权”的证据表。

### 标准映射 2: CWE-915 Improperly Controlled Modification of Dynamically-Determined Object Attributes
- **条款**: CWE-915 Improperly Controlled Modification of Dynamically-Determined Object Attributes（批量赋值/对象属性批量改写）。
- **对应检查**: 接口把请求体自动绑定到内部对象，攻击者提交额外字段（role、isAdmin、tenantId、ownerId、status、permissions）后，这些字段被写入并生效；常见于注册、资料更新、邀请、导入、GraphQL mutation 与 RPC 参数对象。
- **落到本 skill**: 用 I 段第 4 模块和 E 段字段验证步骤，逐接口列出可写字段与生效结果，确认服务端是否使用允许列表与字段级授权；产出“接口—敏感字段—是否可写—是否生效”矩阵。

## A2 — 触发场景 (Future Trigger) ★

### 用户会在什么情境下需要这个 skill?
- 新上线或改版 REST API，需要按接口检查认证、授权、参数校验、限流与文档暴露。
- 使用 GraphQL，担心 introspection 开放、单端点绕过鉴权、字段级数据泄露或复杂查询拖垮服务。
- 使用 WebSocket 或 RPC，连接建立后消息级操作缺少权限校验或频率限制。
- 接口响应返回了手机号、身份证、内部备注、成本价等不该给当前角色的字段，怀疑过度数据暴露。
- 注册或资料更新接口能传入 role、tenantId 等字段并生效，怀疑批量赋值导致提权。
- Swagger/OpenAPI、GraphiQL、调试接口在生产可直接访问，需要评估暴露面并按接口整改。

### 语言信号 (用户的话里出现这些就应激活)
- “Swagger 页面公网能打开”
- “GraphQL introspection 没关”
- “一个 GraphQL 端点什么都能查”
- “接口返回了别人的手机号”
- “响应里带内部备注和成本价”
- “注册传 isAdmin=true 就成管理员”
- “mutation 能改任意字段”
- “WebSocket 连上以后就不校验权限了”
- “RPC 接口没有限流”
- “接口能翻页翻出全量数据”

英文关键词：GraphQL introspection、mass assignment、excessive data exposure、rate limiting、WebSocket authorization。

### 与相邻 skill 的区分
- 会话固定、JWT alg、OAuth/SAML/SSO 信任链：走 `auth-session-and-federation-security`；本 skill 只检查接口是否要求并正确使用身份，不深入令牌机制。
- 授权矩阵、水平/垂直越权、IDOR 的系统化建模：走 `access-control-and-idor`；本 skill 按接口逐点执行对象级与功能级检查，权限模型设计由它负责。
- 支付、退款、优惠券、订单状态机、并发与重放：走 `business-logic-and-race-security`；本 skill 不判断业务流程不变量。
- SQL 注入、命令注入、XSS、路径穿越等通用注入：走 `non-memory-vulnerability-analysis`。
- SSRF、XXE、模板注入：走 `ssrf-xxe-and-template-injection`。
- CORS、CSRF、点击劫持、postMessage、前端令牌存储：走 `client-side-and-frontend-security`；本 skill 只检查 API 侧的跨域与认证配合。
- 管理端口、中间件默认配置、容器与部署面：走 `middleware-misconfig-audit`；本 skill 只处理应用接口与文档面。

## E — 可执行步骤 (Execution)

1. **建接口清单**
   - 动作: 从网关、前端/移动端代码、流量、OpenAPI、GraphQL schema、WebSocket 握手、RPC 方法列表收集端点；标注协议、方法、认证方式、调用方、版本、对外/内部。
   - 完成标准: 清单覆盖 REST、GraphQL、WebSocket、RPC 与旧版本接口，没有只存在于文档或只存在于流量的盲区。
   - 判停条件: 若接口来源不完整，先停在清单阶段补资产与流量证据，不进入逐接口验证。

2. **建立匿名访问基线**
   - 动作: 在授权范围内用匿名会话请求清单中的端点，记录状态码、响应体、限流头与是否触发副作用；重点看健康检查、指标、文档、下载、Webhook 与内部版本。
   - 完成标准: 每个端点都有“匿名可达/不可达/需业务确认”的结论。
   - 判停条件: 若请求会触发真实写操作或通知，停止主动请求，改用路由与网关配置核查。

3. **逐接口检查认证**
   - 动作: 确认令牌在网关与应用两层都被验证；检查 OPTIONS/HEAD、WebSocket 握手、GraphQL 单端点、RPC 方法是否都要求身份；检查是否存在“只在网关校验、应用内直连可绕过”的路径。
   - 完成标准: 形成“端点 × 认证方式 × 校验位置 × 匿名结果”表。
   - 判停条件: 无，继续。

4. **逐接口检查功能级与对象级授权**
   - 动作: 用低权账号调用高权操作；用两个自己控制的同权账号做对象级 A/B，覆盖路径 ID、查询参数、请求体、批量列表、嵌套资源、导出、GraphQL 节点与字段、WebSocket 消息、RPC 参数。主动写操作只在测试数据上做。
   - 完成标准: 每个端点都有“主体—动作—对象—预期/实际”记录，GraphQL 与 WebSocket 细化到字段和消息。
   - 判停条件: 一旦涉及真实用户数据或生产写操作，立即停止，改在测试环境复现。

5. **检查参数校验与批量赋值**
   - 动作: 提交未知字段、敏感字段（role、isAdmin、tenantId、ownerId、status、permissions）、错误类型、重复参数、超长与深层嵌套；确认服务端是否使用 schema/允许列表，敏感字段是否被写入并生效。
   - 完成标准: 形成“接口—可写字段—敏感字段是否生效”矩阵，未知字段有明确接受/拒绝结论。
   - 判停条件: 无，继续。

6. **检查过度数据暴露**
   - 动作: 对照匿名、普通用户、管理员、跨租户四种视角，检查响应字段、分页与导出是否返回不应可见的数据；检查错误响应是否泄露堆栈、内部路径或配置。
   - 完成标准: 每个响应模型都有“字段 × 可见角色”的对照表，异常字段附原始响应。
   - 判停条件: 无，继续。

7. **检查资源控制**
   - 动作: 验证按 IP、账号、令牌、租户、操作的限流与配额；检查分页最大页大小、总量上限、导出上限；GraphQL 检查深度、复杂度、别名；WebSocket 检查连接数、订阅数与消息频率；RPC 检查超时与并发。
   - 完成标准: 形成“端点 × 限流维度 × 阈值 × 超限行为”表。
   - 判停条件: 若主动压测可能影响可用性，停止并改用配置与网关策略核查。

8. **检查文档与调试面**
   - 动作: 检查 Swagger/OpenAPI、GraphiQL、introspection、调试端点、指标、健康检查详情、测试接口、旧版本文档的可达性与认证要求；确认文档是否暴露内部主机、密钥、示例账号或未上线接口。
   - 完成标准: 每个文档/调试端点都有“可达性—认证—暴露内容—处置建议”记录。
   - 判停条件: 无，继续。

9. **汇总证据、定性与修复建议**
   - 动作: 将问题归为设计缺陷（响应模型未分级、授权模型缺失）、实现缺陷（某端点漏校验、字段过滤遗漏）、配置问题（文档暴露、限流关闭）；给出影响、复现条件、证据、修复层级与复测方法。
   - 完成标准: 输出可排期的接口问题清单，每条能定位到具体端点、方法与字段。
   - 判停条件: 无，结束。

## B — 边界 (Boundary) ★

### 不要在以下情况使用此 skill
- 只问 JWT 签名、alg 混淆、OAuth/SAML/SSO 配置：走 `auth-session-and-federation-security`。
- 只问越权模型、RBAC/ABAC 设计、授权矩阵：走 `access-control-and-idor`。
- 只问支付、退款、优惠券、库存、审批流与并发：走 `business-logic-and-race-security`。
- 只问 SQL 注入、命令注入、XSS、路径穿越：走 `non-memory-vulnerability-analysis`。
- 只问 SSRF、XXE、模板注入：走 `ssrf-xxe-and-template-injection`。
- 只问 CORS、CSRF、点击劫持、postMessage、前端存储：走 `client-side-and-frontend-security`。
- 只有一份接口文档、没有环境、没有账号：只能做静态风险排序，不能断言接口存在漏洞。
- 未取得授权，或主动测试会压垮服务、读取真实数据、修改生产对象：不得执行。
- 目标是批量抓取数据、绕过真实接口配额或攻击第三方 API：不属于防御性审查范围。

### 作者在书中警告的失败模式
- 本清单未覆盖“文档里有权限、所有接口就安全”的误判：注解、网关规则与框架默认值都可能被单个路由绕过。
- 本清单未覆盖“只测 REST、不测 GraphQL/WebSocket/RPC”的失败：单端点与消息级操作最容易漏对象级校验。
- 本清单未覆盖“只看请求参数、不看响应字段”的失败：过度暴露往往在正常响应里，而不是错误响应里。
- 本清单未覆盖“前端做了校验、后端就安全”的失败：批量赋值与未知字段必须由服务端允许列表控制。
- 本清单未覆盖“限流只做在网关”的失败：内部直连、绕过网关或单账号多令牌都可能突破单层限流。
- 本清单未覆盖“文档只是文档”的失败：Swagger、introspection 与调试端点会直接扩大可攻击面。

### 作者的盲点 / 时代局限
- 本清单未覆盖 gRPC 反射、protobuf 兼容性、流式 RPC 与双向流的完整安全模型。
- 本清单未覆盖 GraphQL 联邦、订阅、持久化查询与字段级授权在复杂 schema 下的实现细节。
- 本清单未覆盖 API 网关、服务网格、mTLS、工作负载身份与零信任之间的组合配置。
- 本清单未覆盖 API 版本生命周期、废弃端点下线与客户端兼容性带来的长期暴露。
- 本清单未覆盖业务级配额、计费与滥用检测，这些需要业务数据与风控系统配合。

### 容易混淆的邻近方法论
- `auth-session-and-federation-security`：解决身份与令牌是否可信，本 skill 解决每个接口是否使用身份。
- `access-control-and-idor`：提供授权矩阵与越权模型，本 skill 按接口逐点执行检查。
- `business-logic-and-race-security`：处理跨步骤业务规则与并发，本 skill 处理接口层输入输出与资源控制。
- `non-memory-vulnerability-analysis`：处理通用注入，本 skill 不替代注入验证。
- `ssrf-xxe-and-template-injection`：处理服务端请求伪造、实体注入与模板注入，本 skill 只负责接口面枚举与分派。
- `client-side-and-frontend-security`：处理浏览器侧跨域与前端存储，本 skill 处理 API 侧配置与响应。

## 审计信息
- 来源类型: 工程清单型（非书籍蒸馏）
- 引文来源: 用户《漏洞筛查覆盖框架》第 8 类
- 未使用书中原文，未引用具体 CVE/事件
- 版本: 0.1.0
