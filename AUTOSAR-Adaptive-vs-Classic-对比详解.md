# AUTOSAR Adaptive vs Classic 详解 —— 全方位对比与选型指南

> 面向人群：正在接触车载 ECU 软件架构的工程师、架构师、面试候选人
> 目标：把 AUTOSAR Classic Platform（CP）和 Adaptive Platform（AP）的差异讲透，从动机、架构、运行机制、工具链到选型决策，一次性梳理清楚
> 读者收益：在面对"该用 CP 还是 AP"、"CP 和 AP 怎么共存"、"为什么智驾/座舱都用 AP"等问题时，能讲清楚技术逻辑与工程权衡

---

## 写在前面：为什么 AUTOSAR 要拆成两套平台？

AUTOSAR 在 2003 年由 BMW、Bosch、Daimler、Continental、Ford、PSA、Siemens、Toyota、Volkswagen 等整车厂与 Tier1 联合发起，初衷是 **降低汽车 ECU 软件的开发复杂度、提升可移植性、降低长期维护成本**。

2017 年，随着 **自动驾驶、智能座舱、车云一体（OTA、V2X）** 等新业务兴起，传统 Classic 平台暴露出三个根本性问题：

| Classic 平台的局限 | 新业务对 ECU 的新要求 |
|---|---|
| 单核 MCU（数十 MHz ～ 数百 MHz） | 多核异构 SoC（数 GHz，CPU+GPU+NPU） |
| 静态配置、固化运行时 | 动态部署、热更新、运行时自适应 |
| CAN/LIN 为主，带宽 < 1 Mbps | 以太网 + SOME/IP + DDS，带宽 100Mbps～10Gbps |
| 单一功能（BCM、EMS、ESP） | 域控制器（智驾域、座舱域、车身域、中央计算单元） |
| 无 POSIX，裸跑 OSEK/Autosar OS | 需要 POSIX 进程、线程、socket、文件 I/O |
| 无安全隔离 | 多应用混跑，需要沙箱 + IAM 权限管理 |

> **关键判断**：不是 Classic "不好"，而是 **它解决不了新场景**。AUTOSAR 选择了"分叉演进"——保留 Classic、补齐 Adaptive，两套并存而非替代。

到 R22-11（2022 年 11 月版）为止，Classic 已稳定到 R4.4，Adaptive 已演进到 R22-11，未来 CP 会进入维护期，**AP 是 AUTOSAR 的主战场**。

---

## 一、定位与目标场景

| 维度 | Classic Platform（CP） | Adaptive Platform（AP） |
|---|---|---|
| 出现时间 | 2003 启动，2008 R3.0 稳定 | 2017 R17-03 首版 |
| 目标硬件 | 微控制器（MCU）—— 单核/多核，RAM < 2MB，ROM < 16MB | 高性能处理器（MPU）—— 多核 SoC，RAM GB 级，Flash GB 级 |
| 典型芯片 | Infineon AURIX TC3xx、NXP S32K、Renesas RH850、TI TMS570 | NXP S32G/S32V、Renesas R-Car、TI TDA4x、NVIDIA Orin/Xavier、高通 8155/8295、地平线 J5/J6 |
| 典型功能 | 车身控制（BCM）、发动机（EMS）、变速箱（TCU）、制动（ESP）、电动转向（EPS）、ADAS 感知前融合之外的部分 | 自动驾驶主控、智能座舱、车云网关、OTA Master、中央计算单元、跨域融合 |
| 应用数量 | 单 ECU 上 5～30 个 SWC | 单 ECU 上数十～数百个 Adaptive Application |
| 实时性 | 硬实时（us 级确定性） | 软实时（ms 级 + 一定 QoS 保证） |
| 算力上限 | < 1 KDMIPS | 数 K～数十 KDMIPS，可调度 GPU/NPU |

**一句话总结**：

> **CP = 强实时、固化、单一功能、控制类 ECU**；
> **AP = 高算力、动态、多应用、计算/通信密集型域控制器**。

---

## 二、整体架构差异

### 2.1 CP 三层架构

CP 是严格的三层静态架构（[AUTOSAR-架构详解-后端转嵌入式.md](./AUTOSAR-架构详解-后端转嵌入式.md) 已有详述）：

```
┌────────────────────────────────────────────────┐
│   应用软件层 (ASW)                              │
│   SWC1  SWC2  SWC3  ... SWCn                  │ ← C 语言，固定算法
└────────────────▲───────────────────────────────┘
                 │ RTE (Runtime Environment)
                 │ ← 编译期生成的"API 网关"
┌────────────────▼───────────────────────────────┐
│   基础软件层 (BSW)                              │
│   Service / ECU Abstraction / MCAL / CDD       │ ← OSEK OS + CAN/LIN/ETH 驱动
└────────────────────────────────────────────────┘
```

**关键特性**：

- 整个系统在编译期由 RTE Generator + BSW Configurator **全部生成**
- OS 是 **OSEK/VDX 派生** 的 `AUTOSAR OS`（SC1/SC2/SC3/SC4 四个可扩展等级）
- 无虚拟内存、无 MMU、无用户态/内核态概念
- 一个 SWC 调用另一个 SWC 的端口，走 **RTE sender/receiver 钩子**，调用开销 ~ 100ns 级

### 2.2 AP 四层架构

AP 引入了 **执行管理（Execution Management）**、**平台健康管理（PHM）**、**持久化（Persistency）**、**IAM（Identity Access Management）** 等新概念，整体架构：

```
┌───────────────────────────────────────────────────────────────┐
│   Adaptive Application (AA)                                   │
│   ├─ Vehicle Speed Service                                    │
│   ├─ Object Fusion Service                                    │
│   ├─ HMI Render Service                                       │
│   └─ OTA Master                                               │
│   (C++17/14, POSIX 进程, 标准 ara:: API)                       │
├───────────────────────────────────────────────────────────────┤
│   ARA (AUTOSAR Runtime for Adaptive) 接口                     │
│   ara::com │ ara::exec │ ara::phm │ ara::per │ ara::sm       │
│   ara::diag │ ara::log  │ ara::iam │ ara::crypto │ ara::core   │
│   (C++ 命名空间，逻辑中间件层)                                  │
├───────────────────────────────────────────────────────────────┤
│   Functional Cluster (FC) 实现                                │
│   com / exec / phm / per / sm / diag / log / iam / crypto     │
│   (操作系统之上的中间件进程 + 共享库)                           │
├───────────────────────────────────────────────────────────────┤
│   Adaptive Platform Foundation                                │
│   POSIX OS (Linux/QNX/AUTOSAR OS) + HW (SoC + 加速器)          │
└───────────────────────────────────────────────────────────────┘
```

