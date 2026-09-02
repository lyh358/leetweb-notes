# 一、自定义算子优化背景：优化前的Attention执行链路

## 1. 为什么要分析优化前的计算图

Attention在算法层面的公式并不复杂：

$$
Score=\frac{QK^T}{\sqrt{d}}
$$

$$
Weight=\operatorname{Softmax}(Score)
$$

$$
Output=Weight\times V
$$

在Attention中，**Score和Weight都用于描述不同位置之间的关联，但含义不同**：

- **Score**：未经归一化的相关性分数；
- **Weight**：Score经过Softmax后得到的注意力权重。

从算法角度看，它主要包含两次矩阵乘和一次Softmax。但模型部署到Ascend 310P3以后，这些操作不会自动变成一个整体，而是可能被编译成多个独立Kernel执行。

因此，端侧性能不仅取决于“进行了多少次计算”，还取决于：

- 生成了多少个Kernel；
- Kernel之间如何传递数据；
- 中间结果是否反复访问GM；
- 是否需要进行数据格式转换；
- Cube和Vector是否能够连续协同。

## =={pink}2. 先理解几个基本概念==

#### =={green}Host和Device是什么==

在昇腾推理环境中，可以把=={yellow}系统分为Host和Device两侧。==

| 部分 | 含义 | 主要职责 |
| --- | --- | --- |
| **=={green}Host==** | 运行Linux的=={yellow}**CPU侧**== | 加载OM、准备输入、申请内存、发起推理并获取结果 |
| **=={green}Device==** | Ascend 310P3 =={yellow}**NPU侧**== | 执行模型中的矩阵乘、Softmax和数据搬运等操作 |

整体关系可以简化为：

> =={green}Host CPU==
> **准备输入、加载模型、发起推理**
>           ↓
> =={green}Device NPU==
> **执行各个Kernel**
>           ↓
> =={green}Host CPU==
> **接收并处理推理结果**

这里分析的九个Kernel，主要是Device侧执行Attention子图时产生的Kernel，不是Host上运行的九个普通函数。

#### Kernel是什么

Kernel是运行在NPU上的一段设备程序。一个Kernel通常完成一种操作或一段已经融合的操作，例如：

- MatMul Kernel完成矩阵乘；
- Softmax Kernel完成归一化；
- TransData Kernel完成数据格式转换。

每启动一次Kernel，都会产生一定的调度、同步和流水线启动开销。因此，对于计算量较小的模型，Kernel过多可能使固定开销占据较大比例。

#### GM是什么

GM是Device侧的全局内存，可以理解成NPU的“大容量外部工作区”。

它容量较大，但访问速度和能耗通常不如L1、L0、UB等片上存储。AI Core执行计算时，通常需要先把数据从GM搬到片上，计算结束后再把结果写回GM。

```sql
GM
↓ 搬入
L1、L0或UB等片上存储
↓
Cube或Vector计算
↓ 写回
GM
```

如果每个Kernel都把中间结果写回GM，下一个Kernel再重新读取，就会产生额外的数据搬运。

#### Cube和Vector是什么

Ascend AI Core内部有不同类型的计算单元：

| 单元 | 擅长的操作 | Attention中的任务 |
| --- | --- | --- |
| **Cube** | 矩阵乘和卷积 | `QKᵀ`、`Weight×V` |
| **Vector** | 逐元素、规约和非线性计算 | Scale、Softmax |
| **MTE** | 数据搬运 | GM与片上存储之间传输数据 |

Attention需要先使用Cube计算矩阵乘，再使用Vector完成Softmax，最后重新回到Cube执行第二次矩阵乘。

#### =={pink}ND和NZ是什么==

=={green}ND和NZ是两种**不同的数据存储格式**。==

- **=={yellow}ND格式==**：**普通**的连续**张量格式**，便于Softmax等=={yellow}**Vector操作**==处理；
- **=={yellow}NZ格式==**：按照**固定小块重新排列**的**分形格式**，更=={yellow}**适合Cube**==高效执行矩阵乘。

可以把它们简单理解为：

```sql
ND：按照普通顺序摆放数据
NZ：按照Cube喜欢的矩阵小块方式摆放数据
```

两种格式表达的=={green}**数据内容相同**==，但=={green}**排列方式不同**==。=={green}格式转换通常由**TransData Kernel**完成==：

```undefined
ND → TransData → NZ
NZ → TransData → ND
```

格式转换基本不产生新的业务信息，却需要读取、重新排列并写入整块数据，因此会带来额外的搬运和调度开销。

## =={pink}3. 优化前的Attention计算链：九个Kernel==

ATC已经能够把第一次MatMul和Scale融合，但Softmax及其前后的格式转换仍然保持独立。

完整执行链路如下：

> Q：ND ─→ =={yellow}**Kernel 1：TransData**==，=={yellow}**ND→NZ**== ─┐
> K：ND ─→ =={yellow}**Kernel 2**：TransData==，ND→NZ ─┼→ =={pink}**Kernel 4：QKᵀ+Scale**==
> V：ND ─→ =={yellow}**Kernel 3：**TransData==，ND→NZ ─┘           ↓
>                                                                                                      Score_NZ
>                                                                                                              ↓
>                                                                                            =={yellow}**Kernel 5：TransData，NZ→ND**==
>                                                                                                              ↓
>                                                                                            =={pink}**Kernel 6：Softmax**==
>                                                                                                              ↓
>                                                                                            =={yellow}**Kernel 7：TransData，ND→NZ**==
>                                                                                                              ↓
>                                                                                            =={pink}**Kernel 8：Weight×V**==
>                                                                                                              ↓
>                                                                                            =={yellow}**Kernel 9：TransData，NZ→ND**==
>                                                                                                              ↓
>                                                                                                       最终输出

九个Kernel分别负责：

| Kernel | 执行单元 | 作用 |
| --- | --- | --- |
| 1 | MTE/格式转换 | 将Q从普通ND格式转换成Cube需要的NZ格式 |
| 2 | MTE/格式转换 | 将K从ND格式转换成NZ格式 |
| 3 | MTE/格式转换 | 将V从ND格式转换成NZ格式 |
| 4 | Cube+Vector | 计算`QKᵀ`，并完成缩放`1/√d` |
| 5 | MTE/格式转换 | 将Score从NZ转回ND，供Softmax使用 |
| 6 | Vector | 对Score执行Softmax，得到注意力权重 |
| 7 | MTE/格式转换 | 将注意力权重从ND转成NZ，供第二次矩阵乘使用 |
| 8 | Cube | 计算`Weight×V`，得到Attention输出 |
| 9 | MTE/格式转换 | 将输出从NZ恢复成业务侧需要的ND格式 |

从九个Kernel可以看出，=={green}真正完成Attention数学计算的主要是==：

```undefined
Kernel 4：QKᵀ + Scale
Kernel 6：Softmax
Kernel 8：Weight × V
```

=={green}其余大部分Kernel都在**处理数据格式和数据流转**。==

## =={pink}4. 优化前为什么性能不理想==

#### =={yellow}问题一：Kernel数量较多==

整个Attention子图=={green}包含九个Kernel。即使每个Kernel的计算量不大==，也需要分别经历：

```undefined
Kernel调度
→ 流水线启动
→ 数据读取
→ 执行计算
→ 同步结束
```

SPF属于小Shape、batch_size为1的轻量模型，矩阵乘本身执行得很快，因此=={green}**Kernel启动和调度开销**==会显得更加突出。

#### =={pink}问题二：中间结果反复访问GM==

=={yellow}Kernel 4完成==`QKᵀ+Scale`=={yellow}后==，Score不能直接交给Softmax使用，而是需要=={yellow}写回GM==。

```sql
Cube计算Score
→ Score写回GM
→ Softmax Kernel重新从GM读取
```

=={yellow}kernel 6 Softmax完成后==，注意力权重又要=={yellow}写回GM==，=={yellow}再==由第二次MatMul=={yellow}重新读取==：

```sql
Vector完成Softmax
→ Weight写回GM
→ Cube重新从GM读取
```

=={yellow}这使中间结果在不同Kernel之间反复搬运，而不是一直保留在高速片上存储中。==

#### =={pink}问题三：Cube和Vector之间频繁转换格式==

=={yellow}矩阵乘更适合NZ格式，Softmax更适合ND格式==，因此链路中形成：

```sql
Cube使用NZ
    ↓
TransData：NZ→ND
    ↓
Vector执行Softmax
    ↓
TransData：ND→NZ
    ↓
Cube执行第二次MatMul
```

这些TransData没有改变Attention的计算结果，但增加了数据读取、重新排列、写入和Kernel调度。

#### 问题四：计算与搬运没有形成连续流水

优化前的多个Kernel彼此独立。前一个Kernel结束后，中间结果通常需要写回GM，下一个Kernel才能继续处理。

因此执行过程更接近：

```undefined
搬运 → 计算 → 写回
搬运 → 计算 → 写回
搬运 → 计算 → 写回
```

而不是：

```undefined
数据搬入一次
→ 片上连续完成多个操作
→ 最终结果写回一次
```

对于小矩阵Attention，真正的Cube计算时间很短，无法掩盖这些搬运和调度开销。

## =={pink}5. 最终瓶颈判断==

优化前Attention的主要矛盾不是计算公式太复杂，也不是Cube执行矩阵乘不够快，而是：

> =={yellow}**多个Kernel**将Attention链路切得**过碎**，使**中间结果反复写回GM**，并在Cube和Vector之间产生**多次格式转换**与**kernel调度**。==

因此，=={yellow}**自定义算子的优化目标**==确定为：

> 把QKᵀ、Scale、Softmax和Weight×V
> =={yellow}**融合到一个Kernel**中==
>         ↓
> =={yellow}**矩阵乘**交给**Cube**==
> =={yellow}**Softmax**交给**Vector**==
>         ↓
> =={yellow}**中间**Score和Weight尽量**保留在片上**==
>         ↓
> =={yellow}**减少Kernel启动**、**GM往返**和**TransData**==

一句话总结：

> 优化前的Attention虽然只有三步核心数学计算，但部署到NPU后被拆成了九个Kernel，真正限制性能的不是矩阵乘算力，而是Kernel调度、中间张量GM读写和ND/NZ格式转换，因此我进一步设计了Cube与Vector协同的FusedAttention自定义算子。

### 面试口语化回答

