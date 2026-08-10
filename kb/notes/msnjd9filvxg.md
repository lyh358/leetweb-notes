# ROS-Protobuf 分布式通信与性能监控：项目全景与技术深挖

## 1. 项目定位与真实完成度

项目目标是面向机器人/自动驾驶多计算节点，统一业务消息和监控指标的数据契约：同一份 Protobuf 消息既可以经 ROS Topic 在节点内传输，也可以经 gRPC 在节点间上报。

必须把完成度拆成三层：

1. `[源码确认]` ROS-Protobuf：通过 traits 和 Serializer 模板特化，让 roscpp 接受 Protobuf 生成类；有 talker/listener 示例。
2. `[源码确认]` Qt 本地监控：读取 `/proc` 展示本机 CPU、内存、网络和进程，使用工作线程、信号槽和 qcustomplot。
3. `[原型]` gRPC 分布式层：有 Protobuf 契约、采集器、Agent、Center 和 CMake；当前备份未编译，Qt 未接入 gRPC，配置下发未形成闭环。

因此准确的一句话是：

> 我基于已有的 ROS-Protobuf 通信层和 Qt 本地监控，设计统一 NodeMetrics 契约并实现 gRPC Agent/Center 原型，把两个子系统向“单一数据契约、多传输后端”的架构演进；完整集群 UI 和配置反向通道仍是后续工作。

## 2. 架构和边界

### 2.1 目标架构

```text
节点内业务通信                         跨节点监控
感知/规划/控制                         Linux /proc
      │                                   │
NodeMetrics/其他 Proto                Collectors
      │                                   │
ROS traits + Serializer                  Agent
      │                                   │ client-streaming
ROS TCPROS/UDPROS                     gRPC/HTTP2
                                          │
                                       Center
                                      latest_缓存
                                          │ server-streaming
                                       Qt 集群大屏（目标，未接通）
```

### 2.2 “统一”的准确含义

统一的是消息 schema 和 Protobuf 序列化语义，不是统一传输协议：

- ROS 通道仍由 ROS Master 发现、TCPROS/UDPROS 传输和 ROS callback queue 调度；
- gRPC 通道使用 HTTP/2、RPC 方法和 stream；
- 两者都使用 Protobuf 生成类与 wire format，因此字段只定义一次。

当前 ROS 示例实际发送的是 `PublishInfo`，顶层 `NodeMetrics` 进入 ROS 通道是架构设计，工作区没有对应 talker/listener 实例。不要说已经用同一 NodeMetrics 完成双通道端到端验证。

## 3. ROS-Protobuf 通信层

### 3.1 原生 ROS 为什么不直接接受任意类

`ros::Publisher::publish<T>` 在编译期需要两类能力：

- `ros::message_traits`：`DataType<T>`、`MD5Sum<T>`、`Definition<T>`、`IsMessage<T>` 等，参与类型识别和连接握手；
- `ros::serialization::Serializer<T>`：`write/read/serializedLength`，负责把对象变成字节并还原。

原生 `.msg` 生成代码提供这些特化，普通 Protobuf 类没有，因此直接 publish 会编译失败。

### 3.2 SFINAE 如何只匹配 Protobuf

所有完整 Protobuf C++ 消息继承 `google::protobuf::Message`。代码使用：

```cpp
template <typename T>
struct DataType<T, typename std::enable_if<
    std::is_base_of<google::protobuf::Message, T>::value>::type> { ... };
```

推导过程：

1. `is_base_of` 在编译期判断类型；
2. 条件为真时 `enable_if<true>::type` 存在，偏特化有效；
3. 为假时替换失败；
4. SFINAE 规定替换失败不是整个编译错误，该候选退出，ROS 原生类型继续匹配自己的实现。

这使扩展对上层透明，也避免把所有类型错误地当成 Protobuf。

### 3.3 traits 每个字段的作用

- `DataType`：当前实现返回 `pb_msgs/` + descriptor 的短消息名；发布订阅双方用它识别类型。
- `MD5Sum`：当前固定返回 `proto_md5`；能让所有 Protobuf 端握手通过，但失去 schema 强校验。
- `Definition`：通过 descriptor 反射生成文件名、full name 和 DebugString。
- `HasHeader=False`：没有 ROS `std_msgs/Header` 约定。
- `IsFixedSize=False`：Protobuf含 string/repeated/可选字段，序列化长度运行时决定。
- `IsMessage=True`：告诉 ROS 可作为消息。

风险：`DataType` 用短名可能发生不同 package 同名冲突，生产化应使用 `descriptor()->full_name()`；MD5 应根据规范化 `.proto` descriptor set 计算稳定哈希。

### 3.4 Serializer 的字节布局

写路径：

```text
T对象 → SerializeToString → pb bytes
     → uint32 length → 写4字节长度 → 写body
```

读路径反向执行：读 4 字节长度、取 body、`ParseFromString`。

