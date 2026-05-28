# BinxinAdmin 架构决策记录（ADR）

> 记录所有重大架构决策及其理由。新决策追加在末尾。

---

## ADR-001：多租户采用"共享数据库 + tenant_id 隔离"

**决策**：所有业务表带 `tenant_id` 字段，通过全局查询作用域自动注入租户过滤。

**理由**：成本低、运维简单、适合中小 SaaS、开源用户易上手。预留"独立数据库"模式接口，未来大客户可升级。

**三层架构**：
- 平台层（Platform）：超级管理员，管理所有租户、套餐、全局配置。
- 租户层（Tenant）：租户管理员，管理本租户用户、角色、业务。
- 用户层（User）：C 端最终用户（小程序/H5/App）。

---

## ADR-002：双后端"契约先行"开发策略

**决策**：先定义框架无关的契约层（DB Schema、API 规范、错误码、业务逻辑伪代码），两个后端各自实现这份契约。每个模块按"契约 → ThinkPHP 实现 → Laravel 实现"紧挨着做。

**理由**：避免"全部做完 TP 再从头做 Laravel"的重复痛苦；业务逻辑只设计一次；Cursor 可一次性生成两个版本。

---

## ADR-003：后端统一分层

```
Controller → Service → (Repository) → Model
配套：Validate、Middleware、Plugin
```

Service 层是业务逻辑核心，框架无关性最强，两个框架的 Service 逻辑高度一致。

---

## ADR-004：主键用自增 BIGINT，不用雪花 ID

**决策**：主键 `id` 用 BIGINT UNSIGNED 自增。

**理由**：避免雪花 ID 超过 JS 安全整数（2^53）导致前端精度丢失。多租户隔离靠 `tenant_id` + 自增即可，无需分布式 ID。若未来分库分表再引入雪花并转字符串传输。

---

## ADR-005：API 业务码风格

**决策**：统一响应 `{code, message, data, timestamp, request_id}`，HTTP 状态码：业务错误用 200、认证失败用 401、系统错误用 5xx。

业务码 5 位分段：10xxx 通用 / 20xxx 认证 / 30xxx 用户 / 4xxxx 业务 / 5xxxx 系统。

---

## ADR-006：插件化业务模块

**决策**：核心模块是骨架，业务模块做成可插拔的插件（类似 WordPress 插件），支持安装/卸载/启用/禁用。

**理由**：接单时按需启用，不污染核心；开源生态可贡献插件。

---

## ADR-007：License 选 Apache 2.0 而非 MIT

**决策**：Apache 2.0。

**理由**：比 MIT 多专利授权与专利反诉条款，对可能商业化的开源项目保护更好；仍足够宽松利于推广。

---

## ADR-008：前端原子化 CSS 选 UnoCSS

**决策**：UnoCSS 而非 TailwindCSS。

**理由**：更快、按需生成、与 Vite 集成更好、可兼容 Tailwind 语法。

---

## ADR-009：C 端 UI 库选 wot-design-uni

**决策**：wot-design-uni 而非 uView。

**理由**：更现代、Vue 3 + TS 原生支持、维护活跃、设计风格更适合当代 App。

---

## ADR-010：账号体系用分离表

**决策**：平台管理员（bx_platform_admin）与租户管理员（bx_tenant_admin）物理分离，两套独立 RBAC，两个登录入口。

**理由**：平台与租户权限语义本质不同；物理隔离比逻辑隔离（tenant_id=0 魔术值）越权风险更低；安全边界从登录入口就分开。符合严肃企业级 SaaS 做法和项目的安全红线。

**代价**：开发量增加约 30-40%（两套账号/角色管理），可接受。

---

## ADR-011：统一归属抽象（owner_type + owner_id）

**决策**：所有"可归属"实体（应用、C 端用户、业务数据）统一用 `owner_type`（platform/tenant）+ `owner_id`（平台=0，租户=租户ID）标识归属。

**理由**：应用和 C 端用户既可挂平台直营，也可挂租户。用统一抽象贯穿全系统，让归属规则、越权校验逻辑只写一套并严格复用，降低复杂度和安全风险，也便于代码生成器统一处理。

**安全要求**：每个数据查询必须同时校验 owner_type + owner_id 两个维度，封装成统一的归属作用域（OwnerScope），严禁各处手写。

---

## ADR-012：应用（Application）为独立实体

