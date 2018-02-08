本篇文章主要介绍腾讯IVWEB团队从0到1在工程化这块的思考和实践。我所认为的工程化包含两个部分：工作流和规范。工作流是从项目初始化、本地开发、打包构建到最终发布的整个过程；规范贯穿这个过程，通过它让项目的目录结构一致、代码质量更高、Git提交格式统一、README文档更加实用。工程化最大的好处是统一团队的开发标准、提升开发效率。

工程化是否是必须的呢？不是。团队人数很少的，或者项目没有那么复杂的时候并不需要工程化。

### 什么时候需要工程化？
曾经我们也是混乱的，新项目手工拷贝旧项目，随意编写git commit格式，构建脚本在每个项目都存在一份，代码也没有相关规范约束。但那时团队人数只有10人左右，经过1年的业务发展和人员膨胀，已经极速扩展到20多人。这个时候就需要依托工具去保障团队的一致开发标准。

另一方面，随着日趋复杂的前端工程的演进。随便一个新项目都会涉及到React、Redux、Vue、Mobx、AntD、ES6、Babel、Webpack等等。徒手搭出这种现代化的工程不仅费时费力，而且倒腾开发环境就需要一整天。这个时候需要工程化去实现开发环境快速搭建的问题。

互联网时代的产品，通常会面对海量用户。很多时候我们会在代码中加各种各样的行为埋点、JS错误监控等代码。申请Id的工作很多时候都是枯燥无味和重复的，因此需要利用工程化去批量化申请这些埋点Id并且内置到项目的配置文件中。

为了解决上述问题，我们于17年2月底开始投入工程化feflow工具的开发和相关规范的制定，目前已经研发出了 feflow 的 [CLI](https://github.com/feflow/feflow) 版本，后续会推出 GUI 版本。

### 架构设计
![](https://user-gold-cdn.xitu.io/2018/2/8/1617621e74c41e6c?w=1528&h=780&f=jpeg&s=130447)

为了让 feflow 的具有高可扩展性，我们设计了4层结构，分别是：插件生态、内核层、参数解析器和控制台。除了贯穿整个开发工作流的基础命令选择通过内部插件内置在CLI 的Core里面，其它非必要命令统一通过插件机制进行扩展。

另一方面，为了使得 feflow 能够适用多种类型的项目。我们开发了多种类型的业务脚手架，如：活动模板、App H5模板、RN模板和业务组件模板。

### 执行过程
当用户在控制台里面输入某个命令。首先会通过CLI 的参数解析器，将这个命令解析成一个object对象，然后传递给CLI 的内核。所有的命令都是通过内核上下文提供的 register 函数 进行注册的，一方面内核自身会读取内置插件 注册的基础命令，另一方面，内核会读取本地已经安装的外部插件注册的命令。如果找到用户输入的命令则开始执行命令对应的回调函数。


### 基础命令设计
```
# 初始化项目
$ feflow init

# 本地开发
$ feflow dev

# 代码质量检查
$ feflow lint

# 打包构建
$ feflow build

# 代码发布
$ feflow publish

# 安装插件、脚手架等
$ feflow install package

# 配置本地客户端，如: npm 的源和 proxy
$ feflow config <key> <value>
```

前面提到，CLI 的命令包含两部分，分别是内置在内核里的基础命令和外部插件提供的命令。那么外部插件要如何设计呢？

### 插件机制设计
#### 插件实现原理
这里有一个非常巧妙的设计，通过使用node提供的module和vm模块，可以通注入feflow全局变量来访问到cli的实例。从而能够访问cli上的各种属性，比如config, log和一些helper等。
```
 loadPlugin(path, callback) {
    const self = this;

    return fs.readFile(path).then((script) => {

      const module = new Module(path);
      module.filename = path;
      module.paths = Module._nodeModulePaths(path);

      function require(path) {
          return module.require(path);
      }

      require.resolve = function(request) {
          return Module._resolveFilename(request, module);
      };

      require.main = process.mainModule;
      require.extensions = Module._extensions;
      require.cache = Module._cache;

      // Inject feflow variable
      script = '(function(exports, require, module, __filename, __dirname, feflow){' +
          script + '});';

      const fn = vm.runInThisContext(script, path);

      return fn(module.exports, require, module, path, pathFn.dirname(path), self);
      }).asCallback(callback);
  }
```

#### 命令注册：

命令需要以feflow.cmd.register进行注册，比如：

```
feflow.cmd.register('deps', 'Config ivweb dependencies', function(args) {
    console.log(args);
    // Plugin logic here.
});
```

说明：
* register有3个参数，第一个是子命令名称，第二个是命令描述说明信息，第三个是对应的子命令执行逻辑函数。
* feflow会将命令行参数args解析成Object对象，传递给插件处理函数

#### 配置
可以通过feflow.version获取当前feflow的版本，feflow.baseDir 获取feflow跟目录（在用户目录下的.feflow），通过feflow.pluginDir 获取插件目录

#### 日志
通过feflow.log来进行相关命令行日志输出

```
const log = feflow.log;
log.info()    // 提示日志，控制台中显示绿色
log.debug()   // 调试日志,  命令行增加--debug可以开启，控制台中显示灰色
log.warn()    // 警告日志，控制台中显示黄色背景
log.error()   // 错误日志，控制台中显示红色
log.fatal()   // 致命错误日志，，控制台中显示红色
```
