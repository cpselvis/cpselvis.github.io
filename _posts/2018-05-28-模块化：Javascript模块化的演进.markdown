ES2015 在2015年6月正式发布，官方终于引入了对于模块的原生支持，如今 JS 的模块化开发非常的方便、自然，但这个新规范仅仅持续了3年。就在7年前，JS 的模块化还停留在运行时的支持；13年前，通过后端模版定义、注释定义模块依赖。对于经历过的人来说，历史的模块化方式还停留在脑海中，久久不能忘怀。

## 为什么需要模块化

模块，是为了完成某一功能所需的程序或者子程序。模块是系统中“职责单一”且“可替换”的部分。所谓的模块化就是指把系统代码分为一系列职责单一且可替换的模块。模块化开发是指如何开发新的模块和复用已有的模块来实现应用的功能。

那么，为什么需要模块化呢？主要是以下几个方面的原因：

- 传统的网页开发正在转变成 Web Apps 开发
- 代码复杂度在逐步增高，随着 Web 能力的增强，越来越多的业务逻辑和交互都放在 Web 层实现
- 开发者想要分离的JS文件/模块，便于后续代码的维护性
- 部署时希望把代码优化成几个 HTTP 请求

## 模块化的演进历程

### 直接定义依赖(1999)

最先尝试将模块化引入到 JS 中是在1999年，称作“直接定义依赖”。这种方式实现模块化简单粗暴 —— 通过全局方法定义、引用模块。Dojo 就是使用这种方式进行模块组织的，使用`dojo.provide` 定义一个模块，`dojo.require` 来调用在其它地方定义的模块。

我们在dojo 1.6中修改下示例：

``` javascript
// greeting.js 文件
dojo.provide("app.greeting");

app.greeting.helloInLang = {
    en: 'Hello world!',
    es: '¡Hola mundo!',
    ru: 'Привет мир!'
};

app.greeting.sayHello = function (lang) {
    return app.greeting.helloInLang[lang];
};

// hello.js 文件
dojo.provide("app.hello");

dojo.require('app.greeting');

app.hello = function(x) {
    document.write(app.greeting.sayHello('es'));
};
```

直接定义依赖的方式和commonjs非常类似，区别是它可以在任何文件中定义模块，模块不和文件进行关联。而在commonjs中，每一个文件即是一个模块。

### namespace模式（2002）

最早，我们这么写代码：

``` javascript
function foo() {

}

function bar() {

}
```

很多变量和函数会直接在 global 全局作用域下面声明，很容易产生命名冲突。于是，命名空间模式被提出。比如：

``` javascript

var app = {};

app.foo = function() {

}

app.bar = function() {

}

app.foo();
```

### 匿名闭包 IIFE 模式(2003)

尽管 namespace 模式一定程度上解决了 global 命名空间上的变量污染问题，但是它没办法解决代码和数据隔离的问题。

解决这个问题的先驱是：匿名闭包 IIFE 模式。它最核心的思想是对数据和代码进行封装，并通过提供外部方法来对它们进行访问。一个基本的例子如下：

``` javascript

var greeting = (function () {
    var module = {};

    var helloInLang = {
        en: 'Hello world!',
        es: '¡Hola mundo!',
        ru: 'Привет мир!'
    };

    module.getHello = function (lang) {
        return helloInLang[lang];
    };

    module.writeHello = function (lang) {
        document.write(module.getHello(lang))
    };
    
    return module;
}());

```

### 模板依赖定义(2006)

模板依赖定义是接下来的一个用于解决模块化问题的方案，它需要配合后端的模板语法一起使用，通过后端语法聚合 js 文件，从而实现依赖加载。

