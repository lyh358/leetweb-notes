好的，我直接输出 Markdown 格式，你可以复制保存为 `.md` 文件，也可以用 Typora / VS Code / Obsidian 导出 PDF。

------

# 华为昇腾310B端侧AI推理优化实习指导文档

## 目标

将华为实习经历打造为秋招中的核心竞争力。

不要将自己定位为：

> 模型部署工程师

而应该定位为：

> **端侧AI推理优化工程师 / NPU性能优化工程师 / AI计算优化工程师**

核心能力链：

```
嵌入式底层
    ↓
Linux系统
    ↓
AI模型部署
    ↓
计算图优化
    ↓
NPU算子优化
    ↓
推理性能调优
```

最终形成：

> 懂硬件、懂嵌入式、懂AI计算优化的端侧智能工程师。

------

# 一、当前华为实习工作的正确定位

## 当前真实工作流程

你的工作链路：

```
算法团队
    |
    |
    ↓
ONNX模型
    |
    |
    ↓
ONNX分析
    |
    |
    ↓
ONNX → OM转换
    |
    |
    ↓
CANN编译
    |
    |
    ↓
Profiling性能分析
    |
    |
    ↓
定位性能瓶颈
    |
    |
    +----------------+
    |                |
    ↓                ↓
算子融合          单算子开发

    |
    |
    ↓

量化策略优化

    |
    |
    ↓

降低推理延迟
降低计算资源开销
```

因此你的核心不是：

> “把模型跑起来”

而是：

> **让模型在昇腾310B NPU上跑得更快、更省资源。**

------

# 二、秋招中的岗位定位

推荐方向：

## 1. 端侧AI推理优化工程师

关键词：

- Ascend310B
- CANN
- ONNX
- OM
- Profiling
- Operator Fusion
- Quantization
- Kernel Optimization

------

## 2. NPU性能优化工程师

关键词：

- AI Compiler
- Graph Optimization
- Operator Optimization
- Hardware Acceleration

------

## 3. 嵌入式AI软件工程师

结合你的：

- 禾赛车规MCU
- ESP32 FreeRTOS
- ROS C++
- 华为昇腾

形成：

```
MCU
 |
RTOS
 |
Linux Embedded
 |
NPU AI Acceleration
```

这是你的独特优势。

------

# 三、实习期间最高优先级积累方向

------

# Priority 1：形成完整性能优化闭环 ⭐⭐⭐⭐⭐

这是最重要的。

不要只做：

```
模型转换
 ↓
运行
```

而要形成：

```
发现问题

↓

Profiling分析

↓

定位瓶颈

↓

提出优化方案

↓

代码实现

↓

性能提升验证
```

------

## 建议记录指标

每个优化案例记录：

| 指标          | 优化前 | 优化后 |
| ------------- | ------ | ------ |
| 推理延迟      | xx ms  | xx ms  |
| 算子数量      | xx     | xx     |
| TOP耗时算子   | xxx    | xxx    |
| NPU利用率     | xx%    | xx%    |
| 显存/内存占用 | xx     | xx     |

------

最终简历：

> 基于CANN Profiling工具分析昇腾310B模型执行瓶颈，定位关键耗时算子，通过计算图优化和算子融合降低模型推理延迟。

------

# Priority 2：深入一个算子融合案例 ⭐⭐⭐⭐⭐

这是区分普通实习生和AI优化工程师的关键。

## 算子融合原理

例如：

优化前：

```
Conv
 |
BatchNorm
 |
ReLU
```

三个算子：

```
Kernel1
Kernel2
Kernel3
```

存在：

- 多次Kernel启动
- 中间Tensor写回DDR
- 数据重复搬运

优化后：

```
Conv + BN + ReLU Fusion
```

减少：

- Kernel Launch
- Memory Access
- Tensor Copy

提升：

- NPU利用率
- 推理速度

------

建议深入：

- Operator Fusion
- Graph Rewrite
- Constant Folding
- Dead Node Elimination
- Shape Optimization

------

# Priority 3：单算子开发 ⭐⭐⭐⭐⭐

这是非常有含金量的方向。

很多学生：

会：

```
PyTorch
ONNX
TensorRT
```

但是不会：

```
为什么这个算子慢？
如何优化？
如何让NPU更高效执行？
```

------

## 单算子开发流程

```
算子定义

↓

输入输出Shape设计

↓

Kernel实现

↓

Memory优化

↓

性能Benchmark

↓

精度验证
```

------

重点理解：

## NPU计算特点

包括：

- Cube计算单元
- Vector计算单元
- Cache结构
- DDR访问
- DMA搬运

------

# Priority 4：量化策略优化 ⭐⭐⭐⭐

目标：

降低计算量。

典型：

