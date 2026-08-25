# ESP32-S3 + FreeRTOS 智慧蜂箱：项目全景与技术深挖

## 1. 项目定位和个人边界

智慧蜂箱是一套 IoT 全链路系统：终端采集温湿度、重量、位置和图像；云端保存数据并提供业务接口；PC 端 YOLOv8 处理图像；小程序/大屏展示状态。

个人可重点主张的部分是终端固件：ESP32-S3 主控端和 ESP32-CAM 图传端。后端、AI 和前端属于系统上下文，除非有对应原始材料，不要描述成自己独立完成。

当前源码确认的主控能力：DHT22、HX711、NEO-6M、OLED、Wi-Fi/MQTT、六个 FreeRTOS 任务、队列/事件组/互斥锁、RAM 环形缓存、NVS 参数、三条 OTA 路径、Web 状态页。

当前源码确认的图传能力：OV2640/esp_camera、JPEG 抓拍、MJPEG 流、PSRAM 双帧缓冲、底层 I2S/DMA、HTTP 接口、可选 SD 保存和 OTA。

## 2. 系统架构

```text
                   ┌──────────── 云/后端/应用层 ────────────┐
主控 MQTT 遥测 ───►│ IoT Broker / 数据库 / API / 小程序大屏 │
                   └────────────────────────────────────────┘

ESP32-S3 主控                         ESP32-CAM 图传端
├─ DHT22 单总线                       ├─ OV2640 DVP
├─ HX711 GPIO 时序                    ├─ esp_camera
├─ NEO-6M 硬件 UART                   │   └─ I2S/DMA → 帧缓冲
├─ OLED 硬件 I2C                      ├─ PSRAM 双帧
├─ FreeRTOS 双核任务                  └─ HTTP /capture /stream
├─ MQTT 遥测/命令
├─ 离线缓存 + NVS
└─ Web/Arduino/MQTT URL OTA
```

为什么拆成两个 MCU：图像带宽和内存需求远高于普通遥测，独立图传端避免摄像头流占满主控的 CPU、PSRAM、网络连接和任务调度资源；控制流走 MQTT，图像流走 HTTP，故障和扩容边界更清楚。

## 3. 主控固件任务模型

### 3.1 六个任务和核绑定

| 任务 | Core | 优先级 | 目的 |
| --- | --- | --- | --- |
| `taskNetManager` | 0 | 4 | Wi-Fi/MQTT 重连、MQTT loop、OTA 处理 |
| `taskUploader` | 0 | 3 | 队列消费、实时上传、离线补发 |
| `taskHttpServer` | 0 | 2 | Web 状态、API 和 Web OTA |
| `taskSensorAcquire` | 1 | 3 | DHT/HX711 周期采样并产生快照 |
| `taskGpsReader` | 1 | 3 | 持续读取 UART/NMEA，降低字节丢失 |
| `taskOledDisplay` | 1 | 1 | 读取共享快照并低频刷新显示 |

网络任务放 Core0，采集和本地业务放 Core1，主要为了把可能阻塞的协议栈与周期采集隔离。核绑定不等于绝对实时：Wi-Fi 系统任务、中断、锁等待和同核高优先级任务仍会影响时延。

### 3.2 为什么业务不放在 `loop()`

Arduino-ESP32 的 `loop()` 本身运行在框架创建的 FreeRTOS task 中。旧式大循环把联网、采样、解析、显示串行执行，一个超时会拖慢所有模块。新版把职责拆成独立任务，`loop()` 只做 `vTaskDelay`，调度和阻塞边界更清楚。

### 3.3 周期和阻塞

代码使用 `vTaskDelay` 释放 CPU，不是裸忙等。但传感器任务用 `safeDelay(sampleMs)`，其周期会包含本次采集执行时间，可能产生漂移；更严格的固定周期应使用 `vTaskDelayUntil`。DHT22、HX711 读数和 Wi-Fi 连接仍包含局部阻塞，只是被隔离在对应任务中。

## 4. 并发对象逐项解释

### 4.1 Queue：采集和上传解耦

生产者 `taskSensorAcquire` 构造完整 `TelemetryMessage`，通过 `g_telemetryQueue` 投递；消费者 `taskUploader` 负责网络发送。

