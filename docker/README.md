# BinxinAdmin 本地开发 Docker 环境

> 用 Docker 隔离统一开发环境：PHP 8.3-fpm + MySQL 8.0 + Redis 7.4 + Nginx 1.27。
> 两个后端（`binxinadmin-backend-tp` / `binxinadmin-backend-laravel`）共用此环境。
> 适用于 Apple Silicon (arm64) 与 Intel/AMD (amd64) 双架构。

---

## 一、前置要求

- 安装 [Docker Desktop](https://www.docker.com/products/docker-desktop/)（macOS 推荐 4.30+）
- 至少为 Docker 分配 4 GB 内存、2 CPU
- 已克隆三个仓库到同一目录平级，目录结构：

  ```
  xinzhi/
    ├── binxinadmin-contract/        本仓库，docker 配置在这里
    │   └── docker/                  ← 你现在这个目录
    ├── binxinadmin-backend-tp/      ThinkPHP 8.1 后端
    └── binxinadmin-backend-laravel/ Laravel 13 后端
  ```

> 如果你的目录结构不同，编辑 `.env` 里的 `TP_PATH` / `LARAVEL_PATH`。

---

## 二、端口规划（已避让本地 MySQL 9 / Redis 8）

| 服务 | 容器内端口 | 宿主机端口（默认） | .env 变量 |
|---|---|---|---|
| Nginx → ThinkPHP | 80 | **8080** | `TP_PORT` |
| Nginx → Laravel | 81 | **8081** | `LARAVEL_PORT` |
| MySQL | 3306 | **3307** | `MYSQL_PORT` |
| Redis | 6379 | **6380** | `REDIS_PORT` |
| PHP-FPM | 9000 | 不映射 | — |

> 如果默认端口仍冲突，编辑 `.env` 任意改。

---

## 三、首次启动

```bash
cd binxinadmin-contract/docker

# 1) 复制环境变量模板
cp .env.example .env

# 2) （可选）编辑 .env 调整密码/端口

# 3) 构建并启动（首次会拉镜像 + 构建 PHP，约 5-10 分钟）
docker compose up -d --build

# 4) 等所有容器变成 healthy
docker compose ps
```

预期输出（4 个容器 STATUS 都是 `Up (healthy)` 或 `Up`）：

```
NAME             IMAGE                  STATUS                    PORTS
binxin_mysql     mysql:8.0              Up X minutes (healthy)    0.0.0.0:3307->3306/tcp
binxin_nginx     nginx:1.27-alpine      Up X minutes              0.0.0.0:8080->80/tcp, 0.0.0.0:8081->81/tcp
binxin_php       binxin/php:8.3-fpm     Up X minutes              9000/tcp
binxin_redis     redis:7.4-alpine       Up X minutes (healthy)    0.0.0.0:6380->6379/tcp
```

---

## 四、日常使用

### 4.1 进入 PHP 容器

```bash
docker compose exec php bash
# 容器内目录：
# /var/www/tp        → ThinkPHP 项目
# /var/www/laravel   → Laravel 项目
```

### 4.2 在容器内安装依赖与跑迁移

```bash
# 进入容器后
cd /var/www/tp
composer install
php think migrate:run
php think seed:run

# 切到 Laravel
cd /var/www/laravel
composer install
php artisan migrate
php artisan db:seed
```

> ⚠️ **重要**：两个后端默认共用同一个数据库 `binxinadmin`，且表前缀都是 `bx_`、表结构 100% 一致。
> 同一时间只跑其中一边的迁移即可；如果想两边并行开发，请在 `.env` 中再加一个库名，或者各跑各的库。

### 4.3 应用层连库连缓存（在 PHP 容器内）

容器内服务名当主机名：

```env
DB_HOST=mysql
DB_PORT=3306
DB_DATABASE=binxinadmin
DB_USERNAME=binxin
DB_PASSWORD=binxin_pass_2026   # 同 .env 的 MYSQL_PASSWORD

REDIS_HOST=redis
REDIS_PORT=6379
REDIS_PASSWORD=binxin_redis_2026  # 同 .env 的 REDIS_PASSWORD
```

### 4.4 从宿主机连库连缓存

用 IDE / Sequel Ace / DBeaver / Redis Insight 等工具：

```
MySQL: 127.0.0.1:3307  user=binxin / root  密码见 .env
Redis: 127.0.0.1:6380  无用户名，密码见 REDIS_PASSWORD
```

### 4.5 浏览器访问

- ThinkPHP：<http://localhost:8080>
- Laravel：<http://localhost:8081>

### 4.6 停止 / 重启 / 清理

```bash
docker compose stop          # 停止但保留容器和数据
docker compose start         # 重新启动
docker compose restart php   # 单独重启某服务
docker compose down          # 移除容器（数据卷保留）
docker compose down -v       # ⚠️ 移除容器并删除数据卷（清空 MySQL/Redis 数据）
```

---

## 五、验证环境正确

```bash
# PHP 版本
docker compose exec php php -v
# 期望：PHP 8.3.x

# PHP 扩展
docker compose exec php php -m | grep -Ei 'pdo_mysql|redis|bcmath|gd|opcache|zip|intl|exif|pcntl|mbstring|swoole|sockets'
# 期望：全部出现

# MySQL 版本与字符集
docker compose exec mysql mysql -uroot -p"$MYSQL_ROOT_PASSWORD" -e "SELECT VERSION(); SHOW VARIABLES LIKE 'character_set_server'; SHOW VARIABLES LIKE 'collation_server';"
# 期望：8.0.x / utf8mb4 / utf8mb4_unicode_ci

# Redis 版本
docker compose exec redis redis-cli -a "$REDIS_PASSWORD" --no-auth-warning INFO server | head -20
# 期望：redis_version:7.4.x

# Nginx 配置语法
docker compose exec nginx nginx -t

# 不影响宿主机
ps -ef | grep -E '(mysqld|redis-server|php-fpm)' | grep -v docker
# 应该看到的是宿主机原生的 MySQL 9 / Redis 8 / PHP 8.2，跟容器互不干扰
```

---

## 六、常见问题排障

### 6.1 启动报端口被占用

```
Error: bind: address already in use
```

→ 改 `.env` 里对应端口，例如 `TP_PORT=8888`，然后 `docker compose up -d`。

### 6.2 Apple Silicon 镜像 platform 警告

所有镜像都已选 arm64 多架构版本，应不会出现 `no matching manifest for linux/arm64/v8` 警告。
如果仍报警告，临时方案是给该服务加 `platform: linux/amd64`（性能会下降，仅救急）。

### 6.3 PHP 容器跑 composer install 慢

进容器后切阿里云镜像：

```bash
composer config -g repos.packagist composer https://mirrors.aliyun.com/composer/
```

### 6.4 MySQL 提示 `Authentication plugin 'caching_sha2_password' cannot be loaded`

我们已用 `--default-authentication-plugin=mysql_native_password`。若仍报错，
重建用户：

```sql
ALTER USER 'binxin'@'%' IDENTIFIED WITH mysql_native_password BY 'binxin_pass_2026';
FLUSH PRIVILEGES;
```

### 6.5 Redis 连接拒绝 / `NOAUTH Authentication required`

`.env` 默认开了密码，业务连接必须带密码。或者把 `REDIS_PASSWORD=` 改为空再 `docker compose up -d` 重启 redis。

### 6.6 文件权限问题（修改文件后容器内看不到）

macOS Docker Desktop 默认挂载是 `cached` 模式，正常应实时同步。
若遇到权限问题：在宿主机给项目目录加 `chmod -R u+w .` 即可，无需进容器。

### 6.7 完全推倒重来

```bash
docker compose down -v        # 删容器和数据卷
docker rmi binxin/php:8.3-fpm # 删自建镜像
docker compose up -d --build  # 重新构建启动
```

---

## 七、目录结构对照

```
docker/
├── docker-compose.yml          编排
├── .env.example                环境变量模板
├── .env                        本机配置（不提交 git）
├── README.md                   本文档
├── php/
│   ├── Dockerfile              PHP 8.3-fpm 自定义镜像
│   └── php.ini                 PHP 配置
├── nginx/
│   ├── nginx.conf              Nginx 主配
│   └── conf.d/
│       ├── tp.conf             ThinkPHP 站点
│       └── laravel.conf        Laravel 站点
├── mysql/
│   └── my.cnf                  MySQL 配置
└── redis/
    └── redis.conf              Redis 配置
```

---

## 八、安全提示

- `.env` 已加入 `.gitignore`，绝不能提交真实密码到仓库
- 开发环境默认密码仅供本地，**生产环境必须改成强随机密码**
- `display_errors = On` 仅本地开发；上线前关闭
- 默认 Redis 已开启密码，避免被局域网扫到
- 容器内 PHP-FPM 端口 9000 不映射到宿主机，仅容器互联

---

## 九、版本锁定（不要随便升）

| 组件 | 版本 | 说明 |
|---|---|---|
| PHP | 8.3-fpm-bookworm | Laravel 13 最低要求；与生产保持一致 |
| MySQL | 8.0 | 严格 8.0，不要 9.x（与契约一致） |
| Redis | 7.4-alpine | 严格 7.x |
| Nginx | 1.27-alpine | 当前稳定线 |
| Composer | 2.x | 从 `composer:2` 官方镜像 COPY |

如需升级，先改 `Dockerfile` / `docker-compose.yml`，更新本文档版本表，提 PR。
