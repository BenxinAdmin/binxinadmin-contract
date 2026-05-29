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

- **SQL 注入**：全部参数绑定/ORM，禁止拼接 SQL
- **XSS**：输出转义，富文本用 HTMLPurifier 白名单过滤
- **CSRF**：API 用 Token，表单校验
- **越权防护**：每个接口校验数据归属（租户隔离 + 数据权限），防水平/垂直越权
- **JWT**：密钥强随机、合理过期、刷新机制、登出黑名单
- **密码**：bcrypt 或 argon2，禁止明文/MD5/SHA1
- **敏感配置加密**：支付密钥、短信密钥等存库时加密（AES-256）
- **接口限流**：Redis 限流防爆破防刷
- **C 端加密**：POST 请求 AES-256-CBC + 签名 + 时间戳防重放
- **文件上传**：类型白名单、大小限制、重命名、隔离目录、防 webshell
- **日志脱敏**：不记录密码、密钥、身份证等敏感信息
- **错误信息**：生产环境关闭 DEBUG，不暴露堆栈/版本/路径
- **依赖安全**：定期扫描 composer/npm 漏洞
- **权限最小化**：数据库账号、文件权限最小化
- **请求参数**：全部经过 Validate 验证，白名单接收

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
