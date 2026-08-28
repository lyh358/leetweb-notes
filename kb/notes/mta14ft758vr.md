# 自动驾驶机器人分布式通信与监控系统——STAR

> 这个项目不是把ROS消息“转换成”Protobuf，也不是因为Protobuf更流行就把ROS消息替换掉，而是**=={yellow}让ROS通信框架同时兼容ROS Msg和Protobuf两类消息==**，从而=={yellow}**保留ROS生态**==，**=={yellow}**同时**接入更通用的Protobuf数据生态**==。

下面按照“把面试官当作完全不了解这个领域的人”的方式重新展开。

# S：项目背景与应用场景

## 1. 自动驾驶机器人为什么需要通信框架

=={green}一台自动驾驶机器人==**=={green}需要同时完成很多工作==**，例如：

```markdown
摄像头、激光雷达
        ↓
感知识别
        ↓
定位与路径规划
        ↓
车辆运动控制
```

=={green}这些功能通常不会全部写进一个大程序，而是**拆分成多个独立进程**：==

- 感知程序负责识别障碍物；
- 定位程序负责确定机器人位置；
- 规划程序负责计算行驶路径；
- 控制程序负责控制速度和方向。

=={green}这样**拆分以后**，系统更容易**开发和维护**，但也带来了一个问题：==

> =={yellow}**不同程序之间**怎样**交换数据**？==

例如，=={green}**感知模块**识别出一个障碍物后，需要把障碍物的位置、速度和类别发送给**规划模块**。规划模块计算出轨迹后，又要把轨迹发送给**控制模块**。==

因此，自动驾驶机器人=={pink}需要一个负责**组织软件模块**并**帮助它们通信**的**中间件**，**ROS**就是在这个位置出现的。==

---

## 2. ROS在机器人系统中扮演什么角色

=={yellow}ROS==**=={yellow}不是==**=={yellow}类似==**=={yellow}Linux==**=={yellow}或====={yellow}**={yellow}Windows**=={yellow}=={yellow}的====={yellow}**={yellow}传统操作系统**=={yellow}=={yellow}，而是==**=={yellow}运行在Linux之上==**=={yellow}的====={yellow}**={yellow}机器人软件框架**=={yellow}=={yellow}。==

可以把ROS理解成机器人=={yellow}各个软件模块之间的“**通信基础设施**”==。

=={yellow}在**ROS中**，**每个功能程序**称为一个**Node**。==例如：

```undefined
感知Node  ──障碍物信息──▶  规划Node  ──规划轨迹──▶  控制Node
```

=={yellow}ROS提供了**多种通信方式**，其中**最常用**的是**Topic发布/订阅**：==

- =={green}发送数据的节点是**发布者**；==
- =={green}接收数据的节点是**订阅者**；==
- =={green}双方**通过同一个Topic交换数据**。==

=={yellow}发布者不需要知道具体有哪些订阅者，订阅者也不需要直接调用发布者，因此**各个模块**能够保持**松耦合。**==

---

## 3. ROS原生使用什么消息

节点之间不仅要建立连接，还必须约定“发送的数据长什么样”。

例如，一条障碍物消息可能包含：

```undefined
障碍物编号
障碍物类型
位置坐标
移动速度
时间戳
```

=={yellow}ROS**原生**通过**.msg**===={yellow}**文件描述这种数据结构**，这套消息体系通常称为**ROS Msg**或rosmsg。==

=={green}1) 开发者**先编写**==**`.msg`=={green}文件==**

=={green}2) ROS在**编译时**为它生成**C++、Python**等**语言对应的代码**。==

=={green}3) **通信时**，ROS再把**消息对象序列化为二进制数据**，通过**TCP  ROS**等方式**发送给订阅者**。==

因此，需要准确区分两件事：

- `.msg`=={yellow}是**开发阶段**使用的**消息定义文件**；==
- =={yellow}**ROS在网络上传输**的并不是==`.msg`=={yellow}文本，而是**序列化后的二进制数据**。==

ROS Msg本身没有问题。它与ROS Topic、rosbag、rostopic等工具结合得非常好。在一个完全由ROS组成的机器人项目中，直接使用ROS Msg通常是最自然的选择。

---

## 4. 为什么还会出现Protobuf

问题出现在=={pink}机器人系统逐渐变大，并开始**与ROS之外的系统协作**之后==。

例如，一个自动驾驶系统除了ROS模块，可能还包含：

- 独立的C++算法服务；
- Python模型推理程序；
- 使用gRPC的远程服务；
- 云端管理平台；
- 数据记录和分析程序；
- 已经使用Protobuf定义接口的其他模块。

=={pink}这些**拓展系统不一定运行在ROS环境**中==。=={yellow}如果业务数据原本已经使用==`.proto`=={yellow}定义，而**接入ROS时又必须重新定义一份**==`.msg`=={yellow}，就会出现两套数据模型。==

例如，同一条状态消息可能需要这样处理：

```markdown
Status.proto
     ↓ 生成Protobuf对象
Protobuf对象
     ↓ 手动转换
ROS Status.msg对象
     ↓
ROS Topic
```

这样会产生三个**工程问题**：

1. =={yellow}**同一个数据结构**需要**维护**==`.proto`=={yellow}和==`.msg`===={yellow}={yellow}**两份定义**；==
2. **=={yellow}需要为不同消息编写重复的转换代码；==**
3. =={yellow}字段修改时，**两套定义**和**转换逻辑**都要同步更新==。

消息数量较少时影响不明显，但自动驾驶系统中的消息类型很多，长期维护成本会逐渐增加。

---

## 5. Protobuf是什么

=={yellow}Protobuf，也就是==**=={yellow}Protocol Buffers==**=={yellow}，是一种===={yellow}Google 开源，==**=={yellow}跨语言、跨平台==**=={yellow}的====={yellow}**={yellow}结构化数据定义**=={yellow}=={yellow}和====={yellow}**={yellow}二进制序列化**=={yellow}=={yellow}框架===={yellow}。==

> **序列化**：内存中的=={green}结构体 / 对象== → =={green}二进制字节==；
> **反序列化**：二进制字节 → 还原内存结构体 / 对象。

类比：

- =={green}JSON：写一封人可以直接看懂的书信==，带引号、大括号，文本格式，体积大。
- =={green}Protobuf：**电报**，**二进制密文**，**必须有密码本**（==`.proto`=={green}）**才能解析**==；体积很小，解析速度快很多

**=={pink}使用流程：==**
**1）**开发者**使用 .proto 文件定义消息**：

```go
message Object {
    int32 id = 1;
    string type = 2;
    double speed = 3;
}
```

**2）** 然后通过 **protoc 编译器**生成C++、Python、Java等**不同语言对应的代码**。

