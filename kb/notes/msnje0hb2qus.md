# 华为实习：5G信道估计与Ascend推理优化完整八股知识库

## 0. 项目知识树与成果边界

知识树：

`5G NR物理层 → OFDM/导频/信道估计 → 复数张量 → Transformer/Attention → ONNX IR/Shape → ATC图编译/融合 → Ascend存储与Cube/Vector/MTE → Profiling → FP16/INT8 → CCE-C自定义算子/Tiling/Softmax → ACL部署与数值验证`

完成度：

- `[实测]` P0图优化：618→272节点，572→324μs，全开融合320μs，cos 0.999999；
- `[实验回退]` Erf-GELU与FP16/INT8；
- `[工程原型]` 基础FusedAttention接口、Host Tiling、Kernel、插件和图替换，待ATC验证；
- `[方案设计]` 多核Head、Ping-Pong、GM→L1/L0流水，当前不是实测成果。

简历“节点降低37%”是早期冗余识别口径，最终节点实际下降约56%；简历将多核/双缓冲写成采用态，面试必须主动校正为设计态。

## 1. 5G NR与物理层基础

### 1.1 分层

应用数据经PDCP、RLC、MAC到PHY；L1完成编码、调制、资源映射、MIMO、OFDM、同步、信道估计、均衡和检测。信道估计是后续解调/波束赋形的前置依赖，因此时延进入时隙预算。

### 1.2 OFDM

把宽带频率选择性信道分成多个窄带正交子载波；IFFT生成时域符号，接收端FFT回频域。循环前缀将线性卷积近似为循环卷积，在CP足够时降低ISI并使频域一拍均衡成立。

### 1.3 资源网格与RB

时频网格由OFDM symbol和subcarrier组成，RB是资源分配基本单位。10RB/20RB会改变频域长度；现有模型权重相同但频域Shape主要为32/48，仍需不同OM/动态档位和Tiling验证。

### 1.4 SRS与DMRS

- SRS：终端上行探测参考信号，帮助基站获取更宽频域的上行信道；
- DMRS：与物理信道相关的解调参考信号，用于解调时估计信道。

二者都是已知参考，但资源位置和业务用途不同。

### 1.5 MIMO信道

`Y=HX+N`。多天线时H是复数矩阵/张量，描述不同Tx-Rx、时间、频率位置的幅度和相位。估计误差会影响均衡、检测、CQI和波束决策。

### 1.6 传统LS/MMSE

LS在导频位置近似 `Ĥ=Y/X`，简单但噪声敏感；MMSE利用信道/噪声统计降低误差，但需协方差和矩阵运算。AI方法可学习非线性先验，但有域外泛化、可解释和部署成本。

### 1.7 插值与时变

导频只占部分资源，需在时频维插值。高速移动增大多普勒、降低相干时间；大时延扩展降低相干带宽。模型训练分布必须覆盖对应场景。

## 2. 复数信号与指标

### 2.1 I/Q

基带复数 `x=I+jQ` 同时表示幅度和相位。框架/硬件可能拆成实虚通道或额外末维，所有Reshape/Transpose必须保持配对语义。

### 2.2 复数相关性

```text
corr = |Σ H_ref·conj(H_pred)| /
       sqrt(Σ|H_ref|² Σ|H_pred|²)
```

它衡量复数幅相结构的整体对齐，但对整体尺度不完全敏感，要配合MSE和最大误差。

### 2.3 MSE与余弦

MSE看逐元素绝对误差，受量纲/幅值影响；余弦看方向，整体缩放仍可能很高。cos 0.999999不等于bit-exact，也不代表所有业务样本都通过。

### 2.4 浮点误差

不同融合和归约顺序导致舍入差异；浮点加法不满足结合律。验收使用业务阈值而不是逐bit比较，同时检查NaN/Inf和最差样本。

## 3. SPF/BESA_AI架构

```text
L1 IPC SRS/DMRS输入
→ SPF粗估计
→ CVSA特征增强
→ Decoder精细估计
→ AdapterNet配置适配
→ IPC返回L1
```

