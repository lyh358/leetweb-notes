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

主要由**=={yellow}诊断事件处理模块===={yellow}**和==**===={yellow}DEM接口==**使用：

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

##### 面试简答口径

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
