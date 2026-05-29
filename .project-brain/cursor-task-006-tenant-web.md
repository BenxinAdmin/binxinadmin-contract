# Cursor 任务书 #006：tenant-web 租户后台

> 用途：建立租户后台前端项目，与 admin-web 形成"两个独立后台"对照。
> 仓库：binxinadmin-tenant-web
> 本地路径：/Users/daxing/projects/BenxinAdmin/binxinadmin-tenant-web
> 前置：task-004（admin-shared v0.1.2）+ task-005（admin-web 已完工）+ TP 后端 task-002
> 配套：frontend-theming.md（差异化方案）、cursor-workflow.md（工作规范）

---

## 角色设定（粘贴给 Cursor）

```
你是资深前端工程师，task-005 admin-web 已由你交付并完成。现在做对称的 tenant-web 租户后台。
租户后台供租户管理员使用（管理本租户的应用、用户、数据等）。
请优先复用 task-005 的实现模式，仅在差异化方案规定的位置做调整。
严格遵守 cursor-workflow.md 的四步自检清单与 commit 规范。
```

---

## 一、与 admin-web 的关系（核心策略）

**结构和组件 100% 复用 admin-web 的模式**，仅在以下位置做差异化：

| 维度 | admin-web 平台后台 | tenant-web 租户后台 |
|---|---|---|
| **后端应用** | platform（仅平台超管） | admin（租户管理员） |
| **API 前缀** | `/platform/*` | `/admin/*` |
| **JWT 守卫类型** | PlatformGuard | TenantGuard |
| **主题色 --bxa-color-primary** | `#1d4ed8` 深蓝 | `#059669` 翠绿 |
| **侧边栏风格** | 深色科技风 `#001529` | **浅色清新风 `#ffffff`，左侧绿色细条强调** |
| **Logo 文字** | `BinxinAdmin · Platform` | `BinxinAdmin · Tenant` |
| **登录页主标题** | 本心系统管理平台 | 本心商户管理后台 |
| **登录页副标题** | Platform Console | Tenant Console |
| **登录页背景** | 深色科技风渐变 + 几何点阵 | 浅色清新风渐变 + 曲线/柔和图案 |
| **顶栏右上身份角标** | 蓝色徽章 `平台` | 绿色徽章 `租户` |
| **Tab 标题** | `平台管理 - BinxinAdmin` | `商户管理 - BinxinAdmin` |
| **Favicon** | 蓝色盾形 + B | **绿色盾形 + B** |
| **meta theme-color** | `#1d4ed8` | `#059669` |
| **token storage key** | `platform_bxa_token` / `platform_bxa_refresh_token` | `tenant_bxa_token` / `tenant_bxa_refresh_token` |
| **端口** | 5173 | **5174**（避免与 admin-web 同时调试冲突） |
| **Vite 代理路径** | `/platform → http://localhost:8080` | `/admin → http://localhost:8080` |

**其他全部一致**：技术栈、目录结构、组件、布局、路由模式、Mock 策略、安全约束、响应式策略。

---

## 二、技术栈（与 admin-web 一致）

| 项 | 版本 |
|---|---|
| Node.js | 24.x（.nvmrc） |
| 包管理器 | pnpm 9+ |
| Vue / TypeScript / Vite | 同 admin-web |
| @bxa/admin-shared | `link:../binxinadmin-admin-shared`（共享包 v0.1.2） |

vite.config.ts 必须带 `resolve.dedupe`（含 `vue/vue-router/pinia/element-plus/@vue/runtime-core`），避免双实例（D-041 教训）。

---

## 三、目录结构（与 admin-web 镜像，文件名替换）

```
binxinadmin-tenant-web/
├── .nvmrc / .gitignore / .prettierrc / eslint.config.js
├── package.json / tsconfig.json / vite.config.ts / uno.config.ts
├── index.html                       Tab 标题: 商户管理 - BinxinAdmin
├── public/
│   └── favicon.svg                  绿色盾形 + B（自绘 SVG）
└── src/
    ├── env.d.ts / main.ts / App.vue
    ├── api/                         同 admin-web 结构
    │   ├── index.ts                 createHttpClient: baseURL/appPrefix='admin'
    │   ├── auth.ts                  login → /admin/auth/login；apiGetUserMenus 直接 return MOCK_MENUS
    │   ├── system.ts                getDictItems 直接 return MOCK_DICTS
    │   └── types.ts
    ├── stores/                      同 admin-web 五件套
    │   ├── auth.ts                  useAuthStore (id='tenant-auth'，区分 admin-web)
    │   └── ...
    ├── router/
    │   ├── index.ts / guards.ts / constants.ts
    ├── styles/
    │   └── theme.css                ★ 覆盖共享包变量为绿色品牌 + 浅色侧边栏
    ├── layouts/
    │   ├── TenantLayout.vue         包装 BasicLayout，注入 Logo 和 Tenant 徽章
    │   └── components/
    │       └── TenantBadge.vue      绿色"租户"徽章
    ├── composables/
    │   ├── useDictResolver.ts       直接返回 MOCK_DICTS 实现（D-042 mock 优先）
    │   └── usePageTitle.ts
    ├── views/
    │   ├── login/
    │   │   ├── index.vue            主标题"本心商户管理后台"，副标题"Tenant Console"
    │   │   └── components/
    │   │       ├── LoginForm.vue
    │   │       └── LoginBackground.vue   浅色清新风 SVG（自绘曲线/圆点，不深色）
    │   ├── dashboard/index.vue      Hero 卡 + 健康检查 /admin/health
    │   ├── profile/index.vue
    │   ├── redirect/index.vue
    │   ├── error/{403,404}.vue
    │   └── demo/                    沿用 admin-web 演示页结构，可同名复用
    │       ├── x-table-demo.vue     含临时模拟非超管 Switch（直接复用 admin-web 模式）
    │       └── theme-demo.vue       演示绿色主色 + 浅色侧栏切换
    └── README.md                    含已知优化项（v0.2.0：bundle 减量 + Router warn 修复）
```