生产部署为 `.pth→ONNX→ATC→OM→Python验证→C++ BESA_AI_APP`，应用数据流可概括 `PacketParse→ModelInfer→PacketSend`。本次成果针对SPF_27B_10RB部署侧，不应扩大到整个系统端到端收益。

## 4. Transformer八股

### 4.1 模型事实

输入输出FP32 `(1,4,32,32,1)`；4层Transformer；约25,428参数；Conv18、MatMul8、Softmax4、Tanh4；后期完整报告约48.9M MACs，Cube61.8%、Vector38.2%。

### 4.2 Self-Attention

```text
Q=XWq, K=XWk, V=XWv
S=QKᵀ/√d_k
P=softmax(S)
O=PV
```

Q表示查询、K表示匹配键、V表示被聚合内容。注意力复杂度对序列长度通常为O(n²d)，小n时调度/搬运可能比MACs更突出。

### 4.3 为什么缩放

若Q/K分量独立同方差，点积方差随d_k增长；除以 `√d_k` 保持logit尺度，防Softmax饱和。

### 4.4 Multi-Head

把通道分为多个子空间，分别计算注意力再拼接。Head之间在拼接前独立，给自定义算子按Head多核并行提供天然边界。

### 4.5 Softmax稳定实现

```text
m=max(x)
p_i=exp(x_i-m)/Σexp(x_j-m)
```

减最大值利用平移不变性避免exp溢出。FP16下归约可用FP32累加；mask位置要用足够小值且不参与归一化。

### 4.6 LayerNorm

对每个样本的特征维计算均值方差：`(x-μ)/sqrt(σ²+ε)*γ+β`。与BatchNorm不同，不依赖batch统计，适合Transformer和batch1。epsilon、归约精度和融合都影响数值。

### 4.7 Residual

`y=x+F(x)`改善信息和梯度传播。部署中Add可能与前后算子融合；dtype/Shape必须一致，额外Cast会切断链路。

### 4.8 FFN与GELU

逐token的两层投影+非线性。GELU可用Tanh近似或Erf精确式；数学等价不保证编译器图等价。

### 4.9 Conv投影

1x1/小卷积可等价承担通道线性投影，并匹配既有布局。是否高效取决于Shape、Cube分块和格式，而不是算子名字。

## 5. ONNX IR与计算图

### 5.1 Graph组成

Graph包含Node、input/output、initializer、value_info；Node有op_type、domain、attribute和张量边。ONNX是语义图，不等同于设备Kernel图。

### 5.2 opset与domain

opset约束标准算子语义版本；自定义算子用自定义domain。ATC是否支持取决于op、version、attribute、dtype和Shape组合。

### 5.3 静态/动态Shape

动态Shape提供灵活性，却使编译期常量推导、内存规划和融合更难。固定batch/10RB配置适合静态化；20RB需独立验证。

### 5.4 Shape推导

ONNX shape inference依据算子schema传播维度，但遇到动态计算、Reshape target和自定义算子时可能不完整。手工KNOWN_SHAPES是补充，也是高风险信任边界。

### 5.5 Constant Folding

当节点所有输入在编译期已知时提前计算并替换为Constant。需迭代，因为上一轮的新常量会使下游继续可折叠。

### 5.6 DCE

从graph outputs反向遍历生产者，保留可达闭包，删除其余Node/initializer。纯函数推理图安全；若有副作用/随机状态需特殊处理。

### 5.7 Topological Order

ONNX节点需满足依赖拓扑。图重写后要重建producer/consumer、name唯一性和value_info，并运行checker/shape inference。

### 5.8 Node、Subgraph、Kernel

618个ONNX Node不等于618个Kernel。ATC可多Node融合一Kernel，也可把一Node拆Task。节点下降是结构线索，时延必须看最终OM。

## 6. P0图优化闭环

### 6.1 原始冗余

Shape36、Gather36、Unsqueeze96、Constant226。固定输入下大量运行时Shape链无业务价值，且把Conv/Add/LayerNorm等融合模式隔开。

### 6.2 实现

固定输入→注入关键中间Shape→最多30轮常量折叠→从输出反向DCE→checker/ORT→重新ATC→OM精度/性能回归。

### 6.3 为什么通用simplifier不够

