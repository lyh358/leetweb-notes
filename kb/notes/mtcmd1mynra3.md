## =={pink}8. 工厂模式在项目中怎样使用==

=={green}**工厂模式**：把「**对象的创建**」和「**对象的使用**==」=={green}分离开。使用者不用 new 具体子类，交给工厂类去创建对象。==

> **普通写法**：使用者自己 `new CpuCollector()` →使用者要知道类名、头文件、构造细节。
> 
> 
> **工厂模式**：使用者告诉工厂 “我要 cpu 采集器”，工厂帮你 new 出来返回接口基类指针。使用者完全不用碰子类。

**=={yellow}项目里的工厂类==**：`CollectorFactory`
**=={yellow}采集器有一堆子类==**：`CpuCollector`、`MemoryCollector`、`NetworkCollector`、`IrqLoadCollector`，全部继承抽象基类 `IMetricCollector`。

### =={pink}❌不使用工厂（坏写法）==

=={yellow}Monitor Agent（使用者）直接 new 各个具体类==：

```cpp
// Agent代码里面直接写死new具体子类
auto cpu = std::make_unique<CpuCollector>();
auto mem = std::make_unique<MemoryCollector>();
collectors.push_back(std::move(cpu));
collectors.push_back(std::move(mem));
```

**缺点：**

1. Agent 要包含每一个采集器的头文件，强依赖所有具体实现类。
2. 如果新增`DiskCollector磁盘采集器`，**必须修改 Agent 的源码**，加一行 new 代码。
3. 如果想要配置文件控制开启哪些采集项，代码很难做。

### =={pink}✅使用工厂模式（项目中的做法）==

`CollectorFactory`就是工厂类。=={yellow}内部维护一张**映射表**：**字符串名字 → 创建对象的函数**==。

```bash
"cpu"       → 创建 CpuCollector
"memory"    → 创建 MemoryCollector
"network"   → 创建 NetworkCollector
"irq_load"  → 创建 IrqLoadCollector
```

=={yellow}Agent **只调用工厂**，**传入字符串名字**，**拿基类智能指针**：==

```cpp
// Agent只知道工厂、基类IMetricCollector，不知道CpuCollector这些子类
auto collector = CollectorFactory::Instance().Create("cpu");
if(collector){
    collectors.push_back(std::move(collector));
}
```

**=={pink}新增磁盘采集器的时候：==**

1. =={yellow}写==`DiskCollector`===={yellow}={yellow}={yellow}**继承**=={yellow}`IMetricCollector`=={yellow}，实现 Collect ()==
2. =={yellow}在工厂**注册**== `"disk"` =={yellow}和它的构造函数==
3. **=={yellow}Agent 完全不用改一行代码！==** =={yellow}配置传字符串==`"disk"`=={yellow}，工厂就生成对应的对象。==

> =={green}这就是文档说的==**=={green}可插拔==**=={green}：新增组件，不需要修改业务主流程。==

## 核心好处（面试必背）

1. **=={yellow}解耦==**：使用者（Agent）只依赖抽象接口`IMetricCollector`，不依赖具体子类。
2. **=={yellow}易扩展==**：增加新采集器，只需要实现子类、注册工厂，不用修改 Agent 主逻辑。
3. **=={yellow}便于配置==化**：可以从配置文件读需要启用的采集器名字字符串，动态创建对象，不用改代码。
4. =={yellow}对象创建==逻辑=={yellow}集中==在工厂一处=={yellow}管理==。

## =={pink}区分：简单工厂、工厂方法、抽象工厂==

=={yellow}你的项目用的是==**=={yellow}简单工厂==**（=={green}一个工厂类，一个 Create 函数，根据参数创建不同产品==），面试说简单工厂就够，不要硬扯工厂方法 / 抽象工厂，你的项目没有用到。

> 面试官问：你项目里的工厂模式是什么？
> 👉面试口语答案：
> “我在性能采集模块使用了简单工厂模式，实现类是 CollectorFactory。
> 所有采集器都继承 IMetricCollector 抽象接口。Agent 作为使用者，不直接 new 具体的 CPU、内存采集子类，而是传入字符串名称给工厂，由工厂负责创建对应的采集器对象，返回基类 unique_ptr。
> 这样 Agent 只依赖抽象接口，和具体采集实现解耦；后续新增磁盘、GPU 采集器，只需要实现子类并注册进工厂，不需要修改 Agent 的采集主循环，实现组件可插拔。”

## 高频追问

### Q：工厂模式和多态是什么关系？

=={yellow}工厂负责==**=={yellow}生产对象==**=={yellow}；多态负责==**=={yellow}调用对象的行为==**。
工厂产出基类指针，运行时多态调用子类的`Collect()`，两者配合工作。工厂解决对象创建，多态解决对象行为调用。

