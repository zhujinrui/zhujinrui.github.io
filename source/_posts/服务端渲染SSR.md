---
title: 服务端渲染SSR（一）
catalog: true
date: 2019-01-31 17:37:09
subtitle:
header-img:
tags:
- node
- ssr
catagories:
- Node

---

#### 一、传统的服务端渲染 VS 客户端渲染

##### 1.传统的服务端渲染
大约十几年前，在传统的像ASP，JSP和PHP等开发模式中，前端是处在一个混沌的状态中，可以说是没有独立的“人格”可言。

前端负责切图和编写静态页面模板，后端将数据渲染到前端提供的页面模板中，最后将页面渲染到浏览器展示。

这个过程中，前端只提供页面模板或者写一些JavaScript脚本，有的甚至JS脚本都是后端来写，前端的作用只局限于切图和样式模板文件，这种角色就是传说中的“切图仔”。

下图这种方式就是服务端渲染，服务端拿到前端的页面模板，在特定的区域用数据填充，再把填充了数据后的HTML返回给客户端，客户端只负责解析HTML。这种方式也被称为 `fat-server,thin-clinet`模式。

 <img src="https://tva1.sinaimg.cn/large/007S8ZIlgy1gegdh7lyqnj30vo0nmgm8.jpg" width="600">

 <img src="https://tva1.sinaimg.cn/large/007S8ZIlgy1gegdhsxvnfj30yg08fglk.jpg" width="600">

前端只负责DOM操作，表单校验，以及动画的处理等；而后端需要提供加载的数据的HTML页面，需要做路由的转化处理，写应用逻辑，保证稳定性等等，大部分的工作都是在后端来处理的。

##### 2.当下的客户端渲染
 目前使用最多的还是客户端渲染模式。客户端发送请求给服务器，服务器把客户端需要的静态文件和接口数据返回给客户端，其中视图的解析过程和渲染过程都放到浏览器端去执行。

 <img src="https://tva1.sinaimg.cn/large/007S8ZIlgy1gegdhzi0nxj30y20nqt9c.jpg" width="600">



<img src="https://tva1.sinaimg.cn/large/007S8ZIlgy1gegdi4mbvuj30yg0dbglo.jpg" width="600">


##### 3.对比
客户端渲染和服务端渲染，实际上对应了两种Web构建模式：**前后端分离模式**和**直出模式**。

> a:下载JS/CSS代码
>
> b:请求数据
>
> c:渲染页面

**模式一：前后端分离模式（对应客户端渲染）**

客户端渲染顺序： a -> b -> c （a,b,c都在客户端进行）

<img src="https://tva1.sinaimg.cn/large/007S8ZIlgy1gegdi9bfwvj30mm0huwee.jpg" width="600">

**模式二：直出模式（对应服务端渲染）**

服务端渲染顺序： b -> c -> a （b,c在服务端进行，最后的a在客户端进行）

<img src="https://tva1.sinaimg.cn/large/007S8ZIlgy1gegdifie6hj30ma0i2q2u.jpg" width="600">



<img src="https://tva1.sinaimg.cn/large/007S8ZIlgy1gegdikonxvj30yg0dewk0.jpg" width="1000">



#####  使用 SSR 技术的主要因素：
1. **减少首屏渲染时间**。CSR 项目的 TTFP（Time To First Page）时间比较长，参考之前的图例，在 CSR 的页面渲染流程中，首先要加载 HTML 文件，之后要下载页面所需的 JavaScript 文件，然后 JavaScript 文件渲染生成页面。在这个渲染过程中至少涉及到两个 HTTP 请求周期，所以会有一定的耗时，这也是为什么大家在低网速下访问普通的 React 或者 Vue 应用时，初始页面会有出现白屏的原因。

2. **SEO**。 CSR 项目的 SEO能力极弱，在搜索引擎中基本上不可能有好的排名。因为目前大多数搜索引擎主要识别的内容还是 HTML，对 JavaScript 文件内容的识别都还比较弱。如果一个项目的流量入口来自于搜索引擎，这个时候你使用 CSR 进行开发，就非常不合适了。


#### 二、同构

同构指的是**客户端和服务端共用一套代码或逻辑**的技术。我们把页面的展示内容和交互写在一起，让代码执行两次。**在服务器端执行一次，用于实现服务器端渲染**，**在客户端再执行一次，用于接管页面交互**。

