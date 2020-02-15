# BFC

### 描述

BFC 可以简单的理解为某个元素的一个 CSS 属性，只不过这个属性不能被开发者显式的修改，拥有这个属性的元素对内部元素和外部元素会表现出一些特性，这就是BFC。

### 触发条件

[参考](https://developer.mozilla.org/zh-CN/docs/Web/Guide/CSS/Block_formatting_context)

> * 根元素\(`<html>`\)
> * 浮动元素（元素的 `float` 不是 `none`）
> * 绝对定位元素（元素的 [`position`](https://developer.mozilla.org/zh-CN/docs/Web/CSS/position) 为 `absolute` 或 `fixed`）
> * 行内块元素（元素的 [`display`](https://developer.mozilla.org/zh-CN/docs/Web/CSS/display) 为 `inline-block`）
> * 表格单元格（元素的 [`display`](https://developer.mozilla.org/zh-CN/docs/Web/CSS/display)为 `table-cell`，HTML表格单元格默认为该值）
> * 表格标题（元素的 [`display`](https://developer.mozilla.org/zh-CN/docs/Web/CSS/display) 为 `table-caption`，HTML表格标题默认为该值）
> * 匿名表格单元格元素（元素的 [`display`](https://developer.mozilla.org/zh-CN/docs/Web/CSS/display)为 `table、table-row`、 `table-row-group、table-header-group、table-footer-group`（分别是HTML table、row、tbody、thead、tfoot的默认属性）或 `inline-table`）
> * [`overflow`](https://developer.mozilla.org/zh-CN/docs/Web/CSS/overflow) 值不为 `visible` 的块元素
> * [`display`](https://developer.mozilla.org/zh-CN/docs/Web/CSS/display) 值为 [`flow-root`](https://drafts.csswg.org/css-display/#valdef-display-flow-root) 的元素
> * [`contain`](https://developer.mozilla.org/zh-CN/docs/Web/CSS/contain) 值为 `layout`、`content`或 paint 的元素
> * 弹性元素（[`display`](https://developer.mozilla.org/zh-CN/docs/Web/CSS/display)为 `flex` 或 `inline-flex`元素的直接子元素）
> * 网格元素（[`display`](https://developer.mozilla.org/zh-CN/docs/Web/CSS/display)为 `grid` 或 `inline-grid` 元素的直接子元素）
> * 多列容器（元素的 [`column-count`](https://developer.mozilla.org/zh-CN/docs/Web/CSS/column-count) 或 [`column-width`](https://developer.mozilla.org/zh-CN/docs/Web/CSS/column-width) 不为 `auto，包括 column-count` 为 `1`）
> * `column-span` 为 `all` 的元素始终会创建一个新的BFC，即使该元素没有包裹在一个多列容器中（[标准变更](https://github.com/w3c/csswg-drafts/commit/a8634b96900279916bd6c505fda88dda71d8ec51)，[Chrome bug](https://bugs.chromium.org/p/chromium/issues/detail?id=709362)）。

### 应用

#### 清除margin重叠

```markup
<template>
  <div id="app">
    <div></div>
    <div>
      <div></div>
    </div>
  </div>
</template>

<style lang="scss">
#app{
  div{
    width: 100px;
    height: 100px;
    overflow: hidden;
    &:nth-child(1){
      background-color: red;
      margin-bottom: 10px;
    }
    &:nth-child(2){
      background-color: green;
      div{
        margin-top: 10px;
      }
    }
  }
}
</style>

```

给\#app下的两个div设置overflow: hidden，产生两个独立的BFC

#### 清除浮动

```markup
<template>
  <div id="app">
    <div></div>
  </div>
</template>

<style lang="scss">
#app {
  background-color: green;
  overflow: hidden;
  > div {
    width: 100px;
    height: 100px;
    background-color: red;
    float: left;
  }
}
</style>

```

正常情况下\#app下的div设置浮动之后，会导致\#app塌陷，\#app触发BFC之后会使其撑开

#### 两列布局

```markup
<template>
  <div id="app">
    <div></div>
    <div></div>
  </div>
</template>

<style lang="scss">
#app {
  background-color: green;
  overflow: hidden;
  padding: 20px;
  > div {
    height: 100px;
    background-color: red;
    &:nth-child(1){
      float: left;
      width: 100px;
    }
    &:nth-child(2){
      background-color: blue;
      overflow: hidden;
    }
  }
}
</style>

```

nth-child\(2\)触发了bfc，使其不会与浮动元素重叠并填满了右侧区域

