# preact 源码解析 2

## 源码解析

### vdom/component.js
该文件提供了 Component 的操作方法
- setComponentProps：设置组件的 props，并可能重新渲染 Component
- renderComponent：渲染 Component
- buildComponentFromVNode：从 VNode 创建 Component
- unmountComponent：卸载 Component

```javascript
import { SYNC_RENDER, NO_RENDER, FORCE_RENDER, ASYNC_RENDER, ATTR_KEY } from '../constants';
import options from '../options';
import { extend, applyRef } from '../util';
import { enqueueRender } from '../render-queue';
import { getNodeProps } from './index';
import { diff, mounts, diffLevel, flushMounts, recollectNodeTree, removeChildren } from './diff';
import { createComponent, recyclerComponents } from './component-recycler';
import { removeNode } from '../dom/index';

// 设置组件 props
export function setComponentProps(component, props, renderMode, context, mountAll) {
  if (component._disable) return;

  // 更新中，此时组件不可用
  component._disable = true;

  // 单独设置 ref 和 key
  component.__ref = props.ref;
  component.__key = props.key;
  delete props.ref;
  delete props.key;

  // see: https://reactjs.org/docs/react-component.html#static-getderivedstatefromprops
  if (typeof component.constructor.getDerivedStateFromProps === 'undefined') {
    // 如果组件还未被挂载，调用 componentWillMount 钩子
    if (!component.base || mountAll) {
      if (component.componentWillMount) component.componentWillMount();
    }
    // 如果组件已经被挂载，调用 componentWillReceiveProps 钩子
    else if (component.componentWillReceiveProps) {
      component.componentWillReceiveProps(props, context);
    }
  }

  // 更新 component.context
  if (context && context!==component.context) {
    if (!component.prevContext) component.prevContext = component.context;
    component.context = context;
  }

  // 更新 component.props
  if (!component.prevProps) component.prevProps = component.props;
  component.props = props;

  // 更新结束，组件恢复可用
  component._disable = false;

  if (renderMode!==NO_RENDER) {
    // 进行同步渲染的几种情况
    if (renderMode===SYNC_RENDER || options.syncComponentUpdates!==false || !component.base) {
      renderComponent(component, SYNC_RENDER, mountAll);
    }
    // 异步队列渲染
    else {
      enqueueRender(component);
    }
  }

  // 应用新 component.__ref
  applyRef(component.__ref, component);
}

// 渲染组件
/**
 * 渲染 Component, 触发必要的生命周期事件，同时高阶组件也在考虑之中
 * @param {import('../component').Component} component The component to render
 * @param {number} [renderMode] 渲染模式, 选项见 constants.js.
 * @param {boolean} [mountAll] 是否直接挂载所有组件
 * @param {boolean} [isChild] ?
 * @private
 */
export function renderComponent(component, renderMode, mountAll, isChild) {
  if (component._disable) return;

  let props = component.props,
    state = component.state,
    context = component.context,
    previousProps = component.prevProps || props,
    previousState = component.prevState || state,
    previousContext = component.prevContext || context,
    isUpdate = component.base, // 存在 component.base 表示已经初始化渲染过了，这次是更新
    nextBase = component.nextBase,
    initialBase = isUpdate || nextBase, // 初始化渲染时，nextBase 可以作为备用 DOM
    initialChildComponent = component._component,
    skip = false,
    snapshot = previousContext,
    rendered, inst, cbase;

  // 调用 getDerivedStateFromProps 钩子，它会返回一个对象来更新状态
  if (component.constructor.getDerivedStateFromProps) {
    state = extend(extend({}, state), component.constructor.getDerivedStateFromProps(props, state));
    component.state = state;
  }

  // 如果 Component 已经初始化渲染过了，这次就是更新，需要调用更新的钩子
  if (isUpdate) {
    // 调用 shouldComponentUpdate 时需要新旧值的比较
    // 因此要先恢复 component 的 props，state，context 为旧值
    component.props = previousProps;
    component.state = previousState;
    component.context = previousContext;
    // 如果 render 模式不是 FORCE_RENDER
    // 且 shouldComponentUpdate() 返回值为 false 则跳过这次渲染
    if (renderMode!==FORCE_RENDER
      && component.shouldComponentUpdate
      && component.shouldComponentUpdate(props, state, context) === false) {
      skip = true;
    }
    // 否则调用 componentWillUpdate 钩子
    else if (component.componentWillUpdate) {
      component.componentWillUpdate(props, state, context);
    }
    component.props = props;
    component.state = state;
    component.context = context;
  }

  component.prevProps = component.prevState = component.prevContext = component.nextBase = null;
  component._dirty = false;

  if (!skip) {
    // 从 component.render 获取需要渲染的内容
    rendered = component.render(props, state, context);

    // context to pass to the child, can be updated via (grand-)parent component
    if (component.getChildContext) {
      context = extend(extend({}, context), component.getChildContext());
    }

    // see: https://react.docschina.org/docs/react-component.html#getsnapshotbeforeupdate
    // getSnapshotBeforeUpdate 返回的值将被作为参数传递给 componentDidUpdate
    if (isUpdate && component.getSnapshotBeforeUpdate) {
      snapshot = component.getSnapshotBeforeUpdate(previousProps, previousState);
    }

    let childComponent = rendered && rendered.nodeName,
      toUnmount, base;

    // 如果 render 返回的 rendered.nodeName 为 function，表示这是一个高阶组件
    if (typeof childComponent==='function') {
      // 建立高阶组件链接
      let childProps = getNodeProps(rendered);
      inst = initialChildComponent;

      // 检查 component._component 是否存在，且是否是相同的构造函数，是否有相同的 key
      // 以上都通过，表示这只是一次更新，只修改 props 就可以了
      if (inst && inst.constructor===childComponent && childProps.key==inst.__key) {
        setComponentProps(inst, childProps, SYNC_RENDER, context, false);
      }
      // 否则卸载旧的组件实例 inst，重新创建子组件实例，更新 component._component
      // 同时递归调用 renderComponent 渲染新子组件实例
      else {
        toUnmount = inst;
        // 创建 render 返回的子组件 childComponent
        component._component = inst = createComponent(childComponent, childProps, context);
        inst.nextBase = inst.nextBase || nextBase;
        inst._parentComponent = component;
        setComponentProps(inst, childProps, NO_RENDER, context, false);
        renderComponent(inst, SYNC_RENDER, mountAll, true);
      }

      base = inst.base;
    }
    else {
      cbase = initialBase;
      // destroy high order component link
      // 如果不是高阶组件，但是之前有子组件，说明之前是高阶组件，但现在不是了
      // unmount 之前的子组件，并清空 component._component 和 cbase
      toUnmount = initialChildComponent;
      if (toUnmount) {
        cbase = component._component = null;
      }

      // 存在 base 或 nextBase 且是 SYNC_RENDER，立即执行 diff 算法获取新的 DOM 返回到 base 里
      if (initialBase || renderMode===SYNC_RENDER) {
        if (cbase) cbase._component = null;
        base = diff(cbase, rendered, context, mountAll || !isUpdate, initialBase && initialBase.parentNode, true);
      }
    }

    // 如果计算得到的新 DOM 和原来的 DOM 不一样，则使用 replaceChild 来替换 DOM
    if (initialBase && base!==initialBase && inst!==initialChildComponent) {
      let baseParent = initialBase.parentNode;
      if (baseParent && base!==baseParent) {
        baseParent.replaceChild(base, initialBase);

        if (!toUnmount) {
          initialBase._component = null;
          recollectNodeTree(initialBase, false);
        }
      }
    }

    if (toUnmount) {
      unmountComponent(toUnmount);
    }

    // 设置 component.base 为新的 DOM
    component.base = base;
    // render 时不是作为子组件渲染的，需要向上追溯设置 _parentComponent.base
    if (base && !isChild) {
      let componentRef = component,
        t = component;
      while ((t=t._parentComponent)) {
        (componentRef = t).base = base;
      }
      base._component = componentRef;
      base._componentConstructor = componentRef.constructor;
    }
  }

  // ?
  if (!isUpdate || mountAll) {
    mounts.push(component);
  }
  else if (!skip) {
    // 确保子组件的 componentDidMount 在父组件的 componentDidUpdate 之前调用
    // Note: disabled as it causes duplicate hooks, see https://github.com/developit/preact/issues/750
    // flushMounts();
    if (component.componentDidUpdate) {
      component.componentDidUpdate(previousProps, previousState, snapshot);
    }
    if (options.afterUpdate) options.afterUpdate(component);
  }

  // 渲染结束，执行 _renderCallbacks 里的所有渲染回调
  while (component._renderCallbacks.length) component._renderCallbacks.pop().call(component);

  // 当前 diff 递归在第一层，并且 Component 非子组件
  // flushMounts 会排队调用已挂载的组件列表的 componentDidMount 方法
  // 但此处似乎和 diff 方法里的 flushMounts 重复了？目前还未确认
  if (!diffLevel && !isChild) flushMounts();
}
```

