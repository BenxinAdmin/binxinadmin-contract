# Cursor 任务书 #002：M0 后端基础设施（双后端骨架）

> 用途：搭建 ThinkPHP 8.1 和 Laravel 13 两个后端的项目骨架与核心基座。
> 前置：任务书 #001（数据库）已完成。
> 配套：db-design-v1.md、03-CONVENTIONS.md。

---

## 角色设定（粘贴给 Cursor）

```
你是资深 PHP 全栈架构师，精通 ThinkPHP 8.1 与 Laravel 13，精通多应用/多守卫架构、多租户、JWT、Casbin RBAC、安全防护（防注入/越权/XSS）。
你正在为 BinxinAdmin 多租户开源中后台搭建后端基础设施。
代码遵循 PSR-12，所有查询防注入，所有归属数据防越权。
```

---

## 总目标

搭建两个后端的骨架，包含 6 大基座能力。两个版本对前端暴露的 API 行为必须 100% 一致。

1. 多应用/多端目录结构
2. 多租户 + 统一归属作用域（OwnerScope）
3. 统一响应 + 错误码 + 全局异常处理
4. JWT 认证基座 + 三守卫中间件
5. 配置加载 + 密钥加密服务
6. 基础中间件（CORS、请求ID、限流、操作日志）

---

## 一、目录结构

### ThinkPHP 8.1（binxinadmin-backend-tp）
```
app/
├── platform/          平台后台应用
│   ├── controller/
│   ├── middleware/
│   └── ...
├── admin/             租户后台应用
├── api/               C端应用
├── common/            公共层
│   ├── model/         所有 Model（共享）
│   ├── service/       业务服务层
│   ├── exception/     异常类
│   ├── response/      统一响应
│   ├── scope/         OwnerScope 归属作用域
│   ├── middleware/    公共中间件
│   ├── crypt/         加密服务
│   └── enum/          枚举（ErrorCode等）
config/
extend/
route/
├── platform.php
├── admin.php
└── api.php
```
启用 ThinkPHP 多应用模式（think-multi-app）。

### Laravel 13（binxinadmin-backend-laravel）
```
app/
├── Http/
│   ├── Controllers/
│   │   ├── Platform/
│   │   ├── Admin/
│   │   └── Api/
│   ├── Middleware/
│   └── Requests/      表单验证
├── Models/            所有 Model
├── Services/          业务服务层
├── Scopes/            OwnerScope 全局作用域
├── Support/
│   ├── Response/      统一响应
│   ├── Crypt/         加密服务
│   └── Enums/         ErrorCode 等
├── Exceptions/
routes/
├── platform.php
├── admin.php
└── api.php
config/auth.php        三个 guard
```

---

## 二、多租户 + 统一归属作用域（OwnerScope）★安全核心

这是整个系统的安全命脉，必须做扎实。

### 需求
1. 带 owner_type/owner_id 字段的表（bx_application、bx_user、bx_third_config、bx_file 等）：查询时自动注入归属过滤。
2. 带 tenant_id 的表（bx_tenant_admin、bx_tenant_role、bx_dept 等）：查询时自动注入租户过滤。
3. 当前请求的归属上下文从 JWT/登录态中获取，存入请求上下文（Context）。
4. 平台超管可"显式绕过"作用域做跨租户操作，但必须经过权限校验，且绕过方法要显式调用（不能默认绕过）。

### ThinkPHP 实现
- 用模型全局查询作用域（globalScope）或基类 Model 的 base 查询条件。
- 建 common/scope/OwnerScope.php 和 TenantScope.php。
- 建 BaseModel（Bx 前缀风格），归属表继承带 OwnerScope 的基类。
- 当前上下文用 think\Context 存储 current_owner_type/current_owner_id/current_tenant_id。

### Laravel 实现
- 用 Eloquent Global Scope。
- 建 app/Scopes/OwnerScope.php、TenantScope.php。
- 用 Trait（HasOwner、BelongsToTenant）让 Model 引入。
- 当前上下文用容器单例或中间件注入。

### 安全要求
- 写操作（create）自动填充当前 owner_type/owner_id/tenant_id。
- 严禁任何地方手写归属过滤，统一走作用域。
- 提供 withoutOwnerScope() 显式绕过方法，调用处必须有平台权限校验。

---

## 三、统一响应 + 错误码 + 异常处理

### 统一响应结构
```json
{"code":0,"message":"success","data":{},"timestamp":1716800000,"request_id":"req_xxx"}
```

### 响应类（两框架各实现）
- success(data, message)：code=0
- error(code, message, data)：业务错误
- paginate(list, page, pageSize, total)：分页结构 {list, pagination:{page,page_size,total,total_pages}}

### 错误码枚举（ErrorCode）
按 03-CONVENTIONS.md 的分段：
- 0 成功
- 10xxx 通用（10001参数错误 10002签名错误 10003限流 10004数据不存在）
- 20xxx 认证（20001未登录 20002Token无效 20003Token过期 20004权限不足 20005账号禁用 20006账号密码错误）
- 30xxx 用户
- 4xxxx 业务
- 5xxxx 系统（50000服务器错误 50001数据库 50002缓存 50003第三方 50004上传）
错误码 → 默认消息的映射表。

