# BinxinAdmin 数据库总体设计（v1.0）

> 本文档是数据库层的真相源，存于 binxinadmin-contract 仓库。
> 两个后端（ThinkPHP / Laravel）均以此为准生成迁移文件。
> 表前缀统一 `bx_`，字符集 utf8mb4_unicode_ci，引擎 InnoDB。

---

## 一、核心设计理念

### 1.1 三层主体
- **平台层 Platform**：超级管理员，管理整个系统。
- **租户层 Tenant**：企业/组织，平台之下的隔离单元。
- **用户层 User**：C 端最终用户（小程序/H5/App 用户）。

### 1.2 两大账号体系（分离表）
- `bx_platform_admin`：平台管理员（不带 owner，全局唯一）。
- `bx_tenant_admin`：租户管理员（带 tenant_id，租户内唯一）。
- 两套独立 RBAC，两个登录入口（/platform/login、/admin/login）。

### 1.3 统一归属抽象（核心）
所有"可归属"实体带两个字段：
- `owner_type` ENUM('platform','tenant')：归属类型。
- `owner_id` BIGINT UNSIGNED：归属 ID（platform 时为 0，tenant 时为租户 ID）。

适用于：应用(bx_application)、C 端用户(bx_user)、以及所有业务数据表。
全系统用统一的 OwnerScope 作用域做归属过滤和越权校验，只写一套，严禁各处手写。

### 1.4 应用(Application)实体
- 一个 owner 下可有多个应用。
- `app_type`：wechat_mini / wechat_oa / h5 / app。
- 每个应用独立的 appid、第三方配置。

---

## 二、通用字段规范（每张表都有）

| 字段 | 类型 | 说明 |
|---|---|---|
| id | BIGINT UNSIGNED AUTO_INCREMENT | 主键 |
| created_at | DATETIME | 创建时间 |
| updated_at | DATETIME | 更新时间 |
| deleted_at | DATETIME NULL | 软删除时间 |
| created_by | BIGINT UNSIGNED DEFAULT 0 | 创建人ID |
| updated_by | BIGINT UNSIGNED DEFAULT 0 | 更新人ID |

业务表额外带：
| 字段 | 类型 | 说明 |
|---|---|---|
| owner_type | ENUM('platform','tenant') | 归属类型 |
| owner_id | BIGINT UNSIGNED DEFAULT 0 | 归属ID |
| status | TINYINT DEFAULT 1 | 状态 1正常 0禁用 |

---

## 三、ER 关系总览

```
bx_tenant (租户)
  │ 1:N
  ├──< bx_tenant_admin (租户管理员)
  │       │ N:N (bx_tenant_admin_role)
  │       └──< bx_tenant_role >── bx_tenant_permission
  │
  ├──< bx_application (应用, owner_type=tenant)
  │       │ 1:1
  │       └── bx_app_config (应用第三方配置)
  │
  └──< bx_user (C端用户, owner_type=tenant)

bx_platform_admin (平台管理员)
  │ N:N (bx_platform_admin_role)
  └──< bx_platform_role >── bx_platform_permission

bx_application (应用, owner_type=platform) ── 平台直营应用
bx_user (C端用户, owner_type=platform) ── 平台直营用户

公共表（不分归属或平台级）：
bx_menu, bx_dict, bx_dict_item, bx_config, bx_oper_log,
bx_login_log, bx_file, bx_third_config, bx_gen_table, bx_gen_column,
bx_crontab, bx_tenant_package
```

---

## 四、表结构详细设计

### 模块 A：租户体系

#### bx_tenant 租户表
| 字段 | 类型 | 说明 |
|---|---|---|
| id | BIGINT UNSIGNED | 主键 |
| tenant_code | VARCHAR(32) | 租户编码（唯一，用于登录定位） |
| name | VARCHAR(100) | 租户名称 |
| logo | VARCHAR(255) | Logo |
| contact_name | VARCHAR(50) | 联系人 |
| contact_phone | VARCHAR(20) | 联系电话 |
| package_id | BIGINT UNSIGNED | 套餐ID |
| expire_at | DATETIME NULL | 到期时间 |
| max_admin_count | INT DEFAULT 5 | 最大管理员数（配额） |
| max_app_count | INT DEFAULT 1 | 最大应用数（配额） |
| max_user_count | INT DEFAULT 0 | 最大C端用户数（0不限） |
| member_data_mode | VARCHAR(10) DEFAULT 'shared' | 会员数据模式 shared共享/isolated独立 |
| status | TINYINT | 状态 1正常 0禁用 |
| remark | VARCHAR(255) | 备注 |
| 通用字段 | | created_at等 |

