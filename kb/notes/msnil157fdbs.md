# 跨项目技术八股索引

这份索引用于把项目问题连接到基础知识。面试回答应先落到项目，再扩展原理，避免只背定义。

## 1. C/C++

### 必会主题

- 指针、引用、const、数组退化、结构体内存布局；
- 栈、堆、静态区，RAII 和智能指针；
- 虚函数、虚析构、多态、对象切片；
- 模板、偏特化、SFINAE、`enable_if`、`is_base_of`；
- C++11 function-local static 初始化、atomic、mutex 和内存可见性；
- `unique_ptr`/`shared_ptr` 的所有权和开销；
- 拷贝、移动、生命周期与悬空引用；
- C++ 异常与嵌入式错误码的取舍。

项目锚点：ROS traits/Serializer、采集器接口、工厂 registry、Center 并发 map。

## 2. 操作系统与 Linux

- 进程与线程、用户态与内核态、系统调用；
- 调度、上下文切换、优先级反转；
- 虚拟内存、页缓存、RSS、swap、MemAvailable；
- `/proc/stat` jiffies 与 CPU 使用率；
- `/proc/net/dev` 累计计数与吞吐率；
- load average 与 CPU 使用率区别；
- mutex、semaphore、condition/event 的语义；
- 阻塞 IO、非阻塞 IO、同步/异步。

项目锚点：FreeRTOS 并发、Linux 指标采集、gRPC 同步 server。

## 3. RTOS 与嵌入式

- 抢占式调度、任务状态、tick、时间片；
- `vTaskDelay` 与 `vTaskDelayUntil`；
- Queue/EventGroup/Semaphore/Mutex；
- 优先级继承、死锁四条件、锁顺序；
- ISR 中可调用的 `FromISR` API；
- stack high-water mark、heap 碎片、WDT；
- UART/I2C/SPI/GPIO/DMA；
- Bootloader、OTA 分区、签名、回滚；
- Flash 擦写寿命和 NVS；
- 环形缓冲和生产者-消费者。

项目锚点：蜂箱六任务、GPS 字节流、离线缓存、OTA、摄像头 DMA。

## 4. 网络与分布式系统

- TCP 可靠字节流、粘包只是应用层边界问题；
- HTTP/1.1 与 HTTP/2 的多路复用、头压缩和 stream；
- MQTT QoS0/1/2、保留消息、心跳、遗嘱；
- gRPC unary/client-stream/server-stream/bidirectional-stream；
- Protobuf field number、wire type、未知字段和兼容规则；
- 断线重连、指数退避、jitter、幂等、业务 ACK；
- 背压、有界队列、丢弃策略、慢消费者；
- TLS、设备身份、Token、证书轮换；
- at-most-once、at-least-once、exactly-once 的现实含义。

项目锚点：蜂箱 MQTT/HTTP；分布式系统 ROS framing、gRPC stream。

## 5. ROS 与机器人中间件

- ROS Master 发现与点对点数据连接；
- Node、Topic、Service、Parameter；
- advertise/subscribe queue 和 callback queue；
- TCPROS 与 UDPROS；
- message traits、MD5Sum、Definition、Serializer；
- ROS1 的中心发现与 ROS2 DDS 的差异；
- bridge 与侵入式修改中间件的权衡；
- 零拷贝、进程内通信、序列化开销。

项目锚点：ROS-Protobuf 透明适配。

## 6. AI 模型与部署

- 训练图与推理图、`eval()`；
- RNN/LSTM/GRU 的状态与梯度；
- 标准化、PCA、训练测试数据泄漏；
- ONNX graph、opset、shape inference；
- 静态 Shape 与动态 Shape；
- 编译器图优化：常量折叠、融合、内存复用；
- FP32/FP16/INT8、PTQ/QAT；
- Host/Device 内存、H2D/D2H、同步；
- 模型迁移数值验证与端到端指标；
- CPU/GPU/NPU 执行特点。

项目锚点：锂电池 `.pth→ONNX→OM→ACL Lite`。

## 7. 数据与算法评估

- 回归指标 RMSE、MAE、MAPE、R²；
- 为什么“准确率”不适合未定义的回归任务；
- 时间序列切分不能随机泄漏未来；
- 跨设备测试与同设备时间外推的区别；
- 消融实验、基线、置信区间；
- 离线指标与板端实际指标的区别；
- 精度、时延、吞吐、内存、功耗的联合评价。

项目锚点：五组电池结果与 EOL 误差；蜂箱长稳与图传基准；gRPC 节点规模压测。

## 8. 每个主题的回答模板

以“为什么用 Queue”为例：

1. 定义：Queue 是带同步语义的有界消息容器；
2. 项目问题：采样周期稳定，网络上传可能阻塞；
3. 实现：Sensor producer 发送 TelemetryMessage，Uploader consumer 接收；
4. 目的：解耦、保持快照一致性、提供有限缓冲；
5. 代价：占 RAM，队列满必须定义丢弃策略；
6. 验证：断网/慢网上传时检查采样周期和消息顺序。

其他技术问题都用同样六步回答。

