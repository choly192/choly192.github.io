---
title: 实现一个 - Promise
categories: Promise
date: 2022-02-12 22:07:38
author: meditation
tags:
  - promise
  - 前端
  - 异步编程
cover: "/image/artical-images/promise.jpg"
---

## Promise 出现的原因?

---

> <b>Promise 的出现主要是为了解决以下两个问题</b>
>
> 1. 回调地狱：某个异步操作需要等待之前的操作完成之后再继续执行，当这样的需求多了之后，使得代码进入无尽的嵌套，代码可读性低且不好维护。
> 2. 异步和同步之间的联系：当一个同步操作需要等待多个异步操作的结果，这样会使得代码的逻辑变得相对来说比较复杂。

---

## Promise 要点

> <b>Promise 实现要点</b>
>
> 1. Promise 实际上就是一个类，由 ES6 提供的一个新的构造函数
> 2. Promise 有三种状态，成功态(`resolved`),失败态(`rejected`),等待态(`pending`)
> 3. Promise 状态一旦发生改变，就不能再发生改变（状态不可逆）
> 4. Promise 通过 new 关键字来创建实例，实例由一个`then`方法。其接收两个参数，一个成功回调，一个失败回调
> 5. Promise 实现链式调用，返回的不是`this`，而是一个新的 promise 实例
> 6. Promise 成功和失败的回调的返回值可以传递给下一次的`then`方法

---

## class 实现 Promise

```javascript
// Promise 的三种状态
const RESOLVED = "RESOLVED";
const REJECTED = "REJECTED";
const PENDING = "PENDING";

const resolvePromise = (promise2, x, resolve, reject) => {
  // 防止自己等待自己 一直处于循环等待
  if (promise2 === x)
    return reject(
      new TypeError("TypeError: Chaining cycle detected for promise #<Promise>")
    );

  let called;
  if(x !== null && (typeof x === 'object' || typeof x ===''function)){
    try{
      let then  = x.then // 去 x 的then方法

      // 通过判断 then 是不是函数来判断是否是promise
      if(typeof then === 'function'){
        then.call(x, y => {
          if(called) return;
          called = true;
          resolvePromise(promise2,y,resolve,reject);
        }, e => {
          if(called) return;
          called = true;
          reject(e);
        })
      } else {
        // 否则就认为  then 是一个普通对象 resolve回去即可
        resolve(x);
      }
    }catch(e){
      // 捕获到异常走 失败逻辑
      if(called) return;
      called = true;
      rejected(e);
    }
  } else {
    resolve(x); // x 为普通值
  }
};

// 创建一个Promise类
class Promise {
  constructor(executor) {
    this.status = PENDING;
    this.value = undefined; // user 自定义 成功的数据
    this.reason = undefined; // user 自定义 失败的原因

    // 分别创建一个队列来保存 成功、失败 的回调函数
    this.onfulfilledCallbacks = [];
    this.onrejectedCallbacks = [];

    // 成功
    const resolve = (value) => {
      // 状态不可逆 PENDING -> RESOLVED
      if (this.status === PENDING) {
        this.value = value;
        this.status = RESOLVED;
        this.onfulfilledCallbacks.forEach((fn) => fn());
      }
    };
    // 失败
    const rejected = (reason) => {
      if (this.status === PENDING) {
        this.reason = reason; // 失败的原因
        this.status = REJECTED; // PENDING -> REJECTED
        this.onrejectedCallbacks.forEach((fn) => fn());
      }
    };

    // 错误处理 抛出异常 throw new Error()
    try {
      executor(resolve, reject); // 立即执行
    } catch (e) {
      reject(e); // 抛出异常 直接失败
    }
  }

  // then 方法
  then(onfulfilled, onrejected) {

    // onfulfilled 是函数就执行，不是函数包一层函数并直接返回数据
    onfulfilled = typeof onfulfilled === 'function' ? onfulfilled : v => v;
    // onrejected 是函数就执行，不是函数包一层函数并直接抛出错误
    onrejected = typeof onrejected === 'function' ? onrejected : err => { throw err };

    // then 链式调用，返回的不是this，而是一个新的promise对象
    let promise2 = newPromise((resolve,reject) => {
      if (this.status === RESOLVED) {
        onfulfilled(this.value);
        // 加setTimeout是因为再当前执行上下文中，不能获取到promise2的值
        setTimeout(() => {
          try{
            let x = onfulfilled(this.value);
            resolvePromise(promise2,x, resolve,reject);
          }catch(e){
            reject(e);
          }
        });
      }

      if (this.status === REJECTED) {
        setTimeout(() => {
          try{
            let x = onrejected(this.reason);
            resolvePromise(promise2,x,resolve,reject);
          }catch(e){
            reject(e);
          }
        })
      }

      // 定时器中执行resolve时 状态仍然是 pending
      if (this.status === PENDING) {
        // promise实例可以 多次 then ，存放成功、失败回调
        this.onfulfilledCallbacks.push(() => {
          setTimeout(() => {
            try {
              let x = onfulfilled(this.value)
              resolvePromise(promise2, x, resolve, reject)
            } catch (e) {
              reject(e)
            }
          });
        });

        this.onrejectedCallbacks.push(() => {
          setTimeout(() => {
            try {
              let x = onrejected(this.reason)
              resolvePromise(promise2, x, resolve, reject)
            } catch (e) {
              reject(e)
            }
          });
        });
      }
    });
    return promise2; // 返回一个新的promise实例
  }
}
```