> 优化前，我先分析了Attention在310P3上的执行图。算法上它只有QK转置、Softmax和Weight乘V几个主要步骤，但ATC编译后按照完整统计口径形成了九个Kernel。其中三个负责Q、K、V从普通ND格式转成适合Cube矩阵计算的NZ格式，第四个完成QK转置和Scale，第五个把Score转回ND，第六个执行Softmax，第七个再把结果转成NZ，第八个执行第二次矩阵乘，最后一个恢复ND输出。
> 
> 
> 这里Host是运行Linux的CPU侧，负责加载模型、准备数据和发起推理；Device是310P3 NPU侧，真正执行这些Kernel。ND是普通张量格式，NZ是适合Cube分块矩阵计算的格式，两者转换需要TransData。优化前最大的问题是Kernel数量多，而且Cube和Vector之间切换时，中间结果要反复写回Device侧的GM，再由下一个Kernel读取，同时还要进行ND和NZ转换。对于小Shape模型，矩阵乘本身很快，这些调度和搬运开销反而更加明显。所以我才考虑把整条Attention链融合成一个Cube和Vector协同的自定义Kernel，让中间结果尽量保留在片上。

---

# =={pink}二、FusedAttention的实施方法概设==

针对优化前存在的**=={yellow}九个Kernel、中间结果反复写回GM、ND/NZ格式转换以及搬运计算串行==**=={yellow}问题==，我主要采用了以下=={green}**六项方法**==。

| 方法 | 怎么实现 | 解决的问题 |
| --- | --- | --- |
| **=={yellow}1.单Kernel融合==** | 将`QKᵀ、Scale、Softmax、Weight×V`合并到一个FusedAttention Kernel | 减少Kernel启动、调度和同步 |
| **=={yellow}2.Cube与Vector协同==** | 两次矩阵乘使用Cube，Scale和Softmax使用Vector | 让不同计算交给最合适的硬件单元 |
| **=={yellow}3.中间结果片上驻留==** | Score和Weight在**L0C、UB、L1之间传递**，只读取Q/K/V并写回最终Output | 避免中间张量反复访问GM |
| **=={yellow}4.多核Head并行==** | **根据Head划分AI Core**，每个Core独立处理一个Head | 避免多个Head串行执行 |
| **=={yellow}5.Tiling与双缓冲==** | **将K维切成两个Chunk**，使用**Ping-Pong Buffer交替搬运和计算** | 重叠MTE搬运与Cube计算 |
| **=={yellow}6.格式转换内联==** | 输入格式与Cube计算格式匹配，**在搬运和Kernel内部完成转置**及输出整理 | 减少独立TransData和Transpose Kernel |

## 1.实施链路

```markdown
第一步：识别原始Attention子图
        ↓
第二步：替换为FusedAttention自定义节点
        ↓
第三步：Host侧完成算子注册、Shape推导和Tiling
        ↓
第四步：Device侧实现Cube、Vector和MTE协同Kernel
        ↓
第五步：通过多核、片上复用和Ping-Pong优化数据流
        ↓
第六步：重新编译OM并进行精度与性能验证
```

## 2.最核心的设计变化

```diff
优化前：
9个独立Kernel
+ Score和Weight反复写回GM
+ Cube/Vector之间多次TransData

优化后设计：
1个FusedAttention Kernel
+ Score和Weight保留在片上
+ Cube、Vector和MTE协同执行3.一句话总结
```

> 我把原来分散执行的Attention链融合成一个Cube和Vector协同的自定义Kernel，通过片上数据驻留减少GM往返，按Head进行多核并行，再利用Tiling、Ping-Pong双缓冲和格式转换内联进一步降低搬运、调度及转换开销。

## 面试口语化回答

> 我的改造可以概括成六点：第一，把QK转置、Scale、Softmax和Weight乘V从九个Kernel融合成一个Kernel；第二，两次矩阵乘使用Cube，Softmax使用Vector；第三，让Score和Weight一直保留在片上，不再反复写回GM；第四，按照Head分配AI Core进行多核并行；第五，把K维分块并使用Ping-Pong双缓冲，使数据搬运与Cube计算重叠；第六，把必要的转置和格式整理融合进搬运及Kernel内部，减少独立TransData。工程上再通过Host侧注册和Tiling、Device侧Kernel以及ONNX子图替换，把这套设计接入模型编译链路。

---

可以。先纠正两个容易说错的名称：

- “双核协同”更准确叫 **Cube与Vector异构协同**，它们是AI Core内部的不同执行流水线，不是两个CPU核心。
- “多核hybrid并行”在本项目中更准确叫 **按Head进行多AI Core并行**。

后面的详设统一按照：**概念是什么 → 为什么使用 → 项目中如何实现 → 正确性如何保证 → 面试怎么回答**展开。

---

# =={pink}三、FusedAttention关键优化方法详设==

## =={pink}1. 单Kernel融合详设==

### =={yellow}1.1 什么是单Kernel融合==

优化前，Attention被拆成多个独立Kernel：

```undefined
TransData
→ QKᵀ+Scale
→ TransData
→ Softmax
→ TransData
→ Weight×V
→ TransData
```

每个Kernel只能完成其中一段，Kernel之间通过GM传递中间结果。

单Kernel融合就是把：

```diff
QKᵀ
+ Scale
+ Softmax
+ Weight×V
+ 必要的格式整理
```

放进同一个Device Kernel中连续执行。

这里不仅是把代码写进同一个函数，更=={yellow}关键的是==：

> =={yellow}**多个计算阶段**进入**同一个执行上下文**，可以**共享L1、L0和UB中的片上数据**。==

### =={pink}1.2 为什么要融合==

融合主要解决三个问题：

1. =={yellow}**减少Kernel**启动和调度；==
2. =={yellow}**避免**Score、Weight**反复写回GM**；==
3. =={yellow}**跨越ATC无法自动融合**的Softmax**边界**。==

ATC可以融合部分线性计算，例如：

```undefined
MatMul + Scale
```

但Softmax包含Max、Exp、Sum和Div等非线性运算，通常会保持为独立Kernel。

因此，需要通过自定义算子主动扩大Kernel边界。

### =={pink}1.3 在项目中怎么实现==

#### =={yellow}1.3.1 FusedAttention自定义算子工程目录==

```objectivec
FusedAttentionCube_build/
│
├── CMakeLists.txt：负责整个自定义算子工程的编译：告诉编译系统整个工程由哪些源码组成，分别需要编译成什么产物。
├── build.sh：负责执行完整构建流程：1.创建build目录  2.执行CMake  3.编译Host和Device代码  4.生成算子安装包  5.安装算子
│
├── cmake/ 			cmake目录负责管理CANN依赖、芯片配置以及Host、Kernel和插件三部分的构建规则。
│   ├── config.cmake：配置CANN安装路径、目标芯片、编译模式、自定义算子的vendor名称、输出和安装目录。
│   ├── func.cmake：封装重复使用的CMake函数
│   └── intf.cmake：处理不同模块之间的编译接口
│	
│	Host侧代码运行在CPU侧，负责告诉CANN：
│	这个算子是什么；
│	接收什么输入；
│	产生什么输出；
│	支持什么数据类型和格式；
│	输入应该怎样分核、分块。
├── op_host/       Host侧：算子定义与Tiling配置
│   ├── fused_attention_cube_op.cpp   Host侧的主要算子定义文件：1.定义算子接口；2.注册Shape推导函数；
│   │							     3.注册Tiling函数；4.绑定Device Kernel
│   └── fused_attention_cube_tiling.h	Host侧和Device侧共享的Tiling参数定义文件。
│	Host侧负责填写，Device Kernel负责读取。
│	可以理解为：Host提前写好一张“施工图”，Device Kernel根据施工图执行分核、分块和内存访问。
│
├── op_kernel/	   Device侧：计算实现
│   └── fused_attention_cube_pingpong.cpp
│
├── framework/	   
│   └── onnx_plugin/	ONNX解析插件
│       └── fused_attention_cube_plugin.cpp		负责连接ONNX与CANN自定义算子，相当于自定义节点与ATC之间的“翻译器”。
│		编译后通常生成类似：libcust_onnx_parsers.so
├── tools/	ONNX计算图处理工具
│   └── fused_attention_replace.py	该脚本负责在原始ONNX模型中识别原Attention子图并替换为自定义算子
│
└── test/	验证工具
    ├── generate_golden.py	生成自定义算子的Golden基准数据。
    ├── compare_output.py	比较自定义算子输出与Golden输出，计算：余弦相似度、MSE、最大绝对误差、平均绝对误差
    └── run_test.sh		组织完整验证流程
```

##### 编译后产物

整个工程编译后，主要形成：

| 产物 | 作用 |
| --- | --- |
| **=={yellow}Host算子定义库==** | **让CANN识别接口、Shape和Tiling** |
| **=={yellow}Device Kernel二进制==** | **在AI Core上执行实际计算** |
| =={yellow}**ONNX解析插件**== | **让ATC识别ONNX自定义节点** |
| =={yellow}**自定义OPP安装目录**== | **保存算子配置、动态库和Kernel** |
| =={yellow}替换后的ONNX== | 包含FusedAttentionCube节点 |
| =={yellow}编译后的OM== | 最终由ACL加载执行的模型 |

整体链路为：

```text
原始ONNX
    ↓ fused_attention_replace.py
包含自定义节点的ONNX
    ↓ ONNX解析插件
CANN自定义算子
    ↓ Host Shape推导和Tiling
生成设备执行参数
    ↓ Device Kernel
编译进OM
    ↓ ACL
Ascend 310P3执行
```

##### 面试时的目录介绍

> 整个FusedAttention工程主要分为五部分。第一部分是`op_host`，其中算子定义文件负责声明Q、K、V和Output接口、Shape、数据类型、格式以及Tiling入口，Tiling头文件负责定义Host传给Device的分核、分块参数。第二部分是`op_kernel`，包含单缓冲和Ping-Pong两个Device Kernel版本，真正完成Cube矩阵乘、Vector Softmax和MTE搬运。第三部分是ONNX插件，它将ONNX里的FusedAttentionCube节点映射成CANN自定义算子。第四部分是子图替换脚本，负责把原始MatMul、Scale、Softmax、MatMul链替换成一个自定义节点。最后是CMake、构建脚本和测试工具，负责算子编译、安装、Golden精度对比和板端性能验证。

##### 最简记忆