**关键特性**：

- AA 是 **独立进程**，各自有独立地址空间
- ARA 只是一组 **C++ 接口规范**，具体实现在底层 FC 中（多数 FC 是独立 daemon 进程 + 客户端库）
- 进程间通信走 **SOME/IP、DDS、共享内存、Unix Domain Socket** 等多种传输
- 应用描述用 **Manifest（manifest.json 等）** 而非编译期 ARXML，**运行时由 EM（Execution Manager）按 Manifest 启动**

### 2.3 一图对比

```
┌──────────────────────────┬──────────────────────────────────┐
│ Classic Platform (CP)    │ Adaptive Platform (AP)           │
├──────────────────────────┼──────────────────────────────────┤
│ 静态：编译期全锁定       │ 动态：运行时按 Manifest 编排      │
│ 三层（ASW/RTE/BSW）      │ 四层（AA/ARA/FC/PF）             │
│ 单 ECU = 一个固定镜像    │ 单 ECU = 一组独立进程            │
│ 无 OS 概念（仅 OSEK 任务）│ 完整 POSIX 进程/线程             │
│ SWC 不能独立更新         │ AA 可独立 OTA 热更新             │
│ CAN/LIN 为主             │ 以太网 + SOME/IP/DDS 为主        │
│ 无安全隔离               │ 应用沙箱 + IAM 权限管控          │
│ RTE Generator 是核心     │ ara::com + ara::exec 是核心      │
│ 工具链：DaVinci CFG Pro  │ 工具链：DaVinci Adaptive + EB    │
└──────────────────────────┴──────────────────────────────────┘
```

---

## 三、操作系统与运行时

### 3.1 CP 的运行时

| 项目 | 内容 |
|---|---|
| OS 核心 | OSEK/VDX 派生的 AUTOSAR OS（OSEK OS + 扩展） |
| 进程概念 | ❌ 没有，所有 SWC 共享地址空间，靠 **Counter / Alarm / Task** 调度 |
| 线程 | BasicTask / ExtendedTask（可挂起、可等待事件） |
| 调度策略 | 完全优先级抢占（Fixed Priority Preemptive Scheduling） + 可选 OSEK OIL 配置 |
| 优先级数量 | SC2/SC3/SC4 支持多优先级（最高可到数百级） |
| 中断 | ISR Category 1（不可调度）/ Category 2（可调度） |
| 内存保护 | SC3/SC4 提供 **OS-Application** + **Trusted/Distrusted** 内存分区，但粒度很粗 |
| 同步原语 | Resource（优先级天花板）、Event、Semaphore、Spinlock |
| 配置方式 | `OIL`（OSEK Implementation Language）或 `ARXML` |

> **CP 的 OS 本质是一个 RTOS 内核**，而 **不是一个通用操作系统**。

### 3.2 AP 的运行时

| 项目 | 内容 |
|---|---|
| OS 核心 | POSIX 操作系统（Linux 通常含 PREEMPT_RT、QNX、Integrity、AUTOSAR OS+POSIX profile） |
| 进程概念 | ✅ 标准 Unix Process，每个 AA 一个或多个进程 |
| 线程 | POSIX Thread（pthread_create/pthread_join），C++ std::thread |
| 调度策略 | POSIX SCHED_FIFO / SCHED_RR / SCHED_DEADLINE（带可执行预算的 EDF） |
| 内存保护 | ✅ MMU 提供完整虚拟内存隔离，进程间无法越界访问 |
| 同步原语 | pthread_mutex（优先级继承 PTHREAD_PRIO_INHERIT）、读写锁、条件变量 |
| 文件系统 | 完整 POSIX 文件 I/O（ara::per 提供持久化封装） |
| 网络 | POSIX Socket → SOME/IP/DDS 走其上 |
| 配置方式 | Application Manifest（JSON）+ Machine Manifest（JSON/ARXML） |

**关键差异**：

- AP 要求 OS 提供 **完整 POSIX 接口**。AUTOSAR 官方推荐 Linux（含 PREEMPT_RT）或 QNX。
- Linux 跑 AUTOSAR AP 时，**内核版本 ≥ 5.15 + PREEMPT_RT 补丁** 是公认基线；QNX 提供 ASIL D 完整认证。
- **AP 不允许在裸芯片上跑**，必须有一个完整的 OS。这是 AP 与 CP 的硬边界。

### 3.3 启动流程对比

**CP 启动**（典型 EMS / BCM ECU）：

```
上电 → BootROM → C-Startup (crt0) → ECU State Manager (EcuM)
     → BswM 初始化 → OS Init → BSW 模块 init (CanIf/Com/NvM/...)
     → RTE Start → SWC Runnable 调度
```

**AP 启动**（典型自动驾驶域控制器）：

```
上电 → BootROM (BL31/BL32/BL33) → U-Boot → Linux Kernel
     → systemd → Execution Manager (EM) 进程启动
     → EM 读取 Machine Manifest + Application Manifest
     → 按 Manifest 编排，启动各 Functional Cluster 守护进程
     → 按 Manifest 启动 Adaptive Application 进程
     → 进入 ara::exec::ExecutionClient::ReportExecutionState()
```

AP 多了一步 **Manifest 驱动的运行时编排**，这是它能"动态部署"的关键。

---

## 四、通信模型

### 4.1 CP 通信：RTE + COM Stack

CP 的核心通信栈是 **CAN/LIN/FlexRay/ETH** 驱动栈：

```
SWC → RTE Port → COM（信号/PDU 打包）→ PduR（PDU 路由）→ CanIf/LinIf
     → CanDrv/LinDrv → Transceiver → 总线
```

