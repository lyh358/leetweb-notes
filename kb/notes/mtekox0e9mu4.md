## =={pink}一、功能基础信息==

---

### 1. 功能标识

- =={yellow}**诊断项ID**==：`DIAGID_FPGA_PARA1 = 0x1E00`
- =={yellow}**故障事件ID**==：`EVTID_FPGA_PARA1_FAULT = 0x1E01-` 0x1E06
- 功能全称：Program Parameters CRC Diagnosis（FPGA参数上电CRC完整性自检）
- =={yellow}**执行周期**==：`DIAG_PERIOD_ST`，**=={yellow}整机上电仅执行一次==**，属于上电自检类诊断
- =={yellow}**故障等级**：FTL7==（Output_Untrusted，=={yellow}点云数据不可信==）
- =={yellow}**防抖配置**：debounce=1，**单次异常直接确认故障**==

### 核心目标

=={green}外设参数（Para1）从Flash加载完成后，对比==**=={green}Flash原始存储CRC (storage Crc)==** =={green}和====={green}**={green}加载实时计算CRC (calc Crc)**=={yellow}=={green}；二者**不一致则上报DEM故障**，标识参数被篡改/加载损坏；同时**支持产线故障注入**校验、**故障快照冻结帧输出**。==

### =={pink}整体五层软件分层架构==

| 分层 | 定位 | 核心职责 | 可裁剪性 |
| --- | --- | --- | --- |
| =={yellow}**配置层**== | **规则定义**者 | CSV配置故障ID、等级、防抖、函数绑定；自动生成`diag_cfg.c`三表 | 项目必配，不可裁剪 |
| =={yellow}**调度层**== | **平台调度**器 | 上电自检周期调度、DEM事件上报、故障生命周期管理 | 平台通用，不可裁剪 |
| =={yellow}**算法层**== | Feature**核心逻辑** | 三态状态机、CRC比对判定、故障注入旁路、KeyInfo快照输出 | 静态代码，跨项目可复用 |
| =={yellow}**数据源层**== | **数据提供**，底层CDD | Flash参数加载、实时CRC计算、双CRC读取接口、加载状态查询 | CDD基础组件，Feature强依赖 |
| =={yellow}**工站层**== | 产**验收闭环**链路 | PTC命令故障注入/整机复位/故障查询、NVM注入参数持久化 | 仅Factory量产固件，Release可裁剪 |

## =={pink}二、算法层核心代码：diag_fpgapara1.c==

### 2.1 宏与类型定义

```c
// CRC无效哨兵值（读取失败/未初始化填充）
#define DIAG_FPGAPARA1_CRC_DEFAULT_VALUE (0xFFFFFFFFU)
// KeyInfo快照偏移与长度：4B存储CRC + 4B计算CRC = 8字节
#define FPGAPARA1_CRC_ST_OFFSET_CRC_STORAGE (0U)
#define FPGAPARA1_CRC_ST_OFFSET_CRC_CALC    (4U)
#define FPGAPARA1_CRC_ST_KEY_INFO_LEN       (8U)

// 自检三态状态枚举
typedef enum
{
    DIAG_FPGAPARA1_WAIT_PERIPARA_LOADED = 0U, // 状态0：等待外设参数加载完成
    DIAG_FPGAPARA1_CRC_RUN,                   // 状态1：执行CRC比对诊断
    DIAG_FPGAPARA1_DONE                       // 状态2：自检完成，不再执行
} DiagFpgaPara1CrcMainStatus;

// CRC双值快照结构体，故障时存入冻结帧
typedef struct
{
    uint32 storageCrc; // Flash原始存储CRC
    uint32 calcCrc;    // 加载过程实时计算CRC
} DiagFpgaPara1CrcInfo;

// 模块静态全局变量
static DiagFpgaPara1CrcInfo s_fpgaPara1CrcInfo;
static DiagFpgaPara1CrcMainStatus s_fpgaPara1CrcMainStatus;
boolean g_diagFpgaPara1CrcFaultInjectEnable = FALSE;
```

### 2.2 初始化函数 DiagFpgaPara1Init

```c
DiagRetCode DiagFpgaPara1Init(void)
{
    DiagRetCode ret = RET_DIAG_NORMAL;
    // 清空CRC快照缓存
    s_fpgaPara1CrcInfo.storageCrc = 0U;
    s_fpgaPara1CrcInfo.calcCrc = 0U;
    // 状态机复位至等待参数加载
    s_fpgaPara1CrcMainStatus = DIAG_FPGAPARA1_WAIT_PERIPARA_LOADED;
    return ret;
}
```

**作用**：上电自检框架启动时仅调用一次，重置缓存与状态机。

### 2.3 主状态机任务 DiagFpgaPara1MainTask

由`DiagSelftestTask`每5ms周期调度，三状态流转逻辑：

```c
DiagRetCode DiagFpgaPara1MainTask(void)
{
    DiagRetCode ret = RET_DIAG_NORMAL;
    switch (s_fpgaPara1CrcMainStatus)
    {
        case DIAG_FPGAPARA1_WAIT_PERIPARA_LOADED:
            // 判断CDD参数是否加载完毕
            if(DiagFpgaPara1WaitPeriParaLoaded() == E_OK)
            {
                (void)DiagFpgaPara1RefreshCrcSnapshot();
                // 切换至CRC诊断执行态
                s_fpgaPara1CrcMainStatus = DIAG_FPGAPARA1_CRC_RUN;
            }
            // 参数未就绪，持续返回未完成，下周期重试
            ret = RET_DIAG_UNFINISHED;
            break;

        case DIAG_FPGAPARA1_CRC_RUN:
            // 执行CRC判定+DEM上报
            ret = DiagFpgaPara1GetCrcDiagResult();
            // 自检仅执行一次，切换完成态不再调度
            s_fpgaPara1CrcMainStatus = DIAG_FPGAPARA1_DONE;
            break;

        case DIAG_FPGAPARA1_DONE:
            // 自检完成，空操作
            break;

        default:
            // 非法状态防御处理
            break;
    }
    return ret;
}
```

**时序流程**：上电Init → 周期轮询等待参数 → 参数就绪刷新CRC快照 → 单次比对判定 → 永久完成不再运行。

### 2.4 辅助函数1：等待参数加载完成

```c
static Std_ReturnType DiagFpgaPara1WaitPeriParaLoaded(void)
{
    Std_ReturnType ret = E_NOT_READY;
    // CDD状态>=LOADED代表加载流程结束（成功/失败均满足）
    if (CddPeriPara_GetStatus() >= CDD_PERIPARA_STATE_LOADED)
    {
        ret = E_OK;
    }
    return ret;
}
```

**约束**：必须等待CDD参数加载结束后再读取CRC，避免空数据误判故障。

### 2.5 辅助函数2：刷新双CRC快照

```c
static Std_ReturnType DiagFpgaPara1RefreshCrcSnapshot(void)
{
    Std_ReturnType ret = E_OK;
    Std_ReturnType retGet, retCal;

    // 读取Flash存储CRC
    retGet = CddPeriPara_GetParaCrcGet(CDD_PERIPARA_PARA_A_ID_PARA1, &s_fpgaPara1CrcInfo.storageCrc);
    // 读取加载实时计算CRC
    retCal = CddPeriPara_GetParaCrcCal(CDD_PERIPARA_PARA_A_ID_PARA1, &s_fpgaPara1CrcInfo.calcCrc);

    // 读取失败填充无效哨兵值0xFFFFFFFF
    if (retGet != E_OK)
    {
        s_fpgaPara1CrcInfo.storageCrc = DIAG_FPGAPARA1_CRC_DEFAULT_VALUE;
        ret = E_NOT_OK;
    }
    if (retCal != E_OK)
    {
        s_fpgaPara1CrcInfo.calcCrc = DIAG_FPGAPARA1_CRC_DEFAULT_VALUE;
        ret = E_NOT_OK;
    }
    return ret;
}
```

### 2.6 核心判定函数：CRC比对+故障注入优先级逻辑

```c
static DiagRetCode DiagFpgaPara1GetCrcDiagResult(void)
{
    DiagRetCode ret = RET_DIAG_NORMAL;
    boolean errFlag = FALSE;
    // 读取产线故障注入指令
    StInjectRes injectRes = FaultInjectSelftestGetCmdResult(DIAGID_FPGA_PARA1);

    // 优先级1：注入强制故障，跳过真实CRC比对
    if (injectRes == ST_INJECT_RES_FAIL)
    {
        ret |= DiagSendResultToDem(EVTID_FPGA_PARA1_FAULT, TRUE);
        return ret;
    }
    // 优先级2：注入强制正常，跳过真实CRC比对
    if (injectRes == ST_INJECT_RES_PASS)
    {
        ret |= DiagSendResultToDem(EVTID_FPGA_PARA1_FAULT, FALSE);
        return ret;
    }

    // 优先级3：无注入，执行真实CRC一致性判断
    // 任意CRC为0/0xFFFFFFFF，视为无效，不报故障
    if ((s_fpgaPara1CrcInfo.storageCrc == DIAG_FPGAPARA1_CRC_DEFAULT_VALUE || s_fpgaPara1CrcInfo.storageCrc == 0U)
        || (s_fpgaPara1CrcInfo.calcCrc == DIAG_FPGAPARA1_CRC_DEFAULT_VALUE || s_fpgaPara1CrcInfo.calcCrc == 0U))
    {
        errFlag = FALSE;
    }
    else
    {
        // 双CRC均有效，不一致则故障
        errFlag = (boolean)(s_fpgaPara1CrcInfo.storageCrc != s_fpgaPara1CrcInfo.calcCrc);
    }
    // 上报故障状态至DEM事件管理
    ret |= DiagSendResultToDem(EVTID_FPGA_PARA1_FAULT, errFlag);
    return ret;
}
```

**判定优先级**：故障注入FAIL > 故障注入PASS > 原生CRC一致性比对
**容错规则**：CRC读取失败填充哨兵值时，强制无故障，规避误报。

### 2.7 KeyInfo冻结帧快照接口

```c
DiagRetCode DiagFpgaPara1GetKeyInfo(const EventId evtId, uint8 *const keyInfo, uint16 *const keyInfoLen, const DiagkeyInfo keyInfoType)
{
    DiagRetCode ret = RET_DIAG_NORMAL;
    // 仅处理本故障事件
    if (evtId == EVTID_FPGA_PARA1_FAULT)
    {
        // 简略/详细快照均支持
        if ((keyInfoType == KEY_INFO_DETAIL) || (keyInfoType == KEY_INFO_SIMPLE))
        {
            // 缓冲区容量充足才填充8字节快照
            if (*keyInfoLen >= FPGAPARA1_CRC_ST_KEY_INFO_LEN)
            {
                *keyInfoLen = FPGAPARA1_CRC_ST_KEY_INFO_LEN;
                memcpy(keyInfo + FPGAPARA1_CRC_ST_OFFSET_CRC_STORAGE, &s_fpgaPara1CrcInfo.storageCrc, sizeof(uint32));
                memcpy(keyInfo + FPGAPARA1_CRC_ST_OFFSET_CRC_CALC, &s_fpgaPara1CrcInfo.calcCrc, sizeof(uint32));
            }
        }
    }
    return ret;
}
```

**作用**：UDS读取冻结帧时输出8字节快照，4B存储CRC+4B计算CRC，售后/产线定位参数损坏根源。

## 三、数据源层 CddPeriPara 接口说明

诊断算法层不自行加载Flash、不计算CRC，仅消费CDD输出数据：

1. **CddPeriPara_LoadPara**：上电异步加载Flash Para1参数块，解析头部/指令/数据段
2. **calcCrc生成**：加载过程增量调用`LibCrc_ParaCrc32`实时累加校验值
3. **storageCrc读取**：从Flash参数头部读出出厂存储CRC
4. **状态接口 **`CddPeriPara_GetStatus()`

- `CDD_PERIPARA_STATE_LOADING`：加载中
- `CDD_PERIPARA_STATE_LOADED`：加载完成（成功/损坏）
- `CDD_PERIPARA_STATE_LOADED_FAIL`：加载失败

1. 配套读取API

- `CddPeriPara_GetParaCrcGet()`：获取Flash原始CRC
- `CddPeriPara_GetParaCrcCal()`：获取加载实时CRC

## 四、工站层：产线故障注入与验收闭环

### 4.1 PTC配套命令

| PTC主/子命令 | 功能 |
| --- | --- |
| 0x4F | 写入NVM故障注入配置（ST模式+期望故障码） |
| 0x10 | 整机电源复位，触发自检重新执行 |
| 0xFF sub 0x1F | 查询DEM确诊故障列表，校验0x1E01故障 |
| 0x4F read | 读取当前生效注入配置 |

### 4.2 完整产验收流程

1. PTC下发0x4F写入注入指令，NVM持久化存储
2. PTC 0x10触发整机电源复位
3. 上电`FaultInjectSelftestInit`从NVM恢复注入配置
4. 上电自检执行CRC诊断，注入指令优先覆盖原生比对
5. PTC查询故障码，匹配预期完成FHTI验收闭环

### 4.3 工站层核心文件

| 文件路径 | 职责 |
| --- | --- |
| `ASW/SWC_PTC/ptc_cmd.c` | PTC 0x4F/0x10/0xFF命令入口分发 |
| `fault_ptc.c` | 诊断PTC子命令解析 |
| `fault_inject_handle.c` | 注入指令解析分发 |
| `fault_inject_selftest.c` | 自检注入逻辑、NVM读写持久化 |
| `mode_ctl_user.c` | 复位前NVM全部写入、整机掉电复位调度 |

## =={pink}五、配置层：CSV自动生成流程==

### =={pink}5.1 配置输入文件==

=={yellow}开发者需要手动维护两个CSV表格，分别是==

- `CATALOG.csv`：全系统=={yellow}诊断功能的==**=={yellow}总目录==，包括**全局故障等级、防抖、抑制参数
- `0x1E.csv`：本=={yellow}诊断项的==**=={yellow}专项明细表==，**专属注入阈值、FHTI时长、故障码映射

### =={pink}5.2 生成工具==

`diag_configAutoGen.py`=={yellow}自动脚本==，修改CSV后执行`update_diag_config.bat`=={yellow}生成==`diag_cfg.c`=={yellow}**三张核心配置表**：==

1. `g_diagTaskTable`：**诊断任务表**，绑定=={yellow}**Init/MainTask/GetKeyInfo函数**==
2. `g_eventTableCfg`：**故障事件表**，绑定=={yellow}**故障事件ID、防抖、故障等级FTL7**==
3. `g_diagItemTableCfg`：**诊断项配置表**，绑定=={yellow}**诊断ID、上电自检周期、故障抑制条件**==

### =={pink}5.3 编译加载流程==

**=={yellow}维护CSV → 运行生成脚本 → 得到diag_cfg.c三张配置表 → 编译进固件 →== `DiagInit`=={yellow}初始化加载==**

### =={pink}5.4 三张静态配置表==

1. `g_diagItemTableCfg`：诊断项配置表
2. `g_eventTableCfg`：故障事件配置表
3. `g_diagTaskTable`：诊断任务绑定表

可以把它们理解成：**=={yellow}诊断项表决定“怎么运行”==，=={yellow}事件表决定“怎么报故障”==，=={yellow}任务表决定“运行哪段代码”==。**

#### =={pink}1. g_diagItemTableCfg===={pink}：诊断项配置表==

##### =={green}它是什么==

=={yellow}一行代表一个完整的诊断功能，例如：==

- 入口电压诊断；
- Rx Start Signal诊断；
- Flash CRC自检；
- FPGA参数CRC自检。

##### =={green}负责什么==

