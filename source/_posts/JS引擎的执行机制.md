---
title: JS引擎的执行机制
catalog: true
date: 2019-01-29 14:19:51
subtitle:
header-img:
tags:
- JavaScript
- async/await
- Promise
- 宏任务与微任务
catagories:
- JavaScript
---


#### 1.浏览器引擎（浏览器内核）
浏览器内核的是一个多线程处理，它主要包含如下几个**线程**：
- GUI渲染线程： 渲染页面的html元素
- JavaScript引擎线程： 页面的交互和dom渲染
- 定时触发器线程：一定时间后，来触发对应的线程
- 事件触发线程：当一个事件触发该线程的时候，就会把它放到js的事件队列中等待执行。常用于异步操作。
- 异步http线程：在XMLHttpRequest在连接后是通过浏览器新开一个线程请求， 将检测到状态变更时，如果设置有回调函数，异步线程就产生状态变更事件放到 JavaScript引擎的处理队列中等待处理。

**联系：**
- JavaScript引擎和GUI引擎互斥，不能一边操作dom一边渲染页面
- JavaScript引擎是单线程，所有需要按照事件处理队列来处理相应的代码。
- JavaScript引擎有一个**监听事件**（monitoring process）的功能，会持续不断的检查js引擎的主线程执行栈是否为空，如果为空就会去取**事件触发线程**存放在事件队列中的回调函数执行。


#### 2.JS引擎执行机制
由于js的**运行环境**是单线程的，一些异步操作还是需要借助于浏览器这个宿主来实现。这里简单的一个图来描述`js`运行的时候的流程。主要运用了浏览器的**js引擎线程和事件触发线程**，有时候开启网络服务和定时器也会用到其他的线程。