**两种交互范式**：

1. **Sender/Receiver（S/R）**——一个 SWC 发，多个 SWC 收，**点对多点**
2. **Client/Server（C/S）**——一个 SWC 是 Server，提供若干 Operation，其他 SWC 是 Client 调用

**S/R 示例（伪代码）**：

```c
// Sender SWC
FUNC(void, MySenderRunnable)() {
    int speed = read_sensor();
    Rte_Send_R_PedalPosition_Speed(&speed);  // 通过 RTE 发送
}

// Receiver SWC
FUNC(void, MyReceiverRunnable)() {
    int speed;
    if (Rte_Read_R_PedalPosition_Speed(&speed) == RTE_E_OK) {
        /* 处理 speed */
    }
}
```

> RTE 在编译期会把这些 send/read 调用替换成 **共享变量拷贝 / 队列发送 / 总线 PDU 发送** 三种实现之一，具体选哪种由 BSW 配置决定。

**CP 通信的痛点**：

- 只能处理 **静态定义好的信号/服务**，增加新接口要重新生成 RTE + 重刷整个 ECU
- 多 ECU 协同需要 OEM 在 System Description 阶段就敲定所有接口
- CAN 总线带宽受限（经典 CAN 500 kbps，CAN FD 2 Mbps），复杂数据建模困难

### 4.2 AP 通信：ara::com + SOME/IP/DDS

AP 引入了 **服务导向（SOA）** 通信，核心是 `ara::com` 命名空间下的三类服务：

| 服务类型 | 类比 CP | 含义 | 适用场景 |
|---|---|---|---|
| **Method** | C/S 的 Operation | 远程过程调用（请求-响应） | 控制指令、参数设置 |
| **Event** | S/R 的 Sender | 发布订阅，订阅者收到通知 | 状态变化、心跳、传感器数据流 |
| **Field** | S/R 的 Notifier + Getter | 由 Event + Getter + Setter 组合 | 服务端维护的状态属性（车速、SoC、温度） |

**AP 通信栈**（默认 SOME/IP，可换 DDS）：

```
AA (ara::com API)
   ↓
ara::com 运行时（服务代理 ProxySkeleton）
   ↓
[ 可选：ara::com binding ] → SOME/IP / DDS / IPC
   ↓
传输层：TCP/UDP（以太网、共享内存）
```

**AP Method 示例（C++17 伪代码）**：

```cpp
// 服务端（SpeedService）
class SpeedServiceImpl : public ara::com::skeleton::ServiceSkeleton {
    ara::com::Future<SpeedResp> GetSpeed(const SpeedReq& req) override {
        int v = ReadSensor();
        return ara::com::MakeReadyFuture(SpeedResp{v});
    }
};
int main() {
    SpeedServiceImpl svc;
    svc.OfferService();   // 服务注册到 Service Registry
    svc.RunProcessingLoop();
}

// 客户端
auto proxy = ara::com::proxy::ServiceProxy::Build<SpeedServiceProxy>();
proxy->GetSpeed(SpeedReq{}).Subscribe([](auto fut) {
    int v = fut.GetResult()->speed;
    UseSpeed(v);
});
```

**Event 示例（发布订阅）**：

```cpp
// 发布者
auto event = ara::com::skeleton::ServiceSkeleton::BuildEvent<LaneEvent>();
event->Send(LaneEvent{left: 1.2, right: 0.7});

// 订阅者
auto sub = ara::com::proxy::ServiceProxy::BuildEvent<LaneEvent>();
sub->Subscribe([](const LaneEvent& e) { Hmi().UpdateLane(e); });
```

### 4.3 对比维度

| 维度 | CP COM | AP ara::com |
|---|---|---|
| 交互模型 | S/R + C/S | Method + Event + Field |
| 总线假设 | CAN/LIN/FlexRay/ETH（PDU 中心） | 以太网 + SOME/IP/DDS（消息中心） |
| 接口定义 | ARXML `SenderReceiverInterface` / `ClientServerInterface` | ARXML `ServiceInterface`（method/event/field） |
| 配置粒度 | 信号-PDU-Frame-Channel 全静态链路 | 服务实例 + endpoint + transport 可在 Manifest 配置 |
| 寻址 | CAN ID / NM 地址 | IP + Port + Service ID + Instance ID（vSOME/IP） |
| 服务发现 | NM（OSEK Network Management），无动态服务发现 | **SOME/IP-SD** 或 DDS Discovery，动态 |
| 多订阅者 | 必须通过 PDU 路由广播 | **天然支持一对多** |
| 序列化 | COM 默认大端位操作 + 信号打包 | SOME/IP 序列化 / CDR（Common Data Representation） |
| QoS | 无（CAN 自身 FC/CRC） | DDS 提供 22+ 种 QoS（可靠性、时效、优先级、历史、过滤等） |

### 4.4 后端类比

| CP | AP | 后端对应 |
|---|---|---|
| SWC | AA | 微服务 |
| RTE | ara::com | gRPC + 服务发现 + 消息队列 |
| ARXML | Manifest | protobuf 描述符 + K8s Deployment YAML |
| 总线信号 | Method/Event/Field | RPC + PubSub |

> **给后端工程师**：AP 的 ara::com **就是车规级 gRPC**，但多了一些车规特有的 QoS（延迟保证、确定性、安全等级）。

---

## 五、应用模型与部署

### 5.1 CP 应用模型

- 应用被建模成 **Software Component (SWC)**，由若干个 **Port** 组成
- Port 又分为 **PPort（ProvidePort）** 和 **RPort（RequirePort）**
- SWC 的内部行为是 **Runnable（可运行函数）**，由 OS 事件、周期性 Alarm、COM 接收事件等触发
- 所有 SWC 在编译期被绑定进 **一个 ELF/HEX 镜像**，运行时是 **一个进程（OS-Application）**
- 增量部署代价极大：**改一行代码 → 重编 BSW → 重生 RTE → 重刷整个 ECU**

```
整个 ECU 镜像 = BSW + RTE + 所有 SWC + OS（一个可执行文件）
```

### 5.2 AP 应用模型