**3）程序调用生成的代码**：

- **=={yellow}序列化==**=={yellow}：填充结构体 → 变成二进制字节数组；==
- **=={yellow}发送==**=={yellow}：二进制字节通过 MQTT/TCP 发送出去；==
- **=={yellow}接收方反序列化==**=={yellow}：二进制字节 → 还原结构体数据。==

它的**=={pink}价值==**主要体现在四个方面：

- **=={yellow}跨语言==**：同一份`.proto`可以生成多种语言代码；
- **=={yellow}跨系统==**：不依赖ROS，可以用于gRPC、服务端和其他程序；
- **=={yellow}结构紧凑==**：采用字段编号、Varint等二进制编码方式；
- **=={yellow}便于演进==**：在遵守字段编号规则的情况下，=={green}可以新增字段，并保持一定的前后版本兼容性==。

=={green}Protobuf确实被广泛用于**分布式服务**、**云边通信**和部分**自动驾驶平台**==，但不能简单说“Protobuf一定比ROS Msg更好”。

二者的定位不同：

| 对比项 | ROS Msg | Protobuf |
| --- | --- | --- |
| 主要定位 | =={yellow}ROS生态内部通信== | =={yellow}通用数据定义与序列化== |
| 数据定义 | `.msg` | `.proto` |
| ROS工具链适配 | =={yellow}原生支持== | =={yellow}默认不识别== |
| 跨系统复用 | =={yellow}相对受限== | =={yellow}更方便== |
| 版本演进 | ROS1的MD5校验比较严格 | 支持按字段编号兼容演进 |
| 典型场景 | ROS节点通信 | gRPC、分布式系统、跨语言通信 |

所以，=={pink}**引入Protobuf的主要原因**不是盲目追求性能==，而是：

> =={yellow}希望**同一份_消息定义_既能用于ROS节点通信**，**也能复用于gRPC和其他非ROS系统**，减少**重复定义**和**转换代码**。==

=={yellow}至于**序列化速度**和**数据体积**，**Protobuf**在部分数据结构上可能**更有优势**==，但本项目没有进行严格的性能基准测试，因此面试中不声称具体提升比例。

---

## 6. 为什么Protobuf不能直接放进ROS

虽然Protobuf能够生成C++消息类，但ROS默认并不认识这种类型。

=={pink}**ROS**在**允许一种类型通过Topic传输**之前，**需要知道**：==

- =={yellow}它是不是==**=={yellow}合法的ROS消息==**=={yellow}；==
- =={yellow}它的==**=={yellow}消息类型名称==**=={yellow}是什么；==
- =={yellow}它的==**=={yellow}类型校验信息==**=={yellow}是什么；==
- =={yellow}它应该==**=={yellow}怎样计算序列化长度==**=={yellow}；==
- =={yellow}它应该==**=={yellow}怎样序列化和反序列化==**=={yellow}。==

=={pink}**这些规则**分别由ROS的**Traits**和**Serialization机制**负责。==

=={green}ROS原生生成的消息已经具备这些信息，但普通Protobuf类没有==。因此，=={green}把一个Protobuf对象直接传给ROS的==`publish()`=={green}接口，ROS无法按照原来的方式处理。==

这就引出了**=={pink}项目的第一个核心需求==**：

> =={yellow}扩展ROS的==**=={yellow}类型识别==**=={yellow}和====={yellow}**={yellow}序列化机制**=={yellow}=={yellow}，让ROS知道==**=={yellow}怎样识别、打包和还原Protobuf消息==**=={yellow}。==

=={green}完成扩展后==，业务层不需要先把Protobuf转换成ROS Msg，而是可=={green}以直接使用原来的ROS接口==：

```scss
Protobuf对象
      ↓
ROS publish()
      ↓
自定义Traits和Serializer
      ↓
ROS Topic
      ↓
ROS subscribe()
      ↓
Protobuf对象
```

=={green}而ROS原生==`.msg`=={green}仍然可以正常使用==，所以**项目实现的是**：

> =={yellow}ROS**同时兼容ROS Msg和Protobuf消息**，而不是用Protobuf彻底替换ROS Msg。==

---

## 7. 引入Protobuf后带来了什么价值

### =={yellow}保留ROS原有通信方式==

上层节点**仍然使用ROS熟悉的发布订阅接口**，**不需要重新设计通信框架**。

### =={yellow}减少两套消息之间的转换==

已有**Protobuf消息可以直接进入ROS通信链路**，减少重复的`.msg`定义和转换代码。

### =={yellow}便于接入非ROS系统==

**同一份`.proto`可以继续用于**gRPC、云端服务或**其他**语言编写的**程序或系统**。

### =={yellow}改善消息的长期演进能力==

Protobuf使用字段编号描述数据。只要遵守不随意修改旧字段编号等规则，就可以更**平滑地增加新字段**。

因此，这个项目的价值可以概括为：

> =={green}ROS==负责机器人**节点的组织**和**发布订阅**，=={green}Protobuf==负责提供**更通用的数据契约**和**序列化能力**。

---

## 8. 为什么继续引出分布式监控

解决了业务消息通信后，系统还有另一个问题。

=={green}自动驾驶机器人通常包含**多个计算密集型模块**==。感知算法可能持续占用CPU，规划模块需要稳定的内存和计算资源，多个节点之间还需要通过网络传输数据。

=={green}如果==**=={green}这些模块分布在多个Linux计算节点==**=={green}上，仅仅知道ROS Topic是否能够通信还不够，还==**=={green}需要对他们的性能和状态进行监控==**=={green}，==需要知道：

- =={yellow}哪个计算节点==**=={yellow}CPU负载==**=={yellow}过高；==
- =={yellow}哪个节点==**=={yellow}内存==**=={yellow}不足；==
- **=={yellow}网络吞吐==**=={yellow}是否异常；==
- =={yellow}某个节点是否已经==**=={yellow}失联==**=={yellow}；==
- 整个系统当前是否健康。

ROS解决的是机器人业务节点之间的通信问题，但它不是完整的Linux主机监控系统。

逐台登录Linux节点运行`top`或查看`/proc`虽然可行，但无法形成集中视图。因此，项目进一步增加了分布式监控链路：

```bash
各Linux节点
    ↓
读取/proc中的CPU、内存、网络数据
    ↓
Monitor Agent
    ↓
通过gRPC流式上报
    ↓
Monitor Center
    ↓
Qt界面集中展示
```

这里=={yellow}选择**gRPC也能自然复用Protobuf**==：

- =={yellow}Protobuf **定义监控数据**；==
- =={yellow}gRPC 负责**跨节点流式传输**；==
- =={yellow}Qt 负责在**中心端展示节点状态**。==

