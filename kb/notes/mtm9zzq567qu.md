# gRPC 底层实现 + 使用

gRPC 是谷歌开源 RPC 框架，**底层基于 HTTP/2 传输，默认 Protobuf 序列化**。

> RPC：远程过程调用，像调用本地函数一样调用另一台机器上的服务方法。

---

## 一、整体架构分层（由上到下）

1. **Stub 层（桩代码）**：根据 `.proto` 文件生成，客户端直接调用本地函数，屏蔽网络细节
2. **Protobuf 序列化层**：对象 ↔ 二进制字节流，体积小、编解码快
3. **HTTP/2 传输层**：底层通信依靠 HTTP/2，TCP 为基础
4. **TCP 网络层**：操作系统 Socket

---

## 二、底层核心实现

### 1. 传输：HTTP/2（gRPC 核心）

gRPC **不使用 HTTP1.1，强制 HTTP/2**，HTTP/2 的关键特性支撑 gRPC：

1. **二进制帧**：把数据切分成二进制 frame，不再是文本 http，解析高效
2. **多路复用**：一条 TCP 连接上，可以并发跑多个请求流 (stream)，不用频繁建连；请求互不阻塞
3. **双向流**：支持四种调用模式（一元、服务端流、客户端流、双向流），HTTP1.1 做不到双向流
4. **头部压缩 HPACK**：压缩 http header，减少开销
5. **长连接**：默认复用 TCP 连接，减少三次握手开销

> gRPC 的请求是封装在 HTTP/2 的 data‑frame 里面，不是普通 JSON http body。

### 2. 序列化：Protobuf

- 定义 `.proto` 描述**服务、接口、数据结构**
- 编译器 `protoc` 根据 proto 文件，自动生成客户端、服务端桩代码 (stub)
- 数据转紧凑二进制，对比 JSON：体积更小、序列化反序列化速度更快；缺点是可读性差。

### 3. 四种调用模式

1. **一元 RPC（Unary）**：客户端发 1 请求 → 服务返回 1 响应（最常用，类似普通 http 接口）
2. **服务端流式 RPC**：客户端发 1 请求，服务端返回一串数据流（例如大量数据推送）
3. **客户端流式 RPC**：客户端连续发一堆请求，服务端返回 1 个应答
4. **双向流式 RPC**：两边同时互发数据流（聊天、实时数据）

### 4. 连接、负载均衡

- 客户端维护连接池，复用 HTTP/2 TCP 连接
- gRPC 内置负载均衡策略：round_robin 轮询、pick_first 选第一个可用节点；支持 name resolver 域名解析。
- 支持拦截器 Interceptor：做日志、鉴权、超时、重试。

### 5. 超时与状态码

gRPC 自带标准状态码：OK、NOT_FOUND、DEADLINE_EXCEEDED、UNAVAILABLE 等；**客户端必须设置超时 deadline**，否则请求可能无限阻塞。

> ❗注意：gRPC 底层**不是自定义私有 TCP 协议**，是标准 HTTP/2 协议，可以抓包看到 http2 帧

---

## 三、简单使用示例（C++，贴合你的技术栈）

### 步骤 1：编写 hello.proto

1.定义请求结构体
2.定义返回结构体
3.定义服务与方法
```
syntax = "proto3";

package helloworld;

// 请求结构体
message HelloRequest {
  string name = 1;
}
// 返回结构体
message HelloReply {
  string message = 1;
}

// 定义服务与方法
service Greeter {
  // 一元RPC
  rpc SayHello (HelloRequest) returns (HelloReply);
}
```

### 步骤 2：编译 proto 生成代码

使用 `protoc` + grpc‑cpp 插件，生成：

- `hello.pb.h / hello.pb.cc`：protobuf 消息编解码
- `hello.grpc.pb.h / hello.grpc.pb.cc`：gRPC 服务、Stub 桩代码

### 步骤 3：服务端 C++

继承生成的 `Greeter::Service`，重写 `SayHello` 接口；启动 gRPC Server 监听端口。

### 步骤 4：客户端 C++

通过 Stub 调用本地风格函数，底层走网络。

```
int main(){
  auto channel = grpc::CreateChannel("127.0.0.1:50051", grpc::InsecureChannelCredentials());
  auto stub = helloworld::Greeter::NewStub(channel);

  helloworld::HelloRequest req;
  req.set_name("test");
  helloworld::HelloReply rsp;

  grpc::ClientContext ctx;
  // 发起RPC调用，看起来像本地函数，实际网络通信
  auto status = stub->SayHello(&ctx, req, &rsp);
  if(status.ok()){
    std::cout << rsp.message() << std::endl;
  }
  return 0;
}
```
