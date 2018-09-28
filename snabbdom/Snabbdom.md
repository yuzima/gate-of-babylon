# Snabbdom 源码解析

Snabbdom 是一个专注于简单性，模块化，强大的功能和性能的 virtual DOM 库。

它出现的原因是作者认为现有的解决方案太过臃肿、太慢、缺乏功能、API 太过面向对象以及缺少他需要的功能。

Snabbdom 核心代码只有 200 行左右，非常简单，高性能，易扩展，它提供了模块化架构，所有非必要功能都委托给模块。你可以将 Snabbdom 塑造成你想要的样子，也可以仅仅使用默认的扩展并获得一个小巧的高性能的 virtual Dom 库。

## 功能

- 核心功能
  - 仅 200 行左右的核心代码
  - 可通过模块扩展
  - 丰富的 hooks，可以 hook 到所有 vnode 和模块，以及 diff 和 patch 的任一阶段。
  - 出色的性能表现
  - Patch 具有等效于 reduce/scan 的函数签名，允许更轻松地与 FRP (functional reactive programming) 库集成。
- 模块功能
  - `h` 函数用于创建 virtual DOM 节点
  - `h` 函数同样适用于SVG
  - 执行复杂 CSS 动画
  - 用于进一步优化 diff 和 patch 的转换程序 Thunk
