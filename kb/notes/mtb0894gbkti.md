# A4：MQTT遥测与HTTP图传双协议设计

## 1. 对应简历条目

> **外设驱动与物联网通信：**通过MQTT/Alink完成遥测上云，ESP32-CAM采用HTTP＋I²S/DMA＋PSRAM双帧缓冲传输JPEG，实现控制流与图像数据流分离。

这一部分的核心叙事是：

> 系统同时存在低频、小数据量的传感器遥测，以及高带宽、大数据量的连续图像。两类数据特征差异很大，因此没有强行共用一种协议，而是让MQTT负责遥测、状态和控制，让HTTP负责JPEG抓拍和MJPEG图像流。

---

## 2. 为什么设计两条通信链路

智慧蜂箱主要传输两类数据。

### 2.1 传感器遥测

包括：

- 温度；
- 湿度；
- 蜂箱重量；
- GPS位置；
- 卫星数量；
- 网络RSSI；
- 告警位；
- 设备状态。

这些数据具有以下特点：

- 单条消息只有几十到几百字节；
- 以秒级或分钟级周期上传；
- 需要按设备和主题管理；
- 需要支持设备状态和云端指令；
- 对偶发丢失的容忍度相对较高。

这类数据适合MQTT。

### 2.2 图像数据

包括：

- 单帧JPEG；
- 连续MJPEG视频流。

这些数据具有以下特点：

- 单帧可能达到数十KB；
- 连续图传数据量远高于普通遥测；
- PC端需要直接获取图像并解码；
- 更关注吞吐量、帧率和观看延迟；
- 不适合转成JSON等文本消息。

这类数据更适合HTTP。

### =={pink}2.3 两条链路的职责==

| 链路 | 协议 | 数据类型 | 通信方式 |
| --- | --- | --- | --- |
| 遥测与控制链路 | MQTT/Alink | 温湿度、重量、GPS、告警、设备状态、下行指令 | 设备发布/订阅，云平台中转 |
| 图像数据链路 | HTTP | JPEG抓拍、MJPEG连续图像 | PC或后端主动请求ESP32-CAM |

因此可以将它们理解为：

- **=={yellow}MQTT：轻量遥测==与控制平面；**
- **=={yellow}HTTP：高带宽图像==数据平面。**

---

## 3. 整体通信架构

---

## =={pink}4. MQTT遥测链路==

### 4.1 为什么选择MQTT

=={yellow}MQTT是**面向资源受限设备**的**发布/订阅协议**==，适合智慧蜂箱的原因包括：

#### =={green}1.协议开销较小==

MQTT报文头较小，适合周期发送少量传感器数据。

#### =={green}2.发布者和消费者解耦==

ESP32-S3只需要向指定Topic发布数据，不需要知道最终是后端、数据库还是大屏使用这些数据。

```markdown
ESP32发布者
     ↓
阿里云IoT Broker
     ↓
后端消费者
```

设备与后端不需要保持直接的点对点连接。

#### =={green}3.适合多设备扩展==

每个蜂箱可以拥有独立设备身份和Topic，后端根据设备ID区分不同蜂箱。

#### =={green}4.支持双向通信==

除了设备上报，云端还可以向设备下发：

- 采样周期；
- 配置参数；
- OTA固件地址；
- 维护指令。

#### =={green}5.提供连接保活机制==

MQTT通过**Keep Alive**和**PING报文**=={yellow}检测连接是否仍然有效==。

---

## =={pink}5. 阿里云设备接入与Alink物模型==

### =={yellow}5.1 产品和设备注册==

我在阿里云IoT平台完成：

1. =={green}创建==智慧蜂箱=={green}**产品**==；
2. =={green}定义==产品=={green}**节点**和**联网方式**==；
3. 为具体蜂箱=={green}创建设备**实例**==；
4. =={green}获取**设备三元组**==；
5. =={green}在**固件**中**配置MQTT连接参数**==。

=={yellow}设备三元组包括：==

- `ProductKey`；
- `DeviceName`；
- `DeviceSecret`。

