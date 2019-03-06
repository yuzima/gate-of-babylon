# ESM 里的循环依赖

前几天一位同事在写 React 组件库时遇到一个由 ESM 循环依赖引起的问题来找我，借此机会就想总结一下 ESM 循环依赖的知识。

ESM 循环依赖的简单来讲就是两个模块在功能上直接或间接的互相依赖彼此。虽然循环依赖并不总是导致问题，但是循环依赖往往会导致两个模块的代码耦合性很高，一处修改就可能导致连锁反应。而一个好的架构通常是在模块间和层级之间实行单向数据流结构的，上级的模块依赖下级的模块，下级的模块入口依赖自己内部。否则在一些大型项目里，尤其是重构的时候，就可能会有问题出现。

当然，在某些时候循环依赖也是需要的，比如树形结构里父节点指向子节点，子节点又会指向父节点。之前另一位同事在开发 `Dialog` babel 在编译时就会输出提醒：

```
Circular dependency: src/components/Dialog/Dialog.tsx -> src/components/Dialog/open.tsx -> src/components/Dialog/Dialog.tsx
```

这里的情况是需要的，因为 `Dialog` 组件上需要暴露 `open` 模块定义的方法，而 `open` 模块需要依赖 `Dialog` 组件对象。

但是在其他情况下，我们应该在 Javascript 里尽量避免循环依赖。

## 问题是怎么引起的

在同步的循环引用依赖时，会出现的问题：

```
             immediate
        ┌-------->-------┐
  ┌-----┴----┐     ┌-----┴----┐
  | module A |     | module B |
  └-----┬----┘     └-----┬----┘
        └--------<-------┘
             immediate
```

### RangeError: exceeding call stack

当 import 形成了同步函数立即调用的循环时，就会发生这种情况

```javascript
// A.js
import B from './B';

export default () => 3 + B();

// B.js
import A from './A';

export default () => 4 + A();

// index.js
import A from './A';

A(); // RangeError: Maximum call stack size exceeded
```

### import 值为 undefined

表达式和函数都是如此，我遇到的情况就是如此，同事在写 `DatePicker` 组件时需要 import `Input` 组件， 最后编译出来写 Demo 的时候，React 报错提示组件为 undefined，出错位置就在 `Input` 上。

这时候我才留意到 Babel 在 build 时的循环依赖提示，由于之前有其他同事的组件需要循环依赖，因此让我疏忽了。

和 `Dialog` 的情况不同的是 `DatePicker` 的循环依赖路径为：

```
Circular dependency: src/components/DatePicker/DatePicker.tsx -> src/index.ts -> src/components/index.ts -> src/components/DatePicker/index.ts -> src/components/DatePicker/DatePicker.tsx
```

为什么是这样的循环依赖呢，这是因为在 `DatePicker` 里 improt 的 `Input` 组件是从 **src/index.ts** 的输出里取的，而不是直接从 Input 的目录里取的，而同时 **src/index.ts** 也 export `DatePicker` 组件，结构如下所示。

```javascript
// src/components/DatePicker/DatePicker.tsx
import { Input } from '../../index'
class DatePicker extends Component {
  ...
}
export default DatePicker

// src/index.ts
export * from './components'

// src/components/index.ts
import { Input } from './Input'
import { DatePicker } from './DatePicker'

export {
  Input,
  DatePicker
}

// src/components/DatePicker/index.ts
export { default } from './DatePicker'
```

这样导致的问题就是在 **src/index.ts** 的输出依赖 `DatePicker` 组件，但 `DatePicker` 组件又依赖从 **src/index.ts** 获得 `Input` 组件，这个时候 **src/index.ts** 还没有准备好，因此导致引入的 `Input` 组件是 undefined。

## 什么样的情况不会引起问题

如果循环依赖的是异步的函数调用，这样的情况就不会引起问题，因为引用只是指向函数但是并没有立即调用

```
              delayed
        ┌-------->-------┐
  ┌-----┴----┐     ┌-----┴----┐
  | module A |     | module B |
  └-----┬----┘     └-----┬----┘
        └--------<-------┘
             immediate
```

比如说循环依赖的函数通过 DOM 监听事件触发

```javascript
// A.js
import B from './B';

export default () => {
  console.log('A called');
  document.addEventListener('click', B, false);
};

// B.js
import A from './A';

export default () => {
  console.log('B called');
  A();
};

// index.js
import A from './A';

A(); // right away : A called, after click : B called, A called
```

或者依赖的 class 不需要被立即执行

```javascript
// A.js
import B from './B';

export default class A {
  static getB() {
    return new B();
  }
};

// B.js
import A from './A';

export default class B {
  constructor() {
    this.a = new A();
  }
};

// index.js
import A from './A';

console.log(A.getB().a); // instance of A
```

## 依赖分析

### 从代码中抽取 imports 信息

我们可以通过使用 `analyze-es6-modules` 这个包来静态分析依赖关系。

```bash
yarn add analyze-es6-modules -D
```

引入必要的配置

```javascript
const analyzeModules = require('analyze-es6-modules')

const configuration = {
  cwd: 'app/assets/javascripts', // js 目录
  sources: ['**/*.js'], // globbing patterns 匹配文件名
  babel: {
    plugins: [
      require('babel-plugin-syntax-jsx'), // babel-plugin 用于转换非标准的语法
      require('babel-plugin-syntax-flow'),
      require('babel-plugin-syntax-object-rest-spread'),
    ],
  },
}

const resolvedHandler = ({ modules }) => {
  // do something with extracted modules
}

const rejectedHandler = () => {
  console.log('rejected!')
}

analyzeModules(configuration).then(resolvedHandler, rejectedHandler)
```

