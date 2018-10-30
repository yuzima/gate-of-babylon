# Webpack 打包文件分析

webpack 是目前前端开发使用最广泛的模块化打包工具，前端工程师在项目中对它算是很熟悉了。因为有像 webpack 这样的打包工具，才让前端工程师在开发中方便地使用各种模块规范写各种 Javascript 模块化代码，而不需要考虑运去兼容行代码的环境。

Javascript 的模块进程经历了很多的变动，比较知名的有 `CommonJS` `AMD` `CMD` `ES6 Modules`。随着 `ES6 Modules` 的落地，现在较广泛被使用的是 `ES6 Modules` 和 `CommonJS` 规范。值得注意的是在 ES6 刚推出模块化语法的时候，各大浏览器都还是不支持的，那么 webpack 如何打包我们的代码来让浏览器或 node 运行的？

## Webpack 打包出来的代码到底是什么样的？

首先 webpack 能够为多种环境或 target 构建编译，用法为：

```javascript
module.exports = {
  target: 'node'
};
```

> 完整 target 列表：[https://webpack.js.org/configuration/target/](https://webpack.js.org/configuration/target/)。

默认情况下，webpack target 值为 **web**，我们先来看 target 为 **web** 的打包结果。

先看一个使用了 `ES6 Modules` 的简单例子：

```javascript
// Foo.js
export default function Foo() {
  const sayHi = () => {
    console.log('Hi~');
  }
  return { sayHi };
}
```

```javascript
// index.js
import Foo from './Foo';

Foo.sayHi();
```

使用 webpack 打包后的结果（删除了自动生成的注释），为了方便阅读理解使用了 webpack 的 development 模式。

```javascript
(function(modules) {
  /* 此处先省略 */
})(
  {
    "./src/Foo.js":
      /*! exports provided: default */
      function(module, __webpack_exports__, __webpack_require__) {
        "use strict";
        eval(
          "__webpack_require__.r(__webpack_exports__);\n/* harmony export (binding) */ __webpack_require__.d(__webpack_exports__, \"default\", function() { return Foo; });\nfunction Foo() {\n  const sayHi = () => {\n    console.log('Hi~');\n  }\n  return { sayHi };\n}\n\n//# sourceURL=webpack:///./src/Foo.js?"
        );
      },
    "./src/index.js":
      /*! no exports provided */
      function(module, __webpack_exports__, __webpack_require__) {
        "use strict";
        eval(
          '__webpack_require__.r(__webpack_exports__);\n/* harmony import */ var _Foo__WEBPACK_IMPORTED_MODULE_0__ = __webpack_require__(/*! ./Foo */ "./src/Foo.js");\n\n\n_Foo__WEBPACK_IMPORTED_MODULE_0__["default"].sayHi();\n\n\n//# sourceURL=webpack:///./src/index.js?'
        );
      }
  }
);
```

简单分析一下结构就是，webpack 创建了一个立即执行函数，这个函数接收一个名为 **modules** 的 Object 类型参数 。在下方代码，把我们定义的模块对象传给了立即执行函数的参数。模块对象的 **key** 为模块对应文件的路径，**value** 为执行模块代码的函数，这个函数接收三个参数，分别表示：
- module：该文件对应的模块对象
- \_\_webpack_exports]\_\_：保存了该模块的 export 的对象
- \_\_webpack_require\_\_：保存了该模块 import 的对象

函数仅仅包含了一条 `eval` 语句，整理一下 `eval` 的内容看下。

先来看 **./src/Foo.js** 的 `eval`，后面部分和 ./src/Foo.js 中的内容一致就是 Foo 函数的声明。
前面部分就是 webpack 编译的 `export` 的结果。

```javascript
__webpack_require__.r(__webpack_exports__); // 设置该模块的 export 模式是 ES6 Modules
/* harmony export (binding) */
/* 设置 __webpack_exports__ 的 default 的属性的 getter 方法为 function() { return Foo; }，相当于：
/* Object.defineProperty(
/*   __webpack_require__,
/*   'default',
/*   { enumerable: true, get: function() { return Foo; } }
/* );
*/
__webpack_require__.d(
  __webpack_exports__,
  "default",
  function() { return Foo; }
);
function Foo() {
  const sayHi = () => {
    console.log('Hi~');
  }
  return { sayHi };
}
```

接着看 **./src/indx.js** 的 `eval`，我们可以看出来 import Foo from './Foo.js' 代码被编译成了 'var _Foo__WEBPACK_IMPORTED_MODULE_0__ = \_\_webpack_require\_\_(/*! ./Foo */ "./src/Foo.js")';