索引：UNIQUE(tenant_code)、INDEX(package_id)、INDEX(status)

#### bx_tenant_package 租户套餐表
| 字段 | 类型 | 说明 |
|---|---|---|
| id | BIGINT UNSIGNED | 主键 |
| name | VARCHAR(50) | 套餐名 |
| max_admin_count | INT | 管理员配额 |
| max_app_count | INT | 应用配额 |
| max_user_count | INT | C端用户配额 |
| menu_ids | JSON | 套餐可用菜单ID集合 |
| price | DECIMAL(10,2) | 价格 |
| duration_days | INT | 有效天数 |
| status | TINYINT | 状态 |
| sort | INT DEFAULT 0 | 排序 |
| remark | VARCHAR(255) | 备注 |
| 通用字段 | | |

### 模块 B：平台账号体系（RBAC 一套）

#### bx_platform_admin 平台管理员表
| 字段 | 类型 | 说明 |
|---|---|---|
| id | BIGINT UNSIGNED | 主键 |
| username | VARCHAR(50) | 用户名（全局唯一） |
| password | VARCHAR(255) | 密码（bcrypt） |
| nickname | VARCHAR(50) | 昵称 |
| avatar | VARCHAR(255) | 头像 |
| phone | VARCHAR(20) | 手机 |
| email | VARCHAR(100) | 邮箱 |
| is_super | TINYINT DEFAULT 0 | 是否超级管理员（拥有全部权限） |
| last_login_at | DATETIME NULL | 最后登录时间 |
| last_login_ip | VARCHAR(45) | 最后登录IP |
| status | TINYINT | 状态 |
| 通用字段 | | |

索引：UNIQUE(username)、INDEX(status)

#### bx_platform_role 平台角色表
| 字段 | 类型 | 说明 |
|---|---|---|
| id | BIGINT UNSIGNED | 主键 |
| name | VARCHAR(50) | 角色名 |
| code | VARCHAR(50) | 角色编码（唯一） |
| data_scope | TINYINT DEFAULT 1 | 数据范围 1全部 2自定义 3本人 |
| sort | INT | 排序 |
| status | TINYINT | 状态 |
| remark | VARCHAR(255) | 备注 |
| 通用字段 | | |

#### bx_platform_admin_role 平台管理员-角色关联
| admin_id | BIGINT UNSIGNED |
| role_id | BIGINT UNSIGNED |
联合主键(admin_id, role_id)

#### bx_platform_role_menu 平台角色-菜单(权限)关联
| role_id | BIGINT UNSIGNED |
| menu_id | BIGINT UNSIGNED |
联合主键(role_id, menu_id)
（平台权限通过菜单+按钮承载，配合 Casbin 做策略）

### 模块 C：租户账号体系（RBAC 另一套，带 tenant_id）

#### bx_tenant_admin 租户管理员表
| 字段 | 类型 | 说明 |
|---|---|---|
| id | BIGINT UNSIGNED | 主键 |
| tenant_id | BIGINT UNSIGNED | 所属租户 |
| username | VARCHAR(50) | 用户名（租户内唯一） |
| password | VARCHAR(255) | 密码 |
| nickname | VARCHAR(50) | 昵称 |
| avatar | VARCHAR(255) | 头像 |
| phone | VARCHAR(20) | 手机 |
| email | VARCHAR(100) | 邮箱 |
| dept_id | BIGINT UNSIGNED DEFAULT 0 | 部门 |
| post_id | BIGINT UNSIGNED DEFAULT 0 | 岗位 |
| is_tenant_owner | TINYINT DEFAULT 0 | 是否租户主账号 |
| last_login_at | DATETIME NULL | |
| last_login_ip | VARCHAR(45) | |
| status | TINYINT | |
| 通用字段 | | |

索引：UNIQUE(tenant_id, username)、INDEX(tenant_id)、INDEX(dept_id)

#### bx_tenant_role 租户角色表
（结构同 bx_platform_role，额外带 tenant_id）
索引：UNIQUE(tenant_id, code)、INDEX(tenant_id)

