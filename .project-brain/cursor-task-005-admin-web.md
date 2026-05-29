# Cursor 任务书 #005：admin-web 平台后台

> 用途：建立平台后台前端项目，使用 @bxa/admin-shared 共享包，对接 TP 后端 platform 应用。
> 仓库：binxinadmin-admin-web
> 前置：task-004 admin-shared 已完成；TP 后端 #002 已可用（http://localhost:8080）
> 配套：frontend-theming.md（差异化方案）、03-CONVENTIONS.md
> 本地路径：/Users/daxing/projects/BenxinAdmin/binxinadmin-admin-web

---

## 角色设定（粘贴给 Cursor）

```
你是资深前端工程师，精通 Vue 3 + TypeScript + Vite + Element Plus + Pinia。
你正在为 BinxinAdmin 多租户开源后台搭建"平台后台"前端项目（admin-web）。
平台后台供平台超级管理员使用（管理租户、应用、全局配置等）。
该项目通过本地 file: 协议依赖 @bxa/admin-shared 共享包。
代码遵循 ESLint + Prettier，TypeScript strict 模式，commit message 用中文。
```

---

## 总目标

搭建 admin-web 项目骨架，能完成以下闭环：
1. 用户访问首页 → 未登录跳转登录页
2. 登录页输入 superadmin 账号密码 → 后端登录成功 → 拿到 token
3. 拉取菜单数据 → 动态生成路由 → 跳转到首页
4. 看到完整布局（侧栏菜单 + 顶栏 + 多标签页 + 内容区）
5. 可切换主题色、暗色模式、字号、登出
6. 按钮级权限 v-permission 生效

**第一次能"打开浏览器看到自己做的管理后台"的时刻。**

---

## 一、技术栈与版本

| 项 | 版本 |
|---|---|
| Node.js | 24.x（.nvmrc） |
| 包管理器 | pnpm 9+ |
| Vue | 3.4+ |
| TypeScript | 5.x（strict） |
| Vite | 5.x |
| Element Plus | 2.x |
| UnoCSS | 最新稳定 |
| Pinia | 2.x |
| Vue Router | 4.x |
| @bxa/admin-shared | link:../binxinadmin-admin-shared（本地 file:） |

---

## 二、目录结构

