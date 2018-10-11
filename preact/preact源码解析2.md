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

    if (typeof childComponent==='function') {
      // set up high order component link
      let childProps = getNodeProps(rendered);
      inst = initialChildComponent;

      if (inst && inst.constructor===childComponent && childProps.key==inst.__key) {
        setComponentProps(inst, childProps, SYNC_RENDER, context, false);
      }
      else {
        toUnmount = inst;

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
      toUnmount = initialChildComponent;
      if (toUnmount) {
        cbase = component._component = null;
      }

      if (initialBase || renderMode===SYNC_RENDER) {
        if (cbase) cbase._component = null;
        base = diff(cbase, rendered, context, mountAll || !isUpdate, initialBase && initialBase.parentNode, true);
      }
    }

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

    component.base = base;
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

  if (!isUpdate || mountAll) {
    mounts.push(component);
  }
  else if (!skip) {
    // 确保子组件的 componentDidMount 在父组件的 componentDidMount 之前调用
    // Note: disabled as it causes duplicate hooks, see https://github.com/developit/preact/issues/750
    // flushMounts();
    if (component.componentDidUpdate) {
      component.componentDidUpdate(previousProps, previousState, snapshot);
    }
    if (options.afterUpdate) options.afterUpdate(component);
  }

  // 渲染结束，执行 _renderCallbacks 里的所有渲染回调
  while (component._renderCallbacks.length) component._renderCallbacks.pop().call(component);

  if (!diffLevel && !isChild) flushMounts();
}
```

### vdom/diff.js