- 应用是 **独立的 Adaptive Application (AA)**，每个 AA 是一个或多个进程
- AA 的接口用 **Manifest** 描述，三类 Manifest 各自承担不同职责：

| Manifest | 作用 | 类比 |
|---|---|---|
| **Machine Manifest** | 描述 ECU 本身：网络接口、ECU 状态、硬件资源 | K8s Node 描述 |
| **Application Manifest** | 描述 AA 的进程模型、依赖、启动顺序、资源需求、状态机 | K8s Deployment YAML |
| **Service Instance Manifest** | 描述服务实例的 endpoint（IP:port、传输协议、安全策略） | K8s Service + Endpoint |
| **Software Cluster Manifest** | 描述一组 AA 的版本、依赖、升级单元（OTA 单元） | K8s Helm Chart / Operator |

**AA 的生命周期状态**（由 `ara::exec` 管理）：

```
kRunning ──┐
           ├──► kTerminating ─► kTerminated
kWaiting ──┘
```

状态迁移通过 `ExecutionClient::ReportExecutionState()` 上报 EM，由 EM 协调整体启动顺序。

**Manifest 片段示例**：

```json
{
  "processes": [
    {
      "name": "FusionApp",
      "executable": "/opt/apps/fusion/fusion_app",
      "startupConfigs": {
        "startupOption": "STARTUP_OPTION_ONE",
        "dependencies": ["com.example.SpeedService", "com.example.RadarService"]
      },
      "resourceConstraints": {
        "cpuQuota": 80,
        "memoryQuota": "512MB"
      },
      "functionGroups": ["PreDriving", "Driving"]
    }
  ]
}
```

> **后端类比**：AA 进程 = K8s Pod；Application Manifest = PodSpec；FunctionGroup = 同一类工作负载分组；EM = K8s Kubelet。

### 5.3 关键差异

| 维度 | CP | AP |
|---|---|---|
| 部署单元 | 一个 ECU 镜像 | 多个 AA 进程 + 多个 Manifest |
| OTA 粒度 | 整个 ECU（可能拆为多个 Logical Block） | **可对单个 AA / 单个 ServiceInstance 进行 OTA** |
| 启动顺序 | 静态（编译期决定） | 依赖图（Manifest 中的 `dependencies`） |
| 启动模式 | 固定一种 | Normal / kRestart / kDiagnostic / kUpdate（Function Group 切换） |
| 应用签名 | 可选 | **强制**（IAM + SM 需要签名验证） |
| 多 OEM 集成 | 困难（一个 SWC 改动要回归全 ECU） | **天然支持**（不同 OEM / Tier1 的 AA 共存） |

---

## 六、安全与功能安全

### 6.1 功能安全（ISO 26262）

| 维度 | CP | AP |
|---|---|---|
| 最高 ASIL 等级 | **ASIL D**（QMA，AURIX TC3xx 等） | 多数 ASIL B（域控） + 部分 ASIL D（如智驾主控） |
| 锁步核支持 | ✅ AURIX TC3xx 内置 Lockstep CPU | ⚠️ 取决于 SoC，需要外部 ASIL D MCU 作为 safety companion（如 Aurix + Orin 双芯片方案） |
| 故障处理 | BSW 的 DEM + DCM + 看门狗 + E2E 保护 | PHM（Platform Health Management）+ SM（State Management）+ supervised entity |
| 看门狗 | WDG（独立硬件看门狗 + Alive supervision） | 多层级看门狗：内部 liveness + 外部 hardware WDT + 通信 deadline supervision |
| 内存保护 | OS-Application 内存分区 + MPU（CP SC3/SC4） | MMU 进程隔离（强于 CP） |

### 6.2 信息安全（ISO/SAE 21434）

CP 时代，车载信息安全要求很弱，主要靠 Secure Onboard Communication + SecOC（带新鲜度的认证消息）。

AP 时代，信息安全被提到关键位置：

- **ara::iam（Identity & Access Management）**：基于 OAuth 2.0 风格的 token，AA 调用其他服务前必须申请权限 token
- **ara::crypto**：提供 RSA、ECC、AES、SHA-2、HMAC、TLS 握手等
- **ara::sm（State Management）**：状态机控制哪些 Function Group 在哪些状态下可以运行
- **ara::diag（DM、DOIP）**：统一诊断入口，强制 UDS over IP
- **Manifest 签名**：每个 Application Manifest 必须由 OEM 信任的 PKI 签名
- **Secure Boot + Secure Update**：从 BootROM 到 Linux kernel 到所有 AA，全链路验证
- **沙箱隔离**：通过 Linux namespaces + seccomp-bpf + capabilities 实现 AA 资源限制

> **AP 在信息安全上是全面升级**，这也是为什么智能车域控（涉及 OTA、V2X、远程诊断）都强依赖 AP 的原因。

---

## 七、运行时服务（Functional Cluster）对照

AP 的 FC 与 CP 的 BSW 模块并非一一对应，下表给出典型映射与新增：

| 功能域 | CP 模块 | AP FC |
|---|---|---|
| 通信 | COM / PduR / CanIf / CanTp / Socket Adaptor | **ara::com**（com）、传输层用第三方绑定（vsomeip、cyclonedds） |
| 操作系统 | AUTOSAR OS | 主机 OS（Linux/QNX/Integrity）+ **ara::exec** 进程编排 |
| 状态管理 | EcuM / BswM | **ara::exec** + **ara::sm** |
| 健康监控 | WDG / DEM | **ara::phm** |
| 持久化 | NvM / MemIf / Ea / Fee | **ara::per** + **ara::com**（备份） |
| 诊断 | DCM / DEM / DcmDsl | **ara::diag** |
| 日志 | DLT（部分 CP 已含） | **ara::log** |
| 时间同步 | StbM | **ara::tsync**（时间同步 FC） |
| 密码学 | SecOC / Crypto（CP 4.x 新增） | **ara::crypto** |
| 权限 | 无 / SecOC 简易认证 | **ara::iam** |
| 标定 / 测量 | XCP / A2L（不属于 AUTOSAR CP 标准） | **ara::ucm**（Update & Configuration Management）+ 第三方测量（如 A2L over XCP） |
| 网络管理 | Nm / CanNm / UdpNm | 部分由 ara::sm + 自定义 FC 实现，标准 AP NM 在 R21-11 引入 |
| 整车状态 | （CP 通常无） | **ara::vhsm**（Vehicle State Management，部分版本实验性） |