```text
op_host
→ 定义“怎么调用、怎么切”

op_kernel
→ 实现“具体怎么算”

onnx_plugin
→ 解决“ATC怎么认识它”

replace.py
→ 解决“模型怎么用上它”

CMake/build
→ 解决“工程怎么编译安装”

test
→ 解决“算得对不对、跑得快不快”
```

---

=={yellow}工程上分成**四部分**==。

#### =={pink}1.3.2 第一步：定义自定义算子接口==

自定义算子接口描述的是：

> =={yellow}这个**算子叫什么、接收什么输入、产生什么输出，以及支持什么Shape、数据类型和数据格式。**==

当前FusedAttention原型接口为：

```scss
FusedAttention(Q,K,V) → Output
```

=={yellow}当前设计配置为：==

| 张量 | Shape | 数据类型 | 数据格式 |
| --- | --- | --- | --- |
| Q | `[1,8,64,64]` | FP16 | **=={yellow}FRACTAL_NZ==** |
| K | `[1,8,64,64]` | FP16 | **=={yellow}FRACTAL_NZ==** |
| V | `[1,8,64,64]` | FP16 | **=={yellow}FRACTAL_NZ==** |
| Output | `[1,8,64,64]` | FP16 | =={yellow}**ND**== |

=={yellow}每个维度分别表示==：

```text
[batch, headNum, seqLen, headDim]
```

数据格式表示张量元素在内存中怎样排列。

当前主要涉及：

| 格式 | 含义 | 使用位置 |
| --- | --- | --- |
| ND | 普通连续张量格式 | 业务输入输出、Vector操作 |
| FRACTAL_NZ | 按矩阵小块排列的分形格式 | Cube矩阵计算 |
| ZZ、zN等 | Cube内部不同矩阵角色对应的片上排列 | L0和UB内部处理 |

同一个FP16矩阵既可以使用ND格式，也可以使用NZ格式：

```text
数据类型相同：都是FP16
数据内容相同：数值没有改变
数据格式不同：元素在内存中的排列顺序不同
```

因此：

- FP16/FP32回答“一个元素是什么”；
- ND/NZ回答“这些元素怎样排列”。

=={yellow}同时声明：==

- =={yellow}**Host侧Tiling入口；**==
- =={yellow}**Device侧Kernel入口。**==

#### =={pink}1.3.3 第二步：识别原始Attention子图==

在ONNX中识别：

```undefined
MatMul
→ Mul/Scale
→ Softmax
→ MatMul
```

确认这段子图没有其他外部分支依赖后，=={yellow}**将其整体替换**==为：

```undefined
FusedAttention
```

如果Score还被其他节点使用，就不能直接删除原子图，否则会改变模型语义。

#### =={pink}1.3.3 第三步：Host侧完成注册和Tiling==

=={yellow}Host侧负责：==

- =={yellow}**注册算子**名称；==
- =={yellow}**检查输入Shape**和**数据类型**；==
- =={yellow}**推导输出Shape**；==
- =={yellow}根据Head数量和矩阵规模**生成Tiling参数**；==
- 确定BlockDim和各Core的数据范围；
- =={yellow}**将Tiling数据传递**给**Device Kernel**。==

#### =={pink}1.3.4 第四步：Device侧连续完成三阶段计算==

> =={yellow}Phase 1：**Cube**计算**QKᵀ**==
> =={yellow}Phase 2：**Vector**计算**Scale**和**Softmax**==
> =={yellow}Phase 3：**Cube**计算**Weight×V**==

=={yellow}中间结果通过片上存储传递，不在Phase之间退出Kernel==。

---

### 1.4 融合后的执行结构

```sql
CopyIn Q/K/V
      ↓
Cube：QKᵀ
      ↓
Vector：Scale + Softmax
      ↓
Cube：Weight×V
      ↓
格式整理
      ↓
CopyOut Output
```

---

### 1.5 怎样保证融合前后结果一致

需要保证：

- Q、K、V输入顺序不变；
- K转置语义正确；
- Scale系数不变；
- Softmax仍按原来的维度执行；
- Weight与V的矩阵乘维度不变；
- 多Head输出顺序不变；
- 输出Shape和数据格式与原模型一致。

验证时使用原始ONNX子图生成Golden输出，再比较：

- 余弦相似度；
- MSE；
- 最大绝对误差；
- 极端输入下的Softmax结果。

---

### 1.6 面试回答

> 单Kernel融合不是简单地把几段代码放在一起，而是重新划分Attention的设备执行边界。我在ONNX中识别QK转置、Scale、Softmax和Weight乘V子图，将其替换成FusedAttention节点，再通过Host侧注册与Tiling、Device侧Kernel实现完整计算。这样多个阶段可以共享片上数据，减少Kernel启动，并避免Score和Weight在阶段之间反复写回GM。

---

## =={pink}2. Cube与Vector异构协同详设：流水设计与内存搬运==

### 2.1 什么是Cube与Vector

Ascend AI Core内部包含不同的执行单元：

| 单元 | 擅长的操作 |
| --- | --- |
| Cube | 矩阵乘和卷积 |
| Vector | 逐元素、规约和非线性计算 |
| MTE | 不同存储层级之间的数据搬运 |
| Scalar | 地址、循环和控制逻辑 |

Attention同时包含矩阵乘和非线性计算，因此不能只使用一种执行单元。

---

### 2.2 为什么这样分配

本算子的分配方式是：

| 操作 | 执行单元 |
| --- | --- |
| `QKᵀ` | Cube |
| Scale | Vector |
| RowMax | Vector |
| Sub和Exp | Vector |
| RowSum和Div | Vector |
| `Weight×V` | Cube |
| 数据搬运 | MTE |

原因是：

- Cube执行矩阵乘的效率远高于Vector模拟矩阵乘；
- Cube不能直接完成Exp、Max等Softmax操作；
- Vector适合逐元素和行规约；
- MTE可以在计算单元工作时独立搬运数据。

---

### 

### =={pink}2.3 在算子中如何协同==

=={yellow}算子有**三个计算阶段**和**五条流水线**。==

#### =={pink}2.3.1 流水线：MTE2、MTE1、Cube、Vector和MTE3==

| 流水线 | 主要职责 | 在本算子中的作用 |
| --- | --- | --- |
| **=={yellow}PIPE_MTE2==** | =={yellow}**GM到片上存储**==的数据搬运 | 将Q、K、V从GM搬到L1或UB |
| **=={yellow}PIPE_MTE1==** | =={yellow}**片内（L0/L1/UB）的数据搬运**==和格式整理 | 将Q、K、Weight、V送入L0A/L0B |
| **=={yellow}PIPE_M==** | Cube矩阵计算 | 执行`QKᵀ`和`Weight×V` |
| **=={yellow}PIPE_V==** | Vector向量计算 | 执行Scale、Softmax和输出格式整理 |
| **=={yellow}PIPE_MTE3==** | =={yellow}**片上结果向外搬运**== | =={yellow}**UB到L1或GM的数据搬运**== |

Scalar负责地址计算、循环和控制，也可以看成辅助控制流水线，但通常说“五条主要数据与计算流水线”即可。

整体关系为：

```undefined
GM
 │
 │ PIPE_MTE2
 ▼
L1
 │
 │ PIPE_MTE1
 ▼
L0A / L0B
 │
 │ PIPE_M
 ▼
L0C
 │
 │ PIPE_V
 ▼
UB
 │
 │ PIPE_MTE3
 ▼
L1或GM
```

需要注意：不同阶段会复用这些流水线，并不是数据永远只沿一个方向走一次。

---

#### 2.3.2 计算阶段：QKᵀ、Softmax、Weight×V

##### Phase 1：计算Score

目标是：

$$
Score=QK^T
$$

执行链路为：

```css
Q、K位于GM
      ↓ PIPE_MTE2
Q、K进入L1
      ↓ PIPE_MTE1
Q进入L0A，K进入L0B
      ↓ PIPE_M
Cube执行QKᵀ
      ↓
Score保存在L0C
      ↓ PIPE_V
Score转换并进入UB
```

流水线依赖关系是：

```undefined
MTE2 → MTE1 → M → V
```

每一级都是前一级的消费者：

- MTE1必须等待MTE2完成；
- Cube必须等待MTE1完成；
- Vector必须等待Cube产生Score。

---

##### Phase 2：计算Softmax

目标是：

$$
Weight=\operatorname{Softmax} \left( \frac{Score}{\sqrt d} \right)
$$

Score进入UB后，由Vector连续执行：

```vbnet
Scale
→ RowMax
→ Sub
→ Exp
→ RowSum
→ Reciprocal
→ Multiply
```

这部分主要使用：

```undefined
PIPE_V
```

Softmax完成后，Weight仍保留在UB中。

为了让第二次矩阵乘使用Weight，需要将它送给Cube：

```markdown
Weight位于UB
      ↓ PIPE_MTE3
Weight进入L1
      ↓ PIPE_MTE1
Weight进入L0A
```

这里的依赖关系为：

```undefined
V → MTE3 → MTE1
```

---

##### Phase 3：计算最终输出

目标是：

$$
Output=Weight\times V
$$

Weight和V来自两条数据路径：

```undefined
Weight：UB → L1 → L0A
V：     GM → L1 → L0B
```

两条路径分别使用：

```undefined
Weight：
PIPE_MTE3 → PIPE_MTE1

V：
PIPE_MTE2 → PIPE_MTE1
```

两侧数据都准备完成后，Cube才能执行：

```undefined
PIPE_M：Weight × V
```

计算结束后：

```markdown
Output位于L0C
      ↓ PIPE_V
完成内部格式到ND的整理
      ↓ PIPE_MTE3
最终Output写回GM
```

完整依赖为：

```markdown
MTE3和MTE2
      ↓
    MTE1
      ↓
      M
      ↓
      V
      ↓
    MTE3
```

---

### 2.5 面试回答

> Cube与Vector协同指的是根据计算特点分配AI Core内部流水线。两次矩阵乘使用Cube，Scale和Softmax使用Vector，数据搬运交给MTE。它们在同一个Kernel中通过L0、UB和L1传递中间数据，并使用Event保证生产者和消费者之间的同步。这样既发挥Cube的矩阵吞吐，也保留Vector处理非线性的能力。

---

## =={pink}3. 中间结果片上驻留详设==

### 3.1 什么是片上驻留

片上驻留是指Score和Weight计算完成后，不立即写回GM，而是保存在靠近计算单元的片上存储中，直接供下一阶段使用。

=={yellow}**优化前**：==

