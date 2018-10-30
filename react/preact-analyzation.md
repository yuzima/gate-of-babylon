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
│   ├── component-recycler.js # Component 循环器
│   ├── component.js
│   ├── diff.js               # 比较算法
│   └── index.js              # VNode 的公共方法
├── clone-element.js          # 复制 VNode
├── component.js              # 基础 Component 类
├── constants.js              # 常量项
├── h.js                      # 创建 VNode 方法
├── options.js                # 全局配置参数
├── preact.d.ts               # typings 定义
├── preact.js                 # 入口文件，输出各种方法
├── preact.js.flow
├── render-queue.js           # 管理渲染队列
├── render.js                 # 渲染 JSX 到一个父 Element
├── util.js                   # 工具方法
└── vnode.js                  # 输出 VNode 方法
```

## 源码解析

> preact 源码中使用通过 JSDoc 注释来定义数据结构。目前 typescript 已经支持使用 JSDoc 部分注释来声明类型信息。具体可以参考这篇文章 [JSDoc support in JavaScript](https://github.com/Microsoft/TypeScript/wiki/JSDoc-support-in-JavaScript)。

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
通过 .babelrc 配置，指定处理 createElement 方法为 h
```
{
  "plugins": [
    ["transform-react-jsx", { "pragma":"h" }]
  ]
}
```
代码里返回的 JSX 代码
```javascript
var profile = <div>
  <img src="avatar.png" className="profile" />
  <h3>{[user.firstName, user.lastName].join(' ')}</h3>
</div>;
```
会被 babel 转换成 `createElement` 形式，由于我们指定了 `pragma` 为 `h`，最终代码会被转换成如下形式。
```javascript
var profile = h("div", null,
  h("img", { src: "avatar.png", className: "profile" }),
  h("h3", null, [user.firstName, user.lastName].join(" "))
);
```
最终变成由 `h()` 来处理，这里的 `h` 方法用于创建 VNode，一个 VNode 树可用来表示 DOM 树的结构。

`h()` 方法接收一个元素名参数 nodeName 和 attributes/props 列表参数，以及可选的子节点元素。

```javascript
import { VNode } from './vnode';
import options from './options';

const stack = []; // 子节点元素堆栈
const EMPTY_CHILDREN = [];

/**
 * @param {string | function} nodeName An element name. Ex: `div`, `a`, `span`, etc.
 * @param {object | null} attributes Any attributes/props to set on the created element.
 * @param {VNode[]} [rest] Additional arguments are taken to be children to
 *  append. Can be infinitely nested Arrays.
 *
 * @public
 */
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

### utils.js
该文件提供了两个工具函数 `extend` 和 `applyRef`，和一个异步回调函数 `defer`。
```javascript
// 扩展对象
export function extend(obj, props) {
  for (let i in props) obj[i] = props[i];
  return obj;
}
// 设置或更新 Ref
export function applyRef(ref, value) {
  if (ref!=null) {
    if (typeof ref=='function') ref(value);
    else ref.current = value;
  }
}
// 尽快地异步调用一个函数，如果 Promise 不可用则使用 setTimeout
// 使用 bind 是因为 const defer = Promise.resolve().then;
// defer 此时的 this 指向了 global，调用 defer(callback) 会导致报错
// https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Errors/Called_on_incompatible_type
export const defer = typeof Promise=='function' ? Promise.resolve().then.bind(Promise.resolve()) : setTimeout;
```

### dom/index.js
`dom/index.js` 这个文件提供了一些操作实际 DOM 的方法。因为对 VDOM 的各种操作最终都是要反应到 DOM 里去的。

