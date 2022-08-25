### 一. 浏览器为什么要有事件循环机制？

由于 js 是单线程的脚本语言，在同一时间，只能做同一件事，为了协调事件、用户交互、脚本、UI 渲染和网络处理等行为，防止主线程阻塞，Event Loop 方案应运而生。

### 二. 为什么 js 是单线程？

js 作为主要运行在浏览器的脚本语言，js 主要用途之一是操作 DOM。

在 js 高程中举过一个例子，如果 js 同时有两个线程，同时对同一个 dom 进行操作，这时浏览器应该听哪个线程的，如何判断优先级？

为了避免这种问题，js 必须是一门单线程语言，并且在未来这个特点也不会改变。

### 三. 两种任务

宏任务：整体 `script` 代码，`setTimeout`，`setInterval`，UI 渲染，I/O 操作

微任务：`new Promise().then()`, `MutaionObserver`(前端的回溯，会在指定的 DOM 发生变化时被调用)

### 四. 为什么有微任务，只有宏任务不行吗？

宏任务遵循「先进先出」的原则执行，无法根据优先级执行。

### 五. 宏任务和微任务的执行顺序

1.  先执行宏任务

2.  遇到微任务就将其放到微任务队列中

3.  遇到宏任务就将其放到宏任务队列中

4.  等此次宏任务执行完毕，检查微任务队列，如果有微任务，全部执行完

5.  接着在宏任务队列执行下一个宏任务，以此循环

![iShot_2022-08-24_16.04.41.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/fb25480c65d74c0a972d65fe6aa131d6~tplv-k3u1fbpfcp-watermark.image?)

### 六. 示例 1

```js
async function async1() {
  console.log('async1 start')
  await async2()
  console.log('async1 end')
}
async function async2() {
  console.log('async2')
}
console.log('script start')
setTimeout(() => {
  console.log('setTimeout')
}, 0)
async1()
new Promise(function (resolve) {
  console.log('promise1')
  resolve()
}).then(function () {
  console.log('promise2')
})
console.log('script end')

// script start
// async1 start
// async2
// promise1
// script end
// async1 end
// promise2
// setTimeout
```

##### 开始一顿猛分析：

###### 第一轮宏任务

1. 定义 `async1()` 函数，未调用
2. 定义 `async2()` 函数，未调用
3. 执行 `console.log('script start')`，打印出 `'script start'`
4. 遇到 `setTimeout`，属于宏任务，放到下一轮执行
5. 调用 `async1()`函数，打印出 `'async1 start'`
6. `async1()` 方法体内遇到执行 `await async2()` 函数，`await` 的作用相当于将后面的函数放入 `new Promise` 中，将之后后面的内容放到 `.then()` 中，比如:

```js
await async2()
console.log('async1 end')
```

就相当于：

```js
new Promise(async2(resolve, reject){
    resolve();
}).then(() => {
    console.log('async1 end');
})
```

所以执行 `async2()` 函数，并打印出 `'async2'`, 同时将 `console.log('async1 end')` 放入微任务队列

7. 遇到 `new Promise` 立即执行其方法体，打印出 `'promise1'`, 并将 `console.log('promise2')`放入微任务队列
8. 向下继续，执行 `console.log('script end')`，打印出 `'script end'`

至此，第一轮宏任务执行结束，打印结果为：
`'script start'`

`'async1 start'`

`'async2'`

`'promise1'`

`'script end'`

同时，微任务队列此时有以下两个微任务等待执行：

`console.log('async1 end');`

`console.log('promise2');`

执行微任务队列的全部微任务，打印出 `'async1 end'` 和 `'promise2'`，
同时宏任务队列此时有：`setTimeout`

###### 第二轮宏任务，宏任务队列目前只有 setTimeout

1.  执行打印出 `'setTimeout'`

全部任务执行完成，打印顺序为：

`'script start'`

`'async1 start'`

`'async2'`

`'promise1'`

`'script end'`

`'async1 end'`

`'promise2'`

`'setTimeout'`

### 七. 示例 2:

