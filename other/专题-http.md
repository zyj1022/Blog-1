# http

![架构图](./http2.jpg)

## 1 基础知识

1. http协议简历在tcp协议之上，http协议的性能瓶颈及其性能优化都是针对tcp操作的
2. 目前在未应用https或http2之前，默认都是http1.1
3. https是运行在ssl/tls上的安全协议
4. spdy基于https，支持多路复用与header压缩，提高效率，拥有https的优点
5. http2是spdy的升级版本，支持2进制解析，拥有spdy的优点

## 2 http 基本优化要点

### 2.1 带宽

目前网络基础建设得到很大提升，带宽不是影响的主要因素

### 2.2 延迟

1. 浏览器阻塞(对于同一域名，一般只有仅限的几个连接，超过被阻塞)
2. dns查询：浏览器获取目标服务器的Ip地址
3. 建立连接：每次连接都经历3次握手和慢启动

### 2.3 数据明文传输

> 怎么解决？加密

### 2.4 tcp3次握手

每次传输数据时，都需要重新建立连接，增加了大量的延迟时间

> 怎么解决？保持长链接

### 2.5 header携带的内容过大

每次传输，header的内容都几乎不怎么变化，浪费空间

> 怎么解决？长链接、压缩

### 2.6 `connection: keep-alive` 使用过多

使用过多会对服务器造成过大的压力

> 怎么解决？

## 3 http1.1相较与http1.0的优点

1. 缓存机制更多：增加更多的缓存策略
2. 带宽优化：支持请求部分资源，而不是整个对象(206 partial content)
3. 错误状态码增多：409表示请求的资源与资源的当前状态发生冲突，410表示服务器的资源被永久性删除
4. 增加host：服务器可以配置多个虚拟主机
5. 保持长连接：`connection: keep-alive`，减少tcp每次的连接(连续请求)

## 4 https相较与http1.1的区别

1. https需要ca安全证书
2. 端口不同： http 80，https 443
3. https传输内容加密
4. https有效防止运营商劫持
5. 对服务器压力更大：密钥计算，消耗cpu资源

## 5 将网站改造成https

1. 购买并安装CA证书
2. https存在ssl握手过程，不优化会存在速度降低

## 6 SPDY

1. 多路复用：降低延迟，对于每一个连接，都可以重复使用，发送多条消息
2. header压缩
3. 基于https协议加密
4. 主动推送：同名称静态文件单个请求，多个推送

> 注意：高版本的chrome对spdy已经不支持

## 7 http2相较与SPDY的区别

1. http2是spdy的升级版本
2. http2支持http、https传输，spdy强制https
3. header压缩算法不同，http2 HPACK，SPDY DEFLATE

### 8 有了http2，不用再做哪些优化

1. HTTP2对数据进行二进制分帧，
2. 在http和tls之间增加的一层分帧层，
3. 每个分片包含分片信息和header压缩信息，
4. 通过一个tcp连接，无序地传输分片，然后再在接收端组装，
5. 而不必考虑代码合并或者sprite图优化了(所有需要的资源一次性地给你，然后分片无序传输给你)，
6. 这个唯一的tcp连接是一个双工字节流连接，服务器主动推送资源
7. 分片之间存在优先级，解决了资源阻塞问题，

## ssl 握手

[ssl运行机制](http://www.ruanyifeng.com/blog/2014/02/ssl_tls.html)

应用的是非对称加密(公钥私钥)与对称加密

1. 基本消息对称加密，提升非对称加密速度
2. 公钥对已经加密过的消息(对称加密)进行加密，私钥解密
3. 公钥放在证书中，共享出去，私钥只解密含有证书的公钥机密消息

握手阶段：在tcp3次握手之前，会有ssl四次握手

1. 客户端->服务端：`clientHello`、`tls版本`、`可用的加密算法集合`、`可用的压缩算法`、`第一个随机数`
2. 服务端->客户端：`serverHello`、`采用的tls版本`、`采用的加密算法`、`采用的压缩算法`、`ca证书，含公钥`、`第二个随机数`
3. 客户端->服务端: `确认加密结束`、`第三个随机数`、`ack`
4. 服务端->客户端：`确认加密`、`ack`

> 相比与 tcp 的3次握手， tls 的4次握手，就是多了 `tls版本`、`加密算法`、`压缩算法`、`ca证书及公钥`、`随机数`，每次会话都需要进行 tls 4次握手，所以 http2 采用了长链接
> 随机数作用：公钥只是用来加密，非对称加密；随机数用于每次会话生成对称加密；第三个随机数会用第二次握手服务器提供的公钥进行加密

**工作过程**：4次握手结束之后，然后就普通 http 通信了，不过数据都才用会话密钥进行加密传输。这里的会话密钥就是通过那3个随机数、采用的加密算法生成的。比如 RSA密钥交换算法 + 3个随机数 = 特定对称密钥

**公钥作用**：服务器提供的公钥，用于第三个随机数加密

**私钥作用**：服务器解密第三个随机数

**对称密钥作用**：握手之后的 http 通信数据加密

## 慢启动等

关键词： `超时重传`、`tcp窗口`、`接收承受最大窗口`、`发送拥塞窗口`

1. 通信开始时，发送方的拥塞窗口大小为 1。每收到一个 ACK 确认后，拥塞窗口翻倍。
2. 由于指数级增长非常快，很快地，就会出现确认包超时。
3. 此时设置一个“慢启动阈值”，它的值是当前拥塞窗口大小的一半。
4. 同时将拥塞窗口大小设置为 1，重新进入慢启动过程。
5. 由于现在“慢启动阈值”已经存在，当拥塞窗口大小达到阈值时，不再翻倍，而是线性增加。
6. 随着窗口大小不断增加，可能收到三次重复确认应答，进入“快速重发”阶段。
7. 这时候，TCP 将“慢启动阈值”设置为当前拥塞窗口大小的一半，再将拥塞窗口大小设置成阈值大小（也有说加 3）。
8. 拥塞窗口又会线性增加，直至下一次出现三次重复确认应答或超时。

[慢启动及原理传输](http://blog.csdn.net/book_zhouqingjun216/article/details/51812786)