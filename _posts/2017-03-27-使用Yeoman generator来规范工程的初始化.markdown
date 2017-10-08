---
layout: post
title: 使用Yeoman generator来规范工程的初始化
date: 2017-03-27 23:25:24.000000000 +09:00
---


![](http://7tszky.com1.z0.glb.clouddn.com/FtM6zxr8gWVy1LQKD1e3lxkBAumx)

### 前言
> 随着开发团队不断发展壮大，在人员增加的同时也带来了协作成本的增加；业务项目越来越多，类型也各不相同。常见的类型有基础组件、业务组件、基于React的业务项目、基于Vue的业务项目等等。如果想要对每个项目进行一些规范上的约束比如Git提交规范、Javascript规范简直难于登天。所有的这些，只是因为还欠缺一个好用的工程化工具，在项目创建的初期自动的将这些目录结构和文件生成、并且集成工程常见的规范来进行约束。

本文分为两部分，首先会谈谈目前团队的痛点以及基于yeoman generator的设计思路；然后会详细介绍如何实现定制的generator，过程中遇到的问题和解决办法。

### 痛点一：工程创建不智能
* 代码目录文件手工拷贝
* 不同场景的工程对目录结构的要求不尽相同

### 痛点二：规范约束难以统一集成
* 难以在新的工程项目中集成新的规范，需要手动加hook
* 缺少增量机制对旧项目集成

### 基于Yeoman generator的设计思路
我们需要给每个工程类型的项目创建一个generator。按照目前前端技术栈的发展情况来看，一个团队一般会有3~5个generator。把这些generator看成一个个的插件，通过工具上层的CLI命令来暴露给开发者使用。

在generator之下，需要开发一系列服务和集成规范。包括和Git仓库打通，也就是通过脚手架初始化目录时，先对开发者鉴权。之后根据开发者输入的项目名称在远程Git仓库里面创建仓库并且授予开发者权限。后期功能完善之后，可以做一些锦上添花的工作，比如进行数据统计，分析各个业务仓库使用的generator版本信息，是否集成了最新的feature等等。

整体系统架构如下：


![](http://7tszky.com1.z0.glb.clouddn.com/FiwG7VvpYL3Tleaii9Q9dowGNJXv)

下面我准备开发一个适用于Now直播活动类搭建的脚手架了，名字是generator-now-activity

### 自定义generator的目录结构
```shell
├───package.json
└───generators/
   ├───app/
    |  ├───templates/
    |   |  ├─── src/
    |   |   |─── _cilintrc.js
    |   |   |─── _eslintrc.js
    |   |   |─── _fis-conf.js
    |   |   |─── _package.json
    |   |   |─── _project.js
    |   |   |─── _README.md
    |   |   |─── editorconfig
    |   |   |─── gitignore
    |   |  └─── vcmrc
   │ └───index.js
   └───utils.js   
```

### 扩展generator
在generator的外层index.js文件里，通过继承yeoman-generator来扩展我们自己的generator，然后模块暴露给外部。
```javascript
const Generator = require('yeoman-generator');

module.exports = class extends Generator {

}

```

### Yeoman的运行周期
一个 Yeoman Generator 被创建后（构造函数必然是最先被调用的），会依次调用它原型上的方法，且每一个方法中的 this 都被绑定为 Generator 实例本身，调用的顺序如下：

* initializing - 初始化一些状态之类的，通常是和用户输入的 options 或者 arguments 打交道，这个后面说。
* prompting - 和用户交互的时候（命令行问答之类的）调用。
* configuring - 保存配置文件（如 .babelrc 等）。
* default - 其他方法都会在这里按顺序统一调用。
* writing - 在这里写一些模板文件。
* conflicts - 处理文件冲突，比如当前目录下已经有了同名文件。
* install - 安装依赖
* end - 结束部分

### 与用户交互
Yeoman提供了API来让generator和用户进行交互，直接通过this.prompts函数，它的内部实现是使用了Inquire.js。
```javascript
  /**
   * 提示用户输入配置项
   * @returns {Promise.<TResult>}
   */
  prompting() {

    return this.prompt([{
      type: 'input',
      name: 'projectName',
      message: '请输入活动的名称 (now-activity):',
      default: 'now-activity-default'
    }]).then((answers) => {
      this.log('活动名称', answers.projectName);
      this.props = answers;
    });

  }
```

### 模板拷贝策略
对于工程src目录部分直接通过深度优先算法拷贝写入。对于工程的规范类、配置的文件需要单独写入，这一类可能需要接受用户的输入，同时需要集中进行维护，因此需要和src的拷贝方式进行区分。

src深度优先拷贝代码如下：
```
const fs = require('fs');
const path = require('path');

function read(root, filter, files, prefix) {
  prefix = prefix || '';
  files = files || [];
  filter = filter || noDotFiles;

  const dir = path.join(root, prefix);
  if (!fs.existsSync(dir)) return files;
  if (fs.statSync(dir).isDirectory())
    fs.readdirSync(dir)
      .filter(filter)
      .forEach(function (name) {
        read(root, filter, files, path.join(prefix, name));
      });
  else
    files.push(prefix);

  return files
}

function noDotFiles(x) {
  return x[0] !== '.';
}

module.exports = {
  read
};
```

在外层通过Yeoman提供的API this.fs.copy()方法来进行文件拷贝
``` javascript
    /**
     * 源代码模板
     */
    const sourceCode = () => {
      const sourceDir = path.join(this.templatePath(), './src/');
      const filePaths = utils.read(sourceDir);
      _.each(filePaths, (filePath) => {
        this.fs.copy(
          this.templatePath('./src/' + filePath),
          this.destinationPath('./src/' + filePath)
        );
      });
    };
```

开发完generator之后，就可以通过yo now-activity来进行使用了。

### generator和其它工具如CLI集成
前面提到的yo now-activity的方式使用可能存在一些问题，因为这种方式要求代码必须上传到github上。对于公司内部的工具，不走正常的开源流程显然是不被允许的。那么，有没有什么方法，不添加generator到Yeoman的generator列表里就能够使用呢？

幸运的是，Yeoman提供了yeoman-environment来帮助我们在其它工具中集成编写好的generator，yo其实也只是yeoman-environment暴露到上层的一个命令而已。

``` javascript
const yeoman = require('yeoman-environment');
const yeomanEnv = yeoman.createEnv();

/**
 * Lookup方法会在本地查找已经安装过的generator
 */
yeomanEnv.lookup(() => {
    yeomanEnv.run('@tencent/now-activity', {'skip-install': true}, err => {
        console.log('done');
    });
});
```

### 一些细节
* 为了方便本地调试看效果，在generator根目录下运行 tnpm  link
* 使用Yeoman提供的API this.log来打印信息，而不要使用console.log
* 如果是内部工具，运行的时候命令为：yo @tencent/now-activity


### 最后
安装示例（限内部）
```shell
$ tnpm install -g yo generator-generator @tencent/feflow-cli
$ feflow init
```

### 参考资料
* [Yeoman Generator API](http://yeoman.io/generator/Base.html)
* [WRITING YOUR OWN YEOMAN GENERATOR](http://yeoman.io/authoring/index.html)
* [Spring Boot + Angular projects](https://github.com/jhipster/generator-jhipster)