---

## =={pink}9. 项目最终全景==

整个项目最终形成=={yellow}**两条链路**==。

```markdown
第一条：机器人业务通信

感知/规划/控制节点
        ↓
ROS Msg或Protobuf消息
        ↓
ROS Topic
        ↓
其他业务节点

第二条：Linux节点性能监控

CPU/内存/网络等运行指标
        ↓
Monitor Agent
        ↓
gRPC流式上报
        ↓
Monitor Center
        ↓
Qt监控界面
```

两部分的联系不是“所有数据都使用同一个通信协议”，而是：

> 通过Protobuf提供可复用的数据定义，根据场景分别使用ROS Topic和gRPC完成业务通信与监控通信。

## 最小口语化回答

> 这个项目面向的是自动驾驶机器人场景。一台机器人通常会把感知、定位、规划和控制拆成多个独立程序，ROS在这里相当于软件中间件，通过Node和Topic组织这些模块，并完成发布订阅通信。
> 
> 
> ROS原生使用`.msg`文件定义消息，编译后会生成对应的消息类和二进制序列化代码。对于完全运行在ROS内部的系统，这套机制很好用。但是系统规模扩大以后，部分算法服务、gRPC服务或者其他平台可能已经使用Protobuf定义数据。如果接入ROS时再重新定义一份`.msg`，就需要同时维护`.proto`、`.msg`和两者之间的转换代码。
> 
> 
> 所以我做的第一部分工作，是扩展ROS的Traits和Serialization机制，让ROS能够识别并序列化Protobuf消息。这样Protobuf对象可以直接使用ROS原有的`publish`和`subscribe`接口，同时原生ROS Msg仍然可以正常使用。它的主要价值是减少重复的数据定义和转换，并让同一份`.proto`更方便地复用到ROS和gRPC等不同系统中。
> 
> 
> 在此基础上，我又考虑了多Linux计算节点的运行状态问题。ROS主要解决业务节点通信，但不能代替主机性能监控，所以系统增加了监控Agent，从`/proc`采集CPU、内存和网络数据，通过gRPC流式上报到中心端，并由Qt界面集中展示。
> 
> 
> =={green}最终这个项目形成了两条链路==：=={yellow}一条是==**=={yellow}兼容ROS Msg和Protobuf的ROS业务通信链路==**=={yellow}，另一条是==**=={yellow}基于Protobuf和gRPC的分布式性能监控链路==**=={yellow}。==

这版的核心故事已经完整扣上了：

```undefined
机器人需要模块化
→ 模块之间需要ROS通信
→ ROS原生使用ROS Msg
→ 外部系统已经使用Protobuf
→ 两套消息造成重复定义和转换
→ 扩展ROS使其直接支持Protobuf
→ 系统运行在多个Linux节点上
→ 还需要集中监控节点状态
→ 形成通信与监控一体化项目
```

---

# T：项目任务与系统目标

## =={pink}1. 总体任务==

=={yellow}针对ROS**原生消息难以直接复用Protobuf数据结构**，以及**分布式Linux计算节点缺少集中性能监控**的问题，我计划搭建一套面向自动驾驶机器人的==**=={yellow}分布式通信与监控系统原型==**=={yellow}。==

整个项目需要完成=={pink}**两项核心任务**==：

1. =={yellow}**扩展ROS**的**消息类型识别**和**序列化机制**，使**ROS Topic**在**保留原生ROS Msg通信能力**的同时，也**能够直接传输Protobuf消息**；==
2. =={yellow}构建**Linux节点性能采集**与**集中监控**链路，将多个节点的CPU、内存和网络等指标**汇总到中心端**进行展示。==

可以把项目概括为：

```markdown
机器人业务通信
    ＋
Linux节点性能监控
    ＋
Protobuf统一数据定义
```

## 2. 系统总体架构

整个系统按照使用场景划分为两条主线。

```bash
┌──────────────────── 业务通信链路 ────────────────────┐
│                                                    │
│   感知节点        规划节点          控制节点           │
│      │              │               │              │
│      └────────── ROS Topic ─────────┘              │
│                     │                              │
│              ROS Msg / Protobuf                    │
│                                                    │
└────────────────────────────────────────────────────┘

┌──────────────────── 性能监控链路 ────────────────────┐
│                                                     │
│  Linux节点A       Linux节点B       Linux节点C         │
│      │               │               │              │
│   /proc采集        /proc采集        /proc采集         │
│      │               │               │              │
│  Monitor Agent   Monitor Agent   Monitor Agent      │
│      └─────────── gRPC流式上报 ───────────┘          │
│                         │                           │
│                  Monitor Center                     │
│                         │                           │
│                    Qt监控界面                        │
│                                                     │
└─────────────────────────────────────────────────────┘
```

两条链路解决的问题不同：

- ROS业务通信链路负责感知、规划、控制等功能节点之间的数据交换；
- 性能监控链路负责采集和展示各个Linux计算节点的运行状态。

两条链路不是共用同一种传输协议，而是复用Protobuf的数据定义能力：

```markdown
              Protobuf数据定义
                ↙          ↘
        ROS Topic通信      gRPC流式通信
        机器人业务数据      节点监控数据
```

## =={pink}3. 业务通信链路的任务==

业务通信侧的目标，是=={yellow}让ROS同时兼容两种消息类型==：

```markdown
ROS原生消息  ──┐
               ├──▶ ROS Topic ──▶ ROS订阅节点
Protobuf消息 ──┘
```

对于=={yellow}原生ROS Msg，保持原来的通信方式不变==。

对于=={yellow}Protobuf消息==，需要解决两个问题：

1. =={green}让ROS将Protobuf类型识别为可以发布和订阅的消息==；
2. =={green}告诉ROS如何计算消息长度，以及如何完成序列化和反序列化==。

完成扩展后，上层业务节点不需要先把Protobuf对象转换成ROS Msg，而是可以继续使用ROS原来的接口：

```scss
publisher.publish(protobuf_message);
```

因此，这一部分的任务不是重新设计一套ROS通信框架，而是为现有ROS通信链路增加Protobuf兼容能力。

## =={pink}4. 性能监控链路的任务==

监控侧的目标，是=={yellow}将**分散在多个Linux计算节点**上的**性能信息**统一**汇总**到**中心端**==。

这条链路需要完成=={pink}**四项任务**==：

### =={yellow}4.1 节点性能采集==

从Linux的`/proc`**文件系统读取**：

- **CPU**运行数据；
- **内存**和交换分区数据；
- **网络**收发数据；
- **系统负载**及相关**运行信息**。

### =={yellow}4.2 采集器模块化==

