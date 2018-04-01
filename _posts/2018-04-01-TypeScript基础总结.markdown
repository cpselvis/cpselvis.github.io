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
```typescript
let a: number = 123;

function add(a: number, b: number): number {
    return a + b;
}
```

### 原始类型
TS里的原始类型包括string, number和boolean，这些也是JS的原始类型。在TS里，你可以显示声明变量为某一种类型。

```typescript
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


### 数组
TS里手动指明一个数组类型很简单，只需要在普通类型注解后面加上[]符号。比如声明一个布尔数组为 :boolean[] 
``` typescript
let boolArray: boolean[];

boolArray = [true, false];
console.log(boolArray[0]);     // true
console.log(boolArray.length); // 2
boolArray[1] = true;
boolArray = [false, false];

boolArray[0] = 'false';        // 错误
boolArray = 'false';           // 错误
boolArray = [true, 'false'];   // 错误
```

### 枚举
枚举在TS里面是原生支持的，使用枚举我们可以定义一些带名字的常量，它的好处是可以让语意更清晰。定义一个枚举值，需要使用 **enum**。

TS 仅支持基于数字的和字符串的枚举。如果是数字枚举，枚举值默认是从0开始，依次自增的。你也可以手动的设置第一个枚举值，比如为1。

```typescript
enum Color {
    RED = 1,
    GREEN,
    YELLOW
}
console.log(Color.GREEN);   // 2

enum Direction {
    UP = 'UP',
    DOWN = 'DOWN',
    LEFT = 'LEFT',
    RIGHT = 'RIGHT'
}
console.log(Direction.UP);   // 'UP'
```
虽然TS支持**异构枚举**（即数字和字符串混搭的枚举），但是并不建议使用这种方式。

### 特殊类型
- any: 任何元素都可以赋值给它，它也可以赋值给任何元素。相当于关掉类型检查，适用 js 代码迁移到 ts。
- null: 可以赋值给任何元素
- undefined: 可以赋值给任何元素
- void: 表示函数没有返回类型

### 接口
和其它语言(比如C++, java)不同的是，TS 里接口可以描述变量、函数类型和类类型，其它语言只能描述类类型。另外，TS中的接口描述变量时可以使用？定义某个变量为可选变量。比如对某个对象进行约束时，如果对象的某个属性设置成了可选，则传入的对象可以不包含这个属性。

```typescript
interface LabelledValue {
    size?: Number,
    label: String
}

function printLabel(labelObject: LabelledValue) {
    return labelObject.label;
}

// 由于size设置成了可选变量，则传入的对象可以不包含size属性
console.log(printLabel({
    label: 'size 1 object'
}));
```

### 接口-描述函数类型
接口出了可以用来约束JS对象之外，也可以用来约束函数类型。比如：
```typescript
interface SearchFunc {
    (source: string, subString: string): boolean;
}

let mySearch: SearchFunc;
mySearch = function(src: string, sub: string): boolean {
  let result = src.search(sub);
  return result > -1;
}
```
说明：对于函数类型的类型检查来说，函数的参数名不需要与接口里定义的名字相匹配。

### 接口 - 描述类类型
与C#或Java里接口的基本作用一样，TypeScript也能够用它来明确的强制一个类去符合某种契约。不同的是，C#或Java里面的接口描述类类型时，只能定义函数，TS里则还可以定义属性。如果某个类继承了这个接口，那么这个类必须包含接口里定义的属性和方法。

``` typescript
interface ClockInterface {
    currentTime: Date;
    setTime(d: Date): any;
}

class Clock implements ClockInterface {
    currentTime = new Date();
    setTime(d: Date) {
        this.currentTime = d;
    }
    constructor(h: number, m: number) { }
}
```

### 泛型
 设计的出发点：可重用性。组件不仅能够支持当前的数据类型，同时也能支持未来的数据类型。
 
 比如我们需要一个函数，同时支持入参是一个数字或字符串，返回值的类型和参数类型相同：
 
 ```typescript
function identity(arg: number): number {
    return arg;
}

function identity(arg: string): string {
    return arg;
}
 ```
 这个时候就需要引入泛型去解决这个问题了。在TS里，泛型的类型变量定义为T，需要使用<>包裹起来，这个时候函数会捕获入参的类型，然后在后面就可以使用这个类型了。如下：。
 ```typescript
