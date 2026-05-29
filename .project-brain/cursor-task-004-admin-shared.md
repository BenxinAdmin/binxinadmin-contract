# Cursor 任务书 #004：admin-shared 前端共享包

> 用途：建立前端共享代码库，被 admin-web（平台后台）和 tenant-web（租户后台）共同依赖。
> 仓库：binxinadmin-admin-shared
> 依赖方式：本地 file: 协议（pnpm 支持）
> 配套参考：00-CHARTER.md、03-CONVENTIONS.md、db-design-v1.md
> 前置：task-001/002 TP后端已完成，三个 health 接口可访问

---

## 角色设定（粘贴给 Cursor）

```
你是资深前端架构师，精通 Vue 3 + TypeScript + Vite + Element Plus + pnpm，
擅长设计企业级中后台前端组件库和工具库。
你正在为 BinxinAdmin 多租户开源后台构建前端共享包，
这个包会被两个独立的前端项目共同依赖（admin-web 平台后台 + tenant-web 租户后台）。
严守开源合规：字体/图标/插画只用开源资源，详见 03-CONVENTIONS.md。
代码遵循 ESLint + Prettier，TypeScript strict 模式。
```

---

## 总目标

建立 binxinadmin-admin-shared 仓库，作为一个独立的 npm 包（不发布，本地 file: 引用），提供两个前端项目共用的：
1. HTTP 请求封装（Axios + 拦截器 + Token 刷新 + 错误处理）
2. 状态管理基座（Pinia store 工厂）
3. 通用业务组件（CRUD 表格、查询表单、详情抽屉、上传组件等）
4. 权限指令（v-permission）
5. 布局组件（侧边栏、顶栏、面包屑、标签页、用户菜单）
6. 主题系统（CSS 变量 + UnoCSS 配置，支持主题色/暗色模式/字号切换）
7. 通用工具函数（日期、格式化、加密、深拷贝、防抖节流等）
8. TypeScript 类型定义（API 响应、菜单、用户等）
9. i18n 基座（中英文，初期只填中文）

---

## 一、技术栈与版本

| 项 | 版本 |
|---|---|
| Node.js | 24.x |
| 包管理器 | pnpm 9+ |
| Vue | 3.4+ |
| TypeScript | 5.x（strict 模式） |
| Vite | 5.x |
| Element Plus | 2.x |
| 原子化 CSS | UnoCSS |
| 状态管理 | Pinia |
| 路由 | Vue Router 4 |
| HTTP | Axios |
| 图标 | @iconify/vue + Element Plus Icons |
| 工具库 | @vueuse/core, lodash-es, dayjs |

构建产物：ESM + Type Declarations（.d.ts），供两个前端项目直接 import。

---

## 二、目录结构

