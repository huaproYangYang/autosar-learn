# Vector DaVinci 工具链详解 —— AUTOSAR 开发平台架构与原理

> 面向人群：有互联网后端开发经验，正在进入车载嵌入式 / AUTOSAR 开发
> 目标：用熟悉的"全栈开发工具链"类比 Vector DaVinci，看懂每个工具干什么、怎么联动、配置怎么变成可执行代码
> 读者收益：能听懂"DaVinci Configurator Pro"、"RTE Generator"、"ARXML Extract"这些术语，能在一个 DaVinci 项目中找到自己的位置

---

## 写在前面：为什么做车载绕不开 DaVinci？

在互联网后端，你的工具链长这样：

```
VSCode / IDEA ──► 写代码 ──► Maven/Gradle 构建 ──► JVM/Go runtime ──► 服务跑起来
       │              │             │                       │
   ESLint 检查     Git commit    Docker 镜像            K8s 调度
```

在车控的 **AUTOSAR Classic** 项目里，主流工具链是 Vector 家的 **DaVinci 系列**，它在你写下任何 `*.c` 代码之前，已经把整个 ECU 的"骨架"搭好了：

```
DaVinci Configurator Pro    DaVinci Developer     编译器（IAR/GHS）
       │                          │                       │
   配置 BSW 模块              写 SWC 业务代码        编译 RTE + BSW + ASW
       │                          │                       │
       ▼                          ▼                       ▼
   生成 ARXML              写 C 代码（Runnable）        .elf / .hex 刷写到 ECU
       └─────── 联调 / 测试（CANoe）──────────────────┘
```

**DaVinci 本质上是一个"配置驱动的代码生成 + 集成开发环境"**，把 AUTOSAR 标准里几百个 BSW 模块的配置变成图形界面，再把配置生成出 ARXML 和 C 代码。

> **后端类比**：Vector DaVenci ≈ IntelliJ IDEA 全家桶（IDEA + Maven + Tomcat + K8s 插件）= 一站式 Java 后端工具链。

---

## 一、Vector 与 DaVinci 家族是什么？

### 1.1 Vector Informatik（公司）