`serializedLength=4+body_size` 必须与 write 完全一致，否则 ROS 预分配 buffer 大小错误。

需要准确解释：ROS transport 本身已有消息边界，这个内部 4 字节前缀不是 TCP 粘包的唯一解决方案，而是自定义 payload framing；由于收发 Serializer 对称，它可以工作，但在 ROS 消息边界内有一定冗余。生产实现还要校验长度上限、`SerializeToString/ParseFromString` 返回值，并处理超过 `uint32_t` 或恶意长度。

### 3.5 上层使用方式

```cpp
ros::Publisher pub = nh.advertise<PublishInfo>("/Sorbai", 1000);
PublishInfo msg;
msg.set_name("sorbai");
pub.publish(msg);
```

对业务方仍是标准 advertise/publish/subscribe；扩展点全部隐藏在编译期 traits 和序列化层，这正是中间件的价值。

### 3.6 现有线程安全问题

现有文档说 `static string + atomic<bool>` 保证 `Definition` 一次构建线程安全，这不准确。两个线程可能同时看到 flag=false 并同时 append 同一个 `std::string`，产生数据竞争。应改为 C++11 线程安全的函数内 static 初始化 lambda，或 `std::call_once`。

`DataType::value()` 也在每次调用给 static string 重新赋值，存在并发写；应在定义时一次初始化为常量 string。

## 4. Linux 指标采集框架

### 4.1 接口、策略和工厂

`IMetricCollector` 提供 `Collect(NodeMetrics*)` 和 `Name()`。CPU、内存、网络、IRQ/Load 分别实现接口；Agent 保存 `vector<unique_ptr<IMetricCollector>>` 并依次采集。

工厂把字符串映射到 creator lambda，返回 `unique_ptr`。目的：主循环依赖抽象，新指标只需新增类并注册。这里同时体现 Strategy（各采集器是可替换采样策略）和 Simple Factory/Registry。

`unique_ptr` 表达 Agent 对采集器的唯一所有权，基类虚析构保证通过接口指针释放派生对象。

### 4.2 CPU `/proc/stat`

`/proc/stat` 的 `cpuN` 字段是开机累计 jiffies，不是瞬时百分比。代码保存每核上次 `{idle,total}`：

```text
idle_all = idle + iowait
nonidle  = user + nice + system + irq + softirq
total    = idle_all + nonidle
usage    = (Δtotal - Δidle) / Δtotal × 100%
```

首次采样没有前值，因此输出 0；第二次以后才有意义。当前实现未计入 steal，和部分系统工具可能存在差异；生产化要处理 CPU 热插拔、计数器回绕、解析异常。

### 4.3 内存 `/proc/meminfo`

读取 `MemTotal`、`MemAvailable`、`SwapTotal`、`SwapFree`：

```text
used = MemTotal - MemAvailable
swap_used = SwapTotal - SwapFree
```

使用 `MemAvailable` 比简单的 `total-free` 更符合 Linux 可回收 cache/buffer 语义。

### 4.4 网络 `/proc/net/dev`

汇总除 `lo` 外所有网卡 RX/TX 累计字节，并用 `steady_clock` 时间差计算 B/s：

```text
rate = (current_bytes - previous_bytes) / Δt
```

`steady_clock` 不受系统时间校准影响，适合计算间隔。首次速率为 0。网卡重置导致 current < previous 时，无符号减法会下溢，当前代码未处理；多网卡汇总也无法定位具体接口。

### 4.5 IRQ 和 Load

`/proc/loadavg` 提供 1min/5min load average；`/proc/stat` 的 `intr`、`softirq` 是累计计数。当前 schema 只传累计值，没有计算每秒速率；面试中不要把它直接说成“当前中断负载百分比”。

## 5. Protobuf 数据契约

`NodeMetrics` 包含 node_id、毫秒时间戳、多核 CPU、内存、网络、IRQ/Load 和 Health。

设计点：

- `repeated CpuCore` 适配不同核数；
- 速率与累计网络字节并存，分别服务实时观察和统计；
- health 由中心根据全局规则填充，Agent 只上报原始指标；
- proto3 字段号是 wire contract，发布后不能复用删除字段号；新增字段保持新旧端兼容，删除应 `reserved`。

Protobuf 未知字段可以被新旧版本跳过，是演进优势；但语义不兼容的字段改变仍需版本治理，不能只依赖编码格式。

## 6. gRPC 服务原型

### 6.1 三个 RPC

```proto
rpc ReportMetrics(stream NodeMetrics) returns (ReportAck);
rpc SubscribeMetrics(SubscribeRequest) returns (stream NodeMetrics);
rpc Configure(ConfigRequest) returns (ConfigAck);
```