### Q：工厂模式一定需要继承、虚函数吗？

=={yellow}一般配合抽象接口 / 继承使用==。如果没有统一基类，工厂返回不同类型对象，就没法统一保存到 vector 容器。你的项目中必须配合继承多态。

### Q：简单工厂的缺点？

=={yellow}如果产品越来越多，工厂内部的注册 / 判断逻辑会膨胀==。但对于本项目采集器数量不多，完全够用。

### Q：是运行时生效还是编译期？

=={yellow}运行时==，程序跑起来根据传入字符串动态创建不同对象。对比 =={yellow}SFINAE / 模板偏特化是编译期==。

> 对比记忆：
> SFINAE 模板偏特化：**编译期**选择代码分支
> 工厂模式 + 多态：**运行时**创建对象、调用子类逻辑

### 面试速答

> =={green}工厂模式把对象创建和对象使用分开。Agent只向工厂传入==`cpu`=={green}、==`memory`===={yellow}={green}等名称，工厂负责返回对应的采集器。后续增加新指标时，只需要实现接口并注册到工厂，不需要修改Agent的核心采集循环。==

---

## =={pink}9. 为什么称为“可插拔”==

这里的可插拔不是动态加载`.so`插件，而是**=={yellow}代码结构上的可配置和可扩展==**。

不同采集器都遵循统一接口，Agent可以根据需要选择创建哪些采集器：

```scss
配置采集项名称
        ↓
工厂创建对应采集器
        ↓
加入Agent采集器列表
        ↓
统一调用Collect()
```

因此，=={yellow}可以**选择性启用采集器**，也**可以继续增加新采集器**，而**不需要重写Agent主流程**==。

---

## 10. 完整采集流程

### =={pink}Agent 是干嘛的：==

项目架构是 **Agent（多份，各个机器） + Monitor Center（一份，中心服务）**

- **=={pink}Monitor Agent==：=={green}部署在==每一个被监控的机器人计算板 / Linux =={green}节点==上，是 =={yellow}“本地采集 + 上报==” 的=={green}探针程序==**
- **=={pink}Monitor Center==：=={yellow}中心服务==，=={yellow}接收==所有 Agent 上报的数据，=={yellow}做汇总、缓存，给 Qt== 监控界面提供数据**

> 类比：每台机器上派一个 “现场观察员”，这个观察员就是 Agent；观察员不停采集本机 CPU、内存，源源不断把报告发给总部（Center）。

### =={pink}Agent 做 4 件核心事情：==

1. **=={yellow}初始化采集器==**：通过工厂模式加载需要的指标采集组件（CPU、memory、network 等），可插拔，不用就不加载。
2. **=={yellow}本地采集指标==**：读取`/proc`，计算各项性能。**采集逻辑全部在本机完成，不依赖 ROS**。
3. **=={yellow}封装 Protobuf 消息==**：把原始数据打包成`NodeMetrics`protobuf 结构体。
4. **=={yellow}gRPC 流式上报==**：建立一次长连接，持续不断发送监控数据，不需要每次上报都新建 RPC 请求。

> ⚠️Agent**=={green}不做数据分析、不做告警判断==**；健康等级、告警阈值判断是交给中心 Monitor =={yellow}Center 完成==。Agent 只负责采集 + 上报原始数据。

### =={pink}和 Monitor Center 的交互（gRPC 流式）==

1. Agent 端（客户端）：调用`ReportMetrics(stream NodeMetrics)` **客户端流 RPC**

- =={yellow}**建立一次 gRPC 连接，循环不断往流里面写监控消息**==；
- 一个节点只需要一条长流，周期性源源不断吐出本机指标。

1. Center 端（服务端）接收各个 Agent 流过来的数据，根据`node_id`区分不同机器，存入 map 做缓存，判断健康状态。
2. Qt 监控界面再通过**服务端流 SubscribeMetrics**从 Center 拉取全部节点实时数据渲染界面。

---

```cpp
Monitor Agent程序启动
    ↓
调用CollectorFactory工厂，创建CPU/内存/网络等采集器（unique_ptr管理，存入vector容器）
    ↓
开启循环，按采样周期周期性工作
    ↓
遍历全部采集器，多态调用Collect()
    ↓
读取本机/proc虚拟文件系统，计算CPU使用率、内存、网络速率等指标
    ↓
把采集结果填充进Protobuf消息 `NodeMetrics`，打上本节点node_id、时间戳
    ↓
通过gRPC客户端流ReportMetrics，持续把NodeMetrics上报给Monitor Center
    ↓
睡眠，等待下一个采样周期，循环往复
```

---

## 11. A2口语化回答