```js
console.log('start')
setTimeout(() => {
  console.log('children2')
  Promise.resolve().then(() => {
    console.log('children3')
  })
}, 0)
new Promise(function (resolve, reject) {
  console.log('children4')
  setTimeout(function () {
    console.log('children5')
    resolve('children6')
  }, 0)
}).then((res) => {
  console.log('children7')
  setTimeout(() => {
    console.log(res)
  }, 0)
})

// start
// children4
// children2
// children3
// children5
// children7
// children6
```

##### 开始一顿猛分析

###### 第一轮宏任务

1. 执行 `console.log('start')`，打印出 `'start'`
2. 遇到第一个 `setTimeout`，放到宏任务队列，等待执行
3. 遇到 `new Promise` 直接执行方法体内的内容，执行 `console.log('children4')`，打印出 `'children4'`,
   遇到第二个 `setTimeout`，放到宏任务队列，等待执行，
   注意：这时候还未调用 `resolve()`，所以 `.then()` 的内容无法放到微任务队列

###### 第二轮宏任务：执行第一个 `setTimeout`

内容为：

```js
console.log('children2')
Promise.resolve().then(() => {
  console.log('children3')
})
```

1. 执行 `console.log('children2')`，打印出 `'children2'`
2. 遇到 `Promise`，由于其方法体内没有内容，故没有代码执行，将 `.then` 的内容放到微任务队列里

此时此轮宏任务结束，开始执行微任务队列所有微任务；
微任务队列只有 `console.log('children3')`，执行并打印出 `'children3'`

###### 第三轮宏任务：执行第二个 setTimeout

内容为：

```js
console.log('children5')
resolve('children6')
```

1. 执行 `console.log('children5')`， 并打印出 `'children5'`
2. 并执行 `resolve('children6')`，将 `.then()` 后面的内容放到微任务队列中
3. 执行微任务队列的所有微任务，此时微任务队列只有一个微任务：

```js
console.log('children7')
setTimeout(() => {
  console.log(res)
}, 0)
```

1. 执行 `console.log('children7')`，打印出 `'children7'`
2. 遇到第三个 `setTimeout`，放到宏任务队列，等待执行

###### 第四轮宏任务：执行第三个 `setTimeout`

内容为：`console.log(res)`

1. 执行并打印出 `‘children6’`

全部任务执行完成，打印顺序为：

`'start'`

`'children4'`

`'children2'`

`'children3'`

`'children5'`

`'children7'`

`'children6'`

### 八. 示例 3:

```js
const p = function () {
  return new Promise((resolve, reject1) => {
    const p1 = new Promise((resolve, reject2) => {
      setTimeout(() => {
        resolve()
      }, 0)
      resolve(2)
    })
    p1.then((res) => {
      console.log(res)
    })
    console.log(3)
    resolve(4)
  })
}
p().then((res) => {
  console.log(res)
})
console.log('end')

// 3
// end
// 2
// 4
```

##### 开始一顿猛分析

###### 第一轮宏任务

1. 定义 `p` 函数
2. `p().then()`

   2.1 执行 `p()` 函数，函数体内，遇到 `new Promise` 直接执行其内容

   2.2 定义了 `p1` 的 `Promise`

   2.3 执行 `p1.then()`，遇到 `setTimeout`，将其加入宏任务队列，因为 `p1` 方法体内有调用 `resolve(2)`，所以将 `.then()` 内容 `console.log(res) // 2`
   放到微任务队列

   2.4 执行 `console.log(3)`，打印出 `"3"`

   2.5 执行 `resolve(4)`，将 `p().then()` 里的内容 `console.log(res)`，放到微任务队列

3. 执行 `console.log('end')`，打印出 `"end"`
4. 执行微任务队列的所有微任务，此时微任务队列有：

```js
console.log(res) // 2
console.log(res) // 4
```

陆续打印出：`'2'` 和 `'4'`

###### 第一轮宏任务

此时宏任务队列有：

```js
setTimeout(() => {
  resolve()
}, 0)
```

执行其内容 `resolve()`，无打印信息

全部任务执行完成，打印顺序为：

`'3'`

`'end'`

`'2'`

`'4'`
