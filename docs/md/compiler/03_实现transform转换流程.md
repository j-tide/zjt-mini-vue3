# 03_实现transform转换流程

### 一、为什么parse之后还需要transform

`parse`只负责识别模板结构。例如：

```text
<div>hi, {{ message }}</div>
```

解析以后可以知道这里有一个element、一个text和一个interpolation，但还不知道：

1. 插值表达式运行时应该从哪里读取；
2. element应该调用哪个VNode创建函数；
3. 相邻的text和interpolation应该怎样合并；
4. 当前模板需要哪些runtime helper。

这些信息由`transform`阶段补充。

### 二、transform采用插件机制

`transform`接收一个AST和多个`nodeTransforms`：

```ts
export function transform(root, options = {}) {
  const context = createTransformContext(root, options);

  traverseNode(root, context);
  createRootCodegen(root);

  root.helpers = [...context.helpers.keys()];
}
```

当前`baseCompile`注册了三个转换插件：

```ts
transform(ast, {
  nodeTransforms: [
    transformsExpression,
    transformElement,
    transformText
  ]
});
```

插件机制的好处是：遍历逻辑只有一份，不同语法能力通过不同插件扩展，而不是把所有逻辑都写进一个巨大的`transform`函数。

### 三、创建transform上下文

上下文保存本次转换过程中的共享状态：

```ts
function createTransformContext(root, options) {
  const context = {
    root,
    nodeTransforms: options.nodeTransforms || [],
    helpers: new Map(),
    helper(key) {
      context.helpers.set(key, 1);
    }
  };

  return context;
}
```

`helpers`使用`Map`去重。同一个模板中出现多个插值，也只需要在生成代码时引入一次`toDisplayString`。

### 四、深度优先遍历AST

`traverseNode`会依次执行当前节点上的所有插件：

```ts
function traverseNode(node, context) {
  const { nodeTransforms } = context;
  const exitFns = [];

  for (let i = 0; i < nodeTransforms.length; i++) {
    const transform = nodeTransforms[i];
    const onExit = transform(node, context);
    if (onExit) exitFns.push(onExit);
  }

  switch (node.type) {
    case NodeTypes.INTERPOLATION:
      context.helper(TO_DISPLAY_STRING);
      break;
    case NodeTypes.ROOT:
    case NodeTypes.ELEMENT:
      traverseChildren(node, context);
      break;
  }

  let i = exitFns.length;
  while (i--) {
    exitFns[i]();
  }
}
```

它的执行顺序不是简单的“插件执行完就结束”，而是：

```text
进入当前节点
  -> 执行插件前置逻辑
  -> 深度优先遍历子节点
  -> 倒序执行插件返回的exit逻辑
```

之所以需要`exit`阶段，是因为父节点的代码生成信息往往要等子节点先转换完成以后才能确定。

### 五、transformsExpression处理插值

模板中的表达式：

```vue
{{ message }}
```

在render函数中不能直接读取`message`，而需要从组件上下文`_ctx`中读取：

```ts
_ctx.message
```

因此`transformsExpression`会在插值节点的exit阶段修改表达式：

```ts
export function transformsExpression(node) {
  if (node.type === NodeTypes.INTERPOLATION) {
    return () => {
      node.content = processExpression(node.content);
    };
  }
}

function processExpression(node) {
  node.content = '_ctx.' + node.content;
  return node;
}
```

同时，遍历到插值节点时会收集`TO_DISPLAY_STRING`：

```ts
case NodeTypes.INTERPOLATION:
  context.helper(TO_DISPLAY_STRING);
  break;
```

### 六、transformElement生成VNode调用信息

element转换发生在exit阶段：

```ts
export function transformElement(node, context) {
  if (node.type === NodeTypes.ELEMENT) {
    return () => {
      const vnodeTag = "'" + node.tag + "'";
      let vnodeProps;
      const { children } = node;
      const vnodeChildren = children[0];

      node.codegenNode = createVNodeCall(
        context,
        vnodeTag,
        vnodeProps,
        vnodeChildren
      );
    };
  }
}
```

这里的`createVNodeCall`会注册`CREATE_ELEMENT_VNODE`helper：

```ts
export function createVNodeCall(
  context,
  tag,
  props,
  children
) {
  context.helper(CREATE_ELEMENT_VNODE);

  return {
    type: NodeTypes.ELEMENT,
    tag,
    props,
    children
  };
}
```

转换以后，element节点上会多出一个`codegenNode`，它已经包含了后续生成`createElementVNode`调用所需要的信息。

### 七、transformText合并相邻文本

对于：

```vue
<div>hi, {{ message }}</div>
```

如果text和interpolation分别生成两个参数，最终的VNode调用不够自然。当前实现会把相邻的文本类型节点合并成`COMPOUND_EXPRESSION`：

```ts
{
  type: NodeTypes.COMPOUND_EXPRESSION,
  children: [
    textNode,
    ' + ',
    interpolationNode
  ]
}
```

核心逻辑是：

```ts
if (!currentContainer) {
  currentContainer = children[i] = {
    type: NodeTypes.COMPOUND_EXPRESSION,
    children: [child]
  };
}

currentContainer.children.push(' + ', next);
children.splice(j, 1);
j--;
```

这样，后面的`codegen`就可以把它生成成：

```ts
'hi, ' + _toDisplayString(_ctx.message)
```

### 八、根节点的codegenNode

遍历结束以后，`createRootCodegen`会决定根节点最终使用哪一个代码生成节点：

```ts
function createRootCodegen(root) {
  const child = root.children[0];

  if (child.type === NodeTypes.ELEMENT) {
    root.codegenNode = child.codegenNode;
  } else {
    root.codegenNode = root.children[0];
  }
}
```

当前happy path假设模板只有一个根节点，所以直接取第一个子节点。

### 九、helper收集结果

转换完成以后，所有helper会被放到根节点：

```ts
root.helpers = [...context.helpers.keys()];
```

例如模板：

```vue
<div>hi, {{ message }}</div>
```

至少需要两个helper：

```text
TO_DISPLAY_STRING
CREATE_ELEMENT_VNODE
```

后面的`codegen`会使用这个列表生成helper解构代码。

### 十、插件测试

仓库中的transform单测用一个简单插件验证插件机制：

```ts
const plugin = node => {
  if (node.type === NodeTypes.TEXT) {
    node.content += 'IamZJT';
  }
};

transform(ast, {
  nodeTransforms: [plugin]
});
```

插件执行以后，text节点内容会发生变化。这说明`transform`本身并不限制插件必须做什么，它只负责提供遍历时机和上下文。

### end

`transform`完成以后，AST已经从“语法树”变成了“可以生成代码的树”：

```text
插值 -> _ctx表达式 + toDisplayString helper
element -> createElementVNode调用信息
相邻文本 -> compound expression
根节点 -> codegenNode
```

下一篇继续看`generate`如何把这些结构拼成真正的render函数代码。
