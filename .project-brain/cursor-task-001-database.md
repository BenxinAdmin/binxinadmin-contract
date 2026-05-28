# Cursor 任务书 #001：数据库设计与迁移文件

> 用途：把本文件喂给 Cursor，让它在两个后端仓库分别生成数据库迁移文件和种子数据。
> 配套参考：db-design-v1.md（完整表结构）、03-CONVENTIONS.md（规范）。

---

## 角色设定（粘贴给 Cursor）

```
你是资深 PHP 全栈工程师，精通 ThinkPHP 8.1 和 Laravel 13，精通 MySQL 8.0 数据库设计与安全。
你严格遵循 PSR-12、参数绑定防注入、字段加密等安全规范。
你正在为 BinxinAdmin（多租户开源中后台框架）创建数据库层。
```

---

## 任务目标

为 BinxinAdmin 创建完整的数据库迁移文件与种子数据，需在两个后端仓库分别实现：
- ThinkPHP 8.1 版本：用 think-migration 扩展（数据库迁移）
- Laravel 13 版本：用 Laravel 原生 Migration + Seeder

两个版本的表结构必须 100% 一致（字段、类型、索引、注释完全相同）。

## ★ 数据库使用策略（重要，ADR-017）

- **共用一个数据库 `binxinadmin`**（不建第二个库）。
- **ThinkPHP 负责实际建表**：跑 think-migration 迁移 + 种子数据，建好全部 29 张表和初始化数据。
- **Laravel 直接连 `binxinadmin` 库读写，日常不跑迁移**（表已由 TP 建好，数据一份，两版一致）。
- **Laravel 迁移文件依然要完整编写并保留**，目的：(1) 示范 Laravel 13 迁移写法；(2) 供用户单独部署 Laravel 版时可独立建表。但日常开发不执行它。
- 两版表结构一致性靠本文档 + db-design-v1.md 契约保证。

---

## 全局规范（必须遵守）

1. 表前缀：`bx_`
2. 引擎：InnoDB，字符集：utf8mb4，排序规则：utf8mb4_unicode_ci
3. 主键：`id` BIGINT UNSIGNED AUTO_INCREMENT
4. 每张表必含通用字段：created_at, updated_at, deleted_at(NULL), created_by(默认0), updated_by(默认0)
5. 每张表、每个字段必须有 COMMENT 中文注释
6. 软删除：用 deleted_at，不物理删除
7. 所有关系字段、查询字段建索引
8. 字段命名 snake_case
9. 时间字段用 DATETIME 类型
10. 状态字段 status TINYINT，默认 1（1正常 0禁用）

---

## 需要创建的表（共 29 张，分模块）

### 模块A 租户体系
- bx_tenant（租户）
- bx_tenant_package（租户套餐）

### 模块B 平台账号 RBAC
- bx_platform_admin（平台管理员，username 全局唯一）
- bx_platform_role（平台角色）
- bx_platform_admin_role（管理员-角色关联，联合主键）
- bx_platform_role_menu（角色-菜单关联，联合主键）

### 模块C 租户账号 RBAC
- bx_tenant_admin（租户管理员，UNIQUE(tenant_id,username)）
- bx_tenant_role（租户角色，UNIQUE(tenant_id,code)）
- bx_tenant_admin_role（关联）
- bx_tenant_role_menu（关联）
- bx_dept（部门，租户级，树形 parent_id）
- bx_post（岗位，租户级）

### 模块D 菜单
- bx_menu（菜单，admin_type 区分 platform/tenant，树形）

### 模块E 应用与配置（核心）
- bx_application（应用，owner_type+owner_id 归属，app_type 类型，UNIQUE(app_key)）
- bx_app_config（应用第三方配置，敏感字段加密标记 is_encrypted）

### 模块F 系统管理
- bx_dict（字典）
- bx_dict_item（字典项）
- bx_config（参数配置）
- bx_third_config（第三方服务配置，owner 归属，config JSON）
- bx_oper_log（操作日志）
- bx_login_log（登录日志）
- bx_file（文件，owner 归属）