它**回答四个问题**：

- =={yellow}这个==**=={yellow}诊断项是否存在==**=={yellow}？==
- **=={yellow}是否启用==**=={yellow}？==
- **=={yellow}多久执行一次（上电/周期）==**=={yellow}？==
- **=={yellow}抑制条件==**=={yellow}？==

因此，它主要=={yellow}描述诊断功能的==**=={yellow}调度属性和运行条件==**。

##### =={green}谁使用==

主要由=={yellow}**诊断框架**和**抑制模块**==使用：

- `DiagCfgTableInit()`：上电检查和加载配置；
- `DiagGetDiagItemPeriod()`：查询运行周期；
- `DiagInhiGetDiagItemState()`：检查当前是否被抑制；
- `DiagPeriodTask()`：判断这一周期是否应该执行该诊断。

##### =={green}怎么用==

=={yellow}每次5 ms诊断任务运行时，框架会查表==：

```markdown
if (诊断项已启用 && 当前没有被抑制 && 配置周期等于5ms)
{
    执行对应诊断任务;
}
```

=={yellow}它相当于诊断功能的==**=={yellow}运行许可证==**=={yellow}。==

---

#### =={pink}2. g_eventTableCfg===={pink}：故障事件配置表==

##### =={green}它是什么==

=={yellow}一行代表一个**可以向DEM上报的故障事件**。==

=={yellow}一个诊断项可以包含一个或多个故障事件==。例如，某个电压诊断项下面可能同时包含：

- 欠压事件；
- 过压事件；
- 信号无效事件。

典型字段包括：

```cpp
{
    EVTID_RX_START_SIGNAL_FAULT, // 故障事件ID
    DIAGID_RX_START_SIGNAL,      // 归属的诊断项
    FTL_LEVEL,                   // 故障等级
    FAIL_DEBOUNCE_COUNT,         // Fail防抖
    PASS_DEBOUNCE_COUNT          // Pass防抖
}
```

##### =={green}负责什么==

它回答：

- 这个=={yellow}故障事件属于哪个诊断项==？
- =={yellow}故障严重程度==是多少？
- =={yellow}**连续失败多少次**才**确认**故障==？
- =={yellow}**连续通过多少次**才**恢复**==？
- =={yellow}故障**确认后如何交给DEM处理**？==

它描述的是=={yellow}诊断结果的==**=={yellow}故障管理规则==**=={yellow}。==

##### =={green}谁使用==

主要由诊断事件处理模块和DEM接口使用：

- `DiagCfgTableInit()`：初始化事件运行状态；
- `DiagSendResultToDem()`：根据事件ID查配置；
- debounce模块：进行Fail/Pass防抖；
- DEM：存储DTC状态、冻结帧和故障记录；
- UDP故障外发、UDS读DTC等模块：读取最终故障状态。

##### =={green}怎么用==

=={yellow}**业务代码**一般**只负责给出一个原始判断**：==

```scss
DiagSendResultToDem(
    EVTID_RX_START_SIGNAL_FAULT,
    isFailed
);
```

之后=={yellow}**框架根据 g_eventTableCfg**== =={yellow}自动完成==：

```markdown
原始Pass/Fail
    ↓
查故障事件配置
    ↓
执行防抖/恢复
    ↓
更新事件状态
    ↓
上报DEM
    ↓
生成DTC、冻结帧或故障消息
```

因此业务模块不需要自己重复实现故障等级、防抖和DEM管理。

---

#### =={pink}3. g_diagTaskTable===={pink}：诊断任务绑定表==

##### =={green}它是什么==

这张表把一个抽象的=={yellow}**诊断ID与实际C函数绑定**==起来。

例如：

```markdown
{
    DIAGID_RX_START_SIGNAL,
    DiagRxStartSignalInit,
    DiagRxStartSignalMainTask,
    DiagRxStartSignalGetKeyInfo
}
```

**通常包含**：

- =={yellow}**诊断项ID**；==
- =={yellow}**初始化 Init 函数**；==
- =={yellow}**周期主任务 MainTask 函数**；==
- =={yellow}关键数据或**冻结帧获取 GetKeyInfo 函数**。==

##### =={green}负责什么==

它回答：

- 这个诊断功能=={yellow}初始化时调用哪个函数==？
- =={yellow}周期执行时调用哪个函数==？
- =={yellow}出现故障时从哪里获取关键数据==？

它相当于**诊断框架与具体业务代码**之间的**=={yellow}函数路由表==**。

##### =={green}谁使用==

主要由=={yellow}**调度框架使用**==：

- `DiagSelftestInit()`：调用上电自检类诊断的Init函数；
- `DiagSelftestTask()`：调用上电自检主任务；
- `DiagPeriodTask()`：调用周期诊断主任务；
- 故障记录模块：调用 `GetKeyInfo()` 获取冻结帧或关键数据。

##### =={green}怎么用==

**=={yellow}框架遍历任务表，通过函数指针调用业务代码==**：

```scss
for (i = 0; i < g_diagTaskNum; i++)
{
    if (周期匹配 && 诊断项未被抑制)
    {
        g_diagTaskTable[i].taskFunc();
    }
}
```

这样新增诊断功能时，只需要增加配置和实现函数，不必在统一调度框架里写大量：

```scss
if (diagId == ...)
{
    ...
}
else if (...)
{
    ...
}
```

---

#### 三张表怎么关联

