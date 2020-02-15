# 基于Vue的虚拟列表实现

## 前言

最近在开发一个项目时遇到一个问题，是在一个信息流列表的页面中，当页面不断滚动加载数据到一定程度后，页面的列表项dom不断增多，到一定数量时页面会出现卡顿，于是决定编写一个虚拟列表组件来进行性能优化。

![&#x6B64;&#x4E3A;&#x76D7;&#x56FE;](../.gitbook/assets/v2-f00bb3f5d9815d660d7bcbd87236af86_hd.jpg)

虚拟列表主要就是减少在dom树中渲染的元素，只渲染屏幕中能被使用者看到的部分（可视列表），屏幕外的部分不生成dom，下面来说下实现。

## 实现

需要实现两个组件，一个为列表容器组件List,一个为列表项容器组件ListItem。

### List

```markup
<template>
  <div class="ac-virtual-list" v-stream:scroll="scroll$" ref="el">
    <div ref="before">
      <slot name="before"></slot>
    </div>
    <div class="ac-virtual-list__main"  ref="main" :style="contentStyle">
      <div :style="{height:`${paddedTop}px`}"></div>
        <ListItem
          ref="items"
          v-for="item in visibleList"
          :key="item.id"
          v-stream:update-rect="{subject:itemUpdated$,data:item.id}"
        >
          <Render :content="item.render"></Render>
        </ListItem>
    </div>
    <div ref="after">
      <slot name="after"></slot>
    </div>
  </div>
</template>
<script>
import Vue from 'vue'
import VueRx from 'vue-rx'
import { debounceTime } from 'rxjs/internal/operators/debounceTime'
import { throttleTime } from 'rxjs/internal/operators/throttleTime'
import { map } from 'rxjs/internal/operators/map'
import { filter } from 'rxjs/internal/operators/filter'
import { startWith } from 'rxjs/internal/operators/startWith'
import { merge } from 'rxjs/internal/observable/merge'
import { scan } from 'rxjs/internal/operators/scan'
import { tap } from 'rxjs/internal/operators/tap'
import ListItem from '@/components/virtual-list-item'
import Render from '@/components/render'
Vue.use(VueRx)
export default {
  name: 'AcVirtualList',
  props: {
    list: { //列表数据
      type: Array,
      default: () => ([]) // {id:string|number, render:Function}[]
    },
    itemMinHeight: {//列表项最小高度
      type: Number,
      default: 30
    },
    throttleTime: {//计算节流间隔
      type: Number,
      default: 300
    },
    outsideRemain: {//可视区域外的列表项数量
      type: Number,
      default: 10
    }
  },
  data () {
    return {
      mainHeight: 0,
      rect: {},
      firstItemIndex: 0,
      lastItemIndex: 0,
      itemsRectCache: {},
      mounted: false
    }
  },
  inject: {},
  activated () {
    this.$refs.el.scrollTop = this.scrollTop
  },
  destroyed () {

  },
  components: {
    ListItem,
    Render
  },
  domStreams: ['scroll$', 'itemUpdated$'],
  subscriptions () {
    this.$subscribeTo(this.scroll$, ({ event }) => {
      this.$emit('scroll', event)
    })
    const scrollEnd$ = this.scroll$.pipe(
      filter(({ event: e }) => {
        return (
          e.target.scrollTop >=
          e.target.scrollHeight -
            e.target.offsetHeight -
            (this.itemMinHeight - 10)
        )
      }),
      map(({ event }) => event)
    )
    this.$subscribeTo(scrollEnd$, e => {
      this.$emit('scroll-end', e)
    })
    const itemsRectCache$ = this.itemUpdated$.pipe(
      startWith(null),
      map(data => {
        if (data) {
          const { event, data: id } = data
          return {
            data: event.msg,
            id
          }
        }
        return null
      }),
      scan((result, current) => {
        if (current) {
          result[current.id] = current.data
        }
        return result
      }, {}),
      debounceTime(200)
    )
    this.$subscribeTo(itemsRectCache$, data => {
      this.itemsRectCache = { ...data }
    })
    return {
      scrollTop: this.scroll$.pipe(
        startWith(0),
        map(data => {
          return data.event ? data.event.target.scrollTop : data
        })
      )
    }
  },
  created () {

  },
  async updated () {
    await this.$nextTick()
    if (this.$refs.el) {
      this.getRect()
    }
  },
  mounted () {
    this.getRect()
    const merged$ = merge(
      this.itemUpdated$,
      this.$observables.scrollTop
    ).pipe(throttleTime(this.throttleTime, undefined, { trailing: true }))
    this.$subscribeTo(merged$, (...args) => {
      // console.log(args)
      this.setVisibleIndex()
    })
  },
  computed: {
    scroller () {
      return this.$refs.el
    },
    paddedTop () {
      return this.list
        .filter((i, index) => index < this.firstItemIndex)
        .reduce((a, b) => {
          const rect = this.itemsRectCache[b.id]
          if (rect) {
            return a + rect.height
          }
          return a + this.itemMinHeight
        }, 0)
    },
    // 渲染出来的列表
    visibleList () {
      const list = this.list.slice(this.firstItemIndex, this.lastItemIndex + 1)
      return list
    },
    contentStyle () {
      const height = this.list.reduce((a, b) => {
        const rect = this.itemsRectCache[b.id]
        if (rect) {
          return a + rect.height
        }
        return a + this.itemMinHeight
      }, 0)
      return {
        minHeight: `${height}px`
      }
    }
  },
  methods: {
    // 生成可视区域的首位列表项序号
    setVisibleIndex () {
      const { scrollTop } = this
      const { height } = this.rect
      // const firstIndex = Math.floor(this.scrollTop / this.itemMinHeight)
      let beforeTotalHeight = 0 // 计算第一个元素之前的所有元素的高度总和
      let innerTotalHeight = 0 // 可视区域所有元素的高度总和
      let firstItemIndex, lastItemIndex
      const beforeSlotHeight = this.$refs.before.offsetHeight
      for (let i = 0; i < this.list.length; i++) {
        const item = this.list[i]
        const rect = this.itemsRectCache[item.id]
        if (rect) {
          if (firstItemIndex === undefined) {
            beforeTotalHeight = beforeTotalHeight + rect.height
          } else {
            innerTotalHeight = innerTotalHeight + rect.height
          }
        } else {
          if (firstItemIndex === undefined) {
            beforeTotalHeight = beforeTotalHeight + this.itemMinHeight
          } else {
            innerTotalHeight = innerTotalHeight + this.itemMinHeight
          }
        }
        if (firstItemIndex === undefined && (beforeTotalHeight + beforeSlotHeight) > scrollTop) {
          firstItemIndex = i > this.outsideRemain ? i - this.outsideRemain : 0
          continue
        }
        if (firstItemIndex !== undefined && innerTotalHeight > height) {
          lastItemIndex = (i + this.outsideRemain > this.list.length - 1) ? this.list.length - 1 : i + this.outsideRemain
          break
        }
      }
      this.firstItemIndex = firstItemIndex || 0
      this.lastItemIndex = lastItemIndex || this.list.length
    },
    getRect () {
      const rect = this.$refs.el.getBoundingClientRect().toJSON()
      for (const prop in rect) {
        this.$set(this.rect, prop, rect[prop])
      }
    }
  },
  watch: {}
}
</script>

```

List组件代码如上。

先来分析模板结构。

```markup
<template>
  <div class="ac-virtual-list" v-stream:scroll="scroll$" ref="el">
    <div ref="before">
      <slot name="before"></slot>
    </div>
    <div class="ac-virtual-list__main" ref="main" :style="contentStyle">
      <div :style="{height:`${paddedTop}px`}"></div>
      <ListItem
        ref="items"
        v-for="item in visibleList"
        :key="item.id"
        v-stream:update-rect="{subject:itemUpdated$,data:item.id}"
      >
        <Render :content="item.render"></Render>
      </ListItem>
    </div>
    <div ref="after">
      <slot name="after"></slot>
    </div>
  </div>
</template>
```

容器的列表项放在**.ac-virtual-list\_\_main**这个div中。

#### contentStyle

```javascript
contentStyle () {
      const height = this.list.reduce((a, b) => {
        const rect = this.itemsRectCache[b.id]
        if (rect) {
          return a + rect.height
        }
        return a + this.itemMinHeight
      }, 0)
      return {
        minHeight: `${height}px`
      }
}
```

itemsRectCache是保存所有列表项的尺寸的对象，因为每次渲染的时候元素的数量并不会太多会导致滚动容器无法滚动，在这里计算出所有列表项的高度总和之后，撑开**.ac-virtual-list\_\_main**让其高度达到所有列表项都渲染出来时的值，达到可以滚动的效果。

#### paddedTop



```javascript
paddedTop () {
  return this.list
    .filter((i, index) => index < this.firstItemIndex)
    .reduce((a, b) => {
      const rect = this.itemsRectCache[b.id]
      if (rect) {
        return a + rect.height
      }
      return a + this.itemMinHeight
    }, 0)
}
```

这里的firstItemIndex变量就是可视列表的第一项的序号，paddedTop就是的值就是可视列表之前的所有项的高度的总和，当列表项滚动出可视区域时dom会被移除，需要一个div来代替所有被移除的部分来撑开滚动容器。

#### visibleList

是由props.list中过滤出来的满足元素在可视区域部分的数组，当ListItem触发update-rect事件时会更新itemsRectCache中的对应id的值，保证paddedTop，contentStyle等值的计算正确，v-strean是[VueR](https://github.com/vuejs/vue-rx)的用法，具体可以参考文档。

下面来看下visibleList是如何生成的

```javascript
  setVisibleIndex () {
      const { scrollTop } = this
      const { height } = this.rect

      let beforeTotalHeight = 0 // 计算第一个元素之前的所有元素的高度总和
      let innerTotalHeight = 0 // 可视区域所有元素的高度总和
      let firstItemIndex, lastItemIndex
      const beforeSlotHeight = this.$refs.before.offsetHeight
      for (let i = 0; i < this.list.length; i++) {
        const item = this.list[i]
        const rect = this.itemsRectCache[item.id]
        const itemHeight = rect ? rect.height : this.itemMinHeight
        if (firstItemIndex === undefined) {
          beforeTotalHeight = beforeTotalHeight + itemHeight
        } else {
          innerTotalHeight = innerTotalHeight + itemHeight
        }
        // 当某个列表项及之前的所有兄弟元素的高度总和大于滚动高度时，这个列表项就是滚动容器中可见的第一项
        if (
          firstItemIndex === undefined &&
          beforeTotalHeight + beforeSlotHeight > scrollTop
        ) {
          firstItemIndex = i > this.outsideRemain ? i - this.outsideRemain : 0
          continue
        }
        // 滚动容器可见的第一项到某一项的高度总和大于滚动容器高度时，这一项就是可见的最后一项
        if (firstItemIndex !== undefined && innerTotalHeight > height) {
          lastItemIndex =
            i + this.outsideRemain > this.list.length - 1
              ? this.list.length - 1
              : i + this.outsideRemain
          break
        }
      }
      this.firstItemIndex = firstItemIndex || 0
      this.lastItemIndex = lastItemIndex || this.list.length
    }
```

通过对列表项的高度计算，判断出滚动容器可视区域的第一项以及最后一项，使用这两个数值过滤出可视列表

```javascript
visibleList () {
    const list = this.list.slice(this.firstItemIndex, this.lastItemIndex + 1)
    return list
}
```

visibleList的计算过程如上所示，但是这个值是需要在滚动时或者列表项的高度发生变化时重新计算的，真个组件中唯一一个调用setVisibleIndex是在mounted生命周期中

```javascript
  mounted () {
    this.getRect()
    const merged$ = merge(this.itemUpdated$, this.$observables.scrollTop).pipe(
      throttleTime(this.throttleTime, undefined, { trailing: true })
    )
    // 列表项尺寸发生变化或者滚动发生时计算可视区域的序号
    this.$subscribeTo(merged$, (...args) => {
      // console.log(args)
      this.setVisibleIndex()
    })
  }
```

在这里订阅了merge过的this.itemUpdated$，this.$observables.scrollTop的数据流，scrollTop在滚动时发出通知，itemUpdated接受到子组件的事件时发出通知，两者只要有其中一个发出事件，就会调用setVisibleIndex进行可视列表的更新，同时使用了throttleTime操作符进行节流避免频繁更新带来的性能损耗

### ListItem

```javascript
<template>
  <div class="ac-virtual-list-item" ref="el">
    <slot></slot>
  </div>
</template>
<script>
export default {
  name: 'AcVirtualItem',
  inject: {

  },
  data () {
    return {
      rect: {}
    }
  },
  computed: {
    el () {
      return this.$refs.el
    }
  },
  created () {
  },
  async updated () {
    await this.$nextTick()
    if (this.$refs.el) {
      this.getRect()
    }
  },
  destroyed () {
    this.rect = null
  },
  mounted () {
    this.getRect()
  },
  methods: {
    getRect () {
      const rect = this.$refs.el.getBoundingClientRect().toJSON()
      for (const prop in rect) {
        this.$set(this.rect, prop, rect[prop])
      }
    }
  },

  watch: {
    rect: {
      immediate: true,
      handler () {
        if (this.rect) {
          this.$emit('update-rect', this.rect)
        }
      }
    }
  }
}
</script>

```

ListItem的实现比较简单，主要就是在updated生命周期中执行this.getRect\(\)更新自身的尺寸数据并同步给父组件，父组件收到通知后便会重新计算可视列表的序号