=={yellow}三元组用于==生成=={yellow}MQTT连接==所需的：

- Client ID；
- Username；
- Password或签名。

不能将真实设备密钥提交到公开代码仓库，应通过配置文件、构建参数或安全存储管理。

### =={pink}5.2 物模型定义==

物模型负责=={yellow}规定**云端如何理解设备数据**==。

项目中需要定义的属性主要包括：

- 当前温度；
- 当前湿度；
- 蜂箱重量；
- 经度；
- 纬度；
- GPS有效状态；
- 设备在线状态；
- 告警状态。

每个属性需要明确：

- 标识符；
- 数据类型；
- 单位；
- 取值范围；
- 读写权限。

例如重量必须统一使用克或千克，不能设备端上传千克而后端按克解释。

### =={pink}5.3 Alink消息格式==

设备=={yellow}将**遥测封装成Alink JSON**==。示意格式为：

```json
{
  "id": "12345",
  "version": "1.0",
  "params": {
    "CurrentTemperature": 26.4,
    "CurrentHumidity": 61.2,
    "HoneyWeight": 12.36,
    "Latitude": 39.9000,
    "Longitude": 116.4000,
    "AlarmFlags": 0
  },
  "method": "thing.event.property.post"
}
```

其中具体字段标识符需要与阿里云物模型保持一致。

设备向属性上报Topic发布后，阿里云按照物模型解析并展示数据，后端再通过规则引擎或消息订阅获取数据。

---

## =={pink}6. MQTT在设备端如何实现==

### =={pink}6.1 初始化网络客户端==

项目使用：

- `WiFiClient`：=={yellow}提供底层TCP连接；==
- `PubSubClient`=={yellow}：实现MQTT协议；==
- `ArduinoJson`=={yellow}：构造和解析JSON。==

```scss
WiFiClient wifiClient;
PubSubClient mqttClient(wifiClient);
```

初始化时配置：

- Broker地址；
- Broker端口；
- Keep Alive；
- Socket超时；
- 消息回调函数。

### =={pink}6.2 连接流程==

```markdown
检查Wi-Fi状态
      ↓
未连接：尝试连接Wi-Fi
      ↓
Wi-Fi成功
      ↓
使用设备身份连接MQTT Broker
      ↓
连接成功后订阅命令Topic
      ↓
设置EventGroup中的MQTT状态位
```

> Broker = 中间人/代理/消息中转站，核心作用是接收、路由、分发消息，让发送方和接收方解耦，不直接通信。

网络任务不会在传感器采集路径中无限等待连接，而是按时间间隔尝试重连。

当前**重连策略**大致为：

- =={yellow}**Wi-Fi**每**5**秒检查并尝试重连；==
- =={yellow}**MQTT**每**3**秒检查并尝试重连；==
- =={yellow}MQTT **Keep Alive**为**60**秒。==

### =={pink}6.3 遥测消息封装==

采集任务生成`TelemetryMessage`，=={yellow}**上传任务**将其序列化为JSON==：

```json
{
  "device_id": "beehive-s3-001",
  "fw": "s3-main-v3.0.0",
  "seq": 1024,
  "uptime_ms": 826000,
  "temperature_c": 26.4,
  "humidity": 61.2,
  "weight_kg": 12.36,
  "gps_valid": true,
  "lat": 39.9000,
  "lng": 116.4000,
  "satellites": 8,
  "rssi": -58,
  "alarm": 0,
  "offline_replay": false
}
```

在阿里云环境中，将这些业务字段映射到Alink的`params`中。

### =={pink}6.4 Topic设计==

最终架构逻辑上划分为=={yellow}**三类Topic**==：

```bash
beehive/<device_id>/telemetry
beehive/<device_id>/status
beehive/<device_id>/cmd
```

分别用于：

- **=={yellow}周期遥测==**=={yellow}；==
- **=={yellow}设备==**=={yellow}在线和运行==**=={yellow}状态==**=={yellow}；==
- =={yellow}云端==**=={yellow}下行指令==**=={yellow}。==

如果接入阿里云标准物模型，则替换为阿里云规定的属性上报和服务下发Topic，但三类数据职责不变。