- 第三方功能
  - JSX 支持 - [snabbdom-pragma](https://github.com/Swizz/snabbdom-pragma)
  - 服务器端 HTML 输出 - [snabbdom-to-html](https://github.com/acstll/snabbdom-to-html)
  - 简洁的 virtual DOM 创建 - [snabbdom-helpers](https://github.com/krainboltgreene/snabbdom-helpers)
  - 模板字符串支持 - [snabby](https://github.com/jamen/snabby)

## 代码 Example

```javascript
var snabbdom = require('snabbdom');
var patch = snabbdom.init([ // 选择模块并初始化 patch
  require('snabbdom/modules/class').default, // 处理切换 class
  require('snabbdom/modules/props').default, // 设置 DOM 元素属性
  require('snabbdom/modules/style').default, // 处理样式并支持动画
  require('snabbdom/modules/eventlisteners').default, // 添加事件监听
]);
var h = require('snabbdom/h').default; // 用于创建 vnode 的帮助函数 `h`

var container = document.getElementById('container');

var vnode = h('div#container.two.classes', {on: {click: someFn}}, [
  h('span', {style: {fontWeight: 'bold'}}, 'This is bold'),
  ' and this is just normal text',
  h('a', {props: {href: '/foo'}}, 'I\'ll take you places!')
]);
// patch 到空 DOM 元素 - 产生了修改 DOM 的副作用
patch(container, vnode);

var newVnode = h('div#container.two.classes', {on: {click: anotherEventHandler}}, [
  h('span', {style: {fontWeight: 'normal', fontStyle: 'italic'}}, 'This is now italic type'),
  ' and this is still just normal text',
  h('a', {props: {href: '/bar'}}, 'I\'ll take you places!')
]);
// 第二次 `patch` 调用
patch(vnode, newVnode); // Snabbdom 能够高效更新旧视图为新状态
```

## 目录结构

```
src
├── helpers               # 帮助函数
│   └── attachto.ts       # attachTo 声明 vnode 应该附加到父元素之外的其他元素
├── modules               # 处理不同功能的模块
│   ├── attributes.ts     # DOM attributes
│   ├── class.ts          # DOM class
│   ├── dataset.ts        # DOM dataset
│   ├── eventlisteners.ts # DOM 事件监听
│   ├── hero.ts           # 处理 transition 动画
│   ├── module.ts         # 模块接口
│   ├── props.ts          # DOM properties
│   └── style.ts          # 样式和动画
├── h.ts                  # `h` 函数，用于创建 vnode
├── hooks.ts              # 定义了各种 hooks
├── htmldomapi.ts         # 封装了各种 DOM 操作
├── is.ts                 # 工具函数，用于变量类型判断
├── snabbdom.bundle.ts    # snabbdom 默认功能模块初始化打包
├── snabbdom.ts           # snabbdom 核心：patch，diff
├── thunk.ts              # 转换程序 Thunk，用于优化 diff 和 patch
├── tovnode               # 将 DOM 节点转换为 vnode
└── vnode.ts              # vnode 数据结构定义
```

## 源码解析

### vnode.ts

该文件定义了 vnode 数据结构

```javascript
export type Key = string | number;

export interface VNode {
  sel: string | undefined; // 选择器 selector
  data: VNodeData | undefined; // 节点数据
  children: Array<VNode | string> | undefined; // 子节点
  elm: Node | undefined; // vnode 绑定的 HTML Element
  text: string | undefined; // 节点文本，用于文本节点
  key: Key | undefined; // 节点 key
}
```

以及节点相关数据 VNodeData 的数据结构

```javascript
export interface VNodeData {
  props?: Props;
  attrs?: Attrs;
  class?: Classes;
  style?: VNodeStyle;
  dataset?: Dataset;
  on?: On;
  ...
}
```

以及 vnode 的构造函数

```js
export function vnode(sel: string | undefined,
                      data: any | undefined,
                      children: Array<VNode | string> | undefined,
                      text: string | undefined,
                      elm: Element | Text | undefined): VNode {
  let key = data === undefined ? undefined : data.key;
  return {sel: sel, data: data, children: children,
          text: text, elm: elm, key: key};
}
```

### h.ts

该文件输出 `h` 函数，snabbdom 推荐使用 `h` 来创建 vnodes，`h` 接受 tag /选择器作为字符串，可选的数据对象和字符串或子节点数组。

```js
var h = require('snabbdom/h').default;
var vnode = h('div', {style: {color: '#000'}}, [
  h('h1', 'Headline'),
  h('p', 'A paragraph'),
]);
```

h.ts 主要功能是接受参数，调用 vnode 构造函数生成 VNode 树，`h` 能接受不同的输入参数情形，它会根据参数判断输入参数是属于子节点 children， 还是属于节点数据对象。

```js
import {vnode, VNode, VNodeData} from './vnode';
import * as is from './is';

// 添加命名空间，仅用于 SVG
function addNS(data: any, children: VNodes | undefined, sel: string | undefined): void {
    /* 递归设置自身及所有子节点 data.ns = 'http://www.w3.org/2000/svg' */
}

// 生成 VNode 树
export function h(sel: string): VNode;
export function h(sel: string, data: VNodeData): VNode;
export function h(sel: string, children: VNodeChildren): VNode;
export function h(sel: string, data: VNodeData, children: VNodeChildren): VNode;
export function h(sel: any, b?: any, c?: any): VNode {
  var data: VNodeData = {}, children: any, text: any, i: number;
  // 判断参数输入情况，正确设置 data， children，text
  if (c !== undefined) { // 参数 c 不为 undefined 则表示存在子节点
    data = b; // 那么 b 就是 VNodeData
    if (is.array(c)) { children = c; } // c 是数组则表示子节点数组 children
    // c 是 string 或 number，该节点就是文本节点，c 为文本节点内容
    else if (is.primitive(c)) { text = c; }
    // c 是 vnode，则 c 为children 数组的唯一子节点
    else if (c && c.sel) { children = [c]; }
  } else if (b !== undefined) { // 仅有参数 b，判断与前面类似，
    if (is.array(b)) { children = b; }
    else if (is.primitive(b)) { text = b; }
    else if (b && b.sel) { children = [b]; }
    else { data = b; } // 若不符合 children 或 text，那么 b 就是 VNodeData
  }
  // 如果子节点中包含文本，则处理为文本节点
  if (children !== undefined) {
    for (i = 0; i < children.length; ++i) {
      if (is.primitive(children[i])) children[i] = vnode(undefined, undefined, undefined, children[i], undefined);
    }
  }
  // 为 SVG 添加命名空间
  if (sel[0] === 's' && sel[1] === 'v' && sel[2] === 'g' && (sel.length === 3 || sel[3] === '.' || sel[3] === '#')) {
    addNS(data, children, sel);
  }
  return vnode(sel, data, children, text, undefined);
}
```

### Snabbdom.ts

创建完 VNode 树，接下来需要根据它来生成真正的 HTML Element 树。

#### init 和 patch

初始化函数 `init` 接受初始化的模块列表，返回函数 `patch` 用于更新 vnode。

```js
import snabbdom from 'snabbdom';

var patch = snabbdom.init([
  require('snabbdom/modules/class').default,
  require('snabbdom/modules/style').default,
]);
```

在初始化过程中，会记录所有模块的钩子，方便在将来触发事件的时候，调用模块里对应的钩子函数。

为了节省时间，patch 更新时会先判断新旧 vnode 根节点是否一致，如果根节点不一致，说明整个 vnode 都被替换了，则不需要花费时间进行对比子节点，直接将 oldVnode 完全替换为新 vnode。如果根节点一致，再进行下一步的 `patchVnode` 对比。

```js
import htmlDomApi, {DOMAPI} from './htmldomapi';

function isUndef(s: any): boolean { return s === undefined; }
function isDef(s: any): boolean { return s !== undefined; }

const hooks: (keyof Module)[] = ['create', 'update', 'remove', 'destroy', 'pre', 'post'];

export {h} from './h';

// init 函数接受初始化的模块列表，返回 patch 方法
export function init(modules: Array<Partial<Module>>, domApi?: DOMAPI) {
  let i: number, j: number, cbs = ({} as ModuleHooks);
  const api: DOMAPI = domApi !== undefined ? domApi : htmlDomApi;
  // 记录模块的钩子
  for (i = 0; i < hooks.length; ++i) {
    cbs[hooks[i]] = [];
    for (j = 0; j < modules.length; ++j) {
      const hook = modules[j][hooks[i]];
      if (hook !== undefined) {
        (cbs[hooks[i]] as Array<any>).push(hook);
      }
    }
  }
  ...
  return function patch(oldVnode: VNode | Element, vnode: VNode): VNode {
    let i: number, elm: Node, parent: Node;

    // 初始化 insertedVnodeQueue
    const insertedVnodeQueue: VNodeQueue = [];
    // 触发所有 `pre` hook 函数
    for (i = 0; i < cbs.pre.length; ++i) cbs.pre[i]();

    if (!isVnode(oldVnode)) { // 若 oldVnode 非 Vnode，则将其设置为一个空 Vnode
      oldVnode = emptyNodeAt(oldVnode);
    }
    if (sameVnode(oldVnode, vnode)) { // 若新旧 Vnode 根节点相同
      patchVnode(oldVnode, vnode, insertedVnodeQueue); // 更新子节点
    } else {
      // 若新旧 Vnode 根节点不同，则创建新 Element，插入到 oldVnode 父节点下面，
      // 最后从父节点删除 oldVnode
      elm = oldVnode.elm as Node;
      parent = api.parentNode(elm);
      createElm(vnode, insertedVnodeQueue);
      if (parent !== null) {
        api.insertBefore(parent, vnode.elm as Node, api.nextSibling(elm));
        removeVnodes(parent, [oldVnode], 0, 0);
      }
    }
  }
}
```

#### patchVnode

`patchVnode` 函数会判断新旧 vnode 类型以及子节点，并根据情况进行最少代价的更新操作。在更新的前后会触发 `prepatch` 和 `postpatch` 钩子函数。

- vnode 非文本节点
  - oldVnode 和 vnode 都有 children：对比更新子节点
  - 仅 vnode 有 children：清空 DOM 里的文本内容，添加 vnode 子节点
  - 仅 oldVnode 有 children：删除 DOM 上所有子节点
  - 都没有 children 且 oldVnode 为文本节点：清空 DOM 里的文本内容
- vnode 为文本节点
  - vnode.text 与 oldVnode.text 不同：更新 DOM 里的文本内容为 vnode.text

```js
function patchVnode(oldVnode: VNode, vnode: VNode, insertedVnodeQueue: VNodeQueue) {
  let i: any, hook: any;
  if (isDef(i = vnode.data) && isDef(hook = i.hook) && isDef(i = hook.prepatch)) {
    i(oldVnode, vnode); // 触发 vnode 上定义的 `prepatch` 钩子函数
  }
  const elm = vnode.elm = (oldVnode.elm as Node); // 获取绑定 HTMLElement
  let oldCh = oldVnode.children;
  let ch = vnode.children;
  if (oldVnode === vnode) return;
  if (vnode.data !== undefined) {
    // 触发所有模块上的 `update` 钩子函数
    for (i = 0; i < cbs.update.length; ++i) cbs.update[i](oldVnode, vnode);
    // 触发新 vnode 上定义的 `update` 钩子函数
    i = vnode.data.hook;
    if (isDef(i) && isDef(i = i.update)) i(oldVnode, vnode);

    if (isUndef(vnode.text)) { // vnode 非文本节点
      if (isDef(oldCh) && isDef(ch)) {
        if (oldCh !== ch) // 都拥有子节点，则对比更新子节点
          updateChildren(elm, oldCh as Array<VNode>, ch as Array<VNode>, insertedVnodeQueue);
      } else if (isDef(ch)) { // 仅 vnode 拥有子节点，清空 DOM 文本内容，添加 vnode 子节点
        if (isDef(oldVnode.text)) api.setTextContent(elm, '');
        addVnodes(elm, null, ch as Array<VNode>, 0, (ch as Array<VNode>).length - 1, insertedVnodeQueue);
      } else if (isDef(oldCh)) { // 仅 oldVnode 拥有子节点，删除 DOM 上所有 oldVnode 子节点
        removeVnodes(elm, oldCh as Array<VNode>, 0, (oldCh as Array<VNode>).length - 1);
      } else if (isDef(oldVnode.text)) { // oldVnode 为文本节点，清空 elm 文本
        api.setTextContent(elm, '');
      }
    } else if (oldVnode.text !== vnode.text) { // vnode 为文本节点且内容与 oldVnode 不同
      api.setTextContent(elm, vnode.text as string); // 设置 elm 内容为 vnode.text
    }
    if (isDef(hook) && isDef(i = hook.postpatch)) { // 触发 vnode 上定义的 `postpatch` 钩子函数
      i(oldVnode, vnode);
    }
  }
}
```

`addVnodes` 和 `removeVnodes` 代码比较简单，就是调用对应的 DOM API 操作，这里就不详细展开，主要讲下 `updateChildren`。

#### updateChildren

updateChildren 函数包含了 virtual DOM 库核心内容 diff 算法。我们知道操作 DOM 是非常缓慢的，因此我们应该尽量复用原来存在的节点，diff 算法描述了如何高效比对两个 vnode 树的子节点，来让每次 patch 的代价最小，这也是 virtual DOM 的性能关键。

先介绍下 snabbdom 的 diff 算法逻辑：

- 同层子节点比较
- 子节点列表遇到为 null 的子节点，指针顺移一位
- 比较 oldStartVnode 和 newStartVnode
- 比较 oldEndVnode 和 newEndVnode
- 比较 oldStartVnode 和 newEndVnode
- 比较 oldEndVnode 和 newStartVnode
- 以上比较如果相同，则对两个 vnode 调用 patchVnode 递归更新它们的子节点，然后处理 DOM 并移动指针
- 剩余的 oldCh 创建一个键值对为 (oldVnode.key, oldIdx) 的 map 对象，然后在该 map 中查找是否有与 newStartVnode.key 相同 key 并获得它在 oldCh 里的位置 idxInOld。
- 根据 idxInOld 在 oldCh 定位获得 elmToMove，比较 elmToMove 与 newStartVnode 的 sel，相同则表示是同一节点，只需要在 DOM 中把 elmToMove 移动到新位置；不相同则需要创建新 element 插入到 DOM。
- 最后，向 DOM 添加剩余的 newCh 节点或从 DOM 移除剩余的 oldCh 节点。

```js
function updateChildren(parentElm: Node,
                          oldCh: Array<VNode>,
                          newCh: Array<VNode>,
                          insertedVnodeQueue: VNodeQueue) {
    let oldStartIdx = 0, newStartIdx = 0; // oldCh 和 newCh 初始位指针
    let oldEndIdx = oldCh.length - 1; // oldCh 结束位指针
    let oldStartVnode = oldCh[0]; // oldCh 起始节点
    let oldEndVnode = oldCh[oldEndIdx]; // oldCh 结束节点
    let newEndIdx = newCh.length - 1; // newCh 结束位指针
    let newStartVnode = newCh[0]; // newCh 起始节点
    let newEndVnode = newCh[newEndIdx]; // newCh 结束节点
    let oldKeyToIdx: any; // 键值对为 (oldVnode.key, oldIdx) 的 map 对象
    let idxInOld: number; // oldCh 中查找到相同 key 的位置
    let elmToMove: VNode; // oldCh 中查找到相同 key 的 vnode 节点
    let before: any; //

    while (oldStartIdx <= oldEndIdx && newStartIdx <= newEndIdx) {
      if (oldStartVnode == null) {
        oldStartVnode = oldCh[++oldStartIdx]; // Vnode might have been moved left
      } else if (oldEndVnode == null) {
        oldEndVnode = oldCh[--oldEndIdx];
      } else if (newStartVnode == null) {
        newStartVnode = newCh[++newStartIdx];
      } else if (newEndVnode == null) {
        newEndVnode = newCh[--newEndIdx];
      }
      // 比较 oldStartVnode 和 newStartVnode
      else if (sameVnode(oldStartVnode, newStartVnode)) {
        patchVnode(oldStartVnode, newStartVnode, insertedVnodeQueue);
        oldStartVnode = oldCh[++oldStartIdx];
        newStartVnode = newCh[++newStartIdx];
      }
      // 比较 oldEndVnode 和 newEndVnode
      else if (sameVnode(oldEndVnode, newEndVnode)) {
        patchVnode(oldEndVnode, newEndVnode, insertedVnodeQueue);
        oldEndVnode = oldCh[--oldEndIdx];
        newEndVnode = newCh[--newEndIdx];
      }
      // 比较 oldStartVnode 和 newEndVnode
      else if (sameVnode(oldStartVnode, newEndVnode)) { // Vnode moved right
        patchVnode(oldStartVnode, newEndVnode, insertedVnodeQueue);
        // 操作 DOM: 把 oldStartVnode 插入到 oldEndVnode 后面
        api.insertBefore(parentElm, oldStartVnode.elm as Node, api.nextSibling(oldEndVnode.elm as Node));
        oldStartVnode = oldCh[++oldStartIdx];
        newEndVnode = newCh[--newEndIdx];
      }
      // 比较 oldEndVnode 和 newStartVnode
      else if (sameVnode(oldEndVnode, newStartVnode)) { // Vnode moved left
        patchVnode(oldEndVnode, newStartVnode, insertedVnodeQueue);
        // 操作 DOM: 把 oldEndVnode 插入到 oldStartVnode 前面
        api.insertBefore(parentElm, oldEndVnode.elm as Node, oldStartVnode.elm as Node);
        oldEndVnode = oldCh[--oldEndIdx];
        newStartVnode = newCh[++newStartIdx];
      }
      else {
        if (oldKeyToIdx === undefined) {
          // 创建一个键值对为 (oldVnode.key, oldIdx) 的 map
          oldKeyToIdx = createKeyToOldIdx(oldCh, oldStartIdx, oldEndIdx);
        }
        // 查找 oldCh 里是否有相同的 key，获得位置 idxInOld
        idxInOld = oldKeyToIdx[newStartVnode.key as string];
        if (isUndef(idxInOld)) { // 未找到相同 key，创建 New element 插入到 DOM
          api.insertBefore(parentElm, createElm(newStartVnode, insertedVnodeQueue), oldStartVnode.elm as Node);
          newStartVnode = newCh[++newStartIdx];
        } else { // 找到相同 key，获取这个 vnode 节点
          elmToMove = oldCh[idxInOld];
          if (elmToMove.sel !== newStartVnode.sel) { // 比较 sel
            // sel 不相同，则还是要创建 New element 插入到 DOM
            api.insertBefore(parentElm, createElm(newStartVnode, insertedVnodeQueue), oldStartVnode.elm as Node);
          } else {
            // sel 相同，则在 DOM 移动这个节点的位置
            patchVnode(elmToMove, newStartVnode, insertedVnodeQueue);
            oldCh[idxInOld] = undefined as any;
            api.insertBefore(parentElm, (elmToMove.elm as Node), oldStartVnode.elm as Node);
          }
          // 移动 newStartIdx
          newStartVnode = newCh[++newStartIdx];
        }
      }
    }
    if (oldStartIdx <= oldEndIdx || newStartIdx <= newEndIdx) {
      if (oldStartIdx > oldEndIdx) {
        before = newCh[newEndIdx+1] == null ? null : newCh[newEndIdx+1].elm;
        // 向 DOM 添加剩余的 newCh 节点
        addVnodes(parentElm, before, newCh, newStartIdx, newEndIdx, insertedVnodeQueue);
      } else {
        // 从 DOM 移除剩余的 oldCh 节点
        removeVnodes(parentElm, oldCh, oldStartIdx, oldEndIdx);
      }
    }
  }
```

### Vnode => HTMLElement

现在我们已经了解了 snabbdom 是如何对两个节点进行 patch 的，接下来学习 snabbdom 是如何把 vnode 转化为 HTMLElement 的。snabbdom.ts 只有最核心的创建 element 功能，snabbdom 把对 DOM 的处理分散到各个模块里，不同的模块只负责处理 DOM 对应的属性，因此必须在初始化的时候就传入所需要处理的功能模块。

下面我们用 class 模块举例，class 模块负责 DOM classList 的更新。

我们先初始化一个 snabbdom，选择 class 作为依赖模块添加进去，接下来我们先看下 class 这个模块代码。

```js
var snabbdom = require('snabbdom');
var patch = snabbdom.init([ // 选择模块并初始化 patch
  require('snabbdom/modules/class').default, // 处理切换 class
]);
```

### class.ts

我们先不研究 updateClass 方法里的具体代码，我们只要知道 updateClass 会对比 oldVnode 和 vnode 里的 class 列表并更新实际的 HTMLElement 的 classList。这里 class.ts 输出了一个对象，这个对象包含两个 hook， `create` 和 `update`，它们都指向 updateClass。

```js
import {VNode, VNodeData} from '../vnode';
import {Module} from './module';

export type Classes = Record<string, boolean>

function updateClass(oldVnode: VNode, vnode: VNode): void {
  // 对比 oldVnode 和 vnode 里的 class 列表并更新 elm 的 classList
}
export const classModule = {create: updateClass, update: updateClass} as Module;
export default classModule;
```

### snabbdom.ts

前面介绍到 init 的时候需要加载需要的功能模块，每个模块都是用来处理不同的 DOM 属性的，每个模块返回它的 hook 函数对象，这些 hooks 都被记录在了 `cbs` 这个数组里，在处理 DOM 的不同阶段会调用 `cbs` 里记录的该阶段里的所有模块 hook 函数。

```js
// init 函数接受初始化的模块列表，返回 patch 方法
export function init(modules: Array<Partial<Module>>, domApi?: DOMAPI) {
  let i: number, j: number, cbs = ({} as ModuleHooks);
  const api: DOMAPI = domApi !== undefined ? domApi : htmlDomApi;
  // 记录模块的钩子
  for (i = 0; i < hooks.length; ++i) {
    cbs[hooks[i]] = [];
    for (j = 0; j < modules.length; ++j) {
      const hook = modules[j][hooks[i]];
      if (hook !== undefined) {
        (cbs[hooks[i]] as Array<any>).push(hook);
      }
    }
  }
  ...
}
```

有了 vnode，就需要把它转换为实际的 DOM，这就是 createElm 的工作。

```js
function createElm(vnode: VNode, insertedVnodeQueue: VNodeQueue): Node {
  let i: any, data = vnode.data;
  if (data !== undefined) {
    if (isDef(i = data.hook) && isDef(i = i.init)) {
      i(vnode);
      data = vnode.data;
    }
  }
  let children = vnode.children, sel = vnode.sel;
  // 如果 sel 为 `!`，这是一个注释标签
  if (sel === '!') {
    if (isUndef(vnode.text)) {
      vnode.text = '';
    }
    vnode.elm = api.createComment(vnode.text as string);
  }
  // 存在 sel
  else if (sel !== undefined) {
    // 解析 selector
    const hashIdx = sel.indexOf('#');
    const dotIdx = sel.indexOf('.', hashIdx);
    const hash = hashIdx > 0 ? hashIdx : sel.length;
    const dot = dotIdx > 0 ? dotIdx : sel.length;

    // 获得 tag
    const tag = hashIdx !== -1 || dotIdx !== -1 ? sel.slice(0, Math.min(hash, dot)) : sel;
    // 根据 tag 创建 element (如果 data 带 namespace，用 createElementNS 创建)
    const elm = vnode.elm = isDef(data) && isDef(i = (data as VNodeData).ns) ? api.createElementNS(i, tag) : api.createElement(tag);

    // 如果 selector 带 `#`，设置 id 值为 `#` 后面的内容
    if (hash < dot) elm.setAttribute('id', sel.slice(hash + 1, dot));
    // 如果 selector 带 `.`，设置 class 值为 `.` 后面的内容
    if (dotIdx > 0) elm.setAttribute('class', sel.slice(dot + 1).replace(/\./g, ' '));

    // 调用加载的模块里所有的 `create` hook
    for (i = 0; i < cbs.create.length; ++i) cbs.create[i](emptyNode, vnode);

    // 循环子节点数组，创建所有非空子节点，并 append 到 elm 下面
    if (is.array(children)) {
      for (i = 0; i < children.length; ++i) {
        const ch = children[i];
        if (ch != null) {
          api.appendChild(elm, createElm(ch as VNode, insertedVnodeQueue));
        }
      }
    }
    // 如果 vnode 为文本节点，创建 textNode 并 append 到 elm 下面
    else if (is.primitive(vnode.text)) {
      api.appendChild(elm, api.createTextNode(vnode.text));
    }

    // 触发 data 里自定义的 `create` 和 `insert` hook 函数
    i = (vnode.data as VNodeData).hook; // Reuse variable
    if (isDef(i)) {
      if (i.create) i.create(emptyNode, vnode);
      if (i.insert) insertedVnodeQueue.push(vnode);
    }
  }
  // 不存在 sel，创建文本节点作为根节点
  else {
    vnode.elm = api.createTextNode(vnode.text as string);
  }
  // 返回 vnode 绑定的 DOM 节点 elm
  return vnode.elm;
}
```

在下面这行代码里，会调用加载的模块里所有的 `create` hook，前面初始化的时候，class 模块输出的 `create` 和 `update` hook 已经被 cbs 记录下来了，在创建完 element 后，此时会循环调用所有 cbs.create 里的 hook 函数。由于 class 模块的 `create` hook 指向的是 updateClass 函数，所以这个时候 class 模块的 updateClass 函数会被调用，用来更新 element 的 classList。

```js
 for (i = 0; i < cbs.create.length; ++i) cbs.create[i](emptyNode, vnode);
```

至此 Snabbdom 的核心功能的代码流程已经解释完了，后续会分析 Snabbdom 的一些其他功能的代码。

