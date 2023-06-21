# 01_实现初始化component主流程

### 一、从createApp开始

在开始实现`runtime-core`之前，先想一下一个最简单的使用场景：

```ts
const App = {
  setup() {
    return {
      msg: 'zjt-mini-vue'
    };
  },

  render() {
    return h('div', {}, this.msg);
  }
};

const rootContainer = document.querySelector('#app');
createApp(App).mount(rootContainer);
```

这个过程看起来只有两行代码，但运行时需要完成很多事情：

1. 把根组件`App`转换成`vnode`；
2. 根据`vnode`的类型创建组件实例；
3. 执行组件的`setup`；
4. 拿到组件的`render`函数；
5. 执行`render`得到组件的子树；
6. 继续把子树渲染成真正的元素。

所以，`component`的初始化并不是直接操作`DOM`，而是先建立一套组件实例和虚拟节点之间的关系。后面的渲染、更新和卸载，都是在这套关系上继续完成的。

### 二、createApp和renderer之间的关系

`createApp`并没有把渲染逻辑写死，它是由`createRenderer`创建出来的。

```ts
// packages/runtime-core/src/createApp.ts
export function createAppAPI(render) {
  return function createApp(rootComponent) {
    return {
      mount(rootContainer) {
        // 先转换成 vNode
        const vnode = createVNode(rootComponent);

        render(vnode, rootContainer);
      }
    };
  }
}
```

可以注意到，`createAppAPI`接收了一个`render`函数，然后返回真正的`createApp`。这样做的好处是：`createApp`只负责组装入口，真正怎么渲染由外部传进来的`render`决定。

`renderer.ts`最后返回：

```ts
return {
  createApp: createAppAPI(render)
};
```

所以整个调用链可以先记成下面这样：

```text
createApp(App).mount(container)
        ↓
createVNode(App)
        ↓
render(vnode, container)
        ↓
patch(null, vnode, container, null, null)
```

这里的`null`表示当前还没有旧节点，也就是第一次初始化。

### 三、vnode如何判断是component

`createVNode`会先保存节点的基本信息，然后通过`getShapeFlag`判断节点类型。

```ts
export function createVNode(type, props?, children?) {
  const vnode = {
    type,
    props,
    children,
    component: null,
    key: props && props.key,
    shapeFlag: getShapeFlag(type),
    el: undefined
  };

  return vnode;
}

function getShapeFlag(type) {
  return isString(type)
    ? ShapeFlags.ELEMENT
    : ShapeFlags.STATEFUL_COMPONENT;
}
```

在这个最简实现中，`type`是字符串时认为它是普通元素，例如`div`；`type`是对象时认为它是有状态组件，例如上面的`App`。

`shapeFlag`使用的是位标记，后续还会继续叠加子节点类型。这样在`patch`中就可以通过与运算快速判断当前节点该走哪条分支。

### 四、patch分发到mountComponent

`renderer.ts`中的`patch`是运行时的分发中心：

```ts
function patch(n1, n2, container, parentComponent, anchor) {
  const { type, shapeFlag } = n2;

  switch (type) {
    case Fragment:
      processFragment(n1, n2, container, parentComponent, anchor);
      break;
    case Text:
      processText(n1, n2, container);
      break;
    default:
      if (shapeFlag & ShapeFlags.ELEMENT) {
        processElement(n1, n2, container, parentComponent, anchor);
      } else if (shapeFlag & ShapeFlags.STATEFUL_COMPONENT) {
        processComponent(n1, n2, container, parentComponent, anchor);
      }
      break;
  }
}
```

当前`App`的`shapeFlag`是`STATEFUL_COMPONENT`，所以会进入`processComponent`：

```ts
function processComponent(n1, n2, container, parentComponent, anchor) {
  if (n1 == null) {
    mountComponent(n2, container, parentComponent, anchor);
  } else {
    updateComponent(n1, n2);
  }
}
```