目的：网络抖动不直接拖慢传感器采样；消息按值复制形成时间一致的快照，避免上传时读到被另一任务改了一半的数据。

队列满时，当前最终版代码把消息转存到离线环形缓存，而不是阻塞采集。代价是 RAM 占用和可能丢历史数据。

### 4.2 EventGroup：表达组合网络状态

`BIT0` 表示 Wi-Fi，`BIT1` 表示 MQTT。任务读取 bit 判断是否具备发布条件；Wi-Fi 断开时同时清除两个 bit，因为 MQTT 依赖 Wi-Fi。

事件组适合表达状态，不承载业务数据；业务数据仍走队列。

### 4.3 Mutex：保护非线程安全对象与共享状态

- `g_stateMutex`：保护 `SharedState` 和 GPS/采集状态；
- `g_mqttMutex`：串行访问 `PubSubClient`，因为网络管理和上传任务都会调用它；
- `g_cacheMutex`：保护环形缓存 head/count；
- `g_i2cMutex`：保护 I2C/OLED 资源。

FreeRTOS mutex 支持优先级继承，可降低低优先级任务持锁导致的优先级反转，但不能消除长临界区。代码使用 20~1000ms 有限等待，避免永久卡死。

### 4.4 共享状态为什么还要保留

队列适合传历史快照，`SharedState` 适合读取“当前最新值”，供 OLED、HTTP 和告警使用。读取采用复制快照：短时间持锁复制结构，释放后再格式化 JSON/HTML，避免把慢操作放在临界区。

## 5. 外设实现

### 5.1 DHT22

单总线数字温湿度传感器，时序敏感且采样频率低。代码调用库读取并检查 NaN、温湿度物理范围；失败时沿用上次有效值。生产化应同时上报 freshness/error flag，避免“旧值看起来正常”。

### 5.2 HX711

24 位称重 ADC，通过 DOUT/SCK GPIO 时序读取。代码设置比例因子、启动时 tare、`get_units(HX711_AVG_SAMPLES)` 多次采样均值滤波，最后换算 kg。

比例因子必须用标准砝码标定：去皮后用已知质量计算 raw/质量，再验证多点线性和温漂。简单均值能抑制随机噪声，但对突变和离群点不如中值/截尾均值。

### 5.3 NEO-6M GPS

使用 ESP32-S3 硬件 UART，扩大 RX buffer，并由独立任务每 10ms 持续读取字节交给 TinyGPSPlus 解析 NMEA。这样避免 DHT、MQTT 或显示阻塞时 UART 缓冲溢出。

GPS 有效性不能只看经纬度，还应考虑定位有效标志、卫星数、数据 age 和冷启动时间。

### 5.4 OLED

硬件 I2C + U8g2 全缓冲模式：在 RAM 中绘制一帧，再 `sendBuffer()` 一次刷新，减少逐像素闪烁。刷新任务优先级最低，避免显示影响核心采集通信。

## 6. MQTT 和网络可靠性

### 6.1 数据路径

采样快照包含序号、uptime、温湿度、重量、GPS、RSSI、告警位和离线重放标志，序列化为 JSON 后发布到 telemetry topic；状态 topic 使用 retained，命令 topic 接收采样周期和 OTA URL。

最终版 `PubSubClient::publish` 默认使用 QoS0。优点是轻量；缺点是没有 broker 级送达确认。当前离线缓存只能处理“本地判断未连接或 publish 返回 false”，不能保证已经返回 true 的消息最终被云端处理。

### 6.2 重连策略

网络管理任务定期检查 Wi-Fi 和 MQTT，分别以 5s/3s 间隔尝试重连，避免在采集路径中永久阻塞。当前策略是固定间隔；大量设备同时断网时更适合指数退避加随机抖动，防止惊群。

### 6.3 离线环形缓存

固定数组 + `head/count`：

- push：尾位置 `(head+count)%N`；
- pop：读 head 后 `head=(head+1)%N`；
- 满时 head 前移，覆盖最旧数据；
- 网络恢复后每轮最多补发 4 条，每条间隔 20ms，防止历史数据独占链路。

这是 RAM 缓存，掉电即丢失。代码也没有区分告警和普通遥测，满时同样覆盖最旧项；README 中“覆盖最旧普通遥测”并不完全准确。产品化可使用 Flash/FRAM 持久队列、告警优先级和写放大控制。