> =={green}**Score → GM → Softmax**==
> =={green}**Weight → GM → 第二次MatMul**==
> {green}**V**== **=={green}→GM==** **=={green}→ L1==** **=={green}→ L0B==**==

=={yellow}**优化后**设计：==

> =={green}**Score → L0C** **→** **UB → Softmax**==
> =={green}**Weight → UB** **→** **L1** **→** **L0A → 第二次MatMul
> V**== **=={green}→GM==** **=={green}→ L1==** **=={green}→ L0B==**

### 3.2 使用哪些存储

| 存储 | 本算子中的作用 |
| --- | --- |
| GM | 保存Q、K、V和最终Output |
| L1 | 缓存矩阵块，为Cube准备数据 |
| L0A | 保存Cube的矩阵A |
| L0B | 保存Cube的矩阵B |
| L0C/LOC | 保存矩阵乘累加结果 |
| UB | 保存Score、Weight、Softmax临时量和输出 |

基本原则是：

> =={yellow}GM负责**输入和最终输出**，片上存储负责计算**过程中的中间数据**。==

### 3.3 Score怎样驻留

第一阶段Cube完成：

$$
Score=QK^T
$$

结果首先保存在L0C中。随后将Score转换并搬入UB，Vector直接在UB中执行：

```vbnet
Scale
→ RowMax
→ Sub
→ Exp
→ RowSum
→ Reciprocal
→ Multiply
```

Softmax完成后，Weight仍然保存在UB中。

### 3.4 Weight怎样交给第二次MatMul

Cube不能直接从UB执行矩阵乘，因此需要：

```undefined
Weight：UB → L1 → L0A
V：GM → L1 → L0B
```

随后Cube执行：

$$
Output=Weight\times V
$$

虽然Weight仍然需要在片上层级之间搬运，但不必写回GM。

### =={pink}3.5 为什么片上存储能够放下==

=={yellow}当前原型矩阵为64×64。==

=={yellow}FP16矩阵大小为：==

$$
64\times64\times2=8192\ \text{Bytes}
$$

=={yellow}即一个矩阵约8 KB。==

因此Q、K、V、Score和Output可以通过分阶段使用与Buffer复用控制片上占用，而不需要同时为所有数据保留独立空间。

方案中的总体片上占用约为：

```undefined
L1：约16 KB
UB：约32 KB
L0：约32 KB
合计：约80 KB
```

### 3.6 面试回答

> 片上驻留就是不把Score和Weight作为中间结果写回GM。第一次数学乘结果先保存在L0C，再进入UB完成Softmax；Softmax结果继续从UB经过L1进入L0A，直接参加第二次矩阵乘。这样GM只需要读取Q、K、V并写回最终Output，减少了中间结果往返。

---

## =={pink}4. 多核Head并行详设==

#### =={pink}4.1 什么是 Head==

=={yellow}**Multi-Head Attention**会把特征拆成**多个相互独立的注意力分支**，每个分支称为一个== **=={yellow}Head==**=={yellow}。==

当前自定义算子原型的输入可以表示为：

```css
Q、K、V：[Batch, Head, SeqLen, HeadDim]
         [  1,    8,      64,      64  ]
```

其中：

- `Batch=1`：一次处理一组输入；
- `Head=8`：=={yellow}**共有8个注意力头；**==
- `SeqLen=64`：=={yellow}**序列包含64个Token**==；
- `HeadDim=64`：=={yellow}**每个Token在一个Head中有64维特征。**==

每个 Head都独立执行完整的 Attention：

```css
Head 0：Q₀K₀ᵀ → Scale → Softmax → Weight₀V₀
Head 1：Q₁K₁ᵀ → Scale → Softmax → Weight₁V₁
...
Head 7：Q₇K₇ᵀ → Scale → Softmax → Weight₇V₇
```

#### =={pink}4.2 什么是多核 Head 并行==

多核 Head并行就是=={yellow}**把不同 Head分配给不同的 AI Core，让多个 Head同时计算。**==

本算子采用：

```ini
BlockDim = 8
```

也就是启动8个 AI Core，每个核负责一个 Head：

```css
                      FusedAttention
                            │
    ┌──────────┬────────────┼──────────┬─────────┐
    ▼          ▼            ▼          ▼         ▼
AI Core 0    AI Core 1    AI Core 2    ……    AI Core 7
  Head 0       Head 1       Head 2             Head 7
    │            │            │                  │
  Q₀K₀ᵀ         Q₁K₁ᵀ        Q₂K₂ᵀ              Q₇K₇ᵀ
    ↓            ↓            ↓                  ↓
 Softmax      Softmax      Softmax             Softmax
    ↓            ↓            ↓                  ↓
Weight₀V₀    Weight₁V₁    Weight₂V₂           Weight₇V₇
    │            │            │                  │
    └────────────┴────────────┴──────────────────┘
                              ↓
                     按Head位置写入Output
```

=={yellow}它属于==**=={yellow}核间并行==**=={yellow}：==

> =={yellow}**8个 AI Core同时处理8个不同 Head**==，而不是让8个核共同计算同一个 Head。

---

#### =={pink}4.3 为什么可以按 Head并行==

=={yellow}不同 Head之间**没有直接的数据依赖**。==

例如 Head 0只使用：

```css
Q[0,0,:,:]
K[0,0,:,:]
V[0,0,:,:]
```

Head 1只使用：

```css
Q[0,1,:,:]
K[0,1,:,:]
V[0,1,:,:]
```

它们：

- 读取不同的输入区域；
- 生成各自独立的 Score和 Weight；
- 写入不同的输出区域；
- 不需要交换中间结果。

因此可以自然地分配到不同 AI Core：

```undefined
Head 0的结果不依赖Head 1
Head 1的结果也不依赖Head 0
```

这是一种非常适合并行的数据划分方式。

---

#### =={pink}4.4 为什么要这么做==

##### =={yellow}原因一：避免单核串行计算8个Head==

如果只使用一个 AI Core，就只能依次执行：

```undefined
Head 0 → Head 1 → Head 2 → …… → Head 7
```

总时间近似为：

$$
T_{\text{single-core}} \approx 8T_{\text{head}}
$$

使用8个 AI Core后，理想情况下8个 Head可以同时运行：

$$
T_{\text{multi-core}} \approx T_{\text{head}}+T_{\text{parallel-overhead}}
$$

实际不会严格达到8倍加速，因为还会受到数据搬运、调度和带宽竞争等因素影响。

---

##### =={yellow}原因二：Head是天然的独立任务==

并行设计最怕多个任务之间频繁交换数据或等待。

而不同 Head：

```undefined
输入独立 → 中间结果独立 → 输出区域独立
```

因此：

- 划分逻辑简单；
- 几乎不需要核间同步；
- 不需要跨核归约；
- 不容易产生写冲突。

相比于把一个 `64×64` 的小矩阵强行拆给多个核，按 Head分核更加自然。

---

##### =={yellow}原因三：提高多核利用率==

如果8个 Head全部放在一个核上计算，其他 AI Core可能处于空闲状态。按 Head分配后：

```undefined
一个Head → 一个AI Core
```

可以让多个核同时工作，缩短整个 Attention算子的端到端执行时间。

---

#### =={pink}4.5 本算子中是如何实现的==

##### =={yellow}第一步：Host侧设置并行核数==

Host侧根据 Head数量设置，同时310P3也正好有8个AI core：

```ini
headNum = 8
blockDim = 8
```

启动 Kernel时，告诉运行时使用8个逻辑 Block，也就是让8个 AI Core参与执行。

---

##### =={yellow}第二步：每个核获得自己的编号==

---

##### =={yellow}第三步：根据Head编号计算地址偏移==

然后每个核读取自己的 Q、K、V：

```ini
qHead = qBase + headOffset;
kHead = kBase + headOffset;
vHead = vBase + headOffset;
```

并写入自己的输出区域：

```ini
outputHead = outputBase + headOffset;
```

---

##### =={yellow}第四步：每个核独立完成一个Head==

每个 AI Core进入相同的 Kernel代码，只是输入输出偏移不同。

所有核执行的是同一套程序，只是处理的数据区域不同。这种方式称为 **SPMD**：

> Single Program, Multiple Data，即多个核运行同一份程序，处理不同的数据。

---

#### =={pink}6. 多核之间需要同步吗==

在本方案中，不同 Head：

- 不读取对方的中间结果；
- 不写入相同的输出地址；
- 不需要共同累加同一个结果。

所以在 Kernel内部，通常**=={yellow}不需要做 Head之间的核间同步==**：

```python-repl
Core 0完成Head 0 → 写自己的输出
Core 1完成Head 1 → 写自己的输出
...
Core 7完成Head 7 → 写自己的输出
```

Kernel整体执行完成后，运行时保证所有 Block结束，Host或后续算子才能使用完整输出。

这里需要区分两种同步：

```sql
核内同步：
MTE、Cube、Vector之间通过事件保证流水依赖

核间同步：
不同Head之间基本不需要，因为没有数据依赖
```

---

#### =={pink}7. 多核Head并行和核内流水如何组合==

=={yellow}两种优化可以叠加：==

```markdown
核间：8个AI Core并行处理8个Head
                 ↓
核内：每个AI Core内部再使用
      Tiling + Ping-Pong双缓冲
                 ↓
      MTE搬运下一Chunk
               ║
      Cube计算当前Chunk
```

整体结构如下：

```sql
AI Core 0 → Head 0 → 核内MTE/Cube/Vector流水
AI Core 1 → Head 1 → 核内MTE/Cube/Vector流水
AI Core 2 → Head 2 → 核内MTE/Cube/Vector流水
...
AI Core 7 → Head 7 → 核内MTE/Cube/Vector流水
```

因此：

- **多核 Head并行**提高核间并行度；
- **Tiling双缓冲**提高单核内部的流水利用率。

---

#### =={pink}8. 这种设计的限制==

##### =={yellow}（1）Head和AI core数量限制并行度==

当前只有8个 Head，所以最多自然拆出8个独立 Head任务。即使硬件上还有更多 AI Core，也没有更多 Head可以直接分配。

##### =={yellow}（2）小Shape导致单核任务较轻==

每个 Head只有 `64×64`，单核计算量较小。此时：

- =={green}Kernel调度；==
- =={green}多核启动；==
- 地址计算；
- =={green}GM带宽竞争==

=={green}等开销的占比会变高。==

##### =={yellow}（3）多核可能竞争内存带宽==

