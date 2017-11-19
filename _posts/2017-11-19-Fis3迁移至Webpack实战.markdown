Webpack从2015年9月第一个版本横空初始至今已逾2载。它的出现，颠覆了一大批主流构建如Ant、Grunt和Gulp等等。腾讯NOW直播[IVWEB团队](https://ivweb.io/)之前一直采用Fis构建，本篇文章主要介绍从Fis迁移到webpack遇到的问题和背后的黑科技，内容包括inline-resource、多页面构建、资源压缩、文件hash、文件目录规则等等。

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

```javascript
entry: glob.sync('./src/pages/**/init.js'),
```

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

### html、css和js压缩
对于html文件里面的内容压缩可以通过给html-webpack-plugin设置minify参数，html-webpack-plugin的压缩配置其实是直接集成了 html-minifier，因此minify的参数设置可以直接参考html-minifier的文档。
``` javascript
config.plugins.push(
    new HtmlWebpackPlugin({
        inlineSource: isDev ? undefined: '\\.css$',
        template: __dirname + '/template/index.tmpl.html',
        filename: `${pageName}.html`,
        chunks: [pageName],
        inject: true,
        minify: {
            minifyJS: true,           // 仅压缩内联在html里面的js
            minifyCSS: true,          // 仅压缩内联在html里面的css
            html5: true,              // 以html5的文档格式解析html的模板文件
            removeComments: false,    // 不删除Html文件里面的注释
            collapseWhitespace: true, // 删除空格
            preserveLineBreaks: false // 删除换行
        }
    })
);
```

设置了上面的minify参数后，看到生成的html文件的内容全部在1行上，需要注意的是：minifyJS和minifyCSS只会压缩内联在这个html文件的css和js内容，对于单独的css文件和js文件并不会压缩。 那么打包出来的css和js文件如何压缩呢？

对于css文件压缩，直接开启css-loader的压缩参数参数minimize为true即可：
```
{
    test: /\.scss$/,
    use: ExtractTextPlugin.extract({
        fallback: 'style-loader',
        use: [
            {
                loader: "css-loader",
                options: {               // 设置css-loader的minimize参数为true
                  minimize: true
                }
            },
            {
                loader: "sass-loader"
            }
        ]
    })
},
```
css-loader开启压缩可能会报错 Module build failed: BrowserlistError: unkonwn version 61 and _chr，解决办法：
``` sh
$ npm i caniuse-db —save    #更新caniuse-db到最新版本
```

对于js文件的压缩，可以通过引入 webpack 内置的 UglifyJsPlugin：
```javascript
const webpack = require('webpack');
plugins: [
    ...
    new webpack.optimize.UglifyJsPlugin(),
    ...
],
```

### 文件Hash
每次功能发布上线，都需要重新构建一次源代码，生成一个新的文件版本列表。此处文件Hash的方式就是一种版本管理的方式，发布时替换有变化的版本的文件，达到增量更新的效果。此处Hash策略是：根据文件内容进行hash，取8位。

JS文件:
``` javascript
output: {
    filename: '[name]_[chunkhash:8].js',     // 进行js脚本hash
    path: path.resolve(__dirname, 'public/'),
    publicPath: isDev ? '/' : cdnUrl + '/',
},
```

Css文件:
``` javascript
plugins: [
    new CleanWebpackPlugin(['./public']),
    new ExtractTextPlugin('[name]_[contenthash:8].css'),  // css文件hash
    new webpack.optimize.UglifyJsPlugin(),
    ...
]
```
Img文件:
``` javascript
rules: [
    {
        test: /\.(png|svg|jpg|gif)$/,
        use: {
            loader: 'file-loader',
            options: {
                name: '[name]_[hash:8].[ext]',    // img文件hash
            }
        }
    },
    ...
]    
```

### 多终端适配
开发过程中，不同分辨率的浏览器适配是个让前端开发者头疼的问题。手淘的rem方案完美解决了这个问题，它的核心思想是页面加载时动态设置body的font-size值和rem和px转换的单位。

为了不改变编程习惯，开发过程中仍然使用px单位来作为基础布局长度单位，以750px宽度的视觉稿作为基准，设置rem和px的转换单位为1rem=75px。那么px2rem如何集成进webpack中呢？首先增加css的解析[px2rem-loader](https://www.npmjs.com/package/px2rem-loader)，然后在html头部引入[lib-flexible](https://github.com/amfe/lib-flexible)文件。
```javascript
{
    test: /\.scss$/,
    use: ExtractTextPlugin.extract({
        fallback: 'style-loader',
        use: [
            {
                loader: "css-loader"
            },
            {
                loader: "px2rem-loader", // 增加px2rem-loader，并且设置rem单位为75px
                options: {
                    remUnit: 75
                }
            },
            {
                loader: "sass-loader"
            }
        ]
    })
},
```

### 其它feature
- 开发环境支持WDS: webpack3.x版本自带webpack-dev-server，开发环境中开启WDS。这样依赖的文件发生变化后，会自动增量构建并且刷新浏览器
- 支持HMR: webpack.config.js文件内容变化后，会触发热更新逻辑，此处通过nodemon来守护webpack的构建进程，eg:
  ``` sh
    "scripts": {
    "dev": "cross-env NODE_ENV=dev nodemon --watch webpack.config.js --exec \"webpack-dev-server --config webpack.config.js --env development\" --progress --colors"
    ...
  },
  ```

由于篇幅原因，关于webpack的打包优化将会用另外一篇文章介绍，敬请期待～
