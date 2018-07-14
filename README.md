# vue-multiple-page

> A Vue multi page project


# 解锁 Vue多页面应用

大部分使用Vue是构建单页面应用，但有时候我们也需要多页面应用，比如手机端的H5页面，可能这块需要一个H5页，另一块需要一个H5活动页，彼此相互独立，根本就没有什么关联，这时候还使用单页面应用增加了页面加载的速度，而且打包了一大堆和本页面无关的代码，增加了页面响应时间。


**准备工作：**
使用vue-cli构建工具，添加项目。
最近vue-cli发布了最新版本v3.0.0，如果使用最新版本的脚手架，就不会暴露出webpack的一些配置文件，没办法自定义了，所以本次配置多页面构建还是使用的Vue CLI 2
```
npm install -g vue-cli

vue init webpack vue-mutile-page
```

使用此模板创建的vue项目，集成的是webpack3，现在webpack已经升级到4了，暂时先按照模板来使用webpack3吧，先把架子搭好，后期想升级到最新也是可以的。

#### 单页面应用打包之后dist目录
直接使用模板创建的项目是单页面应用的，配置文件都是写好的，所以我们可以直接执行命令打包
```
npm run dev
```
打包完成之后会生成dist目录，只生成一个入口文件index.html，然后static中有打包好的js和css文件.
```
├── dist
|    ├── index.html
|    └── static
|       ├── css
|       └── js
```


## 多页面应用
上面是常见的单页面应用打包之后的目录，那多页面打包之后应该是什么样的呢？

多页面应用就意味着有多个页面，单页面打包之后只生成一个html文件，这个文件就是项目的入口文件，多页面也就是说每个页面都应该有一个自己的html文件，页面和页面之间是相互独立的，那在打包之后dist目录应该生成每个页面的html文件。比如我有page1和page2，dist目录中应该生成下面的结构，各自的页面名称对应各自的html文件。那要怎么做到呢，请看下文分解。

```
├── dist
|    ├── page1.html
|    ├── page1.html
|    └── static
|       ├── css
|       │   
|       └── js
|       
```

### 改写项目目录
想要搭建多页面的应用，首先要改写刚刚生成的项目目录结构。

下面是我修改之后的目录结构
```
.
├── .babelrc
├── .editorconfig
├── .eslintignore
├── .eslintrc.js
├── .gitignore
├── .postcssrc.js
├── README.md
├── build
│   ├── build.js
│   ├── check-versions.js
│   ├── utils.js
│   ├── vue-loader.conf.js
│   ├── webpack.base.conf.js
│   ├── webpack.dev.conf.js
│   └── webpack.prod.conf.js
├── config
│   ├── dev.env.js
│   ├── index.js
│   ├── prod.env.js
│   └── test.env.js
├── package.json
├── src
│   ├── assets
│   │   └── logo.png
│   ├── components
│   │   └── HelloWorld.vue
│   └── pages
│       ├── page1
│       │   ├── App.vue
│       │   ├── index.ejs
│       │   └── index.js
│       └── page2
│           ├── App.vue
│           ├── index.ejs
│           └── index.js
└── static
```
修改的点
1. 在src文件下创建pages目录，这是我们创建多页面放置的地方
2. pages目录下是根据页面名称命名的文件夹，文件夹下包含三个文件，后缀为：`.vue .ejs .js`这三个文件，后缀为.vue 和 .js 的文件分别对应项目创建时的App.vue 和main.js文件，现在都放到页面文件中，.ejs文件对应的是根目录下的index.html文件，也就是项目的入口文件。
3. 把src下的App.vue 和main.js删除，根目录下的index.html也删除，因为这些都已经移到页面中了。

项目目录整理好之后，那接下来我们就可以写配置文件了，配置文件的修改都在build目录下。更改的目录主要有下面这几个
```
├── build
│   ├── utils.js
│   ├── webpack.base.conf.js
│   ├── webpack.dev.conf.js
│   └── webpack.prod.conf.js
```

### 修改utils.js文件
在utils文件中增加两个函数，一个是获取多页面路径作为entry文件的入口，另一个是生成页面多页面plugin的配置，废话不多说，上代码。

