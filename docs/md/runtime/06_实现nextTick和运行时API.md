# 06_实现nextTick和运行时API

### 一、为什么需要异步更新

在组件更新流程中，响应式数据变化以后并没有直接执行组件的更新runner，而是交给`scheduler`：

```ts
{
  scheduler() {
    queueJobs(instance.update);
  }
}
```

这样做是因为一次用户操作中可能连续修改多次状态：

```ts
for (let i = 0; i < 100; i++) {
  count.value = i;
}
```

如果每次赋值都同步执行render，组件就会重复生成很多中间结果。我们更希望把同一个更新任务合并起来，在当前同步代码执行结束后统一刷新。

### 二、queueJobs实现任务去重

当前的任务队列是一个数组：

```ts
const queue = [];
const p = Promise.resolve();
let isFlushPending = false;
```

加入任务时先判断队列中是否已经存在：

```ts
export function queueJobs(job) {
  if (!queue.includes(job)) {
    queue.push(job);
  }

  queueFlush();
}
```

这里的`job`通常就是`instance.update`。同一个组件在一次同步操作中被触发多次，也只会在队列中保留一个runner。

### 三、queueFlush保证只开启一次刷新

如果每次调用`queueJobs`都创建一个Promise回调，就会产生很多重复的刷新任务。因此通过`isFlushPending`做一次保护：

```ts
function queueFlush() {
  if (isFlushPending) return;

  isFlushPending = true;
  nextTick(flushJobs);
}
```

队列真正刷新时，先把状态恢复成`false`，再逐个执行任务：

```ts
function flushJobs() {
  isFlushPending = false;

  let job;
  while (job = queue.shift()) {
    job && job();
  }
}
```

这里使用`while`不断取出队首，所以在刷新过程中加入的新任务也可以继续被处理。

### 四、nextTick的实现

`nextTick`的核心就是一个已经完成的Promise：

```ts
const p = Promise.resolve();

export function nextTick(fn) {
  return fn ? p.then(fn) : p;
}
```

它支持两种调用方式：

```ts
nextTick(() => {
  console.log('DOM更新后的回调');
});

await nextTick();
console.log('DOM更新完成');
```

由于`queueFlush`把`flushJobs`放到Promise回调中，所以当前同步代码结束后，队列会在微任务阶段执行。`nextTick`则可以让用户代码排到同一条异步链路之后。

### 五、nextTicker案例

仓库中的`nextTicker`示例大致是：

```ts
function onClick() {
  for (let i = 0; i < 100; i++) {
    count.value = i;
  }

  nextTick(() => {
    console.log('instance');
  });
}
```

这段代码可以用来观察两件事情：

1. 多次修改`count`不会立刻执行100次完整更新；
2. `nextTick`回调会等更新队列刷新以后再执行。

### 六、getCurrentInstance的实现

有一些运行时API需要知道“当前正在执行哪个组件的setup”。当前实现使用一个模块级变量保存它：

```ts
let currentInstance = null;

export function getCurrentInstance() {
  return currentInstance;
}

export function setCurrentInstance(instance) {
  currentInstance = instance;
}
```

执行组件`setup`之前先设置，执行结束以后清空：

```ts
setCurrentInstance(instance);
const setupResult = setup(
  shallowReadonly(instance.props),
  { emit: instance.emit }
);
setCurrentInstance(null);
```

所以`getCurrentInstance`只能在这段setup执行期间拿到当前实例：

```ts
setup() {
  const instance = getCurrentInstance();
  console.log(instance);
}
```

这种写法的本质是把“当前调用上下文”暂时保存起来，让setup内部的API不需要层层传递instance参数。

### 七、provide的实现

`provide`首先通过`getCurrentInstance`拿到当前组件，然后把数据保存到当前实例的`provides`上：

```ts
export function provide(key, value) {
  const currentInstance = getCurrentInstance();

  if (currentInstance) {
    const { provides } = currentInstance;
    provides[key] = value;
  }
}
```

使用方式：

```ts
setup() {
  provide('foo', 'fooVal');
}
```

当前实现把`provides`放在组件实例上，是为了让后代组件可以沿着组件关系找到它。

### 八、inject的实现

子组件调用`inject`时，同样先拿到当前实例，再读取父组件的`provides`：

```ts
export function inject(key, defaultValue) {
  const currentInstance = getCurrentInstance();

  if (currentInstance) {
    const parentProvides = currentInstance.parent.provides;

    if (key in parentProvides) {
      return parentProvides[key];
    } else if (defaultValue) {
      return isFunction(defaultValue)
        ? defaultValue()
        : defaultValue;
    }
  }
}
```

它支持直接传入默认值：

```ts
const baz = inject('baz', 'bazDefault');
```

也支持传入一个默认值函数：

```ts
const baz = inject('baz', () => 'bazDefault');
```

默认值函数只有在没有找到对应key时才会执行。

### 九、这些API和组件实例的关系

当前几个运行时API的依赖关系可以画成：

```text
setup开始
  -> setCurrentInstance(instance)
       ├── getCurrentInstance
       ├── provide
       └── inject
  -> 执行setup
  -> setCurrentInstance(null)
```

所以，`getCurrentInstance`、`provide`和`inject`并不是全局随便读取数据，而是依赖组件初始化时建立的父子实例关系。

### 十、当前实现的边界

当前代码优先实现了最短可运行链路，因此和Vue完整运行时相比还有一些简化：

- 更新队列只实现了基础去重，没有处理完整的优先级和递归调度；
- `provide`数据没有实现完整的原型继承策略；
- `inject`主要覆盖当前示例中的父子组件场景；
- `nextTick`只围绕当前Promise队列提供基础能力。

先把这些核心关系跑通以后，再继续补充边界会更容易。

### end

现在，组件更新已经有了“任务队列”，组件内部也具备了读取当前实例和跨层级传值的能力。下一篇看`createRenderer`，理解为什么同一套`runtime-core`不仅可以渲染DOM，也可以接入其它平台。
