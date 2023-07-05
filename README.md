<p align="center">
  <a href="https://github.com/iamzjt-front-end/zjt-mini-vue3">
    <img width="96" src="https://vuejs.org/images/logo.png" alt="Vue logo" />
  </a>
</p>

<h1 align="center">zjt-mini-vue3</h1>

<p align="center">
  <strong>从源码出发，手写一个最简 Vue 3 模型</strong>
</p>

<p align="center">
  reactivity · runtime-core · compiler-core
</p>

<p align="center">
  <a href="https://github.com/iamzjt-front-end/zjt-mini-vue3">
    <img src="https://img.shields.io/github/stars/iamzjt-front-end/zjt-mini-vue3?style=flat-square&logo=github" alt="GitHub stars" />
  </a>&nbsp;
  <a href="https://github.com/iamzjt-front-end/zjt-mini-vue3/blob/main/LICENSE">
    <img src="https://img.shields.io/github/license/iamzjt-front-end/zjt-mini-vue3?style=flat-square" alt="License" />
  </a>&nbsp;
  <a href="https://pnpm.io/workspaces">
    <img src="https://img.shields.io/badge/pnpm-workspace-F69220?style=flat-square&logo=pnpm&logoColor=white" alt="pnpm workspace" />
  </a>&nbsp;
  <a href="https://vuejs.org/">
    <img src="https://img.shields.io/badge/Vue-3.x-42b883?style=flat-square&logo=vuedotjs&logoColor=white" alt="Vue 3" />
  </a>
</p>

<p align="center">
  简体中文 · <a href="./README_EN.md">English</a>
</p>

<p align="center">
  <a href="#-快速开始">快速开始</a> ·
  <a href="#-整体链路">整体链路</a> ·
  <a href="#-系列文章">系列文章</a> ·
  <a href="#-实现清单">实现清单</a>
</p>

---

## ✨ 项目简介

这是一个 Vue 3 源码学习仓库，以 TDD 为驱动，用尽可能少的代码完成可运行的 happy path：

- **reactivity**：响应式系统，理解 effect、依赖收集、触发更新；
- **runtime-core**：运行时核心，理解 vnode、组件、patch 和 scheduler；
- **compiler-core**：编译核心，理解 template 如何变成 render 函数。

整个项目采用 monorepo 组织，并通过 pnpm 管理多个 package。这里的目标不是复刻完整 Vue，而是沿着真实代码和提交记录，把一条从数据到页面的链路走通。

> 如果你想真正理解 Vue，而不是只记住几个 API，可以从这套 mini-vue 开始。

## 🧭 学习路线

```text
reactivity
    ↓
runtime-core
    ↓
runtime-dom
    ↓
compiler-core
    ↓
template → render → vnode → DOM
```

推荐按照下面的顺序阅读：

1. 先理解 reactivity，弄清楚“数据变化为什么能触发更新”；
2. 再阅读 runtime-core，弄清楚“更新以后如何生成和对比 vnode”；
3. 最后阅读 compiler-core，弄清楚“template 如何变成 render 函数”。

## 🚀 快速开始

```shell
git clone https://github.com/iamzjt-front-end/zjt-mini-vue3.git
cd zjt-mini-vue3

pnpm install
pnpm test -- --run
pnpm build
```

常用目录：

```text
packages/
├── reactivity/       # 响应式系统
├── runtime-core/     # 运行时核心
├── runtime-dom/      # 浏览器平台实现
├── compiler-core/    # 模板解析、转换和代码生成
├── shared/           # 公共工具
└── vue/              # 对外入口和示例

docs/
├── md/               # 系列文章
├── imges/            # 文章配图
└── xmind/            # 流程图和思维导图
```

## 🧩 模块地图

| 模块 | 目录 | 主要职责 |
| --- | --- | --- |
| reactivity | `packages/reactivity` | reactive、ref、effect、computed |
| runtime-core | `packages/runtime-core` | vnode、组件、patch、diff、scheduler |
| runtime-dom | `packages/runtime-dom` | DOM 创建、属性、事件和平台适配 |
| compiler-core | `packages/compiler-core` | parse、transform、codegen |
| shared | `packages/shared` | ShapeFlags、工具函数和公共类型判断 |
| vue | `packages/vue` | 组合 compiler 与 runtime，并提供示例 |

## 🔁 整体链路

### 响应式更新链路

```text
reactive / ref
      ↓
effect 收集依赖
      ↓
数据发生变化
      ↓
scheduler 合并更新任务
      ↓
组件重新执行 render
      ↓
patch 新旧 vnode
      ↓
runtime-dom 更新真实 DOM
```

### 模板编译链路

```text
template 字符串
      ↓
baseParse
      ↓
AST
      ↓
transform
      ↓
codegenNode + helpers
      ↓
generate
      ↓
render 函数
```

## 📚 系列文章

下面是与仓库同步维护的源码学习文章。reactivity 篇保留掘金专栏链接，runtime 和 compiler 篇直接链接到仓库内文档。

### 一、reactivity 篇

<details open>
<summary>17 篇主线文章 + 1 篇补充笔记</summary>

