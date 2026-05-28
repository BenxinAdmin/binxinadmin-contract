# Cursor 任务书 #003：Docker 开发环境

> 用途：搭建 BinxinAdmin 的 Docker 开发环境，精确锁定 PHP 8.3 + MySQL 8.0 + Redis 7 + Nginx。
> 放置位置：binxinadmin-contract 仓库的 docker/ 目录（两个后端共用）。
> 背景：开发者本地是 PHP 8.2 / MySQL 9 / Redis 8，与项目要求不符，故用 Docker 隔离统一环境。

---

## 角色设定（粘贴给 Cursor）

```
你是资深 DevOps + PHP 工程师，精通 Docker、Docker Compose、PHP 8.3 运行环境配置、Nginx 配置。
你正在为 BinxinAdmin 多租户后台搭建本地开发的 Docker 环境。
要求：环境版本精确、可一键启动、两个后端（ThinkPHP/Laravel）共用、适合开源用户开箱即用。
```

---

## 总目标

在 binxinadmin-contract/docker/ 目录下创建一套 Docker 开发环境，包含 4 个服务：PHP 8.3、MySQL 8.0、Redis 7、Nginx。两个后端项目（binxinadmin-backend-tp、binxinadmin-backend-laravel）都能挂载到这套环境运行。

---

## 目录结构

```
binxinadmin-contract/
└── docker/
    ├── docker-compose.yml          编排文件
    ├── .env.example                环境变量模板（端口、密码等）
    ├── php/
    │   ├── Dockerfile              PHP 8.3-fpm 自定义镜像
    │   └── php.ini                 PHP 配置
    ├── nginx/
    │   ├── conf.d/
    │   │   ├── tp.conf             ThinkPHP 站点配置
    │   │   └── laravel.conf        Laravel 站点配置
    │   └── nginx.conf              Nginx 主配置
    ├── mysql/
    │   └── my.cnf                  MySQL 配置（utf8mb4）
    ├── redis/
    │   └── redis.conf              Redis 配置
    └── README.md                   使用说明
```

---

## 各服务详细要求

### 1. PHP 服务（php/Dockerfile）
- 基础镜像：php:8.3-fpm（官方镜像，注意 ARM64/Apple Silicon 兼容，用多架构镜像）
- 安装扩展：
  - pdo_mysql（连 MySQL）
  - redis（连 Redis，用 pecl 安装）
  - bcmath（数学计算）
  - gd（图片处理）
  - opcache（性能）
  - zip、intl、exif、pcntl、mbstring
- 安装 Composer 2（从官方镜像 copy）
- 设置工作目录 /var/www
- php.ini 配置：上传大小限制、内存限制、时区 Asia/Shanghai、opcache 开启
- （可选）安装 swoole 扩展，供未来高性能场景，但默认 fpm 模式

### 2. MySQL 服务
- 镜像：mysql:8.0（明确 8.0，不要 latest，不要 9.x）
- 环境变量：root 密码、初始数据库 binxinadmin、字符集 utf8mb4
- my.cnf：
  - character-set-server=utf8mb4
  - collation-server=utf8mb4_unicode_ci
  - default-authentication-plugin=mysql_native_password（兼容性）
  - 时区设置
- 数据卷持久化（防止容器删除丢数据）
- 端口映射（默认 3306，可通过 .env 改，避免和本地 MySQL 9 冲突，建议映射到 3307）

### 3. Redis 服务
- 镜像：redis:7-alpine（明确 7，不要 8.x）
- redis.conf：设置密码（可选）、持久化（AOF）、最大内存策略
- 数据卷持久化
- 端口映射（默认 6379，建议映射到 6380 避免和本地 Redis 8 冲突）

### 4. Nginx 服务
- 镜像：nginx:stable-alpine
- 配置两个站点：
  - ThinkPHP：根目录 public，入口 index.php，支持多应用路由（platform/admin/api）
  - Laravel：根目录 public，入口 index.php，支持路由分组
- PHP-FPM 转发到 php 容器 9000 端口
- 端口映射：TP 站点（如 8080）、Laravel 站点（如 8081）
- gzip、静态资源缓存、安全头

---

## docker-compose.yml 要求

- 版本用 compose 规范（不写 version 字段，新版本已不需要）
- 4 个服务用同一个自定义网络互通
- php 容器挂载两个后端项目目录（通过相对路径或 .env 配置路径）
- 依赖关系：nginx depends_on php，php depends_on mysql/redis
- 所有端口、密码、路径通过 .env 配置，提供 .env.example
- 数据卷：mysql_data、redis_data 持久化

### 关键：端口避让
开发者本地已有 MySQL 9（3306）和 Redis 8（6379），Docker 里的服务端口要避开：
- MySQL 容器映射到宿主机 3307
- Redis 容器映射到宿主机 6380
- 容器内部仍用标准端口，仅宿主机映射端口避让

---

## 挂载后端项目的方式

php 和 nginx 容器需要访问两个后端代码。方案：
- 在 .env 里配置两个后端项目的本地路径（相对 docker-compose 或绝对路径）
- docker-compose volumes 挂载进容器
- 例如宿主机 ../../binxinadmin-backend-tp 挂载到容器 /var/www/tp
- ../../binxinadmin-backend-laravel 挂载到容器 /var/www/laravel

> 注意：开发者目录结构是 /Users/daxing/projects/xinzhi/ 下平级放着各仓库。
> docker 目录在 binxinadmin-contract/docker/，所以后端项目相对路径是 ../../binxinadmin-backend-tp 等。

---

## README.md 要求

写清楚：
1. 前置：装 Docker Desktop
2. 复制 .env.example 为 .env，按需改端口/密码
3. 启动：docker compose up -d
4. 进入 php 容器：docker compose exec php bash
5. 在容器内跑 composer install、迁移命令
6. 访问地址：TP http://localhost:8080，Laravel http://localhost:8081
7. 停止：docker compose down
8. 连接数据库：宿主机 127.0.0.1:3307，账号密码见 .env
9. 常见问题（端口冲突、权限、Apple Silicon 镜像）

---

## 交付物
- 完整的 docker/ 目录所有文件
- 可执行：在 docker/ 目录 docker compose up -d 后，4 个容器正常运行
- php 容器内 php -v 显示 8.3，mysql 显示 8.0，redis 显示 7

## 验证标准
1. docker compose up -d 后 4 个容器 healthy
2. docker compose exec php php -v → PHP 8.3.x
3. docker compose exec php php -m → 包含 pdo_mysql、redis、bcmath、gd、opcache
4. 宿主机 127.0.0.1:3307 能连上 MySQL 8.0
5. 宿主机 127.0.0.1:6380 能连上 Redis 7
6. 不影响宿主机本地的 PHP 8.2 / MySQL 9 / Redis 8

---

## 注意事项
- Apple Silicon（M 系列芯片）：所有镜像用支持 arm64 的版本，避免 platform 警告。
- 不要用 latest 标签，版本必须精确锁定。
- 密码等敏感信息放 .env，.env 不提交 git（.gitignore 已配 .env），只提交 .env.example。
- MySQL 数据卷持久化，避免重启丢数据。
