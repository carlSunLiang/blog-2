
#AST语法结构树初学者完整教程

##编写你的第一个 Babel 插件

不太喜欢上来就讲大道理，先来个小栗子，做个简单而又实用的功能，做完后，理论你就理解一大半了。

我们需要antd里面的一个组件Button，代码如下：
```
import { Button } from 'antd'
```

我们只想引入Button组件，现在却导入antd/lib/index.js所有组件，导致打包后文件大了好多，这显然不是我们想要的。

我们想要的是只引入antd/lib/button
```
import Button from 'antd/lib/button'
```

但是这样写又过于麻烦，多个组件得写很多行，每行都有个相同的antd/lib代码冗余，又不雅观又不好维护。

我们可以使用Babel插件来解决这个问题。（[完整的代码示例][1]）

 1. 建立好如下目录结构
    ```
    - root
        - node_modules
            - babel-plugin-import
                - index.js
                - package.json
        - test.js
    ```

 2. test.js文件内容:
    ```
    const babel = require('babel-core');
    // 转换前的代码
    const code = `import { Button } from 'antd'`;

    var result = babel.transform(code, {
      plugins:["import"]
    });

    // 转换后的代码
    console.log(result.code);
    // 输出：import Button from 'antd/lib/button'
    ```

    babel-core是babel的核心包，transform函数可以传入code,返回转换后的代码，ast语法树和source-map映射地图。具体API参考 [babel-core][2]

 3. package.json的main属性指向index.js，不解释
 4. 好了，开始编写我们的插件吧,index.js文件内容：
    ```
    /* 单词首字母小写 */
    const toLowerCase = word =>
      Array.from(word).map((char, index) =>
        !index ? char.toLowerCase() : char).join('')

    module.exports = ({ types }) => (
      {
        // 插件入口
        visitor: {
          // 处理类型： Import声明
          ImportDeclaration(path) {
            const { node } = path
            const { type, specifiers, source } = node
            // 过滤掉默认引用，如 import antd from 'antd'
            const importSpecifiers = specifiers.filter(specifier =>
                types.isImportSpecifier(specifier))
            // 例子只处理了一个组件的引用
            if (importSpecifiers.length === 1) {
              const [importSpecifier] = importSpecifiers
              // 小写处理组件引用，如Import { Button }，改为: antd/lib/button
              const element = toLowerCase(importSpecifier.imported.name)
              // 引入改为默认引入，如Import { Button }, 改为: import Button
              importSpecifier.type = 'ImportDefaultSpecifier'
              source.value += `/lib/${element}`
            }
          }
        }
      })
    ```
    短短的几行代码，即使看了注释，相信亲爱的读者仍然一头雾水。请登录 [ast语法树在线生成网站][3]，输入`import Button from 'antd'`,对比右侧生成的语法树与代码，**我们从代码开始讲起**。

    - 代码中ImportDeclaration代表只处理代码中`import`这样的代码（请注意网站生成的语法树）。
    - 我们拿到specifiers是引用声明，就是`import { Button }`中的Button，并且判断长度为1（长度大于1的话得多行引入，要修改语法树结构）。做了以下两件事：

        - 拿到Button ,转成小写，拼接到`source.value`也就是`'antd'`后面。
        - 把Specifier类型为`ImportSpecifier`也就是`{ Button }`改为`ImportDefaultSpecifier`，这样就把 `Import { Button }`改成了`Import Button`。

    OK，就这么简单，也许你还没有明白，没关系，联系我吧。（mail: hongji_12313@163.com）

    然后我们再来看理论（理论的例子全部在[ast][4]仓库中）。

##基础

Babel 是 JavaScript 静态分析编译器，更确切地说是源码到源码的编译器，通常也叫做“转换编译器（transpiler）”。 意思是说你为 Babel 提供一些 JavaScript 代码，Babel 更改这些代码，然后返回给你新生成的代码。

    静态分析是在不需要执行代码的前提下对代码进行分析的处理过程 （执行代码的同时进行代码分析即是动态分析）。 静态分析的目的是多种多样的， 它可用于语法检查，编译，代码高亮，代码转换，优化，压缩等等场景。

###抽象语法树（ASTs）
这个处理过程中的每一步都涉及到创建或是操作抽象语法树，亦称 AST。