动态Shape/版本兼容可能让推导不完整，Windows底层工具还发生崩溃。受控脚本能只折叠已证明常量的节点并保留回归链。

### 6.4 错误KNOWN_SHAPES

曾把骨干写成 `[1,4,4,32]`，正确为 `[1,32,16,32]`。ATC可编译但OM运行Reshape崩溃，说明compile success不是语义正确证据。

### 6.5 旧OM问题

修改ONNX但未重新ATC，脚本仍加载旧产物，形成“优化无效”假象。解决：产物hash/manifest、隔离目录、绝对路径与编译时间打印。

### 6.6 结果

618→272（约56%）；Shape36→0、Gather36→0、Unsqueeze96→1、Constant226→88；ATC碎片子图收敛；572→324/320μs；cos 0.999999。

加速来自减少调度、减少中间GM读写和恢复融合，不是“删346个节点就省346个Kernel”。

## 7. 编译器与算子融合

### 7.1 编译流程

前端解析/规范化→Shape/类型推导→图优化→融合pattern匹配→算子选择→格式插入→内存规划→任务/Kernel生成。

### 7.2 融合的收益

减少Kernel launch、同步、GM中间写回和TransData；有时还能共享归约结果。融合也可能增加片上压力、寄存器/UB占用和尾块复杂度。

### 7.3 Pattern脆弱性

融合通常匹配特定op拓扑、常量形式、dtype和attribute。一个Cast、Transpose或不同GELU表达都可能让pattern失配。

### 7.4 Erf实验

ORT cos1.0、MSE `7.07e-9`；NPU 320→458μs。Tanh版本命中CANN专属GELU融合，Erf被拆Kernel。结论：算法等价≠编译图等价≠硬件性能等价。

## 8. Ascend硬件与存储层次

### 8.1 AI Core功能单元

- Cube：矩阵/卷积高吞吐；
- Vector：激活、归一化、逐元素/归约；
- Scalar：控制与标量；
- MTE：GM与片上层级搬运。

实际架构细节以310P3/CANN公开文档为准，不臆造核数/带宽。

### 8.2 存储

GM容量大、延迟高；L1做较大块缓存/复用；L0A/L0B喂Cube输入，L0C累加；UB适合Vector和搬运。Kernel优化核心是让搬运与计算重叠并增加片上复用。

### 8.3 数据格式

逻辑ND/NCHW与内部Fractal/5D storage format不同。TransData改变布局；单算子dump若未反排布会相关性低，不能直接判算错。

### 8.4 对齐与小矩阵

Cube喜欢特定tile/对齐；模型 `[16,32]×[32,16]` 等小矩阵难填满计算阵列。Padding虽增MACs，可能改善硬件利用，也可能得不偿失，需实测。

### 8.5 Roofline

算术强度=operations/bytes。低强度偏带宽受限，优先融合/复用；高强度偏计算受限，关注Cube/Vector利用。小Kernel还受launch和尾块限制，单一Roofline不够。

## 9. Profiling方法

### 9.1 基线控制

固定硬件、驱动/CANN、OM、Shape、精度、输入、预热、循环和计时边界。572μs是OM稳态口径，不等于含IPC/H2D/D2H的完整L1时延。

### 9.2 msprof观察

模型/Task/Kernel时间线、算子耗时、执行间隙、数据搬运和硬件利用线索。Profiling有扰动，最终数字要轻量复测。

### 9.3 CPU/NPU对照

CPU ONNX约7171μs、NPU约572μs，约12.5×，可说明部署收益；不同后端不能作为严格同硬件优化A/B。P0前后同NPU才是核心。

### 9.4 同系列数据

MTE 54.4%等部分来自Spf_256T，只能作为“搬运可能是瓶颈”的参考假设，不能写成目标模型直接实测。

### 9.5 理论MACs冲突

早期2.4M估算不完整，主口径采用后期逐算子48.9M。理论时延忽略launch、格式、搬运和利用率，不能替代实测。

## 10. 浮点与混合精度

### 10.1 FP32、FP16、BF16、INT8

- FP32：1/8/23，范围和精度较好；
- FP16：1/5/10，范围小、精度低；
- BF16：1/8/7，范围接近FP32但尾数少；
- INT8：定点量化，存储/吞吐优势大但需scale/zero-point。

