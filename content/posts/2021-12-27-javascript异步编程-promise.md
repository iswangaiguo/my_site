---
title: JavaScript异步编程-Promise
date: 2021-12-27T08:58:31+00:00
categories:
  - JavaScript
  - 编程
tags:
  - Promise
---

## 一、回调模式

早期对于异步操作的处理主要是通过回调函数的方式进行，将函数传入异步操作，当该操作完成后该函数将会被调用，这里会存在一个问题，当嵌套多个回调函数的时候，会陷入回调地狱，代码的可读性变得极差。

```javascript
method1(function(err, result) {
    if (err) {
        throw err;
    }
    method2(function(err, result) {
        if (err) {
            throw err;
        }
        method3(function(err, result) {
            if (err) {
                throw err;
            }
            method4(function(err, result) {
                if (err) {
                    throw err;
                }
                method5(result);
            });
        });
    });
});
```

## 二、Promise

Promise代表一个承诺，表示将来在某时刻完成。Promise是ECMAscript 6提供的一个异步编程的API，可以通过它来传递异步消息。

### 2.1 Promise生命周期

当进行异步操作并且尚未完成时，Promise状态位挂起态（Pedding），当操作完成时，Promise会根据操作的结果转为一下两种状态：

* fulfilled: 意味着操作成功完成。
* rejected: 意味着操作失败

### 2.2 基本使用

使用`new`创建Promise，该类接收一个函数，该函数的两个参数分别是`resolve`和`reject`，其中在异步操作成功完成的时候使用`resolve`函数传递成功参数，在异步操作异常的时候使用`reject`函数传递失败参数。

```javascript
// 创建Promise
const promise = new Promise(function(resolve, reject) {
  // 耗时操作
  setTimeout(() => {
    resolve("success done");
  }, 2000);
});

// 接收处理结果
promise.then(function(content) {
  // 完成
}, function(err) {
  // 拒绝
});

promise.then(content => {
  // 完成
});

promise.then(null, function(content) {
  // 拒绝
});

// 等同于
promise.catch(function(err) {
  // 拒绝
});
```

如果在Promise内部抛出异常，Promise的拒绝函数将会执行。

```javascript
let promise = new Promise(function(resolve, reject) {
  throw new Error("Explosion!");
});

promise.catch(function(error) {
  console.log(error.message); // "Explosion!"
});
```

### 2.3 链式调用

对`then`或`catch`的调用，实际上是返回了另一个Promise，当前Promise完成或拒绝后，下一个Promise才会执行。处理函数的返回值将会传递给下个处理函数，拒绝函数也是同样。

```javascript
let p1 = new Promise(function(resolve, reject) {
  resolve(41);
});

p1.then(function(value) {
  console.log(value); // 41
}).then(function() {
  console.log("finish"); // finish
});

p1.then(function(value) {
  return value + 1;
}).then(function(value) {
  console.log(value); // 42
});
```

### Promise.all()

Promise接收多个Promise作为参数，仅当所有Promise都完成时，所返回的Promise才算完成。

```javascript
let p1 = new Promise(function(resolve, reject) {
  resolve(42);
});

let p2 = new Promise(function(resolve, reject) {
  resolve(43);
});

let p3 = new Promise(function(resolve, reject) {
  resolve(44);
});

let p4 = Promise.all([p1, p2, p3]);

p4.then(function(value) {
  console.log(Array.isArray(value)); // true
  console.log(value[0]); // 42
  console.log(value[1]); // 43
  console.log(value[2]); // 44
});
```

当任一Promise被拒绝时，那么所返回的Promise会被立刻拒绝，而不会等待其他的Promise。

```javascript
let p1 = new Promise(function(resolve, reject) {
  resolve(42);
});
let p2 = new Promise(function(resolve, reject) {
  reject(43);
});
let p3 = new Promise(function(resolve, reject) {
  resolve(44);
});
let p4 = Promise.all([p1, p2, p3]);
p4.catch(function(value) {
  console.log(Array.isArray(value)); // false
  console.log(value); // 43
});
```

### Promise.race()

`Promise.race`类似`Promise.all`，不同的是任意有个Promise被解决或被拒绝，`Promise.race`就会返回该Promise。

```javascript
let p1 = Promise.resolve(42);

let p2 = new Promise(function(resolve, reject) {
  resolve(43);
});

let p3 = new Promise(function(resolve, reject) {
  resolve(44);
});

let p4 = Promise.race([p1, p2, p3]);

p4.then(function(value) {
  console.log(value); // 42
});