将**不同类型的指标拆分成独立采集器**，使CPU、内存和网络采集逻辑可以分别维护和扩展。

### =={yellow}4.3 指标流式上报==

**在各计算节点运行Monitor Agent**，将**采集结果封装为Protobuf**消息，**通过gRPC客户端**流持续**上报**到**Monitor Center**。

### =={yellow}4.4 中心汇总与展示==

**Monitor Center接收**不同节点的数据，**维护节点状态**和最新性能指标，再通过**Qt界面展示**CPU、内存、网络等运行信息。

## 5. 系统模块划分

| 系统模块 | 主要职责 |
| --- | --- |
| =={pink}**ROS-Protobuf兼容层**== | 让ROS识别Protobuf类型，并完成消息序列化与反序列化 |
| ROS发布订阅示例 | 验证Protobuf消息可以直接使用ROS Topic完成收发 |
| =={yellow}**Linux性能采集模块**== | 从`/proc`采集CPU、内存、网络等运行指标 |
| =={yellow}**Monitor Agent**== | 周期性执行采集，并向中心端流式上报数据 |
| =={yellow}**Monitor Center**== | 接收和汇总多个节点的性能数据，维护节点状态 |
| =={yellow}**Qt监控界面**== | 展示Linux节点的性能指标和变化趋势 |
| =={green}**工程构建模块**== | 使用**CMake**、**Docker**和**Shell**组织编译及运行环境 |

## 6. 个人职责

这是一个个人学习项目，我负责系统的整体分析、设计与代码实现，主要包括：

- 分析ROS的消息Traits和Serialization机制；
- 实现ROS对Protobuf消息的类型识别和序列化支持；
- 编写并运行Protobuf消息的ROS发布订阅示例；
- 整理Linux性能采集模块；
- 搭建基于Protobuf和gRPC的Agent/Center监控原型；
- 整理Qt性能展示模块；
- 使用CMake组织工程，并通过Docker和Shell配置运行环境。

## 7. 项目完成目标

项目需要达到以下基本目标：

1. Protobuf消息能够直接使用ROS的发布订阅接口；
2. 扩展后不影响原生ROS Msg的通信方式；
3. Linux节点能够采集CPU、内存和网络等性能指标；
4. Agent能够通过gRPC向中心端流式上报监控数据；
5. 中心端能够汇总并展示节点运行状态；
6. 各模块能够通过CMake和容器环境进行组织和部署。

## T部分口语化总结

> 在明确了项目背景以后，我给自己设定的任务主要有两部分。第一部分是扩展ROS的消息机制，让ROS在原生支持ROS Msg的基础上，也能直接识别和传输Protobuf消息，并且上层仍然使用原来的`publish`和`subscribe`接口。
> 
> 
> 第二部分是搭建分布式性能监控链路。在每个Linux计算节点上采集CPU、内存和网络数据，通过Monitor Agent使用gRPC流式上报到中心端，再由中心端汇总并通过Qt界面展示。
> 
> 
> 所以整个系统可以看成两条主线：一条负责机器人感知、规划和控制节点之间的业务通信；另一条负责多个Linux计算节点的运行状态监控。两条链路分别使用ROS Topic和gRPC传输，但都可以复用Protobuf的数据定义。这个项目是个人学习项目，从ROS通信扩展、性能采集到gRPC监控原型和工程环境，都由我完成整理和实现。

---

# A1：扩展ROS序列化层，使其兼容Protobuf消息

## 1. 要解决的问题

ROS原生只能直接发布满足其消息类型要求的对象。=={green}对于普通ROS Msg，**构建工具**会**自动生成**==：

- =={yellow}消息**类型信息**==；
- =={yellow}**MD5校验**信息==；
- =={yellow}消息**定义**==；
- =={yellow}**序列化**和**反序列化**代码。==

---

**=={pink}MD5==**
MD5 是一种**哈希算法（消息摘要算法）**，可以把任意大小的文件 / 文本，计算出一个**128 位、32 位十六进制字符串**，这个字符串就叫 **MD5 值**。

> 简单理解：给文件生成一个独一无二的 “=={yellow}数字身份证==”。

---

而通过`.proto`生成的Protobuf类，只继承自`google::protobuf::Message`。虽然它本身具有序列化能力，但ROS并不知道：

- 它是不是一条合法消息；
- 消息类型叫什么；
- 消息长度是多少；
- 应该怎样将它写入ROS缓冲区；
- 接收后应该怎样还原。

因此，=={yellow}一个Protobuf对象默认不能直接传给ROS的==`publish()`=={yellow}接口。==

我的=={pink}**处理思路**==不是把每一种Protobuf消息手动转换成ROS Msg，而是=={pink}**在ROS底层增加一套通用规则**==：

```cpp
只要一个C++类型继承自google::protobuf::Message
                    ↓
就将它识别为Protobuf消息
                    ↓
为它提供统一的 ROS Traits 信息（ROS）
                    ↓
使用Protobuf自己的接口完成序列化（proto）
                    ↓
通过原有ROS Topic完成传输（ROS）
```

---

## =={pink}2. 整体实现链路==

以项目中的`PublishInfo`消息为例：

```cpp
.proto文件
    ↓ protoc生成
PublishInfo C++类
    ↓ 继承google::protobuf::Message
SFINAE识别为Protobuf类型
    ↓
Traits提供ROS消息身份信息
    ↓
Serializer完成二进制序列化
    ↓
ROS Topic通过TCP ROS发送
    ↓
订阅端读取二进制数据
    ↓
ParseFromString反序列化
    ↓
得到PublishInfo对象
```

=={pink}扩展后，发布者仍然使用ROS原来的接口：**publish**==

```rust
ros::Publisher pub =
    node.advertise<superbai::sample::PublishInfo>("/Sorbai", 1000);

pub.publish(proto_msg_info);
```

订阅者也继续使用ROS原来的订阅和回调机制，因此上层业务代码不需要增加一层“Protobuf转ROS Msg”的逻辑。

---

## 3. 模板偏特化是什么

#### 3.1 基础概念

=={green}C++模板==用于描述=={yellow}一套可以**处理多种数据类型**的**通用代码**。==

例如，可以先定义一个通用模板：

```cpp
template<typename T>
struct TypeInfo {
    static const char* name() {
        return "unknown";
    }
};
```

=={green}如果希望**某一类类型**采用**不同实现**，就可以**对模板进行特化**==。

##### =={pink}全特化==

=={yellow}**只针对某一个确定类型**：==

```cpp
template<>
struct TypeInfo<int> {
    static const char* name() {
        return "int";
    }
};
```

##### =={pink}偏特化==

=={yellow}不是只针对一个具体类型，而是**针对满足某种形式或条件的一类类型**。==