```
binxinadmin-admin-shared/
├── package.json              name: "@bxa/admin-shared", version: "0.1.0", type: "module"
├── tsconfig.json             strict 模式
├── tsup.config.ts            打包配置（用 tsup 打包，输出 ESM + d.ts）
├── .nvmrc                    内容 24
├── .gitignore
├── .eslintrc.cjs
├── .prettierrc
├── README.md                 使用说明
├── src/
│   ├── index.ts              统一导出入口
│   ├── http/                 HTTP 请求层
│   │   ├── request.ts        Axios 实例 + 拦截器
│   │   ├── token-manager.ts  Token 存取与刷新
│   │   ├── error-handler.ts  错误处理
│   │   └── types.ts          请求/响应类型
│   ├── stores/               Pinia store 工厂
│   │   ├── auth.ts           认证 store 工厂（返回可定制的 store）
│   │   ├── menu.ts           菜单 store 工厂
│   │   ├── tabs.ts           多标签页 store
│   │   └── theme.ts          主题 store
│   ├── components/           通用业务组件
│   │   ├── x-table/          列表页 CRUD 表格（核心）
│   │   ├── x-search/         查询表单（联动 x-table）
│   │   ├── x-form/           表单（基于 Element Plus 二次封装）
│   │   ├── x-drawer/         详情/编辑抽屉
│   │   ├── x-upload/         文件上传（对接后端文件接口）
│   │   ├── x-icon/           图标组件（@iconify 包装）
│   │   ├── x-dict-tag/       字典标签
│   │   └── x-dict-select/    字典下拉
│   ├── directives/           指令
│   │   ├── permission.ts     v-permission 按钮级权限
│   │   └── debounce.ts       v-debounce
│   ├── layouts/              布局组件
│   │   ├── BasicLayout.vue   主布局（侧边栏 + 顶栏 + 内容区）
│   │   ├── BlankLayout.vue   空白布局（登录页等用）
│   │   ├── components/
│   │   │   ├── Sidebar.vue   侧边栏（支持折叠、多级菜单）
│   │   │   ├── Header.vue    顶栏（面包屑、用户菜单、主题切换、登出）
│   │   │   ├── TabsBar.vue   多标签页栏
│   │   │   ├── Breadcrumb.vue
│   │   │   └── UserDropdown.vue
│   ├── theme/                主题系统
│   │   ├── variables.css     CSS 变量定义（主题色、字号、间距）
│   │   ├── dark.css          暗色模式覆盖
│   │   ├── uno.config.ts     UnoCSS 配置（共享包提供基础配置）
│   │   └── tokens.ts         主题 token TS 定义
│   ├── utils/                工具函数
│   │   ├── date.ts           日期处理（dayjs 封装）
│   │   ├── format.ts         数字/金额/手机号等格式化
│   │   ├── crypto.ts         前端加密辅助（AES，对接后端加密接口）
│   │   ├── storage.ts        localStorage/sessionStorage 封装
│   │   ├── tree.ts           树形数据处理（菜单/部门常用）
│   │   ├── download.ts       文件下载
│   │   └── validate.ts       常用校验
│   ├── i18n/                 国际化
│   │   ├── index.ts          i18n 实例
│   │   ├── locales/zh-CN.ts  中文（基础词条）
│   │   └── locales/en-US.ts  英文（仅占位）
│   ├── types/                TS 类型定义
│   │   ├── api.ts            API 响应类型 ApiResponse<T> / Pagination
│   │   ├── menu.ts           菜单类型
│   │   ├── user.ts           用户/角色类型
│   │   └── common.ts         通用类型
│   └── router/               路由相关
│       ├── guards.ts         路由守卫工厂（auth/permission）
│       └── dynamic-routes.ts 动态路由生成（基于菜单树）
└── playground/               最小演示（可选，用于开发时调试）
    └── ...
```

---

## 三、核心模块详细要求

### 3.1 HTTP 请求封装（src/http/）

**Axios 实例工厂**：
```typescript
createHttpClient(options: {
  baseURL: string                    // 由调用方传入（如 http://localhost:8080）
  appPrefix: 'platform' | 'admin' | 'api'  // 三端守卫前缀
  tokenStorageKey?: string           // 默认 'bxa_token'
  refreshTokenStorageKey?: string    // 默认 'bxa_refresh_token'
  onUnauthorized?: () => void        // 401 回调（跳登录）
  onPermissionDenied?: () => void    // 权限不足回调
})
```

**请求拦截器**：
- 自动注入 Authorization: Bearer {token}
- 自动注入 X-Request-Id（前端生成 uuid）
- 自动注入 X-Tenant-Code（如有，租户后台用）

**响应拦截器**：
- 解析后端统一响应 {code, message, data, timestamp, request_id}
- code=0 → 返回 data
- code=20003 (Token过期) → 尝试用 refresh_token 刷新，刷新成功重发原请求
- code=20001/20002 → 触发 onUnauthorized（跳登录）
- code=20004 → 触发 onPermissionDenied（提示）
- code=10003 → 提示限流，message 给用户看
- code=5xxxx → 提示系统错误
- 其他业务错误 → ElMessage.error(message)，并 reject 错误对象供调用方处理

**Token 管理**：
- 双层存储：Pinia 内存 + localStorage（防刷新丢失）
- 刷新机制：用 refresh_token 调 /xxx/auth/refresh，失败则清空跳登录
- 防并发刷新：多个请求同时 401 时，只发起一次刷新请求，其他请求挂起等待

### 3.2 通用 CRUD 表格（x-table）★ 核心组件

这是整个共享包最值钱的组件，必须做得通用、强大、易用。

