# preact 源码解析
preact 是 React 的 3kb 轻量化方案，拥有同样的 ES6 API。
搭配 `preact-compat` 可以使用任何 react 库。

该文章目的不是为了介绍 preact，而是希望通过解析 preact 代码来学习 react。
对于学习 react 的新手来说，阅读 preact 代码比直接阅读 react 代码更简单，门槛更低。
这里不对 VDOM 做具体介绍，想了解 virtual dom 的可以移步 [snabbdom 源码解析](../snabbdom/snabbdom.md)。

## 代码结构目录
```
src
├── dom
│   └── index.js              # DOM 操作
├── vdom
│   ├── component-recycler.js
│   ├── component.js
│   ├── diff.js
│   └── index.js
├── clone-element.js
├── component.js
├── constants.js              # 常量项
├── h.js                      # 创建 VNode 方法
├── options.js                # 全局配置参数
├── preact.d.ts               # typings 定义
├── preact.js                 # 入口文件，输出各种方法
├── preact.js.flow
├── render-queue.js
├── render.js
├── util.js                   # 工具方法
└── vnode.js                  # 输出 VNode 方法
```

## 源码解析

> preact 源码中使用通过 JSDoc 注释来定义数据结构。目前 typescript 已经支持使用 JSDoc 部分注释来声明类型信息。具体可以参考这篇文章 ![JSDoc support in JavaScript](https://github.com/Microsoft/TypeScript/wiki/JSDoc-support-in-JavaScript)。

### vnode.js
该文件定义了 VNode 的数据结构，并输出了一个空函数为 VNode。

preact 的 VNode 数据结构比 Sanbbdom 中的更简单，只有 `nodeName`， `children`，`key`，`attributes` 这四个属性。

```javascript
/**
 * Virtual DOM Node
 * @typedef VNode
 * @property {string | function} nodeName 创建的 DOM 节点标签名
 * @property {Array<VNode | string>} children 子节点
 * @property {string | number | undefined} key 用于在列表中标识此 VNode
 * @property {object} attributes 该 VNode 的属性
 */
export const VNode = function VNode() {};
```

### option.js
依然是通过 JSDoc 注释定义了 options 的数据结构，并输出空对象为 options。

- syncComponentUpdates：如果为 true，prop 变化会触发 component 同步更新，默认为 true
- vnode：用于处理所有创建的 VNodes 的函数
- afterMount：钩子：挂载组件后
- afterUpdate：钩子：组件最新一次渲染导致的 DOM 更新后
- beforeUnmount：钩子：卸载组件前立即调用
- debounceRendering：钩子：每次申请 rerender 时都会调用，可以用来做 rerender 防抖
- event：在任何 Preact 事件监听器之前调用，返回值（如果有）将把被传递给事件监听器的浏览器事件替换掉

```javascript
/**
 * @typedef {import('./component').Component} Component
 * @typedef {import('./vnode').VNode} VNode
 */

/**
 * Global options
 * @public
 * @typedef Options
 * @property {boolean} [syncComponentUpdates]
 * @property {(vnode: VNode) => void} [vnode]
 * @property {(component: Component) => void} [afterMount]
 * @property {(component: Component) => void} [afterUpdate]
 * @property {(component: Component) => void} [beforeUnmount]
 * @property {(rerender: function) => void} [debounceRendering]
 * @property {(event: Event) => Event | void} [event]
 */

/** @type {Options}  */
const options = {};

export default options;
```
options 用于设置一些全局的配置，例如，默认情况下 state 变化是异步更新，prop 变化是同步更新，如果你不希望 prop 变化引起组件的同步更新，那么可以通过设置 `options.syncComponentUpdates` 为 false。
```javascript
import { options } from 'preact'

options.syncComponentUpdates = false;
```

### h.js

这里的 `h` 方法用于创建 VNode，一个 VNode 树可用来表示 DOM 树的结构。

```html
<div id="foo" name="bar">Hello!</div>
```
可以用以下函数构建
```javascript
h('div', { id: 'foo', name : 'bar' }, 'Hello!');
```

`h()` 方法接收一个元素名参数 nodeName 和 attributes/props 列表参数，以及可选的子节点元素。

```javascript
import { VNode } from './vnode';
import options from './options';

const stack = []; // 子节点元素堆栈
const EMPTY_CHILDREN = [];

export function h(nodeName, attributes) {
  let children=EMPTY_CHILDREN, lastSimple, child, simple, i;
  // 有子节点参数，添加到 stack 里
  for (i=arguments.length; i-- > 2; ) {
    stack.push(arguments[i]);
  }
  // attributes 包含 children 字段且 stack 还没有添加过子节点参数
  // 添加 attributes.children 到 stack
  if (attributes && attributes.children!=null) {
    if (!stack.length) stack.push(attributes.children);
    delete attributes.children;
  }
  // 处理 stack 里的所有子节点
  while (stack.length) {
    // 子节点是列表，循环添加到 stack
    if ((child = stack.pop()) && child.pop!==undefined) {
      for (i=child.length; i--; ) stack.push(child[i]);
    }
    // 特殊类型子节点的处理
    else {
      if (typeof child==='boolean') child = null;
      if ((simple = typeof nodeName!=='function')) {
        if (child==null) child = '';
        else if (typeof child==='number') child = String(child);
        else if (typeof child!=='string') simple = false;
      }
      if (simple && lastSimple) {
        children[children.length-1] += child;
      }
      else if (children===EMPTY_CHILDREN) {
        children = [child];
      }
      else {
        children.push(child);
      }
      lastSimple = simple;
    }
  }

  // 创建 VNode 设置属性并返回
  let p = new VNode();
  p.nodeName = nodeName;
  p.children = children;
  p.attributes = attributes==null ? undefined : attributes;
  p.key = attributes==null ? undefined : attributes.key;

  // if a "vnode hook" is defined, pass every created VNode to it
  if (options.vnode!==undefined) options.vnode(p);

  return p;
}
```
因为兼容性的原因，它被导出为 `h()` 和 `createElement()`
```javascript
// preact.js
import { h, h as createElement } from './h';

export {
  h,
  createElement,
  ...
};
```

### constants.js
该文件用于存放一些静态常量，包括 render 模式，用于缓存 props 的 DOM 属性名，以及检测 DOM 属性是否带 `px` 的正则表达式
```javascript
// 渲染模式
/** 不对组件进行 re-render */
export const NO_RENDER = 0;
/** 同步 re-render 组件及其子节点 */
export const SYNC_RENDER = 1;
/** 同步 re-render 组件，即便其生命周期试图阻止 */
export const FORCE_RENDER = 2;
/** 使用异步队列 re-render 组件及其子节点 */
export const ASYNC_RENDER = 3;

// 用于缓存 props 的 DOM 属性名
export const ATTR_KEY = '__preactattr_';

/** DOM 属性数值不应添加'px' */
export const IS_NON_DIMENSIONAL = /acit|ex(?:s|g|n|p|$)|rph|ows|mnc|ntw|ine[ch]|zoo|^ord/i;
```

### dom/index.js
`dom/index.js` 这个文件提供了一些操作真实 DOM 的方法。因为对 VDOM 的各种操作最终都是要反应到真实的 DOM 里去的。

```javascript
import { IS_NON_DIMENSIONAL } from '../constants';
import { applyRef } from '../util';
import options from '../options';

/** 这里是一些类型定义... */

// 创建 node，对于 svg 标签，需要使用 createElementNS 设置 namespace
export function createNode(nodeName, isSvg) {
  /** @type {PreactElement} */
  let node = isSvg ? document.createElementNS('http://www.w3.org/2000/svg', nodeName) : document.createElement(nodeName);
  // 设置 normalizedNodeName
  node.normalizedNodeName = nodeName;
  return node;
}

// 移除节点
export function removeNode(node) {
  let parentNode = node.parentNode;
  if (parentNode) parentNode.removeChild(node);
}

// 在 node 上设置命名属性，对某些名称和事件处理程序特殊处理
// 如果 value 值为 null，则该属性/处理程序将被移除
export function setAccessor(node, name, old, value, isSvg) {
  if (name==='className') name = 'class';
  if (name==='key') {
    // ignore
  }
  // 设置 ref
  else if (name==='ref') {
    applyRef(old, null);
    applyRef(value, node);
  }
  else if (name==='class' && !isSvg) {
    node.className = value || '';
  }
  else if (name==='style') {
    if (!value || typeof value==='string' || typeof old==='string') {
      node.style.cssText = value || '';
    }
    if (value && typeof value==='object') {
      if (typeof old!=='string') {
        for (let i in old) if (!(i in value)) node.style[i] = '';
      }
      for (let i in value) {
        node.style[i] = typeof value[i]==='number' && IS_NON_DIMENSIONAL.test(i)===false ? (value[i]+'px') : value[i];
      }
    }
  }
  // 插入 HTML
  else if (name==='dangerouslySetInnerHTML') {
    if (value) node.innerHTML = value.__html || '';
  }
  // 事件处理器
  else if (name[0]=='o' && name[1]=='n') {
    // 判断是否使用事件捕捉
    let useCapture = name !== (name=name.replace(/Capture$/, ''));
    name = name.toLowerCase().substring(2);
    // addEventListener 添加事件监听器
    if (value) {
      if (!old) node.addEventListener(name, eventProxy, useCapture);
    }
    else {
      node.removeEventListener(name, eventProxy, useCapture);
    }
    // 添加到 node._listeners 对象里
    (node._listeners || (node._listeners = {}))[name] = value;
  }
  // node 中已经存在的属性名，同时该属性名不是 list 或 type，并且该元素也不是 svg
  else if (name!=='list' && name!=='type' && !isSvg && name in node) {
    // 对于其他的 DOM property，尝试直接设为 node 属性
    // IE & FF throw for certain property-value combinations.
    try {
      node[name] = value==null ? '' : value;
    } catch (e) { }
    // 值为空或 false 且属性名不是 'spellcheck'，则从 node 的 attributes 中移除该属性
    if ((value==null || value===false) && name!='spellcheck') node.removeAttribute(name);
  }
  else {
    let ns = isSvg && (name !== (name = name.replace(/^xlink:?/, '')));
    // spellcheck 和其他的属性行为不一样，属性设为 false 的时候也不能被删除
    // https://developer.mozilla.org/en-US/docs/Web/HTML/Element/input#attr-spellcheck
    // value 为 null 或 false，删除属性
    if (value==null || value===false) {
      if (ns) node.removeAttributeNS('http://www.w3.org/1999/xlink', name.toLowerCase());
      else node.removeAttribute(name);
    }
    // 否则设置 node 属性
    else if (typeof value!=='function') {
      if (ns) node.setAttributeNS('http://www.w3.org/1999/xlink', name.toLowerCase(), value);
      else node.setAttribute(name, value);
    }
  }
}

/**
 * 代理 event 到事件处理器
 * @param {Event} e The event object from the browser
 * @private
 */
function eventProxy(e) {
  // 前面介绍 options.event 会在任何 Preact 事件监听器之前调用，
  // 返回值（如果有）将把被传递给事件监听器的浏览器原生 event 替换掉
  // 如果 options 设置了 event 处理器，则使用 options.event 对 e 进行替换
  return this._listeners[e.type](options.event && options.event(e) || e);
}
```