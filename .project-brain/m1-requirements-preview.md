# M1 阶段需求预录（认证授权）

> 本文件记录 M1 阶段（认证授权）要实现的需求规格，避免遗漏。
> 当前阶段（M0/task-002）只做简单登录验证链路，正式登录安全在 M1 实现。

---

## 一、渐进式登录安全策略（核心需求）

来源：Derek 提出（参考 BenXin/booksee 项目超管条件触发验证码的设计）。
设计原则：平时体验好（不强制验证码），被攻击时防得住（失败后触发验证码 + 锁定）。

### 平台超管登录（最高安全级别）
```
1. 账号密码登录（bcrypt 校验，已存 bx_platform_admin.password）
2. 登录失败计数（Redis 记录，维度：账号 + IP）
3. 失败达 3 次 → 要求图形验证码（验证码现生成存 Redis，校验后失效）
4. 失败达 5 次 → 账号锁定 15 分钟（Redis 记锁定状态）
5. 登录成功 → 清除失败计数，记录 last_login_at / last_login_ip
6. 全程写 bx_login_log（IP、设备、浏览器、OS、结果、失败原因）
7. （可选增强，M1后期或M2）异地登录提醒、二次验证
```

### 租户管理员登录
- 同上渐进式策略
- 失败阈值、锁定时长可由平台在后台配置（存 bx_config）
- 登录入口 /admin/login，需租户上下文（tenant_code 或域名定位租户）

### C 端用户登录
```
- 微信小程序：wx.login 一键登录（code 换 openid/unionid，无需密码）
- 公众号：网页授权登录
- 手机号 + 短信验证码：短信验证码登录
- 账号密码：可选
- 防护：短信发送限流（同号码60秒1条、单日上限）、登录失败限流
- 走 bx_user + bx_user_auth 身份打通流程（见 db-design-v1.md 第八节）
```

### 涉及的已有基础设施（task-001 已建）
- bx_login_log：登录日志表
- bx_platform_admin / bx_tenant_admin：账号表（含 last_login_at/ip 字段）
- Redis：失败计数、锁定状态、验证码、短信码存储
- 图形验证码：用开源库（如 think-captcha 或自实现），字体用开源字体

### 图形验证码注意
- 验证码字体必须用开源字体（合规红线）
- 验证码图片现生成，答案存 Redis 设过期（如 5 分钟）
- 校验后立即失效，防重放

---

## 二、M1 其他需求（认证授权完整功能）
- JWT access_token + refresh_token 双令牌（M0 已搭基座，M1 完善刷新/登出黑名单）
- Casbin RBAC 策略引擎接入（M0 未装，M1 装并接入 role_menu）
- 三端登录/登出/刷新令牌完整接口
- 修改密码、找回密码
- 当前登录用户信息接口（/userinfo）
- 权限标识下发（前端按钮级权限 v-permission 依赖）

---

## 三、待 M1 启动时细化
本文件是需求预录，正式做 M1 时会产出对应的 cursor 任务书并细化。

---

## 四、密码 / 凭据输出安全规范（从教训中得出）

**背景**：项目初期 superadmin 密码两次出现在不该在的地方（一次被 commit 进公开仓库，一次被完整复制到 AI 对话）。任何明文出现在终端/文档/对话的密码都应视为已知泄露。

### 立即可做（小改进，可在 task-004 后端补丁里做）
1. PlatformAdminSeed 不再把密码打印到 stdout，改为写入项目根的 `.local-credentials.txt`（.gitignore 已配置）
2. 输出引导文字："密码已写入 .local-credentials.txt，请立即查看并保存到密码管理器，然后 rm 此文件"
3. seeder 提供 `--force` 参数，允许覆盖已有 superadmin（解决"幂等性导致无法重置"问题）

### M1 时实现（管理命令）
1. `php think admin:reset-password <username>` 重置任意管理员密码
2. 同样写入本地文件，不打印到 stdout
3. Laravel 版同步实现 `php artisan admin:reset-password`

### 长期原则
- 任何 CLI 命令产生敏感信息（密码、密钥、Token）必须默认写文件，不打印
- 文件路径加入 .gitignore
- 命令输出仅提示路径，不显示内容
- 提供"读完即删"的引导

> 这是从真实事故中得到的教训，必须落地。

---

## 五、XSS 防护工程化（M1 同步推进）

阶段 0 的 XSS 规范只写在文档里没实施。M1 阶段必须把以下落地：

### 前端（admin-shared + 两个前端）
1. **装 DOMPurify**：所有 v-html 场景**必须**过 DOMPurify
   - 在 admin-shared 提供 `safeHtml(html)` 工具函数和 `<x-safe-html>` 组件
   - eslint-plugin-vue 加 `vue/no-v-html` 警告（强制使用 x-safe-html 替代）