### =={pink}6.5 QoS选择==

项目的=={yellow}**周期遥测**主要使用**QoS 0**==。

> QoS 0（Quality of Service Level 0，服务质量等级0）是MQTT协议中定义的三种消息传递服务质量等级之一，也是最低、最简单的等级。

**核心特点**

=={yellow}"至多一次"==（At most once）=={yellow}不保证送达==：消息发送后不做任何确认，不保证接收方一定能收到=={yellow}不重试==：发送失败不会重新发送=={yellow}发后即焚==：发送方不需要存储消息状态，发送完即可丢弃=={yellow}性能最高==：因为不需要握手确认，传输开销最小，速度最快

**原因是：**

- =={green}温湿度和重量会**周期更新**==；
- =={green}**偶发丢失**一条**不会影响**当前状态==；
- =={green}QoS 0**开销较小**；==
- 适合资源和网络带宽受限的设备。

代价是：

- Broker不返回发布确认；
- `publish()`返回成功只代表本地发送过程没有立即失败；
- 不能保证后端一定完成处理。

因此，不能说QoS 0实现了“消息可靠送达”。

对于=={pink}防盗或严重告警==，更合理的产品化方案是：

- =={yellow}使用QoS 1；==
- 增加业务序号；
- 后端发送应用层ACK；
- 设备超时重传；
- 后端根据设备ID和序号去重。

### 6.6 下行命令

设备订阅命令Topic，接收JSON指令。

例如修改采样周期：

```json
{
  "sample_interval_ms": 10000
}
```

设备处理流程：

```javascript
收到MQTT消息
      ↓
解析JSON
      ↓
检查参数范围
      ↓
更新SharedState
      ↓
写入NVS
```

例如下发OTA地址：

```json
{
  "ota_url": "http://server/firmware.bin"
}
```

网络任务读取待处理地址，再执行HTTP固件下载和更新。

### =={pink}6.7 MQTT线程安全==

`PubSubClient`=={yellow}会被**两个任务使用**：==

- =={yellow}**网络任务**==调用`connect()`和`loop()`；
- =={yellow}**上传任务**==调用`publish()`。

因为它不是线程安全对象，所以=={yellow}使用**互斥锁**==`mqttMutex`=={yellow}**保护**。否则两个任务可能同时修改Socket状态和收发缓冲。==

---

## =={pink}7. MQTT断网处理==

### =={pink}7.1 采集与联网解耦==

=={yellow}网络断开后：==

- =={yellow}传感器任务继续采集==；
- =={yellow}GPS任务继续解析；==
- =={yellow}OLED继续显示；==
- =={yellow}上传任务停止真实发布；==
- =={yellow}遥测写入RAM环形缓存。==

### =={pink}7.2 恢复补发==

MQTT恢复后：

- 上传任务先检查缓存；
- 每轮最多补发4条；
- 每条之间主动让出CPU；
- 补发失败时重新放回缓存；
- 同时继续处理新数据。

这样避免网络恢复瞬间大量历史消息抢占链路。

---

## =={pink}8. HTTP图像链路==

### =={pink}8.1 为什么使用HTTP==

ESP32-CAM=={yellow}采集的是JPEG二进制数据。HTTP天然支持==：

- 直接返回`image/jpeg`；
- 通过普通浏览器查看；
- OpenCV直接读取；
- 使用MJPEG连续传输图像；
- 无须在设备端对图像分片和重组。

=={yellow}HTTP在这里采用**请求—响应模式**：==

```markdown
PC端发起HTTP GET
        ↓
ESP32-CAM采集或取得一帧
        ↓
返回JPEG或持续返回MJPEG
```

PC或后端是客户端，ESP32-CAM是图像服务器。

---

## =={pink}9. ESP32-CAM图像采集路径==

=={yellow}图像采集路径为==：

```css
OV2640感光与JPEG压缩
        ↓
DVP并行像素数据
        ↓
esp_camera驱动
        ↓
I²S/DMA搬运
        ↓
PSRAM 帧缓冲区
        ↓
HTTP响应
```