> 为了监控机器人各个Linux计算节点，我需要先采集CPU、内存和网络等本机指标。这些数据主要来自Linux的`/proc`虚拟文件系统，它是内核向用户态暴露系统运行状态的一组接口。比如我从`/proc/stat`读取CPU累计时间，从`/proc/meminfo`读取内存信息，从`/proc/net/dev`读取网络累计收发字节数。CPU利用率和网络速率都不是直接读取出来的，而是通过前后两次采样的差值计算。
> 
> 
> 在代码结构上，我没有把所有采集逻辑都写进Agent，而是定义了一个`IMetricCollector`抽象接口。CPU、内存、网络和负载采集器分别继承这个接口，并重写`Collect()`。Agent使用基类指针统一保存这些对象，通过虚函数多态调用各自的采集逻辑。
> 
> 
> 对象由`CollectorFactory`根据名称创建，并以`unique_ptr`返回。这样既能明确对象所有权、自动释放内存，也把采集器的创建和使用解耦。后续增加磁盘或GPU指标时，只需要新增一个采集器并注册到工厂，不需要修改Agent的核心采集流程。这就是简历中所说的基于继承、多态、智能指针和工厂模式设计可插拔性能采集框架。

---

# A3：基于gRPC和Protobuf构建分布式监控链路

## =={pink}1. 为什么还需要gRPC==

A2已经能够从单台Linux计算节点采集CPU、内存和网络指标，=={yellow}但数据仍然留在本机==。

当系统中存在多个计算节点时，=={yellow}如果逐台登录查看，无法形成统一的监控视图==。因此，需要建立一条跨节点通信链路：

```markdown
各节点采集性能数据
        ↓
持续上报到中心节点
        ↓
中心汇总不同节点的数据
        ↓
监控界面订阅并实时展示
```

=={green}这条链路需要满足两个特点：==

- =={yellow}节点能够**持续上报指标**，而**不是每次重新建立请求**==；
- =={yellow}**中心能够持续向监控端**推送最新数据。==

因此，项目选择了gRPC的流式通信。

---

## =={pink}2. gRPC是什么==

=={yellow}gRPC是==**=={yellow}Google 开源的高性能 RPC（远程过程调用）框架==**=={yellow}。==

普通函数调用发生在同一个程序中：

```ini
result = GetMetrics();
```

**=={yellow}远程过程调用==**=={yellow}则允许客户端**像调用函数一样，请求另一台计算机上的服务**：==

```scss
客户端调用ReportMetrics()
            ↓ 网络
服务端执行ReportMetrics()
```

**=={yellow}gRPC通常使用==`.proto`**=={yellow}同时定义：==

- 通信的数据结构；
- 服务提供哪些接口；
- 每个接口接收什么参数；
- 每个接口返回什么结果。

然后通过`protoc`生成：

- 客户端Stub；
- 服务端Service接口；
- Protobuf消息类；
- 消息序列化和反序列化代码。

因此，开发者主要关注业务接口和数据，不需要自己处理TCP连接、消息拆包等底层细节。

---

## =={pink}3. Protobuf和gRPC是什么关系==

这两个概念容易混淆，可以这样区分：

- **Protobuf负责描述和编码数据**；
- **gRPC负责把数据从一个节点传到另一个节点，并组织远程服务调用**。

=={yellow}可以类比为：==

```undefined
Protobuf：规定信件的内容格式
gRPC：负责投递信件
```

项目=={yellow}使用==**`NodeMetrics`=={yellow}表示一个节点某个时刻的性能快照==**，其中包含：

- 节点ID和时间戳；
- 各CPU核心占用率；
- 内存和交换分区；
- 网络收发速率；
- 系统负载与中断；
- 节点健康等级。

=={yellow}**同一份Protobuf定义**可以被**Agent、Center和监控端**共同使用==。

---

## 4. 为什么监控链路选择gRPC

项目没有继续使用ROS Topic完成性能监控，主要有三个原因。

### =={pink}4.1 致命依赖：ROS Topic 强依赖 roscore==

- =={yellow}如果==`roscore`=={yellow}挂掉，==**=={yellow}全部 Topic 通信直接瘫痪==**=={yellow}，监控链路也跟着失效。==
- =={yellow}监控的本意：==**=={yellow}当 ROS 业务系统崩溃的时候，我依然要采集机器 CPU 内存，排查为什么 ROS 挂掉==**=={yellow}。==

> 如果监控链路本身跑在 ROS Topic 上，一旦 ROS 崩，监控直接失效，就**=={green}失去故障排查最重要手段。==**

### =={pink}4.2 gRPC原生支持Protobuf==

A2采集得到的数据已经封装为`NodeMetrics`，gRPC可以直接使用这份消息定义，不需要再编写一套转换逻辑。

### =={pink}4.3 gRPC支持流式通信==

节点性能指标是周期性产生的连续数据。gRPC支持在一次RPC中持续发送多条消息，适合这种监控场景。

所以这里不是因为ROS不能跨机器通信，而是：