2. **保留 Vue 默认转义**：所有 `{{}}` 和 `:attr` 不动，依赖 Vue 内置安全机制
3. **登录页等公开页面加 CSP meta**：限制 script-src/style-src/img-src 来源

### 后端（TP，Laravel 版同步）
1. **装 ezyang/htmlpurifier**：富文本字段（公告、用户简介等）入库前过白名单
2. **统一响应里富文本字段标注**：哪些是"已 purify 的安全 HTML"、哪些是"纯文本"
3. **OperLog 中间件扩展**：参数里检测到 `<script` 等可疑 token 自动脱敏成 `[XSS_REDACTED]`
4. **错误响应不回显原始用户输入**：错误信息走错误码 + 通用文案，不把"你输入的 XXX 不合法"原样回显

### Nginx 增强（已有基础安全头，需补 CSP）
当前已有：
```
add_header X-Frame-Options 'SAMEORIGIN' always;
add_header X-Content-Type-Options 'nosniff' always;
add_header X-XSS-Protection '1; mode=block' always;
add_header Referrer-Policy 'strict-origin-when-cross-origin' always;
```

M1 加：
```
add_header Content-Security-Policy "default-src 'self'; script-src 'self' 'unsafe-inline'; ..." always;
add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;  # HTTPS 上线后
add_header Permissions-Policy "geolocation=(), microphone=(), camera=()" always;
```
> 注：CSP 配置要根据实际加载源调整（CDN、字体源等），过严会破坏页面，过松失去防护。M1 实施时让 Cursor 在 report-only 模式跑一阵子收集违规再正式上线。

### 验证清单（M1 完工时验证）
1. v-html 全仓库扫描：除 `<x-safe-html>` 之外不允许有 v-html
2. 模拟 XSS payload 注入：`<script>alert(1)</script>` / `<img src=x onerror=alert(1)>` 等，
   后端入库前应被 purify、前端展示应不触发
3. CSP 违规上报：浏览器 Console 应该没有 CSP 违规
4. Nginx 响应头：用 curl -I 确认 7 个安全头全部存在

---

## 六、M1 阶段完整安全工程化清单（综合补充）

第五节只列了 XSS。我系统盘点了项目所有安全项，M1 阶段必须一次性把以下做完，避免后续散乱补救。

### 6.1 认证强化（M1 核心）
- 渐进式登录安全（验证码 + 锁定，本文件第一/二节已详）
- JWT 增强：claims 加 device_fingerprint 哈希（防 token 偷取异地使用）
- Refresh Token 单设备策略：新登录使旧 refresh 失效
- 关键操作二次确认：修改密码/删除/改权限 需当前密码

### 6.2 授权完整（M1 核心）
- Casbin RBAC 接入
- 接入 bx_platform_role_menu / bx_tenant_role_menu 表
- 接口权限标识强制绑定（按钮级 v-permission + 接口级注解）
- 垂直越权防护（普通用户不能访问管理员接口）

### 6.3 输入输出安全
- XSS 工程化（详见本文件第五节）
- CSRF：关键操作二次确认（6.1 已覆盖）
- 命令注入审计：grep 全仓库 exec/shell_exec/system，每处必须参数白名单
- JSON 解析深度限制：json_decode($x, true, 32) 限制嵌套

### 6.4 错误与日志
- 生产 DEBUG=false 强制配置项
- 错误响应不回显原始用户输入
- OperLog XSS payload 自动脱敏（`<script` 等 token → `[XSS_REDACTED]`）
- 登录日志 bx_login_log 完整记录（IP/UA/结果/失败原因）
- 安全事件审计独立表（权限变更/密码重置/敏感操作）

### 6.5 安全头与传输
- Nginx CSP 头（default-src 'self'，先 report-only 跑收数据）
- HSTS 头（生产 HTTPS 上线时启用）
- Permissions-Policy 头（禁用不需要的浏览器能力）

### 6.6 应急流程文档化
- 凭据泄露应急 SOP（参考 D-030/031/051/052 已有的处理经验）
- 安全事件上报模板
- SECURITY.md 加入仓库（v1.0 准备）

### 6.7 工具与自动化
- 装 `ezyang/htmlpurifier`（后端）+ `dompurify`（前端 admin-shared）
- 装 `firebase/php-jwt` ≥ v7（已装）
- eslint-plugin-vue 启用 `vue/no-v-html`
- composer.json / package.json 加 audit script
- CI 集成 composer audit + pnpm audit（push 触发）

### 6.8 M1 安全验证清单
- 自动化：模拟 XSS / SQL注入 / 命令注入 / 越权 payload 全跑过
- 手工：登录爆破触发验证码、锁定、解锁全流程
- curl 跨守卫越权 + 横向越权 + 纵向越权 各 3 种 case
- 浏览器：v-html 渲染恶意 HTML 不触发
- Nginx 响应头：curl -I 7 个安全头齐全