**新增能力（CP 没有）**：

- **ara::ucm**：OTA Master，统一管理"哪些 AA 可以升级"
- **ara::iam**：基于签名和 token 的访问控制
- **ara::tsync**：基于 PTP（IEEE 1588）的时间同步
- **ara::com（完整 SOA）**：服务注册中心、动态服务发现
- **ara::sm**：完整的车辆状态机（Off → Running → Drive → Park → Off）

---

## 八、开发流程与工具链

### 8.1 CP 开发流程

```
1. OEM 写 System Description (ARXML)
2. OEM 在 System Description 中定义所有 ECU、SWC、Port、信号、PDU
3. 工具链：DaVinci CFG Pro 配 BSW；DaVinci Developer 配 SWC 行为
4. RTE Generator 生成 RTE
5. 各 SWC 实现 Runnable（手写 C 代码）
6. BSW + RTE + SWC 一起交叉编译
7. 链接生成一个 ELF/HEX
8. 通过刷写工具（vFlash/INCA）写入 ECU
9. 上电运行，BSW 通过 DEM 上报 DTC，OEM 通过 DCM 诊断
```

**关键工具**：

- Vector DaVinci Configurator Pro / Developer
- EB tresos Studio / AutoCore
- ETAS RTA-BSW
- Elektrobit RTE Generator

### 8.2 AP 开发流程

```
1. AA 开发者在本地用 CMake + GCC 编译 AA 二进制
2. OEM 提供 Application Manifest（OEM + AA 开发者合作）
3. Service Instance Manifest 在集成阶段生成
4. Machine Manifest 在 ECU 量产阶段生成（同一平台所有 ECU 共享）
5. 用 Vector DaVinci Adaptive / EB tresos / ETAS RTA-VRTE 校验 Manifest
6. 把 AA + Manifest 打包成 SWCL（Software Cluster），签名后入库
7. OTA Master 把 SWCL 推送到车上
8. 车上 UCM（Update Config Master）校验签名 → 写入磁盘 → 通知 EM 重启相关 AA
9. EM 按新 Manifest 启动 AA
```

**关键工具**：

- Vector DaVinci Adaptive（覆盖 Manifest 编辑、ARXML/JSON 校验、AP 服务建模）
- EB tresos Studio for Adaptive
- ETAS RTA-VRTE（含 vVIRTUALtarget 仿真环境）
- Apex.AI / Autoware 参考实现（开源）

### 8.3 工具链差异

| 维度 | CP 工具链 | AP 工具链 |
|---|---|---|
| 主配置入口 | ARXML | ARXML + JSON（Manifest） |
| 配置生成 | 编译期 RTE/BSW 生成 | Manifest 校验 + 运行时由 EM 解释 |
| 编译流程 | 一次完整编译 | **AA 独立编译** + 集成阶段只组装 |
| 调试 | Trace32 / iSystem / Lauterbach + 调试 OS | GDB（Linux）+ Trace32（QNX） + vVIRTUALtarget |
| 仿真 | dSPACE HIL / Vector VT System | vVIRTUALtarget（纯软件仿真器）/ dSPACE / AVL |
| 测试 | TESSY / CTC++ / RTRT | GoogleTest + CppUnit + Apex.AI 自带测试框架 |
| 持续集成 | 相对简单 | **更接近云原生**（GitLab CI / GitHub Actions） |

---

## 九、应用场景与选型决策

### 9.1 典型 ECU 分类

| 域 | ECU | 主要功能 | 推荐平台 |
|---|---|---|---|
| 车身 | BCM（车身控制模块） | 灯光、雨刮、车窗、门锁 | **CP** |
| 车身 | PEPS（无钥匙进入） | 低频唤醒、加密、车身防盗 | **CP** |
| 底盘 | EMS（发动机） | 喷油、点火、扭矩 | **CP** |
| 底盘 | TCU（变速箱） | 换挡逻辑 | **CP** |
| 底盘 | ESP（车身稳定） | 制动、油门、横摆控制 | **CP** |
| 底盘 | EPS（电动转向） | 转向助力、车道保持 | **CP**（部分高端转向 AP） |
| 底盘 | 制动主缸 ESC | 制动冗余 | **CP**（ASIL D） |
| ADAS | 前视摄像头 | 车道线、目标识别 | **AP** |
| ADAS | 毫米波雷达 | 目标融合 | **CP（感知前）+ AP（融合）** |
| ADAS | 自动驾驶域控 | 感知、规划、控制 | **AP** |
| 座舱 | 仪表 | HUD/液晶仪表 | **AP** |
| 座舱 | 智能座舱主控 | 中控大屏、语音、HMI | **AP** |
| 网联 | T-Box | 远程通信、OTA、诊断 | **AP**（部分功能 CP） |
| 网联 | 中央网关 | 跨域路由 | **CP（部分 CAN 路由）+ AP（ETH 路由）** |
| 新能源 | VCU（整车控制） | 扭矩分配、能量管理 | **CP + AP** |
| 中央计算 | 中央计算单元 | 跨域融合 + 区域控制器 | **AP** |

### 9.2 选型决策树

```
Q1: ECU 是否需要硬件 MMU？
├── 否 → CP（绝大多数 MCU 没有 MMU）
└── 是 ↓
    Q2: ECU 算力是否需要 GHz 级 + 多核 + 大内存？
    ├── 否 → 仍可用 CP（如部分 NXP S32K3 64MB RAM）
    └── 是 ↓
        Q3: 是否需要动态 OTA / 多应用混跑 / SOA 通信？
        ├── 否 → CP（功能简单，强实时）
        └── 是 → AP
```

### 9.3 必须用 AP 的场景

满足以下任一条件，基本必须用 AP：

1. 算力 ≥ 1 GHz，多核 + 至少 4GB RAM
2. 需要 **运行时动态加载 / 卸载应用**
3. 需要 **高频 OTA**（单个 AA 独立升级）
4. 跨域融合（同时处理感知 + 规划 + 控制）
5. 需要完整 POSIX（线程、socket、文件系统）
6. 与云端 / V2X 频繁交互
7. SOAFEE / AUTOSAR AP + ROS2 协同运行