虽然不同核访问不同地址，但都需要从 GM读取 Q、K、V。如果多个核同时搬运数据，可能争用内存带宽，因此实际加速不会严格等于核数。

##### =={yellow}（4）Head数与核数不一致时需要循环分配==

当前是：

```undefined
8 Head = 8 Core
```

所以一核一 Head最简单。

更通用的分配方式可以是：

```bash
for (uint32_t head = blockIdx;
     head < headNum;
     head += blockDim) {
    compute_one_head(head);
}
```

例如4个核处理8个 Head：

```undefined
Core 0 → Head 0、4
Core 1 → Head 1、5
Core 2 → Head 2、6
Core 3 → Head 3、7
```

但当前原型采用的是更直接的8核对应8个 Head。

---

#### 面试简答

> 多核Head并行就是利用不同Attention Head之间相互独立的特点，把不同Head分配给不同AI Core。在我的自定义算子原型中输入包含8个Head，因此Host侧设置BlockDim为8，Device侧每个核通过BlockIdx确定自己的Head编号，并根据Head编号计算Q、K、V和Output的地址偏移。这样每个核独立完成一个Head的QK转置、Scale、Softmax和Weight乘V，最后写入不同的输出区域。不同Head之间没有中间结果依赖和写地址冲突，因此基本不需要核间同步；每个核内部再结合Tiling、双缓冲以及MTE、Cube、Vector流水协同。它的主要目的就是避免8个Head在单核上串行执行，提高多核利用率。

---

## 5. 单核内的流水线并行与同步机制：Tiling与Ping-Pong双缓冲

一个 AI Core 负责一个 Head，在核内依次完成：

```undefined
QKᵀ → Scale → Softmax → Weight×V → 输出
```

主要由以下流水线协同：

| 流水线 | 作用 |
| --- | --- |
| **MTE2** | 将 Q、K、V 从 GM 搬入 L1 或 UB |
| **MTE1** | 将数据从 L1 搬入 Cube 使用的 L0A/L0B |
| **PIPE_M** | Cube 执行 `QKᵀ` 和 `Weight×V` |
| **PIPE_V** | Vector 执行 Scale、Softmax及格式处理 |
| **MTE3** | 将最终结果从片上存储写回 GM |
| **Scalar** | 计算地址偏移、控制循环并下发指令 |

整体链路为：

```sql
GM
 ↓ MTE2
L1 / UB
 ↓ MTE1
L0A / L0B
 ↓ Cube
L0C
 ↓ Vector
UB
 ↓ MTE3
GM
```

---

### 5.1 流水线并行是怎么实现的

核心方法是 **Tiling + Ping-Pong双缓冲**。

先把一个 Head 内的大块数据沿 K 维切成多个 Chunk。例如原始 K 维为 64，可以切为两个 32：

```ini
K = 64 → Chunk 0（32）+ Chunk 1（32）
```

然后准备两组片上缓冲区：

```undefined
Ping Buffer → 处理偶数 Chunk
Pong Buffer → 处理奇数 Chunk
```

不同流水线可以同时操作不同缓冲区：

```sql
时间    MTE2/MTE1                  Cube
T0      搬运 Chunk 0 到 Ping       等待
T1      搬运 Chunk 1 到 Pong       计算 Ping 中的 Chunk 0
T2      等待/搬运下一块             计算 Pong 中的 Chunk 1
```

这样，Cube 计算当前块时，MTE 可以提前搬运下一块，尽量把数据搬运时间隐藏在计算时间中。

在当前 `64×64` 的小 Shape 下，只有两个 Chunk，因此流水重叠空间有限。双缓冲属于辅助优化，主要收益仍来自 **Kernel融合和中间结果片上驻留**。

### 5.2 为什么必须同步

不同流水线可以异步执行。如果没有同步，可能发生：

- Cube 还没读取完 Ping，MTE 就将下一批数据覆盖进去；
- 数据还没搬入 L0A/L0B，Cube 就开始矩阵乘；
- Score 还没计算完整，Vector 就开始 Softmax；
- Cube 还没产生最终结果，MTE3 就写回 GM。

因此，同步的本质是：

> **生产者通知消费者“数据已经准备好”，消费者处理完成后再通知生产者“缓冲区可以复用”。**

---

### 5.3 如何进行同步

主要使用事件机制：

```scss
set_flag(生产者流水线, 消费者流水线, EVENT_ID);
wait_flag(生产者流水线, 消费者流水线, EVENT_ID);
```

### 5.1 Tiling策略

#### 5.1.1 什么是Tiling？

**Tiling（分块）**就是把一个较大的计算任务切成若干个较小的数据块，再逐块完成计算。

例如，把长度为 64 的数据切成两块：

```css
完整数据： [0 ................................ 63]

Chunk 0：  [0 ........ 31]       长度 32
Chunk 1：                  [32 ........ 63]  长度 32
```

这样做不是改变计算结果，而是改变计算的执行方式，让每次只处理一小块数据。

它主要解决：

- 数据无法全部放入片上；
- 需要多核划分；
- 需要搬运与计算重叠；
- 需要处理不同Shape和尾块。

Tiling通常分成两部分：

```undefined
Host侧：计算怎么切
Device侧：按照参数执行
```

#### 5.1.2 什么是Chunk？

**Chunk**就是 Tiling 后得到的一个“数据块”。

```undefined
Tiling：切分这个动作或策略
Chunk：切分后得到的每一个小块
```

例如：

```undefined
64 → 32 + 32
```

表示沿某个维度进行了 Tiling，得到了两个 Chunk，每个 Chunk 的长度为 32。

#### 5.1.3 算子中对什么进行了Tiling？

当前 FusedAttention 原型使用的单个 Head 形状可以理解为：

```css
Q、K、V：[64, 64]
```

两个 64 的含义不同：

```markdown
             特征维度 headDim = 64
                        ↓
Q、K、V：        [64, 64]
                  ↑
             序列长度 seqLen = 64
```

也就是说：

- `seqLen = 64`：一个序列包含 64 个 Token；
- `headDim = 64`：每个 Token 在一个 Head 中由 64 个特征值表示。

第一次矩阵乘为：

$$
Score=QK^T
$$

其 Shape 变化是：

$$
[64,64]\times[64,64]^T=[64,64]
$$

这里进行 `64 → 32 + 32` 切分的，主要是矩阵乘法的**规约维 K**，也就是 `headDim=64` 这一维，而不是把 64 个 Head 切开。

为了避免和输入矩阵 K 混淆，可以记成：

```vbnet
矩阵 K：Attention 中的 Key

K 维：矩阵乘法中需要相乘并累加的公共维度
```

---

#### 5.1.4 为什么沿 K 维切成两个 32

原始计算可以写成：

$$
Score_{ij}=\sum_{k=0}^{63}Q_{ik}K_{jk}
$$

把 K 维切成两块后：

$$
Score_{ij} = \sum_{k=0}^{31}Q_{ik}K_{jk} + \sum_{k=32}^{63}Q_{ik}K_{jk}
$$

因此执行过程是：

```markdown
Chunk 0：计算 k=0～31
         ↓
         初始化部分结果 L0C

Chunk 1：计算 k=32～63
         ↓
         累加到同一个 L0C

最终得到完整 Score
```

数学结果没有改变，只是将一次长度为 64 的累加拆成了两次长度为 32 的累加。

---

#### 5.1.5 为什么选择 32，而不是其他数字？

选择 32 主要考虑三点：

1. **适配 Cube 的分块计算**
Ascend Cube矩阵计算适合按照规则、对齐的数据块执行。32 是较规整的计算粒度，可以继续匹配底层的矩阵分块要求。
2. **控制片上缓存占用**
一次只搬运和计算 32 个特征，L1、L0A、L0B 中每次需要保存的数据更少，更容易进行片上存储规划。
3. **为双缓冲提供两个阶段**
64 被切成两个 32 后，可以形成：
Cube计算前32维
║

MTE提前搬运后32维

从而为后面的 Ping-Pong 双缓冲创造搬运与计算重叠的机会。

需要注意：**32 并不是普遍最优值**。它是当前 `64×64` 原型 Shape 下结合硬件计算粒度、缓存使用和流水设计选择的 Tiling 参数。实际工程中通常由 Host 侧 Tiling 根据 Shape 和片上存储容量计算或选择。

#### 面试简答

> Tiling就是把一次较大的计算切成多个小块，切出来的每一块叫Chunk。在这个算子的单个Head中，Q、K、V的Shape是64×64，其中一个64是序列长度，另一个64是Head的特征维度。第一次QK转置矩阵乘时，我沿公共的规约K维把64切成两个32：第一块计算前32维并初始化L0C，第二块计算后32维并累加到L0C，最后得到完整Score。这样既能降低单次片上缓存占用，也能让Cube计算当前Chunk时，MTE提前搬运下一个Chunk，为后面的Ping-Pong双缓冲创造条件。

---

### 5.2 Ping-Pong双缓冲

#### 5.2.1 什么是 Buffer

数据从 **GM（片外全局内存）**搬进 AI Core 后，不能直接凭空交给 Cube 计算，需要先存放在片上存储空间中。

这块临时保存待计算数据的空间就是 **Buffer（缓冲区）**。

可以简单理解成：

```sql
GM：仓库
Buffer：工作台
Cube：负责计算的工人
```

数据先从仓库搬到工作台，Cube 再从工作台取数据进行矩阵计算。

---

#### 5.2.2  为什么单缓冲会产生等待

假设只有一套 Buffer：

```sql
GM → Buffer → Cube
```

两个 Chunk 只能依次处理：

```sql
① MTE把Chunk 0搬入Buffer
② Cube计算Chunk 0
③ MTE把Chunk 1搬入同一个Buffer
④ Cube计算Chunk 1
```

对应时间轴是：

```css
时间  ─────────────────────────────────────→

MTE：  [搬Chunk 0]             [搬Chunk 1]
Cube：             [算Chunk 0]             [算Chunk 1]
```

这里存在两个问题：

- MTE搬运 Chunk 0 时，Cube没有数据可算；
- Cube计算 Chunk 0 时，MTE不能把 Chunk 1 写入同一块 Buffer，否则会覆盖 Cube正在读取的数据。

所以，搬运和计算基本只能串行进行。

---

#### 5.2.3 什么是 Ping-Pong 双缓冲

双缓冲就是准备两套相互独立的 Buffer：

