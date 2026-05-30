# BinxinAdmin 技术规范总纲

> 所有模块开发必须遵守。Cursor 需求文档会反复引用本规范。

---

## 一、数据库规范

- 表前缀：`bx_`
- 主键：`id` BIGINT UNSIGNED 自增
- 多租户字段：`tenant_id` BIGINT UNSIGNED（平台级表为 0）
- 时间字段：`created_at`、`updated_at`、`deleted_at`（软删除，DATETIME）
- 审计字段：`created_by`、`updated_by`（BIGINT UNSIGNED，记录操作人 ID）
- 状态字段：`status` TINYINT（1=正常，0=禁用）
- 字符集：utf8mb4 / utf8mb4_unicode_ci
- 引擎：InnoDB
- 所有关系字段、查询字段必须建索引
- 字段命名：snake_case，下划线分隔
- 每张表、每个字段必须有 COMMENT 注释

## 二、API 规范

### 统一响应结构
```json
{
  "code": 0,
  "message": "success",
  "data": {},
  "timestamp": 1716800000,
  "request_id": "req_xxx"
}
```

### 业务码分段
| 区间 | 含义 |
|---|---|
| 0 | 成功 |
| 10xxx | 通用错误（参数、签名、限流、数据不存在） |
| 20xxx | 认证授权（未登录、Token、权限、账号状态） |
| 30xxx | 用户模块 |
| 4xxxx | 业务模块（按业务分区间，如 40xxx订单 41xxx课程） |
| 5xxxx | 系统错误（服务器、数据库、缓存、第三方、上传） |

### HTTP 状态码使用
- 200：正常响应（含业务错误）
- 401：未登录/Token 失效（前端拦截器统一跳登录）
- 403：权限不足（可选）
- 422：参数验证失败（可选）
- 429：限流
- 5xx：系统异常（供监控识别）

### 数据格式约定
- 字段命名 snake_case
- 时间格式 `YYYY-MM-DD HH:mm:ss` 或时间戳
- 布尔值用 true/false（非 0/1）
- 列表/数组永不返回 null（返回 []）
- 超长 ID（17 位以上）转字符串返回
- 分页结构：`{list: [], pagination: {page, page_size, total, total_pages}}`
- API 版本：`/api/v1/`

### 列表查询约定
- 过滤：`?status=active&role=admin`
- 排序：`?sort=created_at&order=desc` 或 `?sort=-created_at`
- 搜索：`?keyword=xxx`
- 分页：`?page=1&page_size=20`

---

## 三、安全规范（红线，每模块必须过检查）

> 本章按 OWASP Top 10 + 业界最佳实践组织，每条标注实施阶段。
> 标记 ✅=阶段0已做 / 🟡=待对应阶段 / 🔴=上线前必修。

### 3.1 注入与输出安全
- ✅ **SQL 注入**：全部参数绑定/ORM，禁止拼接 SQL
- 🟡 **XSS 防护（M1）**：
  - Vue 默认对 `{{}}` 和 `:attr` 自动转义（已生效）
  - v-html 必须过 DOMPurify（共享包提供 `<x-safe-html>` 组件 + `safeHtml()` 工具）
  - eslint vue/no-v-html 强制警告
  - 后端富文本字段过 HTMLPurifier 白名单
  - Nginx 加 CSP / HSTS / Permissions-Policy（在 4 个基础头之外）
  - 错误响应不回显用户原始输入
- 🟡 **命令注入**：所有 exec/shell_exec/system 调用必须参数白名单 + escapeshellarg
- 🟡 **SSRF 防护**：外发 URL 请求必须 IP 白名单 + 禁止 127/10/172.16/192.168 段 + 超时

### 3.2 身份认证与会话安全
- ✅ **JWT**：HS256 + 强随机密钥（256位）+ access(2h)/refresh(14d) + Redis 黑名单
- ✅ **密码哈希**：bcrypt cost=12，禁止明文/MD5/SHA1/无盐
- 🟡 **登录爆破防护（M1）**：失败 3 次触发图形验证码 + 5 次锁定 15 分钟（详见 m1-requirements-preview.md）
- 🟡 **设备指纹（M1）**：JWT 内嵌设备指纹哈希，token 偷取后异地使用拒绝
- 🟡 **Refresh Token 单设备（M1）**：同账号新登录使旧 refresh token 失效
- 🟡 **关键操作二次确认（M1）**：修改密码、删除业务数据、改权限等需当前密码验证

### 3.3 授权与越权防护
- ✅ **水平越权**：OwnerScope 自动归属过滤（数据层）+ 跨守卫 token 拒绝（接口层）
- ✅ **多租户隔离**：owner_type + owner_id 统一抽象 + 物理表隔离（platform_admin / tenant_admin）
- 🟡 **垂直越权（M1）**：Casbin RBAC 策略引擎，每接口绑定权限标识，前后端双校验
- ✅ **JWT 类型隔离**：admin_type claim + Guard 拒绝跨类型访问（实测通过）

