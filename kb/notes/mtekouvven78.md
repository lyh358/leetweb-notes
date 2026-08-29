## 一、需求整体概述
### 1 需求编号与诊断标识
需求：SWSR-006-04 Flash CRC Diagnosis（机制短名`FLASH_CRC_DIAG`）  
诊断基础ID：`DIAGID_FLASH_CRC_DIAG = 0x0E00`，上电自检类型`DIAG_PERIOD_ST`，整机上电仅执行一次，无5ms周期调度。

### 2 6项待校验Flash数据对象，一一对应故障事件
| 校验对象 | 故障事件EVTID | 故障码 | 故障等级 | CRC存储位置 |
| --- | --- | --- | --- | --- |
| FPGA bit镜像文件 | EVTID_FACS_BIT_FAULT | 0x0E01 | Output Untrusted(d7) | Bit文件头部`verifyValue` |
| FPGA参数Para1 | EVTID_FACS_PARAMETER_1_FAULT | 0x0E02 | Output Untrusted | Para文件末尾4字节 |
| FPGA参数Para2 | EVTID_FACS_PARAMETER_2_FAULT | 0x0E03 | Output Untrusted | Para文件末尾4字节 |
| 工厂VOV参数 | EVTID_FACS_VOV_VALUE_FAULT | 0x0E05 | Output Untrusted | 独立FACT CRC分区表 |
| 工厂VBR查找表 | EVTID_FACS_VBR_TABLE_FAULT | 0x0E06 | Output Untrusted | 独立FACT CRC分区表 |
| 高温开关参数 | EVTID_FACS_HIGATEMP_SWITCH_FAULT | 0x0E07 | Output Untrusted | 独立FACT CRC分区表 |


+ 排除项：RTM校准参数0x0E04本期不实现；
+ 故障规则：CRC存储值与实时计算值不一致则上报故障；最多重试3次（初检+2次重载重读）；无自动恢复逻辑，故障上报后不会自动清除；
+ 冻结帧：每条故障固定8字节，前4Byte存储CRC、后4Byte实时计算CRC，供售后定位Flash损坏/篡改问题。

### 3 执行前置约束
1. 系统模式不为Fault故障态才允许运行自检；
2. 依赖`CddPeriPara`外设参数模块加载完成，未就绪则持续等待；
3. 故障抑制：入口电压故障`IR_PWRIN_VOLT_L2`触发时，屏蔽本类所有Flash CRC故障上报。

## 二、Flash分区与CRC三种存储架构
### 类型A：FPGA Bit（头部内嵌CRC）
+ 数据分区：`MAIN_FPGA_BIT_A/B`；
+ CRC位置：镜像头`ImageHeadInfo.verifyValue`；
+ 计算范围：头偏移128字节至文件末尾；
+ 读取API：`SWC_FlashSupport_CalculateRequestSync`。

### 类型B：FPGA Para1/Para2（文件尾部CRC）
+ 数据分区：`MAIN_FPGA_PARA1_A/B`、`FACP_FPGA_PARA2`；
+ CRC位置：文件最后4字节；
+ 读取API：`CddPeriPara_GetParaCrcGet/GetParaCrcCal`；
+ 校验失败支持重载`StartSingleLoadPara`重新加载参数。

### 类型C：工厂小参数VOV/VBR/HIGHTEMP（独立CRC分区）
+ 数据区：工厂小参数分区；CRC单独存放`FACT_PARA_CRC32_TABLE`4KB分区；
+ 每条CRC记录格式：`[offset(31bit) + CRC32(4Byte)]`；
+ 读取API：`CddParaM_ReadSync + CddParaM_GetIcValue`。

### 分区长度兼容方案
包头新增3Byte芯片CRC标识，包尾预留区同步缩减2Byte，整机UDP遥测包总长保持501字节不变，上位机无需修改收包逻辑。

## 三、平台配置自动生成规则
### 1 配置源文件（Fault Matrix.csv）
表格字段映射自动生成三类代码文件：

1. `diag_config_types.h`：枚举`DIAGID`、6个`EVTID`故障ID；
2. `diag_config_tables.c`：`g_eventCfgTable`故障策略表（故障等级、Debounce、DTC、抑制掩码）；
3. `diag_item_table.c`：诊断函数挂载表，绑定`Init/Selftest/GetKeyInfo`三个入口。

### 2 核心配置参数
+ 自检周期：`DIAG_PERIOD_ST`，无5ms任务；
+ Debounce：故障/恢复均为1次判定，上电自检无需多周期滤波；
+ 故障DTC：统一`0xC10BA09`；故障动作：整机输出不可信、允许关机；
+ 冻结帧长度固定8Byte，存储双CRC数值。