### 全局异常处理
- BusinessException（业务异常）→ 转统一响应，HTTP 200
- ValidateException（参数验证）→ code=10001，可带字段错误详情
- 认证异常 → HTTP 401，code=20001/20003
- HTTP 404 → code=10004
- 其他未捕获异常 → 生产环境 HTTP 500 + code=50000（隐藏堆栈），开发环境保留详情
- 所有异常记录日志（脱敏）

---

## 四、JWT 认证基座 + 三守卫

### JWT 要求
- 库：ThinkPHP 用 firebase/php-jwt；Laravel 用 tymon/jwt-auth 或 php-open-source-saver/jwt-auth（注意 Laravel 13 兼容性，选维护活跃的）。
- Token payload 含：sub(账号ID)、admin_type(platform/tenant/user)、tenant_id、owner_type、owner_id、exp、iat、jti。
- 密钥从环境变量读取，强随机。
- 支持 access_token + refresh_token 双 Token 机制。
- 登出：jti 加入 Redis 黑名单直至过期。

### 三个守卫中间件
- PlatformGuard：校验 admin_type=platform，注入平台上下文，仅 platform 路由用。
- TenantGuard：校验 admin_type=tenant，注入 tenant_id + owner 上下文，仅 admin 路由用。
- UserGuard：校验 admin_type=user，注入 owner 上下文，仅 api 路由用。
- 跨类型访问直接拒绝（平台 Token 不能访问租户接口）。

### 安全
- 密码 bcrypt 加密。
- 登录失败计数 + 锁定（Redis）。
- Token 校验失败统一返回 401。

---

## 五、配置加载 + 密钥加密服务

### 密钥加密服务（CryptService）
- AES-256-CBC 加解密，密钥从环境变量。
- 用于 bx_app_config、bx_third_config 的敏感字段。
- Model 访问器自动加解密（存入加密，读取解密，对外掩码）。

### 配置加载（带继承）
- 配置读取服务 ConfigService：
  - 应用级配置：读 bx_app_config（按 application_id）。
  - 服务配置（存储/短信）：先查租户级 bx_third_config(owner=tenant)，无则回退平台级(owner=platform)。
- 全部走 Redis 缓存，配置变更时清缓存。

---

## 六、基础中间件

- CORS：跨域处理（前端独立部署需要）。
- RequestId：每请求生成 request_id，注入响应头 X-Request-Id 和日志上下文。
- RateLimit：基于 Redis 的接口限流（防爆破/防刷），可配置。
- OperLog：操作日志中间件，记录到 bx_oper_log（写操作记录，脱敏）。
- 隐私保护：生产环境关闭 DEBUG，错误不暴露堆栈/路径/版本。

---

## 七、Docker 开发环境

提供 docker-compose.yml（放契约仓库或各后端仓库），一键启动：
- PHP 8.3-fpm（装好 swoole 可选、redis、pdo_mysql、bcmath、gd 等扩展）
- MySQL 8.0（utf8mb4，初始化 root 密码和数据库 binxinadmin）
- Redis 7
- Nginx（配置好 platform/admin/api 的路由转发，或单入口）
- 两个后端可分别用此环境

---

## 八、交付物与验证

### 交付物
- 两个后端的完整骨架代码（目录结构、基类、中间件、响应、异常、JWT、作用域、加密、配置服务）
- docker-compose.yml + Dockerfile + Nginx 配置
- 一个 health-check 接口（/platform/health、/admin/health、/api/health 返回统一响应）
- README 说明如何启动

### 验证标准
1. docker-compose up 能一键启动全套环境。
2. 三个 health 接口返回标准统一响应格式。
3. 用错误的 Token 类型访问，被守卫正确拒绝（如平台 Token 访问 /admin 接口返回 401）。
4. 归属作用域生效：模拟两个租户，查询自动隔离。
5. 两个后端的 health 接口和错误响应格式完全一致。

---

## 九、安全检查清单（交付前自检）
- [ ] 所有 DB 查询用 ORM/参数绑定，无 SQL 拼接
- [ ] 归属/租户作用域全局生效，无手写过滤遗漏
- [ ] JWT 密钥来自环境变量，非硬编码
- [ ] 密码 bcrypt，敏感配置 AES 加密
- [ ] 生产环境 DEBUG 关闭，错误不泄露堆栈
- [ ] 限流中间件就位
- [ ] CORS 白名单可配置，非全开
- [ ] 日志脱敏（不记录密码/密钥/Token）

---

## 十、注意事项
- 业务逻辑放 Service 层，Controller 只做参数校验+调用+响应。
- 两框架 Service 层逻辑保持高度一致，便于维护和代码生成器。
- 本任务只搭骨架和基座，不实现具体业务模块（登录、用户管理在 M1/M2）。
- 但要提供一个最简登录示例（平台超管登录）验证 JWT + 守卫链路通畅。