**配置式 API**：
```vue
<x-table
  :api="userApi"              // 查询函数 (params) => Promise<{list, pagination}>
  :columns="columns"          // 列配置
  :search-fields="searchFields"  // 查询字段配置
  :toolbar-buttons="toolbar"  // 工具栏按钮（新增/批量删/导出等）
  :row-actions="rowActions"   // 行操作按钮
  :selection="true"           // 是否多选
  :default-sort="{ prop: 'created_at', order: 'desc' }"
  @row-click="onRowClick"
/>
```

**列配置示例**：
```typescript
const columns: XTableColumn[] = [
  { prop: 'id', label: 'ID', width: 80 },
  { prop: 'username', label: '用户名', minWidth: 120 },
  { prop: 'status', label: '状态', dictType: 'common_status' },  // 自动字典展示
  { prop: 'created_at', label: '创建时间', formatter: 'datetime' },
  { type: 'action', label: '操作', width: 200, slot: 'row-actions' }
]
```

**查询字段示例**（联动 x-search）：
```typescript
const searchFields: XSearchField[] = [
  { prop: 'keyword', label: '关键字', type: 'input', placeholder: '用户名/手机号' },
  { prop: 'status', label: '状态', type: 'dict-select', dictType: 'common_status' },
  { prop: 'created_at', label: '创建时间', type: 'date-range' }
]
```

**功能要求**：
- 分页（对接后端 {list, pagination:{page, page_size, total, total_pages}}）
- 排序（点击表头，传 sort/order 给后端）
- 查询表单折叠（超过 3 个字段自动折叠"更多"）
- 列设置（用户可选择显示/隐藏列、列宽度、列顺序，存 localStorage）
- 刷新、重置、密度切换
- 多选 + 批量操作
- 行操作按钮（按钮权限标识 v-permission 自动接入）
- 字典自动渲染（dictType 自动调字典 API 转 label）
- 响应式：PC 完整、平板缩列、手机改卡片或横向滚动
- 空数据/加载中/错误三态

### 3.3 权限指令 v-permission

```vue
<el-button v-permission="['user:add']">新增</el-button>
<el-button v-permission="['user:edit', 'user:delete']">编辑</el-button>  <!-- 任一即可 -->
<el-button v-permission.all="['user:edit', 'user:delete']">操作</el-button>  <!-- 全部要 -->
```

- 当前用户权限标识从 menu store 获取
- 没权限直接 v-if 移除元素（不只是 disabled）

### 3.4 主题系统

**CSS 变量驱动**：
```css
:root {
  --bxa-primary: #1677ff;       /* 主题色，由两个前端项目分别覆盖 */
  --bxa-success: #52c41a;
  --bxa-warning: #faad14;
  --bxa-danger: #ff4d4f;
  --bxa-bg-base: #ffffff;
  --bxa-bg-sidebar: #001529;
  --bxa-text-primary: #333;
  --bxa-text-secondary: #666;
  --bxa-border: #e5e5e5;
  --bxa-radius: 6px;
  --bxa-sidebar-width: 220px;
  --bxa-header-height: 56px;
  --bxa-font-base: 14px;
  ...
}

[data-theme="dark"] {
  --bxa-bg-base: #141414;
  --bxa-text-primary: #fff;
  ...
}
```

**两个项目通过覆盖 --bxa-primary 等变量实现差异化品牌色**。

**字号切换**：提供 small/default/large 三档，改 --bxa-font-base，全局缩放。

### 3.5 布局组件

**BasicLayout**：
- 左侧固定宽度侧边栏（可折叠/响应式抽屉）
- 顶部固定高度顶栏
- 顶栏下方多标签页栏（可关闭）
- 内容区滚动
- 暴露插槽供项目自定义 Logo、用户菜单等

**Sidebar**：
- 支持菜单数据从 props 传入（不依赖具体 store）
- 多级菜单（树形结构）
- 折叠模式 hover 弹出
- 当前激活项高亮（基于路由）
- 图标用 @iconify（开源图标，从 menu 数据的 icon 字段读取，如 "mdi:account"）

**Header**：
- 折叠按钮、面包屑、搜索（占位）、主题切换、字号切换、全屏、消息（占位）、用户头像菜单
- 用户菜单：个人中心、修改密码、退出登录

---