`__webpack_require__` 这个方法会去加载 "./src/Foo.js" 对应的模块内容并返回模块 'Foo' 的 export 对象。

```javascript
/** ./src/index.js **/
__webpack_require__.r(__webpack_exports__);  // 同样先设置该模块的 export 模式为 __esModule
/* harmony import */
// 创建了一个变量，使用 __webpack_require__ 来加载 'Foo' 模块的 export 对象
var _Foo__WEBPACK_IMPORTED_MODULE_0__ = __webpack_require__(/*! ./Foo */ "./src/Foo.js");
// 调用 export 对象 default 属性返回的 Foo 上的 sayHi 方法
_Foo__WEBPACK_IMPORTED_MODULE_0__["default"].sayHi();
```

这里我们大概已经能了解到 webpack 模块化是怎么运作的了，webpack 实现了一个类似于 CommonJS 的同步模块加载机制，只要代码里遇到 `__webpack_require__`(就是我们源码里的 import)，就会去加载该模块。接着我们再看下具体逻辑：

1. webpack 启动后先初化一个模块缓存对象
2. 然后定义了最核心的模块加载函数 **\_\_webpack_require\_\_**。
3. 在 **\_\_webpack_require\_\_** 暴露了一些方法和对象。
4. 最后使用 **\_\_webpack_require\_\_** 加载入口文件所对应的模块，并返回入口模块 export 输出的值。

```javascript
function(modules) {
  // webpack 启动
  // 模块缓存
  var installedModules = {};

  // The require function
  function __webpack_require__(moduleId) {
    // 检查该模块是否在缓存中，存在直接返回该模块的 exports 对象
    if (installedModules[moduleId]) {
      return installedModules[moduleId].exports;
    }
    // 否则创建新的模块，并添加到缓存中
    var module = (installedModules[moduleId] = {
      i: moduleId,
      l: false,
      exports: {}
    });

    // 执行模块函数
    modules[moduleId].call(
      module.exports,
      module,
      module.exports,
      __webpack_require__
    );

    // 标记这个模块已经被加载
    module.l = true;

    // 返回这个模块的 exports 输出
    return module.exports;
  }

  // 暴露所有模块集合对象 (__webpack_modules__)
  __webpack_require__.m = modules;

  // 暴露模块缓存对象
  __webpack_require__.c = installedModules;

  // d: 定义用于 export 的 getter 函数
  __webpack_require__.d = function(exports, name, getter) {
    if (!__webpack_require__.o(exports, name)) {
      Object.defineProperty(exports, name, { enumerable: true, get: getter });
    }
  };

  // 定义 export 对象上的 __esModule 属性
  __webpack_require__.r = function(exports) {
    if (typeof Symbol !== "undefined" && Symbol.toStringTag) {
      Object.defineProperty(exports, Symbol.toStringTag, { value: "Module" });
    }
    Object.defineProperty(exports, "__esModule", { value: true });
  };

  // 创建一个假的命名空间对象
  // mode & 1: value is a module id, require it
  // mode & 2: merge all properties of value into the ns
  // mode & 4: return value when already ns object
  // mode & 8|1: behave like require
  __webpack_require__.t = function(value, mode) {
    if (mode & 1) value = __webpack_require__(value);
    if (mode & 8) return value;
    if (mode & 4 && typeof value === "object" && value && value.__esModule)
      return value;
    var ns = Object.create(null);
    __webpack_require__.r(ns);
    Object.defineProperty(ns, "default", { enumerable: true, value: value });
    if (mode & 2 && typeof value != "string")
      for (var key in value)
        __webpack_require__.d(
          ns,
          key,
          function(key) {
            return value[key];
          }.bind(null, key)
        );
    return ns;
  };

  // getDefaultExport 函数，用于与非 harmony 模块的兼容性
  __webpack_require__.n = function(module) {
    var getter =
      module && module.__esModule
        ? function getDefault() {
            return module["default"];
          }
        : function getModuleExports() {
            return module;
          };
    __webpack_require__.d(getter, "a", getter);
    return getter;
  };

  // 判断对象上是否存在某属性
  __webpack_require__.o = function(object, property) {
    return Object.prototype.hasOwnProperty.call(object, property);
  };

  // __webpack_public_path__
  __webpack_require__.p = "";

  // 加载入口文件模块 './src/index.js' 并返回 exports
  return __webpack_require__((__webpack_require__.s = "./src/index.js"));
}
```