Babel 使用一个基于 ESTree 并修改过的 AST，它的内核说明文档可以在这里找到。
```
    function square(n) {
      return n * n;
    }
```
AST Explorer 可以让你对 AST 节点有一个更好的感性认识。 这里是上述代码的一个示例链接。

同样的程序可以表述为下面的列表：

```
- FunctionDeclaration:
  - id:
    - Identifier:
      - name: square
  - params [1]
    - Identifier
      - name: n
  - body:
    - BlockStatement
      - body [1]
        - ReturnStatement
          - argument
            - BinaryExpression
              - operator: *
              - left
                - Identifier
                  - name: n
              - right
                - Identifier
                  - name: n
```

或是如下所示的 JavaScript Object（对象）：

```
{
  type: "FunctionDeclaration",
  id: {
    type: "Identifier",
    name: "square"
  },
  params: [{
    type: "Identifier",
    name: "n"
  }],
  body: {
    type: "BlockStatement",
    body: [{
      type: "ReturnStatement",
      argument: {
        type: "BinaryExpression",
        operator: "*",
        left: {
          type: "Identifier",
          name: "n"
        },
        right: {
          type: "Identifier",
          name: "n"
        }
      }
    }]
  }
}
```

你会留意到 AST 的每一层都拥有相同的结构：

```
{
  type: "FunctionDeclaration",
  id: {...},
  params: [...],
  body: {...}
}
{
  type: "Identifier",
  name: ...
}
{
  type: "BinaryExpression",
  operator: ...,
  left: {...},
  right: {...}
}
```
    注意：出于简化的目的移除了某些属性
这样的每一层结构也被叫做 节点（Node）。 一个 AST 可以由单一的节点或是成百上千个节点构成。 它们组合在一起可以描述用于静态分析的程序语法。

每一个节点都有如下所示的接口（Interface）：
```
interface Node {
  type: string;
}
```
字符串形式的 type 字段表示节点的类型（如： "FunctionDeclaration"，"Identifier"，或 "BinaryExpression"）。 每一种类型的节点定义了一些附加属性用来进一步描述该节点类型。

Babel 还为每个节点额外生成了一些属性，用于描述该节点在原始代码中的位置。
```
{
  type: ...,
  start: 0,
  end: 38,
  loc: {
    start: {
      line: 1,
      column: 0
    },
    end: {
      line: 3,
      column: 1
    }
  },
  ...
}
```
每一个节点都会有 start，end，loc 这几个属性。

###Babel 的处理步骤

Babel 的三个主要处理步骤分别是： 解析（parse），转换（transform），生成（generate）。.

####解析

解析步骤接收代码并输出 AST。 这个步骤分为两个阶段：*词法分析（Lexical Analysis） *和 语法分析（Syntactic Analysis）。.

#####词法分析

词法分析阶段把字符串形式的代码转换为 令牌（tokens） 流。.

你可以把令牌看作是一个扁平的语法片段数组：
```
n * n;
[
  { type: { ... }, value: "n", start: 0, end: 1, loc: { ... } },
  { type: { ... }, value: "*", start: 2, end: 3, loc: { ... } },
  { type: { ... }, value: "n", start: 4, end: 5, loc: { ... } },
  ...
]
```
每一个 type 有一组属性来描述该令牌：
```
{
  type: {
    label: 'name',
    keyword: undefined,
    beforeExpr: false,
    startsExpr: true,
    rightAssociative: false,
    isLoop: false,
    isAssign: false,
    prefix: false,
    postfix: false,
    binop: null,
    updateContext: null
  },
  ...
}
```
和 AST 节点一样它们也有 start，end，loc 属性。.

#####语法分析

语法分析阶段会把一个令牌流转换成 AST 的形式。 这个阶段会使用令牌中的信息把它们转换成一个 AST 的表述结构，这样更易于后续的操作。

####转换

转换步骤接收 AST 并对其进行遍历，在此过程中对节点进行添加、更新及移除等操作。 这是 Babel 或是其他编译器中最复杂的过程 同时也是插件将要介入工作的部分，这将是本手册的主要内容， 因此让我们慢慢来。

####生成

代码生成步骤把最终（经过一系列转换之后）的 AST转换成字符串形式的代码，同时创建源码映射（source maps）。.

代码生成其实很简单：深度优先遍历整个 AST，然后构建可以表示转换后代码的字符串。

###遍历

想要转换 AST 你需要进行递归的树形遍历。

