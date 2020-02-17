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

SPA应用**computed**为**getters**时的部分代码如下

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

由上可知调用了**createComputedGetter**来创建getter并最终**Object.defineProperty**赋值给vue实例，使其可以通过**vm.xxx**访问到**computed**的计算结果

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

**createComputedGetter**就是创建 **computed**的核心方法，首先获取到**watcher**实例也就是之前**initComputed** 中实例化的**watcher**，根据初始化**computed**时的参数以及wacher中的构造函数代码，首次读取**computed**的值时**watcher.dirty===true**,那么便会执行**watcher.evaluate\(\)**，而这个方法中又会执行**wartcher.get\(\),**

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

 **getter** 便是在**computed**中传入的函数，如果这个函数有调用到**options.data**上面的值，则会进入**reactiveGetter**

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

因为在**computed watcher**中执行了**pushTarget**，所以此时**Dep.target**为**computed watcher**，执行**dep.depend\(\)**会使**computed watcher**订阅当前**reactiveGetter**的**dep**，这样当这个字段有更新时，就会调用**watcher.update**,将**computed watcher**的**dirty**变为**true**,使其下次取值时能够重新进行计算，接着继续进行**watcher.get**

```javascript
get(){
    //...
    popTarget()
}
```

上一篇中讲过，在实例渲染时会实例化一个渲染**watcher**，在这个**watcher**的回调中执行渲染函数，会读取**vm**上的字段，也就是在这里首次执行了**createComputedGetter**返回的**getter**，渲染**watcher**实例化过程中也会调用**pushTarget**，因此在**computed watcher**之前会有一个渲染**watcher**，在computed watcher的**get**方法中将自身出栈，此时的**Dep.target**为渲染**watcher**，也就是执行**createComputedGetter**的以下代码片段时，**Dep.target**为渲染**watcher**

```javascript
 if (Dep.target) {
        watcher.depend()
 }
return watcher.value
```

此时经过**watcher.evaluate\(\)**之后**watcher.deps**包含了之前读取过的所有字段的**reactiveGetter**闭包中的**dep**实例

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

执行**watcher.depend\(\)**会将之前所有**reactiveGetter**的**dep**对象与渲染**watcher**进行绑定，当这些**reactiveSetter**被调用时会通知所有的渲染**watcher**进行更新，这也就是为什么渲染函数中没有直接读取**data**中的值而是读取了经过**computed**计算的值，当**data**数据更新时也能触发试图更新，读取最新的**computed**值



