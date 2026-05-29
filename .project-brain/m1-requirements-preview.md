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