![image.png](https://tva1.sinaimg.cn/large/007S8ZIlgy1gegdpe5acaj30lz0pjq54.jpg)


#### 3.什么是宏任务与微任务？

- 宏任务：当前调用栈中执行的代码称为宏任务。（主代码块，定时器等）。

- 微任务：当前（此次事件循环中）宏任务执行完，在下一个宏任务开始之前需要执行的任务，可以理解为回调事件。（promise.then,process.nextTick等）。

- 宏任务中的事件放在callback queue中，由`事件触发线程`维护；微任务的事件放在微任务队列中，由`js引擎线程`维护。


##### 3.1JS的执行步骤
- 执行一个宏任务,过程中如果遇到微任务,就将其放到微任务的【事件队列】里
- 当前宏任务执行完成后,会查看微任务的【事件队列】,并将里面全部的微任务依次执行完
- 等到所有的微任务执行完成后，开始执行下一个宏任务。



> 一个经典的代码片段
```js
 setTimeout(function(){
     console.log('4')
 });
 
 new Promise(function(resolve){
     console.log('1');
     for(var i = 0; i < 10000; i++){
         i == 99 && resolve();
     }
 }).then(function(){
     console.log('3')
 });
 
 console.log('2');
```
输出结论：1、2、3、4

其中`setTimeout`作为`宏任务`存在的，而`Promise.then`则是具有代表性的微任务。
**所有会进入的异步都是指的事件回调中的那部分代码**。也就是说`new Promise`在实例化的过程中所执行的代码都是同步进行的，而`then`中注册的回调才是异步执行的。在同步代码执行完后才去检查是否有异步任务完成，并执行对应的回调。


#### 4.async/await
> async 函数返回的是一个 Promise 对象。它表示函数内部有异步操作。

```js
async function testAsync() {
    return "hello async";
}

const result = testAsync();
console.log(result);
```
看到输出的是一个Promise对象：
```
Promise { 'hello async' }
```
> await 在等待一个表达式的结果，这个结果可以是`Promise`对象，也可以是一个值。如果它等的是一个值，那这个值就是它要等的东西。如果它等到的是一个Promise对象，await会让出线程，阻塞Promise对象中的后面的代码。


**我们以开篇的经典面试题为例，分析这个例子中的宏任务和微任务。**
```js
async function async1(){
    console.log('async1 start');
    await async2();
    console.log('async1 end');
}

async function async2(){
    console.log('async2');
}

console.log('script start');

setTimeout(function(){
    console.log('setTimeout');
}, 0)

async1();

new Promise( funxtion (resolve){
    console.log('promise1')
    resolve();
}).then(function(){
    console.log('promise2')
})

console.log('script end')

```
**输出结果**：
```
1. script start 
2. async1 start
3. async2
4. promise1
5. script end
6. promise2
7. async1 end
8. setTimeout

```




##### 具体过程分析如下：

1. 首先是两个函数的声明，虽然有async关键字，但是不调用我们就不看。然后就是打印同步代码`console.log('script start');`

宏任务1 console.log('script start') | 空的微任务队列
---|---

2. **将setTimeout加入宏任务队列中**

宏任务1 console.log('script start') | 空的微任务队列
---|---
宏任务2  console.log('setTimeout');| 空的微任务队列

3. **调用async1，打印同步代码**` console.log('async1 start');`

宏任务1 console.log('script start')  console.log('async1 start') | 空的微任务队列
---|---
宏任务2  console.log('setTimeout');| 空的微任务队列

4. **分析一下await async2()**
前文提过await，它先计算出右侧的结果，然后看到await后，中断async函数：
- 先得到await右侧表达式的结果。执行 `async2()`，打印同步代码` console.log('async2')`，并且`return Promise.resolve(undefined)`。
- await后，中断async函数，先执行async外的同步代码

目前就直接打印出`console.log('async2')`
宏任务1 console.log('script start')  console.log('async1 start') console.log('async2') | 空的微任务队列
---|---
宏任务2  console.log('setTimeout');| 空的微任务队列

被阻塞后，要执行async之外的代码。
5. **执行`new Promise()`**，Promise构造函数是直接调用的同步代码，所以 `console.log('promise1')`：

宏任务1 console.log('script start')  console.log('async1 start') console.log('async2') console.log('promise1')| 空的微任务队列
---|---
宏任务2  console.log('setTimeout');| 空的微任务队列

6. **代码运行到`promise.then()`**。发现这是一个微任务，所以暂时不打印，只是推入到当前宏任务的微任务队列中。微任务会在当前宏任务的同步代码执行完毕，才会依次执行：

宏任务1 console.log('script start')  console.log('async1 start') console.log('async2') console.log('promise1')| console.log( 'promise2' )
---|---
宏任务2  console.log('setTimeout');| 空的微任务队列

7. **打印同步代码**`console.log('script end')`

宏任务1 console.log('script start')  console.log('async1 start') console.log('async2') console.log('promise1') console.log('script end') | console.log( 'promise2' ) 
---|---
宏任务2  console.log('setTimeout');| 空的微任务队列

8. **回到async内部，执行**`await Promise.resolve(undefined)`

因为`await`操作符是是会等到Promise正常处理完成并返回处理结果。在我们这里，就是等到`Promise.resolve(undefined)`正常处理完，并返回结果。那么`await async2()`就算执行结束了。

回忆平时我们用promise，调用resolve后，何时能拿到处理结果？是不是需要在then的第一个参数里，才能拿到结果。

（调用resolve时，会把then的参数推入微任务队列，等主线程空闲时，再调用它）。

所以这里的`await Promise.resolve()`就类似于：
```js
Promise.resolve(undefined).then((undefined) => {
})
```

把then的第一个回调参数 `(undefined)=>{} `推入微任务队列。

`await async2()` 执行结束，才能继续执行后面的代码，如图：

宏任务1 console.log('script start')  console.log('async1 start') console.log('async2') console.log('promise1') console.log('script end') | console.log( 'promise2' )     (undefined)=>{}
---|---
宏任务2  console.log('setTimeout');| 空的微任务队列


此时当前宏任务1都执行完了，要处理微任务队列里的代码。

微任务队列，先进选出的原则：
- 执行微任务1，打印promise2
- 执行微任务2，没什么内容..

但是微任务2执行后， awaitasync2() 语句结束，后面的代码不再被阻塞，所以打印：
```js
console.log( 'async1 end')
```

**宏任务1执行完成后，执行宏任务2**,宏任务2的执行比较简单，就是打印：
```js
console.log('setTimeout')
```




##### Reference:
- [event loop一篇文章足矣](https://www.jianshu.com/p/de7aba994523)
- [8 张图帮你一步步看清 async/await 和 promise 的执行顺序](https://mp.weixin.qq.com/s/2fnJADWMneTg6Zxl_oVahA)