```
binxinadmin-admin-web/
├── .nvmrc                          内容: 24
├── .gitignore                      含 .local-credentials.json / .env* / node_modules / dist / .vite / .DS_Store
├── .prettierrc
├── eslint.config.js
├── package.json
├── tsconfig.json
├── vite.config.ts
├── uno.config.ts                   引用 @bxa/admin-shared/uno.config
├── index.html                      标题: 平台管理 - BinxinAdmin
├── public/
│   └── favicon.svg                 蓝色版（参考 frontend-theming.md）
├── src/
│   ├── main.ts                     入口，install 共享包 + Pinia + Router + i18n
│   ├── App.vue
│   ├── env.d.ts
│   ├── api/                        平台后端 API 封装
│   │   ├── index.ts                createHttpClient 实例（baseURL=http://localhost:8080, appPrefix=platform）
│   │   ├── auth.ts                 login / logout / refresh / getUserInfo / getUserMenus
│   │   ├── system.ts               字典 / 配置 / 文件等系统 API（占位，按需扩展）
│   │   └── types.ts                平台特有类型
│   ├── stores/                     平台级 Pinia store（基于共享包工厂）
│   │   ├── index.ts                pinia 实例
│   │   ├── auth.ts                 useAuthStore (基于 createAuthStore)
│   │   ├── menu.ts                 useMenuStore
│   │   ├── tabs.ts                 useTabsStore
│   │   └── theme.ts                useThemeStore
│   ├── router/                     路由
│   │   ├── index.ts                Router 实例 + 基础路由（登录/404/布局壳）
│   │   ├── guards.ts               用 createAuthGuard + createPermissionGuard，传入平台配置
│   │   └── constants.ts            常量路由（不需要权限的）
│   ├── styles/
│   │   ├── theme.css               ★ 覆盖共享包 CSS 变量，定义平台后台品牌色（深蓝 #1d4ed8）
│   │   ├── index.css               全局样式入口
│   │   └── transitions.css         路由过渡动画
│   ├── layouts/                    项目级布局封装
│   │   ├── PlatformLayout.vue      包装共享包 BasicLayout，注入"平台"标识/Logo/顶栏插槽
│   │   └── components/
│   │       ├── PlatformLogo.vue    Logo 区（"BinxinAdmin · Platform"）
│   │       └── PlatformBadge.vue   顶栏右上 "平台" 蓝色徽章
│   ├── views/                      页面
│   │   ├── login/
│   │   │   ├── index.vue           登录页（标题"系统管理平台"，深色科技风背景）
│   │   │   └── components/
│   │   │       └── LoginForm.vue
│   │   ├── dashboard/
│   │   │   └── index.vue           首页 Dashboard（基础卡片+欢迎语，详细数据后续）
│   │   ├── profile/
│   │   │   └── index.vue           个人中心
│   │   ├── error/
│   │   │   ├── 404.vue
│   │   │   └── 403.vue
│   │   └── demo/                   ★ 用于验证共享包能力的演示页
│   │       ├── x-table-demo.vue    用 x-table 展示一个假数据列表（验证组件能用）
│   │       └── theme-demo.vue      主题切换演示
│   ├── composables/                组合式函数
│   │   ├── useDictResolver.ts      实现 DictResolver 契约，对接后端字典 API（M3 字典模块前先用 mock）
│   │   └── usePageTitle.ts         动态设置 document.title
│   ├── directives/                 项目级指令（如有）
│   ├── i18n/                       平台后台特有词条
│   │   └── platform-zh-CN.ts       合并到共享包基础 i18n
│   └── utils/                      项目级工具（如有）
└── README.md
```

---

## 三、关键实现要求

### 3.1 入口 main.ts

```typescript
import { createApp } from 'vue'
import App from './App.vue'
import router from './router'
import { pinia } from '@/stores'
import { setupSharedComponents } from '@bxa/admin-shared'  // 共享包提供的 install
import ElementPlus from 'element-plus'
import 'element-plus/dist/index.css'
import '@bxa/admin-shared/theme/variables.css'    // 共享包基础变量
import '@bxa/admin-shared/dist/admin-shared.css'  // 共享包样式
import '@/styles/theme.css'                        // 项目品牌色覆盖（必须在共享包样式之后）
import '@/styles/index.css'
import 'uno.css'

const app = createApp(App)
app.use(pinia).use(router).use(ElementPlus)
setupSharedComponents(app)   // 注册共享组件、指令等
app.mount('#app')
```

### 3.2 平台品牌色覆盖（styles/theme.css）

```css
:root {
  /* 覆盖共享包默认主题色，启用平台深蓝 */
  --bxa-primary: #1d4ed8;
  --bxa-primary-hover: #2563eb;
  --bxa-primary-active: #1e40af;

  /* 侧边栏深色科技风 */
  --bxa-bg-sidebar: #001529;
  --bxa-sidebar-text: rgba(255, 255, 255, 0.85);
  --bxa-sidebar-text-active: #ffffff;

  /* 登录页变量 */
  --bxa-login-bg-gradient-start: #0f172a;
  --bxa-login-bg-gradient-end:   #1e293b;
  --bxa-login-title-color:       #ffffff;
  --bxa-login-card-bg:           rgba(255,255,255,0.95);
}

[data-theme='dark'] {
  --bxa-primary: #3b82f6;
}
```

### 3.3 路由设计

**常量路由**（不需要权限，前端写死）：
- `/login` → views/login/index.vue
- `/404`、`/403`、`/redirect/:path(.*)`
- `/` → PlatformLayout（壳），children 动态生成

**动态路由**：从后端 `GET /platform/menu/user-menus` 获取菜单，用共享包 `buildDynamicRoutes()` 转换并 `router.addRoute('Layout', route)` 注入。