首先需要安装一个node模块[glob](https://www.npmjs.com/package/glob)，glob可以读取相应后缀名的文件，便于检索出需要的文件。
```
npm install --save-dev glob
```

在utils.js中添加如下代码
```
const glob = require('glob')
const HtmlWebpackPlugin = require('html-webpack-plugin')
const merge = require('webpack-merge')

const PAGE_PATH = path.resolve(__dirname, '../src/pages')

// 多页面入口文件配置
exports.entries = function () {
//  读取pages下的 页面名称文件下的后缀为js的文件
  var entryFiles = glob.sync(PAGE_PATH + '/*/*.js')
  var map = {}
  entryFiles.forEach((filePath) => {
    // 因为后缀为js的文件名在index，而我们想获取的是
    var pathArry = filePath.split('/')
    var filename = pathArry[pathArry.length - 2]
    map[filename] = filePath
  })
  // 整理成文件名为Key，路径为value的Object对象
  return map
}

// 多页面输出配置
exports.htmlPlugin = function () {
  let entryHtml = glob.sync(PAGE_PATH + '/*/*.ejs')
  let arr = []
  entryHtml.forEach((filePath) => {
    let pathArry = filePath.split('/')
    // 获取页面名
    let filename = pathArry[pathArry.length - 2]

    let conf = {
      template: filePath,
      filename: filename + '.html',
      chunks: ['manifest', 'vendor', filename],
      inject: true
    }

    if (process.env.NODE_ENV === 'production') {
      conf = merge(conf, {
        minify: {
          removeComments: true,
          collapseWhitespace: true,
          removeAttributeQuotos: true
        },
        chunksSortMode: 'dependency'
      })
    }
    arr.push(new HtmlWebpackPlugin(conf))
  })
  return arr
}
```

### 修改webpack.base.conf.js

代码如下，修改的部分只有入口文件的配置
```
const path = require('path')
const utils = require('./utils')
const config = require('../config')
const vueLoaderConfig = require('./vue-loader.conf')

function resolve (dir) {
  return path.join(__dirname, '..', dir)
}

const createLintingRule = () => ({
  test: /\.(js|vue)$/,
  loader: 'eslint-loader',
  enforce: 'pre',
  include: [resolve('src'), resolve('test')],
  options: {
    formatter: require('eslint-friendly-formatter'),
    emitWarning: !config.dev.showEslintErrorsInOverlay
  }
})

module.exports = {
  context: path.resolve(__dirname, '../'),
  // 修改的部分=====开始
  entry: utils.entries(),
  // 修改的部分 === 结束
  output: {
    path: config.build.assetsRoot,
    filename: '[name].js',
    publicPath: process.env.NODE_ENV === 'production'
      ? config.build.assetsPublicPath
      : config.dev.assetsPublicPath
  },
  resolve: {
    extensions: ['.js', '.vue', '.json'],
    alias: {
      'vue$': 'vue/dist/vue.esm.js',
      '@': resolve('src'),
    }
  },
  module: {
    rules: [
      ...(config.dev.useEslint ? [createLintingRule()] : []),
      {
        test: /\.vue$/,
        loader: 'vue-loader',
        options: vueLoaderConfig
      },
      {
        test: /\.js$/,
        loader: 'babel-loader',
        include: [resolve('src'), resolve('test'), resolve('node_modules/webpack-dev-server/client')]
      },
      {
        test: /\.(png|jpe?g|gif|svg)(\?.*)?$/,
        loader: 'url-loader',
        options: {
          limit: 10000,
          name: utils.assetsPath('img/[name].[hash:7].[ext]')
        }
      },
      {
        test: /\.(mp4|webm|ogg|mp3|wav|flac|aac)(\?.*)?$/,
        loader: 'url-loader',
        options: {
          limit: 10000,
          name: utils.assetsPath('media/[name].[hash:7].[ext]')
        }
      },
      {
        test: /\.(woff2?|eot|ttf|otf)(\?.*)?$/,
        loader: 'url-loader',
        options: {
          limit: 10000,
          name: utils.assetsPath('fonts/[name].[hash:7].[ext]')
        }
      }
    ]
  },
  node: {
    // prevent webpack from injecting useless setImmediate polyfill because Vue
    // source contains it (although only uses it if it's native).
    setImmediate: false,
    // prevent webpack from injecting mocks to Node native modules
    // that does not make sense for the client
    dgram: 'empty',
    fs: 'empty',
    net: 'empty',
    tls: 'empty',
    child_process: 'empty'
  }
}

```

### webpack.dev.conf.js文件

完整代码比较多，只贴出了关键代码
```
  plugins: [
    new webpack.DefinePlugin({
      'process.env': require('../config/dev.env')
    }),
    new webpack.HotModuleReplacementPlugin(),
    new webpack.NamedModulesPlugin(), // HMR shows correct file names in console on update.
    new webpack.NoEmitOnErrorsPlugin(),
    // https://github.com/ampedandwired/html-webpack-plugin

    //-------- 注掉 配置中的 HtmlWebpackPlugin 这个插件---------------

   // new HtmlWebpackPlugin({
    //  filename: 'index.html',
    //  template: 'index.html',
    //  inject: true
   // }),

   // ------------------------------------------------------------

    // copy custom static assets
    new CopyWebpackPlugin([
      {
        from: path.resolve(__dirname, '../static'),
        to: config.dev.assetsSubDirectory,
        ignore: ['.*']
      }
    ])

    // 添加----------utils中我们写的另一个函数utils.htmlPlugin-----------
  ].concat(utils.htmlPlugin())
  // ---------------------------------------

```

### webpack.prod.conf.js文件
生产环境的配置和开发环境一样，先注掉配置文件中的webpack插件HtmlWebpackPlugin的那段代码，然后在将utils中我们之前写的htmlplugin函数，生成的数组添加到plugin上。操作和上面webpack.dev.conf.js文件相同。

---
好了一切配置完成，现在就可以实验一下了，执行
```
npm run build
```
如果没什么问题的话就可以看到dist目录中生成，在文章开始我们设想的打包之后的页面结构。


**参考资料**：

官网Entry Points介绍
https://webpack.js.org/concepts/entry-points/#src/components/Sidebar/Sidebar.jsx

HtmlWebpackPlugin 
https://github.com/jantimon/html-webpack-plugin

Vue多页面应用 blog
https://segmentfault.com/a/1190000011265006




