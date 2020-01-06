## 在使用vue-cli 3.x打包ts代码为库时，发现不能生成.d.ts声明文件

已在tsconfig.json中指定declaration为true的情况下无法生产类型声明文件

经过搜索发现这个[issue](https://github.com/vuejs/vue-cli/issues/1081#issuecomment-385696405)   
改造vue.config.js文件的部分配置如下即可生成.d.ts文件

```javascript
//  /vue.config.js
module.exports={
  ...,
  parallel:false,
  chainWebpack:config=>{
    config.module
      .use('ts-loader')
      .loader('ts-loader')
      .tap(options=>{
        options.happyPackMode=false
        options.transpileOnly=false
      })
    config.module
      .use('ts-loader')
      .loader('ts-loader')
      .tap(options=>{
        options.happyPackMode=false
        options.transpileOnly=false
      })
  }
}
```

## 在npm包中读取上层项目的环境变量   

**场景**：   
因业务需要，需要在npm库的代码中获取到上层项目代码中webpack注入的process.env环境变量，但是在vue-cli中打包会导致库的代码中也会将process.env变量转为npm库项目中的环境变量文件，于是将vue-cli中默认的webpackDefinePlugin做删除，启动项目后process.env.XXX变量已经是undefine了，但是process.env还是会被编译成一个空对象。   
经过排查发现webpack 4.x会在项目中自动将前端代码中的process.env做替换，于是将webpack构建目标改为node，打包后会发现触发了vue-loader将.vue文件的运行环境识别为SSR，会有报错，将vue-loader的optimizeSSR选项设为false即可

部分配置如下
```js
module.exports = {
  ...,
  chainWebpack: config => {
    config.plugins.delete('define')
    config.module
      .rule('vue')
      .use('vue-loader')
      .loader('vue-loader')
      .tap(options => {
        // 修改它的选项...
        options.optimizeSSR = false
        return options
      })
  },
  configureWebpack: {
    optimization: {
      nodeEnv: false
    },
    target: 'node'
  },
}

```

## ts 环境变量声明

声明通过webpack.DefinePlugin写入的变量


```ts
/// <reference path="../node_modules/@types/webpack-env/index.d.ts" />

declare global {
  namespace NodeJS { //在这里声明所需的环境变量类型，与 *@types/webpack-env/index.d.ts* 下的声明合并
    interface Process {
      env: {
        NODE_ENV: 'production'|'debug'|'development'
        APP_ID:string
      }
    }
  }
}
```

