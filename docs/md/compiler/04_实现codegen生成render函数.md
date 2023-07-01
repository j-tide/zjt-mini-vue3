# 04_实现codegen生成render函数

### 一、codegen的目标

`parse`和`transform`完成以后，AST中已经有了节点类型、表达式、VNode调用信息和helper列表。但运行时不能直接执行这棵AST，还需要把它转换成JavaScript代码。

`codegen`的目标可以简单写成：

```text
AST
  -> JavaScript字符串
  -> new Function
  -> render函数
```

当前的生成结果大致是：

```ts
const { toDisplayString: _toDisplayString } = Vue
return function render (_ctx, _cache) {
  return _toDisplayString(_ctx.message)
}
```

### 二、创建codegen上下文

代码生成过程中会不断拼接字符串，因此先建立一个上下文：

```ts
function createCodegenContext() {
  const context = {
    code: '',
    push(source) {
      context.code += source;
    },
    helper(key) {
      return '_' + helperNameMap[key];
    }
  };

  return context;
}
```

这里的上下文只有两个核心能力：

1. `push`负责追加代码；
2. `helper`负责把内部的Symbol转换成生成代码中的函数名。

### 三、runtime helper的名字映射

transform阶段使用Symbol表示helper：

```ts
export const TO_DISPLAY_STRING = Symbol('toDisplayString');
export const CREATE_ELEMENT_VNODE = Symbol('createElementVNode');

export const helperNameMap = {
  [TO_DISPLAY_STRING]: 'toDisplayString',
  [CREATE_ELEMENT_VNODE]: 'createElementVNode'
};
```

代码生成时再通过`helperNameMap`得到字符串：

```ts
helper(TO_DISPLAY_STRING)
// _toDisplayString
```

使用Symbol而不是直接到处写字符串，可以减少不同阶段之间的硬编码耦合。

### 四、generate生成函数外壳

`generate`先生成helper前置代码，再生成render函数：

```ts
export function generate(ast) {
  const context = createCodegenContext();
  const { push } = context;

  getFunctionPreamble(push, ast);

  const functionName = 'render';
  const args = ['_ctx', '_cache'];
  const signature = args.join(', ');

  push('function ' + functionName + ' (' + signature + ') {');
  push('return ');
  genNode(ast.codegenNode, context);
  push('}');

  return {
    code: context.code
  };
}
```

最终返回的不是函数，而是一个包含`code`字段的对象。后面由`vue`入口把这段代码交给`new Function`执行。

### 五、生成helper前置代码

`getFunctionPreamble`根据`ast.helpers`生成helper解构：

```ts
function getFunctionPreamble(push, ast) {
  const VueBinding = 'Vue';
  const aliasHelper = s =>
    helperNameMap[s] + ': _' + helperNameMap[s];

  if (ast.helpers.length > 0) {
    push(
      'const { ' +
      ast.helpers.map(aliasHelper).join(', ') +
      ' } = ' +
      VueBinding
    );
    push('\n');
  }

  push('return ');
}
```

例如`ast.helpers`包含`TO_DISPLAY_STRING`时，会得到：

```ts
const { toDisplayString: _toDisplayString } = Vue
return function render ...
```

这里的`Vue`只是`new Function`的参数名，并不是固定的浏览器全局变量。

### 六、genNode按节点类型分发

`genNode`是代码生成阶段的分发中心：

```ts
function genNode(node, context) {
  switch (node.type) {
    case NodeTypes.TEXT:
      genText(node, context);
      break;
    case NodeTypes.INTERPOLATION:
      genInterpolation(node, context);
      break;
    case NodeTypes.SIMPLE_EXPRESSION:
      genExpression(node, context);
      break;
    case NodeTypes.ELEMENT:
      genElement(node, context);
      break;
    case NodeTypes.COMPOUND_EXPRESSION:
      genCompoundExpression(node, context);
      break;
  }
}
```

parse阶段定义的节点类型，在这里都有对应的生成函数。

### 七、生成text和expression

text直接生成字符串：

```ts
function genText(node, context) {
  const { push } = context;
  push("'" + node.content + "'");
}
```

简单表达式则直接把内容写入代码：

```ts
function genExpression(node, context) {
  const { push } = context;
  push(node.content);
}
```

经过`transformsExpression`处理以后，插值中的内容已经是`_ctx.message`，因此这里不需要再次判断作用域。

### 八、生成interpolation

插值需要调用运行时helper：

```ts
function genInterpolation(node, context) {
  const { push, helper } = context;

  push(helper(TO_DISPLAY_STRING) + '(');
  genNode(node.content, context);
  push(')');
}
```

输入：

```vue
{{ message }}
```

输出表达式：

```ts
_toDisplayString(_ctx.message)
```

### 九、生成element

element会被生成成`createElementVNode`调用：

```ts
function genElement(node, context) {
  const { push, helper } = context;
  const { tag, props, children } = node;

  push('(' + helper(CREATE_ELEMENT_VNODE) + ')(');
  genNodeList(getNullable([tag, props, children]), context);
  push(')');
}
```

`getNullable`用于补齐参数位置：

```ts
function getNullable(args) {
  return args.map(arg => arg || 'null');
}
```

例如没有props的element也要保留第二个参数：

```ts
_createElementVNode('div', null, 'hello')
```

参数列表由`genNodeList`负责生成：

```ts
function genNodeList(nodes, context) {
  const { push } = context;

  for (let i = 0; i < nodes.length; i++) {
    const node = nodes[i];

    if (isString(node)) {
      push(node);
    } else {
      genNode(node, context);
    }

    if (i < nodes.length - 1) {
      push(', ');
    }
  }
}
```

### 十、生成复合表达式

`transformText`生成的复合表达式中，`children`既可能是AST节点，也可能是字符串` + `：

```ts
function genCompoundExpression(node, context) {
  const { push } = context;
  const { children } = node;

  for (let i = 0; i < children.length; i++) {
    const child = children[i];

    if (isString(child)) {
      push(child);
    } else {
      genNode(child, context);
    }
  }
}
```

所以：

```text
[TEXT, " + ", INTERPOLATION]
```

会生成：

```ts
'hi, ' + _toDisplayString(_ctx.message)
```

### 十一、单测快照

纯文本：

```ts
const ast = baseParse('hi');
transform(ast);
const { code } = generate(ast);
```

生成结果：

```ts
return function render (_ctx, _cache) {return 'hi'}
```

插值：

```ts
const ast = baseParse('{{ message }}');
transform(ast, {
  nodeTransforms: [transformsExpression]
});
```

生成结果：

```ts
const { toDisplayString: _toDisplayString } = Vue
return function render (_ctx, _cache) {
  return _toDisplayString(_ctx.message)
}
```

综合案例：

```ts
const ast = baseParse('<div>hi, {{ message }}</div>');
transform(ast, {
  nodeTransforms: [
    transformsExpression,
    transformElement,
    transformText
  ]
});
```

生成结果：

```ts
const { toDisplayString: _toDisplayString, createElementVNode: _createElementVNode } = Vue
return function render (_ctx, _cache) {
  return (_createElementVNode)(
    'div',
    null,
    'hi, ' + _toDisplayString(_ctx.message)
  )
}
```

### end

`codegen`做的事情可以概括为：

```text
节点类型 -> 对应生成函数
AST节点 -> JavaScript字符串
helper列表 -> 前置解构
```

下一篇把`baseCompile`和`runtime-core`接起来，完整走一遍`template -> render -> vnode -> DOM`。
