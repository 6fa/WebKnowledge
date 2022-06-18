# HTTP缓存

参考：

- [HTTP 缓存](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Caching)
- [前端浏览器缓存知识梳理](https://juejin.cn/post/6947936223126093861)
- [HTTP缓存和浏览器的本地存储](https://segmentfault.com/a/1190000020086923)
- [浏览器缓存看这一篇就够了](https://zhuanlan.zhihu.com/p/60950750)
- [浏览器的缓存机制](https://www.cnblogs.com/suihang/p/12855345.html)
- [一文读懂前端缓存](https://juejin.cn/post/6844903747357769742?utm_source=gold_browser_extension)

### HTTP缓存和浏览器本地缓存的区别
- HTTP缓存
  - 是对页面资源的缓存
  - 目的是提升页面性能、减轻服务器压力
- 浏览器本地缓存
  - 是指cookie、localStroage、sessionStroage、indexDB等
  - 目的就是为了储存一些用户数据、页面需要的数据等

### 缓存位置
浏览器缓存位置一般分为四种，从上往下依次检查是否命中：
- Service Worker：
  - 运行在浏览器背后的独立线程，可以用来实现缓存功能
  - 相当于自己去操作缓存，如缓存哪些文件、如何匹配缓存、如何读取缓存等
- Memory Cache：
  - 内存中的缓存，主要包含当前页面已经抓取的资源，自动加入memory cache
  - 一个页面中两个相同请求，只会请求一次
  - 读取比磁盘中的缓存速度快
  - 随着进程的释放（关闭tab页面时）而释放
- Disk Cache：
  - 磁盘缓存，相比内存缓存，胜在储存量和时效性上
  - 绝大部分的缓存都来自Disk Cache，在HTTP 的协议头中设置
  - 可以分为强缓存、协商缓存、启发性缓存
- Push Cache：
  - 推送缓存，是HTTP/2的内容，只在会话（session）中存在，关闭会话（标签或窗口）后释放
  - 缓存时间短，在Chrome中只有5分钟左右

### 强缓存
发出请求时，会先检查缓存数据库看是否已经存在，然后比较是否过期，没有过期则从缓存中获取资源；否则请求从服务器中获取（如果存在协商缓存，过期则走协商缓存步骤）。

触发强缓存的字段是Expires、Cache-Control。

#### 1.Expires
表示缓存的到期时间（是一个时间点），值是GMT格式的字符串:
```javascript
//服务器端
const maxAge = 3
//3秒后过期
//注意使用了toUTCString()方法
res.setHeader('Expires', (new Date(Date.now() + maxAge*1000)).toUTCString() )
//Expires: 'Sun, 07 Nov 2021 17:12:42 GMT'
```

缺点是受限于本地时间，如果修改本地时间，或时差都可能导致缓存失效。

#### 2.Cache-Control
在HTTP 1.1中，新增了Cache-Control字段。优先级高于Expires，为了兼容HTTP/1.0 和 HTTP/1.1，实际项目中两个字段都会设置。

Cache-Control表示缓存的有效期（是一个相对时间段），如：
```javascript
//服务器端
const maxAge = 3
//3秒后过期
res.setHeader('Cache-Control', `public, max-age=${maxAge}`)
```
Cache-Control字段常用值：
- max-age：单位为秒
- public：公有缓存，所有内容都可以被缓存，包括客户端和代理服务器
- private：私有缓存，内容只有客户端才能缓存，默认为private
- must-revalidate：超过了max-age，浏览器必须向服务器发送请求，验证资源是否还有效
- no-cache：缓存但需要重新验证，即会缓存，且取得缓存内容前要先验证是否过期
- no-store：不得缓存响应内容，每次请求资源都会从服务器下载

关于max-age=0与must-revalidate：
- max-age=0的意思是应该重新验证，而must-revalidate是必须重新验证。"max-age=0; must-revalidate" 则和 "no-cache"等价

### 协商缓存
当强制缓存失效时，会使用协商缓存，由服务器决定缓存是否失效。

触发协商缓存的字段组：Last Modified/If-Modified-Since，Etag/If-None-Match。

#### 1.Last Modified & If-Modified-Since
- 服务器通过Last Modified字段告知客户端，资源最后一次修改的时间
```javascript
//服务器端
//stats是fs.stat()读取文件信息时的参数
//fs.stat(filepath, (err,stats)=>{})
//stats.mtime即文件被修改的时间
res.setHeader('Last-Modified', stats.mtime.toUTCString())

//比如：
Last-Modified: Mon, 08 Nov 2021 06:37:03 GMT
```
- 浏览器会储存这些信息头，下次请求该资源时，请求头上会带上If-Modified-Since字段，值是上次的Last-Modified值
- 服务器比较两者，如果相等表示未修改，则返回304；否则响应200，返回新资源
- 缺点：
  - 如果文件是动态生成的，该方法的更新时间永远是生成的时间，尽管文件可能没有变化，所以起不到缓存的作用
  - 如果资源更新的速度是秒以下单位，那么该缓存是不能被使用的，因为它的时间单位最低是秒

#### 2.Etag & If-None-Match
- Etag是HTTP/1.1的，而前面的Last-Modified属于HTTP/1.0
- Etag优先级高于Last-Modified
- Etag的值是资源的唯一标识符（一般都是hash生成），资源有变化这个值就会改变，由服务端生成
- 在客户端，将etag值储存为If-None-Match
- 缺点：
  - 服务器需要根据资源内容计算Etag值，所以会有性能损失

### 启发性缓存
如果服务器没有返回明确的缓存策略时（没有设置Expires、Cache-Control:max-age），浏览器会使用启发性缓存：根据其他头部信息来计算过期时间，一般是Date和Last-Modified 两者差值的10%。

### 缓存流程总结
请求资源：
1. 查看是否有缓存
  a. 从Sevice Worker、Memory Cache、Disk Cache依次查找缓存
  b. Disk Cache中，如果有强缓存且未失效，则使用强缓存，状态码为200
  c. 如果有强制缓存但是失效了，则使用协商缓存，由服务器对比资源是否还有效，有效则返回304，否则返回200和新结果
2. （如果是新资源）把响应内容存入 disk cache (如果 HTTP 头信息配置可以存的话)
3. 把响应内容 的引用 存入 memory cache (无视 HTTP 头信息的配置)
4. 把响应内容存入 Service Worker 的 Cache Storage (如果 Service Worker 的脚本调用了 cache.put())

### 用户操作对缓存的影响
- F5刷新页面
  - 因为 TAB 并没有关闭，因此 memory cache 是可用的，会被优先使用(如果匹配的话)。其次才是 disk cache
- 操作浏览器上的前进、后退按钮，在地址栏上回车
  - 和上面一样，会使用缓存
- 强制刷新页面(ctrl+F5)
  - 浏览器不使用缓存，因此发送的请求头部均带有 Cache-control:no-cache(为了兼容，还带了 Pragma:no-cache),服务器直接返回 200 和最新内容