**决策**：bx_application 表为独立实体，`app_type` 区分 wechat_mini（微信小程序）/ wechat_oa（公众号）/ h5 / app。每个应用有独立的 appid、密钥、第三方配置。一个 owner（平台或租户）下可有多个应用。

**理由**：满足"一个后台管理多个小程序/应用"的核心需求。

---

## ADR-013：敏感配置平台统一分配

**决策**：租户的应用 appid、支付密钥、短信密钥等敏感配置由平台代填或审核，租户不能自填。所有密钥类字段在数据库中 AES-256 加密存储，传输到前端时脱敏（掩码显示）。

**理由**：安全优先，防止租户误配或泄露；平台对所有接入有管控。

---

## ADR-014：后端多应用/多端模式

**决策**：后端用多应用模式组织代码。
- ThinkPHP 8：app/platform（平台后台）、app/admin（租户后台）、app/api（C端）、app/common（公共）。
- Laravel 13：路由分组 + 目录分层 + 多守卫（config/auth.php 配置 platform_admin / tenant_admin / user 三个 guard），Controllers 按 Platform/Admin/Api 分目录，routes 分 platform.php/admin.php/api.php。

**理由**：与分离账号体系契合，三端代码物理隔离，安全清晰。

---

## ADR-015：管理后台前端拆分为两个独立项目（方案 X）

**决策**：系统后台（平台）和租户后台（商户）做成两个独立前端项目。
- binxinadmin-admin-web：平台超管用，对接 platform 应用。
- binxinadmin-tenant-web：租户管理员用，对接 admin 应用。

**代码复用**：抽取共享包 binxinadmin-admin-shared（npm 私有包或 git submodule），含 Axios 封装、CRUD 组件、权限指令、布局、主题、工具函数、TS 类型。两个前端项目都依赖它，避免重复劳动。

**理由**：用户选择彻底隔离，与分离账号体系理念一致。通过共享包避免重复开发。

**端对应关系**：
| 前端 | 后端应用 | 账号表 | 守卫 | 登录入口 |
|---|---|---|---|---|
| admin-web | platform | bx_platform_admin | PlatformGuard | /platform/login |
| tenant-web | admin | bx_tenant_admin | TenantGuard | /admin/login |
| uniapp | api | bx_user | UserGuard | /api/login |

---

## ADR-016：C 端用户身份/凭证/资料三层分离

**决策**：C 端用户拆为三层：
- bx_user（身份层）：自然人主体，归属在租户层（移除 application_id）。
- bx_user_auth（凭证层）：一个 user 多条登录凭证，(application_id+open_id) 定位 user_id。
- bx_user_app_profile（资料层）：isolated 模式下按应用隔离会员数据。

**身份打通策略（策略 A）**：同租户下多应用通过 unionid/手机号打通为同一个人；跨租户不打通。打通必须限定在同一 owner（租户）范围内，严禁跨租户匹配。

**会员资料模式（可配置）**：租户级配置 member_data_mode：
- shared：会员等级/积分/余额存 bx_user，所有应用共享一份。
- isolated：存 bx_user_app_profile，按应用隔离。

**理由**：解决"一人在多应用 openid 不同导致一人多账号、用户表膨胀"的问题。身份按租户唯一，凭证表虽多但结构简单（千万级无压力）。可配置满足不同客户（连锁品牌共享 vs 代运营独立）需求。

**安全约束**：unionid/手机号打通严格限定本租户范围。

---

## ADR-017：两后端共用一个数据库

**决策**：开发学习阶段，TP 和 Laravel 共用同一个数据库 `binxinadmin`。
- ThinkPHP：负责跑迁移建表 + 种子数据（think-migration）。
- Laravel：直接连 `binxinadmin` 库读写，日常开发不跑迁移。
- Laravel 迁移文件依然完整编写并保留，作为：(1) 示范"Laravel 13 怎么写迁移"；(2) 用户单独部署 Laravel 版时可独立建表。

**理由**：满足"省时间(只建一次表)、数据一致(一份数据)、便于学习对照"的诉求。因 Laravel 不跑迁移，避免了两套迁移工具在同一库的 migrations 记录表冲突问题。

**代价**：日常开发时 Laravel 迁移文件不实际执行（仅备用）。可接受。

**注意**：两版表结构一致性靠 db-design-v1.md 契约保证，与库数量无关。
