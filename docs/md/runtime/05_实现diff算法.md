# 05_实现diff算法

### 一、为什么需要diff

当新旧子节点都是数组时，如果直接把旧节点全部删除，再把新节点全部创建，虽然逻辑简单，但会浪费很多真实DOM操作。

例如旧节点和新节点分别是：

```text
旧: A B C D
新: A C E B
```

其中`A`、`B`、`C`都可能是可以复用的节点。diff要解决的问题就是：

1. 哪些节点可以继续复用；
2. 哪些旧节点需要删除；
3. 哪些新节点需要创建；
4. 哪些已有节点只需要移动位置。

当前实现的入口是`patchKeyedChildren`，它只在“旧数组 -> 新数组”时执行。

### 二、key是节点身份

创建vnode时会把`props.key`保存到节点上：

```ts
const vnode = {
  type,
  props,
  children,
  key: props && props.key,
  shapeFlag: getShapeFlag(type),
  el: undefined
};
```

两个节点是否是同一个节点，当前通过`type`和`key`共同判断：

```ts
function isSameVNodeType(n1, n2) {
  return n1.type === n2.type && n1.key === n2.key;
}
```

这里的`key`不是为了改变显示内容，而是为了告诉diff：“这个节点在新旧列表中代表同一个身份”。

### 三、双端对比的四个指针

`patchKeyedChildren`首先准备四个边界：

```ts
let i = 0;
let l2 = c2.length;
let e1 = c1.length - 1;
let e2 = l2 - 1;
```

可以把它们理解为：

- `i`：从左侧开始比较的位置；
- `e1`：旧数组右侧边界；
- `e2`：新数组右侧边界；
- `s1`、`s2`：中间区域开始的位置。

双端对比的顺序是：

```text
先比较左侧
    ↓
再比较右侧
    ↓
处理长度变化
    ↓
处理剩余中间区域
```

### 四、第一步：左侧对比

旧数组和新数组左侧相同的部分，可以直接递归patch：

```ts
while (i <= e1 && i <= e2) {
  const n1 = c1[i];
  const n2 = c2[i];

  if (isSameVNodeType(n1, n2)) {
    patch(n1, n2, container, parentComponent, anchor);
  } else {
    break;
  }

  i++;
}
```

例如：

```text
旧: A B C
新: A B D E
    ↑ ↑
```

`A`和`B`已经确认是同一个节点，只需要分别递归更新，不需要进入复杂的中间diff。

### 五、第二步：右侧对比

右侧也是同样的思路，只是从数组末尾向前移动：

```ts
while (i <= e1 && i <= e2) {
  const n1 = c1[e1];
  const n2 = c2[e2];

  if (isSameVNodeType(n1, n2)) {
    patch(n1, n2, container, parentComponent, anchor);
  } else {
    break;
  }

  e1--;
  e2--;
}
```

例如：

```text
旧: A B C
新: D E B C
        ↑ ↑
```

`B`和`C`可以先被处理掉，中间只剩下新增的`D`和`E`。

### 六、第三步：只新增或只删除

经过左右两端对比以后，如果旧数组已经处理完，但新数组还有节点：

```ts
if (i > e1) {
  if (i <= e2) {
    const nextPos = e2 + 1;
    const anchor = nextPos < l2 ? c2[nextPos].el : null;

    while (i <= e2) {
      patch(null, c2[i], container, parentComponent, anchor);
      i++;
    }
  }
}
```

这表示新数组比旧数组长，剩余节点全部是新节点，直接创建即可。

反过来，如果新数组已经处理完，但旧数组还有节点：

```ts
else if (i > e2) {
  while (i <= e1) {
    hostRemove(c1[i].el);
    i++;
  }
}
```

这表示旧数组中剩余的节点已经不在新数组里，需要删除。

### 七、第四步：处理中间区域

最复杂的是两边都还有未处理节点：

```text
旧: A [B C D] E
新: A [D F B] E
```

当前代码先记录中间区间：

```ts
const s1 = i;
const s2 = i;
const toBePatched = e2 - s2 + 1;
let patched = 0;
```

