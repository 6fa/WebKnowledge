# Koa2 笔记

参考：
- [Node.js 蚕食计划（六）—— MongoDB + Koa 入门](https://www.cnblogs.com/wisewrong/p/13280266.html)
- [Koa中文网](https://www.koajs.com.cn/)
- [可能是目前最全的koa源码解析指南](https://developers.weixin.qq.com/community/develop/article/doc/0000e4c9290bc069f3380e7645b813)
- [基于Koa2搭建Node.js实战项目教程](https://camp.qianduan.group/koa2)

本文目录：
- [1.Koa的使用](#1)
  - [1.1 介绍](#11)
  - [1.2 context](#12)
  - [1.3 应用方法](#13)
  - [1.4 中间件](#14)
  - [1.5 路由 koa-router ](#15)
  - [1.6 提取中间件&路由](#16)
  - [1.7 项目结构](#17)
- [2.Koa源码解析](#2)
  - [2.1 application.js](#21)
  - [2.2 context.js](#22)
  - [2.3 request.js](#23)
  - [2.4 response.js](#24)
  - [2.5 洋葱模型的实现](#25)

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
### 1.3应用方法


#### 1.3.1app.listen( )
app.listen( )是对http.createServer的封装：

```javascript
const Koa = require('koa');
const app = new Koa();
app.listen(3000);


//实际是下面代码的语法糖
const http = require('http');
const Koa = require('koa');
const app = new Koa();
http.createServer(app.callback()).listen(3000)

```

#### 1.3.2app.callback( )
app.calbback返回一个适合 http.createServer() 方法的回调函数，用于处理请求。

里面包含了中间件的合并、上下文的处理、对res的处理等。

#### 1.3.3app.use(fn)
app.use(fn)为应用添加中间件，将中间件放入一个缓存队列中，koa会通过插件（koa-compose）进行递归组合、调用。

<span id='14'></span>
### 1.4中间件（middleware）


#### 1.4.1概念
中间件是指封装了处理请求操作的方法，所有请求处理都在中间件内部完成。

使用use方法来注册一个中间件，中间件第一个参数是封装了原生req、res的ctx对象，第二个参数是next方法。next方法配合async/await语句，可以很轻松地控制处理流程：
- 使用await next()等待将控制器交给下一个中间件，等待执行完毕再回到本中间件。这样实现流程的层层展开和层层闭合，所以中间件模型有**洋葱模型**之称。

而原生Node.js中的请求处理完全放在http.createServer的一个回调里，要控制操作流程相对困难。

#### 1.4.2注册中间件
通过app.use( ) 注册中间件，**每次收到请求都会调用这个中间件**，且会传入ctx和next两个参数。
- ctx即Context对象
- next( )的作用是将处理的控制权交给下一个中间件，next( )后面的代码会等下一中间件及后面的中间件执行完才执行

#### 1.4.3执行流程
```javascript
const Koa = require('koa')
const app = new Koa()


// 记录执行的时间
app.use(async (ctx, next) => {
  let stime = new Date().getTime()
  await next()
  let etime = new Date().getTime()
  ctx.response.type = 'text/html'
  ctx.response.body = '<h1>Hello World</h1>'
  console.log(`请求地址: ${ctx.path}，响应时间：${etime - stime}ms`)
});

app.use(async (ctx, next) => {
  console.log('中间件1 doSoming')
  await next();
  console.log('中间件1 end')
})

app.use(async (ctx, next) => {
  console.log('中间件2 doSoming')
  await next();
  console.log('中间件2 end')
})

app.use(async (ctx, next) => {
  console.log('中间件3 doSoming')
  // await next();                     //最后一个可以不用调用next
  console.log('中间件3 end')
})

app.listen(3000, () => {
  console.log('server is running at http://localhost:3000')
})
```

结果：
```
中间件1 doSoming
中间件2 doSoming
中间件3 doSoming
中间件3 end
中间件2 end
中间件1 end
请求地址: /，响应时间：3ms
```

通过async、await实现流程的**层层展开**和**层层闭合**，所以中间件模型有洋葱模型之称。

需要注意的是，如果没有调用next( )，则它后面的中间件是不会执行的。

<span id='15'></span>
### 1.5路由（koa-router）
路由用来描述url与处理函数之间的对应关系。

#### 1.5.1自实现路由
如果不借助koa-router，自己实现路由的话：

```javascript
const Koa = require('koa')
const app = new Koa()

app.use(async(ctx, next)=>{
  if(ctx.request.path === '/'){
    ctx.response.body = "<h1>index page</h1>"
  }else {
    await next()
  }
})
app.use(async(ctx, next)=>{
  if(ctx.request.path === '/home'){
    ctx.response.body = "<h1>home page</h1>"
  }else {
    await next()
  }
})

app.listen(3000, () => {
  console.log('server is running at http://localhost:3000')
})
```

虽然写法简单，但是处理多个路由将会显得很繁琐。借助koa-router库可以更简单配置路由。

#### 1.5.2用法
例子：

```javascript
const Koa = require('koa')
const Router = require('koa-router')
const app = new Koa()
const router = new Router()

//添加路由
router.get('/', async(ctx,next)=>{
  ctx.response.body = `<h1>index page</h1>`
})

router.get('/home', async(ctx,next)=>{
  ctx.response.body = `<h1>home page</h1>`
})

//调用路由中间件
app.use(router.routes())
	.use(router.allowedMethods())		//根据ctx.status设置response响应头

app.listen(3000, () => {
  console.log('server is running at http://localhost:3000')
})
```

例子中除了调用router.routes外，还调用了router.allowedMethods，它的作用是：
当所有路由中间件执行完成之后，若ctx.status为空或者404的时候,**用来丰富response对象的header头**。

#### 1.5.3router.use( )
按定义的顺序调用中间件

```javascript

//添加路由
router.get('/', async(ctx,next)=>{
  ctx.response.body = `<h1>index page</h1>`
  //注意需要调用next方法
  next()									
})

//无路由中间件
router.use(async(ctx,next)=>{
  //...
})

//只有路由匹配时才运行中间件
router.use('/home', async(ctx,next)=>{
  //...
})
//匹配多路由
router.use(['/home','/cart'], async(ctx,next)=>{
  //...
})


//调用路由中间件
app.use(router.routes()).use(router.allowedMethods())

app.listen(3000, () => {
  console.log('server is running at http://localhost:3000')
})
```

#### 1.5.4多中间件

```javascript
router.get(
  '/users/:id',
  function (ctx, next) {
    return User.findOne(ctx.params.id).then(function(user) {
      // 首先读取用户的信息，异步操作
      ctx.user = user;
      next();
    });
  },
  function (ctx) {
    console.log(ctx.user);
    // 在这个中间件中再对用户信息做一些处理
    // => { id: 17, name: "Alex" }
  }
);
```

#### 1.5.5嵌套路由

用法：fatherRouter.use( path, childRouter.routes(), childRouter.allowedMethods() )

```javascript
var forums = new Router();
var posts = new Router();

posts.get('/', function (ctx, next) {...});
posts.get('/:pid', function (ctx, next) {...});
forums.use('/forums/:fid/posts', posts.routes(), posts.allowedMethods());

// 可以匹配到的路由为 "/forums/123/posts" 或者 "/forums/123/posts/123"
app.use(forums.routes());
```

#### 1.5.6路由前缀
路由前缀不能是动态的：

```javascript
var router = new Router({
  prefix: '/users'
});

router.get('/', ...); // 匹配路由 "/users" 
router.get('/:id', ...); // 匹配路由 "/users/:id"
```

#### 1.5.7URL参数
koa-router支持将动态路由参数添加到**ctx.params**：

```javascript
router.get('/:category/:title', function (ctx, next) {
  console.log(ctx.params);
  // => { category: 'programming', title: 'how-to-node' } 
});
```

#### 1.5.8get请求时的传参
处理get请求时，传参方式有两种：
- 使用查询字符串，参数放在url后面：如http://xxx/index?id=123&name=myname
  - 可以使用**ctx.request.query**或者**ctx.request.queryString**获取
- 参数放在url中间，即动态路由

```javascript
// http://xxx/programming/how-to-noode

router.get('/:category/:title', function (ctx, next) {
  console.log(ctx.params);
  // => { category: 'programming', title: 'how-to-node' } 
});
```

#### 1.5.9处理post请求
将请求参数放在body中时，需要用到post请求。

post请求通常通过表单、JSON形式发送，但是无论是Node还是koa，都没有提供解析post请求参数的功能。需要通过自己解析上下文 context 中的原生 node.js 请求对象req，将 POST 表单数据解析成 querystring（例如：a=1&b=2&c=3），再将 querystring 解析成 JSON 格式（例如：{"a":"1", "b":"2", "c":"3"}）。

需要额外使用koa-bodyparse、koa-body等插件。


<span id='16'></span>
### 1.6提取中间件 & 路由

把中间件、路由提取到另外的文件，可以更方便管理和维护。

#### 1.6.1中间件的分离

```javascript
//新建middleware文件夹存放中间件
// 新建midlleware/index.js，用于集中管理中间件

const mw1 = require('./mw1') //引入所有中间件
const mw2 = require('./mw2')
const mw3 = require('./mw3')

module.exports = (app)=> { //传入app
  app.use(mw1())
  app.use(mw2())
  app.use(mw3())
}
```
```javascript
// middleware/mw1.js
module.exports = ()=>{
  return async(ctx,next)=>{
    //...
  }
}
```

#### 1.6.2路由的分离

```javascript
//新建router.js集中管理路由
const Router = require('koa-router')
const router = new Router()

//controller文件夹存放路由的回调
const HomeController = require('./controller/home')

module.exports = (app)=>{
  router.get('/', HomeController.index)
  router.get('/home', HomeController.home)
  
  app.use(router.routes()).use(router.allowedMethods())
}
```

controller存放路由回调：

```javascript
// controller/home.js
module.exports = {
  index: async(ctx, next) => {
    ctx.response.body = `<h1>index page</h1>`
  },
  home: async(ctx, next) => {
    ctx.response.body = '<h1>HOME page</h1>'
  }
}
```

#### 1.6.3修改app.js

```javascript
//app.js
const Koa = require('koa')
const app = new Koa()
const bodyparser = require('koa-parser')

const router = require('./router')  //引入路由文件
const middleware = require('./middleware') //引入中间件文件

app.use(bodyparser)


middleware(app)
router(app)

app.listen(3000, () => {
  console.log('server is running at http://localhost:3000')
})
```

<span id='17'></span>
### 1.7项目结构

推荐项目结构：

```javascript
├─ controller/          // 用于解析用户的输入，处理后返回相应的结果
├─ service/             // 用于编写业务逻辑层，比如连接数据库，调用第三方接口等
├─ errorPage/           // http 请求错误时候，对应的错误响应页面
├─ logs/                // 项目运用中产生的日志数据
├─ middleware/          // 中间件集中地，用于编写中间件，并集中调用
│  ├─ mi-http-error/
│  ├─ mi-log/
│  ├─ mi-send/
│  └── index.js
├─ public/              // 用于放置静态资源
├─ views/               // 用于放置模板文件，返回客户端的视图层
├─ router.js            // 配置 URL 路由规则
└─ app.js               // 用于自定义启动时的初始化工作，比如启动 https，调用中间件，启动路由等

```

service用于放置业务逻辑，比如注册：

```javascript
// service/home.js
module.exports = {
    register: async(name, pwd) => {
      let data 
      if (name == 'ikcamp' && pwd == '123456') {
        data = `Hello， ${name}！`
      } else {
        data = '账号信息错误'
      }
      return data
    }
  }
```

```javascript
//controller/home.js

// 引入 service 文件
const HomeService = require('../service/home')
module.exports = {
  // ……
  //register 方法 
  register: async(ctx, next) => {
    let {
      name,
      password
    } = ctx.request.body
    let data = await HomeService.register(name, password)  //注意这里
    ctx.response.body = data
  }
}
```

<span id='2'></span>
## 2.Koa源码解析
koa源码供共4个文件：

```
── lib
   ├── application.js
   ├── context.js
   ├── request.js
   └── response.js
```

<span id="21"></span>
### 2.1 application.js
application.js是koa的入口文件，暴露出应用的类：

```javascript
//依赖模块，下面只列出核心需要关注的内容
const compose = require('koa-compose');
const context = require('./context');
const request = require('./request');
const response = require('./response');
const Emitter = require('events');
const convert = require('koa-convert');

// koa应用继承了node的events模块，
module.exports = class Application extends Emitter {
  //--------------------------------1
  constructor(options) {
    super();
    options = options || {};
    //...
    this.env = options.env || process.env.NODE_ENV || 'development';
		//...
    //middleware数组：存放所有通过use函数引入的中间件
    this.middleware = [];
    
    //通过Object.create创建的context、request、response对象
    this.context = Object.create(context);
    this.request = Object.create(request);
    this.response = Object.create(response);
    //...
  }
  
  //--------------------------------2
  //调用listen方法时，通过http.createServer创建服务器
  listen(...args) {
    debug('listen');
    // this.callback()是重点，返回的是对应http.createServer的参数函数
    const server = http.createServer(this.callback());
    return server.listen(...args);
  }
  
  //--------------------------------3
  //返回一个类似(req, res) => {}的函数
  callback() {
    //compose即koa-compose插件，它的作用是重新组合中间件
    const fn = compose(this.middleware);

  	//...

    const handleRequest = (req, res) => {
      //调用createContext函数，传入node的req、res，生成ctx对象
      const ctx = this.createContext(req, res);
      //调用handleRequest函数（app实例上的，注意区分本函数handleRequest）
      //传入组合过的 中间件数组 处理请求
      return this.handleRequest(ctx, fn);
    };

    return handleRequest;
  }
  
  //--------------------------------4
  // createContext函数：生成ctx对象的函数
  createContext(req, res) {
    const context = Object.create(this.context);
    const request = context.request = Object.create(this.request);
    const response = context.response = Object.create(this.response);
    context.app = request.app = response.app = this;
    context.req = request.req = response.req = req;
    context.res = request.res = response.res = res;
    request.ctx = response.ctx = context;
    request.response = response;
    response.request = request;
    context.originalUrl = request.originalUrl = req.url;
    context.state = {};
    return context;
  }
  //--------------------------------5
  //handlerRequest函数：处理请求。fnMiddleware是compose过的中间件数组
  handleRequest(ctx, fnMiddleware) {
    const res = ctx.res;
    res.statusCode = 404;
    const onerror = err => ctx.onerror(err);
    
    //respond()不是实例方法
    //它的作用是根据body类型做一些处理，最后才res.end(body)
    const handleResponse = () => respond(ctx);
    onFinished(res, onerror);
    return fnMiddleware(ctx).then(handleResponse).catch(onerror);
  }
	//--------------------------------6
  //use函数
  use(fn) {
    if (typeof fn !== 'function') throw new TypeError('middleware must be a function!');
    if (isGeneratorFunction(fn)) {
      //兼容koa1的generator写法，使用convert来转换
      fn = convert(fn);
    }
    debug('use %s', fn._name || fn.name || '-');
    //向middleware中添加中间件
    this.middleware.push(fn);
    return this;
  }
  
  //......
}

//--------------------------------7
//主要是判断你之前中间件写的 body 的类型，做一些处理，然后才使用res.end(body)
function respond(ctx) {
  // allow bypassing koa
  if (false === ctx.respond) return;

  if (!ctx.writable) return;

  const res = ctx.res;
  let body = ctx.body;
  const code = ctx.status;

  // ignore body
  if (statuses.empty[code]) {
    // strip headers
    ctx.body = null;
    return res.end();
  }

  if ('HEAD' === ctx.method) {
    //...
    return res.end();
  }

  // status body
  if (null == body) {
    //...
    return res.end(body);
  }

  // responses
  if (Buffer.isBuffer(body)) return res.end(body);
  if ('string' === typeof body) return res.end(body);
  if (body instanceof Stream) return body.pipe(res);

  // body: json
  body = JSON.stringify(body);
  if (!res.headersSent) {
    ctx.length = Buffer.byteLength(body);
  }
  res.end(body);
}
```

application.js主要处理了以下事情：

- 初始化中间件middleware数组
- 引入了context.js、request.js、response.js，初始化context、request、response对象
- 暴露了一些公共API，如listen、use方法
  - use方法会将中间件函数收集到middleware数组中
  - listen方法会通过http.createServer创建服务器，并且传入参数函数callback()
  - callback会进行中间件的组合合并（通过koa-compose）、生成中间件函数的ctx对象、处理请求，最后返回的是符合http.createServer参数的函数

```javascript
//koa的listen方法是对http.createServer的封装
//约等于:
http.createServer(app.callback()).listen(...)
```

<span id="22"></span>
### 2.2 context.js
context.js的主要功能：
- 错误事件处理
- 代理response、request对象的一些属性和方法

```javascript
const util = require('util');
const createError = require('http-errors');
const delegate = require('delegates');
const Cookies = require('cookies');
//...


const proto = module.exports = {
  //...
  onerror(err) {
    if (null == err) return;
		//...
    // app触发error事件
    this.app.emit('error', err, this);
  }
  
  get cookies() {
    //...
  },
  set cookies(_cookies) {
    //...
  }
}

//context对象代理response的一些方法、属性
delegate(proto, 'response')
  .method('attachment')
  .method('redirect')
  //...

//context对象代理request的一些方法、属性
delegate(proto, 'request')
  .method('acceptsLanguages')
  .method('acceptsEncodings')
	//...
```

<span id='23'></span>
### 2.3 request.js
基于node原生req对象，以getter、setter形式封装了一些属性和方法:

```javascript
module.exports = {
  
  // 在application.js的createContext函数中，
  //会把node原生的req作为request对象(即request.js封装的对象)的属性
  // request对象会基于req封装很多便利的属性和方法
  get header() {
    return this.req.headers;
  },

  set header(val) {
    this.req.headers = val;
  },

  // 省略了大量类似的工具属性和方法
};
```

<span id='24'></span>
### 2.4 response.js
和request.js差不多, 且返回的body支持多种格式：

```javascript
module.exports = {
  // 在application.js的createContext函数中，
  //会把node原生的res作为response对象（即response.js封装的对象）的属性
  // response对象与request对象类似，基于res封装了一系列便利的属性和方法
  get body() {
    return this._body;
  },

  set body(val) {
    // 支持string
    if ('string' == typeof val) {
    }

    // 支持buffer
    if (Buffer.isBuffer(val)) {
    }

    // 支持stream
    if ('function' == typeof val.pipe) {
    }

    // 支持json
    this.remove('Content-Length');
    this.type = 'json';
  },
 }
```

<span id='25'></span>
### 2.5洋葱模型的实现
调用app.listen()时，会在内部调用http.createServer()，并把this.callback()作为参数传进去。
callback的作用：
- 使用koa-compose处理所有中间件，它是实现洋葱模型的核心
- 基于node的req、res封装成ctx对象
- 处理响应内容

```javascript
callback() {
    // compose处理所有中间件函数。洋葱模型实现核心
    const fn = compose(this.middleware);

    // 每次请求执行函数(req, res) => {}
    const handleRequest = (req, res) => {
      // 基于req和res封装ctx
      const ctx = this.createContext(req, res);
      // 调用handleRequest处理请求
      return this.handleRequest(ctx, fn);
    };

    return handleRequest;
  }

//handleRequest接受ctx和compose后的fn
 handleRequest(ctx, fnMiddleware) {
    const res = ctx.res;
    res.statusCode = 404;

    // 调用context.js的onerror函数
    const onerror = err => ctx.onerror(err);

    // 处理响应内容
    const handleResponse = () => respond(ctx);

    // 确保一个流在关闭、完成和报错时都会执行响应的回调函数
    onFinished(res, onerror);

    // 中间件执行、统一错误处理机制的关键
    return fnMiddleware(ctx).then(handleResponse).catch(onerror);
  }
```

koa-compose的精简代码：

```javascript
module.exports = compose

function compose(middleware) {  //middleware是中间件数组
    return function (context, next) {  //返回的这个函数会在每次请求时都调用
      																	//因为每次请求会调用callback
       																	//其实就是将所有中间间组合成一个函数
      
      let index = -1
      return dispatch(0)
      function dispatch (i) {
        if (i <= index) return Promise.reject(new Error('next() called multiple times'))
        index = i
        let fn = middleware[i]
        if (i === middleware.length) fn = next
        if (!fn) return Promise.resolve()
        try {
          //中间件调用next时，会执行dispatch.bind(null, i + 1)
          return Promise.resolve(fn(context, dispatch.bind(null, i + 1)));
        } catch (err) {
          return Promise.reject(err)
        }
      }
    }
}
```