```
FP32

↓

FP16

↓

INT8
```

------

# PTQ流程

Post Training Quantization

```
FP32模型

↓

Calibration Dataset

↓

统计激活范围

↓

生成scale/zero point

↓

INT8模型
```

------

重点优化：

## 1. Calibration数据选择

错误：

随机数据。

正确：

覆盖真实输入分布。

------

## 2. 混合精度

不要：

全部INT8。

而是：

```
Conv:
INT8


Attention:
FP16


Softmax:
FP32
```

达到：

速度提升

同时：

保持精度。

------

# Priority 5：ONNX计算图优化 ⭐⭐⭐⭐

不要只会：

```
onnx → om
```

要理解：

```
ONNX Graph

↓

优化

↓

OM
```

关注：

- Node Fusion
- Graph Rewrite
- Constant Folding
- Operator Replacement

------

# Priority 6：端侧推理工程化 ⭐⭐⭐⭐

结合你的嵌入式背景。

建议做：

C++推理框架封装。

结构：

```
Application

↓

Inference API

↓

Model Manager

↓

ACL Runtime

↓

Ascend310B
```

功能：

- 模型加载
- Tensor管理
- 输入输出管理
- 推理接口
- 性能统计
- 异常处理

------

# 四、实习期间建议主动争取的任务

## 第一优先级

### 负责一个完整模型优化闭环

例如：

```
ONNX模型

↓

Profiling

↓

发现瓶颈

↓

算子优化

↓

重新编译OM

↓

性能提升
```

最终得到：

> 一个可以讲30分钟的面试案例。

------

## 第二优先级

### 一个自定义算子/融合案例

例如：

- LayerNorm
- Attention
- Resize
- Activation

最好有：

```
优化前:
50ms

优化后:
30ms

提升40%
```

------

## 第三优先级

### 一个量化优化案例

例如：

```
FP32

↓

INT8

↓

Latency下降30%

↓

Accuracy下降<1%
```

------

# 五、华为实习简历最终推荐写法

## 华为技术有限公司

### 端侧AI推理优化实习生

参与基于昇腾310B NPU平台的端侧深度学习模型部署与推理性能优化，负责ONNX模型分析、OM转换、计算图优化、算子优化及量化策略调优。

- 基于Ascend CANN工具链完成ONNX模型到OM模型转换，通过Profiling工具分析模型执行流程，定位关键耗时算子及性能瓶颈。
- 针对昇腾310B NPU计算特点，开展计算图优化与算子融合优化，减少Kernel调度及中间Tensor数据搬运，提高模型推理效率。
- 参与高性能算子开发与优化，包括算子接口设计、计算逻辑实现、性能测试和精度验证。
- 针对不同网络结构设计FP16/INT8混合精度量化策略，通过校准数据优化和精度回归测试，实现模型压缩与推理加速。
- 完成优化后模型在ARM64 Linux边缘设备上的部署验证，建立模型精度、推理时延和资源占用之间的优化闭环。

------

# 六、面试重点准备问题

## 1. 为什么算子融合可以降低延迟？

答案：

不是因为减少计算量。

主要原因：

- 减少Kernel启动次数
- 减少DDR访问
- 减少中间Tensor读写
- 提升计算密集度

------

## 2. ONNX为什么需要转换OM？

回答：

ONNX是通用模型交换格式。

OM是昇腾针对NPU优化后的执行模型。

转换过程：

```
ONNX

↓

图优化

↓

算子映射

↓

内存规划

↓

OM
```

------

## 3. INT8为什么更快？

因为：

- 数据量减少
- 内存带宽压力降低
- NPU整数计算效率更高

但是：

需要解决：

- 量化误差
- 精度下降

------

# 七、你的最终技术画像

你的目标不是：

> 会调用AI模型的嵌入式工程师

而是：

> **理解芯片底层资源，掌握嵌入式系统开发，同时具备AI计算图优化和NPU性能调优能力的端侧智能工程师。**

------

# 八、未来适合岗位

## AI芯片方向

- 华为昇腾
- 海思
- 寒武纪

## 自动驾驶方向

- 地平线
- 大疆
- 小米汽车
- 比亚迪智能驾驶

## 机器人方向

- 边缘计算
- AI机器人平台

## AIoT方向

- 智能终端
- 端侧大模型

------

# 最终建议

你的实习剩余时间不要追求“做更多模块”。

最有价值的是：

> **拿一个模型，完成一次完整的端侧AI性能优化闭环，并留下量化指标。**

一个优秀案例：

```
ONNX模型

↓

Profiling分析

↓

发现瓶颈

↓

算子融合/量化/Kernel优化

↓

OM重新编译

↓

310B部署

↓

Latency降低XX%
```

这会成为你秋招中最有竞争力的一段经历。