![image](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAA5oAAAFoCAYAAAA2MISwAAAQAElEQVR4AeydB5wURdrG3wFBUDKfSJRFUUBAMeuJx5oxnGI+9VRQzDkhZjChYkLPCIp45ogRs3hiToigGFBUDIhIUpIC3/5Laq63d2Z2ZnfyPPyo6e7qiv/ume2n37eq6rRu3XqFghjoHtA9oHtA94DuAd0Dugd0D+ge0D2ge0D3QLrugTqmf3lIQE0SAREQAREQAREQAREQAREQgcIlIKFZuNdOLc82AdUnAiIgAiIgAiIgAiIgAiKQFAEJzaQwKZEIiEC+ElC7REAEREAEREAEREAE8o+AhGb+XRO1SAREQAQKnYDaLwIiIAIiIAIiUOIEJDRL/AZQ90VABERABEqFgPopAiIgAiIgAtkjIKGZPdaqSQREQAREQAREQAQqE9CRCIiACBQpAQnNIr2w6pYIiIAIiIAIiIAIiEDNCCiXCIhA7QlIaNaeoUoQAREQAREQAREQAREQARHILAGVXmAEJDQL7IKpuSIgAiIgAiIgAiIgAiIgAiKQHwTit0JCMz4bnREBERABERABERABERABERABEagBAQnNGkBLVxaVIwIiIAIiIAIiIAIiIAIiIALFSEBCsxivqvpUGwLKKwIiIAIiIAIiIAIiIAIiUEsCEpq1BKjsIiAC2SCgOkRABERABERABERABAqJgIRmIV0ttVUEREAE8omA2iICIiACIiACIiACcQhIaMYBo2gREAEREAERKEQCarMIiIAIiIAI5AMBCc18uApqgwiIgAiIgAiIQDETUN9EQAREoOQISGiW3CVXh0VABERABERABERABMzEQAREIJMEJDQzSVdli4AIiIAIiIAIiIAIiIAIJE9AKYuGgIRm0VxKdUQEREAEREAEREAEREAEREAE0k+gJiVKaNaEmvKIgAiIgAiIgAiIgAiIgAiIgAjEJSChGRdNuk6oHBEQAREQAREQAREQAREQAREoLQISmqV1vdVbT0BbERABERABERABERABERCBjBGQ0MwYWhUsAiKQKgGlFwEREAEREAEREAERKA4CEprFcR3VCxEQARHIFAGVKwIiIAIiIAIiIAIpE5DQTBmZMoiACIiACIhArgmofhEQAREQARHIbwISmvl9fdQ6ERABERABERCBQiGgdoqACIiACEQJSGhGUWhHBERABERABERABESg2AioPyIgArkhIKGZG+6qVQREQAREQAREQAREQARKlYD6XQIEJDRL4CKriyIgAiIgAiIgAiIgAiIgAiKQmEB6z0poppenShMBERABERABERABERABERCBkicgoZmmW0DFiIAIiIAIiIAIiIAIiIAIiIAI/EVAQvMvDvosTgLqlQiIgAiIgAiIgAiIgAiIQA4ISGjmALqqFIHSJqDei4AIiIAIiIAIiIAIFDsBCc1iv8LqnwiIgAgkQ0BpREAEREAEREAERCCNBCQ00whTRYmACIiACIhAOgmoLBEQAREQAREoVAISmoV65dRuERABERABERCBXBBQnSIgAiIgAkkQkNBMApKSiIAIiIAIiEC+EWjSpIm1a9fOunTpYhtssIFtuOGGCmKQ83uAe5F7sm3btsY9mr3vjWoSARHINwISmvl2RdQeERABERABEUhAoFmzZta9e3fr1KmTsb9s2TKbP3++zZ07V0EMcn4PcC9yTzZv3tzdo926dbOmTZsmuKN1qqgJqHMlTUBCs6QvvzovAiIgAiJQKATq1q1ra621lnXs2NGWLl1qv/76qxOYS5YsMR7sly9fbgpikOt7gHuRexLByT3KcVlZmbt3uYcL5fumdopAMRPIVt8kNLNFWvWIgAiIgAiIQA0J8ICOBRPLEA/vixcvrmFJyiYC2SWwaNEi91IE6zuCs04dPXpm9wqoNhHIHQF921Nir8QiIAIiIAIikH0CjMVs2LChc4vMfu2qUQRqT2DOnDm2+uqrW/v27WtfmEoQAREoCAISmgVxmdTIhAR0UgREQASKmABWTMa7zZ07t4h7qa6VAgHEZosWLTRmsxQutvooAhUEJDQrIOi/CIhA+gmoRBEQgfQQYPZO3A/TU5pKEYHcEli4cKGbLTm3rVDtIiAC2SAgoZkNyqpDBERABPKDgFpRYARYHqJ+/fomoVlgF07NjUuAe7levXrWuHHjuGl0QgREoDgISGgWx3VUL0RABERABAqWQPyGN2rUyP7888/4CXRGBAqQAPc0L1EKsOlqsgiIQAoEJDRTgKWkIiACIiACIpBNAlh9WCoim3WqrpUEtMkYAZbn4SVKxipQwSIgAnlBQEIzLy6DGiECIiACIiACVQngNov1p+oZxYhA4RLgnuberkkPlEcERKBwCEhoFs61UktFQAREQARKjEAkErEVK1aUWK/V3WInwD0diUSKvZul1D/1VQRiEpDQjIlFkSIgAiIgAiIgAiIgAiIgAiJQqARy324JzdxfA7VABERABERABERABERABERABIqKgIRmjMupKBEQAREQAREQAREQAREQAREQgZoTkNCsOTvlzC4B1SYCIiACIiACIiACIiACIlAgBCQ0C+RCqZkikJ8E1CoREAEREAEREAEREAERqEpAQrMqE8WIgAiIQGETUOtFIIsEdt11Vxs/frzdeOONrtZNN93Unn76aXvggQfcsT6qEjjvvPMcM9hVPfu/mGTT/S9H6ntHH320vf766/bOO+9Er2HqpSiHCIiACFQlIKFZlYliREAEREAERCDtBFRgegggYBGyCFpKZMuxF7rEZTIEhRniLBwQbaTJZBuSLZt20J5gGzkmnjJgt8cee9jEiRNt8803t+OPP55oBREQARFICwEJzbRgVCEiIAIiIAIiIAIQeO+992y33XazAw44gMN8Dym379Zbb7Wtt97aCTPE2ddff20E9gmcI03KBac5A4L80EMPtbvuuivaVto3bdq0aE2bbLKJrb766jZp0qRonHZEQAREIF0EJDTTRVLliIAIiIAIiIAIiEAeEEBkNmrUyE4++WQLi17EZzguD5ocaoIORUAEioGAhGYxXEX1QQREQAREQASSIIB7adCN8uqrr7bxgfGVSRRhjBsMlrH99ttXyoY7ZtiVlbGI1BPMRzmVMlYcpNo+yrjppptsjTXWsM0228yNM0RkVRTl/uMiiquor5fy3YmVH6QlUI5PQztpbzCOMihrZbZqN6Qljy+TfeLCGZs1a+bGs/p0cINfOF34mDb7PL69Pg31rLnmmvbEE08Y1mUfH97C4ogjjrDVVlvN2FIefSYd/adc4gi0a9SoUe5e4RxpFEqQgLosAikSkNBMEZiSi4AIiIAIiEAhEkBY9OrVy26//XbnSnncccdZ165dndBItj+Uscsuu6RcBq60L774oqsX901EEOUginzdlJ1q+y655BKjH7NmzbJ3333Xle9ddhFNCKhx48a5+CFDhlj37t2rTHjTqVMn69mzZzQN7Rk0aJDhVkrZtHfGjBnGWMZkRCD5cZ8NuqwyBvLAAw+0oEirV6+eDRgwwAlC6qAu8g4dOtTi1UM8og9rJenJN2XKFKO9vuwNNtjAfv/9d3v//fcpLm5gPCb3wsKFC6PXE56UQ3kzZ850TKjjzTffNMqNW5hOiIAI5IxAPlcsoZnPV0dtEwEREAEREIE0EEA8ILIQXd5tEmsXgu+PP/5IqobalIGoQcT4ip599lmbO3duVLzUpmxfZnCLINtqq62MsZO+3meeecYQu3CgPp8eoTVmzBh3SBqEGyKQNsKIEy+//LIby4j45Li6EHZPRRySZ+ONN2bjAnW8+uqrUddW6uJ6YOXs27evSxP+IJ7zpCM950ePHu2EJWKe4//7v/+z3377LaE1k3TxAuUgVIcPHx5NAkNYRiO0IwIiIAJJEChhoZkEHSURAREQAREQgSIggMBB2GD5C3bnu+++s2SFZm3LQNx5d0zv7urbUtuyfTl+iyBEkH388cc+ym19/zt06OCO+cByh8Bkn/Djjz86JrDhuKYBay0us7ieDqmwpuKiGiwL7r49Ph4rJAK8TZs2PqrSFssr50nnTyA4EZYITB9X0y0CvaysLKZQ/eWXX2parPKJgAiUKAEJzRK98HnbbTVMBERABEQgIwQQNrUVTzUtgzGFiC0sirhi4vYZFlk1LTsRLNxdEXo+4EqL4E6Up7bnEGtYMLFqevdZ+o7lNFh2TfvLeFSEuu8TW9x/g2XjWks7gnGp7EtUpkJLaUVABOIRkNCMR0bxIiACUQLaEQEREIGaEsCyx+Q0jAfEBbOm5aSaDyFHnQjbYGD8pHcfTrXMZNLj3sqSIZdeemnULTaZfKRp1aqVc9HFqspxrIALa7A/ft+PTcWKizUXq26s/IoTAREQgWwRkNDMFmnVIwIiIALpJaDS8pTAtttum3ctw3qIJQ8X1WDjcCElPhgXbz8dZfiyEUGIIX+czrIpE9dSxhnmywQ2cA+7znJMPO31gWOuBzx8XHCLiES044YcjA/uM7YU99rtttsuGJ3UvnfDxX02bBFNh2tuUo1QIhEQgaIhIKFZNJdSHREBERABEcg9ATMe8M8aPNjWWWedfGiOawMWPGZO3WGHHaIznyJWmAkVYeMSVfNR0zK8u64XfQgYXFqD9da0bJrsxVFQCBHHTKksecLss6QjYF3FnZX9TAVEIn1DNFIHnOHOfjgQz3ni2XLMdYIHceGAiERAM7kSHP15+kTfOKbvzOCLIGVMLOUS70MwrY8Lbpn4iJcAzIjr4ykv7J7rz2krAiIgAvEISGjGI6N4ERABERABEaghgfXXX98uuPBCO/CggwzRUcNi0poN10omvmG8IOP6WMLi3nvvtfDYwUSVhss499xz3Uyuicpgoh3GZiL6qJfxhR9++KEhyIJ1hctOpX2Io/bt21daRxM3XWZnRdS+88477hzjJpmgJ1hvuvcRiRMnTnTLodDfeIxgBhfOk47rwoy3cIjXJkTkhRX3FefhSD7CqquuWslNF+bl5eUWvN6kI7Rs2TLh0ie0HzHKUjOkJ1Afy8ewVRABERCBZAlIaCZLSulEQAREQAREIEUCjNe7cvhw27p37xRzZiY5IsaP6UOIYD2jpkRjAjkfDMEyGO+IoKMsrGykQwyxRIY/Jo40vl62WBnDaUgXLJsyY7WPNOSlHvIQEEe0hbI5TxwhXC9pSMs5AmkJ7PtAHupGrPk48hDH1sf5LfkJ/pgtfactBOr0ZbLlPFvKY8t50hHIx3kfOE+6YFvoN/0nvQ/h+n1+4n0avyUvZZCG/lA+W4594DhWu7Cm/vzzzz5ZrbcqQAREoLgJSGgW9/VV70RABERABHJMoEWLFnbUUUfZqaedZoyJzHFzKlWPeycWV6yLuF5i7cOCFQ7Mohp01axUSAYPgu3LYDUquhoCXHvGbbKMihep1WTR6cIloJaLQNoISGimDaUKEgEREAEREIH4BHBFvOTSS22fffaJnyiDZ3CHREz6KtjfZZddDDdPLFiEoBXLW8DYBq1gPn+61moNDQAAEABJREFUt9W1L931qbzYBLgvuBbBs2eeeaabDXfMmDHBaO2LgAhkjUBhViShWZjXTa0WAREQARHIcwL169c31jMMBqv4t8eee9oVV15pjFmsOMza/wULFhjrSHprJfvjxo2zsLtm1hoUqijf2xdqbtEeMnnTWmut5ca0+nuFe/iMM86woAtvJgB069bNkg1M/kTIRDtUpgiIQHoIFL3QTA8mlSICIiACIiACsQlss802duSRR9rVV1/twk477WRYL5kQiAf21q1bGwEXWl8Cx9ttv31WZ6ZFUGKdDAbGAPo2ZWLLTKUHH3ywbV/R1+rKT3f7qJPxh8F6WVKEWYFxBQ3Gx9pnEiGWYQmfw+pLCMdn4nj33Xe30047zbiP0lE+7a6u759++qlddtllFhzfmQ2LNgKzX79+1i+JMHDgQPdd4zuHlZXA/tlnn+2+i3wnKS8dzFSGCIhAzQlIaNacnXLWnIByioAIiEBBE+BB1otLHnrpzNixY23YsGH22muvOXdUXFI/+eQT+/LLL1349ttvSWZY7u4cPdquuPxymzZtmovL5cfee+9tjzzyiBMWvh1XVlhcmZE2USCNTx9vu/rqq9v+++/vLKnpEkvx6grGIyj79+9vJ554oiF2/TnE/+mnn27BpTv8ueAWMYYLKSJviy22sD59+kRPn3TSSUaIRmRwp1evXk6kt2rVKloL+/SLcbPMVBs9EWeHWW0fe+wx23LLLV27q+s7L0rOOeccYxunyIxEI3D5/iQTuIaHHXaYEdgnkG/ChAmubV27djW+l16A8l3lO+tO6kMERCBrBCQ0s4ZaFYlAvhNQ+0RABKojwMMqlpPevXvb1KlTnbDkYXfkyJFOYP7yyy+2aNGiuMWwDMdZgwbZK6+8EjdNtk+w1MjixYvtmGOOcdYg6u/YsaOxDAb7sQLnSOPPYSm74447LCxMESwNGjSwsrIyu+qqq6qcJw95fTnp2u63335u4qXvv//eiREEGQFRTXvWWGMN4zgYsB76+n26Rx991Pbcc087//zz3XIl/rzflpeX2y233OKu57HHHuujM7a9/vrrjTYxzvfrr7+2efPm2dChQy3YD8RxUNR36dLFlixZYpMmTYrbLsQnbrFs4ybK0xN853zgJQ/fRQLik4AA5TzfWb67WD35HsvimacXVM0qKgISmkV1OdUZERCBoiOgDuUFAR5MeUjlYXXUqFFOYPJQywNsMg3EqokFc8yddxpLRCSTJ1tpECwsN8ISJ1gfmSCIumnzQQcdZIS6devanDlz3D7HnCOND02bNrW1117bGjdu7KOi29mzZ9v06dPtzz//jMZlcgeRhTVu7ty5FolEDDHow9/+9jc3qU3btm0rxXPeCw9cRrH+cX2x9CJaEHSIOyyltB2hzfhWREybNm2c+CMt5zIZ1lxzTfv4449d20844QTr3LmzszzSfsIOO+xQyQK68847W7t27ax9+/ZufCVsNtxwQxs/frwLiEtYMZ6YvrHNZPuzXTbfTwIWXa4VAasnFs/Bgwc791u+29lul+oTgVIhIKFZKlda/RQBERABEUiZAJONYAEJCkxc/JItaOnSpXbfvffaxRddZLjRJpsv2+kQm//+97+dxREB5es/5JBDnLWMyWC8FTBo+fPp/Pbtt9+OilHEKQFh6sOrr75qML377rtdusMPP9xYUsXS+A+hiLUVoYiLaXl5uRNm5RVbJrRB+H700UfROOIJw4cPd+6l/fv3d2J02223tfEVggwrLcvSIOrg4ZtKX5hQCYvnDTfcYD///LM/VestlsUXXnjBEIqIWqzBvOig4OXLl7OJBlyyyyv6RqBf0RMVO9y3FRtDLOPq/MMPPxj3L/uE6667zrl5k6YUAqKT+wIeWDtxd4cRbPfaay93b5YCB/VRBLJFQEIzW6RVjwiIgAiIQEERwMLFA6h3keUBPZUOTHjtNRt05pn27LPPppItZ2nfeOMNGz16dKX6sYStt956hrULqyX7WPAqJQocYDFDmBEYS0hg3weEE2Uh5ohD3DJuMlBEbXYNl9e+ffs6S+Zvv/1muOXinktdBIRmkyZNnCWQYx98O9566y176aWXjPGPLPeCGCPPvvvu68bZeostYvXyyy+PvjxgTCdLggSFaG068tVXX9l///tf++KLL5wFnGszceLElIrEcglb7tsxY8Y4iyZWZV5+YMkkPP/882kVyCk1MMeJvejEykngBQgvlRjPyXc/x81T9SJQFAQkNIviMqoTIiACIiAC6SSAdQPXOsQEbncplR2J2DXXXOOsSLibppQ3y4lvvPHG6DIWWBaZACfYBB7AsUZircNdln2sQcE0+bKPWywWUqyof/zxh2sW4hjrJtfh888/NwQcQgtXWI4JjE9lwiAEMJkQYO+9957hessxZfTo0cMQazfddJMb7xiJRJz45jxh3XXXdS6qvgziahNerbD8Xnzxxa69tI9xmf/5z39SKrJevXrGeFRmGfbLlNBPltXxx2wfeOCBlMpNPXH+50B0cl9zv7PPd1+CM/+vm1qY/wQkNPP/GqmFIiACIiACWSSAVYMxXLjWYQ1KterzzzvPPpo4MdVsOUmPBXPIkCF22223VZnEiPF6Dz30kOE+iqUSCxn7xHEu3OBly5Y5CxxilIA4JbDvw3PPPWcLFy60O++807nOMs4wXS7F//rXv6xZs2bGhEuISd8+2sVEOPQTwYZwmzVrlnFMQHz6tGyxil5wwQU2aNCgaGBSI+4HLISIU3iMHz8+KtJPPfVU16/333+fIvIiwBWL/JCK6+sDa2TiXuuP2TLBUF40OA8agcjkxRLXmv2iFpx5wFtNKH4CEprFf43VQxEQAREQgSQIeNc5knrLBvvFHLDcYcFj7F547B9WwZtvvtlwH8W6xtqK7BPHuTAXBN38+fMrRcdyncXiWClRmg4QkQhY+lTbInGNxWW2vLzcjeVEnPkyf/31V3vwwQcNkeYDkykxXjMddft6/LZhw4bGy48RI0b4qKS2CHpEN9fXh7DrLPFYspMqsIQSITIlOEvogudRV4utKRKaxXZF1R8REAEREIGUCSAycZXjARORmXIBRZhhxYoVzh0U10sfNtlkE0NQci7YZSYLCgtI3D8JWAB9wLL44osvGhPYBPOnY59ymRE4XBbWV0QgbqK4vjKpUdB9dI899ghncUu7MAEPVksC1lyfiDU1WVsTkeYDLqpMGOTT1Ha7/fbbG31hRljccRGItD/VcnGFRkiSlxB2naVvu+66a6rFlkx6fg8kOEvmcqujGSBQJEIzA2RUpAiIgAiIQEkQYOIPXAyZ9IdxWiXR6QSdxC10lVVWcWMUWeZjq622MiYBwp24vLzc2IazN2/e3OrUqWMsv8GkOEyyw2ye5AuHnj17OpfUQw89NFxM2o9xffazjGJ9RGgySRAT/nDsA9cfV1PfAJagYQIehBgBi68/57dYa5nV9sknnzRca3EP9udqu4UngpglZ7CuMiPuPffcU6NiWdcV12j6GnSdZTmWWJbpGlVS5JkkOIv8Aqt7GSMgoZkxtCrYhEAEREAE8pwAIpNxWFiPsFzkeXOz0jwEFBY6BBkV+kmAsPCFXWM5T2AmV1xvmXSHYybQwXrGhDQcBwMiiqVCWrRoEY3GanfJJZcYgf3oiVruzJw50xDOLEHC5D5YXiORiJst1lsj2eJiiqupr44xnrjLYgUkMHmQP8eWtSlx1WVtzW+++ca4hxjXxznaTz8I7BOXanj44YeNNrNmps9LWZFIxHCl5Vo8/vjjMdct9en9luuCUKafWEbpG/vw8JZphBRjVdn6fNpWJQAffie41uxz3XlJwe9I1dSKEQERkNDUPSACJUZA3RUBEfiLAA+HPCgysyxWr79iS/uTpUv69+9vWDRnzJjhYGBZwxrGJDmIHBcZ+sCSySQ7b775pmGpvPTSSw1LHFZOlgnB2jdw4EAn8BB7rLeJlc0X06tXL2Pm1h122MGuuOKKSjO6Wi3+IR6pH3ffc88913bccUdDNH744YcJS0UMH3vssc7yyqRA3CvBDAhqxq2WV1h4mdAIMYpAJ02m+nLYYYdZ27ZtrXv37o4VVln6R52IWiyvuPgiJn0855IJWDeZsZdtMulLPQ0i0wvOCRMmGPe2BGep3xXqfywCEpqxqChOBERABLJLQLVlmQDCwYtM3CuzXH3eVYf4W3XVVY01KHFtpYHTp09nY1gncX8tKytzay7ykO1OrPzA0ta+fXsn4PxkOLidXnjhhYZrJqKTCYRYRoPyn332WUP0BcUQ6bkeXAvGTzLpEOWurKJWG8pmIiOsdwjpn376qdryEMnxJgMiM21nBl32CbChj8xWS32Z6AvWWSycQ4cOddZOZotFQFM/wplxsIxTRVgH3YA5HwyRSMStI8pLgEgkEjyl/RQJ8F1AcDKuW4IzRXhKXhIE6pREL9VJERABERABEVhJgDUysUBgyUTYrIyOsSmNKAQdljkmzcGyi5sl7qDsv/XWW3b33Xe7pUiwSmKtDFu9ttlmG2MypWnTpkWBbb311rbddtsZbpqRSMSYRAirJ8uKRCIR23LLLaNp/c7XX39tF110kVEO6RFttM2fr+l23333tbPOOsstfcJkRFhOGYt7//33G2MsN91005SKRtxFIpFKVteWLVsaYh1hTWGZ6AuuulhZcXuljmDgmhx11FGG5TnW+WBaBDdsBwwY4JZkYQma4Hntp05AgjN1ZspRGgQkNEvjOquXIiACIiACFQSYWZZJahiTKZFZAaTiPwKJCWMQK4zHxAWUmUorTtl//vMfF9gnHm64aG688cbGwzXxzMDKjLOTJ092ohS32GuuucZ22WUXThuT2CDur7vuOsPllBlVsQAhYp966qloOhIj0FgqhGtD+VgOia8UkjxA/CK6zjzzTJcDKykvGA488EBjsiLajHhmgiBENRZMlzDwgeX1lVdeMVxSaQuiDJfijh07GhxwmSXgNom78ZQpU6K5a9OXHj16GOM0d9ttN1uyZIkThNGCU9jBusnMtXAIZuOFATMJMw7UW6FxZaYvzNCLmGYiomAe7VdPgHtWFs7qOSlF6RCQ0Cyda62eioAIiEBJE2Atwq5duxoTeSBkShpGoPOIJ5gwwQzCBDfThx56KJDir13SsVzJE088YbjF4sLJGfIxRhDXWMQknLfddltn0UTQIOS+//57J+4YB8i5f/7zn24sJoJv3LhxFBMNCDTSUW40MoUdlk554YUXDFdG1tXEesf4UtpNMfTjhhtusP32288QvQhb2oHQ5vzEiRON8Y+kowzEMNZvxmpSNtvTTjvNWQ+xIPqAddTXQTmEmvYF0X7ttde6Org28VxhEetcE+pKJpCWPLHScu0S9SVWnmCc9v9HQILzfyy0V9oEJDRL+/qr9yIgAiJQEgQQP3SUh3a2CvEJILKw3sVKgYUTa+Xzzz/vxmuShrTkYR/rWHgGV1NoTp4AABAASURBVOLDAQGG5QeBFz5X22NEGsuBIKoefPBBC46lDJdN2+kL4ytpE+cRlvQRUUkZWCuJ8+dJg8UXK2EwxBODpK9JoA7aHqw3XA7XAyEfjo93TFryxDpP+31/2I+VRnGpEcgDwZlag5VaBNJMQEIzzUBVnAiIgAiIQH4R8CITd838aplaIwIiUAoEJDhL4Sqn0sfSSSuhWTrXWj0VAREQgZIiwBg9icySuuTqrAjkNQEJzry+PGpcBggUlNDMQP9VpAiIgAiIQBESQGQy8Q8PdrJkFuEFVpdEoIAJ8LuE6zi/TYwDZpIqXLRZdqmAu6Wmi0AVAhKaVZAoIkUCSi4CIiACeUWAhzUe2niAYxmLvGqcGiMCIiACKwnEE5wsGbQySUoblgVKKYMSi0CGCUhoZhiwiheB3BBQrSJQmgQQmYMHDzZmCWXJitKkoF6LgAgUEoGw4OzXr5/xsixVwbn//vsXUrfV1hIgIKFZAhdZXRQBEcgTAmpGRgkERaaWL8koahUuAiKQAQJBwTl27FhLRXAOPPJIa92mjUlsZuDCqMgaE5DQrDE6ZRQBERABEcgXArURmfnSh1jtWLZsmdWtWzfWKcWJQMES4J7+448/Crb9mW44ghOPDMZwJis4vfVzt913t3XWWSfTTVT5IpAUAQnNpDApkQiIgAiIQL4S2GuvvYzJNHCXLTZL5u+//2716tXLV/SZbJfKLmIC3NOLFi0q4h6mp2vJCk4mPgvWuP8BBwQPtS8COSOQVqFZv359a9KkiTHbX5sK871CGxOD/zFo3bq1tWzZ0ho3bqw39Dn7yqtiESguAjxg9e7d20aNGmXFJjK5UvPnz7dVV12VXQURyAMC6WkC9zT3dnpKK/5SqhOcvbfZphKErl272s59+1aK04EI5IJAWoQmwmHttde27t27W1lZmRNXLVq0sObNmyuIQfQeQGS2a9fOuFd69uxp7du3twYNGuTivledIiACRUCANTJ5oDr99NOLUmRyiX777Tf3Yq5OnbT8uaZIBRHIKQHuZQL3dk4bUoCVxxKcTz31lLWoeOYOd4exmmussUY4WscikFUCtfrLtcoqq1iHDh2ccODtFF+AOXPm2Ny5c403VQsWLDAFMfD3APcE98evv/7q7hFeRDCuCgGa1btelYmACBQ8AUQmnUBksi3WsHTpUps1a5Y1bdq0WLuofpUYAe7ln3/+2bi3S6zraesuz9t+DOeKilLxGFt//fUNI0/FofvPMzpi0x3oIycEVKlZjYUmbrIMNkYsIBz0Zkq3UyoEmOCCFxIELJuEVPIrrQiIQOkS8CKTiTJKgQIP5ZFIxBo2bFgK3VUfi5iAv4e5p4u4m1nr2p577mk8g3/55Zf2008/WVhwbr7FFrbllltmrT2qSATCBGokNJktrFOnTm6CAixU4UJrd6zcpUTgzz//dD+SvIVjPGsp9V19FQERSI0A4/9LTWRCCMvP119/bauttprGawJEoSAJ4Pm2+uqr2/Tp000zzqbnEv69Tx9XEL8RYcHZqFEjd46JgZh8yR3oQwSyTKBGQnPNNdd0f+zmzZuX5eaqupwRyHDFWDa5r3CpyXBVKl4ERKAACXiRictYqVgyg5eJoQeITeZE8A+QwfPaF4F8JsA9y7371VdfuaFV+dzWQmnb4YcfXqWpQcHpPQ0ZnoTYrJJYESKQBQIpC03eqLZq1cqNsctC+1RFiRBYsWKFLVy40E0QFIlESqTXte+mShCBUiDAWO6rr77aWE9u5MiRpdDlmH3k5e7nn39uy5cvd2Ox+HscM6EiRSBPCHCP4rHEPfvZZ58Z93CeNK3gm9GnvDxuHxCcfojSt99+a+3ato2bVidEIJMEUhaavBnB5QFhkMmGqezSI8CaWgxeZ4mc0uu9elxEBNSVNBJAZA4ePNguv/xyY/KLNBZdkEX9/vvvxgP7N9984wQnll4e5JkvoVmzZtZMQQxyfA9wL/KsyL2JwORe5Z7lZXJBfunysNEDBgywd99911566SUb+9hjdteYMfbvG26wyy67zAafdZYdf9xxdnhFmpNPOsnOP+88u/LKK/OwF2pSKRBISWgyHTU/HvyhKwU46mP2CSxZssQth5L9mlWjCIhAvhEIiszar5GZb72rXXsYbvDFF1/Y5MmTDXdErBYzZswwBTHI9T3AvTht2jR3b3KPcq/W7m5X7jCB0aNHO2GJwHysQmgiOBGen02daj/++KN5t9lwPh2LQLYJpCQ0GzRoYFgyMcdnu6GqrzQILF68WNP4l8alVi9FICGBbbbZxgavtGRKZMZHhYcRS0jxMM/kfAUX5swxtbm4GHAvck9yb8a/c3UmGwSYBIhnd2b7VWjoZu4udA5cT65rNu6fdNSRktAspI6lA47KyD4BXmJEIhE3o3H2a1eNIiAC+UBgr732sn79+hnushKZ+XBF1IZSI6D+Fi4BJl3q0KGD9ezZ03r06GFdunRRKDIG3bt3tw022MA6duxo+T7cLCWhiessFs3C/fqp5YVCgHutUNqqdoqACKSPwJFHHmm9e/e2UaNGmURm+riqJBEQgYInkLADWOrWWWcd69SpkyE2cZ9llm6WPZk9e7YpFAcDfz3xGmC5IK73uuuua+wnvEFydDIloZmjNqpaERABERCBEiDAGpldu3a1008/XSKzBK63uigCIpAeAiwNx5h21irFFR0Rwjrl6SldpSQmkJuzXF+uM8KTiTTXW2+9vJzjREIzN/eHahUBERABEQgQQGRyiMhkqyACIiACIlA9AUQmVi3W2kV4VJ9DKYqNANZrrj+utMxCnk/9y5nQzCcIaosIiIAIiEDuCHiROWzYsNw1QjWLgAiIQIERYJ1S3GURGszaX2DNV3PTSIC1UxGba621ljVq1CiNJdeuKAnN2vErttzqjwiIgAhkjQDr7ElkZg23KhIBESgyAm3btjVm65fILLILW8PuMNMz90O7du1qWEL6s0lopp+pShSBNBNQcSJQfAS8yGSyClkyi+/6qkciIAKZJcCEP0wAI3fZzHIutNJ///13YwkUXKrzoe0SmvlwFdQGERCBwiOgFteYAJNWXH311TZ27FgbOXJkjctRRhEQAREoVQLNmzc3LFil2n/1Oz4BLNwtW7aMnyCLZyQ0swhbVYmACIhAqRNAZA4ePNhYI/O1115LOw4VKAIiIAKlQKBZs2a2aNGiUuiq+pgiAe4LLJp169ZNMWf6kxeU0Nx0003t6aeftnfeeadSuPHGG6uQOfroo+3111+3Bx54oNK58847r1LecFnjx4+3XXfdNZqHsklDvmhkYIfz4TyB09Fd2kE5wRCvTJ+J88H0fp94n0ZbERABESgUAkGRqTUyC+WqpaWdKkQERCCNBOrXr2916tQxlrhIY7EqqkgILF++3AjcJ7nuUsEITcTVTTfdZNOnT7fNN988Gp544omYDDfYYAPnUrDmmmsaotMnuuSSS6J5jzvuOJs1a5a9++670bjy8nJ75plnXHKEbVlZmc2bN8+22mor49idSOGDuhG8ZAm2+/bbb7dddtmlihAmHfUgqHfYYQcbMmRItG3kp789e/YkmYIIiIAIFAyBbbbZxgavtGRKZBbMZVNDi5qAOleoBLBUrVixolCbr3ZniQD3SZaqiltNQQhNRCaiDHF2/PHHV+oMwjEch1BDIE6YMMEYFIvorJQpyYNNNtnE6tWrZ6+++qox4JrjJLO6ZFhGDzzwQJs4caIdcMABLs5/3HrrrXbppZcaQhirqI+n7UOHDjWmqg6KXn+e/obL8ue0FQEREIF8JLDXXntZv379DHdZicx8vEJqkwiIQN4QUENEoIgI5L3QRHhhTUSsIc6SYY8gRCC+8cYb9uabbxqik3KSyRtMg0CdM2eOIe5mzpxpHAfPV7e/2267uSRYJ91O6APL6ZQpUyq1r2/fvk7UjhkzJpQ6+UNvRfWutkHXXkStj2eLSy9saCP74VqII4TjdSwCIiACyRA48sgjrXfv3jZq1CiTyEyGmNKIgAiIgAjkGwG1p2YE8l5oIhqxJk6aNCnpHm633XaGQETIffDBB064IeCSLqAiIdbI7t2728cff1xxZG7LMfEuIokPpu9HoNKOeMnpF/2jn6TBLba6PKSLF7D+HnHEETZu3Lioyy2WXdJzjj54d1y28+fPt/fee8+5JGNdDfaPfeI8A8pQEAEREIFkCbBGZteuXe3000+XyEwWmtKJgAiIgAiIQJEQyLDQrD2lNdZYwxXy3XffuW11H2FxhMhDuCHgqssbPL/xxhu7Q4QqO37r44lLFLASNmrUyFgjLlG64Lma5Anmp++M62TMKVZYfw6BCQcYwIJ9zrHF2sA+ghcrcLB/ft/3nXQKIiACIpAMAUQm6RCZbBVEQAREQAREQARKi0DeC81UL0cscYRFDsscQizZ8mKJMkQa8cmWka50YVdY3FwRpeHyO3To4KIQjW4n9AGHTp062Y0xZul9//33be7cuRbsH/u49iJIQ0XpUAREQATiEvAic9iwYXHT6IQIiIAIiEBhEFhrrbVs//33N54hk2lxq1atDO9CtvHSr7baarbzzjtXKXP33Xc3/ob06NEjXtZq42nvaaedZpRVbeIMJ6CfDKVLxCLDTchp8QUhNLG0eRGViBbii/GcXFTcQhmDSNhjjz2MOC50ovz+HMKuffv27uYnvw98wYjnvE8bb4s7KhP64D4bL004Pl4exqZuvfXWzhUWa2U4nz+uzvqLlZNZazfbbDO3xEtw7CV1M6OvF+SIchYDjidafZ3apo+AShKBQifA7x0PCPRDIhMKCiIgAiKQ/wSOOeYYt3zgIYccErOxe++9t5100km2zz77xDy/00472UMPPWRs119/fbdKA94sPJcj+nwmnsUp49///rc99dRTdtFFFxled0ER1qtXLydAyUdAMPJMnyiwioSvgy3lbb/99kZZHKcSaCOTblKGz8c+fePZOF6grz59cDto0CA755xz7MQTT3TRtDWVvrhMBfyR90ITt80//vgjqYl4GOfYrFkzQ0yxFEgwfP3111ZWVuZu/uquF5P+UCc3QrAMjonnfHVlcB63WS/cOI4VKIuZcbEocj6ZPKSraUBs0idm8EU0B62j7FMuVmFEOX317SJeQQRKkIC6nCQBLzL5DZPITBKakomACIhAHhB47rnn7OeffzaGU51wwgmVWvS3v/3NdtxxR2OpDFZD4LhSgoqDVVZZxRl0EFs8ZzZr1qwi1lzcddddZ6ymcMcdd9hLL71kZ555pvE8joHjrLPOMuYVoW6XIfSBwEMwMiyMuuMFnmvJyovOe++91xB3GEu22GIL45iAuN13332NCTK9AclviUNAUgazpCO8MfKw4gVx9AuxOGTIELfs4JAY2wEDBpC0Ujj33HONtn/44Yfm/y726dPHqCte4HylQgr8IO+FJm6buG8yiQ0XJRFvRBviCHEaTofbKDc+YjRJ/CM1AAAQAElEQVR8LnjMzcQXADdZ6g6e45h4zpMueC7WvhduiLZY57GM8raFmXH5wpGmujykiRd8vxGK8dL4eL5ATBjERER8kYn3/YMRfZw+fbqbKIhzCiIgAiIQj0C3bt3s6quvtrFjx9rIkSPjJUtjvIoSAREQARFIFwGMMVgXp02b5kQlz4GUjScfwpM1O6+88kqi3ORuscQmJ3luZMszpN/WqVPHMFpgeOFZHC9DXFoRgwg80lUXPvroIysvL48b+vfvX6kIhC8RS5cuZWMtW7Z04hYPSSIQvF4sPvLII4Z2IJ5wzz33mO8r4vKUU04xno/LK+pH0MYLwaUWsYqynBf9RGTS14ULF1K8W+4wXhnEY011CYvkI++FJpy5eAg8bgpM7MT5wDFLdiD8uLFJxw3hz/stIowbCTHq42Jt+XIhSPkyxDpPPOdJF+t8MI523HfffYaYDLqpkgaReeihh7o1Nnn7QxwhmAfRSb+I9wGrgd9HePMl9WWTF1HOGxjK9+kon7Q8ALL18YzBxJoafJNE/3gLxJeR+n1abUVABEQgFgFE5uDBg40/qq+99lqsJIorFQLqpwiIQMESQGxiYURYIQx5zr3iiisMgYg18rHHHjOetxs0aGAXXnhhTDfauXPnGqsZNG3a1HHAjZZyf/jhB3fMc3jwmdNFVnzwnIzrLc+0jNts2LCh0Zbzzz/feB4lYJHEMhkrYImtKMZZDQ866CDjmR9hd9NNNxnHX375pfG8O2/ePJK5fZ6ZCbQZIe1OrPzAEIPL7quvvmoIxZXRSW/QJttss43RH/qBVfOqq64yrLNJF1IkCQtCaMIahY9LLG9CvKmbLaKKcYQsX8IYRYQS6cOBmwkRVp1llMHLfBG4ScNlcPzss8+6SXNIxzGBNxdDKszotMcHRBoiEcvhySefbMxA68+xRWTeddddhoimjGAgD2MyGePJl4T0PuDuCgdvAQ3mY5/y+ILgiuDzbLTRRs4lokmTJhZsJ23ixyJYlu+3Xx6GMhVEQAREIBaBoMjUGpmxCClOBHJPQC0QgWQJIAIRhjyj4uqJYBwxYoTx7Mvz5aqrruqsfTyfnnHGGYb469SpU7T4Zs2aOc+WL774wrBk4rH3yiuvmPeciyYM7fz6669GWoTZt99+6yyMPMMyL8myZcuM8NVXX9nnn39u33//vbVt29YtY8gx4ccff4yWyLhOBC7DOCiTExhplixZwm7SAQ4IRjIgeHl+ri4gTqkfw84FF1zgJjVCqCN2u3TpYl7oUmaphIIRmlwQLHOYlYMBQYYw8+fYkjZW4EtSXl7uTOCcR2Dh1ko8xwQEbTANccHg85COePIG2+P3KZe0pGHLsT/H1reb8/ECdZA2GIL5EM+0lXTBMmAQzEPdtIF0seKDeeOVGUyjfREQARHgbe3glZZMiUzdDyIgAiKQEoG8SIxBBMMIoo6A0GPiH54tEW+4jr711luGwNpvv/2sdevWhpUPkUn6n376yRBkvjMYfchz1FFHOXdVLI2INcokTefOnaNjJr1lErGK4eOaa65xIpaxoLi+YoFE4C5fvtwInKcdCFdcYjEycUxgYiHKJxx++OGG2KPdWDXpI+UjojlPYNgYHn4ExHEkEiHa5UMsUiaB/fXWW8/+/ve/R912sbjSTxiFAxOStmjRwjHBPRdDFJMitWvXzs0fw1hX3+94W4xQTJjkGlQEHwUlNIuAt7ogAiIgAgVPgMkS+vXrZ7jLSmQW/OVUB0RABEqUwCeffOLG1yOqEEZgWLx4sSHWDjvsMEMg3XLLLYYbLZNIIoKwTiIu8dbDsEEeH+6//363QsLrr79uuKO+/fbbTqBRhk/DtkGDBm5lB7Yc+8ALTLwTOWZSHIQc+wQEI8IQayZitFmF9ZRjPAwRdASssMwOO3HiRENoIqLxDMSjD4/HP//80xCfCELGTRLwjMSTkXMtWrQwzlEv7q64uuI6i0AkjvLJj8jlGBEJO/YJiHHqpr2059JLL3WTkMIDKyfxsUKbNm2cMI91rtDjJDQL/Qqq/SIgAiKQRQK8oe7du7eNGjXKJDKzCF5ViYAIiECaCSCaXn75ZefphwUxWPwpp5ziZojFTZbZU//zn/+42VyZFwTX2mDa4D5Cj2FqiDeGVyDm/Hm863AjZZZbXFnZMtkQgpc0/G3BPRexywRAiEAEG+eY1RVRh7W0cePGbowox4g9XGXpC1bY559/3rUToXnZZZc5ayzzlmy77bZuvCbiFFEYDMSRD5GIWOQc9VNvMNA3RC6iHMHN30Nm0Q1PjsQ5JhRCtJIfhmwRqvQ/HLC2Mo6UeNgyQRHpiyGkRWgWAwj1Ib8JMKNlTQPTXRP4QfABiwxvzvgRzO+eq3UikD8E+B517drVzTookZk/10UtEQEREIF0E3j00Ufd+o8IHyx4Q4cOdUt14DqLZTNWfYzLRKgxzwfjEf/73/8az1oIN9IvWrSITczQo0cPY5LKb775xp3HjReXWAQkEX6oGuJy7ty5hoWV4WB+eBhprr/+eicsfR4siQwJQ8gywRCBdDUJiMwNN9zQmO8FsY04hFH9+vXtwAMPrFQk4zOxAiN2Z8+eXelcqR3kpdDcf//9jbVu8uVi4OtNm4IDnhO1jXSY/P1bmFhpedvBlyNeGvzYKSeclzhM+eH4GMdFE8Ugbt4CpRqwuBDGjh1rEyZMsKlTpxqDwwmUyZsz3P/GjBnjXEd4iEaI8qMoAVo0t486kiYCfD8oirfHbBVEQAREQASKlwCT8jCT+JZbbmm33XabmzGVySYReocccogxZhIxFSSAu2nHjh0NkUg8FkwspkzYw3GiwLhEZpelXtIxDpP62U82MCsuY0djBZ6fCbHOEcdY0nj18KzO8yLnH3/8cTYuYAXl+RKRzDwoLrLig/GXMLv77rsrjkr7f94JTQYMY+L+17/+Zbw5r+7ycHF5AOLGTyYcd9xxVYo85phjDD9uvjhVTlZE0KaTTjrJmekrDqv8x5zPTYX/OAKRtx6nnHKKYcpHpPoMiEvqYJrohx9+2PwsVv683yIkGRxNf7i5fTxbxClvlRhszXE4kB5/ddwAkgmkJU+wHESybxttqC7Qp2D+dO8jDGsSsLj4wI8Vgem5CfjKe+HKgzP7/FggRrnv+EHxFlSsn+kTnummo/JEIPME+I2lFr4nbBVEQAREQASKmwAi8rrrrjOW5WA8JK6gjDnETRRRxVhJZosNUsDtlWcsb5XkHHkY/8h4TNxliQsHnjspl5UPwi68pEWA8neI51FcYLEislwhxwTWqySdt3pi6fSBZ7kFCxa4CYWYVAhrK2NL/Xm/JY4yYgV0CfUx+y3P/AhMxDTPlTyXM7nQnnvu6SYTIv8bb7xho0ePZrfkQ14JTW4yBiAzgJgplRF3iLNEVwkhxxsUbgBmhkoU8JXmZg6Xh484JnCsWfiKB8/TJgYB45NdXl5uHAfPs88NjzkeccKCt5HIX7NXsa4nA6CZtQp/a5YlQejSN2bN4gsR9F2nLAK+38y0tc4661Ra/oQ3MXzBWCuU9TlJGw6IW4QTX7xkAmnJEyxn7bXXrjTDFv2OF/r27Wu87QrmL7R9L2L5wSAERSjWUKyfAwcOdFZP7hEsnsQVWj/V3gQEdComAe5z/rhzUiITCgoiIAIiUNwEsEg++OCDzliCOMSCx7M5z7CMmcTQgfDDwMKkQEEaTz75pN18883BKLePKGWHWWrZhsOrr75qCDfqCp/jGAHLs1d5xXP4ZpttRpQxDpRjQjxjwLHHHmsYkz777DPDUvrdd985z7ZTTz3VOOcKSuIDoYxIRrCiUSZPnmy0FwY8a/OsyPKDiM4kiiupJHkjNBGAgwcPdvAvvvhiN/0xNw6LxSKw3Ik4H7xhQCwyiDZRYKBtrCL4oiAQp02bZohK3uKQjnoRntxUvMkhDmEWS2xyDusZA5RbtmzJofuSclOy9g9xL7zwgnM9wD2A9StZENcljPGB3zdfCtbAZNYvkhx88MFuViq+7LSZuHiBtYf8W5p4W9LEys9bGDiUV3yhEwUWoQ2/zYpVXqHGIUC98OS686CNxROXW96QEfjhK9T+qd0ikIiAF5l8D7j3E6XN1DmVKwIiIAIikB0CGFQwmjDbLG6rrIWJSBs+fLhrABY/JgXiAEsnz6Ls+4AY7N+/v3t+xyuRMZpsGc+JMQixNn36dJ+8ypa/M7jaVjlREcFkOnjplcd5LvVtrEjq/mMAoV7qZ9ZYlmPhWR6LJi6yjLNk7ClpSOsyBT722GMPQzhiycWgwnhQvA0Z1ofOwICEwERUMwYUF1lciRmKFShGuxUE8kJoIrx4Q8JNjn81bzawKuFiypsVLh5vUSraG/N/JBIxbgZuwkQBy2PMAioiEW4IJ1xeEYCITUQuC63SDlwBaBtvdxCJtLkiW6X/vKlhADIWy0gkYghlbnC+sCTkbQjn2Y8XmL2KtyTc/LwBItxzzz1GHG9K+CJjtucYoROvnNrGI6x5U5MoIL7gzqDs2tZXCPl54EZ48mNI33mD5UUnlk6udyH0Q20UgeoIcC/z+8I9zm9xdel1vqQIqLMiIAJFRgADCpZCZorFKsnMrkxkw5qRuM5i4SwvLzeePbFuMlYziABRiXEEMRmM9/sYbnBZ5fnax6WyZVhaoudRhpvhnVde0Ub2aTPP77feeqshCoPP3jzvY0RCJJIG0czkPghPhpLxPEceRCmTCSEwOY8xjKFxYU9Lhs0xjI9nfYaSMfwqlb5RZyTylydkKvkKJW1OhSYXizckXFDeVuDHHbx5mUqZtxSY6BGiiDAuSBgu5xF+vF1IFJgtKpw3eIz7LDcgNxtiAsE4YsQIw40VN1dEHgKMtvJ2hJuZL48vg3V6EIXMMMXMW8yWxbo9sdrs84S31MFCtVgbx48fb+MDAastgXOkQfSG8/tjRHUi0c050vj0tdkmmkWsNuXmc96g6ORe4ZgfIR7OGdOZz21X20QgEQFEJvcya2TyYiVRWp0TARHIFwJqhwjUnACGEITitddeazyLr7vuusaQLzwMN954Y0N0nnzyyYY1kGflcE0IMgQo1r5wYE4SnoNZOiQo+MJlxDomPc/lPFfHOh+M45mYeUf4G8YQNEQfRptgmuA+50hD+/BGRHQ2atTIDR3jOR/9wfA2ho/MmjXLGAeKZuHZnsmDfLjppptsyJAhLiBEaXOwnlj78Lj//vuNMnjOx6gWa2xqrLyFFpcToQlgbkiEJG8KEE74SjN4NgwQ4clDD1MTIyYxT2PSDqbjLQJm7XguosF4BjD7vLyFYBIgLjSBi83EP+Xl5cZaPIhbvlxDKm4gpmtu3bq1YW1FZJIeCyZfAF8eApUvIe0kLV9KrJ+MDSXNFlts4VwKgm9lEK6cCwbeCDGGk3pjBc6RJpgnvI+ojpU3GEeacD5/zJcr/GMRPuZawB7XZZ+vFLeITCze3srJ2ywvOHE/LEUm6nNhEuAPBKi5YgAAEABJREFUNL+3iEyGAhRmL9RqERABEcgTAgXSDCyNzCGCuKTJH330kSG0+HuAeGNLGs6lGnCxxUMQgebzMgyJCXXY+ji/RZBiVEH48uyP9TH8/Bk+Jg0TCbGsCLqCZ++wIOb5neDrYUsarJ8YX+gvx7QVge01CToAzYIHG4yCz9HBfcZ98vxPuymbQHmIXvrEsQ+IUfQHTLCoImYpy58vpm1OhGavXr0Mqw/qnQuIyR7YiLdYAddVzNKsx4OVkRuMh3kuRIcOHQyfcsz7QQGXaB9LJCZ2bmAEAReXG4HyFi9ebNxQ+KgzNpLJfHCjRcjy9gMrLOISEcmbDfL4wFsgRCqDjhFf3ESIXF+2T8ebk7KyMsMq6ePSvUW8U3eiQJpwvViNEdzJBAQ3b5BwayY9LMPlldIxghMLEBZOAn3nxwM3DB7gOVYQgXwlwHhjHiYkMvP1CqldIiACIpA9AjyfszRJOmpEVDKBji8LwXfZZZc5442P81v0AM9QwfT+XG22WB4JlJEoUO/bb78dMwljSBmTGStg1Q1nQnQy/I8+hc9hbGN+GPQKnBGf4TTFcJwToclbgiOOOMIYpMv4H4SeD4gVTOS8TfFxbLkY+FNjNcRyFHwLgh91dRY+f7EQqri7YlXlonJxuWEQvT4NW8ZqIrownzP4mRsCKx9+2rSBNLEClk9mwsUNgQc3LLWkQ3i++OKLhkjmBsYSeOeddxplcj4YcGn1A6q5AcOBc6QJ5knXPuIYK+sNN9zg3max7wPXheCPeSAlIMKJizcTbrraVkjlIDqxcvJjyT4P8NzbEpyFdBVLp628+OvXr5/xfZYls3Suu3oqAiIgAiIgApkkkILQTG8zMCcj9LAqIvR8YOZXhCNCzMexxYyOJfH77783gm9NixYt3No4iDZEXHUh0YxXvky2zPqK6yyictKkSTZ06FBjGRXewmDZJE044Nv9j3/8w02h7Ac944rLciEITayl4TzhY5YuoY1B4cz6PMF0nCPNjBkzgtGV9hGiuAIkCqSplKnigOtBWxkIzssArK7wJ1AvrqBYZDnmgZTySYf1lzc9FUXof4AAIhPBycuRCRMmmF8mhQf7QDLtikDOCGBxxyVo1KhRxnc6Zw1RxSIgAiIgAiIgAkVFIGdC01PEuojbK4KFwNqRTKSDCOLYB9KQ1ufzW5YNQbAiXJkVNpbrLeZsFlv1eZjiGNHkj2NtMXfjBsm0x7fddptbloTxolhXGTyMKRyXWp83Eok4Cy3tZqZZ4jGVU8YPP/zAYVIBCy9imb4Q6A9iev78+YZllXMEBMtDDz1kFqdUxl/S1kSBNLGyI+pxaYATFlfGnWIBJi1uynvuuadzfYbBeuutZ1gyaTfnFWIT8IITCyczeeL6jYVTgjM2L8VmhwCu3dyLvAiRyMwOc9UiAiIgAiIgAqVCIOdCkxmaGLzrBREPPIhHRJCPY0sa0oYvDMt/MK2yt6YxMxSDdf3YRNw6ma7Z50PE4iYbHpjrz/stIpIBugwSbtasmeEayhhMxpMyoRD1hteQRJhhWaV8ymE2Vlx+11hjDWPpFgQx8fECbra45m699dZ20kknuUAbELoIOupnrCr5EX9M24zFlONwYPylZxBvS5pwPn+M5Zb2INLpB2sHwQHBxDhVloLB8syAa1yafT5t/yIQ7xN+vHxAcBJ4yPeCE2txvHyKF4F0E0BkUia/uWwVREAEREAEREAERCCdBHIuNBFLCCsvhhCJiEVEkI9jSxrSBjvPzFKIHlxqg/GJ9rEOJjrPFMOsF8RkQUx08/jjj7vJgZg1i0lvcKdlORXGTQbrRWTSPlxuw+VTJ+ers2wibqkTt9tgGYwPpWwEK2NAOYcFETdbXHu9+CQ+nYGJlmg363diEcaFd8GCBcZU0Lg2s8+YzXTWWUplIToRmwREJg/+uDFqHGfG7gIVvJIA9xq73HtsFURABERABERABEQg3QRyLjRr0yEWl2UMIRMH+XKwhmKFHD9+vI2vCAcffHB0dlcm6SE9D/g+vd9iccQtlNlmEVFffPGFHXPMMW5CHNIwwyyuq+xjZUR4su8Da2gyCysCtE+fPoYVkPVBmTGX9YiwulKmTx/eetdg0jBOMnweSydjIbEgIsaxquKCyXhJXGnD6XHh9W7H8bakCefDRRbxSn247uI6zORFCE0/NpaZeBG/zJx7/fXXG/0mX7gsHSdHgPuRFwc89LM/ePBgk+BMjp1SpUbAv9AgF/cb29wHtUAEREAEREAERCCdBCKRSDqLq3FZOROaWAxjCaB4YzR9Wi/IWAMTKydWQqZg9gRwW2VWW0QmgVllGXPILLVYinD/nDt3rk/utrjDIlpxsX3yySeNiXAok7oQrVg4y8vLjfJY+oSxmi7jyg/EASIMV9KVUdENwpaJclhzk3Gf0ROBHUQa4x6xHjIFtD8ViUSMc/6Y/BzTd9bmZApmxlVhhe3cubNP5rZh1+MhQ4bYkFAgjUsc+MBiiqDFsnrPPffY/vvvb1g/qCuQzBBGCCImJCIP41F5cMXCHEyn/eQJcB/5iYPYZxwubrXMXpx8KUopArEJeJHJvcV3NXYqxYrASgLaiIAI5C0BnjcjkfwQEnkLqcQbhqbgPsk1hpwJTdwxmWgmHBgXiGURK2D4HMdYz8rLy40ZXBE5iEgEGCARhyzPgcXNiyrWzEGEMWss4xqx4n3++eckjwaWIqGMa6+91hgHSd0sWsuCrRtvvLFRLutmkp9Jh6IZV+6QFisglsVwYIwjrra4Aq9MXmXD2E3aSfBCk/YgIBG3TAjkA8fdu3c3+oDlkzGgWL++/PLLSuVi5fUM4m1JUylTxcGYMWOMa4DwZYmTbbfd1rnKIoCwCMOqIpn7Dxf6zUPrlClT7M0337RYfFxifSRNACEAb7hitWZGUASnJg5KGqEShgjwko17iPuJl0Sh0zoUAREoEAJqpghAAMMIW+YdYasgAmEC3Bv+Pgmfy+ZxzoTm8OHDDcGYathxxx1t9OjRhoUNQYQw8sBw52QmVESaj0P4sH4lAg1rJZPYBPOQ7v333zfcURGMHCPAaB8Wu+22287YkoZzqQbGVtImLKs+LxMRBQUy8fQnOL6TNTwHDBhgsUTi8ccfb5wnH/1DqLJPQHzyQEm9LEGSKJCGtOQhrw/BY8QjFmKYwA3XXZ/Ob3l4PfHEEy3sTuzPa1szAghOP3EQjLt27WpcLwnOmvEs1VyITH7DWCOT+6lUOajfIiACIpAhAlkvFksVqxFgmMl65aow7wlwX6ANSlpoZvMqIUARbMwci2hKpm6sdUFxmEyeeGlwKw2KX46xViEu4+UhHsEXSyj6GXZJEw7cWLQ7URqfhzSkJY+PC29pN0IZfliKgxMghdPqOHMEEAjcMwRcIL3gZD9ztarkQicQFJm42Rd6f9R+ERABERCBvwgwGeOqq67614E+YxAo3SiGvzGxaj4QyJlFMx86rzaIQKERwMqJ6yOCk7YzfhbXaQQFxwoi4AkwttdbMiUyPRVtRUAERKA4CGDRZPJK5u4ojh6pF+kg4F8+cH+ko7zallFFaNa2QOUXARHIPAEEpx/HyT6CAisn4iLztauGfCeAe3W/fv0Md1mJzHy/WmqfCIiACKROgMlemIgS6xXj8VIvQTmKjUAkEjGWIsQb8c8//8yL7klo5sVlqLYRSiACMQkgMhGcp59+uk2YMMEQFwhOhEbMDIosegJYuJlAatSoUSaRWfSXWx0UAREoYQJYrVjVgBUVJDZL+EZY2fUWLVoYz4Xh1TVWns7JRkIzJ9hVaXEQyJ9e8MOC4MSlNl0TB7F8UP70UC1JhgCu1Izb5cWDRGYyxJRGBERABAqbwMyZMw2x2bx5c/Nuk4XdI7U+VQKsqIHIZIJQrNyp5s9kegnNTNJV2SKQZQIIzuDEQcGZahEgqTTnwIMOsqZNm6aSJT/SlmgrEJl0nZcNbBVEQAREQARKgwBic/r06cZ4TQnO0rjm9BKBiTW7cePGhrvsDz/8QHRehZSEJv7gedV6NaZoCTB1d9F2LksdQ3QiOghUiRDBrTKZiYNOPOkk44fr4IMPJqtCHhPgBQLXlib6a81+vgW1RwREQAREIHMEcJdktQLEBgIECxcB4anQ3IqJQcuWLY3A+FxeMkyZMsXmzJmTuZurFiWnJDQZWBqJRGpRnbKKQGICjDGIRCK2bNmyxAl1NmkCCE7vVsv+4MGDrTrBuemmm7ryt9hyS+vTp4/b10f+EUBkci1pmUQmFBRSJKDkIiACRUSA53T+ziM4p06daixJh6Xru+++M4XiYMD1/Oqrr4zri8DEXTYf1suM9zVKSWguWbLEvBCIV6DiRaA2BHg789tvv5ksmrWhGDsvf3wQnIzfY3/gwIHGxEHhmWr7DxhQqYCDDjrIvTmrFKmDnBNAZHL9+GMjkZnzy1GQDVhllVWM31wW91ZoaPnDID/awr1Rt25d07/CJMAzO5MFYenE2qUwx1n9Cp0D15Prunjx4oK4MVMSmijmefPmualzC6J3amTBEWAg++zZswuu3YXUYEQmghNxwsRBzFCKYPEz1W677baVutOgYUOTC20lJDk/wP2Za8bMslzLnDdIDSgYAvXq1XMvjrp06WI9e/Y0tuutt54piEH4HuDe2HDDDd09gpse907B3OiZaGgelbn//vtb37597W9bb209evSwjh07OtdQXh7lUTPVFBGwlIQmvHhIRQywryAC6STAHzHeni5YsCCdxaqsOAT4LvuJgxCcTBz0yKOPWuvWravk2GTTTW277barEq+I7BNAZOL+zBqZXL/st0A1FioBxEL37t2tXbt2zmuE34Bff/3VFMQg3j3APcL8HNwzCBruoUK9/4up3Q8++KAxYd/RRx9tZw4aZBddfLFdN2KE3X7HHXbTzTfb5VdcYeece64x38Jhhx1me++9dzF1v1JfdJDfBFIWmphrsWo2adIkv3um1hUcAe4pfM+xnBdc4wu8wQgWLJx8t5lEYP3113eCk33ftYP/9S9bc801/aG2OSAQFJlaviQHF6BAq8TK0alTJ+vQoYPxNxzXsUJxuypQ5EXTbETmokWLnMshfx/at29v3EvcU0XTyQLtyMSJE2O2fPXVV7c2bdo4SzTzLWy3/fbuux8zsSJFIDMEoqWmLDTJyYxW/MgwjTLHCiJQWwLMBsYDkNxma0uydvmxKCP2v/zyS1dQ586dba211jIEJ9/5gzQLreOSiw/G0g4ePNiwZEpk5uIKFGadzKtQVlbmZpHm91Uv8grzOuZDq7l3sHwyIzn3FPdWPrSrVNtw7TXXxOw6f6/xTGrUqFH0PBbQ6IF2RCCLBGokNBlgPG3aNLdeT/BGzmK7i7OqEu0VIpO368yIVqII8qLbV111VbQdS5cutZ9++skQnOz773mvXr1sp513jqbTTnYIMH62X79+EpnZwV1UtbRt29b9rcaKWVQdU2dyRoB7Cey6k6EAABAASURBVEMDVrOcNUIVOwIfffSR2/LhBSYeSRwzsSJbROaPP/7IroIIZJ1AjYQmreQG/uyzzwy3CoQCNzjxCiKQLAH+UDHeA3ccpmrmbWmyebOZrlTqWqNVqypdRWQiOHmL7U8yC23bdu38obYZJsDyJUzYxMQ/smRmGHaRFY/lidmJEQZF1jV1J8cEuKe4t7jHctyUkq7+mquvdh5HWDC9wMSllr/bgMEo9PRTT7GrIAI5IVBjoUlrFy5caJ9//rnxpgSfcEQD4+wQEEwYpLCqiUFlBtwbvJjgXuEeYo2nb775xlj7iWOF3BC4cMiQmBWznilrNCFwJkyYYI8//rjdOXq0tWzRImb6LEaWRFW4y/Iwx5I0XIOS6LQ6mTYCjKnGWyRtBaogEQgQ4EUk91ggSrtZJMDfBl5EMm6WaoMCk2PCgw88wEZBBHJGoFZCk1az3uGsWbNs8uTJhlUKyweWKcZ6KdQ1MajMALdr3rR98cUXhkUcayb3kUJuCcydM8deeP55u/++++zGG2+0iy66yE45+WQ7fMAAO/OMM+zyYcNs5G232aOPPGLjx4+3jz/+OLcNLpHa/SRNhdNdtTRfCDRo0MBwef/999/zpUlqR5ERwLMNiyYv1Iusa3ndHS8wWeKKWYG36d3bDXUJN/r5556zqVOnhqN1LAJZJVAnXbUhOJnMBesmghMRofCZE1Pi8D8O06dPNyxkevhJ1zcvPeWMGDHC7r77bhs3bpy98/bbNu3LL90sg+kpXaWIgAhkm0DDhg3dEiauXn2IQIYI4PWCp1KGilexAQJeYJ599tmGwGTZkscee8ylCL/8xQDE2Ex3Uh8ikEMCaROaOeyDqhaBoiXATK+4ozOhBzPAdunSxU1Zrq04lNI9sO6667rp+XG713wAyf3cITQ1JCE5VrlKVQz1co9xrxVDX/K1D2GByVAKLzB9m68aPtzvui0iE+9Cd6APEcghAQnNHMJX1SIQjwAu14x96dGjh1uzrFmzZsZU8vxRV/jTjekVh9LhwPcEFz3WgezevbsTnXLXg0r8gCDH2hQ/hc6IQO0J4M1Wr1692heUPyXkTUuSEZjBxjKEjeO33nrLeSaxryACuSYgoZnrK6D6RSBEgHFVXbt2NYQmY1gZ97xgwQJbtGiRMfmCwlJxWFpaDJjQhu8AM13yfeDFCzMssg19fXQoAiKQRQKsPBCJRLJYY/FXlarA9ESGX3mlYcUszgmAfC+1LTQCEpqFdsXU3qImgJssboJYInioxmpX1B1W50QgRQJYUObOnWvMCdCxY0fjoSzFIpRcBERABPKOAL9lzCLrx2DGcpGtrtGIzNmzZ1eXTOdFID0EkihFQjMJSEoiAtkgwDiXddZZx5goiZCNOlWHCBQqASz7WPyZ2l+WzUK9imq3CIhAOgSmp/j888/7XW1FIC8ISGhm/zKoRhGISYAHZpZ/wUU2ZgJFioAIVCKA5Z8lFsrKyowxiZVO6kAEREAE8phAOgVmHndTTStxAhKaJX4DqPueQG63TZs2NaaIZxxablui2kWgsAjwcgZ3Wh7aCqvlaq0IiEApEuC3qrYusqXITX0uTAISmoV53dTqIiPQqlUrY8KTIutW7bujEkQgCQKM1+Q7xHJASSRXEhEQARHIOgEJzKwjV4V5QEBCMw8ugppQ2gRw+Vt99dVt4cKFpQ1CvS8YAvnWUFxomf0Sr4B8a5vaE5/Apptuak8//bQ98MAD0UTsE8e5aGSe79BW2nzjjTdmrKVwIVRXAWkI1aXT+ewRkMDMHmvVlH8EJDTz75qoRSVGoEGDBsZDcol1W90VgbQSQGyWuNBMK89iKwzx9c4771i8wPlM9fm8886LWy/tGT9+vO26666Zql7l5ogAApMZZAm//PKL1WQW2Rw1XdWKQNoISGimDaUKEoGaEcDdT0KzZuyUSwQ8AZYC4qWNP9a2MAkccMABtttuu9l7772X1g5Q7uabb26E22+/3XmQsOWYwPm0VhgtzOySSy5x9VLPcccdZ7NmzbJ33303GldeXm7PPPNMIId2C5lAUGBOnTpVArOQL6baXmsCEpq1RqgCRKB2BCKRSO0KUG4REAF5BegeEAERSJ5ABlJKYGYAqooseAISmgV/CdUBERABERABEcgsAcYf4ubpw9VXX224fBKfbM1HH320vf7661E30mOOOaZKVlxYCcETHPt62XIcPM8+7eCcD6m2D9fV8ePHR9tGObi8UrYP4TpitcOnJS9lMHaTMZw+Pt6WskjvA8ex0oYZ0qZY6YJx1E87fNnsExdMo/2aE5DArDm7cE4dFx8BCc3iu6bqkQiIgAiIgAikjQBiplevXuZdTXH/7Nq1q1uSKdlKEF5HHHGEjRs3Luoy2rhxY1tjjTUSFoGwYgkb3E4JQ4YMsTXXXNNok8/Ifm3bh7vuiy++GG3bE088YbvssotRP/XQ/u7duxv1+3Yw2zHnwoE85P3666+TcgMmfXV9pA763bdvXzv55JNdO7ke9Jv+cz5WQEBfddVVxnqztJvA/tChQ01iMxax5OMkMJNnpZQFTaBWjZfQrBU+ZRYBERABERCB4iWAUEFgIRBvvfVW11HGTyLE/vjjD3dc3QeCZquttnLjEhmv6NOPGTPGjZX0x7G21HnooYdGTzGWccqUKVZWVuaEUjraR+HHH3+8G0vJPuHZZ5+1uXPn2gYbbMCh9ezZ02bOnBkdS0k7WAvRnQx80J4DDzzQZsyYYcmO+6yuj4HibeTIkdHxq+SbOHGicX2oN5jO7yOg2Yc1WwL7zHSOaOVYITUCEpip8VLq0iYgoZmp669yRUAEREAERKDACWy88cZWr149N4FNsCvfffedJSs0N9lkE2vWrJn9+OOPwSLs559/tt9//71SXLwDXEm96+dmm20WTZaO9vnCEGveffamm26qZG39+OOPrVOnTpUsqT6f36666qqGYKVPw4cP99FJb+P10RdAuTDzx2wnTZrExjp06OC2wQ8EflmFIA8KZM5TBmW1adOGQ4UkCXTr1s2YQZagSX6ShKZkJU9AQrPkb4HSAqDeioAIiIAIpEYAQYmwTC1X5dSUwWyrlWOrP8JlFYFJStw+CczYyrEPlF3b9iHyhgwZYt59FvfgYHuxxGLFReTSHtL7+v0W6yeCmnRYfX18ddtk+kgZuLymUi55CAhk2uxDWESTRiE2AayXe+21lzHmd+DAgSaBGZuTYkUgHgEJzXhkFC8CIpAtAqpHBERABKoQwCLnXW6TdUOtUkgSEYyRZPwjYx4RlPGycA6hS7r27dtbeFIdBDAus7jOYh2NV04wvrZ99GNcEwlt2kW7wwHra7At2v8fgaDA7N27t40dO1bLlPwPj/ZEIGkCEppJo1JCEShcAkwWgUuYf/jhDTqzP/KAVbi9Sq7l9NW/yWcfS0T4ATG5kkotlfqbbQKHHHKIde7cOdvVJqwPqx6us7ioBhPiqkl8MC7evhdBWPyCaVq1amWMFQzGVbePMMMd1KdLR/t8WcGtd/cNxvl9xkYyZpW20wcfzxaXWdxSBw0aZP73lvhUQriPPi9iOFwmY0epD3dYn85vsX5Onz7d4EWZPl7b+AQQmIy9xT2WVKeffroTmK+99hqHCiIgAikSkNBMEZiSi4AIFA4BHsp22GEHw5WNt/lYJNLRegQ6Qh3RWpvyKIcXAGxrUw4PkYhnL6j9ljjO1aZs5c0ugfMvuMAOOvhgq1+/fuyKsxyLqMJKx/eI7xPVs8Vql6zQZOIcJvBhhlR/r3NfYlFbbbXVKDJmQCjhLhoUSgMGDKg0djId7QsLYdq2xx57uLGpvmFMwkO//XE8gUebebFHOvpHWezHC6Svro8+L6wOO+wwf2i8OMMt9s0334xOEBQ9uXKH3wAE8ZlnnrkyxtwkSvx2BfsTPVmCO4hL7x6LwPzll1+cuHzssceM/RJEoi6LQNoISGimDaUKEoHCIYDg2nrrrY2HtMJpdeot9VaYDz74IJoZFzxmYuQBLxqZ4g6WGcaFYWHwD84pFpG25DxsMuYKywVi2gfGmC1dutSwzKStMhWUFQI777yzXTl8uOGyl5UKq6mE7wwTyjCGkZcYWOvuvffeameMDRaL6GKGVJY4oQzu2Q8//LDKJEPBPOz7GVJJTz5EAa6gnPOhtu1DCDM204+/pK5w25o0aeKWNqENhEaNGtmFF14YFXi+LWwp78orr3TW2hEjRkSXSOFcrJBMH8nHcikIH+onsIQKbrz8nnM+VvBt4beKPAT6Rzmci5Wn2OO4h5jYB3EJe8QlfZZ7LBQURCC9BOqktziVJgIiIALFTQALBRaWCRMmuBkzEZ256jEi0z9s8iAfbAdCmgepYn+ZEOxzMe03b97cjjzqKDvttNNsrbXWynnXEHP+JUZ5eblbvoNGhWeSJS5e4B71ZbDl/uWlD2X7POwT/DFiiPpIT+Ac5ZCPe9ynI57zBNJjheVcrPbxnSANW9IQEGvk9cG3jbo4HyyfNMH6aQfHPi3pfbvDL/QSpaVcAnVRFmWSnvKIIxBPGkK47GA69n3wbSGPD5TjzxfxNto1xCW/h4hKP7EPJy+//PKo9VLusRBREIH0EpDQTC9PlSYCOSeAEMJdijfXBPaZdj/YMB6icNkMuk7h7kV6H8hHWeF8/jzbUaNGuQkxcMMKpqtun/Tk94G6fR7aRNv8Oba0159nS34C8Zz3gWPOEziP+xvuZlhicHXF+kg8gTQ+hOuk7/SNdnDOp2OLhRCXwTfeeMNwWUN0hjnRH8oIxrNPHOcohzZg3aF9bOlDsP20lTYTT2CfOPISKI+JUrASBR+YOZcoUC/lEcL9o37ijqoQOGxJQyBPvDJph+8X+UnvA8fk82lilQMP8pOGtAqVCWzYq5ddfMkltu+++1Y+keMjvAX4HjBGku9I8H7x158t8ZzPdnOD7ct23aovtwQQld5iyXhLhCUCs2vXrm7WWNyPGXuJa+ynn36a28aWZO3qdCkRkNAspautvhY9AR7orrrqKmPMj39zjRiqzupGPv44425JPrbACo7rQTQg3Px4R9I1bty40ngp8iQKiAlEBW5n1EEZiMA5c+a4bNTBMW5snCNQH/VyziVa+cHYJMZJkYaAW1lwHBlv/8m7cOFCo8xYb/8pir7jCohrIOUQEjHbbrvtjPZiJcAll/FPffv2paiUAu3D7Y32saVerCoUgvg69NBD7a677jLiCQhK4rzYRPBSt19Hj3yJQiz2jJuj7zDweRG+Bx10kOH6R70whHWYv0/vt+utt54Frwf5/HXDKoNrL+57wbpoU1lZmXGONL4sbasS+MceexjutFyTqmczG8N96O87amIfSzr3JC85+C5gIaRt4UA858mXqVBd+zJVr8rNLQH+ZnlB6a2V3hWWpUhoHS7CvDREWA4bNswQl8QriIAIhAhk6FBCM0NgVawI5IIArlbMQMjMh75+xAsizB/H2vIgiPDDkJWRAAAQAElEQVTxD/tsEVteGCAIsJ4xNoryfBn8UUco+ePqtkzkgThCSFEH6akbEeProK3BOtinXuonDXkI1Ev97BNefvllN3kHlgyOkw2pMEMkweTjjz92xdN2BCoCy0Wk4YM6unfvbv4h3heJqxvugAhd4pJZ1oB0PiCGw2v8jR492rn/wsCnY4vQp2/sP/vss24cXXV9ZDxorPvOXzcEMRaw4PWhTdwPvHygrnSHffbZx8ZUiHXCgRXiOd/DZptvbu3atasUmECHABvuveNPOMFOOPFEa926NVFZCQsWLDBvdcdKyT6zrnJPZqUB1VSS7+2rpvlFdzoo/hCANQlYIglYIn3AMslvvg8cIygRnEBkjKW3ViIsEZUEWS2hoyACuSEgoVk77sotAnlDABGGdQhrphdxvnG81fX7ibYIQB4kCVijfFqsZ82aNbPweCem1EfY+nTVbXkgQJh5ERNM7+vwIi54jnqpnzQ+PlwOM0cyQU+bNm18kmq3qTLzIglLpi+c9iIAEIg+rjZb6kCQIczC5XAdq6sL91Sunw+IeMpBKM6dO9fef/99Dl3gPuF+4bq4iIoPBHywfz4NVmh4VSSJ+Z9ySBs8CRuEJEtAYPlCKNMOn4b98HX059K1xbJKQAjne1j255+2fPnySgE+YRbc539WpA3HZ+oYQRm2VPICKFP1MSaVsam77757lSoQ3VjYgyey3b5g3Yn26cf2229veAn4dOzzsmj99df3USlvmSwKEZZyxgLKwG8dgbHwCEiCt0wiJn1AUDIjsARlAV1cNbWkCEholtTlLpXOlnY/+eOcKgFc4RgDiFD1Lq08nAfL4eGW8VjBuFT2ESmIlUTti1cH9XIulfpSSZuoTb4c2o91jgdFXHG9kEOQExe2Cvp8NdnSV4RzOC+C28fBhH3WM2TrA5ZpRAFtRDT6eLZYQZlx0redbfihnXQ1CckwRHh6oYwwZ5+4mtSXTJ5HHnnEHnn44YIJCHyucThg3aa/CPUR111nt95yS14uu4B7+h133GHMSJso3HnnnYa1mT4FA66NzNLatm1bQ6AhKv/2t78ZIuPUU091SfmtYgIXd1DNB99LyruughkvO6pJnvbTDFk455xz3IROvnAEJuJo7733dlEc813wARFKu93JOB877bSTHXzwwcZ3PU6SnEZjQUT4pSMwQQ/l+ZDM70xOO6/KRUAEKhGQ0KyEQwciUJoEeLjhIRaxFLZKpY1IEgVhyUMMhZMSx7lwfDaPsaZiVUWAI+SCAXdfRDpiNB1toq9hAUm5QWutF6I8zHIumUA7g+32+5l6YKW9WLyxfNM+hBRbrLYE9n0c+wrxCTz26KN2boVoKWReTEq29tprG/d206ZNq3R26tSpxoQtiC9/EqtmnTp1DMGBhRDB6O8nnybelhdbK1assC222MIuuOCCSpbFeHmIp/5///vf1QrmoJhGYCO0ye8DQpn6cf33ceEtgpO2MVZ68ODBdvHFFxvWOtLx0o8XRuHAuVVWWcXIGz7HMflIoyACIiACuSYgoZnrK6D6RSBNBBCIuC/GEjxB18hkq8Ot0aeNJ2p46MM10qdLtKV906dPt1jtIx8unbh2Ik44DgbiOEeaYHxt92lTsswQdFgaYz3oY5VDhCJGaRMWKbjAh2MC+8SxnyhQPvUgrsPpuI64UuJ6TGAyH8ZzYg0Jpw0f00YsiOG04XQ1PQ5fV0Q3cfCFM+XSZtrPvUWg/cRxTiE2Ae55BAhWvdgp8icWr4jDDz/cmEwqGBhHx33dokUL++abb+zcc881hFmw5Yg7rFW8DGnZsqUhLhGj9evXN8aDNmnSxIlQJiD79ttvg1nj7iNITznlFHvuuedso402MsRc3MSBE1gUO3Xq5JaVwbpaXUAAI6Bpry+GMnr06GFfffWVqx/R+dBDDxmTtdE/3F+598kze/ZsO+OMM+z555+3+fPn20cffeSK6dOnj5GOCZWCgd8ZXKf5Pgfj2Sc9+VwB+hABERCBHBOQ0MzxBVD1IpBOAkyIg+AZMGBAtFjGXfLQFI2IscMDHg8tXoQwri+YhwciRAEPS7iuUQRCgrFRPFBxnExg0hfEVnA2W+pkvBdihAmINttsM6N+Xx771IslkTQ+Pl3bZJjRV0QTIgkW4bp5iEYcIkY5xzFbLMRsCVgpwqy8gA+KSsqHNbN6etbk5zpyjZgIg2MC/GkTVgw4EecDViMso/6YSX2wLpKH/vj48IydPj7elus1fvx4YyxoMA19CN537HMvwjeYzgte+hJrHGowbQnvGyIJN+frR4ywH3/4Ie9RMEsxIdxQ7pf77rvPuJ/5/nNfvPXWW4awxGrIRC/kwTp34YUXGpbu/fbbzxBjCKa///3vRh7cTRFu/H5wHrdvH7gfqYdyYgXa9eGHHxqeG3wPY6WJFYfgQ7xVF0gXzv+Pf/zDENYsg4SFkd+wm2++2QjMWs3YQ1x6mciIvFgo6R/ClDzEERDV1dUfPE968imIgAiIQD4QkNDMh6ugNohAmggw4QrCgYca/xCGFSyR6xZV+9lHhwwZYuRjLCJv1znnAwKFsWLMOEkaxj699NJLbkZSn6a6LSKKN/e4tFEGgToXL17ssiI4WeqDh1LOEViy5NJLLzX65hKl+YNyw8yoIsiM2VERUogkzoUD/UIceusix8zcimimD4QvvviiCivSkY9xnqTxYhHWWHE8a84hdGFHnmD9uL2GmZGevLhDw5T0iHQe5NlHwJCGgDsjDIivTYAX9xplEmAR67p5wYvoxVpXmzqLNe9TTz1lZw0aZG9XCLJC6WPHjh2NcOyxxxqzgfoXVbzw4AULvxdY8/guYZ1FHJGmffv2rovcp7hycz9/+eWXLm7ZsmWGO+1JJ51kvLRZZ511bMmSJcb9y+8GgoyXZDfccINxf7tMMT4Yqzxv3jxr0KCB8V2j3hjJkohKPgljTGkbYwuxMtI3BPa0adPcZE/c//zGMvkTpcIOay3fI44VREAERKAYCEhoFsNVVB9EIEAA0cBYIR5sCDy4IVx4sPMihYe64DEPaTzIkZ7APqInmIYqKIfzBOr473//S3RKE5OE66Is2uMKqvgItz/chookbhIM+sW+D/SNtLTRx1EucZzzceQj+GO24Tp9GTwMYlminHA7yRcM5AnW5fOQjwBPuJIunI/zBPL4c+wT5wN5YefPB7fh9vs84X6Sn3L8ebbBNNQZ7IOvgzTkIz8sSUOcP++3xFEmgTSk9efCW9yoKS8cX+rHF15wgT304INOjBQii++//96wzDHu0As6xini6skxk9gg/D755JMq3dtyyy3tiiuusObNmzsX0hdeeMFxYJKgE0880RCavBghcG/xwgoxx6RPfE+rFLgyAqGLeyuiDjfYfffdd+WZzGx4ycMSH7jFXnbZZUY7EZW479IH3IKxzGJh9S3AwotXRPA78eqrrxqB3w5e3lQX8HogPcGXWxJbdVIERCBvCUho5u2lUcNEIP8JMFYI90jGJNJa3CnjPQzxsESaQgi4lmJBDI4vLIR2F0Ib+/btazxk40ZZCO3NZhv/85//GAI8m3Wmuy5c3HnxgbA76qijXPG4hbI+KO6vjLnEZZbvlju58oOxlKzDihi8/vrrneWS5WhwO2XSndatW7t1cnE1pWyEGV4G3v18ZTExN9tss40hMPEe+OGHH4yXZIjhmInTEIlrLO365ptvrG7duoZVFisullkY4BaMxZN9rKxUiSv5okWL3Lq2HBOw3BIeffRRw4LrA8Iasc7Wx7HFM4X0BPIriEAuCahuEYCAhCYUFERABKolwNvyoFhkTNSBBx5oQffMoEULq1YwYC2rtpIcJGAcJK6zwaoZQ4oYGjNmTDBa+7UkgIDHLZsHfixStSxO2fOUAMLooosuMsZOIvL4LrHuIxPm4D6Ly2u46YxZvP/++43JhJg4C1fzV155xRBUvLzCQogVdPLkybbeeusZwwOwFCYzJrF3797ObZZxmsxei4s37vnhNqTrmP4zGRITmDEsgEmAKJvfR4QzVl/c3fES4Dzn+D5geWU8KsfBgKCmv3hYkI5yOc+WYx+C1lDOK4iACIhAiEDWDyU0s45cFYpAYRLAasn4Jh76CLxBRzDw8FSYPfqr1VgesJDQJx94gI01HvKvHPqsCQGs3VhasGSF3YdrUp7y5C8BBBMvmVhDknGHXHPGcyM4+/XrF3OZEdxh8ZAYNWqUm7UWYcV9gjWTcZ/Es34mQhOXVCYJwuIZb9y0p4Plkkm6cLFF+D7++OPGBFqZtmqyTigM6DeusrgF0yZENBzYDwYsrQhrxnWzTmbwHGuJ7rjjjs4TIBivfREQARHIdwISmslcIaURAREwLJI8OAUDD4KFjgZrQHl5uQX7haVB1oHkriyc4FXdvcALCRizTa5kpSpEAryMuvvuu61v376GuyjfLyb1wZKIZY9ZZRmHiTttrP4hTHnR48dcIlq7dOnixCmWQO43XEwRorwkimUdDZaL4MMiiMUUkcryKbQFq+b+++8fTJq2fSYwwzJLP5hNl0mxDj30ULcuZ//+/V1fmCAIEc1LLl8xs27DasMNN/RRbksarJmIURdR8cGM0owF9S/H/DbodVKRTP9FQAREIKcEJDRzil+V14aA8oqACIiACOQPAS9+mNSGtTKvvfbaSo1jbCbusYjEPffcs9I51uC8+OKLDSslLrIIMsrAtRT3WAQrGRCLzORKOiyAxMULiDwsn3hjPPzww9FkuLZSJlZN0kRPpGkHCybCkMnSeEHHeEzcdineC2wmR+I4GBCZkUjEsO76eMaiMr60YcOGxjhOHw/j8BhNvEzom0+jrQiIgAjkmkCdXDdA9YuACBQVAXVGBESgRAkgfpjghnGGzK7K5EaM5WZCHEQUWFhHkuVPnnvuOQ4rBaydCDHGY15zzTXGWGnWokRQMVaTxLiV4mLLpEKM/8TiSXw44G6KVTESiThLIpZMnwaRSZmRSMSNCSWtP5eOLTPMYi0dNGiQIYb9LLeM20QkM7aS2Wg5pi0IU3jRDgQl53078AJo1qyZG2OKOA/2l3QI8GBApPu82oqACIhArglIaOb6Cqh+ERABEcg4AVUgAtkh8OWXX7oJfJg9F2vkZ599ZrjSBoUlLrWM2wy3CPGHmGLW2nXXXdeYPIrJxr766iuXFCGGi/aSJUuM8nAxRcy5k4EP0iF4WSaFMZKUGzjtdonD4ogLLTPeIl7dicAH5TOus7pAukA25xqLyy7jSp999lkbOHCglZWVuWVfGDPKbLQTJ06MZkGI41qLdZW++hmZEZXbbrutISixyPbs2dOwEgctntFCtCMCIiACeUhAQjMPL4qaVFoEWGOOUFq9Vm9FIL0EIpGIFdz3KL0Icl4aYwm9KGNcImIRi9xhhx1mPt5vmX2WZT6CjUbsMeEPYw9xjcUNlGVROEaI4RqKxZMZsBnniTsqFs5hw4Y5cUdZWDFxwWU5zuTidwAAEABJREFUFCx9tIP4WOHKK6+0d9991zp27GiUgYUxmI4xkb69ibakC+Zjhl2ELmNDGZeKRRM3YKycTZo0MQRuMD2W4AsvvNDKy8udKPXWV0R0586djfGlCEwmMsKVltm+segy+y5jYrEahwPpgnVoXwREQARyQUBCMxfUVacIBAgwVgfXqUCUdkVABFIkwPhALF0pZlPyNBF49dVXnZUxkSALnmMSH8Yx+uqxKo4YMcI22mgje+qppwxxiug85JBD7KeffrJjjjnGfv31V7vgggts3LhxzmqKEHv77beNJXMQX4guP9nULbfcYolEJvXijnvWWWcZa39GIhHz7r2cI8yaNcsQt9UFJiWiLH7LyXffffcZM+Vinb3hhhuMiY0Yc4qQ/OCDD4wZmEmXKNB/rJmIaYQ1aRHnuBMzIRLsGGPK5D/h9tEnLKfkURABERCBXBKQ0MwlfdUtAhUEmEExEolU7Om/CIhATQnwsoaH/ZrmV77aEWDpmrDgSXSMEMRF9K233nIV33bbbXbCCScYs7EiqBBnnGD74IMPGuM6sVa+//77RLvA9T711FONtSlZlxLBiGstrqqMD3WJqvmgDATpwQcfbE8++aRLjZBLZiZll7jiA+slFkXGZlYcOmH50UcfsRsNd955pw0dOtTOP//8aBw7uNDi3ks/OfaBNXyxYtJv2ujj4XXiiScay51sueWWlWbLxnpM6NOnj8HM5ymSrbohAiJQgATqFGCb1WQRKCoCuE3xIMF4nKLqmDojAlkigDslQhMrT5aqVDW1JMAkOEz44wUhv4GIrljF4krKmErSxDofnACHMaKkj5UuURxCL175ifKlcg5BGa4D6y0c4BEui7GsNelLuBwdi0DmCKhkEUhMQEIzMR+dFYGsEJg5c6abVTArlakSESgyAqy9iHtl2PWxyLqp7oiACIiACIhA9QTyKIWEZh5dDDWldAnMmzfPsMbwwFy6FNRzEUidQIMGDSwSidgvv/ySemblEAEREAEREAERyBgBCc3/odWeCOSUwPfff29MWy8X2pxeBlVeQARwmV199dWNpTRwQS+gpqupIiACIiACIlD0BCQ0i/4SF3oHS6f9TArE+CKEZqNGjUqn4+qpCNSAAC9lWCqCGT/xCKhBEcoiAiIgAiIgAiKQQQISmhmEq6JFIFUCCxYssM8//9y5ArLYOGulpVpGVtKrEhHIEQEm/eG7gZs5lszZs2fnqCWqVgREQAREQAREIBEBCc1EdHROBHJAgLGaU6dONRYBx7LZokULa9q0qVuQHCuOwqrOxVgcqnIoViYNGzY0hCUCk+/Dr7/+alOmTDFZMnPwA6UqRSBAIBKJ2IoVKwIx2hUBERCB/xGQ0PwfC+2JQN4QWL58uZvc5OOPPzamt8dqwxg0rDkKdUwMSosB3wdEJUtATJ482WbMmGFLly7Nm+9rNQ3JyWn41K1bNyd1q9LSIcBv8Z9//lk6HVZPRUAEUiIgoZkSLiUWgewS4AF7/vz59uOPP9pXX33l3GpxrVX4XCw+Lx0GjF1msqy5c+caL1yy+y0szNoY812vXr3CbHxWWq1K0kGACbnwwklHWSpDBESg+AikJDT5QcGFr23btrbeeutZ165dFcQgLfdAly5dbO2117Y111zTcBctvq9afvfojDPPtAGHH2577Lmn9e7d27p162atWrUyWURyf93+7//+L/eNUAsKjsDChQud5b/gGq4GFxQB/kbwUiNtjVZBIiACRUUgKaHJDwkCoEePHtapUydr1qyZ88nnzbLCH+4NuzjUjgOuN7x9R9yss846TuhwnxXVty2PO8MYuPLycttnn33syKOOssFnn23Dr7rK7hg92q4bMcIuuOACO/744+2fBx5oO+60k228ySZ53Jviado222xjZ1dcC4nN4rmm2erJkiVLDG8IvbjLFvHSq4d7C5d27rXS631p9Vi9FYGaEqhWaLJGWdcKqx1Ckx8UxooxM+bixYvdGBnGgSgsFYultWfAG3hc45joA+HZsWNHKysrMwRoTW9w5UuOwMUXXRQ3IROwrNO5s22+xRa2yy672IEVYvOH77+Pm14n0kfgtddeswkTJkhspg9pSZX0888/GzNXRyKRkuq3Opt5ApFIxBo0aGDcY5mvTTWIgAjEIFAQUQmFJmuU4dK4bNkymzNnjvHwXxC9UiMLngAvMhCc3INYOCU2M39Jf/rppyqV8JDaunVrY6ZPf/Keu++2WGn9eW3TS+Cxxx6T2Ewv0pIpjZfCs2bNMl4WlUyn1dGsEMDjaObMmfbbb79lpT5VIgIiUJgE4gpNXOl4wOdHpKgGehfmdSrZVvOCA5GJZTMS0Vv5TN4IZw0aFC3eC8z111/fxfE7wM4H779vL730ErsKWSQgsZlF2EVWFROJITglNovswuawO4hM/iZwb+WwGapaBESgAAjEFZrt2rUz/O41yLsArmIRNDFRF3CnXW211WyNNdYw/cssAcYar7XWWta5c2dX0cSJE531Evd4fg/uueceF6+P7BMIik3Gbma/BaqxEAmwxuE333zjxmvimcBLpELsh9qcewLcO9xDjP3lnuLeyn2r1AIREIF8JhBTaOKuyNhM3oLmc+PVttIhwB82Xn5g3SydXmevpwiXq6++2po3a+bGG3/yyScIzEoNwGX2l19+qRSng+wSQGyOGjXK+vXrZ1yz7Nau2gqVAMNfpk+fbt999501btzYsG7y8o41EAu1T2p3dghwj3CvIDCZ/Id7CJHJPZWdFqgWERCBQiYQU2gy8ydj5Aq5Y2p7cRFgfDB/2HgJUlw9y21v9tprL0NgIlyYdIZZZ2ONv3zn7bft1VdfzW1jVbsj8Omnn5oXm1w/F6kPEUiCAOPeJ0+ebDNmzLDly5e7sdctW7Y0BTGIdw/wUoJ7BYE5ZcoU4x5K4lZTEhEQARFwBKoITSxGvPFkBlCXQh8ikCcEePmhZR5qfzFgiEAZM2aMWzNz7NixdvrppxvWMkoPj8FkLI5cZiGTP+HTTz+1YcOGuevHtcyflpVwSwqk67y0Qyx88cUX9tFHHxneC1OnTjUFMQjfA9wbkyZNMu4V7hnunQK5zdVMERCBPCFQRWgyXbX87vPk6qgZlQggNHHhYV3XSid0kBQBLzCxYLJkEeKSwBIawQLuqhCgweP77r3XGCcbjNN+7gngxhwUm1zf3LdKLSgkAliqGH/Nb2uxBvVrsdWUAfcG90gh3dNqqwiIQH4RqCI0eYjXD0t+XSS15i8CvAAhcI/+FaPPZAggQI488kjnIkt6xCUCBaHCcayAqyzxr7/+ultag32F/CPANeR69u7d27Bscq3zr5VqkQiIgAhUIqADERCBEiFQRWgy8LtE+q5uFiABhGYkomVOkrl0iI6zzz7bCAiSww47zLnHsl9d/htvvNHmzZtnWDOrS6vzuSeA2KQVvFDgurOvIAIiIAIiIALJE1BKEUg/gSpCM/1VqEQREIFsEsCyhXssApMxN4gQP/4ylXbce889ppmnUyGW27QjR440XiJw3SU2c3stVLsIiIAIiIAIpIVAgRcioVngF1DNFwEIICwQmGPGjHETxIQn+CFNquGtt95KNYvS55gAYpPZgyU2c3whVL0IiIAIiIAIiIAVq9DUpRWBkiDgBSYWTMbpXX755W4G2fAEPyUBQ510BLBeS2w6FPoQAREQAREQARHIIQEJzRzCL72q1eN0EQgKTMrEPZbAshccK5Q2AYnN0r7+6r0IiIAIiIAI5AMBCc18uApqgwgkSQCByYQvuEaSJZUJfkgfMyiyKAkExeY222xTlH1Up0RABERABERABPKXgIRm/l4btUwEogQQCrjHIjCZ8AXrJUIimkA7RUcgHR3iHhk1apT169fPLX+SjjJVhgiIgAiIgAiIgAgkQ0BCMxlKSiMCOSLABD8IzH4VQoFxdxKYOboQBVwt7tSITcbwcj8VcFfyoelqgwiIgAiIgAiIQJIESk5oHn300fb666/bO++848IDDzyQJKqqyc477zwbP3687brrrpVOhuvwdcVKWymjDkSgggDusQiCdM4gW1Gs/pcwAcTmsGHD3IzE3FsljEJdL0oC6pQIiIAIiEA+EigpoYkAPPTQQ23cuHG2+eab25AhQ2zNNde0G2+8MaVrgzhFPO6xxx4J8z3xxBOuHuoilJeX2zPPPJMwj06WLgEvMLFgso/1kqAZZEv3nkhnz3G5DopN7rF0lq+yREAERKASAR2IgAiUPIGSEprbbbedzZgxwy655BJ34RF9L774onXv3r2KVdIliPGB9bJ58+Z2++23G0IyRhJFiUBKBHjgZ4IfBCYZEZesh4gw4FhBBNJFgHuK+ws3Wu457r10la1yREAEREAE8p+AWigC2SRQMkITgYj18uOPP67Ed9asWVavXj3beOONDYsnbrVBC+emm25qTz/9tAvsI0532mknu/XWWyuVk44D6sVS6gOWU18u+z6eLWl9e3Hh9enY0lfcdMPxnFPIHwLdunUzJvchIAA0g2z+XJtibwlik3tOYrPYr7T6JwIiIAIiUAAEiraJJSM0O3To4C4iwtLtrPz47rvv7I8//rA2bdo48Yhbba9evZzoJEnfvn1t9dVXd+617733HlEZCQhJLKu48+Jme9xxxznrK5UhKhs1amTEcQ5r6uLFi+3999+3uXPnWs+ePUkWDYhmDj744AM2CnlGgDFyWC8HDhxoU6dONR76mR00z5qp5hQ5AW8150WHLJtFfrHVPREQAREQARHIAYHCFpppAPbzzz/b77//Hi3p2WefdeINN1ssgzvssIPh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NDohAiIgAiJQjASCYrNbt27F2EX1qYgIIDJ32WUXJ36PP/74Sj275JJLLBxXKYEOskogEom4+vQhAqVCANfZSCT+fS+hWSp3gvopAiIgAiIQJYDYHDt2rA0cONBY/iR6QjuVCOy6665u/KF318TtdNSoUS6Oc5USJzjA/dSXwXhGn5dxja+//rohpoLZOU+6YHy8MshHOaQ/7bTT3NhEXxftxdJHGsrCKrjGGmvYZptt5txlKZNziYJviy+TLWXFykN5nCfQHvLGSheMoyzS+8CxP0/bt9pqK5s4caLdeuutPrrKlv6PGDHCYvWNMuDgy6ddV199tcGdfFUKU4QIFCcB9SoHBCQ0cwBdVYqACIiACOSewGuvvWaIpn79+tlee+2V+wblWQsQSYMGDbKZM2dG3TXffPNN22CDDZJuqRc5jRo1MsYN4vI5ZcoUo1zKf//99431Bnv27FmpzI033tgdf/DBB1ZdGS5hxQfuW/vss4898cQTrr24vzZr1swGrBy3iPWPNsyaNcveffddlyYZt1jGOr744osuPe2nfCyMYZF24IEH2scffxxNBzffz4rmxfzP+ErKoq2UzZZjLzY32WQTY13GSZMmxczvIxGhJ598soX7BruhQ4e6ZPSdOu677z7bcsstXZw+REAERCCTBOpUW7gSiIAIiIAIiECREvj0009t2LBhbq1Nic3KFxmBxfT1w4cPj55ArH399dfR4+p2+vbta4g9xJkfNzh69GijXMonjnGHrMWG8PTlITwRas8884xVV4bPwzZo+UN8zZgxw8rKypxY5XxNAq6p9NvnffbZZ504DgpuRC4COphuzJgxLgv9dDuhD/rbvXv3StZK2kwfsGIiErFQku27775jk3KAHUIVQQtrCvB1sK8gAiIgApkkIKGZSboZLFtFi4AIiIAIpIcAa2x6sam1Nv9iishBoP32229VZi6F11+pqv9EMGKxxHLpUyN4KPf//u//XBTWOoSat2IiwJo3b24vv/yyO59MGST8448/jLLY94G2IrRatWrlo2q0pU24nOJ+6t1vwwX9+OOPlaIQyYhl389KJysO6C/9DreZctLR5ooqDHaI+p9//pnDaKCO6IF2ip5A586drVOnTjH7ufvuu7uZuHv06BHzPDPpJvr+rL/++m6G45iZFZk1AocccogxdGCttdbKWp3JVCShmQwl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)

#### 面试简答口径

> 配置层主要有三张表。第一张是诊断项表，配置诊断ID、执行周期、使能状态和抑制条件，决定一个诊断什么时候能运行；第二张是故障事件表，配置事件归属、故障等级和Fail/Pass防抖，决定诊断结果如何转换成DEM事件；第三张是任务绑定表，通过函数指针把诊断ID与Init、MainTask和GetKeyInfo函数绑定，决定框架实际调用哪段业务代码。三张表由CSV经过脚本自动生成到 `diag_cfg.c`，上电时由诊断框架初始化，运行时由调度、抑制和DEM模块共同查表使用。

---

## **六、关键时序与流程图说明**

### 6.1 上电完整UML时序

1. `DiagInit`调用`DiagFpgaPara1Init`重置状态机至WAIT
2. 每5ms `DiagSelftestTask`调度`DiagFpgaPara1MainTask`
3. 状态WAIT：轮询CDD加载状态，未就绪返回UNFINISHED
4. 参数LOADED：调用接口读取双CRC快照，切换至CRC_RUN
5. CRC_RUN：执行注入判定+原生CRC比对，上报DEM故障
6. 状态切换DONE，清除自检标志，后续周期不再执行

### 6.2 主状态机流转逻辑图

![image](https://cdn.nlark.com/yuque/0/2026/png/27841183/1786893423244-984f72c4-a523-450c-b78e-9ef263151573.png)

- WAIT：轮询CDD加载状态，未就绪持续等待
- CRC_RUN：读取快照 → 读取注入指令 → 优先级判定 → DEM上报 → 切换DONE
- DONE：空操作，自检永久结束

### 6.3 五层端到端全流程时序

![image](https://cdn.nlark.com/yuque/0/2026/png/27841183/1786893341835-9440759d-438d-4504-b0b5-af26dcf8881a.png)

1. 配置层：CSV定义0x1E00/0x1E01基础规则
2. 工站层：PTC下发注入、NVM存储、整机复位
3. 调度层：上电自检周期调度、DEM故障防抖确认
4. 算法层：三态机、CRC比对、注入旁路、KeyInfo快照
5. 数据源层：CDD Flash加载、双CRC输出

## 七、核心设计要点总结

1. **单次上电自检**：DONE状态永久锁定，整机运行期间不再重复校验，匹配上电参数完整性自检需求
2. **异步依赖等待**：CDD参数加载为异步流程，采用周期轮询不阻塞主任务
3. **无效CRC容错**：读取失败填充0xFFFFFFFF哨兵值，直接屏蔽故障上报，降低误报概率
4. **故障注入高优先级**：产线FHTI校验时可强制模拟故障/正常，独立于硬件真实CRC
5. **8字节故障快照**：故障冻结帧携带双CRC原始值，快速定位参数篡改/加载损坏
6. 与ModeCtrl联动：模式切换`STEP_ID_WAIT_FPGA_PARA_READY`等待参数加载完成，保证CRC校验在参数下发前执行

## 八、配套测试校验链路

![image](https://cdn.nlark.com/yuque/0/2026/png/27841183/1786893333666-8c50d122-362d-4a1b-ba68-7738d7a27e64.png)

1. 配置校验：PTC 0x1E00查询功能是否启用、故障参数配置正确性
2. 运行校验：上电后查询系统故障码，确认无原生CRC故障（Happy Path）
3. 注入校验：下发故障注入指令复位整机，校验0x1E01故障可正常触发上报
