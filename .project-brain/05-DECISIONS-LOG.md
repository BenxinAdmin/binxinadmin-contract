# BinxinAdmin 决策日志

> 记录每个重要决策、时间、理由。便于追溯"当初为什么这么定"。

---

| 编号 | 决策 | 理由 | 关联 ADR |
|---|---|---|---|
| D-001 | PHP 版本定 8.3 | 最稳定，Laravel 13 最低要求，生态成熟 | - |
| D-002 | 双后端同时做（契约先行） | 避免做完一遍再重来，逻辑只设计一次 | ADR-002 |
| D-003 | 做成多租户架构 | 对标 BenXin SaaS，一套系统多租户共用 | ADR-001 |
| D-004 | 管理后台前端从零搭建 | 完全可控、纯净，不依赖第三方脚手架 | - |
| D-005 | 仓库立即公开，GitHub+Gitee 双开 | 尽早建立开源影响力 | - |
| D-006 | 技术细节由架构师全权决定 | 减少 Derek 决策负担，按业界最佳实践 | - |
| D-007 | License 选 Apache 2.0 | 专利保护，适合可能商业化的开源项目 | ADR-007 |
| D-008 | 原子化 CSS 选 UnoCSS | 更快、按需生成、Vite 集成更好 | ADR-008 |
| D-009 | C 端 UI 库选 wot-design-uni | 更现代、Vue3+TS 原生、维护活跃 | ADR-009 |
| D-010 | 主键自增 BIGINT 不用雪花 | 避免 JS 精度丢失，多租户靠 tenant_id 隔离 | ADR-004 |
| D-011 | 账号体系用分离表 | 平台管理员与租户管理员物理隔离，两套RBAC，安全边界更强 | ADR-010 |
| D-012 | 统一归属抽象 owner_type+owner_id | 应用/C端用户/业务数据统一归属机制，可挂平台或租户，越权校验复用一套 | ADR-011 |
| D-013 | 应用(Application)为独立实体 | app_type 区分微信小程序/公众号/H5/App，每应用独立 appid 等配置 | ADR-012 |
| D-014 | 敏感配置平台统一分配 | 租户不能自填 appid/支付密钥，由平台代填或审核，密钥 AES 加密存储 | ADR-013 |
| D-015 | 后端多应用/多端模式 | platform/admin/api 三端分离，与分离账号体系契合 | ADR-014 |
| D-016 | 管理后台拆两个独立前端项目 | 系统后台 admin-web + 租户后台 tenant-web，彻底隔离，靠共享包复用代码 | ADR-015 |
| D-017 | Node.js 用 24.x（Active LTS） | 支持至2028-05，新项目标准；不用26(未转LTS)/20(已EOL) | - |
| D-018 | 前端包管理器用 pnpm | 多前端仓库+共享包场景，依赖/速度/workspace 优于 npm | - |
| D-019 | C端用户身份/凭证/资料三层分离 | 解决一人多应用openid不同导致一人多账号问题；身份按租户唯一 | ADR-016 |
| D-020 | 用户身份按租户打通(策略A) | 同租户多应用=一个人(unionid/手机号)，跨租户不打通 | ADR-016 |
| D-021 | 会员资料模式后台可配置 | member_data_mode: shared共享/isolated独立，适应不同客户 | ADR-016 |
| D-022 | 用 Docker 统一开发环境 | 本地PHP8.2/MySQL9/Redis8与项目要求(8.3/8.0/7)不符，Docker隔离保证环境一致 | - |
| D-023 | Docker配置放契约仓库docker/目录 | 两个后端共用的基础设施，端口避让(MySQL3307/Redis6380) | - |
| D-024 | 两后端共用一个数据库binxinadmin | TP负责建表+种子，Laravel直接连用不跑迁移(省时间/数据一致)；Laravel迁移文件保留作示范+单独部署备用 | ADR-017 |
| D-025 | 渐进式登录安全策略(M1实现) | 平时不强制验证码，失败3次触发图形验证码、5次锁定15分钟；记入m1-requirements-preview.md | - |
| D-026 | task-002决策:不装Casbin/简单登录/NOT NULL补丁迁移/跑链路验证 | 本轮专注骨架+安全边界，业务细节留对应阶段 | - |
| D-027 | 前端架构:本地file:依赖+轻度差异化+平板手机响应式+分3次任务书 | 共享包复用率最大化，差异只在主题色/Logo/登录页/favicon/顶栏角标 | - |
| D-028 | git commit message 一律用中文 | 前缀保留约定式英文关键字(feat/fix/docs等)，冒号后说明用中文 | - |
| D-029 | 本地项目根目录改为 /Users/daxing/projects/BenxinAdmin/ | 原 xinzhi 已弃用；详见 dev-environment.md | - |
| D-030 | 严禁明文密码/密钥进入仓库 | 含 .project-brain；所有敏感凭据只存本地密码管理器；Claude产出文档前必须grep扫一遍 | - |
| D-031 | 首次 superadmin 密码因文档误写已泄露需重置 | 2026-05旧密码已push至公开仓库04-NEXT-ACTIONS.md历史，已执行PlatformAdminSeed重置 | - |
| D-032 | seeder幂等导致首次重置失败 | DELETE记录后再跑seeder才生效；M1将做--force参数和reset-password命令 | m1-requirements-preview |
| D-033 | 凭据输出规范：写文件不打stdout | 从两次密码事故得出；密码/密钥/Token默认写本地gitignore文件，CLI仅提示路径 | m1-requirements-preview |
| D-034 | 前端项目引用共享包用 link: 协议 | admin-web/tenant-web 在 package.json 用 "@bxa/admin-shared":"link:../binxinadmin-admin-shared" | - |
| D-035 | GitHub 新建仓库永远不勾任何初始化选项 | 防止 push rejected；初始 README/LICENSE/.gitignore 由本地代码携带 | - |
| D-036 | admin-shared v0.1.1 补 sidebar 变量 | task-005 实施时发现 v0.1.0 sidebar 变量遗漏，补 7 个变量+桥接 el-menu token；零破坏性升级 | - |
| D-037 | 共享包变量契约原则 | 完整>稳定：发现合理遗漏的变量应立即补，而非用 :deep 在调用方私下解决 | - |
| D-038 | task-005 admin-web 完成 | 41文件，13项验证全过，4架构亮点，懒注入打破循环依赖+动态路由首次访问装载+登录立即清密码+重置默认验证sidebar变量 | - |
| D-039 | v0.2.0 候选优化项 | element-plus按需引入/iconify按需加载/路由级chunk分割，目标gzip 455KB→200KB | - |
| D-040 | 共享包 v0.1.2 候选 | sidebar-logo slot + permissionCodes 递归收集；task-005 实施时记录的两个改进点 | - |
| D-041 | pnpm link协议双实例问题处理 | admin-web/admin-shared 各装一份pinia/vue/element-plus导致 getActivePinia 返回null；vite.config.ts 加 resolve.dedupe 强制单实例 | - |
| D-042 | 沉淀 Cursor 工作规范文档 | task-005 字典mock事件后 Cursor 反思的四步自检清单，写入 cursor-workflow.md 作为项目规范 | cursor-workflow.md |
| D-043 | 修复时禁用"听起来合理的技术包装" | 必须先讲设计差距再讲技术手段；事后合理化解释是反模式 | cursor-workflow.md |
| D-044 | 共享包 v0.1.2 修复守卫混用陷阱 | next参数与return混用导致Invalid navigation guard；改纯返回式(Vue Router 4推荐)；零破坏升级 | - |
| D-045 | Vue Router 4 守卫签名规范 | 共享包/项目所有路由守卫一律不接 next 参数，用返回值控制流程(true放行/false取消/RouteLocationRaw重定向) | cursor-workflow.md |
| D-046 | commit message 前缀必须匹配仓库状态 | 首次 commit 必须 feat:；不能在 No commits yet 时用 fix:；审 commit message 前必须看仓库状态 | cursor-workflow.md |
| D-047 | 共享包 v0.1.3 候选(侧边栏装饰条变量) | --bxa-sidebar-accent-position/-width/-color 三件套；tenant-web 启动时补全，admin-web 零破坏 | - |
| D-048 | 共享包 className 命中也属反模式 | :deep 和直接命中.bx-sidebar 是同类问题——硬编码共享包内部实现，违反 D-037 完整>稳定原则 | cursor-workflow.md |
| D-049 | v0.1.3 装饰条 = 框架级思维样板 | 主动扩到四方向+MutationObserver反射data-attr+双默认值守护+Changelog写入反模式指正；体积+0.5KB | - |
| D-050 | CSS变量驱动DOM选择器的标准模式 | 用MutationObserver监听根变量变化，反射到组件data-attr，让CSS变量间接驱动[data-x]选择器 | - |
| D-051 | 第三次密码事故(tenant_admin) | Cursor 交付报告直接贴出密码明文；旧密码已轮换；规范扩展到聊天/报告 | - |
| D-052 | 凭据规范的最终形态 | seeder/CLI 既不 stdout 打印也不交付报告贴出；只提示文件路径；密码用户自己 cat 一次抄到密码管理器；交付报告凭据描述格式标准化 | cursor-workflow.md |

---

## 待决策（标记，到相应阶段再定）
- 代码生成器模板引擎：Twig vs 框架自带（到阶段 3 定）
- API 文档工具：Apifox vs Swagger/OpenAPI 自动生成（到 M4 定，倾向 OpenAPI 自动生成 + Apifox 导入）
- uni-app 与前端共享代码的方式：npm 私有包 vs monorepo（到阶段 5 定）
- 队列驱动：Redis vs 数据库（默认 Redis，到需要时确认）
