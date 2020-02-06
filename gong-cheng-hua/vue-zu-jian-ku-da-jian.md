# Vue组件库搭建

​ 参照项目 [https://github.com/sm00thCr1m1n4l/vue-components-demo](https://github.com/sm00thCr1m1n4l/vue-components-demo) 对一些关键点做些说明

1. 使用vue-cli新建一个项目

   ```text
   vue create vue-components
   ```

2. 添加/example作为用例目录
3. 修改 **vue.config.js** 如下

  ```javascript
  //修改**/example/index.ts** 作为开发时的入口
  module.exports = {
    pages:{
      index: './example/index.ts'
    },
    configureWebpack: {
      devtool: 'sourcemap'
    }
  }
  ```

4. 新建 **webpack.components.js**作为打包生产环境的webpack配置

```javascript
const glob = require('glob')
const path = require('path')
const VueLoaderPlugin = require('vue-loader/lib/plugin')
const nodeExternals = require('webpack-node-externals')
const pathFormat = (dir) => dir.replace(new RegExp(`\\${path.sep}`, 'g'), '/')
const srcDir = pathFormat(path.resolve(__dirname, './src'))
const MiniCssExtractPlugin = require('mini-css-extract-plugin')
const OptimizeCSSAssetsPlugin = require('optimize-css-assets-webpack-plugin')
let sourceFileDir = [
  ...glob.sync(path.resolve(__dirname, './src/**/*.*'))
].filter(i => {//遍历src下的所有文件，过滤出需要打包的部分
  const ext = path.extname(i)
  return ['.js', '.ts', '.tsx', '.vue', '.tsx'].includes(ext) && !(/\.d\.ts$/g.test(i))
})
const { peerDependencies, devDependencies, dependencies, name } = require('./package')

const externals = [
  {},
  nodeExternals(),
  (context, request, callback) => {
    //babel和tslib相关的依赖提取到组件库外部
    if (/core-js|corejs|tslib/g.test(request)) {
      return callback(null, 'commonjs ' + request)
    }
    // 库内部依赖间使用@/xxx引用，将打包后文件引用组件库其他部分的代码抽离出来
    if (/^@\/.*/g.test(request)) {  
      return callback(null, `commonjs ${name}/dist/${request.replace('@/', '').replace(/\.(ts|js|jsx|tsx|vue)$/g, '.js')}`)
    }
    callback()
  }
  // ...componentsDir
]
//将package.json中的依赖提取到组件库外部
Object.keys({ ...peerDependencies, ...devDependencies, ...dependencies }).forEach((d) => {
  externals[0][d] = {
    commonjs: d,
    commonjs2: d,
    root: d === 'vue' ? 'Vue' : undefined
  }
})
const entry = {}
//生成
sourceFileDir.forEach(dir => {
  const chunkName = dir.replace(srcDir, '').replace(/^\//, '').replace(/\..*$/, '')
  entry[chunkName] = dir
})

module.exports = {
  entry,
  mode: 'production',
  output: {
    libraryTarget: 'commonjs2',
    path: path.resolve(__dirname, './dist')
  },
  module: {
    rules: [
      // ... 其它规则
      {
        test: /\.tsx?$/,
        loader: [
          'babel-loader',
          {
            loader: 'ts-loader',
            options: {
              compiler: 'ttypescript'
            }
          }
        ],
        exclude: /node_modules/
      },
      {
        test: /\.jsx?$/,
        loader: [
          'babel-loader'
        ],
        exclude: /node_modules/
      },
      {
        test: /\.vue$/,
        loader: [
          {
            loader: 'vue-loader',
            options: {
              optimizeSSR: false
            }
          }
        ]
      },
      {
        test: /\.(png|jpe?g|gif|webp)(\?.*)?$/,
        use: [
          {
            loader: 'url-loader',
            options: {
              limit: 4096,
              fallback: {
                loader: 'file-loader',
                options: {
                  name: 'img/[name].[hash:8].[ext]'
                }
              }
            }
          }
        ]
      },
      {
        test: /\.css$/,
        use: [
          MiniCssExtractPlugin.loader,
          {
            loader: 'css-loader',
            options: {
              importLoaders: 1
            }
          }, // 将 CSS 转化成 CommonJS 模块
          {
            loader: 'postcss-loader',
            options: {
              sourceMap: false
            }
          }
        ]
      },
      {
        test: /\.scss$/,
        use: [
          MiniCssExtractPlugin.loader,
          {
            loader: 'css-loader',
            options: {
              importLoaders: 2
            }
          }, // 将 CSS 转化成 CommonJS 模块
          {
            loader: 'postcss-loader',
            options: {
              sourceMap: false
            }
          },
          'sass-loader' // 将 Sass 编译成 CSS，默认使用 Node Sass
        ]
      }

    ]
  },
  plugins: [
    // 请确保引入这个插件！
    new VueLoaderPlugin(),
    new MiniCssExtractPlugin()
  ],
  resolve: {
    modules: ['node_modules'],
    alias: {
      '@': path.resolve(__dirname, './src'),
      vue$: 'vue/dist/vue.runtime.esm.js'
    },
    extensions: [
      '.mjs',
      '.js',
      '.jsx',
      '.vue',
      '.json',
      '.wasm',
      '.ts',
      '.tsx'
    ]
  },
  externals,
  optimization: {
    minimizer: [new OptimizeCSSAssetsPlugin({})],
    splitChunks: {
      cacheGroups: {
        style: {
          name: 'index',
          test: /\.css$|\.scss$/,
          chunks: 'all',
          enforce: true
        }
      }
    }
  },
  watch: process.env.ENV === 'watch'
}
```

5. 新建一个 **babel-plugin-import-config.js**作为调用组件库的项目的babel-plugin-import的配置

```javascript
/**
 * babel-plugin-import 插件配置
 */
const dasherize = require('dasherize')
const path = require('path')
const fs = require('fs')
const pkgJSON = require('./package.json')
/**
 * options
 * {
 *  style:boolean  是否自动导入样式代码
 * }
 */
const modulePath = path.join(process.cwd(), `./node_modules/${pkgJSON.name}`)
module.exports = ({ style = true } = {}) => {
  console.log(this)
  return {
    libraryName: pkgJSON.name,
    libraryDirectory: 'dist/components',
    camel2DashComponentName: false,
    customName: (name, file) => {
      let fileName = path.join(modulePath, `./dist/components/${dasherize(name)}.js`)
      if (fs.existsSync(fileName)) {
        return fileName.replace(modulePath, pkgJSON.name)
      }
      fileName = path.join(modulePath, `./dist/directivies/${dasherize(name)}.js`)
      if (fs.existsSync(fileName)) {
        return fileName.replace(modulePath, pkgJSON.name)
      }
      throw new Error(`找不到${name}`)
    },
    style: style === false ? false : (name, file) => {
      const scssFilePath = path.resolve(__dirname, `src/styles/components/${path.basename(name).replace('.js', '')}.scss`)
      if (fs.existsSync(scssFilePath)) {
        return scssFilePath
      }
    }
  }
}
```

6.按需引用代码使用

```javascript
  module.exports = {
    ...
    plugins: [
        ...
        [
          'import',
          require('vue-components/babel-plugin-import-config')(),
        ]
    ],
  }
```

\`\`\`