---

## 四、关键样式（styles/theme.css）

```css
:root {
  /* 主色：翠绿 */
  --bxa-color-primary: #059669;
  --bxa-color-primary-hover: #047857;
  --bxa-color-primary-active: #065f46;

  /* 浅色清新风侧边栏（核心差异化）*/
  --bxa-bg-sidebar: #ffffff;
  --bxa-sidebar-text: #374151;            /* 深灰文字 */
  --bxa-sidebar-text-active: #059669;     /* 激活态：绿色文字 */
  --bxa-sidebar-active-bg: #ecfdf5;       /* 激活背景：极浅绿 */
  --bxa-sidebar-hover-bg: #f9fafb;        /* hover：极浅灰 */
  --bxa-sidebar-border-color: #e5e7eb;    /* 浅灰边框 */
  --bxa-sidebar-logo-color: #059669;      /* Logo 区绿色 */

  /* 登录页变量：浅色清新 */
  --bxa-login-bg-gradient-start: #ecfdf5;
  --bxa-login-bg-gradient-end:   #ffffff;
  --bxa-login-title-color:       #064e3b;
  --bxa-login-card-bg:           #ffffff;
}

html.dark {
  /* 暗色模式下品牌主色保持绿色不变 */
  --bxa-color-primary: #10b981;
}
```

**视觉细节**：浅色侧边栏视觉上要有"清新商户感"，可在 Sidebar logo 区左侧加一条 3-4px 的绿色细条作为品牌标记（CSS border-left 实现，不依赖共享包）。

---

## 五、登录页背景设计

与 admin-web 风格反向：admin-web 是深色科技风（深蓝渐变 + 几何点阵），tenant-web 必须**浅色清新风**：

- 背景：浅绿到白的渐变（`linear-gradient(135deg, #ecfdf5 0%, #ffffff 100%)`）
- 装饰：纯 SVG 自绘的**柔和曲线/同心圆/绿色淡色斑点**，给人"商户友好"的视觉
- 不要太多元素，保持留白
- 严禁引入网络图片或商业素材

---

## 六、Mock 策略（吸取 task-005 教训）

**必须从一开始就按"mock 优先"实现**，不能写"先 try 再 fallback"——这是 cursor-workflow.md Step 1-4 的硬要求：

```typescript
// api/auth.ts
export async function apiGetUserMenus() {
  // 任务书规定后端接口未实现，前端 mock 优先。直接 return，不发请求。
  return MOCK_MENUS
}

// composables/useDictResolver.ts
export const tenantDictResolver: DictResolver = async (type) => {
  // 任务书规定 M3 字典模块前 mock 优先。直接 return，不发请求。
  return MOCK_DICTS[type] || []
}
```

**Mock 菜单结构**：包含"控制台 / 应用管理 / 用户管理 / 个人中心 + 功能演示子菜单"等租户视角菜单（不要复制 admin-web 的菜单语义，租户管的是"应用/用户/订单"，不是"租户/平台配置"）。但**为了演示一致性，沿用 demo/x-table-demo 和 demo/theme-demo 演示页**，验证共享包能力。

---

## 七、Vite 配置（针对租户特化）

```typescript
// vite.config.ts 关键
export default defineConfig({
  server: {
    port: 5174,                       // 避开 admin-web 的 5173
    proxy: {
      '/admin': {                     // 注意是 /admin，不是 /platform
        target: 'http://localhost:8080',
        changeOrigin: true,
      },
    },
  },
  resolve: {
    alias: { '@': fileURLToPath(new URL('./src', import.meta.url)) },
    dedupe: ['vue', 'vue-router', 'pinia', 'element-plus', '@vue/runtime-core'],
  },
  optimizeDeps: {
    include: ['@bxa/admin-shared'],
  },
})
```

---

## 八、Token 存储 Key（避免与 admin-web 冲突）

createHttpClient 的 token storage key 必须用 `tenant_` 前缀：