=={yellow}这个项目**面对的不是某一个固定的Protobuf消息**，**而是所有继承自**==`google::protobuf::Message`=={yellow}**的类型**，因此**更适合使用模板偏特化**。==

### =={pink}3.2 在项目中的作用==

=={yellow}项目**为ROS的**==**`DataType`=={yellow}、==`MD5Sum`=={yellow}、==`Definition`=={yellow}和==`Serializer`**=={yellow}**等模板**增加了**Protobuf版本的偏特化。**==

---

ROS1 C++ 通信不靠虚类，**全部依靠模板特化（traits + Serializer）**，`.msg`编译工具 gencpp 就是自动生成这些特化代码，下面的几种模板类是ROS实现msg通信的消息系统核心：

#### =={pink}四大核心triats模板：==

全部在=={green}命名空间 ros::message_traits==，都是 =={green}struct 模板==，专门给=={green}消息类型 T 打**元信息标签**==。

=={yellow}1. **DataType<T>**==
- 作用：提供消息的**DataType（完整消息类型名）**
- 发布订阅时，topic 注册、topic 类型校验拿这个字符串。

=={yellow}2. **MD5Sum<T>**==
- 作用：**提供** ROS 消息 **MD5 校验和**；**节点握手时**两边**对比**这个 **MD5 字符串**，**不一致直接拒绝**连接。

=={yellow}3. **Definition<T>**==
- 作用：**返回完整展开的消息 Definition 文本**；用来调试、rosbag、动态消息解析；**MD5Sum 原始输入就是这份文本**。

=={yellow}4. **IsMessage<T>**==
- **布尔 trait**：**标记**这个类型**是不是合法** ROS 消息**类型**。
- `IsMessage<T>::value == true` roscpp **才承认**这个类型**可以用于 publish/subscribe**。**必须特化为 true，否则直接编译报错**。

=={pink}序列化核心模板===={yellow}**Serializer<T>**==命名空间 `ros::serialization`，**真正负责二进制编解码**，是消息网络传输 /rosbag 写入的核心

> =={pink}ROS 官方注释：==**=={pink}想要让任意类型在 roscpp 通信，只需要特化 Serializer + message_traits。==**

---

可以将它=={pink}理解为==：

```markdown
ROS原来的模板
    ├── 普通ROS Msg → 使用原有处理规则
    └── Protobuf类型 → 使用新增处理规则
```

=={green}这样，无论以后增加==`PublishInfo`=={green}、==`NodeMetrics`=={green}还是其他Protobuf消息，都**不需要为每种类型重新编写一套ROS适配代码**。==

### 面试回答

> 模板偏特化就是在通用模板之外，为满足某一类条件的类型提供专门实现。在这个项目中，我不是针对某一个Protobuf消息进行适配，而是针对所有继承自`google::protobuf::Message`的类型，统一提供ROS Traits和Serializer实现。这样以后新增Protobuf消息时，不需要修改ROS通信层。

---

## =={pink}4. SFINAE是什么==

### 4.1 基础概念

SFINAE的完整名称是：

> =={yellow}**Substitution Failure Is Not An Error，替换失败不是错误。**==

C++在为模板代入具体类型时，如果某个候选模板的类型表达式不成立，不会立刻导致整个程序编译失败，而是将这个候选模板从匹配集合中移除，再尝试其他实现。

项目中主要使用两个工具：

```cpp
std::is_base_of<Base, T>
std::enable_if<condition>
```

其中：

- `std::is_base_of`判断`T`是否继承某个基类；
- `std::enable_if`只在条件成立时提供一个有效的`type`。

项目中的判断条件可以简化为：

```php
std::is_base_of<
    google::protobuf::Message,
    T
>::value
```

它的含义是：

> 判断类型`T`是不是一个Protobuf消息类。

再通过`enable_if`控制模板是否生效：

```php
typename std::enable_if<
    std::is_base_of<google::protobuf::Message, T>::value
>::type
```

如果`T`是Protobuf消息，这段表达式能够生成一个有效类型，编译器选择Protobuf专用实现。

如果`T`不是Protobuf消息，表达式替换失败，这个候选实现被移除，ROS继续选择原来的模板逻辑。

### 4.2 为什么要使用SFINAE

如果直接修改ROS公共逻辑，在运行时通过`if/else`判断消息类型，容易影响原生ROS Msg，而且需要引入额外的运行时分支。

使用SFINAE后，类型选择发生在编译阶段：

- Protobuf消息走新增实现；
- ROS Msg走原有实现；
- 两条路径互不影响；
- 不需要运行时判断；
- 新增Protobuf消息不需要修改底层代码。

#### 面试回答

> SFINAE是一种模板编译期选择机制。在项目中，我通过`std::is_base_of`判断消息类型是否继承自Protobuf的`Message`基类，再使用`std::enable_if`控制对应的Traits和Serializer偏特化是否生效。这样只有Protobuf消息会进入新增逻辑，原生ROS Msg仍然走ROS原来的实现。

---

## 5. ROS Traits是什么

### 5.1 基础概念

Traits可以理解为一个类型的“编译期档案”。

ROS拿到一个C++类型后，不会通过大量运行时判断来分析它，而是查询对应的Traits，了解这个类型具有什么性质。

例如：

| Trait | 回答的问题 |
| --- | --- |
| `IsMessage` | 这是不是一条合法的ROS消息 |
| `DataType` | 这条消息的类型名称是什么 |
| `MD5Sum` | 通信双方使用的消息结构是否一致 |
| `Definition` | 消息的结构定义是什么 |
| `HasHeader` | 是否包含ROS标准Header |
| `IsFixedSize` | 序列化长度是否固定 |

ROS在建立发布订阅关系和处理消息时，会使用这些信息完成类型识别、连接校验和序列化策略选择。

### 5.2 项目中怎样扩展Traits

项目对继承自`google::protobuf::Message`的类型提供了以下Traits。

#### `IsMessage`

```graphql
返回True
```

作用是告诉ROS：

> 这个Protobuf类型可以被视为一条消息，可以进入发布订阅流程。

这是Protobuf获得ROS“消息身份”的关键。

#### `DataType`

项目通过Protobuf的Descriptor获取消息名称，并生成类似下面的类型标识：

```undefined
pb_msgs/PublishInfo
```

作用是让ROS在建立连接和调试时能够识别消息类型。

#### `Definition`

通过Protobuf反射接口获取Descriptor，再提取：

- `.proto`文件名称；
- 消息完整名称；
- 字段及消息结构描述。

作用相当于为ROS提供这条Protobuf消息的结构说明。

#### `MD5Sum`

ROS1在发布者和订阅者建立连接时，会比较消息的MD5标识，避免结构不兼容的消息错误连接。

