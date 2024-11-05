---
title: "一个作业评测系统的全栈实现"
tags: [code, wip]
---

<!--more-->


> 修订历史
> - 2024.03.02 创建笔记
> - 2024.03.25 完善


## 背景知识
- **grpc** grpc 可以定义四种类型的服务方法:
    1. `rpc SayHello(HelloRequest) returns (HelloResponse);`发送一个请求，得到一个响应
    2. `rpc LotsOfReplies(HelloRequest) returns (stream HelloResponse);`服务器流式 rpc，其中客户端向服务器发送请求，并获得一个流来读取一系列消息。客户端从返回的流中读取，直到没有更多的消息。gRPC 保证在单个 RPC 调用中的消息是有序的
    3. `rpc LotsOfGreetings(stream HelloRequest) returns (HelloResponse);`客户端流式 rpc，其中客户端写入一系列消息并将其发送到服务器，同样使用提供的流。一旦客户端完成了消息的写入，它就等待服务器读取消息并返回响应。同样，gRPC 保证在单个 RPC 调用中对消息进行排序
    4. 双向流式 rpc，其中双方使用读写流发送一系列消息。这两个流独立运行，因此客户端和服务器可以按照自己喜欢的顺序读写: 例如，服务器可以等待接收所有客户端消息后再写响应，或者可以交替读取消息然后写入消息，或者其他读写组合。每个流中的消息是有序的
    ```
    // hello.proto
    service Greeter {
        rpc SayHello (HelloRequest) returns (HelloResponse) {}
    }

    // server.go
    type server struct {
        pb.UnimplementedGreeterServer
    }
    func (s *server) SayHello(ctx context.Context, in *pb.HelloRequest) (*pb.HelloResponse, error) {
        ...
    }
    ... 
    s := grpc.NewServer()
    pb.RegisterGreeterServer(s, &server{})
    err = s.Serve(lis)
    ...

    // client.go
    ...
    conn, err := grpc.Dial(*addr, grpc.WithTransportCredentials(insecure.NewCredentials()))
    c := pb.NewGreeterClient(conn)
    r, err := c.SayHello(ctx, &pb.HelloRequest{Name: *name})
    ...
    ```

- **Oauth2** 授权码模式：
    > https://docs.github.com/zh/apps/oauth-apps/building-oauth-apps/authorizing-oauth-apps
    1. 用户在应用程序中，应用程序尝试获取用户保存在资源服务器上的信息，比如用户的身份信息和头像，应用程序首先让重定向用户到授权服务器，告知申请资源的读权限，并提供自己的client id。
    2. 到授权服务器，用户输入用户名和密码，服务器对其认证成功后，提示用户即将要颁发一个读权限给应用程序，在用户确认后，授权服务器颁发一个授权码（authorization code）并重定向用户回到应用程序。
    3. 应用程序获取到授权码之后，使用这个授权码和自己的client id/secret向认证服务器申请访问令牌/刷新令牌（access token/refresh token）。认证服务器对这些信息进行校验，如果一切OK，则颁发给应用程序访问令牌/刷新令牌。
    4. 应用程序在拿到访问令牌之后，向资源服务器申请用户的资源信息
    5. 资源服务器在获取到访问令牌后，对令牌进行解析（如果令牌已加密，则需要进行使用相应算法进行解密）并校验，并向授权服务器校验其合法性，如果一起OK，则返回应用程序所需要的资源信息。

- **CORS** 
    > https://www.ruanyifeng.com/blog/2016/04/cors.html

    CORS需要浏览器和服务器同时支持。目前，所有浏览器都支持该功能。只要服务器实现了CORS接口，就可以跨源通信。

    CORS请求分成两类：简单请求（simple request）和非简单请求（not-so-simple request）。
    
    简单请求，浏览器在头信息中增加一个Origin字段。浏览器根据服务器是否返回 Access-Control-Allow-Origin 字段，判断是否得到跨域许可。
    
    非简单请求浏览器会先发一次请求方法是 OPTIONS 的 "预检" 请求（preflight），判断返回的 Access-Control-Allow-Origin 字段是否存在。

- **grpc interceptor**
- **grpc-web** 浏览器不能直接发起 gRPC 请求，因此浏览器通过 HTTP 协议或者 WebSocket，将 protobuf 封装后的请求数据发给中间代理，中间代理取出封装的数据转发给后端 RPC 服务。中间代理收到后端 RPC 服务的响应后，通过 HTTP 协议或者 WebSocket 返回给浏览器
- **go embed 包** 通过 `//go:embed` 指令，可以在编译阶段将静态资源文件打包进编译好的程序中，并提供访问这些文件的能力。*简化部署*
- **JWT**
- **go crypto/subtle 包** 实现了时间攻击安全的函数，这些函数的执行时间不会因输入值的不同而有所差异，从而防止了一些侧信道攻击。例如使用 ConstantTimeCompare 验证用户密码
- **crypto/bcrypt** bcrypt 是大量消耗 cpu 的 hash 算法，相较于MD5、SHA-1、SHA-256等哈希算法更适合用于做密码的哈希。

### 参考
- [Autograder - 一个适合项目作业的评测系统](https://zhuanlan.zhihu.com/p/479027855)
- [Autograder 使用文档](https://autograder-docs.howardlau.me/)