```typescript
createHttpClient({
  baseURL: import.meta.env.VITE_API_BASE || 'http://localhost:5174',  // 通过 vite proxy
  appPrefix: 'admin',                                                   // 后端 admin 应用
  tokenStorageKey: 'tenant_bxa_token',
  refreshTokenStorageKey: 'tenant_bxa_refresh_token',
  // ...
})
```

这样同主机同时调试 admin-web (5173) 和 tenant-web (5174) 时，**两套 token 互不覆盖**（D-027 token 前缀策略）。

---

## 九、登录注意（后端未必有租户管理员账号）

**TP 后端 task-002 只创建了平台超管账号 superadmin**，没有创建租户管理员账号。tenant-web 的登录链路需要租户管理员账号才能完整测试。

**两种处理方案**：

**方案 A：临时种子（推荐）**
让 Cursor 在 binxinadmin-backend-tp 加一个临时 seeder `TenantAdminDemoSeed`，创建一个 demo 租户 + 一个 demo 租户管理员账号（如 `tenant_admin` / 强随机密码写本地文件）。M2 用户管理上线后此 seeder 可保留作演示数据。

> 密码生成规则参考 PlatformAdminSeed：随机生成 + 写本地文件 + 不入仓库（D-033）。

**方案 B：本轮只跑登录页 UI + Mock 化登录**
后端真实接口不接，登录页接 mock 直接返回假 token + 假用户，进入 dashboard 看 UI 即可。等 M1 阶段做完整登录功能时再连真后端。

**默认采用方案 A**——双后台都能真实登录是验证完整链路的关键。如果方案 A 工作量超出预期，Cursor 主动告知再切方案 B。

---

## 十、十二项验证清单（与 task-005 对照）

| # | 验证项 |
|---|---|
| 1 | Node 24 + pnpm 9.15 |
| 2 | `pnpm install` 成功，@bxa/admin-shared link 到 v0.1.2 |
| 3 | `pnpm type-check` 0 错误 |
| 4 | `pnpm lint --max-warnings 0` 0 错误 |
| 5 | `pnpm build` 0 错误 |
| 6 | `pnpm dev` 启动到 http://localhost:5174 无 console error |
| 7 | favicon 自绘 SVG（绿色盾形 + B） |
| 8 | .gitignore 完整（含 .env* / .local-credentials.json） |
| 9 | 登录页零硬编码密码，主标题"本心商户管理后台" |
| 10 | Vite 代理 /admin/health 通后端 |
| 11 | mock 菜单 + mock 字典不发请求（Network filter 空） |
| 12 | 4 个浏览器手工测试：登录 → dashboard → x-table demo 临时模拟开关 → theme demo 重置默认 |

---

## 十一、安全/工程硬约束（继承 task-005）

1. **零硬编码密码** — 登录表单不预填，登录成功立即 password = ''
2. **.gitignore 完整** — `.local-credentials.json` / `.env*` / `node_modules` / `dist` / `.vite` / `.DS_Store` / `*.log`
3. **不打包敏感信息** — API 地址走 `.env.development.example`
4. **字体/图标/图片/favicon/登录页背景** 必须开源或自绘
5. **vite.config.ts 必须含 resolve.dedupe**（D-041 教训）
6. **首次 commit 用 `feat:` 前缀**（D-046 教训）

---

## 十二、不在本任务范围

- 完整租户业务页面（应用管理、用户管理等业务模块）→ 阶段 1+
- 真实租户级菜单接口对接 → M2 用户管理
- C 端 API、Laravel 后端对接

---

## 十三、提交规范

请先告诉我执行计划：
- vite.config.ts、theme.css、登录页背景设计要点
- 临时种子方案 A/B 选择（推荐 A，工作量预估告诉我）
- 5 个差异化点的实施方式

确认后再开始。**严格遵守 cursor-workflow.md 四步自检清单**：
- 修复前回查任务书意图
- 不用技术名词包装绕过
- 验证不能只靠编译时检查

做完先不要 commit，提供改动清单 + 验证日志，等我 review。

**首次 commit message 模板**（D-046 前缀必须 feat）：

```
feat: 完成租户后台前端骨架（task-006）

- 脚手架：与 admin-web 同构 Node 24/pnpm/TS strict/Vite 5/EP/UnoCSS/Pinia
- 业务主干：登录 → 取菜单 → 动态路由 → TenantLayout → Dashboard
- 主题差异化：翠绿品牌 #059669 + 浅色清新风侧边栏（共享包 v0.1.2 变量驱动）
- Mock 哲学：menus / dictionary 直接 return，符合"mock 优先"意图
- Token 前缀：tenant_bxa_token，避免与 admin-web 同主机调试冲突
- 端口 5174，proxy /admin → backend-tp 8080
- favicon 自绘绿色盾形 SVG，登录页"本心商户管理后台 / Tenant Console"
- 配套：（若走方案 A）backend-tp 加 TenantAdminDemoSeed 创建演示租户管理员
```

如果同步改了 backend-tp，**那一侧用独立 commit**（不混在 tenant-web 仓库），message 模板：
```
feat: 新增租户管理员演示账号 seeder（供 tenant-web 联调）
```
