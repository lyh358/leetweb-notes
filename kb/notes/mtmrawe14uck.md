# =={pink}Autosar‑CP：ASW (SWC) 上报故障，从应用层 → DEM → DCM → UDS 上位机完整流程==

> 核心角色

- **=={yellow}ASW/SWC==**=={yellow}：应用软件组件==，=={green}做故障监控，只上报==**=={green}EventId==**，**不直接操作 DTC 码**
- **=={yellow}DEM(Diagnostic Event Manager)==**=={yellow}：**BSW** 模块==，=={green}**去抖、故障确认、Event‑DTC 映射、存储冻结帧**、NvM 持久化；===={green}**DEM 返回 DTC + 冻结帧给 DCM**==
- **=={yellow}DCM(Diagnostic Communication Manager)==**=={yellow}：**BSW** 诊断通信==，=={green}**生成UDS 协议报文**；===={green}不和 ASW 直接打交道，只和 DEM 交互==
- **=={yellow}CanTp==**=={yellow}**（CAN‑Transport Protocol）**：**BSW 模块**==，CAN 总线诊断传输层 / 网络层；**=={green}DCM 往下调用 CanTp，输出 CAN 报文给CAN总线==**
- =={yellow}**上位机 ：CAN 总线上位机**==，发 UDS报文 0x19 读 DTC，0x14 清 DTC

> 重要概念区分

1. **EventId**：ASW 层内部事件 ID，SWC 调用 Dem_SetEventStatus 使用
2. **DTC**：对外给诊断仪看的故障码 (P/U/C 码)，在 DEM 配置中把 EventId 绑定到 DTC；**ASW 不知道 DTC 编号，只知道 EventId**
3. =={green}UDS是==**=={green}应用层服务==**=={green}（0x10、0x19、0x22…）；CanTp 是==**=={green}下层运输工具==**=={green}，负责把 UDS 字节包搬上 CAN 总线。==

## =={pink}完整链路==分两大阶段

### =={yellow}阶段 1：ASW 检测到故障，上报给 DEM（故障发生路径，==**=={yellow}主动上报==**=={yellow}）==

1. **ASW‑SWC 内部监控逻辑**
应用组件运行，检测传感器 / 执行器 / 算法异常，得到监控结果。

`Dem_SetEventStatus()`：**=={green}ASW 上报诊断事件的标准 API，通过 RTE 调用 DEM==**=={green}。==

1. **DEM 内部处理（最关键）**
DEM 收到 EventId + 状态，执行：

① 检查**使能条件 EnableCondition**：该监控是否允许运行（如点火档位、温度条件），不满足直接丢弃事件CSDN博...。 ② **Debounce 去抖**（计数器去抖 / 时间去抖）：过滤瞬时毛刺，避免误报 DTC。不是上报一次 FAILED 就直接报 DTC。
③ 去抖到达失败阈值 → **Event 确认失败**，DEM 把 Event 映射到配置好的 DTC。
④ 更新**DTC 状态字节 (ISO‑15031 状态位)**：confirmedDTC、testFailed 等 bit 置位。
⑤ 抓取**冻结帧 Snapshot / 扩展数据 ExtData**（故障时刻的车速、电压、温度）。
⑥ 调用 NvM，把 DTC、冻结帧存入 Flash/EEPROM 持久化，下电不丢失CSDN博...。
⑦ 通知 FIM 模块：故障发生，可做功能抑制（关闭某些功能）。

> ⚠️这里：**DEM 完成 DTC 生成、存储；DCM 此时完全没参与，没有报文发出**。故障存在 ECU 内部，**只有上位机主动读 UDS 才会把 DTC 发出去**，不是故障一发生就自动上送 CAN 报文。

---

### =={pink}阶段 2：上位机（诊断仪）UDS 读取 DTC，DCM 查询 DEM，组装 UDS 响应返回（查询路径）==

> 上位机主动发 UDS `0x19 ReadDTCInformation` 请求，DCM 接收解析，向 DEM 拿数据，组装报文回复上位机CSDN博...。

报文交互时序：

1. **=={yellow}上位机发送 CAN 报文 (UDS 请求)==**：SID=0x19，子功能 0x02 reportDTCByStatusMask，带上状态掩码。
2. **=={yellow}CanTp==**=={yellow}（ISO‑TP 网络层）==**=={yellow}接收 CAN 帧==**=={yellow}，==**=={yellow}重组完整 UDS 请求==**=={yellow}，==**=={yellow}传给 DCM==**=={yellow}。==
3. **DCM 模块处理**
  - **解析 UDS 请求**：SID、子功能、掩码参数；
  - 会话 / 权限校验：0x19 大部分子功能默认会话就允许；
  - **DCM 调用 DEM 提供的一组 API，向 DEM 查询 DTC 数据**（DCM 不会自己存 DTC）：
    - `Dem_SelectDTCByStatusMask()`：按掩码筛选 DTC
    - `Dem_GetNextDTC()`：遍历筛选出来的 DTC
    - `Dem_GetDTCStatus()`：获取 DTC 状态字节
    - `Dem_GetNextFreezeFrameData()`：拿冻结帧数据CSDN博...
4. **DEM 返回 DTC 列表、DTC 码、DTC 状态字节、冻结帧 / 扩展数据给 DCM**。
5. **DCM 组装 UDS 肯定响应报文**：SID+0x40=0x59，把 DTC、状态、冻结帧按 UDS 协议格式打包。
6. **DCM 把完整 UDS 响应交给 CanTp**，**CanTp 做分包，输出 CAN 报文到总线。**
7. **CAN 总线 → 上位机**收到 UDS 响应，解析得到 DTC 故障码、冻结帧。

---

## UDS 是什么 & 在 AUTOSAR‑CP 故障上报链路中的角色

**UDS：Unified Diagnostic Services，统一诊断服务**。它是**汽车诊断的应用层协议**，定义一套标准化服务命令
=={yellow}**UDS 报文** === `[SID] + （可选）子功能 + 业务参数`
`字段说明SID 1 字节服务 ID，如0x19读 DTC、0x14清 DTC、0x10会话控制Subfunction 1 字节子功能，部分服务需要，例如0x19‑02读匹配状态 DTC；bit7 = 抑制正响应位参数各个服务自定义参数，DTC 掩码、DID 号等`