#### （一）建立新节点索引表

先遍历新节点，建立`key -> index`的映射：

```ts
const keyToNewIndexMap = new Map();

for (let i = s2; i <= e2; i++) {
  const nextChild = c2[i];
  keyToNewIndexMap.set(nextChild.key, i);
}
```

这样，旧节点就可以通过key快速找到自己在新数组中的位置。

#### （二）遍历旧节点

接着遍历旧中间区间：

1. 如果旧节点在新数组中找不到，直接删除；
2. 找得到，就patch旧节点和新节点；
3. 记录它在新数组中的位置；
4. 根据位置是否递增，判断后续是否需要移动。

核心数据结构是`newIndexToOldIndexMap`：

```ts
const newIndexToOldIndexMap = new Array(toBePatched);

for (let i = 0; i < toBePatched; i++) {
  newIndexToOldIndexMap[i] = 0;
}
```

它的下标对应新节点的位置，值对应旧节点的索引加一：

```text
0       -> 新节点还不存在，需要创建
旧索引+1 -> 新节点可以复用旧节点
```

之所以使用`旧索引 + 1`，是为了保留`0`这个特殊值。

### 八、如何判断节点是否移动

遍历旧节点时，代码会持续记录当前遇到的最大新索引：

```ts
let moved = false;
let maxNewIndexSoFar = 0;

if (newIndex >= maxNewIndexSoFar) {
  maxNewIndexSoFar = newIndex;
} else {
  moved = true;
}
```

如果新索引一直递增，说明节点的相对顺序没有被打乱，不需要额外移动。如果出现了回退，就说明至少有节点发生了位置变化。

例如旧节点按顺序找到的新索引是：

```text
0, 1, 2, 3
```

它本身就是递增序列；如果变成：

```text
2, 0, 1
```

就说明有节点被移动了。

### 九、最长递增子序列

发生移动时，不需要把所有节点都移动一遍。可以保留最长递增子序列中的节点，让其它节点移动到正确位置：

```ts
const increasingNewIndexSequence =
  moved ? getSequence(newIndexToOldIndexMap) : [];
```

`getSequence`返回的是索引序列，而不是节点值。当前实现使用二分查找维护递增序列，并通过`p`数组回溯出最终结果：

```ts
function getSequence(arr) {
  const p = arr.slice();
  const result = [0];

  // 通过二分查找维护递增序列
  // ...

  return result;
}
```

比如：

```text
newIndexToOldIndexMap: [3, 1, 2]
最长递增子序列:       [1, 2]
```

这意味着可以尽量保留后两个节点，只移动第一个节点。

### 十、为什么要倒序处理

中间区域最后从右向左遍历：

```ts
for (let i = toBePatched - 1; i >= 0; i--) {
  const nextIndex = i + s2;
  const nextChild = c2[nextIndex];
  const anchor =
    nextIndex + 1 < l2 ? c2[nextIndex + 1].el : null;

  if (newIndexToOldIndexMap[i] === 0) {
    patch(null, nextChild, container, parentComponent, anchor);
  } else if (moved) {
    // 不在最长递增子序列中的节点，需要移动
    hostInsert(nextChild.el, container, anchor);
  }
}
```

倒序处理配合`anchor`，可以让新节点插入到正确位置，并且避免前面节点的插入影响后面节点的定位。

### 十一、diff流程总结

当前的keyed children diff可以概括为：

```text
左侧相同 -> patch
右侧相同 -> patch
旧的处理完 -> 创建新的
新的处理完 -> 删除旧的
中间区域
  -> key建立映射
  -> 删除不存在的旧节点
  -> patch可以复用的节点
  -> 计算最长递增子序列
  -> 倒序创建和移动
```

这也是从“简单更新element”演进到“可以处理列表变化”的关键一步。

### end

diff的目标不是让代码看起来更复杂，而是尽可能减少真实DOM操作。`key`负责确认节点身份，双端比较负责快速跳过两侧已经稳定的部分，最长递增子序列负责减少移动次数。

下一篇继续看更新队列、`nextTick`以及组件实例相关的运行时API。