[📑 01_Vue3源码的介绍](https://juejin.cn/post/7168664872547254285)

[📑 02_TDD开发环境搭建](https://juejin.cn/post/7169351734051995678)

[📑 03_01_实现effect&reactive&依赖收集&触发依赖](https://juejin.cn/post/7170480677614256158)

[📑 03_02_理解Proxy和Reflect](https://juejin.cn/post/7171655019425431583)

[📑 04_实现effect返回runner](https://juejin.cn/post/7172683900282634254)

[📑 05_实现effect的scheduler功能](https://juejin.cn/post/7173498493334454285)

[📑 06_实现effect的stop和onStop功能](https://juejin.cn/post/7174161779264585741)

[📑 07_实现readonly功能](https://juejin.cn/post/7175279305327378490)

[📑 08_实现isReactive和isReadonly](https://juejin.cn/post/7176086344815837242)

[📑 09_优化stop功能](https://juejin.cn/post/7179866542857781285)

[📑 10_实现reactive和readonly的嵌套对象转换功能](https://juejin.cn/post/7179867852877332517)

[📑 11_实现shallowReadonly和isProxy](https://juejin.cn/post/7180887790899920956)

[📑 12_实现ref功能](https://juejin.cn/post/7181710097863671864)

[📑 13_实现isRef和unRef功能](https://juejin.cn/post/7182379390183931960)

[📑 14_实现proxyRefs功能](https://juejin.cn/post/7185443608827265061)

[📑 15_实现computed计算属性](https://juejin.cn/post/7189847454152392760)

[📑 16_实现相对完善的reactive](https://juejin.cn/post/7194275202212036667)

[📑 17_实现相对完善的effect](https://juejin.cn/post/7196690584286462008)

[📑 18_一些未曾注意到的细节](./docs/md/reactivity/18_一些未曾注意到的细节.md)

</details>

### 二、runtime-core 篇

<details open>
<summary>从组件初始化到跨平台渲染</summary>

[📃 01_实现初始化component流程](./docs/md/runtime/01_实现初始化component主流程.md)

[📃 02_实现element渲染流程](./docs/md/runtime/02_实现element渲染流程.md)

[📃 03_实现组件通信和插槽](./docs/md/runtime/03_实现组件通信和插槽.md)

[📃 04_实现组件更新流程](./docs/md/runtime/04_实现组件更新流程.md)

[📃 05_实现diff算法](./docs/md/runtime/05_实现diff算法.md)

[📃 06_实现nextTick和运行时API](./docs/md/runtime/06_实现nextTick和运行时API.md)

[📃 07_实现自定义渲染器](./docs/md/runtime/07_实现自定义渲染器.md)

</details>

### 三、compiler 篇

<details open>
<summary>从 template 到 render 函数</summary>

[📰 00_parse的实现原理&有限状态机](./docs/md/compiler/00_parse的实现原理&有限状态机.md)

[📰 01_编译模块概述](./docs/md/compiler/01_编译模块概述.md)

[📰 02_实现parse解析流程](./docs/md/compiler/02_实现parse解析流程.md)

[📰 03_实现transform转换流程](./docs/md/compiler/03_实现transform转换流程.md)

[📰 04_实现codegen生成render函数](./docs/md/compiler/04_实现codegen生成render函数.md)

[📰 05_实现compiler与runtime联动](./docs/md/compiler/05_实现compiler与runtime联动.md)

</details>

## ✅ 实现清单

| 模块 | 已实现能力 |
| --- | --- |
| **reactivity** | reactive、readonly、shallowReadonly、ref、proxyRefs、computed、effect、scheduler、stop、onStop、依赖收集和触发 |
| **runtime-core** | component、element、Text、Fragment、props、emit、slots、proxy、getCurrentInstance、provide/inject、element 更新、keyed children diff、nextTick |
| **runtime-dom** | DOM host 操作、自定义 renderer 接入 |
| **compiler-core** | interpolation、element、text 解析，transform 插件，codegen，template → render |
| **infrastructure** | monorepo、pnpm workspace、Vitest 测试环境 |

## 🧪 示例与验证

示例位于 `packages/vue/examples`，可以重点阅读：

- `hello-world`：最基础的组件和 render；
- `componentUpdate`：组件 props 和响应式更新；
- `componentSlot`：默认、具名和作用域插槽；
- `patchChildren`：不同子节点形态和 keyed diff；
- `nextTicker`：更新队列和 nextTick；
- `apiInject`：provide / inject；
- `customRenderer`：接入其它平台的 renderer；
- `compiler-base`：template 编译为 render。

测试覆盖当前核心 happy path：

```text
Test Files  11 passed
Tests       48 passed | 1 skipped
```

## ⚠️ 当前实现边界

这是一个源码学习项目，当前实现刻意聚焦在 happy path，暂未覆盖：

- 完整的 HTML 属性、指令、注释和自闭合标签解析；
- `v-if`、`v-for` 等编译转换；
- 完整的事件、class、style 更新策略；
- Vue 生产版中的静态提升、patch flag 和缓存优化；
- 完整的错误恢复、组件卸载和平台差异处理。

这些边界会保留在文章和代码中，方便继续沿着真实的实现顺序迭代。

## 📮 勘误与交流

如果发现错误，欢迎在 [Issues](https://github.com/iamzjt-front-end/zjt-mini-vue3/issues) 中指出。

如果这个项目对你有帮助，欢迎点个 `star` ⭐，也欢迎一起交流源码学习。

## 📄 许可证

本项目采用 [MIT](https://spdx.org/licenses/MIT) 许可证。