- **总部**：德国 Stuttgart
- **成立**：1988 年
- **地位**：全球汽车 ECU 工具链市占率第一（约 50%+）
- **核心客户**：几乎所有欧系 OEM（VW、BWM、Audi、Porsche、Daimler），以及大部分中国 Tier1
- **官网**：[vector.com](https://www.vector.com)

### 1.2 DaVinci 不是单一工具，是一整套

```
┌─────────────────────────────────────────────────────────────┐
│                  Vector DaVinci 家族                        │
├─────────────────────────────────────────────────────────────┤
│  DaVinci Configurator Pro    BSW + ECU 配置（图形化）        │
│  DaVinci Configurator Classic  旧版配置器                   │
│  DaVinci Developer            ASW / SWC 开发 IDE            │
│  DaVinci Adaptive Designer    Adaptive AUTOSAR 设计         │
│  DaVinci Safe                  功能安全工具                  │
│  CANoe / CANalyzer             总线仿真与测试                │
│  vADASdeveloper                高级驾驶辅助仿真              │
│  CANape                        标定与测量                    │
│  Indigo                         诊断上位机                   │
└─────────────────────────────────────────────────────────────┘
```

> **后端类比**：Vector 的产品矩阵 ≈ JetBrains 全家桶（IDEA + WebStorm + DataGrip + YouTrack）。一个 OEM 一旦买了 Vector，基本全栈都用它。

### 1.3 DaVinci 在 AUTOSAR 工具链中的位置

```
        OEM 系统工程师（OEM 内部）
              │
              ▼
        ┌──────────────┐
        │ System Design │ ← 用 PREEvision / SystemDesk 写系统级 ARXML
        └──────┬───────┘
               │  系统 ARXML（信号 / 端口 / 总线）
               ▼
        ┌──────────────┐
        │ ECU Extract  │ ← 抽出某个 ECU 自己的 ARXML
        └──────┬───────┘
               │
        ┌──────▼───────────────────────────────────┐
        │           DaVinci 工具链                │
        │  ┌────────────────────────────────┐     │
        │  │ DaVinci Configurator Pro       │     │
        │  │  - 配置 BSW（COM/CanIf/OS/...） │     │
        │  │  - 配 SWC 端口 / RTE 视图      │     │
        │  └────────────────────────────────┘     │
        │  ┌────────────────────────────────┐     │
        │  │ DaVinci Developer              │     │
        │  │  - 写 SWC C 代码               │     │
        │  │  - Runnable 逻辑               │     │
        │  └────────────────────────────────┘     │
        └──────┬───────────────────────────────────┘
               │  编译（IAR / GHS / Tasking）
               ▼
        ┌──────────────┐
        │   .elf / .hex │ ← 刷写到 ECU
        └──────┬───────┘
               │
               ▼
        ┌──────────────┐
        │   CANoe /    │ ← 总线仿真 + 测试
        │   HIL 测试   │
        └──────────────┘
```

---

## 二、整体架构：三层工具职责

把 DaVenci 家族拆开看，每一层职责非常清晰：

```
┌──────────────────────────────────────────────────────────┐
│ ① 系统层（System Level）         ── 不在 DaVenci 内     │
│    PREEvision / SystemDesk / Rhapsody                    │
│    描述整车：哪些 ECU、有哪些信号、走哪条总线            │
└──────────────────────────────────────────────────────────┘
                       │  ARXML
                       ▼
┌──────────────────────────────────────────────────────────┐
│ ② ECU 配置层（ECU Level）        ── DaVenci 主力        │
│    DaVenci Configurator Pro + Developer                 │
│    描述一个 ECU：BSW 模块如何配置、SWC 端口如何连接       │
│    ↓ 生成 ARXML                                          │
│    ↓ 生成 RTE / BSW C 代码                                │
└──────────────────────────────────────────────────────────┘
                       │  C 代码 + 编译
                       ▼
┌──────────────────────────────────────────────────────────┐
│ ③ 集成与测试层（Integration & Test）── CANoe / VT System │
│    网络仿真、HIL 测试、自动化测试                        │
└──────────────────────────────────────────────────────────┘
```

**后端类比**：

| 层 | 后端类比 |
|---|---|
| 系统层 | 系统架构师画微服务依赖图（StopLight/ArchiMate） |
| ECU 配置层 | Spring Cloud Config + IDEA 开发 |
| 集成测试层 | WireMock + JMeter + K8s staging |

---

## 三、DaVinci Configurator Pro（最核心工具）

### 3.1 定位

**DaVenci Configurator Pro** = 图形化 BSW + ECU 配置工具。打开一个 ECU 工程，你会看到：

```
左侧树状导航（BSW 模块）
├── ECU Configuration
│   ├── ECU Instance
│   ├── Communication Stack
│   │   ├── Can
│   │   ├── CanIf
│   │   ├── CanTp
│   │   ├── PduR
│   │   └── Com
│   ├── Diagnostic Stack
│   │   ├── Dem
│   │   ├── Dcm
│   │   └── FiM
│   ├── Memory Stack
│   │   ├── NvM
│   │   ├── MemIf
│   │   └── Fee
│   └── System Services
│       ├── Os
│       ├── EcuM
│       ├── BswM
│       └── WdgM
├── Software Components
│   ├── Swc_VehicleControl
│   ├── Swc_BodyControl
│   └── ...
└── RTE
    └── Connections
```

### 3.2 它做什么？

**核心：把 ARXML 变成可视化配置，把可视化配置再生成代码和 ARXML。**

| 输入 | 处理 | 输出 |
|---|---|---|
| 系统级 ARXML（OEM 给的） | 解析、抽取 | ECU 级 ARXML |
| 工程师手动配置（GUI） | 校验一致性 | 校验通过的 ARXML |
| BSW 模块描述（vendor 提供的） | 驱动配置项 | 生成 BSW 源码（.c/.h） |

### 3.3 一个具体例子：配置 CAN 通信

后端同学熟悉的"配置驱动开发"，我们看看 DaVenci 怎么配一条 CAN 消息：

```
1️⃣ 新建一个 CAN 通道
   Can / CanController / CanHardwareObject
   → 配波特率 500kbps、CAN ID 0x123、发送/接收

2️⃣ 定义 PDU（协议数据单元）
   CanIf / CanIfPduCanId
   → 把 CAN ID 绑定到 PDU 句柄

3️⃣ 路由 PDU 到上层
   PduR / PduRRoutingPath
   → "这个 PDU 从 CanIf 出来，往 Com 走"

4️⃣ 定义信号
   Com / ComSignal
   → "车速信号"：位宽 16bit、起始位 0、字节序 little-endian

5️⃣ 把信号映射到 PDU
   Com / ComSignalToPduMapping
   → 哪个信号在哪个 PDU 的哪个位置

6️⃣ 把信号映射到 SWC 端口
   RTE / SignalMapping
   → "车速信号" → Swc_Dashboard 的 VehicleSpeed 端口
```

**全程图形界面操作**，最后导出 BSW 配置代码 + ARXML：

```
Generated/
├── Com/Com.c
├── Com/Com_PBcfg.h        ← 配置生成的结构体
├── PduR/PduR_PBcfg.c
├── CanIf/CanIf_PBcfg.c
└── Rte/Rte.c
```

> **后端类比**：DaVenci Configurator Pro ≈ **Spring Cloud Config Server + Nacos**：用 YAML/GUI 配置服务依赖，运行时从配置中心拉取。它生成的不是 Java Bean，而是 C 结构体 + 初始化代码。

### 3.4 关键概念：Post-Build Configuration（PBcfg）

Vector 工具链里的核心思想：**配置和代码分离**。

```c
/* Com_PBcfg.h（工具生成，不可手动改） */
typedef struct {
    uint16 ComSignal_VehicleSpeed_Handle;
    uint16 ComSignal_VehicleSpeed_BitPosition;
    uint16 ComSignal_VehicleSpeed_BitSize;
    /* ...几百个字段 */
} Com_ConfigType;

extern const Com_ConfigType Com_Config;  /* 链接时指向真正的配置 */

/* Com.c（手写逻辑） */
void Com_Init(const Com_ConfigType* ConfigPtr) {
    Com_ConfigPtr = ConfigPtr;  /* 拿到配置 */
    /* ...初始化逻辑 */
}
```

> 后端类比：`PBcfg` ≈ `application.yml`，编译时打包进二进制，但和 `*.java` 业务代码解耦。

---

## 四、DaVinci Developer（ASW 开发 IDE）

### 4.1 定位

**DaVenci Developer** = 专门写 SWC 业务代码的 IDE。基于 Eclipse 改装，深度集成 AUTOSAR。

### 4.2 它做什么？

- 创建、编辑 SWC 类型（Component Type）
- 编写 Runnable 的 C 代码
- 调试、运行单元测试
- 与 Configurator Pro 联动，同步 SWC 定义

### 4.3 一打开工程长这样

```
Project Explorer
├── src/
│   ├── Swc_VehicleControl.c
│   │   ├── void R_Main(void)            ← Runnable
│   │   ├── void R_HandleSpeed(void)    ← Runnable
│   │   └── uint16 irv_LastSpeed;       ← Inter-Runnable Variable
│   └── Swc_Dashboard.c
├── Header/
│   ├── Rte_Swc_VehicleControl.h         ← RTE 工具生成
│   └── Com_Signal.h
└── ARXML/
    └── Swc_VehicleControl.arxml         ← Component 定义
```

### 4.4 一个 Runnable 长什么样

```c
/* Swc_VehicleControl.c */
#include "Rte_Swc_VehicleControl.h"

/* Init Runnable，SWC 初始化时调用一次 */
void R_Init(void) {
    /* 可以读 NvM 数据 */
    uint16 lastSpeed;
    if (Rte_Call_NvM_ReadBlock(&lastSpeed) == E_OK) {
        irv_LastSpeed = lastSpeed;
    }
}

/* Server-Runnable，被 Client 调用 */
Std_ReturnType R_SetTorque(float32 torque) {
    if (torque < 0.0f || torque > 100.0f) {
        return E_NOT_OK;
    }
    /* 写到 Sender Port */
    Rte_Write_P_VehicleControl_MotorTorque(torque);
    return E_OK;
}

/* Timing Event 触发的 Runnable */
void R_Main(void) {
    uint16 speed = 0;
    Rte_Read_R_VehicleControl_VehicleSpeed(&speed);
    if (speed > 120) {
        /* 限速逻辑 */
        Rte_Write_P_VehicleControl_MotorTorque(0.0f);
    }
}
```

### 4.5 RTE API 由谁生成？

**RTE API 全是 Configurator Pro / Developer 自动生成的**！你写代码时 IDE 自动补全，函数实现是空壳，由 RTE Generator 在编译前生成：

```c
/* Rte_Swc_VehicleControl.h（生成） */
Std_ReturnType Rte_Write_P_VehicleControl_MotorTorque(float32 value);
Std_ReturnType Rte_Read_R_VehicleControl_VehicleSpeed(uint16* value);
Std_ReturnType Rte_Call_NvM_ReadBlock(uint16* data);
```

> 后端类比：与 Spring 的 `@Autowired` 类似 —— 你声明接口，框架注入实现。AUTOSAR 里你声明 port 接口，RTE 生成实现代码。

---

## 五、完整的 DaVinci 工作流（端到端）

下面是从零到一个可刷写 ECU 软件的全流程：

```
   系统工程师          OEM / Tier1 系统工程师             Tier1 应用工程师           Tier1 集成工程师
       │                       │                              │                            │
       ▼                       ▼                              ▼                            ▼
┌────────────┐          ┌────────────┐                  ┌────────────┐              ┌────────────┐
│ PREEvision │          │   System   │                  │  DaVinci   │              │  DaVinci   │
│            │          │   ARXML    │                  │  Config    │              │  Config    │
│ 整车架构   │ ─────►   │  (整车级)  │  ───ECU Extract─►│  Pro       │ ─── ARXML ──►│  Pro       │
│ ECU 划分   │          │ 信号定义   │                  │  (本 ECU)  │              │ (BSW 配置) │
└────────────┘          └────────────┘                  └────┬───────┘              └────┬───────┘
                                                            │                          │
                                                            │ 配置 BSW                  │
                                                            ▼                          ▼
                                                     ┌────────────┐              ┌────────────┐
                                                     │  生成 BSW  │              │  生成 RTE  │
                                                     │  C 代码    │              │  C 代码    │
                                                     └────┬───────┘              └────┬───────┘
                                                          │                          │
                                                          └──────────┬───────────────┘
                                                                     │
                                                                     ▼
                                                ┌──────────────────────────────────┐
                                                │       DaVinci Developer          │
                                                │  - 写 SWC 业务逻辑                │
                                                │  - Runnable、端口读写、IPC        │
                                                └──────────────┬───────────────────┘
                                                               │
                                                               ▼
                                                ┌──────────────────────────────────┐
                                                │  编译器（IAR / GHS / Tasking）    │
                                                │  → 输出 .elf / .hex              │
                                                └──────────────┬───────────────────┘
                                                               │
                                                               ▼
                                                ┌──────────────────────────────────┐
                                                │       刷写 + 标定                 │
                                                │  (CANape / Vector Flasher)        │
                                                └──────────────┬───────────────────┘
                                                               │
                                                               ▼
                                                ┌──────────────────────────────────┐
                                                │       CANoe / HIL 联调            │
                                                │  自动化测试 / 故障注入             │
                                                └──────────────────────────────────┘
```

---

## 六、RTE Generator：连接 ASW 和 BSW 的桥梁

### 6.1 RTE 是什么？

**RTE (Runtime Environment) = AUTOSAR 的运行时中间件**，是 ASW 调用的所有 API 的实现者。

### 6.2 RTE 由谁生成？

**Vector 的 RTE Generator（rte.exe / rte_gen）**。它读两份输入：

```
输入 1: 系统 ARXML（信号、端口、PDU 总线映射）
输入 2: ECU 抽取 ARXML（Runnable 定义、任务分配）
                  ↓
           RTE Generator
                  ↓
输出: Rte.c + Rte.h（每个 SWC 一份）
      Rte_Internal.c（RTE 内部实现，可能数万行）
      Rte_Cfg.h（编译时宏定义）
```

### 6.3 RTE 生成的关键代码

**生成的 RTE 函数示例**：

```c
/* Rte_Swc_VehicleControl.c（生成） */
Std_ReturnType Rte_Write_P_VehicleControl_MotorTorque(float32 value) {
    /* 写 Sender Port → 触发 COM 发送 */
    return P_Swc_VehicleControl_MotorTorque_SR->value = value;  /* 实际可能更复杂 */
}

Std_ReturnType Rte_Read_R_VehicleControl_VehicleSpeed(uint16* value) {
    /* 读 Receiver Port → 从 COM 队列取 */
    *value = R_R_Swc_VehicleControl_VehicleSpeed->value;
    return E_OK;
}

/* Runnable 调度入口 */
void Rte_MainFunction(void) {
    /* 按 OS 配置触发 Runnable */
    if (TimingEvent_Main_Elapsed) {
        Swc_VehicleControl_R_Main();
    }
    /* ...所有 Runnable 检查 */
}
```

> **后端类比**：RTE ≈ Spring IoC 容器 —— 你声明依赖（Port），RTE 生成注入代码；你声明切面（Runnable 触发条件），RTE 生成调度代码。

### 6.4 RTE 内部核心数据结构

```c
/* RTE 内部维护的通信队列（生成） */
typedef struct {
    uint8  state;        /* NEW / READ / INVALID */
    void*  value;        /* 数据指针 */
    uint32 timestamp;    /* 时间戳 */
} Rte_BufferType;

extern Rte_BufferType Rte_Buffer_Swc_A_VehicleSpeed;
extern Rte_BufferType Rte_Buffer_Swc_B_VehicleSpeed;
```

---

## 七、与编译器 / 链接器的集成

### 7.1 Vector DaVenci 不含编译器，只做配置 + 代码生成

它输出的 C 代码必须用第三方编译器编译。常见组合：

| 编译器 | 厂商 | 特点 |
|---|---|---|
| **IAR Embedded Workbench** | IAR（瑞典） | 业界最广，AUTOSAR 默认 |
| **Green Hills MULTI** | GHS（美） | 安全关键行业最强 |
| **Tasking VX** | Altium（荷） | 欧洲车规常用 |
| **HighTec GCC** | HighTec（德） | 开源 GCC 的车规商业版 |
| **Wind River Diab** | Wind River | 老牌编译 |

### 7.2 编译流程

```
DaVenci 生成的 .c/.h
            │
            ▼
   编译器（IAR / GHS）
            │
            ├── 编译 → .o
            │
            ├── 链接 → .elf / .hex
            │       （链接脚本决定代码放到 Flash/RAM 哪个物理地址）
            │
            └── 调试信息 → .map 文件（地址 → 符号映射）
            │
            ▼
   刷写到 ECU Flash
```

### 7.3 链接脚本（.ld 文件）

车控 MCU 的链接脚本比 Linux 简单得多：

```ld
MEMORY {
    FLASH (rx)  : ORIGIN = 0x00000000, LENGTH = 2M
    RAM   (rwx) : ORIGIN = 0x40000000, LENGTH = 256K
}

SECTIONS {
    .text : { *(.text) } > FLASH
    .data : { *(.data) } > RAM AT > FLASH  /* 启动时从 FLASH 拷到 RAM */
    .bss  : { *(.bss)  } > RAM
}
```

> **后端类比**：嵌入式无 OS 加载器，链接器直接把代码放到物理地址；后端有 OS 加载器，链接器输出的是相对地址。

---

## 八、调试与追踪

### 8.1 DaVenci 与调试器的关系

DaVenci 配置生成代码，但**调试还是要回到编译器配套的调试器**：

| 调试器 | 厂商 |
|---|---|
| IAR Embedded Workbench 内置 | IAR |
| TRACE32 | Lauterbach（业界最强） |
| winIDEA | iSystem |
| UDE | PLS |
| PLS | — |

### 8.2 常见调试场景

```
1) 单步跟踪 Runnable ──► 用 IAR/Lauterbach，下断点在 R_Main
2) 看 Task 调度 ──► 用 OS Awareness 插件，看哪个 Task 在跑
3) 看变量实时值 ──► 用 watchpoint
4) 看 CAN 总线 ──► 用 CANalyzer 配合
5) 看运行时错误 ──► DET 日志 + DEM 故障码
```

---

## 九、DaVenci Adaptive（Adaptive AUTOSAR 部分）

### 9.1 定位

**DaVenci Adaptive Designer** = Adaptive AUTOSAR 的图形化配置工具。

### 9.2 与 Classic 工具的差异

| 维度 | DaVenci Configurator Pro (Classic) | DaVenci Adaptive Designer (Adaptive) |
|---|---|---|
| 操作系统 | OSEK OS（静态配置） | POSIX-like OS（动态配置） |
| 进程模型 | Runnable + Task | Process（运行可执行文件） |
| 通信 | COM / PDU | ara::com（SOME/IP） |
| 配置形态 | C 结构体（PBcfg） | JSON / ara::conf |
| 部署 | 链接进二进制 | 部署 manifest + 可执行 |
| 典型应用 | 车身 / 动力 | 自动驾驶 / 座舱 |

### 9.3 Adaptive 工作流

```
1) DaVenci Adaptive Designer 配 ara::com 服务接口
2) 生成 ara::com IDL (类似 Protobuf)
3) C++ 程序员写 Application 代码
4) 编译 → 可执行文件 + ara.json（部署描述）
5) UCM (Update Config Manager) 推到 ECU
6) ara::exec 启动进程
```

---

## 十、与 ETAS ISOLAR、EB tresos 对比

| 维度 | Vector DaVenci | ETAS ISOLAR | EB tresos |
|---|---|---|---|
| 厂商 | 德 Vector | 德 ETAS（Bosch） | 德 Elektrobit |
| 市场 | 全球 OEM/Tier1 | Bosch / VW 系 | 戴姆勒 / 保时捷 |
| 工具链完整度 | ★★★★★ | ★★★★★ | ★★★★ |
| BSW 集成 | MICROSAR 配套 | RTA 系列 | EB 自家 |
| 学习曲线 | 中等 | 中等 | 较陡 |
| 中国市场 | 主流 | 部分 | 部分 |
| 评价 | 行业标配 | 工业风 | 模块化灵活 |

> **后端类比**：
> - Vector DaVenci ≈ IntelliJ IDEA（商业、整合度高）
> - ETAS ISOLAR ≈ Eclipse（开源基因、大企业风格）
> - EB tresos ≈ VS Code（轻量、模块化）

---

## 十一、Vector 工具链完整生态

Vector 不止 DaVenci，配套工具一起用：

```
工具矩阵（按角色）
═══════════════════════════════════════════════════════════
网络设计 / 协议开发
  • CANdelaStudio      ───► 诊断数据库（CDD/DTC）
  • PREEvision         ───► 整车网络设计（AUTOSAR 系统级）

AUTOSAR 配置
  • DaVenci Configurator Pro  ───► Classic BSW + ECU 配置
  • DaVenci Developer         ───► SWC C 代码开发
  • DaVenci Adaptive Designer ───► Adaptive 配置

仿真与测试
  • CANoe                ───► 总线仿真 + 自动化测试
  • CANalyzer            ───► 总线分析（只听）
  • VT System            ───► HIL 测试硬件
  • vADASdeveloper       ───► ADAS 仿真

诊断与刷写
  • Indigo               ───► UDS 诊断上位机
  • Vector Flasher       ───► ECU 刷写工具
  • DiVa                 ───► 自动化诊断测试

标定与测量
  • CANape               ───► ECU 标定、A2L 解析
  • vADASmeasure         ───► ADAS 数据记录

功能安全
  • DaVenci Safe         ───► ISO 26262 流程工具
  • EXETA                ───► 安全分析

其他
  • MICROSAR (BSW 代码 + RTE + OS)
  • vADASdeveloper
  • vCDM (Central Diagnostics Management)
═══════════════════════════════════════════════════════════
```

---

## 十二、学习 DaVenci 的实战路径

### 12.1 推荐 6 周上手

**第 1 周：装工具、跑通 demo**

- 申请 Vector 试用版（公司 license / 试用 license）
- 跑通 DaVenci 自带 **SimpleCAN Demo**：一个 ECU 接收 CAN 信号并响应

**第 2 周：理解 ARXML 流转：系统 ARXML → ECU Extract → BSW 配置**

- 看 demo 的 ARXML 文件结构
- 学会读 .arxml（用 Vector 的 ARXML Viewer 或 notepad++ XML 插件）

**第 3 周：自己配一个 CAN 通信**

- 用 Configurator Pro 新建 CAN 通道、PDU、Signal
- 绑到 SWC 端口上

**第 4 周：写 SWC 代码**

- 在 Developer 里写 Runnable
- 实现信号读取 + 处理 + 写回

**第 5 周：编译刷写 + CANoe 联调**

- 配 IAR 工程
- 编译出 .hex
- CANoe 仿真另一 ECU，发 CAN 消息，看自己 ECU 反应

**第 6 周：诊断 + 故障**

- 加 DCM/DEM 配置
- 实现一个 UDS 服务（0x22 读 DID）
- 用 Indigo 验证

### 12.2 资源清单

- **官方培训**：Vector 学院开课（5 天线下课，含证书）
- **官方文档**：每模块都有 User Guide（PDF 几十页到几百页）
- **demo 工程**：Vector 安装目录下自带 `Demo/Classic` 系列
- **中文资料**：B 站搜 "Vector DaVenci 教程"，知乎搜 "AUTOSAR 工具"
- **开源替代**：Arctic Core（理解 DaVenci 生成的代码干了什么）

---

## 十三、关键术语表（贴在工位）

| 术语 | 含义 | 后端类比 |
|---|---|---|
| **DaVenci Configurator Pro** | BSW + ECU 配置工具 | Spring Cloud Config |
| **DaVenci Developer** | SWC 开发 IDE | IntelliJ IDEA |
| **DaVenci Adaptive Designer** | Adaptive AUTOSAR 配置 | K8s 部署描述编辑器 |
| **ARXML** | AUTOSAR XML 配置文件 | YAML / JSON |
| **RTE Generator** | 把 ARXML 编译成 RTE C 代码的引擎 | Spring Bean 定义解析器 |
| **PBcfg** | Post-Build Configuration，编译时打包的配置 | application.yml |
| **ECU Extract** | 从系统 ARXML 抽出某个 ECU 的 ARXML | 微服务的 service.yaml |
| **Runnable** | 可被 RTE 调度的函数 | Controller handler |
| **System Description** | 整车级 ARXML | 微服务依赖图 |
| **MICROSAR** | Vector 的 BSW + OS + RTE 套件 | Spring Cloud 全家桶 |
| **CDD** | Complex Driver，复杂驱动 | 厂商私有 native 代码 |
| **DaVenci Safe** | 功能安全工具 | SOC2 审计工具 |
| **CAPL** | CANoe 仿真脚本语言 | WireMock stub |
| **A2L** | ASAM 标定描述 | OpenAPI 描述文件 |

---

## 十四、小结

- **Vector DaVenci** 是 AUTOSAR 行业最主流的工具链，覆盖 Classic + Adaptive
- **核心三件套**：Configurator Pro（BSW 配置）+ Developer（ASW 开发）+ RTE Generator（代码生成）
- **配置驱动**：所有 ECU 配置以 ARXML 为载体，工具链读取 ARXML 生成的 BSW/RTE C 代码
- **后端类比**：Vector DaVenci ≈ IntelliJ IDEA + Maven + Spring Initializr 的车控版
- **学习曲线**：装工具 → 跑 demo → 自己配 CAN → 写 SWC → 编译刷写 → 联调，6 周可上手

把这份文档配合《AUTOSAR 架构详解》一起看，能形成"架构 + 工具"完整图景。

Happy coding & safe driving 🚗