**路由守卫**：
- 用共享包 `createAuthGuard({ loginPath: '/login', whiteList: ['/login','/404','/403'] })`
- 用共享包 `createPermissionGuard({ unauthorizedPath: '/403' })`

**注意**：后端目前 `/platform/menu/user-menus` 接口未必有，**先在 src/api/auth.ts 做 mock 兜底**：调用失败时返回硬编码的菜单结构（含 Dashboard、演示页、个人中心），保证前端骨架能跑通。M2 用户管理时实现真实接口。

### 3.4 登录流程

1. 登录页提交 → 调 `POST /platform/auth/login` → 拿 access_token + refresh_token
2. authStore 存 token（共享包 token-manager 自动处理双层存储）
3. 调 `GET /platform/auth/me` → 存用户信息到 authStore
4. 调 `GET /platform/menu/user-menus`（失败则用 mock）→ 转动态路由 → addRoute
5. router.push('/') → 跳到 Dashboard

### 3.5 平台后台差异化（严格按 frontend-theming.md 实现）

| 元素 | 实现 |
|---|---|
| 主题色 | `#1d4ed8` 深蓝（styles/theme.css 覆盖 --bxa-primary） |
| 侧边栏 | 深色科技风 #001529 |
| Logo 文字 | `BinxinAdmin · Platform`（PlatformLogo 组件） |
| Tab 标题 | `平台管理 - BinxinAdmin`（index.html + 路由 meta） |
| Favicon | 蓝色版 favicon.svg（自绘，用纯 SVG 即可，不依赖图片） |
| 登录页标题 | `系统管理平台` |
| 登录页副标题 | `Platform Console` |
| 登录页背景 | 深色科技风（CSS 渐变 + SVG 几何，自绘） |
| 顶栏右上身份角标 | 蓝色徽章 `平台`（PlatformBadge 组件） |
| meta theme-color | `#1d4ed8` |

### 3.6 字典 Resolver 实现

实现 `composables/useDictResolver.ts`，对接后端字典接口。当前后端字典接口未必有，先做 mock：

```typescript
// useDictResolver.ts
const MOCK_DICTS: Record<string, DictItem[]> = {
  common_status: [
    { label: '正常', value: 1, type: 'success' },
    { label: '禁用', value: 0, type: 'danger' }
  ],
  // ...
}

export function useDictResolver() {
  return {
    async getDict(type: string) {
      // TODO: 接入真实后端 /platform/system/dict/items/:type
      return MOCK_DICTS[type] || []
    }
  }
}
```

在 main.ts 通过 provide 注入到全局，供共享包的 x-dict-tag / x-dict-select 使用。

### 3.7 响应式适配

按 frontend-theming.md 实现"平板可用 + 手机可看"：
- ≥1024px：完整布局
- 768-1023px：侧边栏自动折叠
- <768px：抽屉式侧边栏，表格横向滚动 + 提示

共享包的 BasicLayout 已经做了基础响应式，本项目只需在 App.vue 或入口处确认行为符合预期。

---

## 四、package.json 关键字段

```json
{
  "name": "binxinadmin-admin-web",
  "version": "0.1.0",
  "private": true,
  "type": "module",
  "engines": { "node": ">=24.0.0", "pnpm": ">=9.0.0" },
  "scripts": {
    "dev": "vite",
    "build": "vue-tsc --noEmit && vite build",
    "preview": "vite preview",
    "type-check": "vue-tsc --noEmit",
    "lint": "eslint . --max-warnings 0"
  },
  "dependencies": {
    "@bxa/admin-shared": "link:../binxinadmin-admin-shared",
    "vue": "^3.4.0",
    "vue-router": "^4.4.0",
    "pinia": "^2.2.0",
    "element-plus": "^2.8.0",
    "axios": "^1.7.0"
  },
  "devDependencies": {
    "@vitejs/plugin-vue": "^5.0.0",
    "typescript": "^5.5.0",
    "vue-tsc": "^2.0.0",
    "vite": "^5.4.0",
    "unocss": "^0.62.0"
  }
}
```

