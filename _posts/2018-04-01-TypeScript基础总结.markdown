> TypeScript(以下简称TS) 是由微软开发的编程语言，是JavaScript的超集，于2013年10月发布第一个正式版0.9。最先代码托管在Codeplex，2014年7月移到了Github。它的代码风格和C#很像，这是因为TS是由C#首席架构师设计并主导开发的。

### 开发环境
编辑器首选MS自家开发的VS Code (推荐)。当然，Webstorm在2016年2月推出的版本内置了TS编译器，atom 需要安装 atom-typescript包，sublime需要安装Typescript-sublime-plugin。

TS程序以.ts扩展名结尾。运行TS程序很简单，只需要安装编译器TS compile即可，需要通过npm 的方式安装它。

``` sh
npm install typescript -g
```
安装完后，在全局会有tsc命令，需要通过它编译TS程序
``` sh
tsc hello.ts
```

## 类型系统
谈到TS，大家印象最深刻的还是TS的静态强类型了。虽然JS异常灵活，但是在大型复杂的web工程里面并不合适。除了TS，其它公司比如FB推出了Flow，Google推出了Clojure，这些都是为了给JS增加类型。

### 类型注解
概念：注解是一种轻量级的为函数或变量添加约束的方式。
语法：变量或者函数后接 :TypeAnnotation

比如：
```sh
let a: number = 123;

function add(a: number, b: number): number {
    return a + b;
}
```

### 原始类型
TS里的原始类型包括string, number和boolean，这些也是JS的原始类型。在TS里，你可以显示声明变量为某一种类型。

```
let num: number;
let str: string;
let bool: boolean;

num = 123;
num = 123.456;
num = '123';     // 错误

str = '123';
str = 123;       // 错误

bool = true;
bool = false;
bool = 'false';  // 错误
```