#### bx_tenant_admin_role 租户管理员-角色关联
| tenant_id | admin_id | role_id |

#### bx_tenant_role_menu 租户角色-菜单关联
| tenant_id | role_id | menu_id |

#### bx_dept 部门表（租户级）
| 字段 | 类型 | 说明 |
|---|---|---|
| id | BIGINT UNSIGNED | 主键 |
| tenant_id | BIGINT UNSIGNED | 租户 |
| parent_id | BIGINT UNSIGNED DEFAULT 0 | 父部门 |
| name | VARCHAR(50) | 部门名 |
| sort | INT | 排序 |
| leader | VARCHAR(50) | 负责人 |
| phone | VARCHAR(20) | 电话 |
| status | TINYINT | |
| 通用字段 | | |

#### bx_post 岗位表（租户级）
| id | tenant_id | name | code | sort | status | 通用字段 |

### 模块 D：菜单与权限（平台与租户共用菜单定义表，用 admin_type 区分）

#### bx_menu 菜单表
| 字段 | 类型 | 说明 |
|---|---|---|
| id | BIGINT UNSIGNED | 主键 |
| admin_type | ENUM('platform','tenant') | 适用端：平台菜单还是租户菜单 |
| parent_id | BIGINT UNSIGNED DEFAULT 0 | 父菜单 |
| name | VARCHAR(50) | 菜单名 |
| type | TINYINT | 类型 1目录 2菜单 3按钮 |
| path | VARCHAR(200) | 路由路径 |
| component | VARCHAR(200) | 前端组件路径 |
| permission | VARCHAR(100) | 权限标识 如 user:add |
| icon | VARCHAR(100) | 图标（开源图标名） |
| sort | INT | 排序 |
| is_visible | TINYINT DEFAULT 1 | 是否显示 |
| status | TINYINT | |
| 通用字段 | | |

索引：INDEX(admin_type, parent_id)、INDEX(permission)

### 模块 E：应用与第三方配置（核心：多应用支持）

#### bx_application 应用表
| 字段 | 类型 | 说明 |
|---|---|---|
| id | BIGINT UNSIGNED | 主键 |
| owner_type | ENUM('platform','tenant') | 归属类型 |
| owner_id | BIGINT UNSIGNED DEFAULT 0 | 归属ID |
| app_type | ENUM('wechat_mini','wechat_oa','h5','app') | 应用类型 |
| name | VARCHAR(100) | 应用名称 |
| app_key | VARCHAR(64) | 应用唯一标识（系统内部，UUID） |
| logo | VARCHAR(255) | |
| description | VARCHAR(255) | |
| status | TINYINT | |
| 通用字段 | | |

索引：UNIQUE(app_key)、INDEX(owner_type, owner_id)、INDEX(app_type)

#### bx_app_config 应用第三方配置表
> 敏感字段加密存储（AES-256），平台填写/审核
| 字段 | 类型 | 说明 |
|---|---|---|
| id | BIGINT UNSIGNED | 主键 |
| application_id | BIGINT UNSIGNED | 所属应用 |
| config_type | VARCHAR(30) | 配置类型 wechat/pay_wechat/pay_alipay等 |
| config_key | VARCHAR(50) | 配置项键 appid/secret/mch_id等 |
| config_value | TEXT | 配置值（敏感项加密存储） |
| is_encrypted | TINYINT DEFAULT 0 | 是否加密存储 |
| 通用字段 | | |

索引：INDEX(application_id, config_type)

### 模块 F：系统管理

#### bx_dict 字典表 / bx_dict_item 字典项表
bx_dict: id, name, type(唯一), status, remark, 通用字段
bx_dict_item: id, dict_type, label, value, sort, status, 通用字段

#### bx_config 参数配置表
| id | config_group | name | config_key(唯一) | config_value | type | is_system | remark | 通用字段 |

#### bx_third_config 第三方服务全局配置表（平台级 + 租户级，用归属机制）
> 存储平台级和租户级的存储/短信等服务配置，应用级配置存 bx_app_config
| 字段 | 类型 | 说明 |
|---|---|---|
| id | BIGINT UNSIGNED | 主键 |
| owner_type | ENUM('platform','tenant') | 归属 |
| owner_id | BIGINT UNSIGNED DEFAULT 0 | 归属ID |
| service_type | VARCHAR(30) | 服务类型 storage/sms |
| provider | VARCHAR(30) | 提供商 alioss/qiniu/alisms/txsms |
| config | JSON | 配置内容（敏感项加密） |
| is_default | TINYINT DEFAULT 0 | 是否该服务的默认配置 |
| status | TINYINT | |
| 通用字段 | | |