### 3.4 数据保护
- ✅ **敏感配置加密**：AES-256-CBC + HMAC-SHA256（防解密 + 防篡改）
- ✅ **Model 访问器自动加解密**：开发无感知，密文存库
- ✅ **加密密钥环境隔离**：.env 读取，不入仓库
- 🔴 **生产/测试/开发密钥完全不同**（上线前强制轮换）
- 🟡 **PII 字段加密（M2 起）**：手机号、身份证等存库可考虑加密或哈希化检索
- 🟡 **数据脱敏展示（M3 起）**：列表展示手机号 138****1234 形式

### 3.5 凭据与密钥管理（D-030/033/052）
- ✅ **严禁明文密码进任何 git 跟踪文件**（含 .project-brain）
- ✅ **严禁打 stdout**（seeder/CLI 写本地文件，权限 0600）
- ✅ **严禁出现在交付报告、对话、邮件、截图**
- ✅ **grep 扫描验证 + git check-ignore 验证 + 网页最终确认**
- 🔴 **密码管理器统一保管**，永不在多人之间传输

### 3.6 传输与请求安全
- ✅ **接口限流**：Redis 固定窗口，登录/敏感接口严格限流
- ✅ **RequestId 链路追踪**：X-Request-Id 全链路串联
- ✅ **CORS 严格白名单**：不允许 *，按域名/路径精确匹配
- 🔴 **HTTPS 强制 + HSTS preload**（生产环境）
- 🟡 **C 端加密通信（阶段 3）**：AES-256-CBC + 时间戳 + nonce + 签名防重放
- 🟡 **CSRF**：JWT Bearer 天然防 Cookie CSRF；关键操作走 3.2 二次确认

### 3.7 安全头（Nginx）
- ✅ **已配**：X-Frame-Options / X-Content-Type-Options / X-XSS-Protection / Referrer-Policy
- 🟡 **M1 补**：
  - Content-Security-Policy（default-src 'self'，按实际加载源调整）
  - Strict-Transport-Security（HTTPS 上线后开启）
  - Permissions-Policy（禁用 geolocation/microphone/camera 等不需要的能力）

### 3.8 输入验证
- ✅ **参数白名单接收**：TP/Laravel Validate
- 🟡 **Controller 强制 Validate（M1）**：每个 Controller 入口必有 Validate，CI 检查
- 🟡 **JSON 嵌套深度限制**：防恶意构造超深 JSON DoS
- 🟡 **请求体大小限制**：Nginx client_max_body_size + PHP post_max_size

### 3.9 文件安全（M3 文件管理模块）
- 🟡 **MIME + 扩展名白名单**双重校验
- 🟡 **内容魔数校验**（防扩展名伪造）
- 🟡 **统一重命名**：UUID + 原扩展名，丢弃原文件名
- 🟡 **存储隔离**：上传目录禁止 PHP 执行（Nginx location 配置）
- 🟡 **CDN 独立域名**：与主站分离，防 cookie 泄露
- 🟡 **病毒扫描**（可选，ClamAV）

### 3.10 错误处理与信息泄露
- ✅ **统一错误响应**：业务码 + message，不暴露堆栈
- 🔴 **生产 APP_DEBUG=false 强制**（上线前 .env 验证）
- 🔴 **不暴露版本号**（Nginx server_tokens off / PHP expose_php = Off）
- 🟡 **error_log 路径权限收紧**（M5 监控时统一）

### 3.11 日志与审计
- ✅ **OperLog 中间件**：所有写操作自动落库（method/path/params/result/ip/UA/request_id）
- ✅ **password 字段自动脱敏** "***"
- 🟡 **XSS payload 脱敏（M1）**：参数中 `<script` 等 token 自动转 `[XSS_REDACTED]`
- 🟡 **登录日志（M1）**：bx_login_log 记录所有成功/失败登录（IP/UA/结果）
- 🟡 **安全事件审计（M5）**：权限变更、密码重置、敏感操作单独安全日志

### 3.12 配置与运行环境
- 🔴 **生产 .env 检查清单**（上线前必过）：
  - APP_DEBUG=false
  - JWT_SECRET / AES_SECRET 与开发完全不同
  - DB 账号非 root，仅授权业务库
  - Redis 密码强随机
- 🔴 **Docker 镜像最小化 + 非 root 用户运行 PHP-FPM**
- 🔴 **数据库账号权限最小化**：业务账号只能 SELECT/INSERT/UPDATE/DELETE 业务库
- 🟡 **文件系统权限**：.env / runtime / storage 权限收紧（0600/0700）

### 3.13 依赖与供应链
- ✅ **composer.lock / pnpm-lock.yaml 锁版本**
- 🟡 **CI 漏洞扫描（M5）**：
  - `composer audit` （需 Composer 2.4+）
  - `pnpm audit`
  - GitHub Dependabot 自动 PR
- 🟡 **License 兼容性检查**：禁用 GPL 传染性依赖
- 🟡 **镜像签名校验**（生产 Docker registry）

