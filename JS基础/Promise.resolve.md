# Promise.resolve()与new Promise(resolve=>resolve())的区别


参考：
- [Promise.resolve()与new Promise(r => r(v))](https://segmentfault.com/a/1190000020980101)
- [new Promise((resolve)=>{resolve()}) 与 Promise.resolve() 等价吗](https://segmentfault.com/q/1010000021636481)
- [探讨：当Async/Await的遇到了EventLoop](https://zhuanlan.zhihu.com/p/86993504)

### 返回值
如果参数是promise实例，前者返回新期约，后者原样返回：

```javascript

let p = new Promise((r)=>{r("123")})

//Promise.resolve将原封不动返回这个实例
p === Promise.resolve(p)  //true


//new Promise() 返回一个新期约
let p2 = new Promise((r)=>{
  r(p)
})

p2 !== p	//true
```

参数是一个thenable对象, 都返回新的解决期约：

```javascript
let thenable = {
  then: function(resolve,reject){
    resolve("123")
  }
}

let p1 = new Promise((r)=>{   //p1: Promise {<fulfilled>: "123"}
  r(thenable)
})

let p2 = Promise.resolve(thenable)  //p2:  Promise {<fulfilled>: "123"}

```


### then的执行时机
new Promise(r => r(v))的then()回调会被推迟两个时序（事件循环）：

```javascript
console.log("script start")

let v = new Promise((resolve)=>{
  console.log("v start")
  resolve("v resolve")
})

let p1 = new Promise((resolve)=>{
  console.log(1)
  resolve(v)
}).then((v)=>{
  console.log(v)
})

new Promise((resolve)=>{
  resolve()
}).then(()=>{
  console.log(2)
}).then(()=>{
  console.log(3)
}).then(()=>{
  console.log(4)
}).then(()=>{
  console.log(5)
})


console.log("script end")

//"script start"
//"v start"
//1
//"script end"
//2
//3
//v resolve   推迟了两个循环
//4
//5
```

而Promise.resolve(v)都then()则正常：

```javascript
console.log("script start")

let v = new Promise((resolve)=>{
  console.log("v start")
  resolve("v resolve")
})

Promise.resolve(v).then((v)=>{
  console.log(v)
})

new Promise((resolve)=>{
  resolve()
}).then(()=>{
  console.log(2)
}).then(()=>{
  console.log(3)
}).then(()=>{
  console.log(4)
}).then(()=>{
  console.log(5)
})


console.log("script end")

//"script start"
//"v start"
//"script end"
//"v resolve"
//2
//3
//4
//5
```
原因：
- 这是因为浏览器发现resolve的是另一个promise时（假设称原promise为promiseA，另一个promise为promiseB），会创建一个名为 PromiseResolveThenableJob的任务去处理promiseB，而这个任务是一个微任务
- 等到promiseB被resolved之后，会再生成去处理promiseA（resolvePromiseA）的微任务
- 执行完resolvePromiseA微任务，才会执行promiseA的then，因此推迟了两个事件循环
- 而Promise.resolve()的then正常执行，是因为参数是promiseB时，会不做处理返回promiseB本身

可以用伪代码将PromiseResolveThenableJob表示为：
```javascript
let promiseA = new Promise((resolve)=>{
  resolve(promiseB)
}).then(()=>{
  console.log("123")
})

//PromiseResolveThenableJob的伪代码，可以理解成
promiseB.then(()=>{
  resolvePromiseA,
  rejectPromiseA
})
```