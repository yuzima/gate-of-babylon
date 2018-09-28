# Snabbdom 源码解析 2

上一篇介绍了 Snabbdom 核心部分的内容，patch 函数，diff 算法，vnode 向 HTMLElement 转化，以及模块机制，这篇介绍一些其他的内容。

## Hooks

hook 是一种挂钩 DOM 节点生命周期的方法。 Snabbdom 提供丰富的 hook 选择。模块既可以使用 hook 来扩展Snabbdom，也可以使用 hook 来在 virtual node 的生命周期中的所需位置执行任意代码。

| Name        | Triggered when                                     | Arguments to callback   |
| ----------- | -------------------------------------------------- | ----------------------- |
| `pre`       | the patch process begins                           | none                    |
| `init`      | a vnode has been added                             | `vnode`                 |
| `create`    | a DOM element has been created based on a vnode    | `emptyVnode, vnode`     |
| `insert`    | an element has been inserted into the DOM          | `vnode`                 |
| `prepatch`  | an element is about to be patched                  | `oldVnode, vnode`       |
| `update`    | an element is being updated                        | `oldVnode, vnode`       |
| `postpatch` | an element has been patched                        | `oldVnode, vnode`       |
| `destroy`   | an element is directly or indirectly being removed | `vnode`                 |
| `remove`    | an element is directly being removed from the DOM  | `vnode, removeCallback` |
| `post`      | the patch process is done                          | none                    |

以下 hooks 可用于模块：`pre`，`create`，`update`，`destroy`，`remove`，`post`。

以下 hooks 在各个 element 的 hook 属性中可用：`init`，`create`，`insert`，`prepatch`，`update`，`postpatch`，`destroy`，`remove`。

要使用钩子，请将它们作为对象传递给数据对象参数的 hook 字段。

 ```js
h('div.row', {
  key: movie.rank,
  hook: {
    insert: (vnode) => { movie.elmHeight = vnode.elm.offsetHeight; }
  }
});
 ```

## Hero 过渡动画

在 Snabbdom 的模块文件里的每个模块都会处理 DOM 相应的功能，命名都很直观，但是有一个 hero.ts 文件，一开始我不知道它是用来处理什么的，后来我看到作者给出的 Hero transtion 的范例，我猜想 hero 应该是来处理 transition 动画的。

### hero.ts

hero 也是一个模块，因此也遵循 Snabbdom 模块输出 hooks 的要求，hero 输出了 `pre` `create` `destroy` `post` 这几个 hook 函数。

- pre：初始化 removed 和 created
- create：记录 vnode.data 里包含 hero.id 的 vnode 节点
- destroy：记录被移除的包含 hero.id 的 vnode 节点

```js
import {VNode, VNodeData} from '../vnode';
import {Module} from './module';

...

var removed: any, created: any;

// `pre` 钩子，初始化 removed 和 created
function pre() {
  removed = {};
  created = [];
}

// `create` 钩子
function create(oldVnode: VNode, vnode: VNode): void {
  var hero = (vnode.data as VNodeData).hero;
  // vnode.data 包含 hero.id 字段，hero.id 和 vnode 都添加到 created 列表
  if (hero && hero.id) {
    created.push(hero.id);
    created.push(vnode);
  }
}

// `destroy` 钩子
function destroy(vnode: VNode): void {
  var hero = (vnode.data as VNodeData).hero;
  if (hero && hero.id) {
    var elm = vnode.elm;
    (vnode as any).isTextNode = isTextElement(elm as Element | Text); // 是否为文本节点?
    // 将边界矩形 DOMRect 保存到 vnode 的一个新属性
    (vnode as any).boundingRect = (elm as Element).getBoundingClientRect(); 
    // 保存文本节点的边界矩形
    (vnode as any).textRect = (vnode as any).isTextNode ? getTextNodeRect((elm as Element).childNodes[0] as Text) : null;
    // 获取当前的 styles 样式
    var computedStyle = window.getComputedStyle(elm as Element, void 0); 
    // 保存一个 computedStyle 副本到 vnode
    (vnode as any).savedStyle = JSON.parse(JSON.stringify(computedStyle));
    // 最后将该 vnode 推入到删除列表里
    removed[hero.id] = vnode;
  }
}

function post() {
    ...
}

export const heroModule = {pre, create, destroy, post} as Module;
export default heroModule;
```

Hero 模块的主要钩子函数是 `post`，通过查阅上面的 hook 列表，可以知道这个钩子它发生在 patch 阶段执行完毕的时候。这个时候我们已经得到了 created 列表以及被移除节点的 removed 映射表，那么 post 钩子的功能是什么呢？post 函数的功能简单来说，就是实现一个被移动的节点的过渡动画。hero.id 用来表示这个节点的 key。