它最开始被应用到 Prototype 1.4 库中。05年的时候，[Sam Stephenson](https://github.com/sstephenson) 开始开发 Prototype 库，Prototype 当时是作为 Ruby on rails 的一个组成部分。由于 Sam 平时使用 Ruby 很多，这也难怪他会选择使用 Ruby 里的 erb 模板作为 JS 模块的依赖管理了。

模板依赖定义的具体做法是：对于某个 JS 文件而言，如果它依赖其它 JS 文件，则在这个文件的头部通过特殊的标签语法去指定以来。这些标签语法可以通过后端模板（erb, jinjia, smarty）去解析，和特殊的构建工具如 borshik 去识别。只得一提的是，这种模式只能作用于**预编译**阶段。

下面是一个使用了 borshik 的例子：

``` javascript

// app.tmp.js 文件

/*borschik:include:../lib/main.js*/

/*borschik:include:../lib/helloInLang.js*/

/*borschik:include:../lib/writeHello.js*/

// main.js 文件
var app = {};

// helloInLang.js 文件
app.helloInLang = {
    en: 'Hello world!',
    es: '¡Hola mundo!',
    ru: 'Привет мир!'
};

// writeHello.js 文件
app.writeHello = function (lang) {
    document.write(app.helloInLang[lang]);
};

```

### 注释定义依赖(2006)

注释定义依赖和1999年的直接定义依赖的做法非常类似，不同的是，注释定义依赖是以文件为单位定义模块了。模块之间的依赖关系通过注释语法进行定义。

一个应用如果想用这种方式，可以通过预编译的方式（Mootools），或者在运行期动态的解析下载下来的代码(LazyJS)。

``` javascript
// helloInLang.js 文件
var helloInLang = {
    en: 'Hello world!',
    es: '¡Hola mundo!',
    ru: 'Привет мир!'
};

// sayHello.js 文件

/*! lazy require scripts/app/helloInLang.js */

function sayHello(lang) {
    return helloInLang[lang];
}

// hello.js 文件

/*! lazy require scripts/app/sayHello.js */

document.write(sayHello('en'));
```

### 依赖注入(2009)
2004年，Martin Fowler为了描述Java里的组件之间的通信问题，提出了依赖注入概念。

五年之后，前Sun和Adobe员工[Miško Hevery](https://github.com/mhevery)（JAVA程序员）开始为他的创业公司设计一个 JS 框架，这个框架主要使用依赖注入的思想却解决组件之间的通信问题。后来的结局大家都知道，他的项目被 Google 收购，Angular 成为最流行的 JS 框架之一。

``` javascript
// greeting.js 文件
angular.module('greeter', [])
    .value('greeting', {
        helloInLang: {
            en: 'Hello world!',
            es: '¡Hola mundo!',
            ru: 'Привет мир!'
        },

        sayHello: function(lang) {
            return this.helloInLang[lang];
        }
    });

// app.js 文件
angular.module('app', ['greeter'])
    .controller('GreetingController', ['$scope', 'greeting', function($scope, greeting) {
        $scope.phrase = greeting.sayHello('en');
    }]);
```

### CommonJS模式 (2009)

09年 CommonJS（或者称作 CJS）规范推出，并且最后在 Node.js 中被实现。

``` javascript
// greeting.js 文件
var helloInLang = {
    en: 'Hello world!',
    es: '¡Hola mundo!',
    ru: 'Привет мир!'
};

var sayHello = function (lang) {
    return helloInLang[lang];
}

module.exports.sayHello = sayHello;

// hello.js 文件
var sayHello = require('./lib/greeting').sayHello;
var phrase = sayHello('en');

console.log(phrase);
```

这里我们发现，为了实现CommonJS规范，我们需要两个新的入口 -- `require` 和 `module`，它们提供加载模块和对外界暴露接口的能力。

值得注意的是，无论是 require 还是 module，它们都是语言的关键字。在 Node.js 中，由于是辅助功能，我们可以使用它。在将Node.js中的模块发送给 V8 之前，会通过下面的函数进行包裹：

``` javascript
(function (exports, require, module, __filename, __dirname) {
    // ...
    // 模块的代码在这里
    // ...
});
```

就目前来看，CommonJS规范是最常见的模块格式规范。你不仅可以在 Node.js 中使用它，而且可以借助 Browserfiy 或 Webpack 在浏览器中使用。

### AMD 模式(2009)

CommonJS的工作在全力的推进，同时，关于模块的异步加载的[讨论](https://groups.google.com/forum/#!msg/commonjs/nbpX739RQ5o/SdpVQDtx88AJ)也越来越多。主要是希望解决 Web 应用的动态加载依赖，相比于 CommonJS，体积更小。

``` javascript
// lib/greeting.js 文件
define(function() {
    var helloInLang = {
        en: 'Hello world!',
        es: '¡Hola mundo!',
        ru: 'Привет мир!'
    };

    return {
        sayHello: function (lang) {
            return helloInLang[lang];
        }
    };
});

// hello.js 文件
define(['./lib/greeting'], function(greeting) {
    var phrase = greeting.sayHello('en');
    document.write(phrase);
});
```

虽然 AMD 的模式很适合浏览器端的开发，但是随着 npm 包管理的机制越来越流行，这种方式可能会逐步的被淘汰掉。

### ES2015 Modules(2015)

ES2015 modules(简称 ES modules)就是我们目前所使用的模块化方案，它已经在Node.js 9里原生支持，可以通过启动加上flag `--experimental-modules`使用，不需要依赖 babel 等工具。目前还没有被浏览器实现，前端的项目可以使用 babel 或 typescript 提前体验。

``` javascript
// lib/greeting.js 文件
const helloInLang = {
    en: 'Hello world!',
    es: '¡Hola mundo!',
    ru: 'Привет мир!'
};

export const greeting = {
    sayHello: function (lang) {
        return helloInLang[lang];
    }
};

// hello.js 文件
import { greeting } from "./lib/greeting";
const phrase = greeting.sayHello("en");
document.write(phrase);
```

## 总结

JS 模块化的演进历程让人感慨，感谢 TC 39 对 ES modules 的支持，让 JS 的模块化进程终于可以告一段落了，希望几年后所有的主流浏览器可以原生支持 ES modules。

JS 模块化的演进另一方面也说明了 Web 的能力在不断增强，Web 应用日趋复杂。相信未来 JS 的能力进一步提升，我们的开发效率也会更加高效。