当前原型中为Protobuf消息提供了固定标识：

```undefined
proto_md5
```

这使Protobuf发布端和订阅端能够通过ROS的类型检查流程，但它不能真正反映不同`.proto`结构之间的差异。工程化版本应该根据完整的Protobuf描述文件生成稳定的Schema哈希。

#### `HasHeader`

项目将其设置为`FalseType`，表示通用Protobuf消息不默认包含ROS标准的`std_msgs::Header`。

#### `IsFixedSize`

项目将其设置为`FalseType`。因为Protobuf消息中可能包含字符串、数组等变长字段，序列化后的长度不是固定值。

### 5.3 Traits在链路中的位置

```markdown
Protobuf消息对象
        ↓
ROS查询IsMessage
判断它能否作为消息
        ↓
ROS查询DataType和MD5Sum
完成类型标识与连接校验
        ↓
ROS查询Definition等信息
提供消息描述
        ↓
进入Serializer
```

#### 面试回答

> Traits可以理解为ROS的编译期类型档案。ROS通过Traits判断一个类型是不是消息、类型名称是什么、长度是否固定、有没有Header，以及通信双方是否兼容。项目中我通过偏特化为所有Protobuf类型补充了这些信息，让ROS能够把Protobuf类当作合法消息处理。

---

## 6. ROS Serialization是什么

### 6.1 基础概念

Traits解决的是“ROS是否认识这种类型”，Serialization解决的是：

> ROS应该怎样把这个对象转换成可传输的字节，以及接收后怎样将字节还原成对象。

ROS的Serializer主要提供三个操作：

| 接口 | 作用 |
| --- | --- |
| `serializedLength()` | 计算序列化后需要多少字节 |
| `write()` | 将对象写入输出缓冲区 |
| `read()` | 从输入缓冲区还原对象 |

如果只补充Traits而不补充Serializer，ROS虽然知道它是一条消息，但仍然不知道怎样实际传输它。

### 6.2 项目中的序列化流程

#### 第一步：计算消息长度

调用Protobuf提供的序列化接口，将消息转换成二进制字符串：

```scss
t.SerializeToString(&pb_str);
```

项目中的最终数据由两部分组成：

```undefined
4字节长度字段 + Protobuf消息体
```

因此：

```undefined
总长度 = 4字节 + Protobuf消息体长度
```

#### 第二步：写入ROS输出流

`write()`执行以下操作：

```markdown
Protobuf对象
    ↓ SerializeToString
二进制字符串
    ↓
先写入4字节消息长度
    ↓
再复制Protobuf消息体
    ↓
ROS输出缓冲区
```

长度字段用于告诉接收端，后面应该读取多少字节作为一个完整的Protobuf消息体。

#### 第三步：ROS完成传输

Serializer只负责将对象转换成ROS能够发送的字节流。后面的连接管理、发送队列和TCPROS传输仍然由ROS负责。

因此，项目没有重新实现网络通信协议。

#### 第四步：订阅端反序列化

接收端的`read()`执行相反过程：

```markdown
ROS输入缓冲区
    ↓
读取4字节长度
    ↓
读取指定长度的Protobuf消息体
    ↓ ParseFromString
恢复Protobuf对象
```

对应的Protobuf接口是：

```scss
t.ParseFromString(pb_str);
```

恢复完成后，订阅回调拿到的就是原始Protobuf消息对象，可以直接访问其中的字段或调用`DebugString()`。

### 6.3 为什么不自己实现Protobuf编码

项目只负责把Protobuf序列化能力接入ROS，不重新实现Protobuf的Wire Format。

原因是：

- `SerializeToString()`已经负责字段编号和具体编码；
- `ParseFromString()`已经负责字段解析和对象恢复；
- 自己实现容易产生协议兼容和边界处理问题；
- 项目的重点是ROS与Protobuf的适配，而不是重新设计序列化协议。

#### 面试回答

> Serialization负责对象和字节流之间的转换。我为Protobuf类型偏特化了ROS的Serializer。在发送端调用`SerializeToString()`得到Protobuf二进制数据，先写入4字节长度，再写入消息体；接收端先读取长度和消息体，再通过`ParseFromString()`恢复对象。网络连接、消息队列和TCPROS传输仍然由ROS负责。

---

## 7. 四项技术是怎样配合的

四项技术并不是相互独立的知识点，而是共同完成同一条链路。

```markdown
模板偏特化
负责：为一类Protobuf类型提供专门实现
        ↓
SFINAE
负责：判断当前类型是否属于Protobuf消息
        ↓
Traits
负责：告诉ROS这是什么消息、具有什么属性
        ↓
Serialization
负责：告诉ROS怎样打包和还原消息
        ↓
ROS Topic
负责：完成发布、订阅和网络传输
```

可以用一句话概括：

> 模板偏特化提供扩展入口，SFINAE负责在编译期筛选Protobuf类型，Traits解决ROS对消息的类型识别问题，Serializer解决消息的二进制收发问题。

---

## 8. 最终实现效果

完成扩展后，系统具备以下能力：

- ROS Topic可以直接承载Protobuf消息；
- 上层继续使用原生`publish()`和`subscribe()`接口；
- 不需要为每一种Protobuf消息单独编写ROS Msg转换器；
- 新增Protobuf消息时，可以复用同一套Traits和Serializer；
- 原有ROS Msg仍然使用ROS原来的通信流程；
- ROS负责节点、Topic和传输，Protobuf负责消息定义与序列化。

---

## 9. A1口语化回答

> ROS默认只能直接处理具备ROS消息特征和序列化规则的类型，而`.proto`生成的C++类只是继承自Protobuf的`Message`基类，所以直接调用`publish()`时，ROS既不知道它是不是合法消息，也不知道应该怎么进行序列化。
> 
> 
> 我的实现主要分成两部分。第一部分是Traits扩展，我通过模板偏特化和SFINAE，判断一个类型是否继承自`google::protobuf::Message`。如果满足条件，就为它提供`IsMessage`、`DataType`、`MD5Sum`、`Definition`等类型信息，让ROS能够识别它。对于普通ROS Msg，这些偏特化不会生效，仍然走原有逻辑。
> 
> 
> 第二部分是Serialization扩展。我为Protobuf类型提供了专用的Serializer。发送时调用`SerializeToString()`将对象转换成二进制数据，写入4字节长度和消息体；接收时读取长度和数据，再调用`ParseFromString()`还原对象。
> 
> 
> 这样，上层节点不需要编写Protobuf到ROS Msg的转换代码，可以直接使用ROS原有的`publish`和`subscribe`接口传输Protobuf消息，同时原生ROS Msg也不会受到影响。

