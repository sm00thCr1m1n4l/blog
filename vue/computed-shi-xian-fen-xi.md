# computed实现分析

Vue实例初始化过程中，会执行**initState\(vm\)**方法，也就是在这里开始进行**computed**的初始化

```javascript
export function initState (vm: Component) {
  vm._watchers = []
  const opts = vm.$options
  if (opts.props) initProps(vm, opts.props)
  if (opts.methods) initMethods(vm, opts.methods)
  if (opts.data) {
    initData(vm)
  } else {
    observe(vm._data = {}, true /* asRootData */)
  }
  if (opts.computed) initComputed(vm, opts.computed)
  if (opts.watch && opts.watch !== nativeWatch) {
    initWatch(vm, opts.watch)
  }
}
```

**computed** 是通过调用 **initComputed** 实现

```javascript
function initComputed (vm: Component, computed: Object) {
    //...
    if (!isSSR) {
      // create internal watcher for the computed property.
      watchers[key] = new Watcher(
        vm,
        getter || noop,
        noop,
        computedWatcherOptions
      )
    }
    if (!(key in vm)) {
      defineComputed(vm, key, userDef)
    }
    //...
}
```

初始化**computed**首先会初始化一个**Watcher**，然后调用**defineComputed**

SPA应用computed为getters时的部分代码如下

```javascript
export function defineComputed (
  target: any,
  key: string,
  userDef: Object | Function
) {
  const shouldCache = !isServerRendering()
  if (typeof userDef === 'function') {
    sharedPropertyDefinition.get = shouldCache
      ? createComputedGetter(key)
      : createGetterInvoker(userDef)
    sharedPropertyDefinition.set = noop
  }
  Object.defineProperty(target, key, sharedPropertyDefinition)
}
```

由上可知调用了**createComputedGetter**来创建getter并最终Object.defineProperty赋值给vue实例，使其可以通过vm.xxx访问到computed的计算结果

```javascript
function createComputedGetter (key) {
  return function computedGetter () {
    const watcher = this._computedWatchers && this._computedWatchers[key]
    if (watcher) {
      if (watcher.dirty) {
        watcher.evaluate()
      }
      if (Dep.target) {
        watcher.depend()
      }
      return watcher.value
    }
  }
}
```

**createComputedGetter**就是创建 **computed**的核心方法，首先获取到**watcher**实例也就是之前**initComputed** 中实例化的watcher，根据初始化computed时的参数以及wacher中的构造函数代码，首次读取computed的值时watcher.dirty===true,那么便会执行watcher.evaluate\(\)，而这个方法中又会执行wartcher.get\(\),

```javascript
get () {
    pushTarget(this)
    let value
    const vm = this.vm
    try {
      value = this.getter.call(vm, vm)\
      }
  }
```

 getter 便是在computed中传入的函数，如果这个函数有调用到options.data上面的值，则会进入reactiveGetter

```javascript
get: function reactiveGetter () {
      const value = getter ? getter.call(obj) : val
      if (Dep.target) {
        dep.depend()
        if (childOb) {
          childOb.dep.depend()
          if (Array.isArray(value)) {
            dependArray(value)
          }
        }
      }
      return value
},
```

因为在computed watcher中执行了pushTarget，所以此时Dep.target为computed watcher，执行dep.depend\(\)会使computed watcher订阅当前reactiveGetter的dep，这样当这个字段有更新时，就会调用watcher.update,将computed watcher的dirty变为true,使其下次取值时能够重新进行计算，接着继续进行watcher.get

```javascript
get(){
    //...
    popTarget()
}
```

上一篇中讲过，在实例渲染时会实例化一个渲染watcher，在这个watcher的回调中执行渲染函数，会读取vm上的字段，也就是在这里首次执行了createComputedGetter返回的getter，渲染watcher实例化过程中也会调用pushTarget，因此在computed watcher之前会有一个渲染watcher，在computed watcher的get方法中将自身出栈，此时的Dep.target为渲染watcher，也就是执行createComputedGetter的以下代码片段时，Dep.target为渲染watcher

```javascript
 if (Dep.target) {
        watcher.depend()
 }
return watcher.value
```

此时经过watcher.evaluate\(\)之后watcher.deps包含了之前读取过的所有字段的reactiveGetter闭包中的dep实例

```javascript
//watcher
depend () {
    let i = this.deps.length
    while (i--) {
      this.deps[i].depend()
    }
  }
```

```javascript
 //Dep
 depend () {
    if (Dep.target) {
      Dep.target.addDep(this)
    }
  }
```

执行watcher.depend\(\)会将之前所有reactiveGetter的dep对象与渲染watcher进行绑定，当这些reactiveSetter被调用时会通知所有的渲染watcher进行更新，这也就是为什么渲染函数中没有直接读取data中的值而是读取了经过computed计算的值，当data数据更新时也能触发试图更新，读取最新的computed值

```javascript
watcher.depend()
```

