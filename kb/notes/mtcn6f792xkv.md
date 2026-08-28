## =={pink}9. 节点健康状态怎样判断==

Center接收到指标后，会根据CPU和内存使用情况进行=={yellow}简单的阈值判断==：

```markdown
最高CPU或内存占用 < 75%
        ↓
HEALTHY

最高CPU或内存占用达到75%
        ↓
WARNING

最高CPU或内存占用达到90%
        ↓
CRITICAL
```

这个功能的目的不是完成复杂的故障诊断，而是将多项原始指标转换成一个直观的节点健康等级，便于监控端快速识别异常节点。

---

## 10. 配置接口的作用

项目还定义了普通请求响应接口：

```scss
rpc Configure(ConfigRequest)
    returns (ConfigAck);
```

配置内容包括：

- 目标节点ID；
- 采样周期；
- 需要启用的采集器。

普通请求响应适合“发送一次配置、返回一次处理结果”的场景，不需要使用流式RPC。

当前原型完成了配置接口和Center侧配置保存，为后续将采样周期及采集项动态同步给Agent预留了入口。

---

## 11. =={pink}三种RPC==为什么这样选择

| 接口 | 通信形式 | 使用原因 |
| --- | --- | --- |
| `ReportMetrics` | =={yellow}客户端流== | =={yellow}Agent持续上报==多条指标 |
| `SubscribeMetrics` | =={yellow}服务端流== | =={yellow}Center持续==向监控端=={yellow}推送==指标 |
| `Configure` | =={yellow}普通请求响应== | =={yellow}一次发送配置，一次返回结果== |

记忆方式可以压缩为：

> 上报是客户端不断发，所以用客户端流；展示是服务端不断推，所以用服务端流；配置是一问一答，所以用普通RPC。

---

## 12. 完整通信流程

```markdown
各Linux计算节点启动Monitor Agent
        ↓
工厂创建CPU、内存、网络等采集器
        ↓
周期性生成NodeMetrics
        ↓
通过ReportMetrics客户端流持续上报
        ↓
Monitor Center接收数据
        ↓
按照node_id缓存最新快照
        ↓
计算节点健康状态
        ↓
通过SubscribeMetrics服务端流推送
        ↓
监控端获得最新节点数据
```

---

## 13. A3口语化回答

> 完成本地性能采集以后，我还需要把多个Linux节点的数据集中起来，所以设计了Monitor Agent和Monitor Center两类角色。Agent部署在各个计算节点上，周期性调用A2中的采集器，将CPU、内存和网络等指标封装成统一的`NodeMetrics`；Center负责接收不同节点的数据，根据`node_id`缓存每个节点的最新状态，并进行简单的健康等级判断。
> 
> 
> 通信部分使用gRPC，因为它原生支持Protobuf，并且支持流式RPC。Agent上报使用客户端流，也就是建立一次`ReportMetrics`调用后，持续发送多条性能快照；监控端订阅使用服务端流，也就是发送一次订阅请求后，由Center持续返回最新指标。配置操作则是一问一答，所以使用普通RPC。
> 
> 
> 多个Agent和订阅端可能并发访问Center中的节点状态表，因此我使用`mutex`和`lock_guard`保护共享数据。这样就形成了从节点采集、流式上报、中心汇总到监控订阅的完整链路。

## 14. 高频追问速答

### gRPC和Protobuf有什么区别？

> Protobuf负责定义和序列化数据，gRPC负责远程调用和网络传输。项目中`NodeMetrics`由Protobuf定义，再通过gRPC在Agent和Center之间传输。

### 为什么上报使用客户端流？

> 因为一个Agent会持续产生多条监控数据。客户端流允许它在一次RPC中连续发送消息，避免每个采样周期都重新发起请求。

### 为什么展示使用服务端流？

> 监控界面只需要订阅一次，之后由Center持续推送最新数据，比反复轮询更符合实时监控场景。

### 为什么不用ROS Topic上报监控数据？

> ROS也能跨节点通信，但监控属于独立的基础设施链路。gRPC不依赖ROS运行环境，并且原生兼容Protobuf和流式服务接口，更便于连接中心服务和非ROS监控端。

### 为什么需要互斥锁？

> 多个Agent可能同时更新节点状态，订阅端也可能同时读取。如果不加锁，会产生数据竞争，所以使用互斥锁保护共享的节点快照表。

---

# 开发工具

## =={pink}1.CMake（构建编译工具）==

CMake 是**C/C++ 跨平台构建系统**，它不直接编译代码；读取`CMakeLists.txt`配置文件，生成 Makefile/Ninja 构建脚本，再调用 gcc/g++ 完成编译、链接。