从 yarn.lock 可以获取所有以 `babel-plugin-syntax` 开头的包

```bash
egrep '^babel-plugin-syntax' yarn.lock
```

接下来是构建一个表示数据，将每个 import 语句都表示为一个有序对

```javascript
const collectDependencies = (modules) => {
  const dependencySet = new Set()
  const separator = ','

  modules.forEach(({ path, imports }) => {
    const importingPath = path

    imports.forEach(({ exportingModule }) => {
      const exportingPath = exportingModule.resolved
      const dependency = [importingPath, exportingPath].join(separator)

      dependencySet.add(dependency)
    })
  })

  return Array.from(dependencySet.values()).map(it => it.split(separator))
}
```

### 构建有向图表

收集完所有的独一无二的有序对后，这些数据可以被认为是有向图表的边界点。

```javascript
const buildDirectedGraphFromEdges = (edges) => {
  return edges.reduce((graph, [sourceNode, targetNode]) => {
    graph[sourceNode] = graph[sourceNode] || new Set()
    graph[sourceNode].add(targetNode)

    return graph
  }, {})
}
```

### 最小化图表只包含循环依赖

到这个时候图表会包含所有的模块，而我们只需要那些涉及到循环依赖的模块。所以大部分不涉及的模块就可以被移除，这些边界点有一个共同的特性：他们至少在一个终点是终端。当一个模块不被任何地方 import，那它就是起始边界点的终端。一个终端点就是不被任何地方 import 的模块。

```javascript
const without = (firstSet, secondSet) => (
  new Set(Array.from(firstSet).filter(it => !secondSet.has(it)))
)

const mergeSets = (sets) => {
  const sumSet = new Set()
  sets.forEach((set) => {
    Array.from(set.values()).forEach((value) => {
      sumSet.add(value)
    })
  })
  return sumSet
}

const stripTerminalNodes = (graph) => {
  const allSources = new Set(Object.keys(graph))
  const allTargets = mergeSets(Object.values(graph))

  const terminalSources = without(allSources, allTargets)
  const terminalTargets = without(allTargets, allSources)

  const newGraph = Object.entries(graph).reduce((smallerGraph, [source, targets]) => {
    if (!terminalSources.has(source)) {
      const nonTerminalTargets = without(targets, terminalTargets)

      if (nonTerminalTargets.size > 0) {
        smallerGraph[source] = nonTerminalTargets
      }
    }

    return smallerGraph
  }, {})

  return newGraph
}
```

这个步骤可以被重复直到图表不能变得更小为止

```javascript
const calculateGraphSize = (graph) => mergeSets(Object.values(graph)).size

const miminizeGraph = (graph) => {
  const smallerGraph = stripTerminalNodes(graph)

  if (calculateGraphSize(smallerGraph) < calculateGraphSize(graph)) {
    return miminizeGraph(smallerGraph)
  } else {
    return smallerGraph
  }
}
```

### 其他的替代方案

如果你不需要生成图片来查看，babel 会在编译时分析模块依赖并输出循环依赖的提示，或者使用 webpack 的 `circular-dependency-plugin`，在 webpack.config.js 的 plugins 里添加这个插件，webpack 会在构建时输出循环依赖的警告。

```bash
WARNING in Circular dependency detected:
app/assets/javascripts/process_allocations.js -> app/assets/javascripts/processes.js -> app/assets/javascripts/process_allocations.js
```

还有一种选择就是在 eslint 里使用来自 `eslint-plugin-dependencies` 的 `dependencies/no-cycles` 规则，或是 `eslint-plugin-import` 的 `import/no-cycle` 规则。两者都能实现需求，但是前者在长的循环的速度上要明显快很多，而且输出结果也更容易分析，非常类似于前面的 webpack 插件。

## 解决有问题的循环依赖

在已经发现问题所在的情况下，可以来谈谈如果解决。解决方法根据问题的不同也不同，不过我们可以遵守下面的策略：将有问题的 export 移动到另一个文件，最好是一个终端文件这样就可以被任何地方导入而不会引起循环；引入事件触发和处理函数来缓解循环；通过注入依赖来反转依赖关系。

思考下面的情况，我们使用最后一条策略来处理这个例子

```javascript

2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
35
36
37
38
39
40
41
// A.js
import B from './B';

export default class A {
  foo() {
    return 'A:foo:' + this.getB().bar();
  }

  bar() {
    return 'A:bar';
  }

  getB() {
    if (!this.b) this.b = new B();
    return this.b;
  }
};

// B.js
import A from './A';

export default class B {
  foo() {
    return 'B:foo:' + this.getA().bar();
  }

  bar() {
    return 'B:bar';
  }

  getA() {
    if (!this.a) this.a = new A();
    return this.a;
  }
};

// index.js
import A from './A';
import B from './B';

console.log(new A().foo() + new B().foo()); // A:foo:B:barB:foo:A:bar
```

循环依赖可以通过将其中一个 class 注入到另一个的 constructor 中被移除掉

```javascript
// A.js
import B from './B';

export default class A {
  foo() {
    return 'A:foo:' + this.getB().bar();
  }

  bar() {
    return 'A:bar';
  }

  getB() {
    if (!this.b) this.b = new B(this);
    return this.b;
  }
};

// B.js
export default class B {
  constructor(a) { // the main change
    this.a = a;
  }

  foo() {
    return 'B:foo:' + this.getA().bar();
  }

  bar() {
    return 'B:bar';
  }

  getA() {
    return this.a;
  }
};

// index.js
import A from './A';
import B from './B';

const a = new A();
const b = new B(a);

console.log(a.foo() + b.foo()); // A:foo:A:barB:foo:A:bar
```

