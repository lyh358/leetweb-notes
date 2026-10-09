# 昇腾 310P3（单芯片，Atlas300I Duo 卡为双 310P3）

# 昇腾 310P3 是**达芬奇 V200 架构推理芯片**，主打大模型 / 视频推理，Atlas300I Duo 推理卡集成**两颗 310P3**华为企业业...

## ✅ 单芯片算力上限（峰值）

- **INT8：140 TOPS**
- **FP16：70 TFLOPS**

> Atlas300I Duo（双芯整卡）：**280 TOPS INT8 / 140 TFLOPS FP16**华为企业业...

## ✅ 单芯片硬件资源

1. **AI Core：8 个 DaVinci V200 AI 核**
2. **CPU：8 核 泰山 V200M @1.9GHz**
3. **Vector 向量核：4 个**
4. **内存：LPDDR4X，单芯片 24GB ECC**
  - 双芯 Atlas300I Duo 整卡：**48GB / 96GB（可选）LPDDR4X，总带宽 408GB/s，支持 ECC**华为企业业...
5. 功耗：单芯片典型约 50W；整卡 Atlas300I Duo 最大功耗 100W

## ✅ 硬件编解码资源（单 310P3）

- H.264/H.265 硬件解码：128 路 1080P@30fps / 16 路 4K@60fps
- H.264/H.265 硬件编码：24 路 1080P@30fps / 3 路 4K@60fps
- JPEG：4K 512FPS 解码，4K 256FPS 编码
- 最大图像分辨率：8192×8192

## ✅ 软件侧资源（MindX，可虚拟化切分）

昇腾支持**NPU 虚拟化（容器 / 逻辑设备）**，单颗 310P3 可以切分成多个逻辑 NPU：

- 算力、内存、编解码资源可以按比例隔离分配
- 单芯片最大可切分 7 个逻辑设备（逻辑卡）
- 逻辑资源：算力、内存、媒体编解码通道都可做配额限制

## 补充说明

1. **峰值算力 ≠ 实际业务算力**：140TOPS 是理论峰值；受模型算子、访存带宽、量化精度、batch 大小影响，真实有效算力一般低于峰值。
2. 310P3**仅做推理，不支持训练**。
3. 芯片自带媒体硬件编解码，视频分析场景无需占用 AI Core 算力。

# 昇腾 310P3（DaVinci V200）片上缓存 / 片上 Buffer 说明

> ⚠️ 重要：**华为官方白皮书不对外公开 310P3 精确的单 AI Core L0/UB/L1 字节数**，下面分为【架构定义 + 行业实测参考值 + 芯片级共享 L2 Cache】，区分**AI Core 内部可编程 Buffer**和**芯片全局 L2 Cache**（很多人容易混淆）昇腾社区

> 达芬奇架构和 GPU 不一样：它不是传统 CPU 那种 L1/L2 Cache（硬件自动透明缓存），而是**软件显式管理的片上 Buffer**（Unified Buffer、L1 Buffer、L0 Buffer），算子 / ASCEND C/TIK 代码手动搬运数据，不是硬件自动 cache。

## 1️⃣ 单 AI Core 内部片上存储（8 个 AI Core / 310P3 芯片）

DaVinci V200 每个 AI Core 包含：

1. **Unified Buffer（UB，统一缓冲区）**：AI Core 最核心片上 SRAM，Cube 计算单元直接读写，**软件可分配**，存放权重 /feature map 分块数据。
2. **L1 Buffer**：MTE 数据搬运专用缓存，用于格式转换、转置、Img2Col 等。
3. **L0 Buffer（L0OUT）**：Cube 计算输出结果缓存，矩阵计算结果先落在 L0，再搬回 UB。
4. Scalar Buffer：标量计算单元的小容量本地缓存。

> 行业实测（TIK 算子开发实测，310P/310P3 同 V200）：
> 
> 
> - Unified Buffer（UB）：**1MB / 每 AI Core**
> - L1 Buffer：**512KB / 每 AI Core**
> - L0OUT Buffer：**256KB / 每 AI Core**
> 
> 
> 👉 单颗 310P3（8 个 AI Core）AI Core 片上 SRAM 合计：
> 8 × (1MB + 512KB + 256KB) = **14MB**（仅 AI Core 内部 Buffer，不含芯片全局 L2）

## 2️⃣ 芯片全局共享 L2 Cache（NPU 共享，片上）

**L2 Cache：整芯片共享，在 AI Core 外部，属于 NPU 片上最后一级缓存，对 GM（LPDDR4X 显存）做硬件缓存，对算子有一定透明加速作用**

- 昇腾 310P3 单芯片 **L2 Cache：4MB**（全 8 个 AI Core + Vector 核共享）

> 注意：L2 Cache 硬件自动管理，**不能由 TIK/ASCEND C 直接显式分配**，和 UB/L1/L0 可编程 Buffer 不是一类。

## 3️⃣ 其他片上缓存（CPU 子系统）

310P3 内置 8 核泰山 V200M CPU：

- 每个泰山 V200M：**L1I:64KB + L1D:64KB，L2:512KB / 核**
- 8 核 CPU 合计：L1 1MB，L2 4MB（CPU 私有的缓存，和 NPU 算力通路完全隔离，不参与 AI 推理张量计算）

## 4️⃣ 汇总（单颗昇腾 310P3，NPU 相关，不含 CPU 缓存）

- 8×AI Core 可编程片上 SRAM（UB+L1+L0）：**14MB**
- NPU 全局共享 L2 Cache：**4MB**

> NPU 相关片上存储总和 ≈ **18MB**
> 👉 这部分是**片上 SRAM**，远小于板载 LPDDR4X（24GB）；大模型推理时，必须不断在【片上 Buffer ↔ LPDDR4X 显存】之间做分块流水，片上缓存是瓶颈之一。

## 5️⃣ 关键提醒（工程部署重点）

1. **UB 是最关键资源**：算子切分 tiling（分块）的上限，完全受 Unified Buffer 大小约束；大模型推理，单块 tensor 不能超过单 AI Core 的 UB 容量。
2. L2 Cache 是硬件自动缓存，**不可被模型手动分配**，只能提升 GM 访存命中率，不能用来存放算子张量。
3. 虚拟化切分 NPU 时：**AI Core 内 UB/L1/L0 跟着 AI Core 切分**；L2 Cache 是芯片共享资源，虚拟化场景一般做带宽隔离，不是按容量硬切。
4. 昇腾官方没有在公开规格书直接打印 UB/L1/L0 数值，**正式交付 / 投标如果需要权威口径，需要走昇腾 FAE 通过 get_soc_spec 接口查询**，上面是 CANN 算子开发社区实测值，用于方案评估、性能测算没问题昇腾社区。