### 3.14 应急响应
- 🟡 **凭据泄露应急流程（M1 文档化）**：
  - 立即轮换：DB 密码、JWT_SECRET、AES_SECRET、admin 密码
  - 强制所有用户重新登录（清空 Redis JWT 黑名单 + 旧 token 全部作废）
  - 检查 OperLog 审计可疑操作
- 🟡 **安全 issue 上报流程（v1.0 前）**：security@... 邮箱 + SECURITY.md

### 3.15 渗透测试与扫描（v1.0 前）
- 🟡 第三方渗透测试报告
- 🟡 CodeQL / SAST 集成 CI
- 🟡 公开 bug bounty 之前先做内部安全演练

---

## 四、开源合规规范（红线）

- **字体**：仅开源字体（思源黑体/思源宋体、阿里巴巴普惠体、HarmonyOS Sans、站酷开源字体、Inter、Roboto）。禁用微软雅黑、苹方等商业字体。
- **图标**：仅开源图标库（Element Plus Icons、Iconify 开源集、Remix Icon、Lucide、Tabler Icons）。禁用 Font Awesome Pro、阿里图标库商业图标。
- **图片/插画**：仅开源或自制（unDraw、Storyset 免费授权、自绘 SVG）。禁用未授权商业图片。
- **第三方库**：检查 License 与 Apache 2.0 兼容，避免 GPL 传染性协议。
- 每仓库根目录放 `LICENSE`、`NOTICE`、`CREDITS.md`（标注所有第三方资源来源和授权）。

---

## 五、代码规范

### PHP
- 遵循 PSR-12，用 PHP-CS-Fixer 统一格式
- base 类 `Bx` 前缀
- Service 分层
- 模型 getter/setter 处理 JSON 字段
- 文件头注释：
```
@project   BinxinAdmin
@mission   <文件职责>
@author    仗键天涯(daxing)
@email     3442535897@qq.com
```
修改他人文件时追加 `@updated 日期 说明`

### 前端
- ESLint + Prettier
- 组件名 PascalCase，文件名 kebab-case
- TypeScript strict 模式
- 组件 <script setup lang="ts"> 风格

### Git
- 约定式提交（Conventional Commits）：feat/fix/docs/refactor/test/chore
- 分支：main（稳定）/ develop（开发）/ feature/*（功能）
- 每次提交信息清晰，可追溯
- **commit message 一律用中文描述**（前缀仍用英文 feat/fix/docs 等约定式关键字，冒号后的说明用中文）
- 示例：
  - ✅ `feat: 实现JWT三守卫与跨类型Token拒绝`
  - ✅ `fix: 修复OperLog字段缺失导致的越权日志丢失`
  - ✅ `docs: 新增前端差异化主题方案`
  - ❌ `feat: implement JWT guards`（不要用英文描述）

---

## 六、多租户开发规范

- 业务表必须带 `tenant_id`
- ORM 全局作用域自动注入 `tenant_id` 过滤（除平台级表）
- 平台级超管可跨租户，租户管理员锁定本租户
- 所有"创建"操作自动写入当前 `tenant_id`
- 所有"查询/更新/删除"自动加 `tenant_id` 条件
- 提供"绕过租户作用域"的显式方法供平台级操作使用（需权限校验）

---

## 七、开发环境与版本规范（统一锁定）

### 后端
| 项 | 版本 | 说明 |
|---|---|---|
| PHP | 8.3 | Laravel 13 最低要求，生态成熟 |
| ThinkPHP | 8.1.x | |
| Laravel | 13.x | |
| MySQL | 8.0 | utf8mb4_unicode_ci |
| Redis | 7.x | 缓存/队列/限流/Token黑名单 |
| Composer | 2.x | |

### 前端 / Node 环境
| 项 | 版本 | 说明 |
|---|---|---|
| Node.js | **24.x（Active LTS，代号 Krypton）** | 支持至 2028-05，新项目标准选择 |
| 包管理器 | **pnpm（首选）** 或 npm 11 | 多项目+共享包场景 pnpm 更优 |
| Vite | 5.x | |
| Vue | 3.4+ | |
| TypeScript | 5.x，strict 模式 | |
| Element Plus | 2.x | 管理后台 UI |
| Nuxt | 3.x | 官网 |
| uni-app | Vue3 + Vite 版 | C端 |
| wot-design-uni | 最新稳定版 | C端 UI |

### 版本管理建议
- Node 用 nvm 管理：`nvm install 24 && nvm alias default 24`
- 各前端项目 package.json 加 engines 字段约束 Node 版本：
  ```json
  "engines": { "node": ">=24.0.0", "pnpm": ">=9.0.0" }
  ```
- 各前端项目根目录放 `.nvmrc` 文件内容为 `24`，方便团队统一
- 不要用 Node 26（Current，未转 LTS，仅评估）；不要用 Node 20（已 EOL）

### 为什么用 pnpm
项目有多个前端仓库（admin-web / tenant-web / admin-shared / uniapp / website）和共享包，pnpm 的依赖管理、磁盘占用、安装速度、workspace 能力都优于 npm/yarn。若个人习惯 npm 也可用 npm 11。