```undefined
Ping Buffer
Pong Buffer
```

在本算子的两个 Chunk 场景中：

```undefined
Chunk 0 → Ping Buffer
Chunk 1 → Pong Buffer
```

这样，Cube读取 Ping时，MTE可以同时写入 Pong：

```markdown
                 ┌──────────────┐
Chunk 0 ───────→ │ Ping Buffer  │ ───────→ Cube计算Chunk 0
                 └──────────────┘
                                                ║ 同时进行
                 ┌──────────────┐
Chunk 1 ───────→ │ Pong Buffer  │ ←─────── MTE搬运Chunk 1
                 └──────────────┘
```

因为 Cube 和 MTE访问的是不同 Buffer，所以不会发生数据覆盖。

---

#### 5.2.4 两个 Chunk 是怎么执行的

##### 第一阶段：填充 Ping

首先把 Chunk 0 搬到 Ping：

```sql
MTE：GM → Ping，搬运Chunk 0
Cube：等待
```

这是流水线的启动阶段，也叫**流水填充**。因为一开始还没有数据，Cube必须等待第一次搬运。

##### 第二阶段：计算与搬运重叠

Chunk 0准备好后：

```sql
Cube：从Ping读取并计算Chunk 0
MTE ：同时把Chunk 1搬入Pong
```

这是双缓冲真正产生并行的阶段：

```css
时间  ─────────────────────────────────────────→

MTE：  [Chunk 0 → Ping][Chunk 1 → Pong]
Cube：                 [计算Ping中的Chunk 0][计算Pong中的Chunk 1]
                       ╰──── 与搬Chunk 1重叠 ────╯
```

##### 第三阶段：计算 Pong

Chunk 0计算完成、Chunk 1搬运完成后：

```sql
Cube：读取Pong并计算Chunk 1
MTE：没有更多Chunk，暂时空闲
```

这是**流水排空阶段**。

---

#### 5. 为什么叫 Ping-Pong

“Ping-Pong”强调的是两套 Buffer **交替使用**，并不是 Ping 永远属于 Chunk 0、Pong 永远属于 Chunk 1。

如果以后有更多 Chunk，轮换关系是：

```undefined
Chunk 0 → Ping
Chunk 1 → Pong
Chunk 2 → Ping
Chunk 3 → Pong
……
```

执行过程则是：

```sql
Cube计算Ping中的Chunk 0
MTE搬运Chunk 1到Pong
              ↓
Cube计算Pong中的Chunk 1
MTE搬运Chunk 2到Ping
              ↓
Cube计算Ping中的Chunk 2
MTE搬运Chunk 3到Pong
```

两套 Buffer就像乒乓球一样来回切换，所以称为 Ping-Pong 双缓冲。

---

#### 5.2.6 它与 Tiling 的关系

两者解决的问题不同：

```undefined
Tiling
  ↓
把K维64切成两个32
  ↓
得到Chunk 0和Chunk 1
  ↓
Ping-Pong双缓冲
  ↓
为两个Chunk准备不同的临时存储空间
  ↓
让搬运下一块和计算当前块同时进行
```

因此可以记成：

> **Tiling负责把任务切开，双缓冲负责让切出来的数据块流水执行。**

如果没有 Tiling，就没有多个 Chunk 可供轮换；如果只有 Tiling、没有双缓冲，多个 Chunk仍可能只能串行搬运和计算。

---

#### 5.2.7 双缓冲并不能消除所有时间

即使使用双缓冲，仍然存在：

- 第一个 Chunk搬运时的**流水填充时间**；
- 最后一个 Chunk计算时的**流水排空时间**；
- 数据依赖造成的等待；
- 搬运时间和计算时间不完全相等造成的空隙。

对于当前只有两个 Chunk的 `64×64` 小 Shape，能够重叠的阶段比较少，所以双缓冲的收益有限。它主要是在建立一种可扩展的搬运—计算流水，算子的主要收益仍来自 **Kernel融合和中间结果片上驻留**。

#### 面试简答

> Tiling把K维64切成两个32以后，我为两个Chunk准备了Ping和Pong两套独立缓冲。使用单缓冲时，搬运和计算会争用同一块空间，只能先搬再算；使用双缓冲后，Cube计算Ping中的Chunk 0时，MTE可以同时把Chunk 1搬到Pong，从而把下一块的数据搬运隐藏在当前块的计算过程中。如果还有更多Chunk，Ping和Pong会交替复用。不过Buffer复用前必须确认上一轮计算已经结束，这就需要下一部分的事件同步机制。

---

### 5.3 算子中的并行机制

#### 5.3.1 并行在哪生效

算子中的流水线并行主要发生在MTE和CUBE之间

```sql
MTE：GM → 片上Buffer
Cube：读取Buffer → 执行矩阵乘法
```

它们是不同的硬件流水线，因此在没有数据冲突和依赖的情况下，可以同时工作。

#### 5.3.2 串行为什么慢

如果不做流水并行，执行顺序是：

```undefined
搬Chunk 0 → 算Chunk 0 → 搬Chunk 1 → 算Chunk 1
```

时间轴如下：

```css
时间  ─────────────────────────────────────────────→

MTE：  [搬运Chunk 0]            [搬运Chunk 1]
Cube：              [计算Chunk 0]            [计算Chunk 1]
```

MTE工作时，Cube在等待；Cube工作时，MTE也没有提前准备下一块数据。

如果：

- 每个 Chunk搬运需要 `Tmove`；
- 每个 Chunk计算需要 `Tcompute`；

那么串行总时间约为：

$$
T_{\text{serial}} = 2T_{\text{move}}+2T_{\text{compute}}
$$

#### 5.3.3 MTE与CUBE的流水线并行

双缓冲提供 Ping 和 Pong 两套独立空间后，可以让两个硬件单元同时处理不同 Chunk：

```sql
Cube：使用Ping计算当前Chunk 0
MTE ：使用Pong搬运下一Chunk 1
```

关键点是：

> 不是 MTE 和 Cube 同时处理同一个 Chunk，而是分别处理前后两个不同的 Chunk。

因为二者访问不同 Buffer，所以不会互相覆盖数据。

##### 第一阶段：流水填充

首先需要将 Chunk 0 搬到 Ping：

```css
MTE：搬运Chunk 0 → Ping
Cube：等待数据
时间段 T0：

MTE   [Chunk 0 → Ping]
Cube  [      等待      ]
```

Cube不能提前计算，因为 Chunk 0还没有准备完成。

---

##### 第二阶段：搬运与计算重叠

Chunk 0搬运完成后：

```css
Cube：读取Ping，计算Chunk 0
MTE ：同时把Chunk 1搬到Pong
时间段 T1：

MTE   [Chunk 1 → Pong]
Cube  [计算Ping中的Chunk 0]
```

这是整个设计中真正的**流水线并行区间**。

两个硬件单元做不同工作：

```markdown
             ┌──────────────┐
MTE ───────→ │ Pong：Chunk 1 │
             └──────────────┘

             ┌──────────────┐
Cube ←────── │ Ping：Chunk 0 │
             └──────────────┘
```

---

##### 第三阶段：流水排空

当下面两个条件都满足时：

```sql
Cube完成Chunk 0计算
MTE完成Chunk 1搬运
```

Cube才能开始计算 Chunk 1：

```css
MTE：没有下一个Chunk，空闲
Cube：读取Pong，计算Chunk 1
时间段 T2：

MTE   [      空闲      ]
Cube  [计算Pong中的Chunk 1]
```

这就是流水排空阶段。

---

##### 完整时间轴

```css
时间 ─────────────────────────────────────────────────────→

阶段：       流水填充             并行执行              流水排空

MTE：    [Chunk 0 → Ping]    [Chunk 1 → Pong]          [空闲]

Cube：   [     等待      ]    [计算Ping中的Chunk 0]     [计算Pong中的Chunk 1]
                           ╰────── 同时执行 ──────╯
```

用流程表示就是：

```sql
MTE搬Chunk 0到Ping
         ↓
┌───────────────────────────────┐
│ Cube计算Ping中的Chunk 0         │
│               ║ 同时           │
│ MTE搬Chunk 1到Pong             │
└───────────────────────────────┘
         ↓
Cube计算Pong中的Chunk 1
         ↓
完整计算结果
```

---

#### 5.3.4 Chunk 0 和 Chunk 1 的计算结果如何合并

这里沿矩阵乘法的规约 K 维进行切分。

Chunk 0计算：

$$
Score^{(0)} = Q[:,0:32]K[:,0:32]^T
$$

这一步用于**初始化 L0C**：

```ini
L0C = Score⁽⁰⁾
```

Chunk 1计算：

$$
Score^{(1)} = Q[:,32:64]K[:,32:64]^T
$$

这一步需要累加到前面的结果：

```makefile
L0C += Score⁽¹⁾
```

最终得到：

$$
Score=Score^{(0)}+Score^{(1)}
$$

因此：

```sql
Cube计算Chunk 0 → 初始化L0C
                         ↓
Cube计算Chunk 1 → 累加L0C
                         ↓
                 得到完整Score
```

两次 Cube计算本身存在累加依赖，所以不能互相并行；能够并行的是：

```markdown
计算Chunk 0
      ║
搬运Chunk 1
```

---

#### 5.3.5 流水并行节省了哪部分时间

串行执行：

$$
T_{\text{serial}} = T_{\text{move0}}+ T_{\text{compute0}}+ T_{\text{move1}}+ T_{\text{compute1}}
$$

流水执行：

$$
T_{\text{pipeline}} = T_{\text{move0}}+ \max \left( T_{\text{compute0}}, T_{\text{move1}} \right) + T_{\text{compute1}}
$$

因为 `compute0` 和 `move1` 同时进行，所以这两部分不再简单相加，而是主要由耗时较长的一方决定。

例如：

```undefined
计算Chunk 0：10 μs
搬运Chunk 1：6 μs
```

二者重叠后，中间阶段约为 10 μs，其中 6 μs搬运时间被 Cube计算隐藏。

#### 5.3.6 流水并行成立的条件

要让二者安全并行，必须满足：

1. **访问不同 Buffer**
Cube读取Ping
MTE写入Pong
2. **Cube计算前，当前 Chunk必须搬运完成**
Chunk 0搬完 → Cube才能计算Chunk 0
3. **Buffer再次写入前，上一次读取必须结束**
Cube用完Ping → MTE才能覆盖Ping
4. **部分结果必须按顺序累加**
Chunk 0初始化L0C
Chunk 1累加L0C

