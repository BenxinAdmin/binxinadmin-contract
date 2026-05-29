# BinxinAdmin 开发者本地环境信息

> 本文档记录 Derek（项目负责人）的本地开发环境路径与配置信息。
> 用于 Cursor 任务书引用、新对话上下文同步、问题排查时定位。
> 路径或配置变更时立即更新此文件。

---

## 一、本地项目根目录

```
/Users/daxing/projects/BenxinAdmin/
```

> 历史变更：早期路径为 `/Users/daxing/projects/xinzhi/`，2026-05 已更名为 BenxinAdmin。

## 二、9 个仓库的本地路径

```
/Users/daxing/projects/BenxinAdmin/
├── binxinadmin-contract/          契约仓库（含 .project-brain/、docker/）
├── binxinadmin-backend-tp/        ThinkPHP 8.1 后端
├── binxinadmin-backend-laravel/   Laravel 13 后端（待初始化）
├── binxinadmin-admin-web/         平台后台前端
├── binxinadmin-tenant-web/        租户后台前端
├── binxinadmin-admin-shared/      前端共享包
├── binxinadmin-uniapp/            C端（待初始化）
├── binxinadmin-website/           Nuxt官网（待初始化）
└── binxinadmin-docs/              VitePress文档站（待初始化）
```

## 三、Docker 环境关键路径

```
docker-compose 位置: /Users/daxing/projects/BenxinAdmin/binxinadmin-contract/docker/
容器内挂载映射:
  ../../binxinadmin-backend-tp      → /var/www/tp
  ../../binxinadmin-backend-laravel → /var/www/laravel
```

## 四、宿主机端口映射

| 服务 | 容器内 | 宿主机 | 说明 |
|---|---|---|---|
| Nginx (TP 站点) | 80 | 8080 | http://localhost:8080 |
| Nginx (Laravel 站点) | 80 | 8081 | http://localhost:8081 |
| MySQL | 3306 | **3307** | 避让本地 MySQL 9 |
| Redis | 6379 | **6380** | 避让本地 Redis 8 |

## 五、宿主机已占用版本（被 Docker 隔离）

| 组件 | 本地版本 | Docker 内版本 |
|---|---|---|
| PHP | 8.2.30 | 8.3.x |
| MySQL | 9.6.0 | 8.0.x |
| Redis | 8.6.2 | 7.x |
| Node | （用 nvm 装 24） | - |

## 六、Git 远程

| 远程名 | 平台 | 组织名 |
|---|---|---|
| origin | Gitee | binxin-admin |
| github | GitHub | **BenxinAdmin**（注意大写、无横线） |

仓库地址格式：
- Gitee：`git@gitee.com:binxin-admin/{repo}.git`
- GitHub：`git@github.com:BenxinAdmin/{repo}.git`

## 七、关键账号（本地保存，不在此文档明文）

| 账号 | 说明 |
|---|---|
| superadmin | TP 后端平台超管。**密码由 PlatformAdminSeed 随机生成并打印到控制台，Derek 已存入本地密码管理器**。绝不写入任何入仓库的文件（包括本文档）。 |

> ⚠️ 任何入仓库的文件都不得包含明文密码、密钥、Token。
> 若需重置超管密码：进 PHP 容器执行 `php think seed:run -s PlatformAdminSeed`，新密码会打印到控制台。
