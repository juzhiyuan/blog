---
title: 在 Map 遍历中使用 async 函数
---

有时需要使用 `Sleep` 函数阻塞代码一段时间，该函数常规实现与调用方式如下：

```js
// Sleep Function
const sleep = ms => new Promise(resolve => setTimeout(resolve, ms))

// Usage
(async () => {
   await sleep(3000)
})
```

但在 `Array.prototype.map` 中使用时，却有着错误的表现，具体如下：

```js
// code snippet 1
[1, 2].map(async num => {
  console.log('Loop Start')
  console.log(num)
  await sleep(3000)
  console.log('Loop End')
})

// expected output
//    Loop Start
//    1
//        Wait for about 3s
//    Loop End
//    Loop Start
//    2
//        Wait for about 3s
//    Loop End

// Actual output
//    Loop Start
//    1
//    Loop Start
//    2
//         Wait for about 3s
//    Loop End
//    Loop End
```

我们期望的是，在每一次循环时，暂停约 3s 钟时间后再继续执行；但实际表现是：每当执行

```js
await sleep(3000)
```

时，没有等待结果返回便进入到了下一次循环。之所以产生错误的表现，原因是：

当 async 函数被执行时，将立即返回 pending 状态的Promise（ Promise 是 Truthy 的）！因此，在 map 循环时，不会等待 await 操作完成，而是直接进入下一次循环，所以应当配合 for 循环使用 async。

验证一下，我们将 code snippet 1 做一下修改：

```js
// code snippet 2
const sleep = ms => new Promise(resolve => {
  console.log('sleep')
  setTimeout(() => {
    console.log('resolve')
    resolve()
  }, ms)
})

const mapResult = [1, 2].map(async num => {
  console.log('Loop Start')
  console.log(num)
  await sleep(3000)
  console.log('Loop End')
})

console.log('mapResult', mapResult)

// Actual output
//    Loop Start
//    1
//    sleep
//    Loop Start
//    2
//    sleep
//    mapResult [ Promise { <pending> }, Promise { <pending> } ]
//    resolve
//    Loop End
//    resolve
//    Loop End
```

可以看到，使用了 async 函数后的 map 方法，其返回值为

```js
// mapResult [ Promise { <pending> }, Promise { <pending> } ]
```

即包含了多个状态为 pending 的 Promise 的数组！

另外，如果只是循环而不需要操作 map 返回的数组，那么也应当使用 for 循环。

对于 forEach 而言，参考 MDN 中它的 Polyfill 可知，若回调函数为异步操作，它将会表现出并发的情况，因为它不支持等待异步操作完成后再进入下一次循环。

感谢 @杨宁 提供的使用 Array.prototype.reduce 解决的[方法](https://codepen.io/juzhiyuan/pen/jONwyeq)：

```js
// https://codepen.io/juzhiyuan/pen/jONwyeq

const sleep = wait => new Promise(resolve => setTimeout(resolve, wait));

const __main = async function() {
  // 你的需求其实是在一组 task 中，循环执行，每个都 sleep，并返回结果
  const tasks = [1, 2, 3];

  let results = await tasks.reduce(async (previousValue, currentValue) => {
    // 这里是关键，需要 await 前一个 task 执行的结果
    // 实际上每个 reduce 每个循环执行都相当于 new Promise
    // 但第二次执行可以拿到上一次执行的结果，也就是上一个 Promise
    // 每次执行只要 await 上一个 Promise，即可实现依次执行
    let results = await previousValue
    console.log(`task ${currentValue} start`)
    await sleep(1000 * currentValue);
    console.log(`${currentValue}`);
    console.log(`task ${currentValue} end`);
    results.push(currentValue)
    return results
  }, []);

  console.log(results);
}

__main()

// Actual output:
//    task 1 start
//    1
//    task 1 end
//    task 2 start
//    2
//    task 2 end
//    task 3 start
//    3
//    task 3 end
//    [1, 2, 3]
```

![https://blog-static-cdn.shaoyaoju.org/2019/08/WechatIMG3.png](https://blog-static-cdn.shaoyaoju.org/2019/08/WechatIMG3.png)

## 参考

1. http://devcheater.com/
2. https://codeburst.io/javascript-async-await-with-foreach-b6ba62bbf404
3. https://zellwk.com/blog/async-await-in-loops/