### 9.4 仍用 CP 的场景

满足以下任一条件，CP 是更合适选择：

1. 强实时硬要求（us 级抖动）
2. 算力 < 500 MHz
3. 单芯片就能搞定（如 BCM、ESP、EPS）
4. 无 OTA 需求或 OTA 频率极低（按月/年）
5. 成本极度敏感
6. 已有完整 CP 生态且不需要新功能

### 9.5 CP 与 AP 共存模式

实际车辆中，**两者基本总是共存**：

```
┌──────────────────────────────────────────────────┐
│                 整车                              │
│                                                  │
│  ┌──────────┐    CAN/CAN FD/ETH     ┌─────────┐  │
│  │ CP ECU 1 │◄─────────────────────►│ AP ECU 1│  │
│  │ (BCM)    │                       │ (座舱)  │  │
│  └──────────┘                       └─────────┘  │
│  ┌──────────┐                       ┌─────────┐  │
│  │ CP ECU 2 │◄─────────────────────►│ AP ECU 2│  │
│  │ (ESP)    │        ETH + SOME/IP  │ (智驾)  │  │
│  └──────────┘                       └─────────┘  │
└──────────────────────────────────────────────────┘
```

**典型集成方案**：

- AP 通过 **服务网关**（如 vSomeIP Daemon 或自定义 Proxy）桥接 CP 的 CAN 信号到 AP 的服务接口
- 跨域场景中，AP 的 EM 通过 **Function Group + Network State** 触发 CP 端进入 Special Mode（如刷写模式）
- AP 作为 OTA Master，统一管理所有 CP ECU 的刷写（OEM 倾向把所有 ECU OTA 集中到 AP 网关或中央计算单元）

---

## 十、技术细节补充

### 10.1 数据类型与序列化

| 维度 | CP | AP |
|---|---|---|
| 编程语言 | **C99 + MISRA C** | **C++17（含部分 C++20）** |
| 标准库 | AUTOSAR Standard Types + Platform Types | `std::*` + `ara::core::*` 增强 |
| 序列化 | 位级 / 大端信号打包 | SOME/IP 序列化 / CDR / Protobuf 风格 |
| 配置数据 | ARXML（XML Schema） | ARXML + JSON |
| 动态内存 | 通常禁用 malloc | **允许但受限**（ara::core::Array、ara::core::Vector） |

### 10.2 错误处理

CP：返回值（Std_ReturnType E_OK/E_NOT_OK）+ Det（Development Error Tracer，仅调试） + Dem（Diagnostic Event Manager，上报 DTC）。

AP：统一 `ara::core::Result<T, E>`（基于 C++ 约定，模式类似 `std::expected`，但 AP 标准在 R22-11 前已有 `Result` 实现）：

```cpp
ara::core::Result<int> GetSpeed() {
    if (sensor_fail_) {
        return ara::core::MakeError<int>(MyErrc::kSensorTimeout);
    }
    return speed_;
}

auto r = svc.GetSpeed();
if (r.HasValue()) { Use(r.Value()); }
else { LOG_ERROR() << "Err: " << r.Error().Message(); }
```

### 10.3 日志

- **CP**：DLT（Diagnostic Log and Trace）协议，部分 BSW 模块和 SWC 通过 PduR 上报
- **AP**：`ara::log` 命名空间，提供 `LOG_DEBUG()`/`LOG_INFO()`/`LOG_WARN()`/`LOG_ERROR()`/`LOG_FATAL()` 五个级别，自动路由到 DLT

### 10.4 持久化

- **CP**：NvM（非易失性内存管理）+ Ea（EEPROM Abstract）+ Fee（Flash EEPROM Emulation），数据按 Block 管理，单 ECU 级别
- **AP**：`ara::per` 提供 K/V 风格接口（`OpenKeyHandle`、`Get`、`Set`、`Save`），底层用文件或 KVDB（如 LMDB、LevelDB 的车规改造版本）

### 10.5 调度与时序

| 维度 | CP | AP |
|---|---|---|
| 调度决策时机 | 编译期确定 | 运行时由 OS 决定 |
| 优先级数 | 数十级 | POSIX SCHED_FIFO 支持数百级 + SCHED_DEADLINE 支持 EDF |
| 任务触发 | OS Alarm、COM callback、RTE event | ara::exec::WaitFrame / Trigger / ConditionVariable / 周期 timer |
| 时序保证 | OSEK 时序分析（Stack + WCET） | 部分通过 SCHED_DEADLINE 提供时间保证，整体弱于 CP |
| 抖动 | < 10us | 100us ~ 1ms（取决于 OS） |

> **关键结论**：在 **控制类高频 ECU（EMS/ESP/EPS）**，CP 仍是不可替代的选择；在 **域控/中央计算**，AP 才能承载复杂应用。

---

## 十一、与 AUTOSAR 相关的现代趋势

| 趋势 | 影响 CP/AP |
|---|---|
| **中央计算 + 区域控制器（Zonal）** | CP ECU 大幅减少，AP 域控 / 中央计算单元 / 区域控制器取代 |
| **跨域融合 OS** | AP + Linux + QNX + RTOS 多 OS 协同（如 Safe Linux with QNX Hypervisor） |
| **车云一体（Cloud-Native）** | 推动 AP 工具链容器化、CI/CD 化、Helm Chart 化部署 |
| **SDV（Software-Defined Vehicle）** | OEM 拥有软件定义能力，要求 OTA + 动态部署 → **AP 主战场** |
| **SOAFEE** | ARM 主导的车载 Linux + 容器化标准，与 AP 互补 |
| **ROS 2 + AP** | Apex.AI（已认证的 ROS 2）与 AP 互联，把 ROS 生态引入车规 |
| **AUTOSAR R24-11 / R25-11** | CP 进入维护期，新特性集中在 AP，包括 `ara::com` 性能优化、UCB（Unified Configuration Baseline）等 |
| **AUTOSAR Rust Working Group** | R23-11 起 AUTOSAR 开始评估 Rust 在 AP 中的角色，但短期内 C++ 仍是 AP 主力 |

