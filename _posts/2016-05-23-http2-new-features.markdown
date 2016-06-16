---
layout: post
title:  "HTTP/2 新特性"
date:   2016-05-23 11:04:54 +0800
categories: http
---
本文介绍了http/2的新特性，包括：

* [多路复用](#多路复用)
* [二进制分帧层](#二进制分帧层)
* [首部压缩](#首部压缩)
* [服务端推送](#服务端推送)

## <a name="多路复用"></a>多路复用

* 同一域名下连接数量限制

HTTP/2之前，浏览器客户端在同一时间，针对同一域名下的请求有一定数量限制。超过限制数目的请求会被阻塞。针对这一原因，我们目前的优化方案有：有多个静态资源 CDN 域名，sprite图。

下图展示了不同浏览器的限制：

![connections per server](/assets/http2-new-features/connections-per-server.png)

* 连接耗时、阻塞

每一个请求过程是：TCP三次握手建立链接 --> HTTP请求 --> HTTP相应 -->断开连接。每次发起一个新的请求，就要重新创建，不能利用之前的连接。同时由于TCP的慢启动，使得具有突发性和短时效的http很低效。

**慢启动**: 是一种[阻塞控制](https://zh.wikipedia.org/wiki/%E6%8B%A5%E5%A1%9E%E6%8E%A7%E5%88%B6)机制。慢启动是指每次TCP接收窗口收到确认时都会增长。增加的大小就是已确认段的数目。这种情况一直保持到要么没有收到一些段，要么窗口大小到达预先定义的阈值。TCP连接会随着时间进行自我「调谐」，起初会限制连接的最大速度，如果数据成功传输，会随着时间的推移提高传输的速度。

HTTP/1.1提出了**持久连接**来解决这个问题。

**持久连接**：HTTP持久连接可以重用已建立的TCP连接，减少三次握手的RTT延迟。浏览器在请求时带上 connection: keep-alive的头部，服务器收到后就要发送完响应后保持连接一段时间，浏览器在下一次对该服务器的请求时，就可以直接拿来用。

以往，浏览器判断响应数据是否接收完毕，是看连接是否关闭。在使用持久连接后，就不能这样了，这就要求服务器对持久连接的响应头部一定要返回content-length标识body的长度，供浏览器判断界限。有时，content-length的方法并不是太准确，也可以使用 Transfer-Encoding: chunked 头部发送一串一串的数据，最后由长度为0的chunked标识结束。

![persistent connnection](/assets/http2-new-features/persistent-connection.png)

为了进一步缩短时间，HTTP/1.1实现了**管线化**（ pipelining）。

**管线化**：是将多个请求一起发给服务器，在发送过程中不需先等待服务端的回应。但是服务器端必须按照客户端的请求顺序恢复请求，这样整个连接还是先进先出的，[队首阻塞](https://en.wikipedia.org/wiki/Head-of-line_blocking)（Head-of-line blocking）可能会发生，造成延迟。

**队首阻塞**：前一个慢请求阻塞后面的所有请求。若上一响应迟迟不回，后面的响应都会被阻塞到。 

![head of line blocking](/assets/http2-new-features/hol-blocking.png)

在兼容HTTP/1.x 的情况下，HTTP/2利用二进制分帧层解决了这个问题。

在HTTP2中，相同域名下的所有通信都在一个连接上完成，这个连接中可以承载任意数量的双向流。这些流都是以消息的形式被发送的，同时消息又由一个或多个帧组成。每次请求/响应使用不同的 Stream ID。多个帧之间可以乱序发送，最后根据帧首部的流标识重新组装。

## <a name="二进制分帧层"></a>二进制分帧层

HTTP/2 采用二进制格式传输数据，而之前采用文本格式传输。二进制数据一个显而易见的好处是：更小的传输体积。

大家都知道计算机的存储在物理上是二进制的，所以文本文件与二进制文件的区别并不是物理上的，而是逻辑上的。这两者只是在编码层次上有差异。简单来说，文本文件是基于字符编码的文件，常见的编码有ASCII编码，UNICODE编码等等。二进制文件是基于值编码的文件，你可以根据具体应用，指定某个值是什么意思。这样一个过程，可以看作是自定义编码。从上面可以看出文本文件基本上是定长编码的，也有非定长的编码如UTF-8。而二进制文件可看成是变长编码的，因为是值编码嘛，多少个比特代表一个值，完全由你决定。

* 帧

HTTP/2 通讯中的最小传输单位，至少含有一个 Frame header，能够表示它属于哪一个 Stream。 

Frame 的基本格式如下（图中的数字表示所占位数）:

![frame layout](/assets/http2-new-features/frame-layout.png)

**Length**: 表示 Frame Payload 部分的长度，另外 Frame Header 的长度是固定的 9 字节（Length + Type + Flags + R + Stream Identifier = 72 bit）。

**Type**: 区分这个 Frame Payload 存储的数据是属于 HTTP Header 还是 HTTP Body；另外 HTTP/2 新定义了一些其他的 Frame Type，例如，这个字段为 0 时，表示 DATA 类型（即 HTTP/1.x 里的 Body 部分数据）

**Flags**: 共 8 位， 每位都起标记作用。每种不同的 Frame Type 都有不同的 Frame Flags。例如发送最后一个 DATA 类型的 Frame 时，就会将 Flags 最后一位设置 1（flags &= 0x01），表示 END_STREAM，说明这个 Frame 是流的最后一个数据包。

**R**: 保留位。

**Stream Identifier**: 流 ID，当客户端和服务端建立 TCP 链接时，就会先发送一个 Stream ID = 0 的流，用来做些初始化工作。之后客户端和服务端从 1 开始发送请求/响应。

**Frame**由**Frame Header**和**Frame Payload**两部分组成。不论是原来的**HTTP Header**还是**HTTP Body**，在HTTP/2中，都将这些数据存储到**Frame Payload**，组成一个个**Frame**，再发送响应/请求。通过**Frame Header**中的**Type**区分这个**Frame**的类型。由此可见语义并没有太大变化，而是数据的格式变成二进制的Frame。二者的转换和关系如下图:

![frame transform](/assets/http2-new-features/frame-transform.png)

* 消息

一个完整的请求或者响应，包含多个 Frame 序列。

![message](/assets/http2-new-features/message.png)

* 流

已建立的连接上的双向字节流。具有唯一的流ID，客户端发起的为奇数ID，服务端发起的为偶数ID。很多个流可以并行的在同一个tcp连接上交换消息。

流有以下特点：

1. 一个HTTP/2连接可同时保持多个打开的流，任一端点交换帧
2. 流可被客户端或服务器单独或共享创建和使用
3. 流可被任一端关闭
4. 在流内发送和接收数据都要按照顺序
5. 流的标识符自然数表示，1~2^31-1区间，有创建流的终端分配
6. 流与流之间逻辑上是并行、独立存在

流的概念提出是为了实现多路复用，在单个连接上实现同时进行多个业务单元数据的传输。逻辑图如下：

![one-http2-connection](/assets/http2-new-features/one-http2-connection.png)

实际传输可能是这样的：

![one-http2-connection-multiplexing](/assets/http2-new-features/one-http2-connection-multiplexing.png)

它允许多个并发HTTP请求共用一个 TCP会话，而不是为每个请求单独开放连接，这样只需建立一个TCP连接就可以传送网页上所有资源，不仅可以减少消息交互往返的时间还可以避免创建新连接造成的延迟，使得TCP的效率更高。


## <a name="首部压缩"></a>首部压缩

由于HTTP协议是一种无状态的协议，服务器自身不会保存太多的信息和先前请求过的元数据信息，这就需要客户端每次都使用多个首部字段来描述所传输的资源及其属性，又由于HTTP1.1是以文本形式传输的，每次都会给HTTP报文增加500-800字节的负荷，如果算上cookie，这个负荷甚至会增加到上千。如果可以有办法减少这些开销，那么对性能又有很大的提升。

在HTTP/1.x中首部是没有压缩的，gzip只会压缩body，HTTP/2提供了首部压缩方案。HTTP/2消息头的压缩算法采用 [HPACK](http://http2.github.io/http2-spec/compression.html)，而SPDY采用的[DEFLATE](http://zh.wikipedia.org/wiki/DEFLATE)。

**HPACK**：它使用一份索引表来定义常用的**HTTP Header**。把常用的**HTTP Header**存放在表里。请求的时候便只需要发送在表里的索引位置即可。因为索引表的大小的是有限的，它仅保存了一些常用的**HTTP Header**，同时每次请求还可以在表的末尾动态追加新的**HTTP Header**缓存。动态部分称之为**Dynamic Table**。**Static Table**和**Dynamic Table**在一起组合成了索引表。

例如**:method=GET**使用索引值**2**表示，**:path=/index.html**使用索引值**5**表示。完整的列表参考：[HPACK Static Table](http://http2.github.io/http2-spec/compression.html#rfc.section.A)。只要给服务端发送一个Frame，该Frame的Payload部分存储**0x8285**。高位设置为**1**表示这个字节是一个完全索引值，key和value都在索引中。类似的，通过高位的标志位可以区分出这个字节是属于一个完全索引值，还是仅索引了key，还是key和value都没有索引。

HPACK不仅仅通过索引键值对来降低数据量，同时还会将字符串进行霍夫曼编码来压缩字符串大小。

![dynamic table](/assets/http2-new-features/dynamic-table.png)

## <a name="服务端推送"></a>服务端推送
一个文档被请求回来时，往往还需要再次请求很多文档内的其他资源，如果这些资源的请求不用客户端发起，而是服务端提前预判发给客户端，那么就会减少一半的RTT。

当服务端需要主动推送某个资源时，便会发送一个 Frame Type 为 PUSH_PROMISE 的 Frame，里面带了 PUSH 需要新建的 Stream ID。意思是告诉客户端：接下来我要用这个 ID 向你发送东西，客户端准备好接着。客户端解析 Frame 时，发现它是一个 PUSH_PROMISE 类型，便会准备接收服务端要推送的流。

相关参考:

* <https://imququ.com/post/http2-new-opportunities-and-challenges.html>
* <https://imququ.com/post/header-compression-in-http2.html>
* <https://imququ.com/post/server-push-in-http2.html>
* <http://io.upyun.com/2015/05/13/http2/>
* <http://jiaolonghuang.github.io/2015/08/16/http2/>
* <http://www.cnblogs.com/ghj1976/p/4552583.html>
* <https://www.zhihu.com/question/34074946/answer/75364178>
* <http://http2.github.io/http2-spec/index.html>
* <http://http2.github.io/http2-spec/compression.html>
* <https://http2.akamai.com/demo>
* <http://www.blogjava.net/yongboy/archive/2015/03/19/423611.html>
* <http://www.kancloud.cn/digest/web-performance-http2/74816>
* <http://www.cnblogs.com/zhangjiankun/archive/2011/11/27/2265184.html>
* <https://httpwg.github.io/specs/rfc7540.html#PushResources>

