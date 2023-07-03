# 05_实现compiler与runtime联动

### 一、前面的代码还缺最后一环

到目前为止，`compiler-core`已经可以完成：

```text
template
  -> parse
  -> transform
  -> codegen
  -> render函数代码
```

`runtime-core`也已经可以完成：

```text
render函数
  -> vnode
  -> patch
  -> 真实节点
```

真正使用`template`时，还需要把两条链路接起来：

```text
template
  -> 编译成render函数
  -> 组件初始化时执行render
  -> 进入runtime-core
```

### 二、baseCompile统一编译入口

`compiler-core/src/compile.ts`把三个阶段组合起来：

```ts
import { baseParse } from './parse';
import { transform } from './transform';
import { transformsExpression } from './transforms/transformsExpression';
import { transformElement } from './transforms/transformElement';
import { transformText } from './transforms/transformText';
import { generate } from './codegen';

export function baseCompile(template: string) {
  const ast = baseParse(template);

  transform(ast, {
    nodeTransforms: [
      transformsExpression,
      transformElement,
      transformText
    ]
  });

  return generate(ast);
}
```

对外只需要调用`baseCompile`，不需要手动依次调用`parse`、`transform`和`generate`。

### 三、runtimeCompiler注册

`packages/vue/src/index.ts`是mini-vue的总入口。它同时引入编译器和运行时：

```ts
import { baseCompile } from '@zjt-mini-vue3/compiler-core';
import * as runtimeDom from '@zjt-mini-vue3/runtime-dom';
import { registerRuntimeCompiler } from '@zjt-mini-vue3/runtime-dom';
```

接着定义`compilerToFunction`：

```ts
function compilerToFunction(template) {
  const { code } = baseCompile(template);
  const render = new Function('Vue', code)(runtimeDom);
  return render;
}
```

这个函数分成两步：

1. `baseCompile(template)`返回生成好的代码字符串；
2. 通过`new Function('Vue', code)`创建函数，并把`runtimeDom`作为参数传进去。

生成代码中的：

```ts
const { toDisplayString: _toDisplayString } = Vue
```

此时拿到的`Vue`就是传入的`runtimeDom`对象。

最后注册到运行时：

```ts
registerRuntimeCompiler(compilerToFunction);
```

### 四、runtime-core保存编译器

`runtime-core/src/component.ts`中使用一个模块级变量保存编译器：

```ts
let compiler;

export function registerRuntimeCompiler(_compiler) {
  compiler = _compiler;
}
```

组件完成`setup`以后，会进入`finishComponentSetup`：

```ts
function finishComponentSetup(instance) {
  const Component = instance.type;

  if (compiler && !Component.render) {
    if (Component.template) {
      Component.render = compiler(Component.template);
    }
  }

  instance.render = Component.render;
}
```

这里的判断有两个条件：

1. 运行时已经注册了编译器；
2. 组件没有直接提供`render`。

如果组件已经提供render，就直接使用它；如果只有template，就在组件初始化阶段把template编译成render。

### 五、template示例的完整执行顺序

仓库里的`compiler-base`示例：

```ts
export const App = {
  name: 'App',
  template: '<div>hi, {{ msg }} {{ count }}</div>',

  setup() {
    const count = ref(0);

    return {
      msg: 'zjt-mini-vue',
      count
    };
  }
};
```

调用：

```ts
createApp(App).mount(rootContainer);
```

完整过程如下：

```text
1. createApp(App)
2. rootComponent -> root vnode
3. mount -> mountComponent
4. setupComponent
5. setupStatefulComponent执行setup
6. finishComponentSetup发现没有render
7. compiler(App.template)
8. baseParse生成AST
9. transform补充codegen信息
10. generate生成render代码
11. new Function得到render函数
12. instance.render = render
13. setupRenderEffect执行render
14. render返回element vnode
15. runtime-core.patch
16. runtime-dom创建真实div
```

这也解释了为什么编译器注册动作要发生在组件挂载之前。

### 六、插值如何从template走到页面

以`{{ msg }}`为例，中间会经过几次形态变化。

#### （一）模板字符串

```vue
<div>hi, {{ msg }}</div>
```

#### （二）parse后的AST

```text
ELEMENT div
  TEXT "hi, "
  INTERPOLATION "msg"
```

#### （三）transform后的信息

```text
INTERPOLATION
  expression -> _ctx.msg
  helper -> toDisplayString

ELEMENT
  codegenNode -> createElementVNode
  children -> compound expression
```

#### （四）codegen后的代码

```ts
const { toDisplayString: _toDisplayString, createElementVNode: _createElementVNode } = Vue
return function render (_ctx, _cache) {
  return (_createElementVNode)(
    'div',
    null,
    'hi, ' + _toDisplayString(_ctx.msg)
  )
}
```

#### （五）runtime执行

生成的render函数被调用时，`_ctx`就是组件代理`instance.proxy`：

```ts
const subTree = instance.render.call(proxy, proxy);
```

因此`_ctx.msg`最终会通过公共代理读取到`setupState.msg`，`_toDisplayString`和`_createElementVNode`则来自`runtimeDom`。

### 七、编译结果为什么能参与响应式更新

编译器只负责把模板转换成render函数，响应式更新仍然由runtime-core的effect负责：

```ts
instance.update = effect(() => {
  const subTree = instance.render.call(proxy, proxy);
  patch(prevSubTree, subTree, container, instance, anchor);
});
```

render函数每次重新执行时，都会读取`_ctx.msg`和`_ctx.count`。这些读取会被reactivity收集，所以：

```text
count.value变化
  -> 触发组件effect
  -> 重新执行编译后的render
  -> 生成新的vnode
  -> runtime-core做patch
  -> runtime-dom更新页面
```

这就是“编译器”和“响应式运行时”各自负责什么：

- compiler把声明式模板翻译成命令式函数调用；
- reactivity负责追踪render读取的数据；
- runtime-core负责比较vnode；
- runtime-dom负责操作真实DOM。

### 八、new Function需要注意什么

当前学习版使用`new Function`执行生成代码：

```ts
new Function('Vue', code)(runtimeDom);
```

这种方式非常适合演示“字符串代码如何变成函数”，但不要对不可信的模板直接使用。真实工程会根据构建目标、运行环境和安全边界采用更完整的编译方案。

### 九、当前compiler-runtime联动的边界

当前链路已经可以跑通基础模板，但仍然有一些限制：

- 组件template只能使用当前已经实现的节点和表达式能力；
- 生成的render参数只有`_ctx`和`_cache`；
- props、指令和事件属性还没有进入完整的编译流程；
- 多根节点、条件渲染和列表渲染尚未实现；
- codegen生成的字符串没有做完整的格式和安全处理。

这些限制和当前compiler-core的happy path范围一致，不影响主链路的学习。

### end

至此，`runtime-core`和`compiler-core`的主流程已经闭环：

```text
template
  -> parse
  -> transform
  -> codegen
  -> render
  -> vnode
  -> patch
  -> DOM
```

这也是整个mini-vue从“模板”走到“页面”的完整过程。
