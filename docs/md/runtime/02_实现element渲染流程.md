# 02_实现element渲染流程

### 一、从vnode到真实element

上一节中，组件的`render`函数返回了一个`vnode`，但是页面上还没有任何真实节点。

例如：

```ts
render() {
  return h('div', { class: 'red' }, 'hello mini-vue');
}
```

这个`h`函数返回的只是一个普通JavaScript对象。运行时接下来要做的事情就是：

1. 判断当前`vnode`是component还是element；
2. 创建真实元素；
3. 设置`props`；
4. 处理文本子节点或数组子节点；
5. 把元素插入父容器。

### 二、ShapeFlags设计

在`createVNode`中，除了保存`type`、`props`和`children`，还会给节点打上`shapeFlag`：

```ts
export const enum ShapeFlags {
  ELEMENT = 1,
  STATEFUL_COMPONENT = 1 << 1,
  TEXT_CHILDREN = 1 << 2,
  ARRAY_CHILDREN = 1 << 3,
  SLOT_CHILDREN = 1 << 4
}
```

这些值使用二进制位，所以一个节点可以同时拥有多个标识：

| 标识 | 二进制含义 | 作用 |
| --- | --- | --- |
| `ELEMENT` | `1` | 当前节点是普通DOM元素 |
| `STATEFUL_COMPONENT` | `10` | 当前节点是有状态组件 |
| `TEXT_CHILDREN` | `100` | 子节点是字符串 |
| `ARRAY_CHILDREN` | `1000` | 子节点是vnode数组 |
| `SLOT_CHILDREN` | `10000` | 组件子节点是插槽对象 |

设置子节点标识的代码如下：

```ts
if (isString(vnode.children)) {
  vnode.shapeFlag |= ShapeFlags.TEXT_CHILDREN;
} else if (isArray(vnode.children)) {
  vnode.shapeFlag |= ShapeFlags.ARRAY_CHILDREN;
}
```

后续在`renderer`中只需要做与运算，就能同时判断节点和子节点类型：

```ts
if (shapeFlag & ShapeFlags.ELEMENT) {
  processElement(n1, n2, container, parentComponent, anchor);
}
```

这种写法比在每次处理时重新判断对象类型更直接，也为后面增加插槽、Fragment等场景留出了空间。

### 三、h函数只是createVNode的别名

手写render函数时，通常使用`h`：

```ts
export function h(type, props?, children?) {
  return createVNode(type, props, children);
}
```

所以：

```ts
h('div', { id: 'app' }, 'hello');
```

本质上就是：

```ts
createVNode('div', { id: 'app' }, 'hello');
```

编译器生成的`_createElementVNode`也会指向同一个`createVNode`实现：

```ts
export {
  createVNode as createElementVNode
};
```

这样，手写render和template编译后的render，最后走的是同一条运行时路径。

### 四、mountElement的主流程

当`patch`判断当前节点是普通元素时，会进入`processElement`：

```ts
function processElement(n1, n2, container, parentComponent, anchor) {
  if (!n1) {
    mountElement(n2, container, parentComponent, anchor);
  } else {
    patchElement(n1, n2, container, parentComponent, anchor);
  }
}
```

第一次渲染时没有旧节点，所以会执行`mountElement`：

```ts
function mountElement(vnode, container, parentComponent, anchor) {
  const { type, props, children, shapeFlag } = vnode;

  const el = (vnode.el = hostCreateElement(type));

  for (const key in props) {
    const val = props[key];
    hostPatchProp(el, key, null, val);
  }

  if (shapeFlag & ShapeFlags.TEXT_CHILDREN) {
    el.textContent = children;
  } else if (shapeFlag & ShapeFlags.ARRAY_CHILDREN) {
    mountChildren(vnode.children, el, parentComponent, anchor);
  }

  hostInsert(el, container, anchor);
}
```

这里的执行顺序很重要：

