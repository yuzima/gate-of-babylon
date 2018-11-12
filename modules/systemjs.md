# 除了 webpack 以外的其他模块化解决方案 —— system.js

虽然目前 webpack 使用范围很广，但是 webpack 配置项复杂，功能繁多，我们很多时候不一定需要它来做模块化，那么离开 webpack 如何实现模块化，这里有一个非常小的库能实现我们需要的模块化功能 —— System.js。

System.js 主要是用于旧版浏览器的 ES6 模块化加载配置。

system.js 包含两个基础版本。

#### 1. s.js 最小版本 loader

s.js 是大小仅 1.5KB 最小化版本的 loader，提供了一个工作流程，将为浏览器里的 ES module 编写的生产代码，转换成 System.register 模块格式，以便在旧版本浏览器中工作，包括 IE 11+。

ES module 语义（如实时绑定，循环引用，上下文 metadata，动态导入和 top level await）都可以通过这种方式得到完全支持，同时支持 CSP 和 cross-origin，这个流程可以作为类似 polyfill 的一种方式。

- 加载并解析模块为 URL，抛出指定模块名（例如 import 'lodash'），就像本地模块加载器。
- 加载通过 System.register 注册的模块
- 核心的可挂钩扩展 loader 支持自定义扩展。

#### 2. system.js loader

3KB 的 system.js loader 基于 s.js 核心并增加了对即将到来的模块特性（目前包含 package name 映射以及 WASM 与模块加载的集成）以及开发和方便的功能。

- 支持加载指定模块名通过 package name 映射表（映射表需要事先配置好），通过 `<script type="system-packagenamemap">` 加载（IE 等浏览器需要 `fetch` polyfill）。
- 包括用于加载全局脚本的全局加载，用于加载使用传统的 script 标签加载依赖项的库。
- 追踪 Hooks 和用于重加载流程的清除注册 API
- 支持加载基于 `.wasm` 文件扩展名的 WASM

分析核心的 s.js 部分，s.js 只 import 了两个功能文件，script-load.js 和 basic-resolve.js。

```javascript
// s.js
import './features/script-load.js';
import './features/basic-resolve.js';
```

system-core 是 system.js 的核心代码。script-load.js 和 basic-resolve.js 都只是往 `systemJSPrototype` 上添加了新的方法而已。

```javascript
// script-load.js
import { systemJSPrototype } from '../system-core';

let err;
// 给 window 添加 error 事件
if (typeof window !== 'undefined')
  window.addEventListener('error', function (e) {
    err = e.error;
  });

// 往 register 方法添加 err 初始化
const systemRegister = systemJSPrototype.register;
systemJSPrototype.register = function (deps, declare) {
  err = undefined;
  systemRegister.call(this, deps, declare);
};

// instantiate 用于实际添加模块的 URL 至页面的 script 标签
systemJSPrototype.instantiate = function (url, firstParentUrl) {
  const loader = this;
  return new Promise(function (resolve, reject) {
    const script = document.createElement('script');
    script.charset = 'utf-8';
    script.async = true;
    script.crossOrigin = 'anonymous';
    script.addEventListener('error', function () {
      reject(new Error('Error loading ' + url + (firstParentUrl ? ' from ' + firstParentUrl : '')));
    });
    // 加载成功后删除 script 标签，保持 head 干净，返回刚注册的模块
    script.addEventListener('load', function () {
      document.head.removeChild(script);
      // Note URL normalization issues are going to be a careful concern here
      if (err)
        return reject(err);
      else
        resolve(loader.getRegister());
    });
    script.src = url;
    document.head.appendChild(script);
  });
};
```

basic-resolve.js 往 systemJSPrototype 添加了一个 resolve 方法，用于解析模块相对于可选项 parentUrl 的模块标识。

```javascript
// basic-resolve.js
import { systemJSPrototype } from '../system-core.js';
import { resolveIfNotPlainOrUrl, baseUrl } from '../common.js';
systemJSPrototype.resolve = function (id, parentUrl) {
  const resolved = resolveIfNotPlainOrUrl(id, parentUrl || baseUrl);
  if (!resolved) {
    if (id.indexOf(':') !== -1)
      return id;
    throw new Error('Cannot resolve "' + id + (parentUrl ? '" from ' + parentUrl : '"'));
  }
  return resolved;
};
```

核心的代码 system-core.js

