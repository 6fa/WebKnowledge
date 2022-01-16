# Koa2 笔记

参考：
- [Node.js 蚕食计划（六）—— MongoDB + Koa 入门](https://www.cnblogs.com/wisewrong/p/13280266.html)
- [Koa中文网](https://www.koajs.com.cn/)
- [可能是目前最全的koa源码解析指南](https://developers.weixin.qq.com/community/develop/article/doc/0000e4c9290bc069f3380e7645b813)
- [基于Koa2搭建Node.js实战项目教程](https://camp.qianduan.group/koa2)

本文目录：
- [1.Koa的使用](#1)
  - [1.1介绍](#11)
  - [1.2context](#12)
  - [1.3应用方法](#13)
- [2.Koa源码解析](#2)

<span id="1"></span>
## Koa的使用

<span id="11"></span>
### 1.1介绍
Koa是一个精简的node框架，所有功能都是通过插件的方式实现。主要做的事：
- 将node的request和response对象封装成 context 对象
- 基于async/await（generator）的中间件洋葱模型
- 源码中仅4个文件：

```
── lib
   ├── application.js	//入口文件
   ├── context.js
   ├── request.js
   └── response.js

//4个文件对应koa的4个对象

── lib
   ├── new Koa()  ---> ctx.app
   ├── ctx
   ├── ctx.req  ---> ctx.request
   └── ctx.res  ---> ctx.response
```

安装：
```
npm install koa

//会自动生成package-lock.json文件和node-modules
//koa源码位于：node-modules --> koa --> lib

//一般npm init后再安装koa
```

一个小例子：
```javascript
//my-koa-app.js
const Koa = require('koa');
const app = new Koa();

app.use(async ctx => {
  console.log(ctx)
  ctx.body = 'Hello World';
});

app.listen(3000);


//也是用node运行
node my-koa-app.js
```

```javascript
//ctx
{
  request: {
    method: 'GET',
    url: '/',
    header: {
      host: '127.0.0.1:3000',
      connection: 'keep-alive',
      'cache-control': 'max-age=0',
      'sec-ch-ua': '"Chromium";v="94", "Google Chrome";v="94", ";Not A Brand";v="99"',
      'sec-ch-ua-mobile': '?0',
      'sec-ch-ua-platform': '"Windows"',
      'upgrade-insecure-requests': '1',
      'user-agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/94.0.4606.81 Safari/537.36',
      accept: 'text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.9',
      'sec-fetch-site': 'none',
      'sec-fetch-mode': 'navigate',
      'sec-fetch-user': '?1',
      'sec-fetch-dest': 'document',
      'accept-encoding': 'gzip, deflate, br',
      'accept-language': 'zh-CN,zh;q=0.9,en;q=0.8'
    }
  },
  response: {
    status: 404,
    message: 'Not Found',
    header: [Object: null prototype] {}
  },
  app: { subdomainOffset: 2, proxy: false, env: 'development' },
  originalUrl: '/',
  req: '<original node req>',
  res: '<original node res>',
  socket: '<original node socket>'
}
```


<span id="12"></span>
### 1.2Context
koa的Context将node的request和response封装在单独的一个对象里。
Context会在每次request中被创建，被传入中间件（每次请求都会调用注册的async函数）：

```javascript
app.use(async ctx => {
  ctx; // is the Context
  ctx.request; // is a koa Request
  ctx.response; // is a koa Response
  ctx.req;   // original node req  （node的request对象）
  ctx.res;	// <original node res>	（node的response对象）
});
```

#### 1.2.1ctx.req 和 ctx.res
为node中的requst和response对象，但是不支持直接调用底层 res 进行响应处理，避免使用下面的属性：
- res.statusCode
- res.writeHead()
- res.write()
- res.end()

#### 1.2.2ctx.request 和 ctx.response
koad的request和response对象。
为了方便访问，ctx.request和ctx.response上有些方法可以直接通过ctx访问。

比如**ctx.body**：

```javascript
 console.log(ctx.body === ctx.response.body) //true
 console.log(ctx.body === ctx.res.body) //true

//不能直接用node的res响应
ctx.res.body = "learning Koa"

//这个可以
ctx.response.body = "learning Koa"

//或者直接通过ctx
ctx.body = "learning Koa"
```

**ctx.response访问器的别名：**
- **ctx.body**
  - 设置响应体为如下值:
  - string
    - Content-Type将默认为'text/html'或'text/plain'
    - charset默认为'utf-8'
  - Buffer
    - Content-Type 默认为 application/octet-stream
  - Stream
    - Content-Type 默认为 application/octet-stream
  - Object || Array （ json-stringified）
    - Content-Type默认为application/json。 这包括普通对象{ foo: 'bar' }和数组['foo', 'bar']
  - null
- ctx.body=
- **ctx.status**
- ctx.status=
- **ctx.message**：响应状态消息。默认情况下, response.message关联response.status
- ctx.message=
- ctx.length
- ctx.length=
- ctx.type
- ctx.type=
- **ctx.lastModified=**
- **ctx.etag=**
- ctx.headerSent
- ctx.redirect()
- ctx.attachment()
- ctx.set()：设置响应头中的字段
- ctx.append()：向响应头添加额外字段
- ctx.remove()

此外response对象还有下列属性或方法：
- **ctx.response.header**
- **ctx.response.get()**：获取响应头中字段值
- ctx.response.is()：检查响应所包含的 "Content-Type" 是否为给定的 type 值
- **ctx.response.redirect()**: 重定向


**ctx.request访问器的别名：**
- **ctx.header**：请求头对象
- ctx.headers：等价于ctx.header
- **ctx.method**
- ctx.method=
- **ctx.url**：获取请求url的地址
- ctx.url=
- **ctx.originalUrl**：获取请求原始地址
- **ctx.origin**：获取url原始地址，包含protocal、host
- **ctx.href**：包含完整的请求url，包含protocal、host、url
- **ctx.path**：请求路径名
- ctx.path=
- **ctx.query**：返回格式化好的参数对象，如{a:1,b:2}
- ctx.query=
- **ctx.querystring**：返回参数字符串，如a=1&b=2
- ctx.querystring=
- ctx.host
- ctx.hostname
- ctx.fresh
- ctx.stale
- ctx.socket
- ctx.protocol
- ctx.secure
- ctx.ip
- ctx.ips
- ctx.subdomains
- **ctx.is(types)**：检查请求所包含的 "Content-Type" 是否为给定的 type 值
- **ctx.accepts(typs)**：指定接受的类型，如ctx.accepts('text/html');
- ctx.acceptsEncodings()
- ctx.acceptsCharsets()
- ctx.acceptsLanguages()
- ctx.get(field)：返回请求头中的字段值

此外request对象还有：
- ctx.request.search
- ctx.request.search = 
- **ctx.request.type**：获取请求 Content-Type，不包含像 "charset" 这样的参数
- **ctx.request.charset**：比如'utf-8'

#### 1.2.3ctx.state
koa推荐的命名空间，用于保存中间件的状态数据（如果使用 webpack 打包的话，可以使用中间件，将加载资源的方法作为 ctx.state 的属性传入到 view 层，方便获取资源路径）：

```javascript
ctx.state.user = await User.find(id);
```

#### 1.2.4ctx.cookies.get(name,[options])、ctx.cookies.set(name,value,[options])
koa使用了express的 cookies 模块，options只是简单传值。
setCookie中，options可选参数：
- maxAge： 一个数字，表示 Date.now()到期的毫秒数
- signed： 是否要做签名
- expires： cookie有效期
- pathcookie： 的路径，默认为 /'
- domain： cookie 的域
- secure： false 表示 cookie 通过 HTTP 协议发送，true 表示 cookie 通过 HTTPS 发送。
- httpOnly： true 表示 cookie 只能通过 HTTP 协议发送
- overwrite： 一个布尔值，表示是否覆盖以前设置的同名的Cookie（默认为false）。 如果为true，在设置此cookie时，将在同一请求中使用相同名称（不管路径或域）设置的所有Cookie将从Set-Cookie头部中过滤掉。

#### 1.2.5ctx.throw([status], [msg], [properties])
抛出错误，默认status为500

```javascript
ctx.throw(400);
ctx.throw(400, 'name required');
ctx.throw(400, 'name required', { user: user });
```

#### 1.2.6ctx.app
应用实例的引用

```javascript
const Koa = require('koa')
const app = new Koa()

app.use(async (ctx)=>{
  console.log(ctx.app === app) //true
})

app.listen(3000,()=>{
  console.log("serving at 127.0.0.1:3000")
})
```

<span id="13"></span>
### 1.3Context