比方说我们有一个 FunctionDeclaration 类型。它有几个属性：id，params，和 body，每一个都有一些内嵌节点。

```
{
  type: "FunctionDeclaration",
  id: {
    type: "Identifier",
    name: "square"
  },
  params: [{
    type: "Identifier",
    name: "n"
  }],
  body: {
    type: "BlockStatement",
    body: [{
      type: "ReturnStatement",
      argument: {
        type: "BinaryExpression",
        operator: "*",
        left: {
          type: "Identifier",
          name: "n"
        },
        right: {
          type: "Identifier",
          name: "n"
        }
      }
    }]
  }
}
```
于是我们从 FunctionDeclaration 开始并且我们知道它的内部属性（即：id，params，body），所以我们依次访问每一个属性及它们的子节点。

接着我们来到 id，它是一个 Identifier。Identifier 没有任何子节点属性，所以我们继续。

之后是 params，由于它是一个数组节点所以我们访问其中的每一个，它们都是 Identifier 类型的单一节点，然后我们继续。

此时我们来到了 body，这是一个 BlockStatement 并且也有一个 body节点，而且也是一个数组节点，我们继续访问其中的每一个。

这里唯一的一个属性是 ReturnStatement 节点，它有一个 argument，我们访问 argument 就找到了 BinaryExpression。.

BinaryExpression 有一个 operator，一个 left，和一个 right。 Operator 不是一个节点，它只是一个值因此我们不用继续向内遍历，我们只需要访问 left 和 right。.

Babel 的转换步骤全都是这样的遍历过程。

###Visitors（访问者）

当我们谈及“进入”一个节点，实际上是说我们在访问它们， 之所以使用这样的术语是因为有一个访问者模式（visitor）的概念。.

访问者是一个用于 AST 遍历的跨语言的模式。 简单的说它们就是一个对象，定义了用于在一个树状结构中获取具体节点的方法。 这么说有些抽象所以让我们来看一个例子。
```
const MyVisitor = {
  Identifier() {
    console.log("Called!");
  }
};
```
    注意： Identifier() { ... } 是 Identifier: { enter() { ... } } 的简写形式。.
这是一个简单的访问者，把它用于遍历中时，每当在树中遇见一个 Identifier 的时候会调用 Identifier() 方法。

所以在下面的代码中 Identifier() 方法会被调用四次（包括 square 在内，总共有四个 Identifier）。).
```
function square(n) {
  return n * n;
}
```
```
Called!
Called!
Called!
Called!
```
这些调用都发生在进入节点时，不过有时候我们也可以在退出时调用访问者方法。.

假设我们有一个树状结构：
```
- FunctionDeclaration
  - Identifier (id)
  - Identifier (params[0])
  - BlockStatement (body)
    - ReturnStatement (body)
      - BinaryExpression (argument)
        - Identifier (left)
        - Identifier (right)
```
当我们向下遍历这颗树的每一个分支时我们最终会走到尽头，于是我们需要往上遍历回去从而获取到下一个节点。 向下遍历这棵树我们进入每个节点，向上遍历回去时我们退出每个节点。

让我们以上面那棵树为例子走一遍这个过程。

- 进入 FunctionDeclaration
    - 进入 Identifier (id)
    - 走到尽头
    - 退出 Identifier (id)
    - 进入 Identifier (params[0])
    - 走到尽头
    - 退出 Identifier (params[0])
    - 进入 BlockStatement (body)
        - 进入 ReturnStatement (body)
            - 进入 BinaryExpression (argument)
                - 进入 Identifier (left)
                - 走到尽头
                - 退出 Identifier (left)
                - 进入 Identifier (right)
                - 走到尽头
                - 退出 Identifier (right)
            - 退出 BinaryExpression (argument)
        - 退出 ReturnStatement (body)
    - 退出 BlockStatement (body)
- 退出 FunctionDeclaration

所以当创建访问者时你实际上有两次机会来访问一个节点。
```
const MyVisitor = {
  Identifier: {
    enter() {
      console.log("Entered!");
    },
    exit() {
      console.log("Exited!");
    }
  }
};
```

###Paths（路径）

AST 通常会有许多节点，那么节点直接如何相互关联？ 我们可以用一个巨大的可变对象让你来操作以及完全访问（节点的关系），或者我们可以用Paths（路径）来简化这件事情。.

Path 是一个对象，它表示两个节点之间的连接。