1. 先通过`hostCreateElement`创建元素；
2. 把真实元素保存到`vnode.el`；
3. 设置当前节点的属性；
4. 处理子节点；
5. 最后插入父容器。

把真实元素保存到`vnode.el`以后，下一次更新时就可以通过旧`vnode.el`找到需要复用的DOM节点，而不是重新创建。

### 五、runtime-dom提供平台能力

`runtime-core`并不知道浏览器API，它只依赖外部传入的host函数。浏览器平台的实现放在`runtime-dom`：

```ts
function createElement(type) {
  return document.createElement(type);
}

function insert(child, parent, anchor) {
  parent.insertBefore(child, anchor ?? null);
}

function remove(child) {
  const parent = child.parentNode;
  if (parent) {
    parent.removeChild(child);
  }
}

function setElementText(el, text) {
  el.textContent = text;
}
```

然后组装成renderer：

```ts
const renderer = createRenderer({
  createElement,
  patchProp,
  insert,
  remove,
  setElementText
});
```

当前阶段的重点是先把核心运行时和平台API分开。`runtime-core`只负责调度，`runtime-dom`负责告诉它“如何创建一个浏览器元素”。

### 六、props和事件也走hostPatchProp

在`mountElement`中，所有属性都会交给`hostPatchProp`：

```ts
for (const key in props) {
  const val = props[key];
  hostPatchProp(el, key, null, val);
}
```

浏览器平台在`patchProp`里区分事件和普通属性：

```ts
function patchProp(el, key, prevVal, nextVal) {
  if (isOn(key)) {
    const name = key.slice(2).toLowerCase();
    el.addEventListener(name, nextVal);
  } else {
    if (nextVal === null || nextVal === undefined) {
      el.removeAttribute(key);
    } else {
      el.setAttribute(key, nextVal);
    }
  }
}
```

`isOn`通过正则判断属性名是不是以`on`开头，并且第三个字符不是小写字母。于是：

- `onClick`会被识别成事件；
- `id`、`class`会被当作普通属性；
- 事件名会从`onClick`转换成浏览器需要的`click`。

这里先实现的是最简happy path，事件更新和复杂的class、style处理，和Vue完整实现还有差别。

### 七、数组子节点继续递归patch

如果子节点是数组，就不能直接给`textContent`赋值，而是要继续调用`patch`：

```ts
function mountChildren(children, container, parentComponent, anchor) {
  children.forEach(vnode => {
    patch(null, vnode, container, parentComponent, anchor);
  });
}
```

例如：

```ts
h('div', {}, [
  h('p', {}, 'hello'),
  h('p', {}, 'mini-vue')
]);
```

执行过程是：

```text
mountElement(div)
  -> mountChildren
     -> patch(p)
        -> mountElement(p)
     -> patch(p)
        -> mountElement(p)
```

这就是虚拟节点树递归转换成真实节点树的过程。

### 八、组件代理对象

组件的`render`函数中经常会写`this.msg`、`this.$el`。这里的`this`并不是组件实例本身，而是`instance.proxy`：

```ts
instance.proxy = new Proxy({ _: instance }, PublicInstanceHandlers);
```

代理的读取顺序是：

```ts
get({ _: instance }, key) {
  const { setupState, props } = instance;

  if (hasOwn(setupState, key)) {
    return setupState[key];
  } else if (hasOwn(props, key)) {
    return props[key];
  }

  const publicGetter = publicPropertiesMap[key];
  if (publicGetter) {
    return publicGetter(instance);
  }
}
```

当前支持的公共属性包括：

```ts
const publicPropertiesMap = {
  $el: i => i.vnode.el,
  $slots: i => i.slots,
  $props: i => i.props
};
```

所以，当`render`中读取`this.msg`时，会先从`setupState`里找；读取`this.$el`时，则会通过公共属性映射拿到根元素。

### end

至此，`vnode`已经可以经过`patch`、`mountElement`和host操作，最终出现在页面中。下一篇继续看组件之间如何传递`props`、触发`emit`，以及插槽是怎么被转换成Fragment的。
