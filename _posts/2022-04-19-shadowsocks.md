---
title: "Shadowsocks源码阅读"
tags: [code, net]
---

今天抽空看了一下shadowsocks的源码。因为代码逻辑很简单，再加上自己学习过Muduo网络库，触类旁通，所以就只记录核心代码，仅供回忆使用。

<!--more-->

shadowsocks代码量很少，才两千多行，逻辑也比较简单，一天就可以看完。根据它的功能，也可以很容易地猜出其实现方式：应该有一个运行在本地的程序（sslocal），解析sock5协议的数据并进行加密，转发到运行在远端的程序（ssserver），远端程序解密后根据sock5协议的目的地址去访问目标服务器。返回数据的流程相似，这里不细说。

sslocal和ssserver都是采用Reactor的模型，都有一个事件循环（EventLoop）。同时也都有TCP、UDP服务器（TCPRelay、UDPRelay），用来接受连接。服务器会往事件循环中注册自身的socket事件，当有新连接来临，便开启新的socket（TCPRelayHandler、UDPRelay）来处理请求，同时往事件循环中注册该事件。

### EventLoop
EventLoop没啥好说的，会调用epoll、kqueue或select，当事件到达，会调用该事件handler（TCPRelay、UDPRelay、DNSResolver）的handle_event。当然还支持定时回调函数。

### LRUCache
LRUCache顾名思义。不过它的容量没限制，需要手动调用sweep才清除，距离上次使用时间超过阈值（构造时指定）的对象。在代码中，每个LRUCache对象的sweep函数会被注册到EventLoop的定时回调中。

### DNSResolver
DNS客户端，也会向事件循环注册可读事件和定时回调，提供一个发送DNS请求的接口resolve，该接口可以注册回调函数。调用resolve会先进行内部查找，找不到才向DNS服务器发送查询请求。

当调用resolve后收到可读事件，即DNS响应到达，会把键值对<hostname, ip>插入LRUCache对象_cache中，然后调用resolve传入的回调函数。而刚刚说的DNSResolver会向事件循环注册定时回调函数，其内容就是调用_cache的sweep()。

### UDPRelay
接收UDP数据的前提是客户端已经和sock5服务端握手成功，客户端通过sock5协议发送UDP的流程如下：

1. 通过TCP建立与服务端握手
2. 通过TCP请求UDP代理，服务端会回应UDP监听地址和端口
3. 发送UDP到第二步获得的地址

以下的讨论都是建立在1、2步已经完成的情况。

UDPRelay被部署在两个地方：本地和远程，功能因此也有所不同，代码里用is_local来进行区分。UDPRelay会往事件循环中监听server_socket的可读事件。当收到数据时，会建立一个新的client socket用来转发该数据。同时该client socket会把自身和数据源地址绑定成键值对记录到_client_fd_to_server_addr中，也会事件循环中监听可读事件，用于把转发数据的回复数据传回。

在这里，sslocal的UDPRelay会检查收到的sock5字段是否正确，然后去掉前3个字节进行加密，通过新的socket发送给ssserver。而ssserver的UDPRelay会把收到的数据解密，检查字段是否正确，最后取出其中数据的部分，通过新的socket发送给目标服务器。

当UDPRelay的handle_event被回调，且唤醒的是client socket，则会查询_client_fd_to_server_addr，将收到的数据传回对应的地址。在这里ssserver会将收到的数据加上ATYP、DST.ADDR和DST.PORT字段，加密其内容传给对应的sslocal。sslocal收到数据会解密，检查字段后，加上RSV和FRAG字段传回相应的用户客户端。

每次新建立的client都是存在LRUCache对象_cache中，UDPRelay往事件循环注册的定期函数就是移出建立时间超出阈值的client。*看起来只是允许UDP一发一收*

### TCPRelay
TCPRelay的逻辑和UDPRelay类似，同样会在事件循环监听_server_socket可读事件。当可读即有新连接到达时，accept并创建一个TCPRelayHandler进行管理，并在事件循环监听其_local_sock的可读事件。

一个TCPRelayHandler会有一个sock5握手的过程，代码中通过状态机_stage来实现握手各个阶段。sslocal和ssserver的状态变化相近。以sslocal为例，最开始的状态是STAGE_INIT，等待握手。收到握手请求后会回应并转为STAGE_ADDR。

接着会收到新的数据包，其中指明接下来的数据是要通过UDP还是TCP传输。如果收到的数据包指明要通过UDP传输，则返回_local_sock的ip地址和端口。注意，TCP和UDP服务器可以绑定在同一个地址上，所以客户端用UDP方式发数据到返回的地址，会被UDPRelay处理。

如果是指明数据通过TCP传输的话，直接去掉前三个字节，检查剩下字段，进行加密，把数据放入发送缓冲区。状态改为STAGE_DNS，停止监听_local_sock的可读事件，调用DNSResolver的resolve查询sserver服务器ip。 

当DNSResolver的resolve返回，说明获得服务器ip，新建连接到sserver的_remote_sock，并将其加入事件循环的监听中，监听_local_sock的可读、_remote_sock的可读可写，状态改为STAGE_CONNECTING。此时从_local_sock读到的数据都会存入发送缓冲区。直到_remote_sock可写，状态变为STAGE_STREAM。

当_remote_sock可写，会把发送缓冲区的数据发到_remote_sock，然后继续监听_local_sock的可读事件。如果缓冲区数据一次性发不完，则不监听_local_sock的可读事件，监听_remote_sock的可写。

当_local_sock可读，会把读到的数据加密，发到_remote_sock。如果数据一次性发不完，则会把剩下数据存入发送缓冲区，不监听_local_sock的可读事件，监听_remote_sock的可写。直到把数据发完，就只监听_local_sock的可读。

这里有点意思，一个方向上，只能监听两个端口的其中一个：_remote_sock不可写还监听_local_sock的可读，会导致内存耗尽；_remote_sock可写还监听_remote_sock的可写，会busy loop。

当_remote_sock可读，会把数据解密，写入_local_sock。其中数据写不完的处理方式同上，只是换个流动方向罢了。

上面谈论的是sslocal，现在说说sserver。它最开始的状态是STAGE_INIT，收到数据后会先解密并检查字段，获得目标服务器地址，状态改为STAGE_DNS。接下来逻辑都和sslocal一样，只是关键地方加解密不同而已。

### 总结
到这，shadowsocks的核心源码分析结束，简单谈一谈其中一些缺陷：

* 加密后的流量近乎是随机，然而过分的随机本身就是一种特征。相比具有固定结构的主流应用层协议，更容易被识别出来
* 没有包长度混淆，加密流量和负载流量的包的长度、时序一一对应，容易被识别
* 协议简陋不安全