### 模块G 开发工具
- bx_gen_table（代码生成表配置）
- bx_gen_column（代码生成字段配置）
- bx_crontab（定时任务）

### 模块H C端用户（身份/凭证/资料三层分离）
- bx_user（用户主体/身份层，owner 归属在租户层，含 union_id/phone 打通钥匙，含 shared 模式会员字段，无 application_id）
- bx_user_auth（登录凭证层，UNIQUE(application_id,auth_type,open_id)，INDEX(user_id)）
- bx_user_app_profile（应用内档案层，isolated 模式用，UNIQUE(user_id,application_id)）

> 重要：用户身份按租户打通（策略A）。同租户多应用通过 unionid/手机号识别为同一人。
> bx_user 不带 application_id；application_id 在 auth 和 profile 表里。
> bx_tenant 表增加 member_data_mode 字段（shared/isolated，默认 shared）。

> 每张表的完整字段定义见 db-design-v1.md，严格按该文档实现。

---

## 关键安全与设计要求

1. **归属字段**：bx_application、bx_user、bx_third_config、bx_file 等表带 owner_type ENUM('platform','tenant') + owner_id BIGINT UNSIGNED DEFAULT 0。

2. **账号分离**：平台管理员和租户管理员是两张独立表，绝不合并。

3. **唯一索引**：
   - bx_platform_admin: UNIQUE(username)
   - bx_tenant_admin: UNIQUE(tenant_id, username)
   - bx_tenant: UNIQUE(tenant_code)
   - bx_application: UNIQUE(app_key)
   - bx_user_auth: UNIQUE(application_id, auth_type, open_id)

4. **敏感字段**：bx_app_config.config_value、bx_third_config.config 中的密钥类内容，迁移文件中用普通字段（TEXT/JSON），加密在应用层 Model 的访问器中实现（后续任务）。迁移中加 is_encrypted 标记字段。

5. **菜单表 admin_type**：用于区分平台菜单树和租户菜单树。

---

## 种子数据（Seeder）

创建以下初始化数据：
1. 平台超级管理员：username=superadmin，密码=随机强密码（输出到控制台，bcrypt 加密存储），is_super=1
2. 默认租户套餐：免费版（1应用/5管理员）、标准版、企业版
3. 平台菜单树：系统管理、租户管理、应用管理、开发工具、监控等（admin_type=platform）
4. 租户菜单树：用户管理、角色权限、内容管理等（admin_type=tenant）
5. 系统字典：性别、通用状态、是否、应用类型(app_type)、第三方服务商等
6. 系统参数默认值

---

## 交付物

### ThinkPHP 仓库（binxinadmin-backend-tp）
- database/migrations/ 下的迁移文件（按模块分文件，命名规范）
- database/seeds/ 下的种子文件
- 可执行：php think migrate:run && php think seed:run

### Laravel 仓库（binxinadmin-backend-laravel）
- database/migrations/ 下的迁移文件
- database/seeders/ 下的 Seeder
- 可执行：php artisan migrate && php artisan db:seed

### 验证
- ThinkPHP 跑迁移 + 种子后，binxinadmin 库中存在全部 29 张表 + 初始化数据。
- Laravel 配置连接同一个 binxinadmin 库，能正常读取 TP 建好的表（用一个简单查询验证，如读取平台超管账号）。
- Laravel 迁移文件编写完整、语法正确（可在空库环境单独跑通验证一次，但日常不在主库执行）。
- 提供一份 schema 说明，确认 TP 迁移与 Laravel 迁移定义的表结构一致。

---

## 注意事项
- 不要在迁移文件中写死任何密钥。
- 所有外键关系用索引实现，暂不加物理外键约束（多租户场景灵活性优先）。
- 迁移文件要可回滚（down 方法完整）。
- 字段注释用中文，清晰说明用途。
