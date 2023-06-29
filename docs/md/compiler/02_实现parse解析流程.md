# 02_实现parse解析流程

### 一、先从单测确定AST结构

`parse`的职责是把字符串模板转换成AST，所以第一步不是急着写解析器，而是先确定不同模板应该得到什么结构。

插值的单测：

```ts
it('simple interpolation', () => {
  const ast = baseParse('{{ message }}');

  expect(ast.children[0]).toStrictEqual({
    type: NodeTypes.INTERPOLATION,
    content: {
      type: NodeTypes.SIMPLE_EXPRESSION,
      content: 'message'
    }
  });
});
```

element的单测：

```ts
it('simple div', () => {
  const ast = baseParse('<div>hello</div>');

  expect(ast.children[0]).toStrictEqual({
    type: NodeTypes.ELEMENT,
    tag: 'div',
    children: [
      {
        type: NodeTypes.TEXT,
        content: 'hello'
      }
    ]
  });
});
```

当前`AST`使用`NodeTypes`区分节点：

```ts
export const enum NodeTypes {
  ROOT,
  INTERPOLATION,
  SIMPLE_EXPRESSION,
  ELEMENT,
  TEXT,
  COMPOUND_EXPRESSION
}
```

### 二、baseParse建立解析上下文

`baseParse`先建立上下文，再解析根节点：

```ts
export function baseParse(content: string) {
  const context = createParseContext(content);

  return createRoot(parseChildren(context, []));
}

function createParseContext(content: string) {
  return {
    source: content
  };
}

function createRoot(children) {
  return {
    children,
    type: NodeTypes.ROOT
  };
}
```

当前上下文只有一个`source`字段。虽然看起来很简单，但把输入放在上下文对象里以后，后续增加偏移量、错误信息和其它解析状态会更容易。

### 三、parseChildren是总调度

`parseChildren`会循环读取输入，直到当前层级结束：

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

这里的判断顺序不能随便调：

1. 先判断插值，避免把`{{ message }}`当成普通文本；
2. 再判断element，避免把标签内容整体当成文本；
3. 最后使用text兜底，消费当前无法被前两种规则识别的内容。

每一个解析函数都必须推进`context.source`，否则循环会一直停留在同一个位置。

### 四、解析插值

插值的固定结构是`{{ expression }}`，因此可以先找到结束分隔符`}}`：

```ts
function parseInterpolation(context) {
  const openDelimiter = '{{';
  const closeDelimiter = '}}';

  const closeIndex = context.source.indexOf(
    closeDelimiter,
    openDelimiter.length
  );

  advanceBy(context, openDelimiter.length);

  const rawContentLength =
    closeIndex - openDelimiter.length;
  const rawContent = parseTextData(
    context,
    rawContentLength
  );

  const content = rawContent.trim();
  advanceBy(context, closeDelimiter.length);

  return {
    type: NodeTypes.INTERPOLATION,
    content: {
      type: NodeTypes.SIMPLE_EXPRESSION,
      content
    }
  };
}
```

这里的`trim`很重要。模板中的`{{ message }}`和`{{message}}`，表达式内容都应该是`message`，而不是带有多余空格的字符串。

### 五、解析element和嵌套关系

解析element时，先解析开始标签：

```ts
function parseElement(context, ancestors) {
  const element = parseTag(context, TagType.start);

  ancestors.push(element);
  element.children = parseChildren(context, ancestors);
  ancestors.pop();

  if (!startsWidthEndTagOpen(context.source, element.tag)) {
    throw new Error(
      'Element is missing end tag: ' + element.tag
    );
  } else {
    parseTag(context, TagType.end);
  }

  return element;
}
```

`ancestors`是处理嵌套element的关键。例如解析：

```vue
<div><p>hi</p>{{ message }}</div>
```

解析`div`时，`ancestors`是：

```text
[]
  -> [div]
      -> [div, p]
      -> [div]
  -> []
```

子节点解析结束以后，当前element必须找到对应的结束标签，否则说明模板结构不完整。

### 六、如何判断当前层级结束

`isEnd`有两个结束条件：

1. `source`已经为空；
2. 当前输入是某个祖先节点的结束标签。

```ts
function isEnd(context, ancestors) {
  const s = context.source;

  if (s.startsWith('</')) {
    for (let i = ancestors.length - 1; i >= 0; i--) {
      const tag = ancestors[i].tag;
      if (startsWidthEndTagOpen(s, tag)) {
        return true;
      }
    }
  }

  return !s;
}
```

这里要从`ancestors`的末尾向前查找，因为离当前节点最近的祖先最有可能匹配当前结束标签。

### 七、parseTag解析开始和结束标签

开始标签和结束标签使用同一个`parseTag`，通过`TagType`区分：

```ts
const enum TagType {
  start,
  end
}

function parseTag(context, type: TagType) {
  const match = /^<\/?([a-z]*)/i.exec(context.source);
  const tag = match[1];

  advanceBy(context, match[0].length);
  advanceBy(context, 1);

  if (type === TagType.end) return;

  return {
    type: NodeTypes.ELEMENT,
    tag
  };
}
```

解析完成以后，通过两次`advanceBy`消费标签本身和标签末尾的`>`。

当前版本只解析简单的标签名，没有处理属性、自闭合标签和指令。

### 八、解析text

普通文本的结束位置由下一个插值或下一个标签决定：

```ts
function parseText(context) {
  const endTokens = ['{{', '<'];
  let endIndex = context.source.length;

  for (let i = 0; i < endTokens.length; i++) {
    const index = context.source.indexOf(endTokens[i], 1);
    if (index !== -1 && endIndex > index) {
      endIndex = index;
    }
  }

  const content = parseTextData(context, endIndex);

  return {
    type: NodeTypes.TEXT,
    content
  };
}
```

公共的字符串消费逻辑被抽成`parseTextData`：

```ts
function parseTextData(context, length) {
  const content = context.source.slice(0, length);
  advanceBy(context, length);
  return content;
}
```

### 九、综合案例

输入：

```vue
<div>hi, {{ message }}</div>
```

最终AST可以理解成：

```text
ROOT
└── ELEMENT div
    ├── TEXT "hi, "
    └── INTERPOLATION
        └── SIMPLE_EXPRESSION "message"
```

对应的单测会同时覆盖element、text和interpolation：

```ts
const ast = baseParse('<div>hi, {{ message }}</div>');
```

这也是后续`transform`和`codegen`使用的输入。

### 十、当前parse的边界

当前解析器已经覆盖了提交记录中的三个基础类型和嵌套场景，但仍然是学习版：

- 只识别简单的字母标签名；
- 不解析属性；
- 不支持自闭合标签；
- 不处理注释；
- 缺少更完整的错误位置和错误恢复。

先通过TDD把基础节点拆出来，后续每增加一种语法，再增加对应的AST结构和单测即可。

### end

`parse`完成以后，模板已经从字符串变成了结构化AST。下一篇不再读取字符串，而是通过插件遍历AST，给节点补充表达式、VNode调用和复合文本信息。
