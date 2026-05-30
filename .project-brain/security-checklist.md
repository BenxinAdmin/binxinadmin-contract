# BinxinAdmin 上线前安全检查清单

> 项目从开发到生产部署前必须逐项过的安全检查表。
> 任何一项未过都不能上线。
> 维护责任：项目经理（Claude） + 部署人（Derek）

---

## 第一类：配置硬化（🔴 必须项）

### 1.1 .env 配置
- [ ] APP_DEBUG=false
- [ ] APP_ENV=production
- [ ] JWT_SECRET 是生产专用强随机值（256位 hex），与开发/测试完全不同
- [ ] AES_SECRET / AES_HMAC_KEY 是生产专用强随机值，与开发/测试完全不同
- [ ] DB 账号是业务专用账号（非 root），仅授权 binxinadmin 库的增删改查
- [ ] DB 密码强随机
- [ ] Redis 密码强随机 + bind 限制（不裸跑公网）
- [ ] 所有密钥仅存密码管理器，绝不入仓库

### 1.2 数据库账号权限
```sql
-- 业务连接账号（生产）
CREATE USER 'binxin_app'@'%' IDENTIFIED BY '<强随机密码>';
GRANT SELECT, INSERT, UPDATE, DELETE ON binxinadmin.* TO 'binxin_app'@'%';
-- 不要给 GRANT/CREATE/DROP/ALTER 权限给业务账号
FLUSH PRIVILEGES;
```
- [ ] 业务连接账号无 DROP/ALTER/CREATE 权限
- [ ] root 账号仅本机访问（bind-address）

### 1.3 Nginx / Web 服务器
- [ ] HTTPS 强制（HTTP 301 跳 HTTPS）
- [ ] TLS 1.2+ only（禁用 SSL3/TLS1.0/1.1）
- [ ] HSTS 头：`max-age=63072000; includeSubDomains; preload`
- [ ] CSP 头配置完整（report-only 跑一周再正式上线）
- [ ] server_tokens off（不暴露 Nginx 版本）
- [ ] client_max_body_size 合理限制（默认 10M，按业务调）
- [ ] 静态上传目录禁止 PHP 执行（location ~ \.(php)$ { deny all; }）

### 1.4 PHP
- [ ] expose_php = Off
- [ ] display_errors = Off
- [ ] log_errors = On + error_log 路径设好
- [ ] open_basedir 限制（仅项目目录 + tmp）
- [ ] disable_functions = exec,passthru,shell_exec,system,proc_open,popen 等

### 1.5 Docker
- [ ] 生产镜像基于 alpine 或 slim
- [ ] PHP-FPM 用非 root 用户运行（USER www-data）
- [ ] 容器只读 root 文件系统（read_only: true，仅 storage/runtime 挂载读写）
- [ ] 不暴露不必要端口（MySQL/Redis 不对公网开放）

---

## 第二类：凭据轮换（🔴 必须项）

### 2.1 开发期间使用过的凭据全部失效
- [ ] superadmin 密码生产环境用新随机密码（开发期间 superadmin 密码已泄露过 3 次，必须轮换）
- [ ] tenant_admin@demo 删除或改密（演示账号不上生产）
- [ ] JWT_SECRET 生产值与开发值完全不同
- [ ] AES_SECRET 生产值与开发值完全不同
- [ ] DB 密码生产值与开发值完全不同
- [ ] Redis 密码生产值与开发值完全不同

### 2.2 凭据存放
- [ ] 所有生产凭据存企业密码管理器（1Password Business / Bitwarden 等）
- [ ] 至少 2 人有访问权限（防单点）
- [ ] 不通过聊天/邮件/文档传输

---

## 第三类：代码安全扫描（🟡 上线前必跑）

### 3.1 仓库扫描（防凭据残留）
```bash
# 在每个仓库根目录跑
git log --all -p | grep -iE "password|secret|api_key|token" > leak_check.txt
# 人工审核 leak_check.txt 是否有真实凭据残留
```
- [ ] 4 个仓库（contract / backend-tp / admin-shared / admin-web / tenant-web）逐个扫
- [ ] 历史 commit 也扫（不止 HEAD）
- [ ] 如果发现真实凭据：先轮换 → 再考虑 git filter-repo 重写历史

### 3.2 依赖漏洞扫描
- [ ] `composer audit`（TP + Laravel）
- [ ] `pnpm audit`（所有前端项目）
- [ ] GitHub Dependabot 启用
- [ ] 任何 high/critical 级漏洞必须修

### 3.3 静态代码扫描
- [ ] PHPStan / Psalm 跑后端
- [ ] CodeQL / SonarQube 跑前端
- [ ] 重点检查：v-html 使用、innerHTML、eval、动态 require/include、shell 调用

---

## 第四类：渗透测试（🟡 v1.0 上线前）

- [ ] OWASP ZAP / Burp Suite 自动扫描
- [ ] 手动测试 OWASP Top 10 全部条目
- [ ] 多租户隔离专项测试（A 租户 token 访问 B 租户数据必须 403）
- [ ] 横向越权专项测试
- [ ] 纵向越权专项测试（普通用户访问管理员接口）
- [ ] 暴力破解测试（验证码 + 锁定机制是否生效）
- [ ] 文件上传专项测试（webshell / 大文件 / 类型伪造）
- [ ] XSS / CSRF / SQL 注入逐项 payload 测试

---

## 第五类：运维监控（🟡 上线时启动）

- [ ] 错误日志告警（5xx 错误 / 异常堆栈）
- [ ] 登录失败异常告警（短时间大量失败）
- [ ] 数据库慢查询日志
- [ ] 文件系统空间监控
- [ ] 备份自动化（数据库 + 上传文件 + .env）+ 备份加密
- [ ] 灾难恢复演练（至少 1 次）

---

## 第六类：应急预案（🟡 上线前文档化）

- [ ] 凭据泄露应急流程（详见 03-CONVENTIONS.md 第 3.14 节）
- [ ] 数据泄露应急流程（依据 GDPR / 个保法 通报义务）
- [ ] DDoS 应对（云厂商 WAF / CDN 切换）
- [ ] 服务回滚流程（前后端独立回滚）
- [ ] 安全 issue 上报渠道（SECURITY.md + 邮箱）

---

## 使用方式

1. 项目接近 v1.0 时打开本清单
2. 逐项检查并打勾
3. 任何 🔴 项未过不能上线
4. 🟡 项可在上线后 30 天内补完
5. 检查结果由部署人 + 项目经理共同签字（截图存档）