以上这就是一个非常简单的 webpack ES6 Modules 实现。整体逻辑就是从入口文件模块开始加载，加载代码过程中遇到 \_\_webpack_require\_\_() 代码时就去加载执行对应的模块。

然后将 target 换成 node 后再次生成代码，生成的结果是一样，说明 Webpack 编译的 ES6 Modules 模块同时满足在 node 和 web 平台执行。

如果我们写代码用的是 CommonJS 规范的 `require/module.exports` 来引入输出的包的，其编译后的代码结果也没有什么太大的差别，对 webpack 来说，它们只是引入输出的方法名不一样而已，其最终实现出来的结果都是一样的。

## Webpack 如何实现异步加载？

刚才测试样例生成的打包代码是同步加载的，但是在前端的模块越来越多，打包文件变得越来越庞大的情况下，在浏览器端，我们不得不考虑模块异步加载的问题。

旧版本的 webpack 提供了 `require.ensure` 的语法来实现异步加载，随着 ES 提案的 `Dynamic import()` 语法的提出，浏览器例如 Chrome 开始支持 `Dynamic import()` 语法，webpack `require.ensure` 该语法目前已被动态导入语法 `import()` 取代。

> `Dynamic import()` 语法目前处于 TC39 流程的 stage 3. 详见：https://github.com/tc39/proposal-dynamic-import