```javascript
// system-core.js
import { global } from './common.js';
export { systemJSPrototype, REGISTRY }

const hasSymbol = typeof Symbol !== 'undefined';
const toStringTag = hasSymbol && Symbol.toStringTag;
// 使用 Symbol 作为注册机名
const REGISTRY = hasSymbol ? Symbol() : '@';

function SystemJS () {
  this[REGISTRY] = {};
}

const systemJSPrototype = SystemJS.prototype;

/*
 * System.import(id [, parentURL]) -> Promise(Module)
 * 加载一个模块，包含一个可选参数 parentUrl
 */
systemJSPrototype.import = function (id, parentUrl) {
  const loader = this;
  // 返回 Promise.resolve 的 ES module 的命名空间
  return Promise.resolve(loader.resolve(id, parentUrl))
  .then(function (id) {
    const load = getOrCreateLoad(loader, id);
    return load.C || topLevelLoad(loader, load);
  });
}

/*
 * System.register(deps, declare)
 * 声明功能，定义符合 System.register 模块格式的模块
 * https://github.com/systemjs/systemjs/blob/master/docs/system-register.md
 */
let lastRegister;
systemJSPrototype.register = function (deps, declare) {
  lastRegister = [deps, declare];
};

/*
 * 返回 System.register 最后一次调用声明的模块
 */
systemJSPrototype.getRegister = function () {
  const _lastRegister = lastRegister;
  lastRegister = undefined;
  return _lastRegister;
};

/*
 * 获取模块或创建加载模块
 */
function getOrCreateLoad (loader, id, firstParentUrl) {
  // 检查注册机是否存在模块，已加载直接获取
  let load = loader[REGISTRY][id];
  if (load)
    return load;

  // 创建该模块命名空间
  const importerSetters = [];
  const ns = Object.create(null);
  if (toStringTag)
    Object.defineProperty(ns, toStringTag, { value: 'Module' });

  // 异步加载模块
  let instantiatePromise = Promise.resolve()
  .then(function () {
    return loader.instantiate(id, firstParentUrl);
  })
  .then(function (registration) {
    if (!registration)
      throw new Error('Module ' + id + ' did not instantiate');
    function _export (name, value) {
      // note if we have hoisted exports (including reexports)
      load.h = true;
      let changed = false;
      // 添加 export 的对象至命名空间对象
      if (typeof name !== 'object') {
        if (!(name in ns) || ns[name] !== value) {
          ns[name] = value;
          changed = true;
        }
      }
      else {
        for (let p in name) {
          let value = name[p];
          if (!(p in ns) || ns[p] !== value) {
            ns[p] = value;
            changed = true;
          }
        }
      }
      if (changed)
        for (let i = 0; i < importerSetters.length; i++)
          importerSetters[i](ns);
      return value;
    }
    // 执行 register 定义的 declare 声明函数
    // 如果 declare 有第二个参数，则传递 import 方法和 meta 作为第二个参数 _context
    const declared = registration[1](_export, registration[1].length === 2 ? {
      import: function (importId) {
        return loader.import(importId, id);
      },
      meta: loader.createContext(id)
    } : undefined);
    // load.e：模块执行代码
    load.e = declared.execute || function () {};
    // setters：每当绑定的依赖有更新，这个function array 会依照顺序执行
    // 对于没有导出的依赖项，可以不定义 setter 函数
    return [registration[0], declared.setters || []];
  });

  // script 加载完成后，继续加载依赖的 deps 模块
  const linkPromise = instantiatePromise
  .then(function (instantiation) {
    return Promise.all(instantiation[0].map(function (dep, i) {
      const setter = instantiation[1][i];
      return Promise.resolve(loader.resolve(dep, id))
      .then(function (depId) {
        const depLoad = getOrCreateLoad(loader, depId, id);
        // depLoad.I may be undefined for already-evaluated
        return Promise.resolve(depLoad.I)
        .then(function () {
          // 如有 setter 需要触发，则压入 depLoad.i 队列
          if (setter) {
            depLoad.i.push(setter);
            // only run early setters when there are hoisted exports of that module
            // the timing works here as pending hoisted export calls will trigger through importerSetters
            if (depLoad.h || !depLoad.I)
              setter(depLoad.n);
          }
          return depLoad;
        });
      })
    }))
    .then(function (depLoads) {
      load.d = depLoads;
    });
  });

  // disable unhandled rejections
  linkPromise.catch(function () {});

  // Captial letter = a promise function
  return load = loader[REGISTRY][id] = {
    id: id,
    // 注册到这个依赖的 setter functions
    // 维护这个是因为以后可能会继续增加
    i: importerSetters,
    // 模块命名空间
    n: ns,

    // 实例化 Promise
    I: instantiatePromise,
    // link
    L: linkPromise,
    // 模块是否添加了 export
    h: false,

    // 实例化完成后会被设置
    // 依赖加载记录
    d: undefined,
    // 模块的执行函数
    // 执行后立即设置为 Null 表示已经执行过
    // in such a case, pC should be used, and pLo, pLi will be emptied
    e: undefined,

    // On execution we have populated:
    // the execution error if any
    eE: undefined,
    // in the case of TLA, the execution promise
    E: undefined,

    // On execution, pLi, pLo, e cleared

    // 最顶层的加载完成 Promise
    C: undefined
  };
}

function topLevelLoad (loader, load) {
  return load.C = instantiateAll(loader, load, {})
  .then(function () {
    return postOrderExec(loader, load, {});
  })
  .then(function () {
    return load.n;
  });
}

function instantiateAll (loader, load, loaded) {
  if (!loaded[load.id]) {
    loaded[load.id] = true;
    // load.L may be undefined for already-instantiated
    return Promise.resolve(load.L)
    .then(function () {
      return Promise.all(load.d.map(function (dep) {
        return instantiateAll(loader, dep, loaded);
      }));
    })
  }
}

function postOrderExec (loader, load, seen) {
  if (seen[load.id])
    return;
  seen[load.id] = true;

  if (!load.e) {
    if (load.eE)
      throw load.eE;
    if (load.E)
      return load.E;
    return;
  }

  // 先执行依赖项代码，除非是循环依赖
  let depLoadPromises;
  load.d.forEach(function (depLoad) {
    if (TRACING) {
      try {
        const depLoadPromise = postOrderExec(loader, depLoad, seen);
        if (depLoadPromise)
          (depLoadPromises = depLoadPromises || []).push(depLoadPromise);
      }
      catch (err) {
        loader.onload(load.id, err);
        throw err;
      }
    }
    else {
      const depLoadPromise = postOrderExec(loader, depLoad, seen);
      if (depLoadPromise)
        (depLoadPromises = depLoadPromises || []).push(depLoadPromise);
    }
  });
  if (depLoadPromises) {
    if (TRACING)
      return Promise.all(depLoadPromises)
      .then(doExec)
      .catch(function (err) {
        loader.onload(load.id, err);
        throw err;
      });
    else
      return load.E = Promise.all(depLoadPromises).then(doExec);
  }

  return doExec();

  function doExec () {
    try {
      let execPromise = load.e.call(nullContext);
      if (execPromise) {
        if (TRACING)
          execPromise = execPromise.then(function () {
            load.C = load.n;
            load.E = null; // indicates completion
            loader.onload(load.id, null);
          }, function () {
            loader.onload(load.id, err);
            throw err;
          });
        else
          execPromise.then(function () {
            load.C = load.n;
            load.E = null;
          });
        execPromise.catch(function () {});
        return load.E = load.E || execPromise;
      }
      // (should be a promise, but a minify optimization to leave out Promise.resolve)
      load.C = load.n;
      if (TRACING) loader.onload(load.id, null);
    }
    catch (err) {
      if (TRACING) loader.onload(load.id, err);
      load.eE = err;
      throw err;
    }
    finally {
      load.L = load.I = undefined;
      load.e = null;
    }
  }
}

global.System = new SystemJS();
```

