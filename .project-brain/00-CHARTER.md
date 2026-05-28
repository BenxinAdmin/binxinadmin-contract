# BinxinAdmin 项目宪章（Project Charter）

> 本文档是 BinxinAdmin 项目的最高基准文档（Single Source of Truth）。
> 所有架构决策、开发规范、协作方式均以本文档为准。
> 任何对核心决策的变更，必须同步更新本文档与 `05-DECISIONS-LOG.md`。

---

## 一、项目身份

| 项 | 内容 |
|---|---|
| 项目名 | **BinxinAdmin** |
| 中文名 | 本心通用后台管理系统 |
| 定位 | 面向中国市场的、PHP 全栈、多租户、可插拔的开源中后台快速开发框架 |
| License | Apache 2.0 |
| 仓库策略 | 多仓库，GitHub + Gitee 双开，立即公开 |
| 作者署名 | 仗键天涯(daxing) / 3442535897@qq.com |

## 二、一句话价值主张

让国内开发者用最熟悉的技术栈（PHP + Vue + uni-app），最快速度搭建一个企业级、可定制、可商用的全栈系统。

## 三、五大差异化竞争力（按重要性）

1. **双后端支持**：ThinkPHP 8.1 + Laravel 13，覆盖两大 PHP 社区，市面罕见。
2. **代码生成器**：核心卖点，一键生成前后端 CRUD（双后端 + 前端 + SQL）。
3. **深度本地化**：微信生态、国内支付、国内云存储、国内短信，全部后台可视化配置。
4. **全栈闭环**：管理后台 + 官网 + 小程序/H5/App，一套搞定。
5. **插件化业务模块**：文章、商城等业务模块可插拔，接单即用。

## 四、已锁定核心决策（不可随意变更）

| 决策项 | 最终结论 |
|---|---|
| PHP 版本 | PHP 8.3 |
| 后端框架 | ThinkPHP 8.1.x + Laravel 13（双版本） |
| 双后端开发策略 | 契约先行，每模块 TP 与 Laravel 紧挨着做 |
| 数据库 | MySQL 8.0 |
| 缓存/队列 | Redis 7 |
| 多租户 | 是。共享数据库 + tenant_id 字段隔离（Discriminator 模式） |
| 管理后台前端 | Vue 3.4+ + TypeScript + Vite 5 + Element Plus 2.x，从零搭建 |
| 前端原子化 CSS | UnoCSS |
| 前端状态管理 | Pinia + Vue Router 4 |
| C 端 | uni-app + Vue 3 + TS，输出 小程序 + H5 + App |
| C 端 UI 库 | wot-design-uni |
| 官网 | Nuxt 3（SSR/SSG，SEO 友好） |
| API 风格 | 业务码风格 {code, message, data, timestamp, request_id} |
| 认证授权 | JWT + Casbin RBAC |
| 字体/图标/图片 | 仅用开源资源（见 03-CONVENTIONS.md 合规章节） |
| 技术细节决策权 | Claude（架构师）全权决定，按业界最佳实践 |

## 五、角色与协作模式

- **Derek（你）**：产品负责人 + 执行者（通过 Cursor 实现代码）。
- **Claude（我）**：项目经理 + 架构师。负责规划、架构设计、Cursor 需求文档、进度管理、技术决策、代码审查指引。不写实际代码，只产出能让 Cursor 完成开发的需求提示词。
- **Cursor**：代码实现工具。
- **协作流**：你提需求 → 我产出 Cursor 需求文档 → 你用 Cursor 实现 → 反馈结果 → 我审查并给下一步。

## 六、仓库清单

| 仓库名 | 用途 |
|---|---|
| binxinadmin-contract | 契约仓库：DB Schema、API 规范、错误码、菜单权限数据、项目大脑（最重要） |
| binxinadmin-backend-tp | ThinkPHP 8.1 后端（多应用：platform/admin/api/common） |
| binxinadmin-backend-laravel | Laravel 13 后端（多守卫：platform/admin/api） |
| binxinadmin-admin-web | 系统后台/平台后台前端（平台超管用，对接 platform 应用） |
| binxinadmin-tenant-web | 租户后台/商户后台前端（租户管理员用，对接 admin 应用） |
| binxinadmin-admin-shared | 前端共享包（CRUD组件/请求封装/权限指令/布局/主题/类型） |
| binxinadmin-uniapp | uni-app C 端（小程序+H5+App，对接 api 应用） |
| binxinadmin-website | Nuxt 3 官网 |
| binxinadmin-docs | VitePress 文档站 |

## 七、契约仓库的核心地位

`binxinadmin-contract` 是双后端策略的核心，存放"框架无关的真相源"：
- 数据库设计（SQL + 字段说明 + ER 图）
- API 接口契约（OpenAPI 3.0）
- 统一错误码表
- 菜单/权限/字典初始化数据（JSON）
- 业务逻辑伪代码
- 项目大脑文档（.project-brain/）

两个后端都"实现"这份契约，前端只对接这份契约，保证两套后端对前端完全一致。
