# gRPC 底层实现 + 使用

gRPC 是谷歌=={green}开源 RPC 框架==，**=={yellow}底层基于 HTTP/2 传输，默认 Protobuf 序列化==**=={yellow}。==

> =={yellow}**RPC：远程过程调用**==，=={yellow}像调用**本地函数一样**调用**另一台机器上的服务**方法。==

---

## 一、整体架构分层（由上到下）

1. **=={yellow}Stub 层（桩代码）==**：=={green}根据== `.proto` =={green}**文件生成**，客户端直接调用本地函数，屏蔽网络细节==
2. **=={yellow}Protobuf 序列化层==**：**对象 ↔ 二进制字节流**，体积小、编解码快
3. **=={yellow}HTTP/2 传输层==**：=={green}底层通信依靠 HTTP/2，TCP 为基础==
4. **=={yellow}TCP 网络层==**：=={green}操作系统 Socket==

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

继承生成 Service 类，重写接口方法，注册到 ServerBuilder 启动服务；启动 gRPC Server 监听端口。

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

---

## gRPC 的多路复用

**HTTP/2 的流（Stream）多路复用：单条 TCP 连接上跑多个逻辑 Stream**

1. **TCP 连接：物理连接，一条**
一条 TCP，只做一次三次握手。
2. **Stream：HTTP/2 的逻辑流，多条，在同一个 TCP 里面并发**

- 每个 RPC 请求对应一个独立 `Stream`，有唯一 stream‑id。
- 多个 Stream 的二进制帧交错混在同一个 TCP 通道传输，接收端根据 frame 的 stream‑id 拆分，还原不同请求的数据。

> ✅ 这叫**连接级多路复用（单 TCP 多逻辑流）**，不是多 TCP 连接。

### 关键区分（面试必说）

1. ❌ 不是 IO 多路复用（select/poll/epoll）：epoll 是操作系统 socket 事件模型，这是网络 IO 模型，和 gRPC 的多路复用不是一回事，很多人搞混。
2. ✅ gRPC 说的多路复用 = **HTTP/2 连接多路复用：同一个 TCP 连接，复用给多个并发 RPC（多个 Stream）**。

### 特点

- 不同 Stream 相互独立，一个 RPC 慢 / 阻塞，**不会阻塞同连接里其他 RPC**（帧交错发送）。
- 不用频繁创建销毁 TCP，减少握手开销。
- 正是这个 Stream 机制，才实现四种 RPC 模式，尤其是双向流。

### 面试极简回答

> gRPC 使用的是 **HTTP/2 连接级多路复用**：在**同一条 TCP 物理连接之上，开辟多个逻辑 Stream**，每个 RPC 对应一个 Stream；数据切分为二进制帧，依靠 stream‑id 区分不同请求。注意这不是 epoll 那种 IO 多路复用。
