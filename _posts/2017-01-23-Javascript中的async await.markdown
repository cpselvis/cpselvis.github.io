---
layout: post
title: Javascript中的async await
date: 2017-01-23 17:52:24.000000000 +09:00
---

async / await是ES7的重要特性之一，也是目前社区里公认的优秀异步解决方案。目前，async / await这个特性已经是[stage 3](https://github.com/tc39/ecma262/tree/82bebe057c9fca355cfbfeb36be8e42f18c61e94)的建议，可以看看[TC39的进度](https://tc39.github.io/process-document/)，本篇文章将分享async / await是如何工作的，阅读本文前，希望你具备Promise、generator、yield等ES6的相关知识。

在详细介绍async / await之前，先回顾下目前在ES6中比较好的异步处理办法。下面的例子中数据请求用Node.js中的request模块，数据接口采用[Github v3 api文档](https://developer.github.com/v3/)提供的repo代码仓库详情API作为例子演示。

### Promise对异步的处理
虽然Node.js的异步IO带来了对高并发的良好支持，同时也让“回调”成为灾难，很容易造成回调地狱。传统的方式比如使用具名函数，虽然可以减少嵌套的层数，让代码看起来比较清晰。但是会造成比较差的编码和调试体验，你需要经常使用用ctrl + f去寻找某个具名函数的定义，这使得IDE窗口经常上下来回跳动。使用Promise之后，可以很好的减少嵌套的层数。另外Promise的实现采用了状态机，在函数里面可以很好的通过resolve和reject进行流程控制，你可以按照顺序链式的去执行一系列代码逻辑了。下面是使用Promise的一个例子：
```javascript
const request = require('request');
// 请求的url和header
const options = {
  url: 'https://api.github.com/repos/cpselvis/zhihu-crawler',
  headers: {
    'User-Agent': 'request'
  }
};
// 获取仓库信息
const getRepoData = () => {
  return new Promise((resolve, reject) => {
    request(options, (err, res, body) => {
      if (err) {
        reject(err);
      }
      resolve(body);
    });
  });
};

getRepoData()
  .then((result) => console.log(result);)
  .catch((reason) => console.error(reason););

// 此处如果是多个Promise顺序执行的话，如下：
// 每个then里面去执行下一个promise
// getRepoData()
//   .then((value2) => {return promise2})
//   .then((value3) => {return promise3})
//   .then((x) => console.log(x))
```

不过Promise仍然存在缺陷，它只是减少了嵌套，并不能完全消除嵌套。举个例子，对于多个promise串行执行的情况，第一个promise的逻辑执行完之后，我们需要在它的then函数里面去执行第二个promise，这个时候会产生一层嵌套。另外，采用Promise的代码看起来依然是异步的，如果写的代码如果能够变成同步该多好啊！

### Generator对异步的处理
谈到generator，你应该不会对它感到陌生。在Node.js中对于回调的处理，我们经常用的[TJ / Co](https://github.com/tj/co)就是使用generator结合promise来实现的，co是coroutine的简称，借鉴于python、lua等语言中的协程。它可以将异步的代码逻辑写成同步的方式，这使得代码的阅读和组织变得更加清晰，也便于调试。
```javascript
const co = require('co');
const request = require('request');

const options = {
  url: 'https://api.github.com/repos/cpselvis/zhihu-crawler',
  headers: {
    'User-Agent': 'request'
  }
};
// yield后面是一个生成器 generator
const getRepoData = function* () {
  return new Promise((resolve, reject) => {
    request(options, (err, res, body) => {
      if (err) {
        reject(err);
      }
      resolve(body);
    });
  });
};

co(function* () {
  const result = yield getRepoData;
  // ... 如果有多个异步流程，可以放在这里，比如
  // const r1 = yield getR1;
  // const r2 = yield getR2;
  // const r3 = yield getR3;
  // 每个yield相当于暂停，执行yield之后会等待它后面的generator返回值之后再执行后面其它的yield逻辑。
  return result;
}).then(function (value) {
  console.log(value);
}, function (err) {
  console.error(err.stack);
});
```

### async / await对异步的处理
虽然co是社区里面的优秀异步解决方案，但是并不是语言标准，只是一个过渡方案。ES7语言层面提供async / await去解决语言层面的难题。目前async / await 在 IE edge中已经可以直接使用了，但是chrome和Node.js还没有支持。幸运的是，babel已经支持async的transform了，所以我们使用的时候引入babel就行。在开始之前我们需要引入以下的package，preset-stage-3里就有我们需要的async/await的编译文件。

无论是在Browser还是Node.js端都需要安装下面的包。
``` shell
$ npm install babel-core --save
$ npm install babel-preset-es2015 --save
$ npm install babel-preset-stage-3 --save
```

这里推荐使用babel官方提供的require hook方法。就是通过require进来后，接下来的文件进行require的时候都会经过Babel的处理。因为我们知道CommonJs是同步的模块依赖，所以也是可行的方法。这个时候，需要编写两个文件，一个是启动的js文件，另外一个是真正执行程序的js文件。

启动文件index.js
```javascript
require('babel-core/register');
require('./async.js');
```

真正执行程序的async.js
```javascript
const request = require('request');

const options = {
  url: 'https://api.github.com/repos/cpselvis/zhihu-crawler',
  headers: {
    'User-Agent': 'request'
  }
};

const getRepoData = () => {
  return new Promise((resolve, reject) => {
    request(options, (err, res, body) => {
      if (err) {
        reject(err);
      }
      resolve(body);
    });
  });
};

async function asyncFun() {
 try {
    const value = await getRepoData();
    // ... 和上面的yield类似，如果有多个异步流程，可以放在这里，比如
    // const r1 = await getR1();
    // const r2 = await getR2();
    // const r3 = await getR3();
    // 每个await相当于暂停，执行await之后会等待它后面的函数（不是generator）返回值之后再执行后面其它的await逻辑。
    return value;
  } catch (err) {
    console.log(err);
  }
}

asyncFun().then(x => console.log(`x: ${x}`)).catch(err => console.error(err));

```
注意点：
* async用来申明里面包裹的内容可以进行同步的方式执行，await则是进行执行顺序控制，每次执行一个await，程序都会暂停等待await返回值，然后再执行之后的await。
* await后面调用的函数需要返回一个promise，另外这个函数是一个普通的函数即可，而不是generator。
* await只能用在async函数之中，用在普通函数中会报错。
* await命令后面的 Promise 对象，运行结果可能是 rejected，所以最好把 await 命令放在 try...catch 代码块中。

其实，async / await的用法和co差不多，await和yield都是表示暂停，外面包裹一层async 或者 co来表示里面的代码可以采用同步的方式进行处理。不过async / await里面的await后面跟着的函数不需要额外处理，co是需要将它写成一个generator的。
