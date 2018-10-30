# React.memo

2018 年 10 月 23 日发布的 React 16.6 的 changelog 里面有一条：

**Add React.memo() as an alternative to PureComponent for functions.**

这表示 React 里的函数式组件也能够变成 PureComponent 了，这是一条令人高兴的消息。

我刚好在前几天正在编写部门项目的组件库文档，在是否采用 functional component 这件事上，我的想法是 functional component 虽然能够在生命周期上节省部分性能，但 react 组件性能的瓶颈还是在 diff 算法上。而 functional component 无法作为 PureComponent 减少不必要的 diff 次数，因此还是不推荐编写函数式组件。

但是 `React.memo` 的出现改变了这一现状，只要使用 `React.memo` 包裹函数式组件，就可以使函数式组件变成 pureComponent。

```javascript
const MyComponent = React.memo(function MyComponent(props) {
  /* only rerenders if props change */
});
```

其实这不是一个非常难的功能，因为我看到的时候就在想，应该 React.memo 把 props 缓存了，然后在渲染的时候比对 props。
因为函数式组件是通过接收参数作为 props，每次重渲染的时候都会重新拿一遍参数。如果要变成 PureComponent，肯定需要缓存原来的 props 才能进行比对。

来看下 [commit](https://github.com/facebook/react/pull/13748/commits/3cb0a3fb43821781bd6cf1e126ce9a72f1884518)，基本上印证了我的想法。

```javascript
// memoizedProps 就是缓存的 props
const prevProps = current.memoizedProps;
// Default to shallow comparison
let compare = Component.compare;
compare = compare !== null ? compare : shallowEqual;
// 比较前后 props
if (compare(prevProps, nextProps)) {
  return bailoutOnAlreadyFinishedWork(
    current,
    workInProgress,
    renderExpirationTime,
  );
}
```