```js
function post() {
  var i: number, id: any, newElm: Element, oldVnode: VNode, oldElm: Element,
      hRatio: number, wRatio: number,
      oldRect: ClientRect, newRect: ClientRect, dx: number, dy: number,
      origTransform: string | null, origTransition: string | null,
      newStyle: CSSStyleDeclaration, oldStyle: CSSStyleDeclaration,
      newComputedStyle: CSSStyleDeclaration, isTextNode: boolean,
      newTextRect: ClientRect | undefined, oldTextRect: ClientRect | undefined;
  for (i = 0; i < created.length; i += 2) {
    id = created[i];
    newElm = created[i+1].elm;
    oldVnode = removed[id]; // 待移除的 vnode
    if (oldVnode) {
      isTextNode = (oldVnode as any).isTextNode && isTextElement(newElm); //Are old & new both text?
      newStyle = (newElm as HTMLElement).style;
      newComputedStyle = window.getComputedStyle(newElm, void 0); //get full computed style for new element
      oldElm = oldVnode.elm as Element;
      oldStyle = (oldElm as HTMLElement).style;
      // 获取元素新的 bounding boxes
      newRect = newElm.getBoundingClientRect();
      oldRect = (oldVnode as any).boundingRect; // 之前保存的 bounding box
      //Text node bounding boxes & distances
      if (isTextNode) {
        newTextRect = getTextNodeRect(newElm.childNodes[0] as Text);
        oldTextRect = (oldVnode as any).textRect;
        dx = getTextDx(oldTextRect, newTextRect);
        dy = getTextDy(oldTextRect, newTextRect);
      } else {
        //Calculate distances between old & new positions
        dx = oldRect.left - newRect.left;
        dy = oldRect.top - newRect.top;
      }
      hRatio = newRect.height / (Math.max(oldRect.height, 1));
      wRatio = isTextNode ? hRatio : newRect.width / (Math.max(oldRect.width, 1)); //text scales based on hRatio
      // Animate new element
      origTransform = newStyle.transform;
      origTransition = newStyle.transition;
      if (newComputedStyle.display === 'inline') //inline elements cannot be transformed
        newStyle.display = 'inline-block';        //this does not appear to have any negative side effects
      newStyle.transition = origTransition + 'transform 0s';
      newStyle.transformOrigin = calcTransformOrigin(isTextNode, newTextRect, newRect);
      newStyle.opacity = '0';
      newStyle.transform = origTransform + 'translate('+dx+'px, '+dy+'px) ' +
                               'scale('+1/wRatio+', '+1/hRatio+')';
      setNextFrame(newStyle, 'transition', origTransition);
      setNextFrame(newStyle, 'transform', origTransform);
      setNextFrame(newStyle, 'opacity', '1');
      // Animate old element
      for (var key in (oldVnode as any).savedStyle) { //re-apply saved inherited properties
        if (parseInt(key) != key as any as number) {
          var ms = key.substring(0,2) === 'ms';
          var moz = key.substring(0,3) === 'moz';
          var webkit = key.substring(0,6) === 'webkit';
          if (!ms && !moz && !webkit) //ignore prefixed style properties
            (oldStyle as any)[key] = (oldVnode as any).savedStyle[key];
        }
      }
      oldStyle.position = 'absolute';
      oldStyle.top = oldRect.top + 'px'; //start at existing position
      oldStyle.left = oldRect.left + 'px';
      oldStyle.width = oldRect.width + 'px'; //Needed for elements who were sized relative to their parents
      oldStyle.height = oldRect.height + 'px'; //Needed for elements who were sized relative to their parents
      oldStyle.margin = '0'; //Margin on hero element leads to incorrect positioning
      oldStyle.transformOrigin = calcTransformOrigin(isTextNode, oldTextRect, oldRect);
      oldStyle.transform = '';
      oldStyle.opacity = '1';
      document.body.appendChild(oldElm);
      setNextFrame(oldStyle, 'transform', 'translate('+ -dx +'px, '+ -dy +'px) scale('+wRatio+', '+hRatio+')'); //scale must be on far right for translate to be correct
      setNextFrame(oldStyle, 'opacity', '0');
      oldElm.addEventListener('transitionend', function (ev: TransitionEvent) {
        if (ev.propertyName === 'transform')
          document.body.removeChild(ev.target as Node);
      });
    }
  }
  removed = created = undefined;
}
```

