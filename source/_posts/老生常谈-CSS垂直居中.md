---
title: 老生常谈-CSS垂直居中
catalog: true
date: 2019-01-28 10:40:44
subtitle:
header-img:
tags:
- CSS
catagories:
- CSS
---



在 CSS 中对元素进行水平居中是非常简单的：如果它是一个**行内元素**， 就对它的父元素应用`text-align:center;` 如果它是一个**块级元素**，就对它自身应用 `margin:auto`。然而如果要对一个元素进行垂直居中，可能光是想想就令人头皮发麻了。

我们将使用如下所示的结构代码，并直接插入 元素中（但实际上我们将要探索的这些技巧是与容器无关的）：
```html
<main>
  <h1>Am I centered yet?</h1>
  <p>Center me, please!</p>
</main>
```
##### 1.基于calc()函数
```css
 main {
     position: absolute;
     top: calc(50% - 3em);
     left: calc(50% - 9em);
     width: 18em;
     height: 6em;
 }
```

显然，这个方法最大的局限在于它要求元素的宽高是固定的。

##### 2.CSS transform

这个方法不需要把容器的宽高写死。translate()变形函数使用百分比时是**以这个元素的宽度和高度为基准进行换算和移动的**，这正是我们所需要的。

```css
main {
    position: absolute;
    top: 50%;
    left: 50%;
    transform: translate(-50%,-50%);
}
```

- 我们**有时不能选用绝对定位**，因为它对整个布局的影响太过强烈。
- 如果**需要居中的元素已经在高度上超过了视口**，那它的顶部会被视口裁切掉。有一些办法可以绕过这个问题，但 hack 味道过浓。
- 在某些浏览器中，这个方法可能会导致元素的显示有一些模糊，因 为元素可能被放置在半个像素上。这个问题可以用transform-style:preserve-3d 来修复，不过这个修复手段也可以认为是一个 hack，而且很难保证它在未来不会出问题。

##### 3.基于viewport
- vw: 1% of viewport’s width
- vh: 1% of viewport’s height
- vmin ：1% of viewport’s smaller dimension
- vmax ：1% of viewport’s larger dimension


在我们的这个例子中，适用于外边距的是 vh 单位：

```css
main {
   width: 18em;
   padding: 1em 1.5em;
   margin: 50vh auto 0;
   transform: translateY(-50%);
}
```

可以看到，其效果堪称完美。当然，这个技巧的实用性是相当有限的，因为**它只适用于在视口中居中的场景**。

##### 4.基于Flexbox
- 这是毋庸置疑的最佳解决方案，因为 Flexbox 是专门针对这类需求所设计的。

我们只需要写两行声明即可：
- 先给这个待居中元素的**父元素设置** `display:flex`（在这个例子中是 <body> 元素）。
- 再给**这个元素自身**设置我们再熟悉不过的 `margin:auto`（在这个例子中是 <main> 元素）。

```css
 body {
     display: flex;
     min-height: 100vh;
     margin: 0;
 }
 main {
     margin: auto;
 }
```

**请注意，当我们使用 Flexbox 时，margin: auto 不仅在水平方向上将元素居中，垂直方向上也是如此。**

Flexbox 的另一个好处在于，它还可以将**匿名容器**(即没有被标签包裹的文本节点)垂直居中。举个例子，假设我们的结构代码是：
```html
<main>Center me, please!</main>
```
我们先给这个 main 元素指定一个固定的尺寸，然后借助 Flexbox 规范 所引入的 `align-items` 和 `justify-content` 属性，我们可以让它内部的文本也实现居中：

```css
main {
    display: flex;
    align-items: center;
    justify-content: center; 
    width: 18em;
    height: 10em;
}
```


##### 5. 关于未来
根据【CSS Box Alignment Module Level 3】(http://w3.org/TR/css-align) 的计划，在未来，对于简单的垂直居中需求， 我们完全不需要动用特殊的布局模式了。因为只需要下面这行代码就可以搞定：

```css
align-self: center;
```

参考资料：
- [前端大全](https://mp.weixin.qq.com/s/uoDa5R0AArnMgjUVuxlH7A)