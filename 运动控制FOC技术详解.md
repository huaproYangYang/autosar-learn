# 运动控制中的 FOC（磁场定向控制）技术详解

> 适用读者：车载电控、电机驱动、嵌入式控制、AutoSAR 应用/集成工程师
> 版本：v1.0（2026-08-13）
> 关键词：FOC、Vector Control、PMSM、BLDC、SVPWM、Clark/Park、MTPA、弱磁、AutoSAR、CDD、MCAL、GTM

---

## 目录

- [1. 为什么需要 FOC](#1-为什么需要-foc)
- [2. FOC 的基本原理](#2-foc-的基本原理)
- [3. 坐标变换：Clark 与 Park](#3-坐标变换clark-与-park)
- [4. SVPWM 空间矢量脉宽调制](#4-svpwm-空间矢量脉宽调制)
- [5. FOC 控制环级联与解耦](#5-foc-控制环级联与解耦)
- [6. PMSM / BLDC / 感应电机的差异](#6-pmsm--bldc--感应电机的差异)
- [7. 位置与速度反馈：传感器](#7-位置与速度反馈传感器)
- [8. 无传感器 FOC 控制](#8-无传感器-foc-控制)
- [9. 启动策略：预定位 / I/F / 高频注入](#9-启动策略预定位--if--高频注入)
- [10. 弱磁控制与 MTPV](#10-弱磁控制与-mtpv)
- [11. 电流采样方案对比](#11-电流采样方案对比)
- [12. FOC 周期、执行时间与 MCU 选型](#12-foc-周期执行时间与-mcu-选型)
- [13. FOC 在 AutoSAR 中的软件架构](#13-foc-在-autosar-中的软件架构)
  - [13.1 MCAL 层（GTM/CCU6/ADC/PWM）](#131-mcal-层gtmccu6adcpwm)
  - [13.2 BSW 与复杂驱动 CDD](#132-bsw-与复杂驱动-cdd)
  - [13.3 RTE 与 SWC](#133-rte-与-swc)
  - [13.4 OS 调度与多核并行](#134-os-调度与多核并行)
- [14. FOC 的功能安全（ISO 26262）](#14-foc-的功能安全iso-26262)
- [15. 故障诊断与 UDS/DTC](#15-故障诊断与-udsdtc)
- [16. 常用开发工具链](#16-常用开发工具链)
- [17. 定点化与优化](#17-定点化与优化)
- [18. 典型应用场景](#18-典型应用场景)
- [19. 调参方法论与经验值](#19-调参方法论与经验值)
- [20. 进阶话题：自适应 / 预测控制 / AI 调参](#20-进阶话题自适应--预测控制--ai-调参)
- [21. 总结与推荐资料](#21-总结与推荐资料)

---

## 1. 为什么需要 FOC

电机在汽车与工业领域无处不在：EPS 转向助力、电动空调压缩机、EPB 电子驻车、ESC 液压泵、增程器、油泵/水泵、主动悬架、伺服驱动、机器人关节等。

早期的 BLDC 采用 **六步换相（梯形波）** 控制，转矩脉动大、噪声高、无法做精细弱磁与高效区运行；而 FOC（Field-Oriented Control，磁场定向控制，也叫矢量控制 Vector Control）通过坐标变换把交流电机等效成"直流电机"来控制，可获得：

- **平滑、低噪声、低转矩脉动** 的转矩输出
- **宽调速范围**（基速以下恒转矩、基速以上弱磁恒功率）
- **高动态响应**（电流环带宽 1–5 kHz，转速环 100–500 Hz）
- **高效率**（MTPA + 弱磁联合优化，效率图（efficiency map）> 95%）
- **天然支持无传感器、参数辨识、扭矩观测**

代价是计算量大（每个 PWM 周期都要做三角函数、PI、Park/Clark 变换），需要 **带硬件 FPU / CORDIC / DSP** 的 MCU（如 AURIX TC3xx、RH850、STM32G4、GD32E503、TC264 等）。

---

## 2. FOC 的基本原理

核心思想：通过两次坐标变换把三相交流量变成两相直流量。

```
三相静止 abc ──Clark──> 两相静止 αβ ──Park──> 两相旋转 dq
       ↑                                          │
       │                                          ▼
    PWM/SVPWM ◀──────── 反 Park + SVPWM ◀───── 直流 PI 控制
```

- **d 轴（direct）**：与转子磁链方向一致，控制 **磁通**
- **q 轴（quadrature）**：与转子磁链正交，控制 **转矩**

对于表贴式 PMSM：基本策略是 **id = 0**（让 d 轴不产生去磁），**iq 决定转矩**（`T = 1.5 · Pp · ψf · iq`）。

对于内嵌式 IPM：因为磁阻转矩存在，要走 **MTPA**（Maximum Torque Per Ampere）轨迹。

---

## 3. 坐标变换：Clark 与 Park

### 3.1 Clark 变换（abc → αβ）

假设三相平衡，等幅值变换（功率不变需再乘 `sqrt(2/3)`）：

```
[ iα ]   [ 1   -1/2     -1/2  ] [ ia ]
[ iβ ] = [ 0   √3/2   -√3/2  ] [ ib ]
                                 [ ic ]    （常省略 ic = -ia - ib）
```

### 3.2 Park 变换（αβ → dq）

```
[ id ]   [ cosθ   sinθ ] [ iα ]
[ iq ] = [ -sinθ  cosθ ] [ iβ ]
```

其中 **θ 是电角度**（机械角 × 极对数 Pp）。

### 3.3 反 Park + SVPWM

把 dq 上的控制电压 `ud, uq` 通过反 Park 变换到 αβ，再由 **SVPWM** 调制出三相 PWM 占空比。

### 3.4 实现细节

- **三角函数查表 + 线性插值**：常见做法，省 CPU。
- **CORDIC 硬件加速**：AURIX TC3xx 内置 CORDIC 协处理器，可在 1–2 个 cycle 出结果。
- **角度归一化**：用 uint32 表示 0–2π（如 0–65535），避免浮点。
- **零序分量**：在三相三线制中 `ia + ib + ic = 0`，所以只需采样两相电流即可。

---

## 4. SVPWM 空间矢量脉宽调制

SVPWM 把 6 个非零基本电压矢量和 2 个零矢量（V0/V7）按"伏秒平衡"等效出任意目标电压矢量。

**优势：**
- 直流母线电压利用率比 SPWM 高 **15.47%**（`Vmax = Vdc/√3` vs `Vdc/2`）
- 谐波更小、转矩脉动更低
- 与 FOC 无缝衔接

**实现步骤：**

1. 判定目标矢量所在扇区（I–VI）
2. 计算相邻矢量作用时间 T1, T2
3. 若 `T1+T2 > Ts`，做 **过调制（over-modulation）** 限幅
4. 计算每相占空比 `Ta, Tb, Tc`
5. 装载到 MCAL PWM 模块

**典型寄存器（AURIX GTM TIM/TOM）：**
- `TIM0_CHx` 触发 ADC 采样点（与 PWM 中心对齐）
- `TOM0_CHx` 输出三相 PWM（带死区 DTM）

---

## 5. FOC 控制环级联与解耦

典型 FOC 是 **三环级联**：

```
转矩/功率指令 ──> 转速环 (PI) ──> iq* ─┐
位置环 (P/PI) ────> 转速指令 ─────────┤
                                          ├──> 电流环 (PI + 解耦) ──> SVPWM
                                      id* (0 或 MTPA 计算)
```

### 5.1 电流环（内环）

- 带宽：1–5 kHz（典型 2 kHz）
- 离散化：**后向欧拉** 或 **Tustin（双线性）**，采样周期 `Ts` 与 PWM 周期一致
- **解耦项**：PMSM 中 `ud = R·id + Ld·did/dt − ω·Lq·iq`，`uq = R·iq + Lq·diq/dt + ω·(Ld·id + ψf)`
  - 反馈项 `ω·Lq·iq`、`ω·Ld·id`、`ω·ψf` 必须补偿，否则 dq 耦合导致动态响应差。
- **前馈解耦** 比纯 PI 反馈快，且不会因为耦合产生超调。

### 5.2 转速环（外环）

- 带宽：100–500 Hz（电流环的 1/5–1/10）
- 离散化：典型 1 ms 周期
- 输出饱和限幅（防止 iq* 超限）
- **惯量辨识 + 转动惯量前馈** 可以显著改善阶跃响应（伺服刚需）

### 5.3 位置环（最外）

- 带宽：10–50 Hz
- 纯 P 会导致稳态误差，常用 **P + 前馈（速度前馈、加速度前馈）** 或 **PI + 速度前馈**
- 数字伺服里还会加 **陷波滤波器** 抑制机械共振

### 5.4 反 Park 与电压利用率
在弱磁区，`ud, uq` 受 `√(ud² + uq²) ≤ Umax` 限制，需要通过 **电压闭环（SVPWM 调制比闭环）** 自动分配 d/q 电压。

---

## 6. PMSM / BLDC / 感应电机的差异

| 维度 | PMSM (永磁同步) | BLDC (无刷直流) | IM (感应电机) | SynRM (同步磁阻) |
|---|---|---|---|---|
| 反电动势 | 正弦 | 梯形 | 接近正弦 | — |
| 转子 | 永磁体 | 永磁体 | 鼠笼/绕线 | 凸极（无永磁） |
| 控制方式 | FOC | 六步 / FOC | FOC（含转差） | FOC（强磁阻转矩） |
| 效率 | 高 | 中（梯形波） | 高（但滑差损耗） | 中高 |
| 弱磁能力 | 强（短磁） | 弱 | 强 | 强 |
| 成本 | 中高 | 低 | 低 | 中 |
| 典型应用 | EPS、空调、增程器 | 风扇、油泵 | 主驱、车载空调 | 主驱（部分车企） |

> **注**：BLDC 在控制算法上也是 FOC，"BLDC" 主要指反电动势波形与绕组分布形态。在汽车动力总成，PMSM 占主流。

---

## 7. 位置与速度反馈：传感器

| 传感器 | 分辨率 | 速度 | 成本 | 典型应用 |
|---|---|---|---|---|
| 增量式光电编码器 | 高（17–23 bit） | 中（万转级） | 中 | 伺服、机器人 |
| 绝对值编码器（SSI/BiSS/EnDat） | 17–25 bit | 中 | 高 | 数控、机器人 |
| 旋转变压器（Resolver） | 等效 14–16 bit | 高（>10万 rpm） | 高 | 电动车主驱 |
| Hall 传感器 | 6 步 / 60°电角度 | 低 | 极低 | 油泵、EPS 助力 |
| SinCos 模拟编码器 | 1–12 bit | 中 | 中 | 牵引 |

**Resolver 解码**：需要 RDC（Resolver-to-Digital Converter）或软件实现：注入正弦励磁 → 采样 sin/cos → `atan2 → 角度 → PLL 平滑`。

**AutoSAR 集成**：通常以 **ICU（Input Capture Unit）** 模式采 Hall 信号；编码器一般用 **SPI / SSI 驱动（MCAL Spi）** + 中断或 DMA。

---

## 8. 无传感器 FOC 控制

**目标**：省去位置/速度传感器，降低成本与失效点。

### 8.1 中高速段（反电动势法）

- **滑模观测器（SMO）**：基于滑模变结构，控制律 `sign(s)`，再低通滤波得到反电动势，最后 PLL 提取转速/角度。
- **扩展卡尔曼滤波（EKF）**：把电机作为状态空间模型（含 R、L、ψf 估计），最优估计转速与位置。计算量大，AURIX/TC264 能跑。
- **龙伯格观测器（Luenberger）**：线性化模型，工程实用。
- **MRAS**：基于模型参考自适应，简单稳定。

### 8.2 零低速段（凸极 / 高频注入）

- **脉振高频注入（HFVI）**：注入 `udh = Uh·sin(ωh·t)` 到 d 轴，q 轴响应电流含转子位置信息，调制/解调后用 PLL/Luenberger 提取。
- **旋转高频注入（HFVI-Rotating）**：在 αβ 上注入旋转矢量，靠凸极性提取位置。
- **INFORM（Indirect Flux detection by Online Reactance Measurement）**：注入离散电压脉冲，测瞬时电流响应。

### 8.3 全速域无传感器切换

- 一般 **低速 HFVI** → **中高速 SMO/EKF** 之间的加权切换（斜坡过渡 50–200 rpm）
- 需要做 **权重函数 + 滤波器** 保证过渡平滑

### 8.4 参数鲁棒性

- 温度升高 → `ψf, R` 变化 → 观测器收敛点漂移
- 常用 **在线参数辨识**（递推最小二乘 RLS、模型参考自适应）实时更新 ψf、R

---

## 9. 启动策略：预定位 / I/F / 高频注入

电机启动时转速为零，反电动势为零，纯 SMO 无法启动，必须有专门的启动策略。

### 9.1 预定位（Pre-alignment）

- 给 `id = Imax, iq = 0`，强制转子转到 d 轴位置（持续时间 ~100–300 ms）
- 适合：低惯量负载（泵、风扇）
- 缺点：大电流冲击、噪音

### 9.2 I/F（电流/频率）启动

- 给定一个虚拟转速斜坡，电流矢量以同步速度旋转
- 当实际反电动势建立起来后切到 SMO
- 实现简单，对凸极率不敏感，适合大多数工业 PMSM

### 9.3 高频注入启动

- 用 HFVI 同时获得位置与速度
- 全速域无传感器，工程体验最好
- 缺点：依赖电机凸极性（IPM 强，SPM 弱，SPM 需做饱和凸极注入）

### 9.4 开环 V/F

- 仅做电压幅值与频率的斜坡，不控电流
- 简单但扭矩不可控，常用于风机、压缩机（对启动冲击不敏感）

---

## 10. 弱磁控制与 MTPV

### 10.1 为什么需要弱磁

基速以上，反电动势超过母线电压，没有更多电压空间加电流 → 必须通过 **id < 0**（去磁）降低合成反电动势：

```
ω · (Ld·id + ψf) ≤ Umax
```

### 10.2 MTPA（Maximum Torque Per Ampere）

基速以下的最优电流轨迹：

- **SPM**：`id = 0`
- **IPM**：解方程 `∂T/∂id = 0`（最小电流产生最大转矩），常见公式：
  ```
  id_MTPA = ψf / (2·(Lq − Ld)) − sqrt( (ψf / (2·(Lq − Ld)))² + iq² )
  ```

### 10.3 弱磁 I 区 / II 区

- **I 区**：电压椭圆与电流圆相交，靠调节 `id` 让 `iq` 自然变化
- **II 区（MTPV）**：电压椭圆切于转矩曲线，转速再高，扭矩以 `1/ω²` 衰减

### 10.4 实现方式

- **前馈查表**：基于电机 MAP 提前算好，离线生成 id/iq 指令表
- **在线闭环**：电压 PI 调节 `id`，MTPV 时切换到 `iq` 闭环
- **电压闭环（Voltage Loop）**：实时计算 `√(ud² + uq²)`，与 `Umax` 比较，差值送 PI 生成 `id` 补偿量

---

## 11. 电流采样方案对比

| 方案 | 采样电阻 | 适用功率 | 优点 | 难点 |
|---|---|---|---|---|
| 三电阻 + 低端采样 | 3 | 中小功率 (<5 kW) | 简单、噪声低、电流连续可读 | 需同步采样、低边开关毛刺 |
| 双电阻 + 低端采样 | 2 | 中小功率 | 省一颗采样电阻 | 需重构第三相，单采样点丢失时丢控制 |
| 单电阻 + 低端采样 | 1 | 小功率 (<1 kW) | BOM 最低 | 必须错开 PWM 死区，零电流附近无法采样 |
| 三电阻 + 高端采样 | 3 | 大功率 | 共模电压低、ADC 范围可用 | 高边开关毛刺、隔离 |
| 集成电流传感器（霍尔/Fluxgate） | 0 | 中大功率 | 隔离、抗扰强 | 成本高、带宽有限 |
| Shunt + 隔离运放（如 AMC1301） | 1–3 | 大功率 | 隔离、共模范围大 | 隔离电源、延迟 |

**采样时刻**：一般放在 PWM **中心点**（对称采样），此时开关毛刺最小，且相邻 PWM 周期电流对称。

**AutoSAR 配置**：
- `McuClockSetting` + `GtmAtom` 输出中心触发 ADC
- ADC 用 `AdcGroup` 模式，硬件触发、HwTrg 信号源 = PWM center
- 采样完成 → DMA → 缓冲区 → 中断 → FOC 函数

---

## 12. FOC 周期、执行时间与 MCU 选型

### 12.1 典型时序

| 控制环节 | 周期 | 说明 |
|---|---|---|
| PWM | 10–50 µs（20–100 kHz） | 受 MOSFET/IGBT 开关损耗限制 |
| ADC 采样 | 与 PWM 同步 | 中心点采样 |
| FOC 计算 | 一个 PWM 周期 | < 70% 周期，否则截断/降级 |
| 转速环 | 1 ms | 通常每 10 个 PWM 周期算一次 |
| 位置环 | 1–4 ms | 由外环调用 |

### 12.2 执行时间预算

以 16 kHz FOC 为例，单次 FOC 允许 < 50 µs。其中：

- Clark/Park/反 Park：~1–3 µs
- SVPWM：~1–2 µs
- 电流 PI + 解耦：~2–4 µs
- 观测器（SMO/EKF）：~5–15 µs
- 通讯（CAN/Eth）：异步

### 12.3 选型考量

| 关键能力 | 最低要求 | 推荐 |
|---|---|---|
| 内核 | Cortex-M4F / RH850 G3M | 双核 lockstep（AURIX TC3xx） |
| FPU | 单精度 | 双精度优先 |
| CORDIC | 有则佳 | AURIX 内置 |
| PWM 模块 | 6 路 + 死区 + 中心触发 | GTM-TOM / CCU6 / FlexPWM |
| ADC | 同步采样 3 路 12 bit ≥ 1 MSPS | 16 bit + 过采样 |
| 编码器接口 | SPI / SSI / ABZ | EnDat / BiSS |
| CAN FD | 至少 1 路 | 2 路以上 + 以太网 |
| 功能安全 | ASIL-B | ASIL-D（EPS / 主驱） |
| 封装 | QFP / BGA | 与域控集成度匹配 |

**国内典型**：旗芯微 FC4150、芯弦 CX3288、灵动 MM32SPIN、峰岹 FU6812、云途 YTM32B1M、芯旺微 KF32A1xx。

**国外典型**：Infineon AURIX TC3xx、ST STM32G4/STellar、NXP S32K3xx、Renesas RH850/U2A。

---

## 13. FOC 在 AutoSAR 中的软件架构

### 13.1 MCAL 层（GTM/CCU6/ADC/PWM）

AutoSAR Classic 的 MCAL（微控制器抽象层）把芯片外设封成统一 API：

| 模块 | API 示例 | FOC 用途 |
|---|---|---|
| `Pwm` | `Pwm_SetDutyCycle`, `Pwm_SetPeriodAndDuty` | 输出三相 PWM |
| `Pwm` (DTM) | `Pwm_SetDeadTime` | 设置死区 |
| `Adc` | `Adc_StartGroupConversion`, `Adc_ReadGroup` | 启动电流/母线电压 ADC |
| `Icu` (EdgeDetect) | `Icu_GetEdgeCount`, `Icu_GetTimeStamp` | Hall / 编码器零位捕获 |
| `Gpt` | `Gpt_StartTimer`, `Gpt_GetTimeElapsed` | 转速环 1 ms 周期定时 |
| `Spi` | `Spi_ReadWrite` | 编码器（SSI/BiSS）、磁隔离通信 |
| `Port` / `Dio` | 通用 GPIO 控制 | 继电器、Enable、Brake |
| `Mcu` | `Mcu_Init`, `Mcu_PerformReset` | 时钟、看门狗、复位源 |
| `Wdg` | `Wdg_SetMode`, `Wdg_Trigger` | 看门狗 |

**GTM-TOM 配置示例（AURIX）：**
- TOM0_CH0..CH2：输出三相 PWM（带 DTM 死区）
- TOM0_CH3：中心点触发 ADC
- TIM0：编码器捕获 / RPM 测量

### 13.2 BSW 与复杂驱动 CDD

**FOC 本身的强实时算法通常放在 CDD（Complex Device Driver）层**，理由：
1. **跨层调度**：FOC 需要在一个 PWM 周期（10–50 µs）内完成，传统 SWC 调度粒度（OS Tick 1 ms）无法满足。
2. **直接访问寄存器**：常需要写 MCAL 没覆盖的寄存器（如 GTM 高级触发、CCU6 中断联动）。
3. **与硬件紧耦合**：电流采样、PWM 装载、故障刹车（Brake）必须同步。

**CDD 接口设计**：
- **输入接口（CDD 上层调用）**：`Cdd_Motor_Init()`, `Cdd_Motor_SetTorque(int16 t)`, `Cdd_Motor_Start()`, `Cdd_Motor_Stop()`
- **输出接口**：通过 RTE 暴露给 ASW（Application SWC）：
  - `MotorSpeed_rpm`, `MotorCurrent_A`, `MotorTemp_C`, `MotorState_e`
  - `MotorFault_e`（如 `NO_FAULT / OVER_CURRENT / OVER_TEMP / STALL`）
- **配置（EcuC / XDM）**：电机参数（`PolePairs=4`, `Ld=8e-3`, `Lq=12e-3`, `Psi_f=0.05`）、PI 参数、采样电阻、PWM 频率

### 13.3 RTE 与 SWC

**应用层 SWC 示例**：
- `MotorAppSWC`：接收 `TorqueRequest_Nm`（如 5 Nm），调用 `Cdd_Motor_SetTorque()`
- `DiagSWC`：接收 `MotorFault_e`，写入 DTC（`Dcm / Dem`）
- `NmSWC`：与整车 CAN NM 协调，进入 BusSleep 时调 `Cdd_Motor_Disable()`

**通信接口（Sender/Receiver）**：
```c
// ARXML 片段（简化）
<SENDER-COMPOSITION>
  <SHORT-NAME>Rte_MotorAppSWC</SHORT-NAME>
  <DATA-PORT>...TorqueRequest...</DATA-PORT>
</SENDER-COMPOSITION>
<RECEIVER>
  <SHORT-NAME>Cdd_Motor</SHORT-NAME>
</RECEIVER>
```

### 13.4 OS 调度与多核并行

**典型任务设计（AURIX TC3xx 双核 lockstep 示例）**：

| Task | Priority | Period | Core | 内容 |
|---|---|---|---|---|
| `FOC_Task` | 最高（中断） | 50 µs（PWM 同步） | CPU0 | FOC 计算 + SVPWM |
| `SpeedLoop_Task` | 高 | 1 ms | CPU0 | 转速 PI |
| `App_Task` | 中 | 10 ms | CPU0/1 | 整车扭矩解析、诊断 |
| `Com_Task` | 中 | 10 ms | CPU0 | CAN 信号收发 |
| `Diag_Task` | 低 | 100 ms | CPU1 | UDS / DTC 处理 |

**中断源**：
- ADC 转换完成中断 → 触发 `FOC_Task`（OS 周期任务的 **alarm trigger**）
- GTM 中断 → 重装载占空比

**多核同步**：FOC 主核完成计算后通过 **Spinlock** 或 **共享变量 + Intercore Interrupt** 通知另一个核。

### 13.5 整车集成视角

```
              ┌──────────────────────────────────────────┐
              │              ASW (Application SWC)       │
              │  MotorAppSWC / DiagSWC / VCU-AppSWC     │
              └────────────┬─────────────────────────────┘
                           │ RTE
              ┌────────────▼─────────────────────────────┐
              │           BSW (CDD Complex Driver)        │
              │   FOC 算法、SVPWM、观测器、故障检测       │
              ├────────────┬─────────────────────────────┤
              │   MCAL     │   Services (OS / Wdg / Dem) │
              │ Pwm/Adc/Icu│                             │
              └──────┬─────┴────────┬────────────────────┘
                     ▼              ▼
                 GTM/ADC/Hall    AURIX TC3xx
```

---

## 14. FOC 的功能安全（ISO 26262）

### 14.1 ASIL 等级

- **EPS（转向助力）**：ASIL-D（最高）
- **电动空调/水泵/油泵**：ASIL-B（部分 ASIL-D）
- **主驱（800V SiC）**：ASIL-D
- **ESC 液压泵**：ASIL-D

### 14.2 典型故障与缓解

| 故障 | 后果 | 缓解机制 |
|---|---|---|
| 电流传感器短路/断路 | 控制失效 → 失控 | 双通道冗余、三取二、Range Check |
| 编码器掉线 | 位置错 | 切换无传感器、立即 Disable |
| 过流 / 短路 | 烧毁 IGBT | Hardware OC（比较器） + SW OC（ADC） |
| 过温 | 失磁 / 烧机 | NTC 采样 + 模型降功率 |
| 欠压（电池掉电） | 中途关断 | LVBD（Low Voltage Battery Detect） |
| MCU 失效 | 不响应 | 看门狗 + 硬件 Brake |
| 软件算错 | 失控 | 双重计算 / 时序监控 |

### 14.3 EOC（End of Cycle）监控

每个 FOC 周期都要自检：
- **指令与反馈一致性**：`iq_ref − iq_fbk` 超阈值 → 故障
- **看门狗**：若 1 ms 内 FOC 没跑完 → 复位
- **PWM 测试**：周期性反插测试脉冲，验证 PWM 输出是否被粘连

### 14.4 FTTI 与安全状态

- **FTTI**（Fault Tolerant Time Interval）：一般 1–10 ms（取决于电机响应）
- **安全状态**：高阻态 / Active Discharge / Mechanical Brake

---

## 15. 故障诊断与 UDS/DTC

### 15.1 常见 DTC

| DTC 代码（示例） | 含义 |
|---|---|
| P0A01 | IGBT/MOSFET 短路 |
| P0A02 | 电机过温 |
| P0A03 | 母线过压 |
| P0A04 | 母线欠压 |
| P0A05 | 编码器信号丢失 |
| P0A06 | 电流传感器偏差 |
| P0A07 | Hall 信号异常 |
| P0A08 | 控制器过温 |
| P0A09 | 预充电失败 |
| P0A0A | 软件 watchdog 复位 |

### 15.2 故障等级

- **Class A**：可降功率运行（如电机过温）
- **Class B**：本周期禁用（如过流）
- **Class C**：永久禁用，需诊断仪清码

### 15.3 UDS 服务

- `0x19 0x04` ReadDTCInformation：上报 DTC 状态（TestFailed、ConfirmedDTC、Pend-ing、TestNotCompleted）
- `0x14` ClearDTC：清码
- `0x85` ControlDTCSetting：诊断期间冻结 DTC 设置
- `0x31` RoutineControl：用于执行 EOL 测试（如相序检测、霍尔对中、Encoder offset 自学习）

---

## 16. 常用开发工具链

| 工具 | 厂商 | 特点 |
|---|---|---|
| **MATLAB/Simulink** | MathWorks | 业界标准，代码生成 Embedded Coder |
| **STM32 MC SDK** | ST | 免费，五步生成代码 |
| **AURIX MC-ISAR / MCAL** | Infineon | 配套 Visual Studio + Tasking / HighTec |
| **Vector MICROSAR** | Vector | AutoSAR + 电机 CDD |
| **ETAS ASCET / COSYM** | Bosch / ETAS | 模型开发、AutoSAR 集成 |
| **MathWorks Motor Control Blockset** | MathWorks | 直接生成 TI/ST/Infineon 代码 |
| **S32 Design Studio** | NXP | S32K3 + FreeMASTER 调参 |
| **永磁同步电机调试助手**（国内多款） | 国内厂商 | 上位机曲线 + 在线调参 |
| **灵思 PID Tuner / 云途 MC Tools** | 国产 | 简化入门 |

---

## 17. 定点化与优化

### 17.1 何时用定点

- 成本敏感 MCU 无 FPU（Cortex-M0、R5F 内核低端）
- 功能安全要求 **确定性执行时间**（浮点库函数不可重入）

### 17.2 常用 Q 格式

- `Q15`（16 bit，1 bit 符号）：范围 [-1, 1)，分辨率 3.05e-5
- `Q24`（32 bit，8 bit 整数位）：适合物理量乘以比例因子后
- 角度用 `Q0.16`（0–2π → 0–65535）

### 17.3 优化技巧

1. **三角函数查表**：256 项 / 1024 项 sin/cos 表 + 双线性插值
2. **atan2 近似**：CORDIC 或分段多项式
3. **PI 控制器整形**：增量式 + 输出限幅 + 抗饱和
4. **观测器周期降低**：SMO 不必每周期更新，可 2–4 周期一次
5. **DMA + 双缓冲**：ADC 完成后 DMA 自动填缓冲，无需 CPU 干预
6. **中断合并**：Hall/编码器中断合并到 PWM 中断尾
7. **编译器优化**：GCC `-O2 -ffast-math`；Tasking `-O2 -cpu=tc1.6.2`

---

## 18. 典型应用场景

| 场景 | 电机 | FOC 要点 |
|---|---|---|
| **EPS 转向助力** | PMSM 6/8 极 | 高带宽、齿槽力补偿、ASIL-D、低噪声 |
| **电动空调压缩机** | PMSM 高转速 (>8 krpm) | 高速弱磁、Bearingless 控制、深度弱磁 |
| **EPS 主驱冗余** | 双绕组 PMSM | 三相冗余、Fault Tolerant |
| **EPB 电子驻车** | 有刷直流 / BLDC | 短时大电流、堵转检测 |
| **ESC 液压泵** | BLDC/PMSM | 压力闭环、高频压力脉动抑制 |
| **水/油泵** | BLDC | 单电阻采样、V/F 启动、简单可靠 |
| **主动悬架作动器** | 直线 PMSM（音圈） | 位置伺服、低迟滞、力控 |
| **车载充电机（OBC）** | 不适用 | PFC + DC/DC，与电机 FOC 不同 |
| **增程器 ISG** | PMSM | 启动发电一体（CSG/ISG）、启停控制 |
| **主驱电机控制器（MCU）** | Hair-pin 永磁 / 感应 | 800V SiC、扭矩安全 ASIL-D、环面控制 |

---

## 19. 调参方法论与经验值

### 19.1 电流环调参步骤

1. 把电机堵转（mechanically locked），给 id 一个小阶跃
2. 调 d 轴 PI：从 P 增益开始加，直到响应快但不振荡；再加 I 消除稳态误差
3. 同理调 q 轴
4. 解耦项：把 Ld、Lq、ψf 准确填入，看响应是否变快
5. 解开堵转，验证低速运行

### 19.2 经验参数起点

```
// 8 极 PMSM / 10 krpm / 48V / 5 Nm
PWM Freq        = 16 kHz
FOC Period      = 62.5 µs
Ts (电流环)     = 62.5e-6 s
L               = 10 mH
R               = 0.5 Ω
Kp_i            = L / Ts = 0.16  （简化模型）
Ki_i            = R / Ts = 8000
```

> 注意这只是粗糙起点，最终仍需实测微调。

### 19.3 转速环

- Kp 调至轻微欠阻尼（小幅阶跃 0→100 rpm，超调 5–10%）
- Ki 调到稳态误差 < 1 rpm

### 19.4 弱磁区

- 用示波器观察母线电压与调制比（`m = √(ud²+uq²) / (Udc/√3)`）
- 接近 1 时表示进入弱磁
- 检查 SVPWM 是否发生 **过调制裁剪**，若有，增大弱磁 id 补偿

---

## 20. 进阶话题：自适应 / 预测控制 / AI 调参

### 20.1 在线参数辨识
- **RLS（递推最小二乘）**：辨识 R、L、ψf、J、B（转动惯量、阻尼）
- **MRAS** 简化版：基于转矩观测反推参数
- **注入扰动法**：在 d/q 上叠加小幅正交扰动，从响应提取阻抗

### 20.2 预测控制
- **MPC（FCS-MPC）**：直接选择使代价函数最小的开关状态，无传统 PI；动态极快，但开关频率不固定
- **DPCC（Deadbeat Predictive Current Control）**：一拍控制，需准确模型

### 20.3 AI / 数据驱动
- **神经网络调参**：把 PI 增益当作神经网络输出，离线训练
- **强化学习扭矩分配**：多电机协调（如差速转向）
- **故障预测**：基于电流/温度时序数据预测轴承失效

### 20.4 与整车协同
- **VCU ↔ MCU**：扭矩闭环、模式切换（Drive / Regen / Coast）
- **域控集中控制**：800V 域控直接把扭矩指令经 ETHERNET → SOME/IP 发给 MCU
- **功能安全冗余**：主驱 MCU + 域控安全核双校验

---

## 21. 总结与推荐资料

### 21.1 关键要点回顾

1. FOC 是现代电机控制的事实标准，坐标变换 + SVPWM + 级联 PI + 解耦是核心。
2. PMSM 是车载主流电机，IPM 走 MTPA + 弱磁联合控制。
3. 启动策略、采样方案、观测器选型需根据电机和工况定制。
4. AutoSAR 集成里，**FOC 主体在 CDD**，MCAL 提供 PWM/ADC/ICU，RTE 暴露给 ASW。
5. 功能安全（ISO 26262）下需双通道 + 监控 + 故障刹车，EPS / 主驱需 ASIL-D。
6. MCU 选型看 FPU / CORDIC / GTM-TOM / 多核 lockstep / 16 bit ADC / 编码器接口。
7. 调参从电流环 → 转速环 → 位置环，先内后外，先 P 后 I，弱磁区单独验证。

### 21.2 推荐学习资源

**经典书籍**
- 《Vector Control of Three-Phase AC Machines》— Nguyen Phung Quang
- 《电机控制（第三版）》— 王成元
- 《永磁同步电机控制系统》— 阮毅
- 《Modern Power Electronics and AC Drives》— Bimal K. Bose

**行业标准**
- ISO 26262（功能安全）
- ISO 21434（信息安全）
- AUTOSAR Classic Platform Release R21-11
- IEC 61508（基础安全）
- GB/T 18488（电动汽车驱动电机）

**在线课程**
- MathWorks Motor Control 培训
- Infineon AURIX 电机控制培训
- Bilibili：搜"电机控制""FOC""SVPWM"有大量国内高校 / 工程师内容

**开源代码**
- STM32 MC SDK（GitHub）
- TI InstaSPIN（MotorWare）
- ODrive（开源伺服）

**社区**
- 电子技术应用（ChinaAET）
- EEWorld 电机控制板块
- 知乎"电机控制"话题

---

> **本文版本**：v1.0
> **适用 AutoSAR 版本**：Classic R21-11（兼顾 R4.4）
> **更新日期**：2026-08-13