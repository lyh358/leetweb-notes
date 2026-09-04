# Autosar‑CP：ASW (SWC) 上报故障，从应用层 → DEM → DCM → UDS 上位机完整流程

> 核心角色

- **ASW/SWC**：应用软件组件，做故障监控，只上报**EventId**，**不直接操作 DTC 码**
- **=={yellow}DEM(Diagnostic Event Manager)==**=={yellow}：**BSW** 模块==，=={green}**事件去抖、确认、Event‑DTC 映射、存储冻结帧**、NvM 持久化；==**=={green}DTC 真正的管家==**
- **=={yellow}DCM(Diagnostic Communication Manager)==**=={yellow}：**BSW** 诊断通信==，=={green}**U生成DS 协议报文**；==**=={green}不和 ASW 直接打交道，只和 DEM 交互==**
- **Tp/CanTp**：网络层，ISO‑TP 分包重组；**DCM 往下调用 Tp，Tp 输出 CAN 报文给总线**
- 上位机 (Tester / 诊断仪)：CAN 总线上位机，发 UDS 0x19 读 DTC，0x14 清 DTC

> 重要概念区分

1. **EventId**：ASW 层内部事件 ID，SWC 调用 Dem_SetEventStatus 使用
2. **DTC**：对外给诊断仪看的故障码 (P/U/C 码)，在 DEM 配置中把 EventId 绑定到 DTC；**ASW 不知道 DTC 编号，只知道 EventId**

## 完整链路分两大阶段

### 阶段 1：ASW 检测到故障，上报给 DEM（故障发生路径，**主动上报**）

1. **ASW‑SWC 内部监控逻辑**
应用组件运行，检测传感器 / 执行器 / 算法异常，得到监控结果。

```
// ASW组件内部
if(SensorVoltage > Limit)
{
    // 检测到故障，上报事件失败
    Dem_SetEventStatus(EVENTID_SENSOR_OVERVOLT, DEM_EVENT_STATUS_FAILED);
}
else
{
    Dem_SetEventStatus(EVENTID_SENSOR_OVERVOLT, DEM_EVENT_STATUS_PASSED);
}
```

`Dem_SetEventStatus()`：**ASW 上报诊断事件的标准 API，通过 RTE 调用 DEM**。

> 参数：EventId（配置好的事件 ID） + 事件状态（FAILED/PASSED/PREFAILED/PREPASSED）

1. **DEM 内部处理（最关键）**
DEM 收到 EventId + 状态，执行：
① 检查**使能条件 EnableCondition**：该监控是否允许运行（如点火档位、温度条件），不满足直接丢弃事件CSDN博...。
② **Debounce 去抖**（计数器去抖 / 时间去抖）：过滤瞬时毛刺，避免误报 DTC。不是上报一次 FAILED 就直接报 DTC。
③ 去抖到达失败阈值 → **Event 确认失败**，DEM 把 Event 映射到配置好的 DTC。
④ 更新**DTC 状态字节 (ISO‑15031 状态位)**：confirmedDTC、testFailed 等 bit 置位。
⑤ 抓取**冻结帧 Snapshot / 扩展数据 ExtData**（故障时刻的车速、电压、温度）。
⑥ 调用 NvM，把 DTC、冻结帧存入 Flash/EEPROM 持久化，下电不丢失CSDN博...。
⑦ 通知 FIM 模块：故障发生，可做功能抑制（关闭某些功能）。

> ⚠️这里：**DEM 完成 DTC 生成、存储；DCM 此时完全没参与，没有报文发出**。故障存在 ECU 内部，**只有上位机主动读 UDS 才会把 DTC 发出去**，不是故障一发生就自动上送 CAN 报文。

---

### 阶段 2：上位机（诊断仪）UDS 读取 DTC，DCM 查询 DEM，组装 UDS 响应返回（查询路径）

> 上位机主动发 UDS `0x19 ReadDTCInformation` 请求，DCM 接收解析，向 DEM 拿数据，组装报文回复上位机CSDN博...。

报文交互时序：

