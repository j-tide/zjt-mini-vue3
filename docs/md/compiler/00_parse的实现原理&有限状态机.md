# 00_parse的实现原理&有限状态机

## 1. 有限状态机(finite state machine)

读取一组输入，然后根据这些输入，来更改为不同的状态。

<img src="https://iamzjt-1256754140.cos.ap-nanjing.myqcloud.com/images/202303230627219.png" width="600" alt="有限状态机"/>

## 2. parse的实现原理与有限状态机

`compiler`的第一步不是直接生成`render`函数，而是先把字符串形式的`template`解析成一棵`AST`。

例如下面这段模板：

```vue
<div>hi, {{ message }}</div>
```

解析以后，大致会得到：

```text
ROOT
└── ELEMENT div
    ├── TEXT hi,
    └── INTERPOLATION message
```

这个过程本质上就是不断读取`context.source`，然后根据当前输入的开头决定下一步进入哪一种状态：

1. 以`{{`开头，进入插值解析；
2. 以`<`开头并且后面是字母，进入element解析；
3. 其它情况，进入text解析；
4. 当前字符串为空，解析结束。

对应到代码，就是`parseChildren`中的判断：

```ts
function parseChildren(context, ancestors) {
  const nodes = [];

  while (!isEnd(context, ancestors)) {
    let node;
    const s = context.source;

    if (s.startsWith('{{')) {
      node = parseInterpolation(context);
    } else if (s[0] === '<') {
      if (/[a-z]/i.test(s[1])) {
        node = parseElement(context, ancestors);
      }
    }

    if (!node) {
      node = parseText(context);
    }

    nodes.push(node);
  }

  return nodes;
}
```

这里的`context.source`就是一个不断缩短的输入串。每解析完一部分，就通过`advanceBy`把已经消费掉的内容移除：

```ts
function advanceBy(context, length) {
  context.source = context.source.slice(length);
}
```

所以，`parse`并不是一次性用一个大正则把整个模板匹配出来，而是：

```text
读取当前输入
    ↓
判断当前状态
    ↓
解析当前节点
    ↓
推进source
    ↓
继续解析下一个节点
```

### （一）插值状态

当输入以`{{`开头时，先消费`{{`，找到下一个`}}`，中间部分就是表达式：

```ts
const closeIndex = context.source.indexOf('}}', 2);

advanceBy(context, 2);
const rawContent = parseTextData(
  context,
  closeIndex - 2
);
const content = rawContent.trim();
advanceBy(context, 2);
```

`{{ message }}`最终会被表示成`INTERPOLATION`节点，内部再嵌套一个`SIMPLE_EXPRESSION`节点。

### （二）element状态

当输入以`<`和字母开头时，先读取开始标签，然后把当前element压入`ancestors`：

```ts
const element = parseTag(context, TagType.start);
ancestors.push(element);
element.children = parseChildren(context, ancestors);
ancestors.pop();
```

这样处理嵌套节点时，子节点解析过程就知道当前处在哪些父节点里面。当遇到结束标签，`isEnd`会从`ancestors`的末尾开始倒序查找，找到对应的父标签后结束当前层级。

如果当前层级结束时没有找到对应的结束标签，就抛出错误：

```ts
throw new Error(
  'Element is missing end tag: ' + element.tag
);
```

### （三）text状态

普通文本的结束位置不是固定的，它可能在插值`{{`之前，也可能在下一个element标签`<`之前。因此`parseText`会分别寻找这两个终止标记，取更靠前的位置：

```ts
const endTokens = ['{{', '<'];
let endIndex = context.source.length;

for (let i = 0; i < endTokens.length; i++) {
  const index = context.source.indexOf(endTokens[i], 1);
  if (index !== -1 && endIndex > index) {
    endIndex = index;
  }
}
```

例如`some {{ message }} text`会被拆成三个节点：

1. `TEXT: some `；
2. `INTERPOLATION: message`；
3. `TEXT:  text`。

## 3. 正则表达式的实现原理与有限状态机

在真正写`parse`之前，仓库里还通过一个正则表达式小案例练习了有限状态机。正则表达式本身也可以理解成：读取字符，根据当前状态和字符决定下一步状态。

相关代码在：

```text
packages/vue/examples/regex/index.js
packages/vue/examples/regex/regex.spec.js
```

这个案例的意义不在于替代JavaScript原生正则，而是先把“输入、状态、状态迁移”这件事情想清楚。回到`compiler`以后，`context.source`就是输入，`parseInterpolation`、`parseElement`和`parseText`就是不同的解析分支。

### end

`parse`阶段只负责把模板字符串转换成`AST`，还不会关心最终要调用哪个运行时函数。下一步的`transform`会遍历这棵树，给节点补充代码生成需要的信息。
