# 尾递归优化

> 在周会上听到同事们提起尾递归优化，因为对它不了解，就特地去查了一些资料学习了一下

## 什么是尾递归优化

尾递归优化 Proper Tail Calls (PTC) 是 ECMAScript 6 的新功能。添加这个功能是为了让递归开发模式，不管是直接的或者是间接的，变得更容易。很多其他的设计模式也能够从 PTC 从收益，例如包装了某些功能的代码，包装函数直接返回它包装了的结果。通过 PTC，执行代码所需要的内存容量将被减少。对于嵌套很深的递归代码，PTC 能够让原本会抛出 stack overflow 错误的代码得以执行。

通常在调用函数的时候，stack 空间会被分配给函数调用相关的数据。这些数据包括返回地址、前一个堆栈指针、函数的参数和函数的局部值的空间. 这个空间被称为 stack frame。使用尾递归优化，尾调用的函数将会复用调用函数的堆栈空间。一个函数满足以下条件可称为尾调用：

- 调用函数处于 strict mode
- 调用函数是一个普通函数或者箭头函数
- 调用函数不是一个 generator 函数
- 被调用函数的返回值被调用函数返回

如果一个函数是尾调用，ECMAScript 6 要求调用必须复用它自身 stack frame 的空间，而不是往调用栈推入新的 stack frame。调用函数的帧称为尾部删除帧，因为它在执行尾调用后不再位于堆栈上。这意味着尾部删除函数将不会出现在堆栈跟踪中。

下面是一个尾递归优化的例子：

```javascript
"use strict"
function factorial(n) {
  if (!n)
    return 1
  return n * factorial(n - 1)
}
```

如果改写成尾递归，只保留一个调用记录，复杂度 O(1)

```javascript
"use strict"
function gcd(m, n) {
  if (!n)
    return m
  return gcd(n, m % n)
}
```

> 目前 PTC 在桌面浏览器仅有 Safari 支持 https://kangax.github.io/compat-table/es6/

### PTC 的好处

### 堆栈空间

很显然，PTC 最明显的好处是减少了堆栈的使用空间。减少堆栈使用量同样也会减少所需要缓存空间，为其他内存访问释放空间。

### 局部对象分配

考虑一个函数为局部对象分配了空间，但是这个对象不暴露到外部。唯一指向该对象的方式就是通过函数 stack frame 里的指针或者这个函数使用的寄存器。Javascript 引擎垃圾回收的时候，它会通过扫描堆栈和 CPU 寄存器的内容来找到一个对本地对象的引用。如果该函数调用另一个函数，而该调用不是尾部调用，那么在调用函数本身返回之前，将不会回收调用函数的任何本地对象。

然而，如果函数尾调用了另一个函数，所有调用者的本地对象会被垃圾回收，因为不会再有堆栈指向这个对象了。

### 尾调用的返回值

另一个 PTC 的好处是，当最后一个节点的函数返回是，它会穿过所有的中间尾调用函数，直接返回给第一个非尾调用的函数。这个减轻了所有中间函数的返回过程。连续尾部调用的调用链越深，它提供的性能优势就越大。这对于直接递归和相互递归都适用

## Syntactic Tail Calls

语义化尾递归 Syntactic Tail Calls (STC) 是指用显式的语法来指出尾递归优化调用，例如：

```javascript
function factorial(n, acc = 1) {
  if (n === 1) {
    return acc;
  }
  return continue factorial(n - 1, acc * n)
}
```

或者使用箭头函数：

```javascript
let factorial = (n , acc = 1) =>
  n == 1 ? acc
         : continue factorial(n - 1, acc * n)
```

目前 STC 还处于提案阶段，STC 出现的背景是在 PTC 被提出的时候，有非常多关于 PTC 的优点和缺点的讨论，比如调试困难以及实现的困难等等。但不幸的是，在那个时候 TC39 流程并不需要大量的实现者参与，所以尽管很多实现者对此还有很多疑虑，这个功能还是在 ES6 里被包含进标准里了。

最近实现者们开始认真去研究实现尾递归，发现部分实现有问题。他们希望可以通过更改 PTC 的设计来解决问题，要求使用 PTC 需要语法上的选择。

### PTC 的问题

#### 性能

一些实现者和开发人员认为，PTC 是一种优惠策略，通过 PTC，引擎将使代码运行地更快。JSC 团队的实现验证了这一信念，该实现在某些情况下显示了一些性能上的优势。但是 V8 团队认为这个信念是不靠谱的，V8 团队认为这和性能无关，而 Chakra 团队则认为，由于一些其他方面的限制，他们可能无法在不降低现有代码性能的情况下实现这一特性。

#### 开发者工具

开发人员非常依赖调用堆栈来调试代码，实现 PTC 将导致许多帧丢失。这个问题可以通过实现某种形式的影子堆栈来减轻（例如 JSC 的 ShadowChicken），它维护了一个可以在调试期间显示给用户的侧 stack frame。但是这也是有代价的，它可能只有在开发者工具打开时才会启用（因此不能解决下面 Error.stack 问题）。它也不是解决潜在问题的灵丹妙药，比如在调试工作流时导致垃圾回收的不确定性。

例如下面的例子：

```javascript
function foo(n) {
  return bar(n*2)
}

function bar() {
  throw new Error()
}

foo(1)
```

bar 会复用 foo 的 stack frame，因为 bar 被尾调用了。但是在错误堆栈里和开发者工具里，foo 的 stack frame 会被丢失。

如果使用 STC 的话，这个问题就会变成一个有意图的选择。开发者工具不需要展示每帧给开发者因为开发者选择省略这些帧。

#### Error.stack

Javascript 错误会在 PTC 启用的情况下因为省略了部分 stack frame 而显示出不同的错误堆栈。
考虑下面的例子：

```javascript
function foo(n) {
  return bar(n*2);
}

function bar() {
  throw new Error();
}

try {
  foo(1);
} catch(e) {
  print(e.stack);
}

/*
output without PTC
Error
    at bar
    at foo
    at Global Code

output with PTC (note how it appears that bar is called from Global Code)
Error
    at bar
    at Global Code
*/
```

这对于使用此信息调试程序的开发人员来说是个问题。假设我们在不进行实际调试的情况下，无法恢复所有被省略的帧，那么 web 分析工具就会受到浏览器的影响。不同浏览器对于相同错误会有不同的调用堆栈。