---

## 五、Vite 配置要点

- 端口固定 5173（避免和共享包 playground 的 5180 冲突）
- 开启 HMR
- alias: `@` → `src`
- proxy: `/platform` 代理到 `http://localhost:8080`（避免 CORS）
- include `@bxa/admin-shared` 到 optimizeDeps（让 Vite 正确处理 link 包）

---

## 六、开源资源合规（红线）

- 字体：思源黑体或系统 sans-serif，不打包字体文件
- 图标：用共享包 x-icon（基于 @iconify），不引入新图标库
- favicon：纯 SVG 自绘（蓝色几何形 + 字母 B 或盾形），不用商业图片
- 登录页背景：CSS 渐变 + 纯 SVG 自绘几何，不用网络图片或商业图

---

## 七、交付物 & 验证

### 交付物
- 完整源代码
- README.md（启动说明、依赖说明、和后端的对接说明）
- 0 type-check 错误、0 ESLint 错误

### 验证清单
1. `pnpm install`（含 `@bxa/admin-shared` link 包）成功
2. `pnpm dev` 启动到 http://localhost:5173
3. 访问 / → 自动跳 /login
4. 用 superadmin 账号密码登录成功（**playground 的安全规范继续遵守：登录表单不预填密码、密码用后清空**）
5. 登录后跳到 Dashboard，看到完整布局（蓝色主题、平台 Logo、平台徽章）
6. 侧边栏菜单点击切换页面正常
7. /demo/x-table 演示页 x-table 渲染正常
8. /demo/theme 演示页主题色切换、暗色切换、字号切换全部生效
9. 顶栏用户下拉 → 登出 → 跳回登录页，token 已清
10. 关闭浏览器重新打开，token 仍在（localStorage 持久化）；token 过期能自动刷新
11. 浏览器开发工具检查：无 console error、无 404 静态资源
12. 平板尺寸（768-1023px）：侧边栏自动折叠
13. 手机尺寸（<768px）：抽屉式侧边栏，演示表格横向滚动

---

## 八、安全/工程硬约束

1. **零硬编码密码**：登录表单不预填、不打印、不写日志
2. **.gitignore 完整**：`.local-credentials.json` / `.env*` / `node_modules` / `dist` / `.vite` / `.DS_Store` / `*.log`
3. **不打包敏感信息**：API 地址通过 `.env.development` / `.env.production` 配置
4. **不入仓库**：`.env.local` 等本地覆盖配置不入仓库
5. **不引入商业字体/图标/图片**

---

## 九、不在本任务范围

- 完整业务管理页面（租户管理、应用管理、用户管理等 → 属于阶段 1+ 业务模块）
- 后端字典/菜单接口的具体实现（前端先 mock）
- C 端 API、Laravel 后端对接
- 单元测试（后续阶段做）

---

## 十、与后端对接的注意点

当前 TP 后端已有的可用接口：
- `POST /platform/auth/login` （username + password → access_token + refresh_token + admin）
- `GET /platform/auth/me`（需 Bearer Token → 返回当前 admin 信息）
- `POST /platform/auth/logout`（需 Bearer Token → token 加入黑名单）
- `GET /platform/health`（健康检查）

**未实现但前端需要**：
- `GET /platform/menu/user-menus`（拉取当前用户菜单）→ 前端 mock 兜底
- `POST /platform/auth/refresh`（refresh_token → 新 access_token）→ 共享包拦截器已处理，后端如未实现先静默
- `GET /platform/system/dict/items/:type`（字典）→ 前端 mock 兜底

这些"后端缺失"的接口，前端用 mock 跑通骨架；待后续 M1/M2/M3 阶段后端补齐。

---

## 十一、提交规范

请先告诉我执行计划，确认后再开始。
做完先不要 commit，提供改动清单 + 验证日志（不含敏感信息），等我 review。
commit message 用中文（前缀 feat/fix/docs 等保留英文约定式关键字）。
示例：`feat: 初始化 admin-web 平台后台骨架，登录链路打通`
