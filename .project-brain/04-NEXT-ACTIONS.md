# BinxinAdmin 下一步行动清单

> 滚动更新。完成一项划掉，新增下一项。

---

## 阶段 0：地基（当前）

### 已完成
- [x] 项目大脑全套文档
- [x] 数据库总体设计 v1（29 张表，db-design-v1.md）
- [x] Cursor 任务书 #001（数据库迁移）
- [x] Cursor 任务书 #002（M0 后端基础设施）
- [x] 本地目录创建 + Git 远程关联（Gitee + GitHub 双开）

### 进行中 / 待办
- [x] Cursor 执行 #001：TP 生成数据库迁移 + 种子数据（28张表 + 迁移#9 NOT NULL补丁）
- [x] Cursor 执行 #002：TP 后端骨架（多应用 + OwnerScope + JWT三守卫 + 统一响应 + 加密 + 迁移#10 OperLog补字段）
- [x] 验证：三个 health 接口 + cross-token 拒绝 + OperLog 新字段 全部通过
- [x] Docker 环境跑通（PHP8.3 + MySQL8.0 + Redis7 + Nginx）
- [ ] 三选一：task-002 Laravel版 / 前端骨架(admin-shared+admin-web+tenant-web) / M1 认证授权
- [ ] superadmin 密码已由 PlatformAdminSeed 随机生成并存入本地密码管理器（绝不入仓库）

### 本地准备（Derek 待办）
- [x] 创建 9 个仓库 + 关联远程
- [ ] 把 .project-brain/ 放入 binxinadmin-contract 仓库
- [ ] 本地装 Node 24（nvm）+ pnpm
- [ ] 本地装 PHP 8.3 + Composer 2 + MySQL 8.0 + Redis 7（或直接用 Docker）

---

## 后续阶段预告
- 阶段 1：M1 认证授权（需求已预录 m1-requirements-preview.md：渐进式登录安全+验证码+Casbin）+ M2 用户管理 + 前端对应页面
- 阶段 2：M3 系统管理 + M6 配置中心
- 阶段 3：M4 代码生成器 ★核心
- 阶段 4：M5 监控 + 插件机制 + P1 内容插件
- 阶段 5：uni-app C 端 + Nuxt 官网 + 文档 + 开源发布 v1.0