举例来说如果我们有以下的节点和它的子节点：
```
{
  type: "FunctionDeclaration",
  id: {
    type: "Identifier",
    name: "square"
  },
  ...
}
```
将子节点 Identifier 表示为路径的话，看起来是这样的：
```
{
  "parent": {
    "type": "FunctionDeclaration",
    "id": {...},
    ....
  },
  "node": {
    "type": "Identifier",
    "name": "square"
  }
}
```
同时它还有关于该路径的附加元数据：
```
{
  "parent": {...},
  "node": {...},
  "hub": {...},
  "contexts": [],
  "data": {},
  "shouldSkip": false,
  "shouldStop": false,
  "removed": false,
  "state": null,
  "opts": null,
  "skipKeys": null,
  "parentPath": null,
  "context": null,
  "container": null,
  "listKey": null,
  "inList": false,
  "parentKey": null,
  "key": null,
  "scope": null,
  "type": null,
  "typeAnnotation": null
}
```
当然还有成堆的方法，它们和添加、更新、移动和删除节点有关，不过我们后面再说。

可以这么说，路径是对于节点在数中的位置以及其他各种信息的响应式表述。 当你调用一个方法更改了树的时候，这些信息也会更新。 Babel 帮你管理着这一切从而让你能更轻松的操作节点并且尽量保证无状态化。（译注：意即尽可能少的让你来维护状态）

###Paths in Visitors（存在于访问者中的路径）

当你有一个拥有 Identifier() 方法的访问者时，你实际上是在访问路径而不是节点。 如此一来你可以操作节点的响应式表述（译注：即路径）而不是节点本身。
```
const MyVisitor = {
  Identifier(path) {
    console.log("Visiting: " + path.node.name);
  }
};
```
```
a + b + c;
Visiting: a
Visiting: b
Visiting: c
```
State（状态）

状态是 AST 转换的敌人。状态会不停的找你麻烦，你对状态的预估到最后几乎总是错的，因为你无法预先考虑到所有的语法。

考虑下列代码：
```
function square(n) {
  return n * n;
}
```
让我们写一个把 n 重命名为 x 的访问者的快速实现：.
```
let paramName;

const MyVisitor = {
  FunctionDeclaration(path) {
    const param = path.node.params[0];
    paramName = param.name;
    param.name = "x";
  },

  Identifier(path) {
    if (path.node.name === paramName) {
      path.node.name = "x";
    }
  }
};
```
对上面的代码来说或许能行，但我们很容易就能“搞坏”它：
```
function square(n) {
  return n * n;
}
n;
```
更好的处理方式是递归。那么让我们来像克里斯托佛·诺兰的电影那样来把一个访问者放进另外一个访问者里面。
```
const updateParamNameVisitor = {
  Identifier(path) {
    if (path.node.name === this.paramName) {
      path.node.name = "x";
    }
  }
};

const MyVisitor = {
  FunctionDeclaration(path) {
    const param = path.node.params[0];
    const paramName = param.name;
    param.name = "x";

    path.traverse(updateParamNameVisitor, { paramName });
  }
};
```
当然，这只是一个刻意捏造的例子，不过它演示了如何从访问者中消除全局状态。

###Scopes（作用域）

接下来让我们引入作用域（scope）的概念。 JavaScript 拥有词法作用域，代码块创建新的作用域并形成一个树状结构。
```
// global scope

function scopeOne() {
  // scope 1

  function scopeTwo() {
    // scope 2
  }
}
```
在 JavaScript 中，每当你创建了一个引用，不管是通过变量（variable）、函数（function）、类型（class）、参数（params）、模块导入（import）、标签（label）等等，它都属于当前作用域。
```
var global = "I am in the global scope";

function scopeOne() {
  var one = "I am in the scope created by `scopeOne()`";

  function scopeTwo() {
    var two = "I am in the scope created by `scopeTwo()`";
  }
}
```
处于深层作用域代码可以使用高（外）层作用域的引用。
```
function scopeOne() {
  var one = "I am in the scope created by `scopeOne()`";

  function scopeTwo() {
    one = "I am updating the reference in `scopeOne` inside `scopeTwo`";
  }
}
```
低（内）层作用域也可以创建（和外层）同名的引用而无须更改它。
```
function scopeOne() {
  var one = "I am in the scope created by `scopeOne()`";

  function scopeTwo() {
    var one = "I am creating a new `one` but leaving reference in `scopeOne()` alone.";
  }
}
```