> 类比：做饭的菜谱，=={yellow}告诉编译器：哪些源文件要编译、要找哪些库、生成什么程序。==

### =={pink}在本项目里面做了什么==

**项目里面有多个模块**：ROS‑Protobuf 兼容层、性能采集库、Agent 程序、Center 服务、Qt 界面、Protobuf/gRPC 代码生成。

1. **寻找依赖库**

查找：`roscpp`、`Protobuf`、`gRPC`、Qt5、Boost 等第三方库头文件和库文件路径。

```
find_package(roscpp REQUIRED)
find_package(Protobuf REQUIRED)
find_package(gRPC REQUIRED)
find_package(Qt5 COMPONENTS Core Gui Widgets REQUIRED)
```

1. **编译 proto 文件，自动生成 C++ 代码**
调用`protoc`、grpc_cpp_plugin，读取`.proto`，生成 Protobuf 消息类、gRPC 服务 Stub 代码。

> 业务消息、监控`NodeMetrics`全部在这里自动生成，不用手写。

1. **划分模块，编译库和可执行程序**

- 编译静态库：ros_protobuf_adapter（ROS‑Protobuf 兼容层）、collector_lib（可插拔采集框架）
- 编译可执行文件：
  - `agent_node`：Monitor Agent 可执行程序
  - `center_server`：Monitor Center gRPC 服务端
  - `qt_monitor_gui`：Qt 监控界面程序

1. **设置头文件路径、链接库、编译选项**。
2. **配合 ROS，支持 catkin 编译**（ROS Noetic 基于 cmake）。

# R：项目结果

## 1. 功能结果

项目最终形成了业务通信和节点监控两部分功能。

在业务通信侧，完成了ROS消息类型识别和序列化机制的扩展，使继承自`google::protobuf::Message`的消息类型能够直接使用ROS原有的`publish()`和`subscribe()`接口，并通过发送端和接收端验证了Protobuf消息的正常收发，同时保留了原生ROS Msg的通信方式。

在性能监控侧，完成了CPU、内存、网络和系统负载等Linux指标采集器，并通过统一接口、继承、多态、智能指针和工厂模式组织成可扩展的采集框架。在此基础上，搭建了Monitor Agent和Monitor Center的gRPC通信原型，实现了节点指标的客户端流式上报、中心端状态缓存和健康判断，以及面向监控端的服务端流式订阅接口。

## 2. 工程结果

项目形成了相对完整的模块化工程结构：

- 使用Protobuf统一定义业务消息和监控指标；
- 使用CMake组织Protobuf、gRPC和C++模块构建；
- 使用Docker和Shell整理ROS Noetic运行环境；
- 保留Qt本地性能监控和数据展示模块；
- 将ROS业务通信与Linux节点监控整合到同一套项目架构中。

整个项目验证了“统一数据契约、多种传输方式”的设计思路：

```undefined
Protobuf负责统一数据定义
ROS Topic负责机器人业务通信
gRPC负责跨节点性能监控
```

## 3. 个人收获

通过这个项目，我对ROS不再只停留在Node和Topic的使用层面，而是进一步理解了ROS的Traits、Serialization和消息类型识别机制。

同时，我也将C++模板偏特化、SFINAE、继承、多态、智能指针和工厂模式应用到了具体工程中，并完成了从Linux性能采集、Protobuf数据封装到gRPC流式通信的整体串联。

## R部分口语化回答

> 最终这个项目主要形成了两部分结果。第一部分是ROS-Protobuf兼容层，我通过扩展ROS的Traits和Serializer，让Protobuf消息可以直接使用ROS原来的发布订阅接口，并通过talker和listener验证了消息收发。
> 
> 
> 第二部分是分布式性能监控原型。我完成了CPU、内存、网络和系统负载采集器，并通过继承、多态、智能指针和工厂模式将它们组织成可扩展框架。在此基础上，使用gRPC搭建了Agent和Center，实现节点指标的流式上报、中心缓存、健康判断和订阅接口。
> 
> 
> 这个项目虽然是个人学习项目，没有业务上线和量化指标，但它让我把ROS底层消息机制、Linux性能采集、C++设计方法和gRPC分布式通信串成了一套完整的学习实践。

---

```cpp
#include <type_traits>

#include <boost/type_traits.hpp>

// 主模板：默认不是ROS消息

template<typename T>

struct IsMessage : boost::false_type {};

// -------- SFINAE 偏特化：匹配所有protobuf派生类 --------

template<typename T>

struct IsMessage<T,

    typename std::enable_if<std::is_base_of_v<google::protobuf::Message, T>>::type

> : boost::true_type {};
```