### =={pink}9.1 为什么使用JPEG==

如果使用RGB原始图像，单帧数据量很大。

以320×240、RGB565为例：

`320 x 240 x 2 = 153600 Bytes`

一帧约150KB。JPEG=={yellow}压缩后通常可以显著减小数据量==，更适合ESP32-CAM通过Wi-Fi传输。

### =={pink}9.2 PSRAM双帧缓冲==

检测到PSRAM后配置：

- Frame Buffer位于PSRAM；
- `fb_count = 2`；
- `CAMERA_GRAB_LATEST`；
- 优先获取最新帧。

=={yellow}**双帧缓冲**的作用是：==

- =={yellow}**一个缓冲区可能正在被网络发送**；==
- =={yellow}**另一个缓冲区可以接收新图像**；==
- =={yellow}**减少**==采集与发送完全=={yellow}**串行**造成的**等待**==；
- 优先最新帧，降低实时观看的积压延迟。

没有PSRAM时降级为：

- 单帧DRAM缓冲；
- QVGA分辨率；
- 空缓冲时再采集。

---

## 10. HTTP接口设计

| 接口 | 方法 | 作用 |
| --- | --- | --- |
| `/` | GET | 本地图传页面 |
| `/capture` | GET | 获取单帧JPEG |
| `/stream` | GET | 获取MJPEG连续视频流 |
| `/status` | GET | 查看RSSI、Heap、PSRAM、分辨率等状态 |
| `/control` | GET | 调整分辨率和JPEG质量 |
| `/ota` | GET/POST | Web OTA页面和固件上传 |
| `/reboot` | GET | 重启摄像头节点 |

---

## 11. 单帧JPEG如何传输

客户端请求：

```bash
GET /capture HTTP/1.1
```

设备执行：

```scss
调用esp_camera_fb_get()
        ↓
获得camera_fb_t
        ↓
设置Content-Type: image/jpeg
        ↓
设置Content-Length
        ↓
将fb->buf写入TCP连接
        ↓
调用esp_camera_fb_return()
```

核心原则是：

> 获取帧缓冲后必须归还，否则连续请求会耗尽帧缓冲。

单帧接口适合：

- 定时抓拍；
- 数据集采集；
- 前端缩略图；
- 低频AI识别；
- 设备图像诊断。

---

## 12. MJPEG视频流如何实现

MJPEG不是复杂的视频编码格式，本质上是通过一个HTTP连接连续发送多张JPEG。

响应类型为：

```bash
Content-Type: multipart/x-mixed-replace; boundary=frame
```

每一帧大致为：

```css
--frame
Content-Type: image/jpeg
Content-Length: <本帧长度>

<JPEG二进制数据>
```

循环过程为：

```markdown
检查客户端是否连接
        ↓
获取一帧JPEG
        ↓
发送boundary和HTTP头
        ↓
发送JPEG数据
        ↓
归还Frame Buffer
        ↓
短暂延时
        ↓
获取下一帧
```

PC端可以直接使用：

```ini
cap = cv2.VideoCapture("http://<cam-ip>/stream")
```

读取帧并交给YOLOv8。

---

## =={pink}13. 为什么图像不通过MQTT传输==

MQTT协议本身可以发送二进制大消息，并不是“完全不能传图片”。但在本项目中不适合承担连续图传。

=={green}如果使用MQTT传输图片，可能需要：==

1. =={yellow}**将图像拆成多个分片**；==
2. =={yellow}为每片增加图片ID、序号和总片数==；
3. =={yellow}处理分片丢失；==
4. =={yellow}在接**收端重新排序和组装**；==
5. 设置较大的MQTT缓冲区；
6. 处理单张图像跨多个消息的超时；
7. 承担Broker的大消息和高频流量压力。

=={yellow}连续图像还可能占用MQTT链路==，=={yellow}使以下消息延迟==：

- 温湿度遥测；
- 防盗告警；
- 心跳；
- OTA命令；
- 参数配置。