### 6.4 NVS 参数热更新

MQTT 命令修改 `sample_interval_ms`，校验 2s~1h 范围后更新共享状态并写 Preferences/NVS，重启后恢复。频繁写 Flash 会磨损，因此只在值变化时写并做写频率限制更稳妥。

## 7. OTA/IAP

三条通道：

1. Web OTA：浏览器带 Basic Auth 上传 `.bin`，`Update.write` 分块写 OTA 分区；
2. ArduinoOTA：局域网开发调试通道；
3. MQTT 命令下发 URL：设备用 HTTPUpdate 拉取固件。

通用流程：下载/上传 → 写非当前应用分区 → 校验写入 → 修改启动分区 → 重启。当前源码完成基础更新，但没有固件签名、HTTPS 证书校验、版本单调性、首次启动健康确认和自动回滚。面试中应主动说明这些生产化缺口。

## 8. 图传端

### 8.1 为什么用 HTTP 而不是 MQTT 图片切片

MQTT适合低频控制和遥测；连续图像需要切片、重组、消息数多且 broker 压力大。HTTP 可以直接把 JPEG 作为 response body，MJPEG 用 multipart boundary 连续发送帧，浏览器和后端容易消费。

### 8.2 I2S/DMA、PSRAM 双帧和最新帧优先

OV2640 的并行 DVP 像素由 `esp_camera` 驱动通过 ESP32 的摄像头/I2S DMA 路径搬入 frame buffer。应用配置而非手写底层 DMA：

- 有 PSRAM：frame buffer 放 PSRAM，`fb_count=2`，`CAMERA_GRAB_LATEST`；
- 无 PSRAM：单 DRAM buffer、降为 QVGA、空 buffer 才采集。

双帧让采集和网络发送有机会重叠；最新帧优先丢弃积压旧帧，降低实时观看延迟，代价是可能掉帧。JPEG 在摄像头侧压缩，减少 Wi-Fi 带宽和 MCU 内存压力。

### 8.3 HTTP 接口

- `/capture`：抓一帧，直接返回 `image/jpeg`，可选保存 SD；
- `/stream`：`multipart/x-mixed-replace`，逐帧写 boundary、Content-Length 和 JPEG；
- `/control`：动态调整分辨率和质量；
- `/status`：heap、PSRAM、RSSI、帧参数；
- `/ota`、`/update`：图传端升级。

当前 `WebServer` 是同步处理模型，`handleStream()` 内部 while 会长期占用请求处理流程，不能把 `g_streamClients` 计数理解成真正的多客户端并发能力。产品化可使用异步 HTTP server、单独采集任务和多消费者帧队列。

## 9. 代码级风险与改进

| 当前实现 | 风险 | 改进 |
| --- | --- | --- |
| `WiFi.setSleep(false)` | 图传/连接稳定但功耗高 | 依据工作模式动态启用省电 |
| 主控 RAM 离线缓存 | 断电丢数据 | Flash/FRAM 持久队列 |
| QoS0 | 缺端到端确认 | QoS1 + 业务 ACK/幂等序号 |
| 固定重试周期 | 多设备惊群 | 指数退避 + jitter |
| OTA 基础鉴权 | 固件被替换风险 | HTTPS、签名、安全启动、回滚 |
| 同步 MJPEG handler | 服务阻塞和并发差 | 异步服务、帧发布模型 |
| 动态 `String` 构建 | 长期可能碎片化 | 固定缓冲区/ArduinoJson 静态文档 |
| `safeDelay(sampleMs)` | 周期漂移 | `vTaskDelayUntil` |
| 队列/任务栈固定 | 资源未经量化 | high-water mark、heap/PSRAM监控 |

## 10. 验证方案

- 外设：砝码多点标定、DHT 对照、GPS 回放/NMEA 异常注入；
- 并发：持续图传 + MQTT 断网重连 + OLED 刷新，检查 WDT、heap、stack watermark；
- 离线：断网超过缓存容量，确认覆盖策略、顺序、补发限速；
- OTA：成功、断电、坏包、版本回退、鉴权失败；
- 图传：分辨率/质量下的 FPS、端到端延迟、丢帧、PSRAM 和多客户端行为；
- 长稳：24/72 小时运行，监控重连次数、最小 heap、任务水位和复位原因。
