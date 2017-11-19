Webpack从2015年9月第一个版本横空初始至今已逾2载。它的出现，颠覆了一大批主流构建如Ant、Grunt和Gulp等等。腾讯NOW直播IVWEB团队之前一直采用Fis构建，本篇文章主要介绍从Fis迁移到webpack遇到的问题和背后的黑科技，内容包括inline-resource、多页面构建、资源压缩、文件hash、文件目录规则和打包优化的实践。

### 区分构建的开发or生产环境？
``` sh
  "scripts": {
    "dev": "cross-env NODE_ENV=dev nodemon --watch webpack.config.js --exec \"webpack-dev-server --config webpack.config.js --env development\" --progress --colors",
    "build": "webpack --config webpack.config.js --env production --progress --colors",
    ...
  },
```
通过在package.json中注入环境变量的方式，注入NODE_ENV=dev代表开发环境，默认为生产环境。这里使用cross-env的原因是：windows下 在package.json中直接使用 NODE_ENV=dev 不生效，需写成 set NODE_ENV=dev，cross-env的写法兼容各个操作系统。


### 资源内联 (inline-resource)
inline-resource的好处是可以减少css,js等的请求数，同时html加载的时候即可同时加载了这些内联的css、js等静态资源，可以有效的减少白屏或者页面闪动的问题。

这里的内联分为2种，一种是静态的html片段,css,js等，这些资源一开始就存在项目的某个目录下；另一种是构建过程中动态生成的css,js文件。

对于html的内联，写法如下：
``` sh
  ${require('raw-loader!../src/assets/inline/meta.html')}
```
对于js的内联，需要增加babel-loader将ES6的语法进行转换，避免浏览器直接解析导致报错。写法如下：
``` sh
<script>${require('raw-loader!babel-loader!../src/node_modules/@tencent/report-whitelist/lib/index.js')}</script>
```

上面讲述了如何内联静态的资源文件，那么如何内联构建过程中动态生成的资源文件呢？
这里需要借助html-webpack-inline-source-plugin来增强html-webpack-plugin的功能。比如：将构建过程中生成的css文件inline到html模板里面去。

``` javascript
const HtmlWebpackPlugin = require('html-webpack-plugin');
const HtmlWebpackInlineSourcePlugin = require('html-webpack-inline-source-plugin');

    new HtmlWebpackPlugin({
        inlineSource: isDev ? undefined : '\\.css$',
        template: __dirname + '/template/index.tmpl.html',
        filename: 'activity.html',
        inject: true,
    }),
    new HtmlWebpackInlineSourcePlugin(),
    ...
```
上面这段代码，html-webpack-plugin本身并不具备inlineSource的属性。引入了html-webpack-inline-source-plugin之后，就可以通过inlineSource属性来匹配哪些文件需要动态的内联进html模板文件中了。

### 多页面构建
多页面构建，或者称为通配(wildcards)构建。即需要构建的页面数量是不确定的，可能A业务有3张页面，B业务有5张页面。因此，我们不能把entry写死了：

```javascript
        entry: {
          activity: './src/pages/activity/init.js',            // 深海寻宝活动首页
          my-reward: './src/pages/my-reward/init.js',          // 我的奖励
          exchange: './src/pages/exchange/init.js'             // 线下兑换奖品
        },
```
为什么上面的写法不可取呢？很明显：上面的写法把entry写死了，这并不通用。后面如果产品需求发生改变，需要新增一张页面，就需要手动修改构建脚本。我们需要的entry是：'./src/pages/\*\*/init.js'，它能够像一些linux的命令，具备匹配某个规则的所有结果的能力。这里的思路是借助**glob**，达到动态entry的目的。

在webpack构建中，一个页面需要一个与之对应的HtmlWebpackPlugin实例，N个页面需要N个与之对应的HtmlWebpackPlugin。此处需要动态的设置HtmlWebpackPlugin的实例个数。

```javascript
    const newEntry = {};

    Object.keys(config.entry).map((index) => {
        const entry = config.entry[index];
        const match = entry.match(/\/pages\/(.*)\/init.js/);
        const pageName = match && match[1];

        newEntry[pageName] = entry;

        config.plugins.push(
            new HtmlWebpackPlugin({
                inlineSource: isDev ? undefined: '\\.css$',
                template: __dirname + '/template/index.tmpl.html',
                filename: `${pageName}.html`,
                chunks: [pageName],
                inject: true
            })
        );
    });
    config.entry = newEntry;
```