相比之下，=={yellow}HTTP可以**直接发送完整JPEG**，并使用长连接持续发送MJPEG==，更符合浏览器和OpenCV的使用方式。

---

## =={pink}14. 两条链路分离带来的好处==

### =={yellow}14.1 资源隔离==

- 主控端不会为图像分片占用大量RAM；
- ESP32-CAM独立使用PSRAM；
- 图像流不会占用主控端MQTT客户端缓冲区。

### 14.2 故障隔离

- 图传中断时，温湿度和重量仍可上传；
- MQTT重连时，局域网图像仍可能继续使用；
- 摄像头节点故障不影响主控采集。

### 14.3 独立调优

MQTT链路可以单独调整：

- 采样周期；
- QoS；
- 缓存；
- 重连策略。

HTTP链路可以单独调整：

- 图像分辨率；
- JPEG质量；
- 帧率；
-PSRAM缓冲策略。

### 14.4 消费端适配简单

- 后端适合处理JSON遥测；
- 浏览器和OpenCV适合读取HTTP图像；
- 不需要为图片单独实现MQTT重组工具。

---

## 15. 当前HTTP方案的边界

当前ESP32-CAM使用同步`WebServer`。

`/stream`处理函数内部存在持续发送循环，因此：

- 一个流客户端可能长期占用请求处理流程；
- `stream_clients`计数不代表真正支持多客户端并发；
- 流传输期间其他HTTP接口的响应能力可能下降；
- 不适合直接描述成高并发图传服务器。

产品化可以改进为：

- ESP-IDF异步HTTP服务器；
- 独立摄像头采集任务；
- 共享最新帧；
- 每个客户端独立发送队列；
- 限制最大客户端数量；
- 增加鉴权和TLS；
- 增加公网访问代理或边缘网关。

此外，直接由PC访问ESP32-CAM更适合局域网环境。跨公网部署时，还需要解决NAT、动态IP和安全访问问题。

---

## 16. 两条链路的测试方法

### MQTT链路

- 在阿里云控制台确认设备上线；
- 检查物模型属性是否正确；
- 对比终端串口值与云端值；
- 验证单位和字段类型；
- 断开Wi-Fi，确认采集仍继续；
- 恢复网络，检查缓存是否补发；
- 下发采样周期，确认设备更新并写入NVS；
- 长时间运行，检查心跳和重连次数。

### HTTP链路

- 访问`/capture`检查单帧JPEG；
- 访问`/stream`检查连续图像；
- 使用OpenCV读取MJPEG；
- 检查YOLOv8是否能够接收正确帧；
- 调整分辨率和JPEG质量；
- 持续图传时观察Heap和PSRAM；
- 反复连接和断开客户端；
- 检查每次取帧后是否正确归还缓冲；
- 验证图传压力不会影响主控端MQTT遥测。

---

## 17. 面试口述版

> 项目中同时存在两类数据：一类是温湿度、重量和GPS等低频、小数据量遥测；另一类是摄像头产生的高带宽JPEG图像。两类数据的特点不同，所以我没有强行使用一种协议，而是设计了MQTT和HTTP两条独立链路。
> 
> 
> 遥测侧由ESP32-S3通过Wi-Fi连接阿里云IoT平台。我在云端完成产品、设备和物模型配置，设备使用三元组鉴权，将传感器数据封装成Alink JSON后发布到属性上报Topic。网络管理任务负责Wi-Fi/MQTT重连和心跳，上传任务从Queue读取消息并发布；断网或发布失败时写入RAM环形缓存，恢复后限速补发。设备还订阅命令Topic，用于修改采样周期和下发OTA地址。
> 
> 
> 图像侧由ESP32-CAM提供HTTP接口。OV2640输出JPEG，底层通过`esp_camera`的I²S/DMA路径搬运到PSRAM双帧缓冲，`/capture`返回单帧JPEG，`/stream`使用multipart方式持续发送MJPEG，PC端YOLOv8直接从HTTP接口拉流。
> 
> 
> 这样设计避免了大图分片和重组占用MQTT链路，也让遥测与图传能够独立运行、独立调优和隔离故障。