第一次渲染时`n1`为`null`，所以继续进入`mountComponent`。

### 五、创建组件实例并执行setup

组件实例是运行时保存组件状态的地方。当前实现里，它至少保存了下面这些内容：

```ts
export function createComponentInstance(vnode, parent) {
  const component = {
    vnode,
    type: vnode.type,
    next: null,
    setupState: {},
    props: {},
    slots: {},
    provides: {},
    parent,
    isMounted: false,
    subTree: {},
    emit: () => {}
  };

  component.emit = emit.bind(null, component);

  return component;
}
```

接着开始初始化组件：

```ts
function mountComponent(initialVnode, container, parentComponent, anchor) {
  const instance = (initialVnode.component =
    createComponentInstance(initialVnode, parentComponent));

  setupComponent(instance);
  setupRenderEffect(instance, initialVnode, container, anchor);
}
```

`setupComponent`目前会依次处理`props`、`slots`，然后初始化有状态组件：

```ts
export function setupComponent(instance) {
  initProps(instance, instance.vnode.props);
  initSlots(instance, instance.vnode.children);

  setupStatefulComponent(instance);
}
```

在`setupStatefulComponent`中，会给组件实例创建一个代理对象：

```ts
instance.proxy = new Proxy({ _: instance }, PublicInstanceHandlers);

const { setup } = Component;

if (setup) {
  setCurrentInstance(instance);
  const setupResult = setup(
    shallowReadonly(instance.props),
    { emit: instance.emit }
  );
  setCurrentInstance(null);

  handleSetupResult(instance, setupResult);
}
```

这里有三个值得注意的点：

1. `setup`拿到的`props`是`shallowReadonly`，组件内部可以读取，但不能直接修改；
2. `setup`的第二个参数是`context`，当前支持`emit`；
3. 执行`setup`之前记录当前实例，`getCurrentInstance`和`provide/inject`都会依赖这段状态。

当`setup`返回一个对象时，运行时把它保存到`setupState`，并通过`proxyRefs`处理`ref`的自动拆包：

```ts
function handleSetupResult(instance, setupResult) {
  if (isObject(setupResult)) {
    instance.setupState = proxyRefs(setupResult);
  }

  finishComponentSetup(instance);
}
```

最后，`finishComponentSetup`把组件上的`render`函数放到实例上。如果组件只有`template`没有`render`，编译器接入以后也会在这里补上，这部分放到`compiler`篇再展开。

### 六、setupRenderEffect完成首次渲染

组件实例准备好以后，`setupRenderEffect`会创建一个响应式`effect`：

```ts
function setupRenderEffect(instance, initialVnode, container, anchor) {
  instance.update = effect(() => {
    if (!instance.isMounted) {
      const { proxy } = instance;
      const subTree = (instance.subTree =
        instance.render.call(proxy, proxy));

      patch(null, subTree, container, instance, anchor);

      initialVnode.el = subTree.el;
      instance.isMounted = true;
    }
  });
}
```

首次执行时，`instance.render.call(proxy, proxy)`会返回组件的子树。例如：

```ts
render() {
  return h('div', {}, this.msg);
}
```

这个返回值还是一个`vnode`，所以不能直接插入页面，而是要再次调用`patch`。这一次节点类型变成了`ELEMENT`，后续就会走`mountElement`，真正创建`div`。

到这里，初始化`component`的主流程就串起来了：

```text
rootComponent
  -> createVNode
  -> createComponentInstance
  -> setupComponent
  -> setupStatefulComponent
  -> setupRenderEffect
  -> component.render
  -> subTree
  -> patch(subTree)
  -> mountElement
```

### end

这一篇先把组件初始化的骨架搭起来。可以看到，组件本身并不负责直接创建`DOM`，它只负责产生子树；真正的元素创建、属性处理和子节点处理，都交给`renderer`继续完成。

下一篇继续看`mountElement`，把`vnode`变成真实元素。