这些先后关系不能依靠默认执行顺序来猜测，需要通过事件明确表达——这就是下一部分的 **`set_flag` / `wait_flag`同步机制**。

---

### 5.4 单核内的同步机制

#### 5.4.1 为什么需要同步

假设没有同步，可能出现三类错误。

##### （1）数据还没搬完，Cube就开始计算

```markdown
MTE正在写入Ping
        ║
Cube已经读取Ping  ✕
```

Cube可能读到不完整或旧的数据。

##### （2）Buffer还没用完，MTE就提前覆盖

如果后面还有 Chunk 2，需要重新使用 Ping：

```markdown
Cube仍在读取Ping中的Chunk 0
             ║
MTE把Chunk 2写入Ping  ✕
```

Chunk 0会被提前覆盖。

##### （3）部分结果还没完成，就进入下一阶段

```markdown
Chunk 1尚未累加完成
          ║
Vector开始计算Softmax  ✕
```

Softmax必须基于完整 Score，不能只处理部分结果。

所以，同步本质上是在保证：

> **消费者必须等待数据就绪，生产者必须等待缓冲区释放，后续阶段必须等待前置结果完整。**

---

#### 5.4.2 同步的基本工具

在较底层的昇腾算子编程中，可以使用事件同步：

```scss
set_flag(生产者流水线, 消费者流水线, EVENT_ID);
wait_flag(生产者流水线, 消费者流水线, EVENT_ID);
```

可以把它理解成“发信号”和“等信号”：

```markdown
生产者完成工作
      ↓
set_flag：发送完成事件
      ↓
消费者wait_flag：等待事件
      ↓
消费者开始下一步
```

这不是让整个 AI Core停下来，而是只约束**存在真实数据依赖的两条流水线**。

> 上面的代码用于表达同步关系，具体接口参数和封装形式需要以实际使用的 Ascend C/CANN 版本为准。

---

#### 5.4.3 第一道同步：数据搬完，Cube才能计算

以 Chunk 0和 Ping为例：

```sql
MTE将Chunk 0搬入Ping
           ↓
MTE发出“Ping数据就绪”事件
           ↓
Cube等待该事件
           ↓
Cube读取Ping并计算Chunk 0
```

概念代码如下：

```scss
// MTE：把Chunk 0搬入Ping
copy_chunk_to_ping(chunk0);

// 通知Cube：Ping中的数据已经准备完成
set_flag(PIPE_MTE1, PIPE_M, EVENT_ID0);

// Cube：等待Ping数据就绪
wait_flag(PIPE_MTE1, PIPE_M, EVENT_ID0);

// Cube开始计算Chunk 0
cube_compute(pingBuffer);
```

如果省略这一步，Cube可能在搬运完成前就开始读取 Ping。

---

#### 5.4.4 第二道同步：Buffer用完，MTE才能复用

如果存在 Chunk 2，就要再次使用 Ping。

此时 Cube计算完 Chunk 0后，需要通知 MTE：

```sql
Cube完成对Ping的读取
          ↓
Cube发出“Ping可以复用”事件
          ↓
MTE等待该事件
          ↓
MTE把Chunk 2搬入Ping
```

概念代码：

```scss
// Cube使用完Ping
cube_compute(pingBuffer);

// 通知MTE：Ping已经可以复用
set_flag(PIPE_M, PIPE_MTE1, EVENT_ID0);

// MTE准备覆盖Ping前等待
wait_flag(PIPE_M, PIPE_MTE1, EVENT_ID0);

// 将Chunk 2搬入Ping
copy_chunk_to_ping(chunk2);
```

因此，每套 Buffer都存在一个生产者—消费者闭环：

```markdown
MTE写入Buffer
      ↓ 数据就绪
Cube读取并计算
      ↓ Buffer释放
MTE下一轮才能覆盖
```

当前算子只有两个 Chunk时，Ping在本轮不一定需要再次复用；但双缓冲结构通常仍按这个完整闭环设计，方便支持更多 Chunk。

---

#### 5.4.5 Ping和Pong为什么需要不同事件

两套 Buffer需要分别维护自己的状态：

```undefined
Ping Buffer ↔ EVENT_ID0
Pong Buffer ↔ EVENT_ID1
```

执行关系为：

```sql
MTE搬完Ping  ── EVENT_ID0 ──→ Cube读取Ping
MTE搬完Pong  ── EVENT_ID1 ──→ Cube读取Pong
```

如果 Ping和 Pong错误地共用同一个事件，就可能发生：

```markdown
Pong的搬运完成事件
        ↓
被误认为Ping已经准备完成
        ↓
Cube读取错误的Buffer
```

所以每套缓冲必须使用可区分的事件，事件编号通常可以根据 Ping/Pong索引选择：

```cpp
int bufferId = chunkIndex % 2;

if (bufferId == 0) {
    // Ping：EVENT_ID0
} else {
    // Pong：EVENT_ID1
}
```

---

#### 5.4.6 第三道同步：完整Score生成后才能做Softmax

两个 Chunk对应的是同一次矩阵乘法的部分结果：

```undefined
Chunk 0：初始化L0C
Chunk 1：累加L0C
```

执行顺序必须是：

```sql
Cube计算Chunk 0
      ↓ 初始化L0C
Cube计算Chunk 1
      ↓ 累加L0C
得到完整Score
      ↓
通知Vector流水线
      ↓
Scale + Softmax
```

也就是：

```vbnet
PIPE_M：完整Score计算完成
              ↓ Event
PIPE_V：开始Scale和Softmax
```

Softmax不能与尚未完成的 Score累加并行，因为 Softmax需要使用一整行完整的 Score计算最大值和归一化分母。

---

#### 5.4.7 后续阶段也有同步边界

整个 Attention链路中的主要同步关系是：

```sql
MTE完成Q/K搬运
        ↓
Cube执行QKᵀ
        ↓
所有K维Chunk累加完成
        ↓
Vector执行Scale + Softmax
        ↓
完整Weight生成
        ↓
Cube执行Weight × V
        ↓
输出结果计算完成
        ↓
Vector完成必要的格式整理
        ↓
MTE3写回GM
```

可以归纳为以下四个关键边界：

##### MTE → Cube

```sql
输入数据搬运完成 → Cube才能开始矩阵乘
```

##### Cube → Vector

```undefined
完整Score生成 → Vector才能执行Softmax
```

##### Vector → Cube

```sql
完整Weight生成 → Cube才能执行Weight×V
```

##### Cube/Vector → MTE3

```undefined
最终结果及格式处理完成 → MTE3才能写回GM
```

---

#### 5.4.8 同步和并行的关系

同步不是越多越好。

##### 同步不足

可能造成：

- 读取未完成的数据；
- Buffer被提前覆盖；
- 使用不完整的中间结果；
- 输出结果错误。

##### 同步过多

例如每条指令后都等待：

```undefined
搬一点 → 等待 → 算一点 → 等待 → 再搬一点
```

会把原本可以并行的 MTE和 Cube重新变成串行，双缓冲失去意义。

因此设计原则是：

> **只在真实的数据依赖和Buffer复用位置同步，其余阶段尽量异步下发，让硬件流水线重叠执行。**

---

#### 5.4.9 一张简化的同步流程图

```markdown
MTE流水线                         Cube流水线

搬运Chunk 0 → Ping
        │
        ├── 数据就绪事件 ID0 ──────→ 等待ID0
        │                              │
        │                              ▼
搬运Chunk 1 → Pong              计算Ping中的Chunk 0
        │                              │
        ├── 数据就绪事件 ID1 ──────→ 等待ID1
                                       │
                                       ▼
                                计算Pong中的Chunk 1
                                       │
                                完整Score生成
                                       │
                                       ├── 完成事件 ──→ Vector
                                       │
                                       ▼
                                Buffer可安全复用
```

从时间上看：

```css
时间 ───────────────────────────────────────────────→

MTE：  [搬Ping] ─发ID0─ [搬Pong] ─发ID1─────────────

Cube：          等ID0→[计算Ping]→等ID1→[计算Pong]
                      ╰ 与搬Pong并行 ╯

Vector：                                  等完整Score→[Softmax]
```

---

#### 面试简答

> MTE和Cube是异步流水线，所以双缓冲还需要事件同步保证正确性。同步主要有两个方向：MTE把数据搬进Ping或Pong后，通过事件通知Cube数据已经就绪；Cube使用完某个Buffer后，再通知MTE该Buffer可以复用。Ping和Pong使用不同的Event ID，避免把两套缓冲的状态混淆。此外，两个K维Chunk必须先在L0C中完成初始化和累加，生成完整Score后，才能通知Vector执行Softmax。我的原则是只在数据就绪、Buffer复用和算法强依赖位置同步，避免过度同步破坏流水并行。

---

### 5.5 为什么当前收益可能有限

虽然 Tiling、双缓冲和流水并行能够隐藏部分数据搬运时间，但在当前算子的 `64×64` 小 Shape中，实际收益可能比较有限，主要有以下原因。

1. 只有两个 Chunk，可重叠的次数太少
2. 存在流水填充和排空开销
3. 搬运和计算时间不一定均衡
4. 小 Shape下固定开销占比较高，双缓冲本身也有控制成本，当能够隐藏的搬运时间很短时，新增的事件和切换开销会抵消一部分收益。

理论分析中：

```undefined
单缓冲：约7.8 μs
双缓冲：约7.6 μs
```

所以整算子理论收益约3%。

这说明：

> 当前方案的主要收益来自单Kernel融合和片上驻留，Ping-Pong更多是在建立可扩展的流水结构。

---

## =={pink}6. 格式转换内联详设==

### 6.1 什么是数据格式（layout布局）

这里的“格式”不是 FP16、FP32这样的数据类型，而是**元素在内存中的排列方式**。

#### （1）ND格式

ND可以理解成普通的连续矩阵布局：

```undefined
一行接一行保存
```

例如：

```undefined
a00 a01 a02 a03
a10 a11 a12 a13
a20 a21 a22 a23
```

这种格式便于通用算子和 Host理解。

#### （2）NZ等分形格式

Cube矩阵计算单元不是逐个元素计算，而是以小矩阵块为单位计算。NZ会把矩阵拆成硬件适合处理的 Tile，再按特定顺序排列：

```sql
普通矩阵
   ↓ 切成多个Tile
Tile 0、Tile 1、Tile 2……
   ↓ 按Cube需要重新排布
FRACTAL_NZ
```