function identity<T>(arg: T): T {
    return arg;
}

let output1 = identity<string>("myString");   // myString
let output2 = identity("myString");           // myString
 ```
由于TS的类型推断，调用时不需要显示的指明类型，推荐使用上面代码中的第二种方式。

## 高级特性

### 类
- 访问修饰符：public可以被自由访问，private在类外无法访问。protected可以被派生类（子类）访问。
- static关键字：可以用来修饰类的属性和方法，静态属性和静态方法存在类上而不是实例上，可以通过 ”类名.” 的方式来访问。
- readonly关键字：属性初始化之后不可修改。
  - 备注：readonly和const区别：const用来修饰变量，readonly用来修饰属性。


### 抽象类
定义：通过 abstract 来修饰的类称为抽象类。

特点：
- 抽象类不能直接实例化，即不能通过 new X()的方式调用。
- 抽象类必须包含一些抽象方法，抽象方法也用 abstract修饰
- 抽象类中的抽象方法不包含具体实现，但是必须在派生类中实现。

值得一提的是：抽象类和接口在描述类类型时，虽然比较类似。但是抽象类只能继承一个抽象类或者一个接口，而接口可以多重继承。

### 装饰器
Decorator是一个函数，用来修饰类、属性、方法和参数。使用 @expression 语。

Decorator 的改变是在编译期改变，而不是运行期。装饰器包括多个规范，TC39在stage-0 和 stage-2分别定了修饰属性、方法的规范和修饰类的规范。

### 类装饰器
 @func 修饰 类A 等价于 A = func(A)，相当于把旧class转换成了新的class。可以理解为一个加工函数，它接受一个类，加工后返回另一个类。
 
``` typescript
@testable
class MyTestableClass {
  // ...
}
 
function testable(target) {
  target.isTestable = true;
  return target;
}

function getMetaData(target) {
    return target.isTestable;
}

console.log(getMetaData(MyTestableClass))  // true
 ```
 
### 方法装饰器
装饰器除了修饰类之外，也可以用来修饰方法： 
```typescript
class A {
    @func()
    static method() {

    }
}

function func() {
    return function (target: any, propertyKey: string, descriptor: PropertyDescriptor) {
        console.log('func 被调用了');
    };
}

A.method();
```
 
@func 修饰 类A 上的 方法 method 是借助 Object 基类上的 defineProperty来实现的。原理如下：
``` typescript
const oldDescriptor = Object.getOwnPropertyDescriptor(A.property, 'method');
const newDescriptor = func(A.property, 'method', 'oldDescriptor');
Object.defineProperty(A.property, 'method', newDescriptor);
```
 
## 工程应用

### tsconfig.json
tsconfig.json是TS项目的配置文件，可以描述整个工程的编译参数。

初始化一个 tsconfig.json 命令：
``` sh
$ tsc --init
```

tsconfig.json里面的常用字段解释如下：
- target: 编译的目标平台，es3, es5, es2015, es2016, es2017
- module: 代码模块加载器 (commonjs, and 或者es2015)
- sourceMap: 是否生成映射文件(true 或 false)
- outDir: 指定输出目录
- exclude: 不包含的编译目录

### 与webpack结合
要想在webpack里面使用TS，只需两步。第一，安装ts-loader；第二，设置webpack配置中的resolve.extension 包含 .ts 和 .tsx。

下面是一个最简单的配置：
``` javascript
// webpack.config.js

module.exports = {
    entry: "./src/index.ts",
    output: {
        filename: "./dist/bundle.js",
    },
    resolve: {
      extensions: ['.ts', '.js', '.json']
    },
    devtool: "source-map",
    module: {
        rules: [{
            test: /\.tsx?$/,
            loader: 'ts-loader'
        }]
    },
};
```
 
## TS的优势
总体而言，TS相比ES6 + babel，包含以下优势:

- 类型检查
- TS对ECMAScript新特性支持的更全面，开箱即用。babel中支持新的ES特性需要安装各种插件，比较繁琐。
- TS 写出来的代码可读性更好
- VS里的jsdoc，调用栈更加清晰。
 