而在这套代码或逻辑中，理想的状况是在浏览器端进一步渲染的过程中，**判断已有的DOM结构和即将渲染出的结构是否相同**，若相同，则不重新渲染DOM结构，只需要进行事件绑定即可。

本文以React为例，分析同构过程，其中同构步骤如下：
- 服务端将请求交由React Router解析
-   React Router生成页面布局
-   服务端将生成结果文本化返回给客户端
-   客户端由React Router生成页面布局
-  React将其与服务端布局进行对比
- 对比成功，复用当前页面；反之则重新渲染

<img src="https://tva1.sinaimg.cn/large/007S8ZIlgy1gegdinu51ej30f00f9t92.jpg" width="500">


##### 服务端只负责首屏渲染：
> **服务端只负责从浏览器器发送请求的第⼀次渲染**，服务器将该url
对应的⻚页⾯内容发送给浏览器，浏览器下载页面引用的js后执
行客户端路由初始化，随后的路由跳转都是在浏览器端。


在整个同构过程中，我们需要保持路由一致，数据一致。那在React中如何做的呢？React利用了`Redux`的状态管理功能，引入了Redux来保证数据的一致性。

##### 1. 渲染方式

**客户端**: 使用React的ReactDOM模块的hydrate()方法把组件挂在⼀个容器上，然后虚拟dom结合js的dom操作相关的api生成真实dom。

<img src="https://tva1.sinaimg.cn/large/007S8ZIlgy1gegdirazplj30yg0n7tcf.jpg" width="500">


**服务端**：使用`ReactDom/server`中的`renderToString()`⽅法将虚拟dom直接转化为HTML标签，然后将HTML字符串返回给客户端。

<img src="https://tva1.sinaimg.cn/large/007S8ZIlgy1gegditi51nj30xe0cugnc.jpg" width="500">

##### 2. 保持路由一致

我们都知道客户端有：`BrowserRouter`和`HashRouter`，但服务端是不支持的，它支持的是`StaticRouter`。但是里面的`routes`是可以公用的。

比如这部分路由的定义是可以共用的：

```js
import React from 'react';
import { Route } from 'react-router-dom';
import Home from './container/Home';
import Login from './container/Login';
export default (
    <div>
        <Route path="/" exact component={Home}></Route>
        <Route path="/login" component={Login}></Route>
    </div>
)
```

**客户端**直接用`BrowserRouter`包裹上面导出的模块即可。客户端的路由是`动态`的，能监听浏览器地址栏url的变化，然后返回对应的视图。
```js
import React from 'react';
import ReactDom from 'react-dom';
import { BrowserRouter, Route } from 'react-router-dom';
import { renderRoutes } from 'react-router-config';
import routes from '../Routes';
import { getClientStore } from '../store';
import { Provider } from 'react-redux';

const store = getClientStore();

const App = () => {
	return (
		<Provider store={store}>
			<BrowserRouter>
				<div>
					{renderRoutes(routes)}
	    	</div>
			</BrowserRouter>
		</Provider>
	)
}

ReactDom.hydrate(<App />, document.getElementById('root'))

```


**服务端**使⽤的是`react-router-dom`提供的`StaticRouter` api，就只是匹配页面reload时浏览器地址栏的url，然后返回对应的视图，不能去监听变化，只是请求的url是什么就返回什么。

服务端的`StaticRouter`不能简单的包裹，因为这两个组件需要传两个`props`：
- location
- context

```js
import React from 'react';
import { renderToString } from 'react-dom/server';
import { StaticRouter, Route } from 'react-router-dom';
import { renderRoutes } from 'react-router-config';
import { Provider } from 'react-redux';

const content = renderToString((
	<Provider store={store}>
		<StaticRouter location={req.path} context={context}>
			<div>
				{renderRoutes(routes)}
		</div>
		</StaticRouter>
	</Provider>
	));
```

##### 3.保持数据一致

前端不能用原生的`ajax`请求，因为ajax请求用的是浏览器的`XMLHttpRequest`对象，该方式只在`浏览器环境`支持。

可以使⽤用`fetch框架`的请求⽅方式，该方式可在多端运行，使用 `isomorphic-fetch `可以同时照顾 `node` 和 `browser` 环境。
<img src="https://tva1.sinaimg.cn/large/007S8ZIlgy1gegdj06e49j30yg08smzw.jpg" width="500">