### vdom/diff.js

`diff.js` 包含了 VNode 的更新比较算法，为了尽量减少在更新时对 DOM 的操作，提升性能，diff 的算法是至关重要的部分。

由于不是每次 diff 后立即对应调用 `componentDidMount` 方法，要等到本次 diff 的所有子节点递归都结束了，才会调用 `flushMounts` 按顺序依次调用，因此把具体的比较算法剥离开来放在 `idiff` 方法里。

```javascript
import { ATTR_KEY } from '../constants';
import { isSameNodeType, isNamedNode } from './index';
import { buildComponentFromVNode } from './component';
import { createNode, setAccessor } from '../dom/index';
import { unmountComponent } from './component';
import options from '../options';
import { applyRef } from '../util';
import { removeNode } from '../dom/index';

// 已挂载并正在等待 componentDidMount 的组件队列
export const mounts = [];

// diff 递归计数，用于跟踪 diff 循环的结束
export let diffLevel = 0;

// 全局标志，指示当前是否在 diff SVG 中
let isSvgMode = false;

// 全局标志，指示 diff 是否执行 hydration
let hydrating = false;

// 排队调用已挂载的组件列表的 componentDidMount 方法
export function flushMounts() {
  let c, i;
  for (i=0; i<mounts.length; ++i) {
    c = mounts[i];
    if (options.afterMount) options.afterMount(c);
    if (c.componentDidMount) c.componentDidMount();
  }
  mounts.length = 0;
}

// 将 VNode 的差异(以及它的深层子节点)应用到实际的 DOM 节点
export function diff(dom, vnode, context, mountAll, parent, componentRoot) {
  // diffLevel 在这里为 0 表示初始进入 diff
  if (!diffLevel++) {
    // 当第一次启动 diff 时，检查是在比较 SVG 还是在 SVG 中比较
    isSvgMode = parent!=null && parent.ownerSVGElement!==undefined;

    // hydration 表示目前的元素将在没有 prop 缓存的情况下进行 diff
    // 如果 dom 中存在 ATTR_KEY 这个属性，则关闭 hydration
    hydrating = dom!=null && !(ATTR_KEY in dom);
  }

  // 调用内部 diff 算法，返回更新好的 DOM 元素
  let ret = idiff(dom, vnode, context, mountAll, componentRoot);

  // 如果新旧父元素不一样，直接 appendChild 到新父元素下面
  if (parent && ret.parentNode!==parent) parent.appendChild(ret);

  // diffLevel 降低到 0 意味着正在结束 diff
  if (!--diffLevel) {
    hydrating = false;
    // 排队调用已挂载的组件列表的 componentDidMount 方法
    if (!componentRoot) flushMounts();
  }

  return ret;
}

// diff 的内容具体实现，剥离开来以允许绕过 flushMounts
function idiff(dom, vnode, context, mountAll, componentRoot) {
  let out = dom,
    prevSvgMode = isSvgMode;

  // empty values (null, undefined, booleans) render as empty Text nodes
  if (vnode==null || typeof vnode==='boolean') vnode = '';

  // Fast case: Strings & Numbers create/update Text nodes.
  if (typeof vnode==='string' || typeof vnode==='number') {

    // update if it's already a Text node:
    if (dom && dom.splitText!==undefined && dom.parentNode && (!dom._component || componentRoot)) {
      /* istanbul ignore if */ /* Browser quirk that can't be covered: https://github.com/developit/preact/commit/fd4f21f5c45dfd75151bd27b4c217d8003aa5eb9 */
      if (dom.nodeValue!=vnode) {
        dom.nodeValue = vnode;
      }
    }
    else {
      // it wasn't a Text node: replace it with one and recycle the old Element
      out = document.createTextNode(vnode);
      if (dom) {
        if (dom.parentNode) dom.parentNode.replaceChild(out, dom);
        recollectNodeTree(dom, true);
      }
    }
    out[ATTR_KEY] = true;
    return out;
  }

  // If the VNode represents a Component, perform a component diff:
  let vnodeName = vnode.nodeName;
  if (typeof vnodeName==='function') {
    return buildComponentFromVNode(dom, vnode, context, mountAll);
  }


  // Tracks entering and exiting SVG namespace when descending through the tree.
  isSvgMode = vnodeName==='svg' ? true : vnodeName==='foreignObject' ? false : isSvgMode;


  // If there's no existing element or it's the wrong type, create a new one:
  vnodeName = String(vnodeName);
  if (!dom || !isNamedNode(dom, vnodeName)) {
    out = createNode(vnodeName, isSvgMode);

    if (dom) {
      // move children into the replacement node
      while (dom.firstChild) out.appendChild(dom.firstChild);

      // if the previous Element was mounted into the DOM, replace it inline
      if (dom.parentNode) dom.parentNode.replaceChild(out, dom);

      // recycle the old element (skips non-Element node types)
      recollectNodeTree(dom, true);
    }
  }


  let fc = out.firstChild,
    props = out[ATTR_KEY],
    vchildren = vnode.children;

  if (props==null) {
    props = out[ATTR_KEY] = {};
    for (let a=out.attributes, i=a.length; i--; ) props[a[i].name] = a[i].value;
  }

  // Optimization: fast-path for elements containing a single TextNode:
  if (!hydrating && vchildren && vchildren.length===1 && typeof vchildren[0]==='string' && fc!=null && fc.splitText!==undefined && fc.nextSibling==null) {
    if (fc.nodeValue!=vchildren[0]) {
      fc.nodeValue = vchildren[0];
    }
  }
  // otherwise, if there are existing or new children, diff them:
  else if (vchildren && vchildren.length || fc!=null) {
    innerDiffNode(out, vchildren, context, mountAll, hydrating || props.dangerouslySetInnerHTML!=null);
  }


  // Apply attributes/props from VNode to the DOM Element:
  diffAttributes(out, vnode.attributes, props);


  // restore previous SVG mode: (in case we're exiting an SVG namespace)
  isSvgMode = prevSvgMode;

  return out;
}

```