现在我们来动态导入 `lodash` 这个依赖。webpack.config.js 配置参考官网的演示 [webpack 动态导入](https://webpack.docschina.org/guides/code-splitting/#%E5%8A%A8%E6%80%81%E5%AF%BC%E5%85%A5-dynamic-imports-)。

`import()` 会返回一个原生 `Promise`，所以我们可以用 Promise 语法来编写代码。

此外需要知道的是，webpack 会自动单独打包使用 `import()` 加载的模块文件，将其生成为一个 **chunkFile**，对 webpack 来说一个单独打包的文件块就是一个 `chunk`，**我们可以理解一个 chunk 就相当于一个 module**，在代码里会使用 `chunkId` 来标识这个模块文件。

在这个例子里，我们通过 /* webpackChunkName: "Foo" */ 这段注释给 `chunk` 命名为 'Foo'。

```javascript
// index.js
async function getModule() {
  return await import(/* webpackChunkName: "Foo" */ './Foo');
}

getModule().then(module => {
  module.sayHi();
});
```

经过打包生成两个文件 **bundle.js** 和 **Foo.bundle.js**。**Foo.bundle.js** 就是被单独打包的 Foo 模块，它会被异步地加载到 **bundle.js** 中。

先看 **Foo.bundle.js** 是如何打包 lodash 的。

```javascript
(window["webpackJsonp"] = window["webpackJsonp"] || []).push([["Foo"],{
  "./src/Foo.js":
    (function(module, __webpack_exports__, __webpack_require__) {
      "use strict";
        eval("..."); // eval 里的内容与上一个同步加载例子里的 Foo 模块相同

    })
}]);
```

该文件会创建或更新全局变量 `window.webpackJsonp`，这个全局变量是一个数组，它存储了以 [[chunkIds], chunkObjects] 为格式的数组项。

**bundle.js** 文件源码。

```javascript
// bundle.js
(function(modules) {
  /* 此处先省略 */
})(
  {
    "./src/index.js":
    /*! no static exports found */
    (function(module, exports, __webpack_require__) {
      eval("async function getModule() {\n  return await __webpack_require__.e(/*! import() | Foo */ \"Foo\").then(__webpack_require__.bind(null, /*! ./Foo */ \"./src/Foo.js\"));\n}\n\ngetModule().then(module => {\n  module.sayHi();\n});\n\n//# sourceURL=webpack:///./src/index.js?");
    })
});
```

整理 `eval` 代码

```javascript
async function getModule() {
  // 异步请求 chunkId 为 'Foo' 的 chunk 并返回 Promise
  return await __webpack_require__.e(/*! import() | Foo */ "Foo")
    // 返回 Foo 模块 export 输出对象
    .then(__webpack_require__.bind(null, /*! ./Foo */ "./src/Foo.js"));
}
getModule().then(module => {
  module.sayHi();
});
```

**\_\_webpack_require\_\_.e** 相当于源码里的 `import()` 异步加载，**\_\_webpack_require\_\_** 这个方法我们刚已经知道是会返回模块 export 输出对象的，这里使用 **.bind(null)** 是为了不改变加载模块内部的 this 对象。
值得注意的是 webpack 在编译代码过程中，webpack 对模块的路径进行了转化，将原来的相对路径编译为了以根路径为起点的相对路径。

最后看下 **function(modules) {}** 具体内容，仅分析与同步加载不同的地方：

```javascript
function(modules) {
  // The module cache
  var installedModules = {};

  // 执行获取到的 chunk 代码并缓存记录
  // webpackJsonpCallback 就是用来接收前面 Foo.bundle.js 里
  // window["webpackJsonp"].push() 的那个 chunk 对象
  // 执行 webpackJsonpCallback 方法表示 script 标签已被加载完毕
  function webpackJsonpCallback(data) {
    var chunkIds = data[0];
    var moreModules = data[1];
    var moduleId, chunkId, i = 0, resolves = [];
    // 更新 installedChunks
    for(;i < chunkIds.length; i++) {
      chunkId = chunkIds[i];
      if(installedChunks[chunkId]) {
        // 将 chunkId 对应的 resolve 压到 resolves 数组里
        resolves.push(installedChunks[chunkId][0]);
      }
      // 更新 chunk 加载状态为 0：已加载
      installedChunks[chunkId] = 0;
    }
    // 更新 modules
    for(moduleId in moreModules) {
      if(Object.prototype.hasOwnProperty.call(moreModules, moduleId)) {
        modules[moduleId] = moreModules[moduleId];
      }
    }
    if (parentJsonpFunction) parentJsonpFunction(data);
    // 遍历执行所有 chunk 的 resolve
    // 这时候异步加载模块文件才算是完成了
    while(resolves.length) {
      resolves.shift()();
    }
  };

  // 记录已加载过的 chunk
  // undefined = chunk not loaded, null = chunk preloaded/prefetched
  // Promise = chunk loading, 0 = chunk loaded
  var installedChunks = {
    "main": 0
  };

  // 获取 chunk 所对应的文件地址的函数
  function jsonpScriptSrc(chunkId) {
    return __webpack_require__.p + "" + ({"Foo":"Foo"}[chunkId]||chunkId) + ".bundle.js"
  }

  // __webpack_require__ 没有变化

  // This file contains only the entry chunk.
  // 异步加载 chunk（相当于 module）
  __webpack_require__.e = function requireEnsure(chunkId) {
    var promises = [];

    // JSONP chunk 加载
    var installedChunkData = installedChunks[chunkId];
    if(installedChunkData !== 0) { // 0 means "already installed".
      // 非 0 但又不为空表示这个 chunk 正在被加载，直接添加它的 Promise 对象到 promises 列表里
      if(installedChunkData) {
        promises.push(installedChunkData[2]);
      } else {
        // 创建一个 promise 并记录到 chunk 缓存里
        var promise = new Promise(function(resolve, reject) {
          installedChunkData = installedChunks[chunkId] = [resolve, reject];
        });
        promises.push(installedChunkData[2] = promise);

        // 开始加载 chunk，创建 script 标签
        var head = document.getElementsByTagName('head')[0];
        var script = document.createElement('script');
        var onScriptComplete;

        // 获取请求 chunk 的文件路径
        script.src = jsonpScriptSrc(chunkId);

        onScriptComplete = function (event) {
          /* 检查请求是否出错，出错就 reject Promise */
        };

        // 请求结果处理
        script.onerror = script.onload = onScriptComplete;
        // 添加 script 标签到 head
        head.appendChild(script);
      }
    }
    return Promise.all(promises);
  };

  // 以下内控没有变化
  // __webpack_require__.m
  // __webpack_require__.c
  // __webpack_require__.d
  // __webpack_require__.r
  // __webpack_require__.t
  // __webpack_require__.n
  // __webpack_require__.o
  // __webpack_require__.p
  // expose the modules object (__webpack_modules__)

  // 异步加载错误处理
  __webpack_require__.oe = function(err) { console.error(err); throw err; };

  // Jsonp 处理，获取全局变量 window["webpackJsonp"]
  var jsonpArray = window["webpackJsonp"] = window["webpackJsonp"] || [];
  var oldJsonpFunction = jsonpArray.push.bind(jsonpArray);
  jsonpArray.push = webpackJsonpCallback;
  jsonpArray = jsonpArray.slice();
  for(var i = 0; i < jsonpArray.length; i++) webpackJsonpCallback(jsonpArray[i]);
  var parentJsonpFunction = oldJsonpFunction;

  return __webpack_require__(__webpack_require__.s = "./src/index.js");
}
```




