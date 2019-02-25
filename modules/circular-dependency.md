# ESM 里的循环依赖

前几天一位同事在写 React 组件库时遇到一个由 ESM 循环依赖引起的问题来找我，借此机会就想总结一下 ESM 循环依赖的知识。

ESM 循环依赖的简单来讲就是两个模块在功能上直接或间接的互相依赖彼此。虽然循环依赖并不总是导致问题，但是循环依赖往往会导致两个模块的代码耦合性很高，一处修改就可能导致连锁反应。而一个好的架构通常是在模块间和层级之间实行单向数据流结构的，上级的模块依赖下级的模块，下级的模块入口依赖自己内部。否则在一些大型项目里，尤其是重构的时候，就可能会有问题出现。

当然，在某些时候循环依赖也是需要的，比如树形结构里父节点指向子节点，子节点又会指向父节点。之前另一位同事在开发 `Dialog` babel 在编译 时会输出提醒：

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

除了 babel 会在编译时分析模块依赖并输出提示之外。我们还可以通过生成依赖关系图来查看整个项目的依赖关系。