## 四、核心业务模块：diag_flash_crc.c/.h
### 1 对外三大标准诊断接口（框架钩子函数）
1. `DiagFlashCrcDiagInit()`：上电初始化，清空动态CRC缓存、复位状态机至等待外设就绪；
2. `DiagFlashCrcDiagSelftest()`：5ms周期协作式自检主逻辑，非阻塞分步执行；
3. `DiagFlashCrcDiagGetKeyInfo()`：UDS/产线读取冻结帧，按故障ID返回8Byte双CRC。

### 2 四大枚举类型（表驱动架构）
1. `DiagFlashCrcCheckType`：区分6类Flash校验方式（BIT/PARA1/PARA2/VOV/VBR/HIGHTEMP）；
2. `DiagFlashCrcMainState`：自检全局四状态机；
3. `DiagFlashCrcItemCfgType`：静态配置，绑定校验类型+对应EVT故障ID；
4. `DiagFlashCrcItemDynType`：运行时RAM缓存，存储每条CRC、是否上报标记。

#### 自检状态机流转
1. `WAIT_DEPS`：等待CddPeriPara参数加载完成，未就绪持续返回未完成；
2. `CHECK`：逐项读取、比对CRC，失败进入重试逻辑；
3. `RETRY_WAIT`：Para重载后等待外设就绪，再次校验；
4. `DONE`：6项全部校验完成，后续周期不再执行自检。

### 3 内部静态工具函数
| 函数 | 功能 |
| --- | --- |
| DiagFlashCrcIsPeriParaReady | 判断外设参数是否加载完毕 |
| DiagFlashCrcIsCrcMismatch | 判定CRC是否失效（0/0xFFFFFFFF视为合法空值，不报故障） |
| DiagFlashCrcReadBitCrc | 读取FPGA Bit头部存储CRC+重算CRC |
| DiagFlashCrcReadPeriParaCrc | 读取Para1/Para文件首尾双CRC |
| DiagFlashCrcReadFacsParaCrc | 工厂小参数独立CRC表读取 |
| DiagFlashCrcReloadCurrentItem | Para校验失败触发参数重载 |
| DiagFlashCrcReportItem | 向DEM上报故障事件 |
| DiagFlashCrcGetItemCfg | 按索引读取静态校验配置表 |


### 4 非阻塞协作式设计核心
1. 单次5ms调度仅执行一步逻辑，不阻塞整机任务；
2. 6项数据顺序遍历，每项独立缓存CRC结果；  
3 Para1/Para2校验失败最多重载2次，三次校验不一致才上报故障；
3. 每项校验完成立即上报DEM，不等待全部检测结束；
4. 冻结帧数据存储在RAM动态数组，读取时直接返回，不重复读取Flash。

## 五、软件分层调用架构
1. **SWC_DIAG调度层**：上电初始化、5ms周期调用Selftest、UDS读取冻结帧；
2. `diag_flash_crc`业务层：状态机、CRC读取、故障上报；
3. CDD底层驱动层：
    - `CddPeriPara`：FPGA参数读写、CRC接口、重载；
    - `CddParaM`：工厂小参数分区、独立CRC表读写；
    - `SWC_FlashSupport`：FPGA Bit镜像CRC计算；
4. 硬件层：片外Flash存储分区读写。

## 六、完整时序流程
1. 整机上电→`DiagFlashCrcDiagInit`初始化状态机与缓存；
2. 5ms周期反复进入`WAIT_DEPS`，等待PeriPara加载完成；
3. 进入`CHECK`，从第0项Bit开始逐项读取CRC比对；
4. Para类CRC不一致：触发重载，切`RETRY_WAIT`等待外设就绪后重检；
5. 校验通过/重试耗尽：调用`DiagSendResultToDem`上报故障/正常状态；
6. 6项全部遍历完毕，状态切`DONE`，自检永久结束；
7. 产线/UDS读取故障信息：调用`GetKeyInfo`读取缓存双CRC冻结帧。

## 七、开发与交付要点
1. 复用AUTOSAR诊断标准框架，完全贴合平台诊断自动生成工具；
2. 表驱动设计，新增Flash参数仅扩展配置表，无需修改核心状态机；  
3 分层解耦：诊断业务不直接操作Flash驱动，统一通过CDD标准接口；
3. 容错设计：空CRC、读取失败不会误报故障，仅有效数值不一致才触发Output Untrusted故障；
4. 产线验收：支持PTC故障注入、UDS读取故障码与8字节CRC冻结帧，快速定位Flash存储损坏问题。