## 10. 高频追问速答

### 为什么不用运行时`if/else`判断？

> 消息类型在编译时已经确定，使用SFINAE可以在编译期选择对应实现，没有额外的运行时分支，而且不会侵入原生ROS Msg的处理逻辑。

### 模板偏特化和SFINAE是什么关系？

> 偏特化负责提供Protobuf类型的专用模板实现，SFINAE负责判断这个专用实现什么时候可以参与匹配。一个提供实现，一个控制启用条件。

### Traits和Serializer有什么区别？

> Traits描述消息的身份和属性，回答“它是什么”；Serializer定义对象和字节流之间的转换，回答“它怎么传”。只有两部分都实现，Protobuf消息才能真正通过ROS通信。

### 为什么通过继承关系判断Protobuf类型？

> `protoc`生成的标准Protobuf消息类都会继承`google::protobuf::Message`，因此可以通过`std::is_base_of`进行统一识别，不需要枚举每一种具体消息。

### 为什么序列化数据前面要增加4字节长度？

> Protobuf消息本身是变长的，接收端需要知道应该读取多少字节作为消息体，所以实现中增加了一个`uint32_t`长度字段。

### 这个方案是否完全等同于原生ROS Msg？

> 发布订阅接口基本保持一致，但ROS工具链兼容性和Schema校验仍有差别。例如当前原型的MD5使用固定标识，工程化时应该根据Protobuf描述文件生成真正的Schema哈希。

---

# A2：设计可插拔Linux性能指标采集框架

## 1. 为什么需要性能采集

ROS业务节点运行在Linux计算机上。即使ROS Topic通信正常，感知、规划或控制程序也可能因为CPU负载过高、内存不足或网络拥塞而运行异常。

因此，每个计算节点都需要一个Monitor Agent，周期性采集本机的：

- CPU占用率；
- 内存和交换分区使用量；
- 网络收发速率；
- 系统负载和中断信息。

采集结果统一写入`NodeMetrics`消息，交给后面的gRPC模块上报。

---

## 2. `/proc`是什么

`/proc`是Linux提供的一个**虚拟文件系统**。

它看起来像普通文件目录，但其中大部分内容并不真正保存在硬盘上，而是由Linux内核根据当前运行状态动态生成。

可以把它理解成：

> Linux内核对用户程序开放的一组“系统状态查询接口”。

例如：

| 文件 | 项目中的用途 |
| --- | --- |
| `/proc/stat` | 获取各CPU核心累计运行时间和中断次数 |
| `/proc/meminfo` | 获取总内存、可用内存和交换分区信息 |
| `/proc/net/dev` | 获取不同网卡累计收发字节数 |
| `/proc/loadavg` | 获取系统一分钟和五分钟平均负载 |

项目直接读取并解析这些文件，不需要额外安装监控工具，也不需要编写内核驱动。

---

## 3. 各项指标怎样计算

### 3.1 CPU占用率

`/proc/stat`记录的不是当前CPU利用率，而是CPU从系统启动以来，在用户态、内核态、空闲等状态下累计运行的时间。

因此，不能只读取一次，而是要比较前后两次采样：

```markdown
第一次：记录总运行时间和空闲时间
                ↓
等待一个采样周期
                ↓
第二次：再次读取累计时间
                ↓
计算总时间增量和空闲时间增量
                ↓
得到这段时间内的CPU占用率
```

项目中的`CpuCollector`会保存上一次每个CPU核心的`idle`和`total`，下一次采集时使用差值计算利用率。

### 3.2 内存使用量

`MemoryCollector`读取`/proc/meminfo`中的：

- `MemTotal`；
- `MemAvailable`；
- `SwapTotal`；
- `SwapFree`。

然后计算：

```undefined
已使用内存 = 总内存 - 可用内存
已使用交换分区 = 交换分区总量 - 空闲交换分区
```

### 3.3 网络收发速率

`/proc/net/dev`提供的是各网卡从系统启动以来累计接收和发送的字节数，不是每秒速率。

因此，`NetworkCollector`保存上一次的收发字节数和时间戳，通过两次采样的字节差除以时间差，得到每秒收发字节数。

项目中会跳过`lo`回环接口，再对其他网卡的数据进行汇总。

### 3.4 系统负载和中断

`IrqLoadCollector`读取：

- `/proc/loadavg`中的一分钟和五分钟负载；
- `/proc/stat`中的硬中断和软中断累计次数。

这些指标可以辅助判断计算节点是否处于持续高负载状态。

---

## 4. 为什么不能把所有采集逻辑写在一个类里

最直接的做法，是在Monitor Agent中依次读取所有`/proc`文件。

但这样会导致：

- Agent同时承担采集、计算和通信职责；
- CPU、内存和网络逻辑耦合在一起；
- 增加或关闭某一类指标时需要修改Agent主流程；
- 后续增加磁盘、温度或GPU指标时，代码越来越庞大。

因此，我将每一类指标封装成独立采集器，并让所有采集器遵循同一个接口：

```cpp
class IMetricCollector {
public:
    virtual ~IMetricCollector() = default;
    virtual void Collect(NodeMetrics* out) = 0;
    virtual std::string Name() const = 0;
};
```

这个接口规定所有采集器都必须具备两项能力：

- `Collect()`：采集指标并写入`NodeMetrics`；
- `Name()`：返回采集器名称。

在此基础上实现：

```markdown
IMetricCollector
    ├── CpuCollector
    ├── MemoryCollector
    ├── NetworkCollector
    └── IrqLoadCollector
```

---

## 5. 继承在项目中怎样使用

继承用于建立统一接口与具体采集器之间的关系。

例如：

```cpp
class CpuCollector : public IMetricCollector {
public:
    void Collect(NodeMetrics* out) override;
};
```

`CpuCollector`继承`IMetricCollector`，并实现自己的`Collect()`方法。其他采集器采用相同方式。

继承解决的是：

> 不同采集器内部实现不同，但对外都提供相同的调用方式。

CPU采集器读取`/proc/stat`，内存采集器读取`/proc/meminfo`，虽然内部逻辑不同，但Agent只需要知道它们都是`IMetricCollector`。

### 面试速答

> 我定义了统一的`IMetricCollector`抽象接口，CPU、内存、网络和负载采集器分别继承它并重写`Collect()`。这样不同指标的采集逻辑彼此独立，但对Agent提供统一接口。

---

## 6. 多态在项目中怎样使用

多态表示：

> 使用同一个基类接口，实际执行不同子类的实现。

Monitor Agent不需要分别编写：

```scss
cpu_collector.Collect();
memory_collector.Collect();
network_collector.Collect();
```

而是将所有采集器保存到同一个容器中：