---

## 十二、一页速查表

| 维度 | CP | AP |
|---|---|---|
| **全称** | AUTOSAR Classic Platform | AUTOSAR Adaptive Platform |
| **首版** | 2008 R3.0 | 2017 R17-03 |
| **目标硬件** | MCU（Infineon AURIX、NXP S32K、Renesas RH850） | MPU/SoC（Renesas R-Car、NVIDIA Orin、NXP S32G） |
| **OS** | OSEK/VDX 派生 AUTOSAR OS | POSIX OS（Linux/QNX/Integrity） |
| **语言** | C99 + MISRA C | C++17（部分 C++20） |
| **架构** | ASW / RTE / BSW 三层静态 | AA / ARA / FC / PF 四层动态 |
| **进程模型** | 单镜像，多 OS-Application | 多进程 + POSIX |
| **内存保护** | MPU（SC3/SC4） | MMU 进程隔离 |
| **通信** | COM 栈 + RTE（CAN/LIN/FlexRay/ETH PDU） | ara::com + SOME/IP/DDS（SOA） |
| **服务发现** | NM（静态） | SOME/IP-SD / DDS Discovery（动态） |
| **接口定义** | ARXML | ARXML + JSON Manifest |
| **部署** | 编译期全锁定，整 ECU OTA | 运行时按 Manifest 编排，单 AA OTA |
| **配置** | ARXML（OEM + Tier1 强耦合） | Manifest（OEM / Tier1 / App 开发者解耦） |
| **安全** | WDG + DEM + E2E | PHM + SM + IAM + Crypto + 签名 |
| **信息安** | SecOC（弱） | 强（IAM + Secure Boot + 签名 OTA） |
| **最高 ASIL** | ASIL D | ASIL B（多数） + ASIL D（智驾主控 + 安全 MCU 伴生） |
| **实时性** | 硬实时 us 级 | 软实时 ms 级 |
| **OTA 粒度** | 整 ECU | 单 AA / 单 SWCL |
| **调试** | Trace32 + 仿真 OS | GDB + vVIRTUALtarget |
| **典型 ECU** | BCM、EMS、ESP、EPS | 智驾域、座舱域、中央计算、网关 |
| **关键工具** | DaVinci CFG Pro / tresos / RTA-BSW | DaVinci Adaptive / tresos Adaptive / RTA-VRTE |
| **生成语言** | RTE 生成 C 代码 | Manifest 驱动运行时 |
| **状态管理** | EcuM + BswM | ara::exec + ara::sm |
| **日志** | DLT | ara::log → DLT |
| **持久化** | NvM | ara::per (KV) |
| **诊断** | DCM + DEM | ara::diag（UDS over IP） |
| **主战场** | 车身、底盘、新能源 | 智驾、座舱、车云、OTA |
| **未来趋势** | 维护期 | 主战场，新特性集中 |

---

## 十三、给不同读者的总结

### 13.1 给嵌入式新手

- **CP = 老汽车 ECU 的标配**，学会它能搞定 70% 的传统车载嵌入式岗位
- **AP = 新汽车 ECU 的趋势**，学会它能进入智能车 / 域控赛道
- **CP 和 AP 都要学**，两者长期共存而非替代
- 学习路径：先 CP（OS + COM + DEM） → 再 AP（ara::com + ara::exec + Manifest） → 实战项目

### 13.2 给架构师

- 一个新 ECU 立项时，先问三个问题：**算力多强？是否要 OTA？是否要 SOA？**
- CP 不会消失：车身底盘 + 强实时仍是 CP 主场
- AP 是大势所趋：域控/中央计算/智驾/座舱基本只剩 AP
- 大多数车是 **CP + AP 混合架构**，网关是核心枢纽
- 选型时还要考虑团队能力：CP 人才多、AP 人才少，AP 学习曲线更陡

### 13.3 给面试候选人

面试高频问题与答案要点：

1. **CP 和 AP 的本质区别是什么？**
   → CP 是静态强实时的微控制器平台；AP 是动态软实时的高性能处理器平台。
2. **AP 出现的根本原因？**
   → 自动驾驶 + SOA + OTA + 多核高算力 + 以太网，CP 无法满足。
3. **CP 通信和 AP 通信的区别？**
   → CP 是 RTE + COM Stack（信号/PDU 中心）；AP 是 ara::com + SOME/IP/DDS（服务/消息中心）。
4. **AP 的 Manifest 三类？**
   → Machine Manifest / Application Manifest / Service Instance Manifest，加上 Software Cluster Manifest 共四类。
5. **AP 如何做 OTA？**
   → UCM 负责校验签名 + 写入 + 通知 EM 重启；可对单个 AA OTA。
6. **AP 的信息安全如何实现？**
   → IAM（OAuth 风格 token）+ Crypto + Secure Boot + Manifest 签名 + OS 沙箱。
7. **CP 和 AP 是否会冲突？**
   → 不会，反而高度互补，AP 通过网关/Proxy 桥接 CP CAN 信号。
8. **AP 上 C++ 是必须的吗？**
   → 是，AP 标准接口全部用 C++，C 是非标准的。
9. **AP 的实时性比 CP 差，对吗？**
   → 是。CP 是硬实时（OSEK），AP 是软实时（POSIX OS），CP 的时序确定性仍优于 AP。

### 13.4 给后端转嵌入式者

| 后端概念 | CP 映射 | AP 映射 |
|---|---|---|
| Spring Cloud 微服务 | 单镜像内的 SWC | AA 进程 + ara::com |
| gRPC | C/S 接口（COM ClientServerInterface） | Method |
| Kafka PubSub | S/R 接口（COM SenderReceiverInterface） | Event |
| K8s + Deployment YAML | OIL + 编译期配置 | Application Manifest（JSON） |
| Helm / Operator | - | Software Cluster Manifest |
| 服务发现 | - | SOME/IP-SD / DDS Discovery |
| Oauth2 | - | ara::iam |
| OOM Killer | - | Linux OOM + ara::phm |
| Sidecar | - | Functional Cluster 守护进程 |
| Helm Chart 升级 | 重新刷写 ECU | SWCL OTA |
| 容器镜像 | - | SWCL + Application Manifest |
| Service Mesh（Istio） | - | 类似 vSOME/IP / Cyclone DDS 的发现 + 路由 |
| 日志系统（ELK） | DLT Viewer | ara::log → DLT → ELK |
| Prometheus | - | 自定义 FC 采集，ara::per 持久化 |