- ReportMetrics：client-streaming。每个 Agent 建一条长流，连续 Write；流结束后 Center 返回一次 Ack。
- SubscribeMetrics：server-streaming。客户端发一次订阅条件，Center 周期推送各节点最新快照。
- Configure：unary。一问一答。

为什么没有直接用双向流：原型把上报、订阅和配置职责分开，接口易理解。若要真正让 Center 实时向 Agent 下发配置，需要 Agent 建立反向订阅或改为双向流；当前 Configure 只把配置存进 Center 的 map，Agent 不会收到。

### 6.2 Agent 数据流

解析 `--center`、`--node-id`、`--interval` → 建立 insecure channel/stub → 工厂创建四个采集器 → 创建 ClientWriter → 循环填充 NodeMetrics、Write、sleep。

缺口：没有参数异常处理、断线重连/退避、截止时间、TLS、流控观测、配置更新、信号退出和本地缓冲。长流断开后进程直接结束。

### 6.3 Center 并发和缓存

同步 gRPC Server 会并发处理不同 RPC。`ReportMetrics` 写 `latest_[node_id]`，`SubscribeMetrics` 读该 map，因此用 mutex 保护。

当前 `SubscribeMetrics` 在持锁期间调用 `writer->Write`，网络慢时会长时间阻塞 ReportMetrics。更好的实现是锁内复制满足条件的快照，解锁后逐个 Write；节点规模增大后可用 shared_mutex、分片锁或每订阅者队列。

### 6.4 健康判定

当前只取最大单核 CPU 和内存占用：≥90 critical，≥75 warning，否则 healthy。它没有持续时间、滞回、网络/中断规则，容易瞬时抖动。生产化需要可配置阈值、连续 N 次/时间窗、恢复阈值和告警抑制。

## 7. Qt 层的准确表述

已有 Qt 工程是本地系统监视器：工作线程采 `/proc`，信号槽更新 QWidget/QTableView/qcustomplot。它能证明 Qt UI、线程和 Linux 本地采集经验。

目标集群大屏应新增：gRPC Subscribe 客户端线程 → NodeMetrics 快照/历史模型 → signal 跨线程通知 UI → QAbstractTableModel `beginResetModel/dataChanged` → 曲线和告警视图。

当前没有这段 glue code，因此不能说“Qt 已经通过 gRPC 实时展示多节点”。

## 8. 构建与部署

gRPC CMake 原型将两份 `.proto` 的生成纳入 build：`protoc --cpp_out` 生成消息，`grpc_cpp_plugin` 生成服务 stub；生成代码编为 `monitor_proto`，采集器编为静态库，Agent/Center 分别链接。

两个原始子系统有各自 Dockerfile/脚本，顶层 `docker/` 只有说明，没有 compose。因此当前是“具备子系统容器环境和统一编排设计”，不是“已经用 compose 一键部署完整集群”。

## 9. 核心选型权衡

### Protobuf vs ROS msg

Protobuf 跨语言、schema 演进和非 ROS 生态互操作更好；ROS msg 与 ROS 工具链原生兼容、MD5/Definition 语义完整。项目选择 Protobuf，是为了跨 ROS/gRPC 统一，但付出了适配和工具兼容成本。

### gRPC vs MQTT

gRPC 强类型、HTTP/2、流式 API、内部服务调用自然；MQTT broker 解耦、弱网和海量设备更成熟。机器人局域网内部监控可选 gRPC；大规模不稳定网络可能 MQTT 更合适。

### 修改 roscpp vs Bridge Node

修改 roscpp 对上层透明、少一次桥接拷贝，但侵入性高、升级维护困难。独立 bridge 更易维护和隔离，但多一个进程、协议转换和延迟。生产化可能优先非侵入 adapter/type adapter。

## 10. 测试计划

- traits/serialization：原生 msg 不受影响、不同 Proto round-trip、空/大消息、损坏长度、schema不一致；
- ROS：talker/listener 频率、消息大小、队列溢出、TCPROS/UDPROS；
- collectors：与 `top/free/sar` 同时间窗对照、网卡重置、CPU热插拔、缺失 `/proc`；
- gRPC：2/10/100 Agent、断网重连、慢订阅者、背压、Center 重启；
- 并发：ThreadSanitizer 检查 traits static 与 Center map；
- 集成：同一个 NodeMetrics 在 ROS 和 gRPC 两条通道 round-trip，并校验字段一致；
- 部署：容器权限、宿主 `/proc` 可见性、网络 namespace 对指标的影响。

## 11. 生产化路线

1. 先修复模板 static 线程安全、真实 schema 哈希和输入校验；
2. 完成 Linux/WSL 构建和单元测试；
3. Agent 加重连、退避、TLS、buffer 和优雅退出；
4. 实现 Agent 配置反向通道；
5. 将 Qt gRPC subscriber 真正接入 Model/View；
6. 增加时序库存储、告警规则、历史查询；
7. 完成 Docker Compose、压测和故障注入。