具体硬件支持与性能以310P3为准。

### 10.2 FP16为什么可能变慢

FP32接口和敏感算子间插Cast，打断融合；Vector路径未必理想加速；小Shape下转换/launch盖过带宽收益。

### 10.3 项目结果

allow_mix_precision318μs（基本未生效）；force_fp16 540；手工FP16 605；mixlist531；均未优于320基线。

### 10.4 混精度策略

Conv/MatMul优先低精度，Softmax/LayerNorm/归约/输出保高精度；但边界Cast要联合融合图评估。精度敏感度与性能敏感度必须一起做。

## 11. INT8量化

### 11.1 线性量化

`q=round(x/scale)+zero_point`，反量化 `x≈scale*(q-zero_point)`。对称量化zero≈0，硬件简单；非对称更贴合偏移分布。

### 11.2 Per-tensor vs Per-channel

Per-tensor一个scale简单但易被异常通道支配；权重per-channel通常精度更好，硬件/算子支持需确认。

### 11.3 PTQ vs QAT

PTQ训练后用校准集估范围，成本低；QAT训练中模拟量化，精度更好但需数据和训练。项目探索停在部署PTQ工具链阶段。

### 11.4 校准

覆盖真实SNR、幅相、信道场景和配置；MinMax简单但怕离群，Percentile/KL等可选。校准集与测试集分离。

### 11.5 QDQ与工具链

标准ONNX QuantizeLinear/DequantizeLinear不保证ATC 8.0.RC3直接接受；项目QDQ编译失败、AMCT不可用，所以不能说INT8完成。

## 12. CCE-C自定义算子

### 12.1 何时自定义

先用图优化/官方融合；只有热点稳定、通用编译器无法优化且收益覆盖维护成本时自定义。自定义算子绑定芯片、CANN和Shape，测试成本高。

### 12.2 工程组成

算子原型/注册与Shape推导→Host Tiling与workspace→Kernel CopyIn/Compute/CopyOut→ONNX适配插件→图替换→编译安装→ATC→单算子/整图验证。

### 12.3 Host Tiling

根据batch/head/sequence/head_dim/dtype、UB/L1/L0容量、对齐和核数计算tile、每核工作、尾块、GM offset和临时空间。TilingData的ABI必须与Kernel一致。

### 12.4 Block与核并行

`blockDim`决定启动多少核/Block。当前基础版本为1，不能说多核已实现。按Head划分通信少，但Head少时并行度不足，尾部需负载均衡。

### 12.5 Queue/Pipe语义

Ascend C常用LocalTensor、Queue和Pipe组织搬入/计算/搬出。Buffer数量、EnQue/DeQue/FreeTensor顺序错误会产生覆盖或死锁。

## 13. FusedAttention深度八股

### 13.1 融合目标

将 `QKᵀ→Scale→Softmax→PV` 尽量一个Kernel，避免完整Score/Probability反复写GM，减少调度与TransData。

### 13.2 分块矩阵乘

Q行块与K列块计算Score tile；K/V可在片上复用。tile需适配Cube基本块和片上容量；尾块padding并mask。

### 13.3 在线Softmax

块j有局部max `m_j` 和sum `l_j`。合并时：

```text
m_new=max(m_old,m_j)
l_new=exp(m_old-m_new)l_old + exp(m_j-m_new)l_j
```

同时按相同比例重标定已累计的输出，不能对每块独立softmax后直接拼接。

### 13.4 Ping-Pong

两组Buffer交替：搬入tile n+1时计算tile n，并可搬出更早结果。只有DMA/MTE与计算确实异步、依赖正确且资源足够时才有收益。

### 13.5 GM→L1→L0

GM大块搬到L1，再分块送L0A/B给Cube，L0C累加；减少K/V重复读GM。代价是同步、地址和容量规划复杂。

### 13.6 融合风险

片上空间不足导致切块更多，寄存器/Buffer压力降低占用，Softmax精度和mask错误，维护与跨版本成本上升。因此“4Kernel→1”不保证更快。

### 13.7 当前边界

