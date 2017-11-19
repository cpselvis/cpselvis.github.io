Webpack从2015年9月第一个版本横空初始至今已逾2载。它的出现，颠覆了一大批主流构建如Ant、Grunt和Gulp等等。腾讯NOW直播IVWEB团队之前一直采用Fis构建，本篇文章主要介绍从Fis迁移到webpack遇到的问题和背后的黑科技。

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



