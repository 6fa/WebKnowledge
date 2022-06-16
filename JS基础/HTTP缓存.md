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

#### Expires
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

#### Cache-Control
