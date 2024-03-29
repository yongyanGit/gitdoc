## http请求过程

## 用户输入url

当用户输入url，操作系统会将输入事件传递到浏览器中，在这过程中，浏览器可能会做一些预处理，比如 Chrome  会根据历史统计来预估所输入字符对应的网站，例如输入goog，根据之前的历史发现 90% 的概率会访问「www.google.com  」，因此就会在输入回车前就马上开始建立 TCP 链接甚至渲染了。

接着是输入url之后，点击回车，这时浏览器会对 URL 进行检查，首先判断协议，如果是 http 就按照 Web 来处理，另外还会对这个 URL 进行安全检查

安全检查完成之后，在浏览器内核中会先查看缓存，然后设置 UA 等 HTTP 信息，接着调用不同平台下网络请求的方法。

## http网络请求

通过 DNS 查询 IP；
通过 Socket 发送数据

## dns查询ip

DNS，英文是Domain Name System，中文叫域名系统，是Internet的一项服务，他将域名和IP地址相互映射的一个分布式数据库

假设用户在浏览器中输入的是www.google.com，大概过程：

如果输入的是域名，则需要进行dns查询，将域名解析成ip；

进行DNS查询的主机或软件叫做DNS解析器，用户使用的工作站或电脑都属于解析器。域名解析就是利用DNS解析器得到对应IP过程，解析器会向域名服务器进行查询处理。

主要过程如下：

1. 从浏览器缓存中查找域名www.google.com的IP地址
2. 在浏览器缓存中没找到，就在操作系统缓存中查找，这一步中也会查找本机的hosts看看有没有对应的域名映射（当然已经缓存在系统DNS缓存中了）
3. 在系统中也没有的话，就到你的路由器来查找，因为路由器一般也会有自己的DNS缓存

如果以上都没有找到，则继续往下向dns域名服务器查询

- 用户电脑的解析器向LDNS（也就是Local   DNS，互联网服务提供商ISP），发起域名解析请求，查询www.google.com的IP地址，这是一个递归查找过程
- 在缓存没有命中的情况下，LDNS向根域名服务器.查询www.google.com的IP地址，LDNS的查询过程是一个迭代查询的过程
- 根告诉LDNS，我不知道www.google.com对应的IP，但是我知道你可以问com域的授权服务器，这个域归他管
- LDNS向com的授权服务器问www.google.com对应的IP地址
- com告诉LDNS，我不知道www.google.com对应的IP，但是我知道你可以问google.com域的授权服务器，这个域归他管
- LDNS向google.com的授权服务器问www.google.com对应的IP地址
- google.com查询自己的ZONE文件（也称区域文件记录），找到了www.google.com对应的IP地址，返回给LDNS
- LDNS本地缓存一份记录，把结果返回给用户电脑的解析器
- 在这之后，用户电脑的解析器拿到结果后，缓存在自己操作系统DNS缓存中，同时返回给浏览器，浏览器依旧会缓存一段时间。

**注意**，
域名查询时有可能是经过了CDN调度器的（如果有cdn存储功能的话）

而且，需要知道dns解析是很耗时的，因此如果解析域名过多，会让首屏加载变得过慢，可以考虑dns-prefetch优化

## tcp/ip请求

有了 IP 地址，就可以通过 Socket API 来发送数据了，这时可以选择 TCP 或 UDP 协议。

http本质是tcp协议。

TCP是一种面向有连接的传输层协议。他可以保证两端（发送端和接收端）通信主机之间的通信可达。他能够处理在传输过程中丢包、传输顺序乱掉等异常情况；此外他还能有效利用宽带，缓解网络拥堵。

建立TCP连接一开始都要经过三次握手：

> 第一次握手，请求建立连接，发送端发送连接请求报文
>
> 第二次握手，接收端收到发送端发过来的报文，可知发送端现在要建立联机。然后接收端会向发送端发送一个报文
>
> 第三次握手，发送端收到了发送过来的报文，需要检查一下返回的内容是否是正确的；若正确的话，发送端再次发送确认包

在TCP连接建立完成之后就可以发送HTTP请求了。可以将数据发送给服务器，并收到返回信息。

当请求结束，需要经厉连接终止协议（四次挥手）。

由于TCP连接是全双工的，因此每个方向都必须单独进行关闭。这原则是当一方完成它的数据发送任务后就能发送一个FIN来终止这个方向的连接。收到一个 FIN只意味着这一方向上没有数据流动，一个TCP连接在收到一个FIN后仍能发送数据。首先进行关闭的一方将执行主动关闭，而另一方执行被动关闭。