基础工程待ATC验证，多核/Ping-Pong/三级流水只是详设。`320→200/95μs` 只能是目标估算，不能作为结果。

## 14. ACL与C++部署

### 14.1 资源生命周期

`aclInit→aclrtSetDevice→CreateContext/Stream→aclmdlLoad→desc/dataset/buffer→H2D→aclmdlExecuteAsync→同步→D2H→逆序释放`。

### 14.2 Async不等于并行

提交到Stream后若立即synchronize，仍是串行关键路径。重叠H2D/Compute/D2H需多Buffer、多Stream/Event和无数据依赖，并用时间线验证。

### 14.3 内存

预分配和复用减少抖动；Host pageable/pinned、Device Buffer和拷贝方向要正确；接口字节数=`元素数*sizeof(dtype)`，内部storage format不应由业务侧猜测。

### 14.4 IPC集成

PacketParse验证长度/版本/Shape，ModelInfer处理Buffer与错误码，PacketSend保留序号/时间戳。生产需超时、错误降级、模型热切换和线程安全。

## 15. 精度调试

### 15.1 forward_cvsa.py

OM加载→Device Buffer→H2D→预热/多轮推理→D2H→实虚重构→归一化→复数相关/MSE。常用平均相关门限≥0.99，但应以业务规范为准。

### 15.2 逐节点比较

固定输入，统一预处理、dtype、logical/storage format；按拓扑或二分找首个误差边界。融合后节点名不一一对应，可在融合边界dump。

### 15.3 Conv dump陷阱

Fractal/5D格式未还原会低相关；先读descriptor和真实storage shape，再TransData/反排布。端到端正确而单点异常时优先怀疑格式。

### 15.4 NaN/Inf

排查输入范围、除零、LayerNorm epsilon、Softmax exp、Cast溢出、自定义Kernel尾块mask与越界；记录首次出现节点。

## 16. 性能优化决策树

1. 复现和精度正确；
2. 固定基线与计时边界；
3. 图/Profiling定位计算、搬运、调度、同步；
4. 先低风险静态化、常量折叠和融合恢复；
5. 再算法等价改写与混精度A/B；
6. 最后自定义算子；
7. 每步重新编译，同时验收性能和业务精度，保留回退。

## 17. 生产化与发布

- OM绑定ONNX hash、SoC、CANN/驱动、Shape、精度和ATC参数；
- 10RB/20RB分别回归；
- 金样覆盖SNR、信道条件和边界值；
- 性能看P50/P95/最大、长稳、温度和系统端到端；
- 异常时回退旧OM或传统信道估计；
- 自定义算子随CANN升级做ABI/精度/性能回归；
- 发布前保存编译日志、Profiling、精度报告和可追溯产物。

## 18. 高频口径纠偏

- 节点37%：早期动态Shape类冗余口径；最终618→272约56%。
- 时延44.1%：572→320μs，OM稳态；不要扩大为L1全链路。
- P1/P2：做过且回退，是方法论成果，不是上线优化。
- FusedAttention：基础工程/设计，不报实测加速。
- MTE54.4%：同系列参考，不是目标模型直接数据。
- MACs：采用后期48.9M，不采用早期2.4M粗估。

## 19. 官方与权威资料索引

- [Attention Is All You Need](https://arxiv.org/abs/1706.03762)
- [ONNX Concepts](https://onnx.ai/onnx/intro/concepts.html)
- [ONNX Shape Inference](https://onnx.ai/onnx/api/shape_inference.html)
- [Ascend ATC Tool Guide](https://www.hiascend.com/document/detail/zh/canncommercial/80RC3/devaids/devtools/atc/atlasatc_16_0001.html)
- [华为昇腾社区文档中心](https://www.hiascend.com/document)

## 20. 无盲区自检

必须能解释：OFDM/CP/RB/SRS/DMRS/LS/MMSE、复数指标、Attention/Softmax/LayerNorm/GELU、ONNX/opset/Shape/折叠/DCE、融合pattern、Cube/Vector/MTE/存储层、Roofline/msprof、FP16/INT8/QDQ、CCE-C/Tiling/在线Softmax/Ping-Pong、ACL异步、逐算子格式和所有数字真实性边界。
