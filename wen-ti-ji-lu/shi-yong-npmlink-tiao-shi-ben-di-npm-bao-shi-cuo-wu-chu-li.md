# 使用npm link调试本地npm包时错误处理

## 使用webpack打包库时，使用npm link在本地的项目中进行联调，发现项目中出现如下报错

```text
  Uncaught TypeError: Cannot assign to read only property 'exports' of object '#<Object>'
```

查了很多资料，都是说 commonjs 模块和es模块混用造成的，但是webpack构建出来的组件库本来就都是commonjs规范的，为什么第三方库却不会报错呢？ 经过搜索发现了这篇帖子 [https://forum.vuejs.org/t/vue-cli-library-build-error-cannot-assign-to-read-only-property-exports-of-object-object/55492](https://forum.vuejs.org/t/vue-cli-library-build-error-cannot-assign-to-read-only-property-exports-of-object-object/55492)

经过测试

```javascript
  // vue.config.js
  config.module.rule('js').test(/\.m?jsx?$/).exclude.clear().add((name)=>{
        console.log('exlude-',name)
        return false
      })
```

在loader的exclude配置中打印所有的模块路径，发现在resolve.symlinks=true时,webpakc将路径解析为磁盘上组件库源码所在的项目路径，例如d:/components,设为false之后，将组件库解析为项目下的node\_modules文件例如：d:/project-test/node\_modules/compoents

```javascript
  // vue.config.js

  config.module.rule('js').test(/\.m?jsx?$/).exclude.add((name)=>{
        console.log('exlude-',name)
        if(name.indexOf('\\path\\to\\my\\components\\')!==-1){
          console.log('this is MyConponents')
          return true
        }
        return false
      })
```

在配置中加入以上代码，精确匹配组件库后exclude,启动项目后不再报错，报上面提到的错误，推测是由于路径问题，组件在库经过webpack打包后的代码再次在项目代码中被webpack babel处理所导致的

## 当调试的组件库与业务代码依赖同一个第三方库时的问题

例如当两者同时依赖vue时，组件库的node\_modules目录下也会有vue库的文件存在，这个时候即使两者依赖的版本一样，两个分别引用的都是各自目录下的vue构造函数，这就会导致一些问题

例如

1. 比如在组件库中往vue原型链上增加一些额外方法时，由于引用不同，这些操作并不会在上层的业务代码中生效
2. Vue.use注册的组件不生效

因为两者引用的是不同的Vue构造函数，解决方案：

```javascript
//vue.config.js
const devAlias=Object.keys(packageJson.dependencies).reduce((result,lib)=>{
  result[lib]=path.resolve(__dirname,'./node_modules',lib)
  return result
},{})
module.exports={
    configureWebpack: {
    resolve: {
      symlinks: false,
      alias:devAlias //在这里将业务代码和组件库下相同的依赖引用全部都显式指向项目代码下的node_modules模块
    },
  },
}
```