> 配置继承：租户读配置时，先查自己(owner=tenant)，无则回退平台(owner=platform)。

#### bx_oper_log 操作日志表
| id | admin_type | admin_id | tenant_id | module | action | method | url | ip | params(JSON) | result | duration | created_at |

#### bx_login_log 登录日志表
| id | admin_type | admin_id | tenant_id | username | ip | location | browser | os | status | msg | created_at |

#### bx_file 文件表
| id | owner_type | owner_id | original_name | file_name | file_path | url | mime | size | storage(local/oss/qiniu) | hash | 通用字段 |

### 模块 G：开发工具

#### bx_gen_table 代码生成-表配置
| id | table_name | table_comment | module_name | business_name | class_name | function_name | tpl_type | options(JSON) | 通用字段 |

#### bx_gen_column 代码生成-字段配置
| id | table_id | column_name | column_comment | column_type | php_type | ts_type | is_query | is_list | is_form | query_type | html_type | dict_type | sort |

#### bx_crontab 定时任务表
| id | name | type | target | rule(cron表达式) | params | status | last_run_at | next_run_at | remark | 通用字段 |

### 模块 H：C 端用户（身份/凭证/资料三层分离）★重要设计

> 核心理念：用户身份(User) 与 登录凭证(Auth) 与 应用内资料(Profile) 三层分离。
> 身份打通策略：策略 A——同租户下多应用通过 unionid/手机号打通为同一个人，跨租户不打通。
> 会员资料模式：租户级可配置 shared(共享)/isolated(独立)。

#### bx_user C端用户主体表（身份层，按租户打通）
| 字段 | 类型 | 说明 |
|---|---|---|
| id | BIGINT UNSIGNED | 主键 |
| owner_type | ENUM('platform','tenant') | 归属类型（注意：归属在租户层，非应用层） |
| owner_id | BIGINT UNSIGNED DEFAULT 0 | 归属ID |
| union_id | VARCHAR(64) | 微信UnionID（同租户内打通多应用的钥匙） |
| phone | VARCHAR(20) | 手机号（另一打通钥匙） |
| nickname | VARCHAR(100) | 昵称 |
| avatar | VARCHAR(255) | 头像 |
| gender | TINYINT DEFAULT 0 | 性别 |
| member_level_id | BIGINT UNSIGNED DEFAULT 0 | 会员等级（shared模式用） |
| points | INT DEFAULT 0 | 积分（shared模式用） |
| balance | DECIMAL(12,2) DEFAULT 0 | 余额（shared模式用） |
| status | TINYINT | 状态 |
| 通用字段 | | created_at等 |

> 说明：同一真人在一个租户内只有一条记录。已移除原 application_id 字段。
> shared 模式下会员数据存本表；isolated 模式下存 bx_user_app_profile。

索引：INDEX(owner_type, owner_id)、INDEX(union_id)、INDEX(phone)、INDEX(status)

#### bx_user_auth C端用户登录凭证表（凭证层，一对多）
| 字段 | 类型 | 说明 |
|---|---|---|
| id | BIGINT UNSIGNED | 主键 |
| user_id | BIGINT UNSIGNED | 指向 bx_user（身份打通核心） |
| application_id | BIGINT UNSIGNED | 来源应用 |
| auth_type | VARCHAR(20) | 登录方式 wechat_mini/wechat_oa/alipay/phone/password |
| open_id | VARCHAR(64) | 该应用下的 openid |
| credential | VARCHAR(255) | 凭证（密码hash/token等，按需加密） |
| 通用字段 | | |

> 说明：同一 user_id 可有多条 auth（应用A的openid、应用B的openid、手机号密码...）。
> 登录时用 (application_id + open_id) 定位到 user_id，实现多应用归一。

索引：UNIQUE(application_id, auth_type, open_id)、INDEX(user_id)

