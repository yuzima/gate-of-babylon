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

简单分析一下结构就是，webpack 创建了一个立即执行函数，这个函数接收一个参数 **modules**。在代码的下方，把我们定义的两个文件的内容作为参数传给了立即执行函数。**key** 为文件的路径，**value** 为一个函数，这个函数接收三个参数，分别表示：
- module：该文件对应的模块对象
- \_\_webpack_exports]\_\_：保存了该模块的 export 的对象
- \_\_webpack_require\_\_：保存了该模块 import 的对象

函数仅仅包含了一条 `eval` 语句，整理一下 `eval` 的内容看下。

先来看 **./src/Foo.js** 的 `eval`，后面部分和 ./src/Foo.js 中的内容一致就是 Foo 函数的声明。
前面部分就是 webpack 编译的 `export` 的结果。

```javascript
__webpack_require__.r(__webpack_exports__); // 设置该模块的 export 模式是 ES6 Modules
/* harmony export (binding) */
/* 设置 __webpack_exports__ 对象的 default 的属性的 getter 方法为 function() { return Foo; }，相当于：
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

接着看 **./src/indx.js** 的 `eval`

```javascript
/** ./src/index.js **/
__webpack_require__.r(__webpack_exports__);  // 同样先设置该模块的 export 模式为 ES6 Modules
/* harmony import */
// 创建了一个变量，来获取 'Foo' 模块的 export 对象
var _Foo__WEBPACK_IMPORTED_MODULE_0__ = __webpack_require__(/*! ./Foo */ "./src/Foo.js");
// 调用 export 对象 default 属性返回的 Foo 上的 sayHi 方法
_Foo__WEBPACK_IMPORTED_MODULE_0__["default"].sayHi();
```

这里我们大概已经能了解到 webpack 模块化是怎么运作的了，接着我们再看下具体逻辑。

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

  // 最后加载入口文件模块 './src/index.js' 并返回 exports
  return __webpack_require__((__webpack_require__.s = "./src/index.js"));
}
```

以上这就是一个非常简单的 webpack ES6 Modules 实现。

## Webpack 如何实现异步加载？

## node 打包工具 rollUp