来整理一下 system.js 的流程是：

1. 创建一个使用 System.register 方法进行注册的模块
2. System.import 加载模块代码，
3. 获取模块 url 路径后，添加 script 标签，触发 load 事件后说明代码已经被执行，此时模块已经使用 System.register 注册完毕，可以 resolve 返回最新被注册的模块的 [deps, declare]。
4. 传入 **_export** 和 **_context** 参数执行模块的 declare 方法获得 declared 对象，返回 [deps, declared.setters]。
5. 模块加载完成后循环加载依赖项，如果某个依赖 setter 需要触发，则记录到依赖加载记录里。
6. 等待所有依赖加载完成并触发了 setters，这个模块就加载结束了。

## 符合 System.register 格式的模块

要使用 system.js 加载模块，首先需要使模块符合 System.register 声明格式。

System.register 可以被认为是一种新的模块格式，用于在 ES5 里支持 ES6 的模块语义。

这种格式支持：
- 实时绑定更新，包括重新导出，星型导出，命名空间导入，以及这些的任意组合
- `import.meta`，以及 `import.meta.url`
- 动态 `import()`
- 顶层 await
- 循环引用，包括函数提升（其中未执行模块中的函数可用于循环引用）

### 格式定义

具体格式参考: [https://github.com/systemjs/systemjs/blob/master/docs/system-register.md#format-definition](https://github.com/systemjs/systemjs/blob/master/docs/system-register.md#format-definition)

```javascript
System.register([...deps...], function (_export, _context) {
  return {
    setters: [...setters...],
    execute: function () {

    }
  };
});
```

System.register 的有点在于和 AMD 的 `define` 一样，它支持浏览器的 CSP 策略，也就是不支持自定义的 JS 评估，只运行来自授权 host 的 script tag。

### 例子

如何转换格式参考一下 ES module 的转换。

```javascript
import { p as q } from './dep';

var s = 'local';

export function func() {
  return q;
}

export class C {
}
```

->

```javascript
System.register(['./dep'], function($__export, $__moduleContext) {
  var s, C, q;
  function func() {
    return q;
  }
  $__export('func', func);
  return {
    setters: [
    // every time a dependency updates an export,
    // this function is called to update the local binding
    // the setter array matches up with the dependency array above
    function(m) {
      q = m.p;
    }
    ],
    execute: function() {
      // use the export function to update the exports of this module
      s = 'local';
      $__export('C', C = $traceurRuntime.createClass(...));
    }
  };
});
```

目前 rollup 和 typescript 都对 system.js 添加了支持，可以自动化进行 System.register 模块化的转换。

## system.js 使用

### 加载一个 System.regitser 模块

```html
<script src="system.js"></script>
<script>
  System.import('/js/main.js');
</script>
```

这里的 main.js 是使用 System.register 模块声明格式的。