```javascript
import { IS_NON_DIMENSIONAL } from '../constants';
import { applyRef } from '../util';
import options from '../options';

/** 这里是一些类型定义 */
/**
 * DOM event listener
 * @typedef {(e: Event) => void} EventListner
 */

/**
 * event type 到 event listener 的映射表
 * @typedef {Object.<string, EventListener>} EventListenerMap
 */

/**
 * 被添加到 Element 上的 Preact 属性
 * @typedef PreactElementExtensions
 * @property {string} [normalizedNodeName] 用于比较的规范节点名
 * @property {EventListenerMap} [_listeners] 组件添加到此 DOM 节点的事件监听器的映射表
 * @property {import('../component').Component} [_component] 渲染该 DOM 节点的组件
 * @property {function} [_componentConstructor] 渲染该 DOM 节点的组件构造函数
 */

/**
 * 经过 Preact 属性扩展的 DOM Element
 * @typedef {Element & ElementCSSInlineStyle & PreactElementExtensions} PreactElement
 */

// 创建 node，对于 svg 标签，需要使用 createElementNS 设置 namespace
export function createNode(nodeName, isSvg) {
  /** @type {PreactElement} */
  let node = isSvg ? document.createElementNS('http://www.w3.org/2000/svg', nodeName) : document.createElement(nodeName);
  // 设置 normalizedNodeName 为 nodeName
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
  // 设置 Ref
  // https://reactjs.org/docs/refs-and-the-dom.html
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

### vdom/index.js
该文件提供了几个简单的 vnode 判断方法和获取数据方法
```javascript
import { extend } from '../util';

// 比较 PreactElement 和 VNode 是否是相同的类型
// 如果 hydrating 为 true，则忽略比较组件构造函数
export function isSameNodeType(node, vnode, hydrating) {
  if (typeof vnode==='string' || typeof vnode==='number') {
    return node.splitText!==undefined;
  }
  if (typeof vnode.nodeName==='string') {
    return !node._componentConstructor && isNamedNode(node, vnode.nodeName);
  }
  // _componentConstructor 为组件的构造函数
  return hydrating || node._componentConstructor===vnode.nodeName;
}

// 检查一个 PreactElement 是否有给定的节点名（大小写不敏感）
export function isNamedNode(node, nodeName) {
  // 在 dom/index.js 的 createNode 会设置其 normalizedNodeName 的值等于传入的 nodeName
  return node.normalizedNodeName===nodeName || node.nodeName.toLowerCase()===nodeName.toLowerCase();
}

// 获取 node 所有的 props，即 attributes 加上 defaultProps 上的属性
// defaultProps 上的属性不在 vnode.attributes 上，需要另外获取
// https://reactjs.org/docs/typechecking-with-proptypes.html#default-prop-values
export function getNodeProps(vnode) {
  let props = extend({}, vnode.attributes);
  props.children = vnode.children;

  let defaultProps = vnode.nodeName.defaultProps;
  if (defaultProps!==undefined) {
    for (let i in defaultProps) {
      if (props[i]===undefined) {
        props[i] = defaultProps[i];
      }
    }
  }

  return props;
}
```
以上代码已经大致清楚 VNode 的创建流程和操作方法，接下来来分析渲染的流程。
默认情况下，组件的 state 变化会引起组件异步更新，`Preact` 使用 `render queue` 来管理需要 re-render 的组件们。
另外还有几种渲染模式，在 `constants.js` 中可以看到。

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

### render-queue.js
`render-queue.js` 会将所有待更新的组件放入 items 列表，然后默认情况下，使用 `Promise.resolve().then` 异步执行渲染方法。

```javascript
import options from './options';
import { defer } from './util';
import { renderComponent } from './vdom/component';

/**
 * 需要 re-render 的脏组件队列
 * @type {Array<import('./component').Component>}
 */
let items = [];

/**
 * 添加一个 component 的 rerender 到队列中
 * @param {import('./component').Component} component The component to rerender
 */
export function enqueueRender(component) {
  // 设置 component._dirty 状态为 true 并推进队列
  if (!component._dirty && (component._dirty = true) && items.push(component)==1) {
    // 把 rerender 作为回调函数传递给 debounceRendering 或者 defer
    // defer 使用 `Promise.resolve().then` 调用 callback，其作用是尽快异步执行传入的 rerender 函数
    (options.debounceRendering || defer)(rerender);
  }
}