当编写一个转换器时，我们须要小心作用域。我们得确保在改变代码的各个部分时不会破坏它。

我们会想要添加新的引用并且保证它们不会和已经存在的引用冲突。 又或者我们只是想要找出变量在哪里被引用的。 我们需要能在给定作用域内跟踪这些引用。

作用域可以表述为：
```
{
  path: path,
  block: path.node,
  parentBlock: path.parent,
  parent: parentScope,
  bindings: [...]
}
```
当你创建一个新的作用域时需要给它一个路径及父级作用域。之后在遍历过程中它会在改作用于内收集所有的引用（“绑定”）。

这些做好之后，你将拥有许多用于作用域上的方法。我们稍后再讲这些。

Bindings（绑定）

引用从属于特定的作用域；这种关系被称作：绑定（binding）。.
```
function scopeOnce() {
  var ref = "This is a binding";

  ref; // This is a reference to a binding

  function scopeTwo() {
    ref; // This is a reference to a binding from a lower scope
  }
}
```
一个绑定看起来如下：
```
{
  identifier: node,
  scope: scope,
  path: path,
  kind: 'var',

  referenced: true,
  references: 3,
  referencePaths: [path, path, path],

  constant: false,
  constantViolations: [path]
}
```
有了这些信息你就可以查找一个绑定的所有引用，并且知道绑定的类型是什么（参数，定义等等），寻找到它所属的作用域，或者得到它的标识符的拷贝。 你甚至可以知道它是否是一个常量，并查看是哪个路径让它不是一个常量。

知道绑定是否为常量在很多情况下都会很有用，最大的用处就是代码压缩。
```
function scopeOne() {
  var ref1 = "This is a constant binding";

  becauseNothingEverChangesTheValueOf(ref1);

  function scopeTwo() {
    var ref2 = "This is *not* a constant binding";
    ref2 = "Because this changes the value";
  }
}
```

##API

Babel 实际上是一系列的模块。本节我们将探索一些主要的模块，解释它们是做什么的以及如何使用它们。

    注意：本节内容不是详细的 API 文档的替代品，正式的 API 文档将很快提供出来。
###babylon

Babylon 是 Babel 的解析器。最初是 Acorn 的一份 fork，它非常快，易于使用，并且针对非标准特性（以及那些未来的标准特性）设计了一个基于插件的架构。

首先，让我们先安装它。
```
$ npm install --save babylon
```
让我们从解析简单的字符形式代码开始：
```
import * as babylon from "babylon";

const code = `function square(n) {
  return n * n;
}`;

babylon.parse(code);
// Node {
//   type: "File",
//   start: 0,
//   end: 38,
//   loc: SourceLocation {...},
//   program: Node {...},
//   comments: [],
//   tokens: [...]
// }
```

我们还能传递选项给 parse()：

```
babylon.parse(code, {
  sourceType: "module", // default: "script"
  plugins: ["jsx"] // default: []
});
```
sourceType 可以是 "module" 或者 "script"，它表示 Babylon 应该用哪种模式来解析。 "module" 将会在严格模式下解析并且允许模块定义，"script" 则不会。

    注意： sourceType 的默认值是 "script" 并且在发现 import 或 export 时产生错误。 使用 scourceType: "module" 来避免这些错误。

因为 Babylon 使用了基于插件的架构，因此 plugins 选项可以开启内置插件。 注意 Babylon 尚未对外部插件开放此 API 接口，不过未来会开放的。

可以在 Babylon README 查看所有插件的列表。.

###babel-traverse

Babel Tranverse（遍历）模块维护了整棵树的状态，并且负责替换、移除和添加节点。

运行以下命令安装：
```
$ npm install --save babel-traverse
```
我们可以配合 Babylon 一起使用来遍历和更新节点：

```
import * as babylon from "babylon";
import traverse from "babel-traverse";

const code = `function square(n) {
  return n * n;
}`;

const ast = babylon.parse(code);

traverse(ast, {
  enter(path) {
    if (
      path.node.type === "Identifier" &&
      path.node.name === "n"
    ) {
      path.node.name = "x";
    }
  }
});
```

###babel-types

Babel Types（类型）模块是一个用于 AST 节点的 Lodash 式工具库。 译注：Lodash 是一个 JavaScript 函数工具库，提供了基于函数式编程风格的众多工具函数）它包含了构造、验证以及变换 AST 节点的方法。 其设计周到的工具方法有助于编写清晰简单的 AST 逻辑。