> gRPC与Protobuf结合自然，并且提供了清晰的客户端流、服务端流和请求响应接口，更适合独立的集中监控链路。

---

## 5. Agent和Center分别负责什么

整个监控系统采用Agent/Center结构。

### 5.1 Monitor Agent

Agent运行在每个被监控的Linux计算节点上，主要负责：

1. 根据配置创建CPU、内存和网络采集器；
2. 按照采样周期执行采集；
3. 将结果填充到`NodeMetrics`；
4. 添加节点ID和时间戳；
5. 通过gRPC持续上报到Center。

每个Agent通过`node_id`标识自己，例如：

```css
robot-A
robot-B
robot-C
```

### 5.2 Monitor Center

Center运行在中心节点上，主要负责：

1. 接收不同Agent上报的指标；
2. 根据`node_id`区分不同计算节点；
3. 缓存每个节点的最新性能快照；
4. 根据CPU和内存阈值判断健康状态；
5. 向监控端持续推送最新指标。

中心端的数据结构可以理解为：

```css
robot-A → 最新NodeMetrics
robot-B → 最新NodeMetrics
robot-C → 最新NodeMetrics
```

---

## 6. 客户端流式上报是怎么回事

项目中的上报接口定义为：

```scss
rpc ReportMetrics(stream NodeMetrics)
    returns (ReportAck);
```

`stream NodeMetrics`表示客户端可以在一次RPC调用中连续发送多条`NodeMetrics`。

通信过程是：

```markdown
Monitor Agent                    Monitor Center
     │                                 │
     │──── 建立ReportMetrics流 ────────▶│
     │──── 第一次性能快照 ──────────────▶│
     │──── 第二次性能快照 ──────────────▶│
     │──── 第三次性能快照 ──────────────▶│
     │              ……                 │
     │──── 结束数据流 ─────────────────▶│
     │◀──────── 返回ReportAck ─────────│
```

Agent内部按照以下流程运行：

```scss
按照采样周期循环
        ↓
创建一条NodeMetrics
        ↓
遍历所有采集器
        ↓
填充CPU、内存和网络指标
        ↓
writer->Write(metrics)
        ↓
等待下一个采样周期
```

### 为什么使用客户端流

如果每产生一次指标就创建一次普通RPC，会不断重复请求建立和结束过程。

客户端流允许Agent建立一次调用后持续发送数据，更符合“节点不断上报监控指标”的场景。

### 面试速答

> 客户端流表示客户端在一次RPC中连续发送多条消息，最后服务端再返回一次结果。在项目中，每个Agent持续产生`NodeMetrics`，所以使用客户端流可以保持一条长期上报通道，而不需要每个采样周期重新调用一次RPC。

---

## 7. 服务端流式订阅是怎么回事

项目中的订阅接口定义为：

```scss
rpc SubscribeMetrics(SubscribeRequest)
    returns (stream NodeMetrics);
```

它表示客户端先发送一次订阅请求，Center随后持续返回多条监控数据。

```css
监控端                            Monitor Center
   │                                    │
   │──── 请求订阅指定节点 ─────────────▶│
   │◀──── robot-A最新指标 ──────────────│
   │◀──── robot-B最新指标 ──────────────│
   │◀──── robot-A下一次指标 ────────────│
   │                 ……                 │
```

订阅请求中可以指定：

- 需要查看哪些`node_id`；
- 推送间隔；
- 节点列表为空时订阅全部节点。

Center根据请求，从缓存中选择对应节点的数据，再持续写入服务端流。

### 为什么使用服务端流

Qt监控端需要持续获得最新指标。如果使用普通请求响应，就需要反复轮询Center。

服务端流允许监控端发送一次订阅请求后，由Center持续推送数据，更适合实时监控界面。

### 面试速答

> 服务端流表示客户端请求一次，服务端可以持续返回多条消息。项目中监控界面需要不断获取各节点最新状态，所以Center通过服务端流持续推送指标，避免监控端频繁发起轮询请求。

---

## 8. Center怎样处理多个节点的数据

多个Agent可能同时向Center上报指标，因此Center需要维护共享的节点状态表：

```cpp
std::map<std::string, NodeMetrics> latest_;
```

其中：

- Key是`node_id`；
- Value是该节点最新的`NodeMetrics`。

Center读取到一条消息后：

1. 计算节点健康等级；
2. 根据`node_id`找到对应位置；
3. 用最新指标覆盖旧指标；
4. 供订阅端读取和推送。

因为多个gRPC请求可能并发读写`latest_`，如果不进行同步，可能出现数据竞争。

项目使用：

```cpp
std::mutex
std::lock_guard<std::mutex>
```

在更新或读取共享数据时加锁。

`lock_guard`离开作用域后会自动释放锁，可以避免异常路径或提前返回导致忘记解锁。