1. **上位机发送 CAN 报文 (UDS 请求)**：SID=0x19，子功能 0x02 reportDTCByStatusMask，带上状态掩码。
2. CanTp（ISO‑TP 网络层）接收 CAN 帧，重组完整 UDS 请求，传给 DCM。
3. **DCM 模块处理**
  - 解析 UDS 请求：SID、子功能、掩码参数；
  - 会话 / 权限校验：0x19 大部分子功能默认会话就允许；
  - **DCM 调用 DEM 提供的一组 API，向 DEM 查询 DTC 数据**（DCM 不会自己存 DTC）：
    - `Dem_SelectDTCByStatusMask()`：按掩码筛选 DTC
    - `Dem_GetNextDTC()`：遍历筛选出来的 DTC
    - `Dem_GetDTCStatus()`：获取 DTC 状态字节
    - `Dem_GetNextFreezeFrameData()`：拿冻结帧数据CSDN博...
4. **DEM 返回 DTC 列表、DTC 码、DTC 状态字节、冻结帧 / 扩展数据给 DCM**。
5. **DCM 组装 UDS 肯定响应报文**：SID+0x40=0x59，把 DTC、状态、冻结帧按 UDS 协议格式打包。
6. DCM 把完整 UDS 响应交给 CanTp，CanTp 做分包，输出 CAN 报文到总线。
7. CAN 总线 → 上位机收到 UDS 响应，解析得到 DTC 故障码、冻结帧。

## 清除 DTC 流程（UDS 0x14）补充

1. 上位机发`0x14`清除 DTC 请求。
2. DCM 校验会话权限，调用`Dem_ClearDTC()`接口。
3. DEM 清除 DTC 状态，清除冻结帧，通知 NvM 擦除存储。
4. DEM 返回结果给 DCM，DCM 回复`0x54`肯定响应给上位机.。

## 模块间调用关系简图

```
ASW‑SWC
    ↓RTE调用Dem API
Dem_SetEventStatus(EventId,status)
    ↓
DEM【去抖、Event→DTC映射、冻结帧、NvM存储】
    ↓DCM ←‑‑‑UDS请求触发才查询（DCM不会主动读DEM）
DCM调用DEM查询接口获取DTC/冻结帧
    ↓
CanTp(ISO‑TP网络层)
    ↓
CAN总线 ←→ 上位机诊断仪(UDS tester)
```

## 高频踩坑点（工程实际）

1. ❌ASW 直接操作 DTC 编号：ASW 只能传 EventId，DTC 是 DEM 配置绑定的，SWC 代码不能写死 DTC 码。
2. ❌上报 Dem_SetEventStatus 一次 FAILED，期待诊断仪立刻读到 DTC：DEM 有去抖，要等 debounce 阈值到，DTC 才会 confirmed 确认。
3. ❌故障上报之后 DCM 自动往外发报文：**DCM 不会主动上报 DTC**；DTC 存在 DEM 内存 / NvM，必须上位机发 0x19 服务才读出。
4. ❌忘记 EnableCondition 配置：监控条件不满足，DEM 直接丢弃 ASW 上报的 Event，DTC 永远出不来。
5. ❌混淆 Dem_SetEventStatus（ASW/SWC 用）与 Dem_ReportErrorStatus（BSW 内部用），ASW 不能调用 Dem_ReportErrorStatus。
6. 冻结帧采集时机：DEM 确认 DTC 那一刻抓取快照；如果配置错误，读到的冻结帧全部是 0。

## 关键 API 小结

表格

| 模块 | API | 方向 | 作用 |
| --- | --- | --- | --- |
| ASW→DEM | `Dem_SetEventStatus(EventId,Status)` | ASW 调用 | 上报监控事件状态（FAILED/PASSED） |
| DCM→DEM | `Dem_SelectDTCByStatusMask` | DCM 调用 | 筛选 DTC 集合 |
| DCM→DEM | `Dem_GetNextDTC` | DCM 调用 | 遍历 DTC |
| DCM→DEM | `Dem_GetDTCStatus` | DCM 调用 | 获取 DTC 状态字节 |
| DCM→DEM | `Dem_ClearDTC` | DCM 调用 | 清除故障码（对应 UDS 0x14） |