/** Rerender 所有队列中的脏组件 */
export function rerender() {
  let p;
  while ( (p = items.pop()) ) {
    // renderComponent 实际执行了 re-render 组件的工作
    if (p._dirty) renderComponent(p);
  }
}
```

### render.js
该文件输出 render 方法，用于首次将 JSX 渲染到一个父 Element 下。实际上是直接调用了 diff 方法来实现。
```javascript
import { diff } from './vdom/diff';

export function render(vnode, parent, merge) {
  return diff(merge, vnode, {}, false, parent, false);
}
```
使用方法
```javascript
render(<div id="hello">hello!</div>, document.body);
```

### component.js
在 `React` 中，`Component` 被用来划分不同的 UI 部分，这个文件描述了基础的 `Component` 构造函数。
```javascript
import { FORCE_RENDER } from './constants';
import { extend } from './util';
import { renderComponent } from './vdom/component';
import { enqueueRender } from './render-queue';

export function Component(props, context) {
  // 熟悉的 react 组件几个内部属性，public 变量
  this.context = context;
  this.props = props;
  this.state = this.state || {};

  this._dirty = true;
  // render 回调队列，记录了 setState 和 forceUpdate 的回调
  this._renderCallbacks = [];
}

// 扩展了 setState forceUpdate render 这几个方法
extend(Component.prototype, {
  // https://reactjs.org/docs/react-component.html#setstate
  setState(state, callback) {
    // 记录 prevState
    if (!this.prevState) this.prevState = this.state;
    // 创建了新 Object 合并 this.state 和 state
    this.state = extend(
      extend({}, this.state),
      typeof state === 'function' ? state(this.state, this.props) : state
    );
    // _renderCallbacks 记录 回调函数
    if (callback) this._renderCallbacks.push(callback);
    enqueueRender(this);
  },

  // https://reactjs.org/docs/react-component.html#forceupdate
  forceUpdate(callback) {
    // _renderCallbacks 记录 回调函数
    if (callback) this._renderCallbacks.push(callback);
    // 以 FORCE_RENDER 模式执行组件渲染，将跳过 shouldComponentUpdate 强制同步渲染组件
    // 但其子组件依然会触发包含 shouldComponentUpdate 在内的正常的生命周期方法
    renderComponent(this, FORCE_RENDER);
  },

  // https://reactjs.org/docs/react-component.html#render
  render() {}
});
```

### vdom/component-recycler.js
`React` 维护了一个组件池以便组件的重用，每当 unmountComponent 时，将会对一部分的组件进行回收。

创建新组件时，如果组件池存在组件的 `constructor` 和与新组件的 `constructor` 一致，则把原来组件的 `nextBase` 赋予给新组件实例。同时从组件池删除掉旧组件。`nextBase` 属性用于存放组件的 DOM 对象，在组件初始化 render 时，component.base 还未设置，nextBase 可以作为备用 DOM 对象进行渲染。

```javascript
import { Component } from '../component';

export const recyclerComponents = [];

// 创建组件
export function createComponent(Ctor, props, context) {
  let inst, i = recyclerComponents.length;
  // 如果构造函数 Ctor 是一个 Component 类，直接使用 Ctor 创建实例
  if (Ctor.prototype && Ctor.prototype.render) {
    inst = new Ctor(props, context);
    Component.call(inst, props, context);
  }
  else {
    // 否则使用 Component 创建实例，手动指定 constructor，设置 render 为默认 doRender
    inst = new Component(props, context);
    inst.constructor = Ctor;
    inst.render = doRender;
  }

  // 查找组件池是否有相同的 constructor 的组件，重复利用 nextBase
  while (i--) {
    if (recyclerComponents[i].constructor===Ctor) {
      inst.nextBase = recyclerComponents[i].nextBase;
      recyclerComponents.splice(i, 1);
      return inst;
    }
  }

  return inst;
}

/** The `.render()` method for a PFC backing instance. */
function doRender(props, state, context) {
  return this.constructor(props, context);
}
```
以上已经把 `Preact` 除了 Component render 以外的内容做了一个介绍，接下来详细分析核心的 Component 操作和 diff 算法部分 => [preact 源码分析 2](./preact源码解析2.md)