## 四、开源资源合规（红线）

- **字体**：仅引用思源黑体或系统默认 sans-serif，不打包字体文件；如需自定义字体，用阿里巴巴普惠体、HarmonyOS Sans
- **图标**：用 @iconify/vue + 开源图标集（mdi、tabler、carbon 等），避免引用 Font Awesome Pro
- **图片**：登录页背景等用 unDraw / Storyset 免费授权图，或纯 CSS/SVG 自制
- 在 README.md 末尾列出所有引用资源的来源和授权

---

## 五、TypeScript 类型规范

所有公开 API 必须有完整类型定义。例如：

```typescript
export interface ApiResponse<T = any> {
  code: number
  message: string
  data: T
  timestamp: number
  request_id: string
}

export interface PaginationData<T> {
  list: T[]
  pagination: {
    page: number
    page_size: number
    total: number
    total_pages: number
  }
}

export interface MenuItem {
  id: number
  parent_id: number
  name: string
  type: 1 | 2 | 3        // 1目录 2菜单 3按钮
  path: string
  component: string
  permission: string
  icon: string
  sort: number
  children?: MenuItem[]
}
```

---

## 六、package.json 关键字段

```json
{
  "name": "@bxa/admin-shared",
  "version": "0.1.0",
  "type": "module",
  "main": "./dist/index.js",
  "module": "./dist/index.js",
  "types": "./dist/index.d.ts",
  "exports": {
    ".": {
      "import": "./dist/index.js",
      "types": "./dist/index.d.ts"
    },
    "./theme/variables.css": "./src/theme/variables.css",
    "./styles/*": "./dist/styles/*"
  },
  "files": ["dist", "src/theme"],
  "scripts": {
    "build": "tsup",
    "dev": "tsup --watch",
    "type-check": "vue-tsc --noEmit"
  },
  "peerDependencies": {
    "vue": "^3.4.0",
    "vue-router": "^4.0.0",
    "pinia": "^2.0.0",
    "element-plus": "^2.0.0"
  }
}
```

> peerDependencies：vue/pinia/element-plus 这些大件由调用方提供，共享包不重复打包。

---

## 七、构建与开发

- 用 **tsup**（基于 esbuild，比 rollup 快很多）打包
- 输出 ESM 格式 + .d.ts 类型声明
- 开发模式 `pnpm dev` watch 变更
- 两个前端项目用 `pnpm add link:../binxinadmin-admin-shared` 或 `"@bxa/admin-shared": "link:../binxinadmin-admin-shared"` 引用

---

## 八、交付物 & 验证

### 交付物
- 完整源代码（src/ 下所有模块）
- 类型定义完整、ESLint 0 错误、TypeScript 0 错误
- README.md：包含安装、引用、各模块使用示例
- LICENSE: Apache 2.0
- CREDITS.md：列出所有开源资源来源和授权

### 验证（在 playground/ 里写最小示例验证）
1. `pnpm install` 成功
2. `pnpm build` 成功输出 dist/
3. `pnpm dev` watch 模式正常
4. playground 里能跑通：
   - createHttpClient 创建实例
   - x-table 渲染一张假数据表格
   - v-permission 指令生效
   - BasicLayout 布局展示正常
   - 主题色切换通过修改 CSS 变量生效

---

## 九、注意事项

- 这是**库**不是应用，所有"页面级"逻辑都不属于本仓库
- 组件设计要**纯净、无副作用**，不假设调用方的具体业务
- 不要硬编码 API 地址、token key 等，全部通过工厂函数参数传入
- 不要硬编码字典数据、菜单数据
- TypeScript strict 模式严格执行，不允许 any 蒙混
- 中文注释，所有公开 API 都要 JSDoc

---

## 十、不在本任务范围（防止 Cursor 扩展）

- 登录页、用户管理页等具体业务页面 → 属于 admin-web / tenant-web
- 路由定义 → 属于具体项目
- 具体 API 请求函数（如 userApi、roleApi）→ 属于具体项目
- 项目级 store 实例 → 属于具体项目（共享包只提供 store 工厂）

请先告诉我执行计划（含每个核心模块的实现思路），确认后再开始。
做完先不要 commit，提供完整改动清单 + playground 验证截图（或日志），等我 review。