**前端得到数据之后，如何把数据存储起来？**

服务端渲染返回的是`HTML字符串`，**不能**执行到React的⽣命周期函数，所以在⾸屏⻚⾯中React的`setState`⽅法⽆法执行，也就意味着浏览器⽆法将⻚⾯上显示的数据以state的形式存起来，如果后续页⾯的点击等操作都是和当前⻚⾯显示数据相关，就不能继续执行了。


这里采用`Redux`做全局状态管理，把所有的数据都存在一个`store`中，也就是全局只有⼀个store，然后把数据以`props`的形式下发给每个子⻚⾯，这样⻚⾯在使⽤用数据时只⽤去props取就可以了。

在**客户端渲染**中，异步数据结合 Redux的使用方式遵循下面的流程：
1. 创建 Store
2. 根据路由显示组件
3. 派发 Action 获取数据
4. 更新 Store 中的数据
5. 组件 Rerender

而在**服务端**,页面内容一旦确定，就没办法重新渲染了，这就要求组件显示的时候，就要把Store都准备好，遵循的流程如下：
1. 创建 Store
2. 根据路由分析 Store 中需要的数据
3. 派发 Action 获取数据
4. 更新Store 中的数据
5. 结合数据和组件生成 HTML，一次性返回


下面，我们分析下服务器端渲染这部分的流程：

###### (1)创建Store。
在服务器端渲染中，Store的创建应该像下面这样，返回一个函数，每个用户访问的时候，这个函数重新执行，为每个用户提供一个独立的 Store：

```js
const getStore = (req) => {
  return createStore(reducer, defaultState);
}
export default getStore;
```
###### (2)获取数据
```js
Home.loadData = (store) => {
  return store.dispatch(getHomeList())
}
```
###### (3)更新数据

更新 Store 中的数据: 我们要在生成 HTML 之前，保证所有的数据都获取完毕，这怎么处理呢？

```js
// matchedRoutes 是当前路由对应的所有需要显示的组件集合
matchedRoutes.forEach(item => {
  if (item.route.loadData) {
    const promise = new Promise((resolve, reject) => {
      item.route.loadData(store).then(resolve).catch(resolve);
    })
    promises.push(promise);
  }
})

Promise.all(promises).then(() => {
  // 生成 HTML 逻辑
})
```

这里，我们使用 `Promise` 来解决这个问题，我们构建一个 Promise 队列，等待所有的 Promise 都执行结束后，也就是所有 store.dispatch 都执行完毕后，再去生成 HTML。这样的话，我们就实现了结合 Redux 的 SSR 流程。

在上面，我们说到，服务器端渲染时，页面的数据是通过 loadData 函数来获取的。而在客户端，数据获取依然要做，因为如果这个页面是你访问的第一个页面，那么你看到的内容是服务器端渲染出来的，但是如果经过 react-router 路由跳转道第二个页面，那么这个页面就完全是客户端渲染出来的了，所以客户端也要去拿数据。

在客户端获取数据，使用的是我们最习惯的方式，通过 componentDidMount 进行数据的获取。这里要注意的是，componentDidMount 只在客户端才会执行，在服务器端这个生命周期函数是不会执行的。所以我们不必担心 componentDidMount 和 loadData 会有冲突，放心使用即可。这也是为什么数据的获取应该放到 componentDidMount 这个生命周期函数中而不是 componentWillMount 中的原因，可以避免服务器端获取数据和客户端获取数据的冲突。

然而在客户端，一开始的store是为空的，然后再异步请求成功后再渲染。这样会造成一个问题，在渲染时，明明是有数据的，但因为客户端的问题，会先出现空白，再有数据的现象。要解决这个问题，也很容易。服务端需要先写出一个全局的对象（就是store的数据）：

<img src="https://tva1.sinaimg.cn/large/007S8ZIlgy1gegdj4ou3qj30yg0nbgq7.jpg" width="500">



具体的ssr实例可以参看 [这里](https://github.com/alexnm/react-ssr)

**Reference**

- [React 中同构（SSR）原理脉络梳理](https://segmentfault.com/a/1190000016722457)
- [react ssr笔记](https://www.my-fe.pub/post/react-ssr-note.html)
- [【redux】详解react/redux的服务端渲染：页面性能与SEO](http://www.cnblogs.com/penghuwan/p/7126054.html)
- 