运行以下命令来安装它：
```
$ npm install --save babel-types
```
然后如下所示来使用：
```
import traverse from "babel-traverse";
import * as t from "babel-types";

traverse(ast, {
  enter(path) {
    if (t.isIdentifier(path.node, { name: "n" })) {
      path.node.name = "x";
    }
  }
});
```
####Definitions（定义）

Babel Types模块拥有每一个单一类型节点的定义，包括有哪些属性分别属于哪里，哪些值是合法的，如何构建该节点，该节点应该如何去遍历，以及节点的别名等信息。

单一节点类型定义的形式如下：
```
defineType("BinaryExpression", {
  builder: ["operator", "left", "right"],
  fields: {
    operator: {
      validate: assertValueType("string")
    },
    left: {
      validate: assertNodeType("Expression")
    },
    right: {
      validate: assertNodeType("Expression")
    }
  },
  visitor: ["left", "right"],
  aliases: ["Binary", "Expression"]
});
```

####Builders（构建器）

你会注意到上面的 BinaryExpression 定义有一个 builder 字段。.
```
builder: ["operator", "left", "right"]
```
这是由于每一个节点类型都有构建器方法：
```
t.binaryExpression("*", t.identifier("a"), t.identifier("b"));
```
它可以创建如下所示的 AST：
```
{
  type: "BinaryExpression",
  operator: "*",
  left: {
    type: "Identifier",
    name: "a"
  },
  right: {
    type: "Identifier",
    name: "b"
  }
}
```
当打印出来（输出）之后是这样的：
```
a * b
```
构建器还会验证自身创建的节点，并在错误使用的情形下抛出描述性的错误。这就引出了接下来的一种方法。

####Validators（验证器）

BinaryExpression 的定义还包含了节点的 fields 字段信息并且指示了如何验证它们。
```
fields: {
  operator: {
    validate: assertValueType("string")
  },
  left: {
    validate: assertNodeType("Expression")
  },
  right: {
    validate: assertNodeType("Expression")
  }
}
```
这可以用来创建两种类型的验证方法。第一种是 isX。.
```
t.isBinaryExpression(maybeBinaryExpressionNode);
```
此方法用来确保节点是一个二进制表达式，不过你也可以传入第二个参数来确保节点包含特定的属性和值。
```
t.isBinaryExpression(maybeBinaryExpressionNode, { operator: "*" });
```
这些方法还有一种更加，嗯，断言式的版本，会抛出异常而不是返回 true 或 false。.
```
t.assertBinaryExpression(maybeBinaryExpressionNode);
t.assertBinaryExpression(maybeBinaryExpressionNode, { operator: "*" });
// Error: Expected type "BinaryExpression" with option { "operator": "*" }
```
####Converters（变换器）

#####babel-generator

Babel Generator模块是 Babel 的代码生成器。它将 AST 输出为代码并包括源码映射（sourcemaps）。

运行以下命令来安装它：
```
$ npm install --save babel-generator
```
然后如下所示使用：
```
import * as babylon from "babylon";
import generate from "babel-generator";

const code = `function square(n) {
  return n * n;
}`;

const ast = babylon.parse(code);

generate(ast, null, code);
// {
//   code: "...",
//   map: "..."
// }
```
你也可以给 generate() 传递选项。.
```
generate(ast, {
  retainLines: false,
  compact: "auto",
  concise: false,
  quotes: "double",
  // ...
}, code);
```
#####babel-template

Babel Template模块是一个很小但却非常有用的模块。它能让你编写带有占位符的字符串形式的代码，你可以用此来替代大量的手工构建的 AST。
```
$ npm install --save babel-template
import template from "babel-template";
import generate from "babel-generator";
import * as t from "babel-types";

const buildRequire = template(`
  var IMPORT_NAME = require(SOURCE);
`);

const ast = buildRequire({
  IMPORT_NAME: t.identifier("myModule"),
  SOURCE: t.stringLiteral("my-module")
});

console.log(generate(ast).code);
// var myModule = require("my-module");
```


  [1]: https://github.com/antgod/babel-plugin-import
  [2]: https://babeljs.io/docs/core-packages/
  [3]: http://esprima.org/demo/parse.html#
  [4]: https://github.com/antgod/ast