它更适合 Cube执行矩阵乘法，但不一定适合 Vector直接按行执行 Softmax。

### 6.2 原始图中的layout冲突

普通业务张量通常使用ND格式，而Cube为了提高矩阵计算效率，通常使用NZ等分形格式，最终输出又需要ND格式。

优化前：

```css
Q、K
 ↓ ND→NZ
QKᵀ
 ↓ NZ→ND
Scale + Softmax
 ↓ ND→NZ
Weight×V
 ↓ NZ→ND
Output
```

每次格式转换都可能被编译成一个独立的 `TransData Kernel`

问题不只是“重新排一下数据”，还包括：

- 多一次 Kernel启动；
- 多一次或多次 GM读写；
- Cube和 Vector之间被独立 Kernel隔开；
- Score、Weight等中间结果无法留在片上。

这些独立 TransData以及相关转置，是重要的额外开销，profiling分析显示：

- NZ/ND转换约占相关链路时延的 `22.5%`；
- 转置搬运约占 `16.9%`。

### =={pink}6.2 什么是格式转换内联==

=={yellow}内联不是完全不需要改变数据排列==，而是：

- =={yellow}**不再单独启动TransData Kernel**，而是**在数据搬运**或**现有计算过程**中**顺便完成格式调整**。==

### =={pink}6.3 具体实现==

#### =={yellow}6.3.1 输入端：通过算子接口声明NZ格式==

在 OpDef中把输入格式声明为 `FORMAT_FRACTAL_NZ`，让 ATC在编译阶段完成格式协商。

逻辑上是：

```markdown
模型上游输出
      ↓
ATC根据算子接口选择或插入布局转换
      ↓
自定义Kernel直接接收NZ输入
```

=={green}这样 Kernel内部**不需要先启动三个独立的 ND→NZ转换**过程，而是直接按照 NZ地址读取 Q、K、V。==

#### =={yellow}6.3.2 第一次矩阵乘：在L1→L0装载时表达K转置==

第一次矩阵乘需要计算：

$$
Score=QK^T
$$

`mad`本身执行的是：

$$
C=A\times B
$$

=={yellow}它**不会**在计算过程中**临时调用一个转置 Kernel**。==因此，=={yellow}本方案通过== **=={yellow}L0B中的Tile排列方式==**=={yellow}让 Cube看到逻辑上的== $K^T$。

数据路径为：

```scss
Q：GM(NZ) → L1 → L0A
K：GM(NZ) → L1 → L0B
                         ↓
               Cube按照L0布局读取
                         ↓
                     Q × Kᵀ
```

实现思想是：

- =={green}Q在== `L1 → L0A`=={green}装载时，通过== `load_cbuf_to_ca`=={green}组织成 Cube需要的布局；==
- =={green}K在== `L1 → L0B`=={green}装载时保留对应的 NZ Tile顺序；==
- =={green}Cube按照 **L0B的分形布局读取**时，**逻辑上看到**的是==

$$
K^T
$$

也就是：

> =={yellow}**K的转置被吸收到MTE1的装载布局中**==，没有额外启动Transpose Kernel。

#### 6.3.3 中间过程：Score和Weight留在UB中

第一次 Cube计算完成后：

```markdown
L0C中的Score
      ↓
转换为FP16并搬到UB
      ↓
Vector直接执行Scale和Softmax
      ↓
Weight继续保留在UB
```

完整链路是：

```sql
Cube：QKᵀ
   ↓
L0C
   ↓ 片内搬运
UB中的Score
   ↓
Vector：Scale + Softmax
   ↓
UB中的Weight
   ↓
第二次Cube矩阵乘
```

这里不再执行：

```undefined
Score写回GM
→ NZ→ND TransData
→ Softmax
→ Weight写回GM
→ ND→NZ TransData
→ Weight×V
```

因此，中间格式适配是在同一个 Kernel的片上数据路径中完成的，Score和 Weight不需要通过独立 TransData Kernel在 GM中来回中转。

这是格式转换内联最主要的收益来源。

---

#### 6.3.4 第二次矩阵乘：装载时适配Score和V

第二次矩阵乘为：

$$
Output=Weight\times V
$$

Weight目前在 UB中，V输入在 GM的 NZ布局中。

数据路径为：

```markdown
Weight：UB → L1 → L0A
V：     GM → L1 → L0B
                    ↓
             Cube计算Weight×V
```

在 MTE1从 L1装载到 L0A/L0B时：

- Weight按照 L0A需要的布局组织；
- V通过 `load_cbuf_to_cb(..., transpose=1)`调整 Tile读取方式；
- Cube最终看到的是正常方向的 V，而不是

$$
V^T
$$

因此，V的方向适配也被吸收到了 L1→L0B的数据装载过程。

---

#### 6.3.5 输出端：在UB内完成zN→ND

第二次 Cube矩阵乘后，结果从 L0C搬到 UB时仍然是 Cube友好的 `zN`分形排列，而算子对外输出要求是 ND格式。

原始做法是：

```markdown
输出zN写入GM
      ↓
启动TransData Kernel
      ↓
读取并转换成ND
      ↓
再次写入GM
```

本方案直接在 UB中使用 Vector指令完成重排：

```undefined
L0C
 ↓
UB中的zN结果
 ↓ vmuls十参数模式
UB中的ND结果
 ↓ MTE3
GM中的ND Output
```

从而让 Vector按 zN顺序读取、按 ND顺序写入，实现片上重排。

因此：

> 输出格式转换不再是独立TransData Kernel，而是FusedAttention内部最后一个Vector处理步骤。

---

#### 6.3.6 完整格式处理链路

```css
输入Q/K/V
    │
    │ OpDef声明FRACTAL_NZ
    ▼
GM中的NZ输入
    │
    │ MTE2：GM→L1
    ▼
L1
    │
    │ MTE1装载时调整Tile布局
    ▼
L0A / L0B
    │
    │ Cube看到需要的矩阵方向
    ▼
QKᵀ
    │
    │ L0C→UB，中间结果不落GM
    ▼
UB：Scale + Softmax
    │
    │ UB→L1→L0，装载时适配Weight和V
    ▼
Weight×V
    │
    ▼
UB中的zN输出
    │
    │ Vector vmuls十参数模式重排
    ▼
UB中的ND输出
    │
    │ MTE3
    ▼
GM中的ND Output
```

---

#### 6.3.7 为什么这样做能提升性能

格式转换内联主要带来四方面收益。

##### （1）减少独立Kernel启动

```undefined
多个TransData Kernel → 融合Kernel内部处理
```

小 Shape下 Kernel启动开销可能比实际计算时间更突出。

##### （2）减少GM往返

Score和 Weight保留在 UB，不再为了格式转换反复：

```undefined
写GM → 读GM → 转换 → 再写GM
```

##### （3）利用现有搬运顺便调整布局

K和 V的方向适配融合在 MTE1装载过程中，不额外执行完整的转置操作。

##### （4）保持Cube和Vector各用合适布局

内联并不强迫所有硬件使用同一种格式，而是在片内边界进行适配：

```sql
Cube使用分形布局
Vector处理Softmax
最终输出转换为ND
```

### 6.6 面试回答

> ND是普通张量格式，NZ是更适合Cube矩阵分块计算的格式。优化前，Cube和Vector之间需要多个独立TransData。我的设计是通过算子输入格式声明、MTE加载参数和片上数据重排，把K转置及部分格式整理结合到搬运和Kernel内部执行，并在UB中整理最终输出。这里不是完全没有格式处理，而是尽量避免中间结果产生独立TransData Kernel。

---

## 7. 六项方法之间的关系

这六项方法不是相互独立的，而是一条完整主线：

```sql
单Kernel融合
解决Kernel过多
        ↓
Cube与Vector协同
解决不同计算如何在同一Kernel执行
        ↓
片上数据驻留
解决不同阶段如何传递中间结果
        ↓
多核Head并行
解决不同Head如何同时执行
        ↓
Tiling与双缓冲
解决单核内部怎样分块并重叠搬运
        ↓
格式转换内联
解决Cube、Vector和业务格式怎样衔接
```

### 最终面试总结

> 这几项技术其实是一层一层推进的。首先通过单Kernel融合重新划分Attention执行边界；然后使用Cube完成矩阵乘、Vector完成Softmax；为了让两个单元协同，将Score和Weight保留在L0、UB和L1中，不再写回GM；再利用Head独立性进行多核并行；单核内部通过Host Tiling将K维分块，并使用Ping-Pong和Event同步重叠搬运与计算；最后把必要的转置和格式整理融入MTE加载及Kernel内部，减少独立TransData。这样就形成了从计算融合、存储优化到并行流水的完整自定义算子设计。

## 8. 自定义算子性能对比与理论收益

#### 四种方案总体对比

| 指标 | 原始非融合 | ATC自动融合 | 自定义单缓冲 | 自定义双缓冲 |
| --- | --- | --- | --- | --- |
| Kernel数量 | 10个 | 9个 | 1个 | 1个 |
| 中间结果GM往返 | 3次 | 3次 | 0次 | 0次 |
| GM搬运量 | 约205 KB | 约197 KB | 约32 KB | 约32 KB |
| 总搬运量 | 约350 KB | 约342 KB | 约80 KB | 约80 KB |
| 独立TransData | 3个 | 3个 | 0个 | 0个 |
| 最大流水并行度 | 1 | 1 | 2 | 3 |
| 理论时延 | 约58 μs | 约53 μs | 约7.8 μs | 约7.6 μs |
| 实测时延 | 1470 μs | 99.9 μs | 待验证 | 待验证 |

#### 自定义算子核心成果：

在Attention计算图中：

1. 将kernel数量从9个压缩到1个
2. 消除了三次中间结果GM往返(3->0)、三个TransData(3->0)
3. 最大流水并行度（1->3）
4. GM搬运数据量降低了84%
5. 总搬运数据量降低了77%
6. 理论时延降低了86%（53μs->7.6μs）

最大并行度为3主要出现在第二次Weight乘V阶段。Cube计算当前Chunk时，MTE2可以从GM预取下一Chunk的V，MTE3同时把UB中的下一Chunk Weight搬到L1，三者使用不同流水线和不同Ping-Pong Buffer，因此可以形成MTE2、MTE3和Cube三路并行。这里的3是局部峰值流水并行度，不是3个AI Core，也不代表性能提升3倍。