#### bx_user_app_profile C端用户应用内档案表（资料层，isolated模式启用）
| 字段 | 类型 | 说明 |
|---|---|---|
| id | BIGINT UNSIGNED | 主键 |
| user_id | BIGINT UNSIGNED | 指向 bx_user |
| application_id | BIGINT UNSIGNED | 所属应用 |
| member_level_id | BIGINT UNSIGNED DEFAULT 0 | 应用内会员等级 |
| points | INT DEFAULT 0 | 应用内积分 |
| balance | DECIMAL(12,2) DEFAULT 0 | 应用内余额 |
| ext | JSON | 应用内扩展资料 |
| 通用字段 | | |

> 说明：仅当租户配置 member_data_mode=isolated 时启用，按 user_id+application_id 隔离。
> shared 模式下此表不产生数据，会员数据读 bx_user 本身。

索引：UNIQUE(user_id, application_id)、INDEX(application_id)

#### 租户会员数据模式配置
在 bx_tenant 表增加字段，或存于 bx_config（租户级参数）：
| member_data_mode | VARCHAR(10) DEFAULT 'shared' | 会员数据模式 shared/isolated |

---

## 五、安全相关设计要点

1. **密钥加密**：bx_app_config 和 bx_third_config 中的密钥类字段，存储前 AES-256 加密，应用层解密使用，前端展示掩码。
2. **越权防护**：所有带 owner_type/owner_id 的表，查询必须经过统一 OwnerScope 作用域。
3. **软删除**：所有表用 deleted_at 软删除，不物理删除（审计需求）。
4. **审计**：created_by/updated_by 记录操作人，配合 oper_log。
5. **联合唯一**：租户内 username 唯一用联合索引(tenant_id, username)，应用内 openid 唯一用(application_id, auth_type, open_id)。

---

## 六、初始化数据（种子数据）

需准备的初始化数据（JSON，存契约仓库）：
1. 默认平台超级管理员账号（密码首次登录强制修改）
2. 平台菜单树（admin_type=platform）
3. 租户菜单树（admin_type=tenant）
4. 默认租户套餐（免费版/标准版/企业版）
5. 系统字典（性别、状态、是否、应用类型等）
6. 系统参数默认值

---

## 七、待后续阶段扩展的表（占位，本阶段不建）
- 业务插件表（文章 bx_article、分类、标签、Banner）
- 会员系统表（等级 bx_member_level、积分明细、签到）
- 支付订单表
- 消息中心表
> 插件表在各插件开发时定义，遵循同样的归属机制和命名规范。

---

## 八、C 端用户登录打通流程（策略 A：按租户打通）

以微信小程序登录为例，其他登录方式（公众号/支付宝/手机号）逻辑类似：

```
1. 用户在【应用A】小程序登录，前端拿到 wx code
2. 后端用 code + 应用A的 appid/secret 换取 openid + unionid
3. 查 bx_user_auth：(application_id=A, auth_type=wechat_mini, open_id) 是否存在？
   ├── 存在 → 取出 user_id，直接登录（老用户）
   └── 不存在 → 进入第 4 步
4. 用 unionid 在【本租户范围内】查 bx_user：是否已有此人？
   （owner_type+owner_id 限定租户，union_id 匹配）
   ├── 找到 → 说明此人在本租户其他应用登录过
   │         → 不创建新 user，仅新增一条 bx_user_auth
   │           （把应用A的 openid 绑定到已有 user_id）
   │         → 实现"打通"：仍是同一个人
   └── 未找到 → 全新用户
             → 创建 bx_user（新身份）+ bx_user_auth（凭证）
             → isolated 模式下，按需创建 bx_user_app_profile
5. 生成并返回 JWT（含 user_id、owner_type、owner_id、application_id）
```

### 会员数据读取（统一服务 MemberService，业务层无感知）
```
getMemberData(user_id, application_id):
    mode = 取本租户 member_data_mode 配置
    if mode == 'shared':
        return bx_user 上的 member_level_id/points/balance
    else (isolated):
        return bx_user_app_profile 中 (user_id, application_id) 的记录
        若不存在则惰性创建默认档案
```

### 关键安全约束
- unionid 打通必须限定在同一 owner（租户）范围内，严禁跨租户匹配。
- 跨租户即使 unionid 相同（理论上不会，因不同租户用不同开放平台账号）也不打通。
- 手机号打通同理，限定本租户范围。

---

## 九、表数量统计
本阶段共 29 张表（原 28 张 + 新增 bx_user_app_profile）。
用户模块从 2 张表（user/user_auth）扩展为 3 张表（user/user_auth/user_app_profile）。
