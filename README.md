<p align="center">
  <a href="https://github.com/vuejs/core">
    <img width="100" src="https://vuejs.org/images/logo.png" alt="Vue logo" />
  </a>
</p>

<p align="center">
  <a href="https://github.com/iamzjt-front-end">
    <img src="https://img.shields.io/badge/Github-iamzjt--front--end-blue" alt="IamZJT" />
  </a>&emsp;
  <a href="https://github.com/vuejs/core">
    <img src="https://img.shields.io/badge/-Vue.js-%232c3e50?style=flat-square&logo=vuedotjs" alt="Vue">
  </a>&emsp;
  <a href="https://github.com/iamzjt-front-end">
    <img src="https://komarev.com/ghpvc/?username=iamzjt-front-end&label=++访客统计++&color=lightgrey" alt="访客统计" />
  </a>&emsp;
</p>

<h1 align="center">
  zjt-mini-vue3
</h1>

<h3 align="center">
  简体中文 | <a href='./README_EN.md'>English</a>
</h3>


## 介绍

🙋 你好！

此仓库为 `Vue3` 源码学习记录，以 `TDD` 为驱动，致力于用最少的代码实现 `happy path` ，来深入学习理解 `Vue3` 源码。
 
手写完成 `Vue3` 的三大模块，`reactivity`（响应式）、`runtime`（运行时）及`compiler`（编译），实现最简 `Vue3` 模型！

为什么要读源码？

> ✨ 加强对编程语言的理解

> 🔥 提升编程能力

> 🚀 提高调试能力

> ⚡️ 提高代码质量


## 系列文章

下面为同步更新的 <a href="https://juejin.cn/column/7168612212133593095"><img src="https://img.shields.io/badge/juejin-掘金专栏-487DF8"></a> 系列文章：

欢迎各位大佬点点`star`⭐、点点赞👍啊！😂

#### 一、reactivity篇

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

[📑 11_实现shallowReadonly和isProxy功能](https://juejin.cn/post/7180887790899920956)

[📑 12_实现ref功能](https://juejin.cn/post/7181710097863671864)

[📑 13_实现isRef和unRef功能](https://juejin.cn/post/7182379390183931960)

[📑 14_实现proxyRefs功能](https://juejin.cn/post/7185443608827265061)

[📑 15_实现computed计算属性](https://juejin.cn/post/7189847454152392760)

[📑 16_实现相对完善的reactive](https://juejin.cn/post/7194275202212036667)

[📑 17_实现相对完善的effect](https://juejin.cn/post/7196690584286462008)


#### 二、runtime-core篇

[📃 01_实现初始化component流程](./docs/md/runtime/01_实现初始化component主流程.md)

[📃 02_实现element渲染流程](./docs/md/runtime/02_实现element渲染流程.md)

[📃 03_实现组件通信和插槽](./docs/md/runtime/03_实现组件通信和插槽.md)

[📃 04_实现组件更新流程](./docs/md/runtime/04_实现组件更新流程.md)

[📃 05_实现diff算法](./docs/md/runtime/05_实现diff算法.md)

[📃 06_实现nextTick和运行时API](./docs/md/runtime/06_实现nextTick和运行时API.md)

[📃 07_实现自定义渲染器](./docs/md/runtime/07_实现自定义渲染器.md)


#### 三、compiler篇

[📰 01_编译模块概述](./docs/md/compiler/01_编译模块概述.md)

[📰 00_parse的实现原理&有限状态机](./docs/md/compiler/00_parse的实现原理&有限状态机.md)

[📰 02_实现parse解析流程](./docs/md/compiler/02_实现parse解析流程.md)

[📰 03_实现transform转换流程](./docs/md/compiler/03_实现transform转换流程.md)

[📰 04_实现codegen生成render函数](./docs/md/compiler/04_实现codegen生成render函数.md)

[📰 05_实现compiler与runtime联动](./docs/md/compiler/05_实现compiler与runtime联动.md)

✍️ 以上文章对应仓库当前的最简实现，后续可以继续补充更多语法和边界情况。


## 目前已实现功能点

#### reactivity

- [x] 实现 reactive
- [x] 实现 ref
- [x] 实现 readonly
- [x] 实现 嵌套reactive和readonly
- [x] 实现 computed
- [x] 实现 effect 并支持分支切换、嵌套及++情况下的依赖收集
- [x] 实现 track -> 依赖收集
- [x] 实现 trigger -> 触发依赖
- [x] 实现 effect.scheduler 调度器
- [x] 实现 基于runner、effect.stop 和 onStop的响应式开启关闭控制
- [x] 实现 isReactive和isReadonly
- [x] 实现 isProxy
- [x] 实现 isRef和unRef
- [x] 实现 shallowReadonly
- [x] 实现 proxyRefs

#### runtime-core

- [x] 支持 component 类型
- [x] 支持 element 类型
- [x] 支持 Text 类型节点
- [x] 初始化 props
- [x] setup 可获取 props 和 context
- [x] 支持 $el、$slots等api
- [x] 支持 component emit
- [x] 支持 单节点、多节点及具名slots
- [x] 可以在 render 函数中获取 setup 返回的对象
- [x] 支持 proxy
- [x] 支持 getCurrentInstance
- [x] 支持 provide/inject
- [x] 支持 nextTick 和异步更新队列
- [x] 支持 element 更新
- [x] 支持 keyed children 的双端对比和节点移动
- [x] 支持自定义渲染器

#### compiler-core
- [x] 解析插值
- [x] 解析 element
- [x] 解析 text
- [x] 支持 transform 插件
- [x] 支持 codegen 生成 render 函数
- [x] 支持 template 编译为 render 函数

#### runtime-dom
- [x] 支持 custom renderer

#### runtime-test
- [x] 支持测试 runtime-core 的逻辑

#### infrastructure
- [x] support monorepo with pnpm


## 勘误

如果发现错误，可以在相应的 issues 进行勘误或者 <a href="https://iamzjt-1256754140.cos.ap-nanjing.myqcloud.com/images/IamZJT-WeChat.jpg">微信：IamZJT_</a> 联系我。

如果喜欢或者有所启发，欢迎 star，对作者也是一种鼓励。
当然也可以添加我的微信，一起交流成长。


## 许可证

所有内容均采用 [MIT](https://spdx.org/licenses/MIT) 进行许可。