---

## 十四、参考资料与延伸阅读

### 14.1 本仓库内文档

- [`AUTOSAR-架构详解-后端转嵌入式.md`](./AUTOSAR-架构详解-后端转嵌入式.md) —— CP 入门全景
- [`AUTOSAR-Adaptive-平台详解.md`](./AUTOSAR-Adaptive-平台详解.md) —— AP 平台深入
- [`Vector-DaVinci-工具链详解.md`](./Vector-DaVinci-工具链详解.md) —— 主流工具链详解
- [`通信中间件DDS-SOMEIP-gRPC详解.md`](./通信中间件DDS-SOMEIP-gRPC详解.md) —— AP 通信栈配套
- [`C++标准演进详解-C++98到C++26.md`](./C++标准演进详解-C++98到C++26.md) —— AP 语言基础
- [`车企C-C++使用情况详解.md`](./车企C-C++使用情况详解.md) —— 行业语言栈全景

### 14.2 官方文档

- **AUTOSAR 官方文档门户**：https://www.autosar.org/standards/
  - CP 最新：R4.4（Classic Platform R4.4 Specification）
  - AP 最新：R22-11、R23-11（Adaptive Platform Specification）
- **AUTOSAR 标准 PDF 索引**：https://www.autosar.org/fileadmin/standards/
- **SWS / RS / EXP / TPS / TR**：AUTOSAR 用这些前缀区分文档类型（SWS = Software Specification，RS = Requirement Specification 等）

### 14.3 开源实现

- **vSomeIP**：SOME/IP 协议的开源实现（GENIVI / COVESA 维护）
  https://github.com/COVESA/vsomeip
- **Eclipse iceoryx**：AUTOSAR AP / ROS 2 共享内存传输
  https://github.com/eclipse-iceoryx/iceoryx
- **Apex.AI**：基于 ROS 2 的 ASIL D 认证中间件（与 AP 兼容）
  https://github.com/ApexAI
- **AOS（AUTOSAR Open Source）**：bosch 维护的 AP 部分实现（仅供学习，量产请用商业实现）
- **AURIX TC3xx + AUTOSAR CP**：Infineon 官方 MCAL + ASIL D OS 示例
- **SOAFEE**：ARM 主导的车载 Linux + 容器化
  https://www.soafee.io/

### 14.4 推荐书籍

- 《AUTOSAR Compendium》（作者团队）——AUTOSAR 圣经级参考
- 《汽车电子系统设计与制造》——整车电子电气架构
- 《功能安全：ISO 26262 应用指南》——CP/AP 的安全基础
- 《Embedded Systems: Introduction to ARM Cortex-M Microcontrollers》——MCU 入门
- 《Linux Device Drivers》（LDD3）——AP 底层 OS 入门

### 14.5 关键术语速查

| 术语 | 含义 |
|---|---|
| AUTOSAR | AUTomotive Open System ARchitecture |
| CP | Classic Platform |
| AP | Adaptive Platform |
| AA | Adaptive Application |
| ARA | AUTOSAR Runtime for Adaptive |
| FC | Functional Cluster |
| SWC | Software Component（CP 的应用单元） |
| RTE | Runtime Environment |
| BSW | Basic Software |
| DEM | Diagnostic Event Manager |
| DCM | Diagnostic Communication Manager |
| PDU | Protocol Data Unit |
| COM | Communication（CP 的通信模块） |
| NM | Network Management |
| SOME/IP | Scalable service-Oriented MiddlewarE over IP |
| DDS | Data Distribution Service |
| DLT | Diagnostic Log and Trace |
| EM | Execution Manager（AP 的进程编排器） |
| PHM | Platform Health Management（AP） |
| IAM | Identity & Access Management（AP） |
| UCM | Update and Configuration Management（AP） |
| UDS | Unified Diagnostic Services |
| DoIP | Diagnostic over IP |
| ECU | Electronic Control Unit |
| MCU | Micro Controller Unit（CP 主流） |
| MPU | Micro Processor Unit（AP 主流） |
| SoC | System on Chip |
| MMU | Memory Management Unit |
| MPU | Memory Protection Unit |
| ASIL | Automotive Safety Integrity Level（A/B/C/D） |
| OEM | Original Equipment Manufacturer（整车厂） |
| Tier1 | 一级供应商（博世、大陆、电装、安波福、采埃孚） |
| SOA | Service-Oriented Architecture |
| SWCL | Software Cluster（AP 的部署/OTA 单元） |
| CRR | Cross-Cutting Requirements and Recommendations |
| VFB | Virtual Functional Bus（AUTOSAR 的虚拟总线抽象） |

---

## 写在最后

CP 与 AP 的差异本质是 **车载电子电气架构演进的必然结果**：

- 当 ECU 数量多、功能简单、通信带宽低时，CP 是最优解；
- 当 ECU 整合为域控/中央计算、功能复杂、通信带宽高、需要持续 OTA 时，AP 是最优解。

理解 CP 和 AP 的差异，不是为了站队哪一方，而是 **理解汽车软件架构十年来的演进逻辑**，才能在面对实际项目时做出理性的技术选型。

**给后端转嵌入式者的最大提醒**：CP 和 AP 都不难，关键是 **主动去做一个完整 demo**。CP 用 EB tresos + S32K314 + Vector VT System；AP 用 Linux + vSOMEIP + DaVinci Adaptive 仿真器。做过一遍，面试和工作都能立得住。

> 如果本文与 [AUTOSAR-架构详解-后端转嵌入式.md](./AUTOSAR-架构详解-后端转嵌入式.md) 和 [AUTOSAR-Adaptive-平台详解.md](./AUTOSAR-Adaptive-平台详解.md) 有任何表述不一致的地方，请以 AUTOSAR 官方最新规范为准。