# BinxinAdmin 前端双后台差异化方案

> 决策 D-027：轻度差异化路径。
> 同一套布局、同一套组件、同一套交互；只在视觉品牌层做差异。
> 共享包（admin-shared）保持纯净，差异在 admin-web/tenant-web 项目级实现。

---

## 一、差异化具体项（task-005/006 必须实现）

| 元素 | admin-web 平台后台 | tenant-web 租户后台 | 实现方式 |
|---|---|---|---|
| 主题色（--bxa-primary） | `#1d4ed8`（深蓝，专业平台感） | `#059669`（翠绿，活力商户感） | 项目 CSS 覆盖 |
| 侧边栏背景 | 深色科技风（#001529） | 浅色清新风（#fff，带左侧绿色细条） | CSS 变量 |
| Logo 文字（顶栏） | `BinxinAdmin · Platform` | `BinxinAdmin · Tenant` | 项目 layout 插槽 |
| 浏览器 Tab 标题 | `平台管理 - BinxinAdmin` | `商户管理 - BinxinAdmin` | index.html + 路由 meta |
| Favicon | 蓝色版 favicon | 绿色版 favicon | public/favicon.ico |
| 登录页标题 | `系统管理平台` | `商户管理后台` | 登录页 props |
| 登录页副标题 | `Platform Console` | `Tenant Console` | 登录页 props |
| 登录页背景 | 深色科技风 SVG（自绘几何） | 浅色清新风 SVG（自绘曲线） | 登录页背景 |
| 顶栏右上身份角标 | 蓝色徽章 `平台` | 绿色徽章 `租户` | UserDropdown 上方插槽 |
| 浏览器主题色（meta） | `#1d4ed8` | `#059669` | index.html theme-color meta |

---

## 二、保持完全一致的部分

- 整体布局结构（左侧栏 + 顶栏 + 多标签页 + 内容区）
- 所有共享组件（x-table、x-search、x-form、x-drawer、x-upload 等）
- 交互模式（菜单折叠、多标签操作、暗色切换、字号切换）
- 字体（思源黑体 / 系统默认 sans-serif）
- 字号、间距、圆角等基础 token

---

## 三、技术实现指引

1. admin-shared 的 theme/variables.css 定义所有 CSS 变量（含主题色默认值）
2. admin-web 在自己的 src/styles/theme.css 里覆盖品牌相关变量
3. tenant-web 同上
4. 登录页虽然结构一样，但通过 props 接收"标题/副标题/背景图"等品牌项
5. 顶栏 logo 区和身份角标用插槽，由项目级 layout 注入

---

## 四、暗色模式

两个项目都支持暗色模式。
暗色模式下品牌主色保持不变（蓝/绿），只是背景文字反色。