```cpp
std::vector<std::unique_ptr<IMetricCollector>> collectors;
```

采集时统一遍历：

```scss
for (auto& collector : collectors) {
    collector->Collect(&metrics);
}
```

虽然变量类型都是`IMetricCollector`，但由于`Collect()`是虚函数：

- 当前对象是`CpuCollector`时，调用CPU采集逻辑；
- 当前对象是`MemoryCollector`时，调用内存采集逻辑；
- 当前对象是`NetworkCollector`时，调用网络采集逻辑。

这个选择发生在运行时，这就是运行时多态。

### 面试速答

> Agent只依赖`IMetricCollector`基类，并通过虚函数调用不同子类的`Collect()`。这样Agent不需要关心采集器的具体类型，遍历一个容器就能完成所有指标采集。

---

## 7. 智能指针在项目中怎样使用

不同采集器通过基类指针统一保存，但这些对象又需要被自动释放。

因此，项目使用：

```cpp
std::unique_ptr<IMetricCollector>
```

`unique_ptr`表示一个采集器在同一时间只有一个所有者。它离开作用域或从容器中移除时，会自动释放对象，不需要手动调用`delete`。

项目中的容器是：

```cpp
std::vector<std::unique_ptr<IMetricCollector>>
```

它表示：

- Agent统一拥有这些采集器；
- 不允许随意复制采集器对象；
- Agent退出时自动释放所有采集器；
- 避免裸指针产生内存泄漏和所有权不清的问题。

工厂创建采集器后，通过`std::move`将所有权转移给Agent的容器。

### 面试速答

> 工厂返回`unique_ptr<IMetricCollector>`，Agent再把它移动到采集器容器中。这样采集器的所有权非常明确，Agent退出时对象会被自动释放，不需要手动管理裸指针。

---

## 8. 工厂模式在项目中怎样使用

如果Agent直接创建具体对象：

```scss
new CpuCollector();
new MemoryCollector();
new NetworkCollector();
```

Agent就会依赖所有具体采集器。每增加一个新采集器，都要修改Agent代码。

因此，项目设计了`CollectorFactory`。工厂内部维护“采集器名称—创建函数”的映射关系：

```bash
"cpu"       → 创建CpuCollector
"memory"    → 创建MemoryCollector
"network"   → 创建NetworkCollector
"irq_load"  → 创建IrqLoadCollector
```

Agent只需要根据名称创建：

```cpp
auto collector =
    CollectorFactory::Instance().Create("cpu");
```

它不需要了解`CpuCollector`的构造细节。

新增磁盘采集器时，基本流程是：

1. 编写`DiskCollector`并继承`IMetricCollector`；
2. 实现`Collect()`；
3. 将`"disk"`及其创建函数注册到工厂；
4. Agent通过名称创建并加入采集器列表。

Agent的采集主流程不需要修改。

### 面试速答

> 工厂模式把对象创建和对象使用分开。Agent只向工厂传入`cpu`、`memory`等名称，工厂负责返回对应的采集器。后续增加新指标时，只需要实现接口并注册到工厂，不需要修改Agent的核心采集循环。

---

## 9. 为什么称为“可插拔”

这里的可插拔不是动态加载`.so`插件，而是**代码结构上的可配置和可扩展**。

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

因此，可以只启用CPU和内存，也可以继续增加磁盘、温度或GPU采集器，而不需要重写Agent主流程。

---

## 10. 完整采集流程

```bash
Monitor Agent启动
        ↓
根据采集项名称调用CollectorFactory
        ↓
工厂返回不同采集器的unique_ptr
        ↓
统一保存到IMetricCollector容器
        ↓
按照采样周期遍历所有采集器
        ↓
通过多态调用各自的Collect()
        ↓
读取并解析Linux /proc
        ↓
统一填充NodeMetrics消息
        ↓
交给gRPC模块上报
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

## 1. 为什么还需要gRPC

A2已经能够从单台Linux计算节点采集CPU、内存和网络指标，但数据仍然留在本机。

当系统中存在多个计算节点时，如果逐台登录查看，无法形成统一的监控视图。因此，需要建立一条跨节点通信链路：

```markdown
各节点采集性能数据
        ↓
持续上报到中心节点
        ↓
中心汇总不同节点的数据
        ↓
监控界面订阅并实时展示
```

这条链路需要满足两个特点：

- 节点能够持续上报指标，而不是每次重新建立请求；
- 中心能够持续向监控端推送最新数据。

因此，项目选择了gRPC的流式通信。

---

## 2. gRPC是什么

gRPC是一种远程过程调用框架。

普通函数调用发生在同一个程序中：

```ini
result = GetMetrics();
```

远程过程调用则允许客户端像调用函数一样，请求另一台计算机上的服务：

```scss
客户端调用ReportMetrics()
            ↓ 网络
服务端执行ReportMetrics()
```

gRPC通常使用`.proto`同时定义：

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

## 3. Protobuf和gRPC是什么关系

这两个概念容易混淆，可以这样区分：

- **Protobuf负责描述和编码数据**；
- **gRPC负责把数据从一个节点传到另一个节点，并组织远程服务调用**。

可以类比为：

```undefined
Protobuf：规定信件的内容格式
gRPC：负责投递信件
```

项目使用`NodeMetrics`表示一个节点某个时刻的性能快照，其中包含：

- 节点ID和时间戳；
- 各CPU核心占用率；
- 内存和交换分区；
- 网络收发速率；
- 系统负载与中断；
- 节点健康等级。

同一份Protobuf定义可以被Agent、Center和监控端共同使用。

---

## 4. 为什么监控链路选择gRPC

项目没有继续使用ROS Topic完成性能监控，主要有三个原因。

### 4.1 监控链路和机器人业务链路职责不同

ROS Topic主要承担感知、规划和控制等机器人业务数据的传递。

节点性能监控属于基础设施层功能，单独建立监控通道后，不需要让监控中心依赖完整的ROS运行环境。

### 4.2 gRPC原生支持Protobuf

A2采集得到的数据已经封装为`NodeMetrics`，gRPC可以直接使用这份消息定义，不需要再编写一套转换逻辑。

### 4.3 gRPC支持流式通信

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

---

## 9. 节点健康状态怎样判断

Center接收到指标后，会根据CPU和内存使用情况进行简单的阈值判断：

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

## 11. 三种RPC为什么这样选择

| 接口 | 通信形式 | 使用原因 |
| --- | --- | --- |
| `ReportMetrics` | 客户端流 | Agent持续上报多条指标 |
| `SubscribeMetrics` | 服务端流 | Center持续向监控端推送指标 |
| `Configure` | 普通请求响应 | 一次发送配置，一次返回结果 |

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
