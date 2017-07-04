# webpack打包优化

标签（空格分隔）： 前端 webpack vue

---
[TOC]
## 理顺思路

### 一句话概述：
优点：把依赖库提前打包出来并保留引用，打包功能模块时不再打包依赖库，只引用依赖，从而减少打包时间。思想来源：[知乎专栏-彻底解决Webpack打包性能问题](https://zhuanlan.zhihu.com/p/21748318)
缺点：本方式产生了依赖包的js，要在html之初引入，和懒加载的观念有所冲突。懒加载是在需要的时候才请求资源，而本方式提前把某些可能不需要的资源加载了进来。

### 优化目标：

#### 目标：
是为了减少发布到用于测试的“伪生产环境”时打包业务模块过程中npm run testbuild的时间【频繁执行】
并不是减少本地开发环境npm run dev的时间
也不是减少发布到“生产环境”时打包业务模块过程中npm run build的时间【不频繁，上线才执行】

#### 注：关于npm run testbuild
由于我们开发的项目在在微信浏览器内打开，并且使用了微信的jssdk，所以需要发布到真实的微信环境进行测试（这个动作我们要经常进行）。但是这只是我们开发人员才知晓的微信公众号（和我们项目对外服务的公众号是等价的，只不过用于真正发布前测试）
所以对于有些朋友，不频繁执行打包操作的话，可能并不需要进行本优化，为了懒加载，甚至不应该进行本优化。

### 另容易误解的地方有：
本地开发目前不需要优化，所以本优化和webpack.config.js无关
生产环境需要懒加载，我们不希望该环境一开始就加载所有的依赖包，所以本次优化和npm run build使用的webpack.production.js无关
和需要经常执行的打包发布到测试生产环境的操作npm run testbuild有关，所以我们新建了webpack.dll.config.js和webpack.dll.production.js。所做的优化基本围绕这两个文件进行。

### 涉及的文件：
新建webpack.dll.config.js：这是打包依赖包的配置文件
新建webpack.dll.production.js：这是npm run testbuild时执行的配置文件，使用webpack.DllReferencePlugin引用了提前打包好的依赖包
修改package.json：新增dllconfig命令、testbuild命令
修改~pathto/index.html：通过script标签引入打包好的依赖包文件。生产环境不使用本优化，不引入依赖包的js，测试生产环境使用的本优化，引入依赖包的js。
修改.gitignore：忽略webpack.dll.config.js打包依赖包过程中产生的manifest.json文件

### 优化结果
优化效果：npm run build的时间为110秒左右，npm run testbuild的为50秒左右。
副作用：需要在测试生产环境的套模板页面另引入一个多余的js：即之前打包的依赖包。
注意：可以考虑把依赖包放在静态资源服务器上，每回修改webpack.dll.config.js时才执行一次npm run dllconfig并上传。

## 操作步骤

### 1.创建依赖包的webpack配置文件 webpack.dll.config.js（名字可以改）
```
/**
 * @file 使用DllPlugin把常用依赖单独打包，提高webpack打包效率
 * @author Liu Junyang
 * 在应用到这些依赖包的后端套模板的html需要使用script标签引用output的filename的js文件
 */
var path = require('path')
const webpack = require('webpack');

const vendors = [
    'vue',
    'vue-router',
    'vue-resource',
    'vue-touch',
    'mint-ui',
    'swiper_3.4.0_hack',
    'hack_swiper'
];

module.exports = {
    output: {
        path: 'static',
        filename: '[name].js',
        library: '[name]',
    },
    entry: {
        "vendors": vendors,
    },
    plugins: [
        new webpack.DllPlugin({
            path: 'manifest.json',
            name: '[name]',
            context: __dirname,
        }),
    ],
    resolve: {
        alias: {
            'vue': path.resolve(__dirname, './node_modules/vue/dist/vue.min.js'),
            'style': path.resolve(__dirname, './src/style'),
            'module': path.resolve(__dirname, './src/module'),
            'libs': path.resolve(__dirname, './src/libs'),
            'layout': path.resolve(__dirname, './src/layout'),
            'images': path.resolve(__dirname, './src/images'),
            'pages': path.resolve(__dirname, './src/pages')
        },
        fallback: path.resolve(__dirname, './src/util')
    },
};
```

### 2.命令行执行`webpack --config webpack.dll.config.js`打包依赖包
### 3.在业务打包的webpack配置文件中通过`DllReferencePlugin`在打包的时候引用依赖包
``` javascript
plugins: [
        new webpack.DefinePlugin({
            'process.env': {
                NODE_ENV: '"production"'
            }
        }),
        new webpack.optimize.UglifyJsPlugin({
            output: {
                comments: false  // remove all comments
            },
            compress: {
                warnings: false,
                drop_debugger: true,
                drop_console: true
            }
        }),
        new webpack.ProvidePlugin({
            $: 'zepto/dist/zepto.min.js',
            jQuery: 'zepto/dist/zepto.min.js'
        }),
        new webpack.DllReferencePlugin({
            context: __dirname,
            manifest: require('./manifest.json'),
        })
    ]
```

### 4.在html文件中通过script标签引入依赖包
``` html
{% if STATIC_JS_DEBUG %}
    <script src="/static/vue/vendors.js"></script>
{% endif %}
```

### 5.关于package.json中的命令
``` JavaScript
{
    "scripts": {
        "dev": "webpack-dev-server --inline --hot --quiet --display-error-details",
        "dllconfig": "rm -rf static/* && webpack --config webpack.dll.config.js",
        "testbuild": "npm run dllconfig && webpack --progress --hide-modules --config webpack.dll.production.js",
        "build": "rm -rf static/*  && webpack --progress --hide-modules --config webpack.production.js",
        "ontestline": "npm run testbuild && npm run html && ./bin/online",
        "online": "npm run build && npm run html && ./bin/online"
  }
}
```
