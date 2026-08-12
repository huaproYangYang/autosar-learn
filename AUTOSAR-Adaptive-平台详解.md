# AUTOSAR Adaptive Platform 详解 —— 面向高性能车载计算的 SOA 中间件

> 面向人群：已了解 AUTOSAR Classic 基础，正在或将要接触 **域控制器 / 中央计算平台 / 自动驾驶** 的嵌入式/C++ 工程师
> 目标：用"后端/互联网"熟悉的概念（Linux 进程、SOA、gRPC、Kubernetes、POSIX），把 **AUTOSAR Adaptive Platform (AP)** 这套新架构讲清楚
> 读者收益：理解 AP 出现的工程动机、ARA 接口规范、各 FC 职责、Manifest 机制、与 Classic 的边界，能在面试和实际项目中讲出"我懂 Adaptive"

---

## 写在前面：为什么又要学一套新 AUTOSAR？

你可能已经看过 [`AUTOSAR-架构详解-后端转嵌入式.md`](./AUTOSAR-架构详解-后端转嵌入式.md)，熟悉了 **Classic Platform** 的 ASW/RTE/BSW 三层模型：

```
Classic 目标硬件：MCU（如 Infineon TC3xx、NXP S32K）
        ↓
AUTOSAR OS（OSEK 衍生，硬实时，调度单位 = Task）
        ↓
RTE + COM + BSW
        ↓
你的 SWC
```

但 2020 年之后，整车电子电气架构开始从"分布式 ECU"向"**中央计算 + 域控**"演进：

```
                ┌───────────────────────────────┐
                │   中央计算单元（车机/自驾/座舱） │
                │   - 高算力 SoC（多核 64 位）     │
                │   - 多 GB RAM、GPU/NPU          │
                │   - Linux/QNX/VxWorks          │
                │   - 跑 AUTOSAR Adaptive        │
                └───────────┬───────────────────┘
                            │ 千兆以太网 / PCIe
       ┌────────────┬───────┴────────┬────────────┐
       ▼            ▼                ▼            ▼
   动力域MCU   底盘域MCU        车身域MCU     智能驾驶域
   (Classic)   (Classic)        (Classic)    (Adaptive)
```

**核心矛盾**：Classic 是为单片机（MCU）的"小、硬实时、静态"设计的，但域控需要：

| 新需求 | Classic 能不能搞定 |
|---|---|
| 多核 64 位 ARM/Intel SoC | ❌ 只能跑 32 位 MCU |
| 动态加载 / OTA 升级 App | ❌ 编译期就绑定 |
| POSIX 进程 + 内存隔离 | ❌ AUTOSAR OS 没有进程概念 |
| 大数据吞吐（GB 级日志、感知数据） | ❌ 信号长度受限 |
| C++17/20 + Boost/三方库 | ❌ 强约束 MISRA-C |
| 千兆以太网 + SOME/IP/DDS | ⚠️ 支持但很重 |
| 高性能计算（AI、视觉、融合） | ❌ 性能/内存不够 |
| 面向服务架构（SOA）运行时 | ⚠️ 不自然 |

**AUTOSAR Adaptive Platform (AP)** 就是为这种"高性能车载计算"量身打造的——你可以把它理解为 **AUTOSAR 版的"Linux + Kubernetes + gRPC 平台"**。

---

## 目录

1. [全景：AP 是什么？](#1-全景ap-是什么)
2. [历史与版本演进](#2-历史与版本演进)
3. [Classic vs Adaptive 深度对比](#3-classic-vs-adaptive-深度对比)
4. [AP 整体架构](#4-ap-整体架构)
5. [ARA 接口规范总览](#5-ara-接口规范总览)
6. [核心 Functional Cluster 一览](#6-核心-functional-cluster-一览)
7. [ara::com —— 面向服务的通信](#7-aracom--面向服务的通信)
8. [ara::exec —— 执行管理](#8-araexec--执行管理)
9. [ara::phm —— 平台健康管理](#9-araphm--平台健康管理)
10. [ara::per —— 持久化存储](#10-araper--持久化存储)
11. [ara::sm —— 状态管理](#11-arasm--状态管理)
12. [ara::diag —— 诊断与刷写](#12-aradiag--诊断与刷写)
13. [ara::log & ara::tracing —— 日志与追踪](#13-aralog--aratracing--日志与追踪)
14. [ara::iam —— 身份与访问管理](#14- araiam--身份与访问管理)
15. [ara::crypto —— 加解密服务](#15-aracrypto--加解密服务)
16. [Manifest 与部署模型](#16-manifest-与部署模型)
17. [开发流程与工具链](#17-开发流程与工具链)
18. [典型应用场景](#18-典型应用场景)
19. [AP 与中间件选型决策](#19-ap-与中间件选型决策)
20. [学习路径与速查表](#20-学习路径与速查表)

---

## 1. 全景：AP 是什么？

**AUTOSAR Adaptive Platform (AP)** = 为高性能车载 ECU（域控、中央计算单元）设计的 **面向服务架构（SOA）运行时平台**。

**后端类比（一句话）**：

```
AUTOSAR Adaptive ≈ Linux 进程模型 + gRPC 服务发现 + Kubernetes Pod 调度 + POSIX 实时扩展
                  + 汽车级功能安全（ISO 26262）+ 整车 OTA + 诊断协议栈
```

它的三个核心特征：

1. **基于 POSIX 操作系统**（Linux / QNX / VxWorks），进程级隔离
2. **C++ API（AraCom / AraExec …）** + Manifest 描述部署模型
3. **服务导向（SOA）**：动态服务发现、订阅、调用、事件

**它的角色**：

```
                ┌────────────────────────────────────┐
                │ Adaptive Application              │
                │ (AA - 应用进程，自己实现业务逻辑)   │
                │   - 智驾感知 / 融合 / 规划          │
                │   - 座舱 HMI 后端                   │
                │   - 车云网关                        │
                └────────────────────────────────────┘
                          ↕ ARA API (C++)
                ┌────────────────────────────────────┐
                │ ARA (AUTOSAR Runtime for          │
                │       Adaptive Applications)       │
                │  ┌──────────────────────────────┐  │
                │  │ Functional Clusters          │  │
                │  │  ara::com / exec / phm / …    │  │
                │  └──────────────────────────────┘  │
                └────────────────────────────────────┘
                          ↕ OS + 硬件
                ┌────────────────────────────────────┐
                │ POSIX OS (Linux/QNX) + 多核 SoC     │
                └────────────────────────────────────┘
```

> **后端类比**：AP 的 ARA ≈ Java 的 JRE / .NET CLR / Node.js 的 Node Runtime；FC ≈ 一个个系统服务（Service），比如 K8s 的 API Server、kubelet、etcd。

---

## 2. 历史与版本演进

| 时间 | 事件 |
|---|---|
| 2017-03 | R17-03 首个 **AUTOSAR Adaptive Platform 正式版本** |
| 2018-10 | R18-10 引入 `ara::com` 服务接口标准化 |
| 2019-11 | R19-11 完善 FC、引入 `ara::crypto`、`ara::iam` 草案 |
| 2020-11 | R20-11 首批量产车型（VW、AUDI PPE 平台）开始集成 |
| 2022-11 | R22-11 强化诊断（`ara::diag`）、确定性执行 |
| 2024-11 | R24-11 完善跨域、时间敏感网络（TSN）、可观测性 |

**当前主流版本**：R22-11 ~ R24-11，OEM 项目从 R21-11 开始大量铺开。

---

## 3. Classic vs Adaptive 深度对比

| 维度 | Classic Platform (CP) | Adaptive Platform (AP) |
|---|---|---|
| **目标硬件** | MCU（16/32 位） | 高算力 SoC（64 位，多核） |
| **典型芯片** | Infineon TC3xx、NXP S32K3、Renesas RH850 | NVIDIA Orin / Xavier、Qualcomm 8295、NXP S32G |
| **OS** | AUTOSAR OS（OSEK 衍生） | POSIX OS（Linux / QNX / VxWorks） |
| **语言** | C（强 MISRA） | **C++**（C++17/20，禁用异常/部分 RTTI） |
| **编译期/运行期** | **静态配置**（编译期绑定一切） | **运行时动态**（Manifest + 服务发现） |
| **执行单元** | Task / ISR | **进程**（独立地址空间） |
| **通信方式** | Signal / PDU / Client-Server | **服务（Service）**：Method、Event、Field |
| **协议** | CAN / LIN / FlexRay / CAN-FD / ETH | 以太网为主（SOME/IP、DDS、TSN） |
| **应用更新** | 整 ECU 刷写 | **自适应应用增量更新 + OTA** |
| **内存** | KB ~ 几 MB | GB 级 |
| **功能安全** | ASIL-D 可达 | ASIL-B/D（与安全 OS 配合） |
| **典型场景** | 发动机/变速箱/制动/车身控制器 | 智驾域、座舱域、车身/网关域、车云一体 |
| **后端类比** | 单进程微控制器 + RTOS | 多进程 Linux + SOA 中间件 + K8s |

**实战判断口诀**：

- **控制信号多、毫秒级硬实时、内存几百 KB** → Classic
- **大量数据流、动态应用、OTA、C++/AI 算法** → Adaptive
- **一辆车可以同时跑**：中央计算用 AP，4 个角的控制 ECU 跑 CP，通过车载以太网 SOME/IP 互通

---

## 4. AP 整体架构

AP 的架构用三层描述：

```
┌────────────────────────────────────────────────────────────┐
│       Adaptive Application (AA)                           │
│       - C++ 进程，独立可执行                                │
│       - 通过 ARA API 调用平台服务                           │
└────────────────────────────────────────────────────────────┘
                         ↕ ARA
┌────────────────────────────────────────────────────────────┐
│        AUTOSAR Runtime for Adaptive Applications           │
│      ┌──────────────────────────────────────────────┐      │
│      │  Functional Clusters (FC)                    │      │
│      │  ara::com / exec / phm / per / sm / diag…   │      │
│      └──────────────────────────────────────────────┘      │
│      ┌──────────────────────────────────────────────┐      │
│      │  Adaptive Platform Foundation                │      │
│      │  - Manifest 处理、配置、日志、可执行装载       │      │
│      └──────────────────────────────────────────────┘      │
└────────────────────────────────────────────────────────────┘
                         ↕ OS / 驱动
┌────────────────────────────────────────────────────────────┐
│  POSIX OS (Linux/QNX) + 硬件抽象层 + 车载以太网              │
└────────────────────────────────────────────────────────────┘
```

**对应后端概念**：

| AP 术语 | 后端类比 |
|---|---|
| Adaptive Application (AA) | 微服务 / Pod 里的业务进程 |
| ARA | 一组 SDK + 库（类似 JDK） |
| Functional Cluster | 系统服务（API Server / kubelet / etcd） |
| Manifest | K8s YAML / Dockerfile / Helm Chart |
| Machine | 一台运行 AP 的 ECU（类似 Node） |
| Service Instance | RPC 服务注册中心中的一个 endpoint |
| ara::com | gRPC + 服务发现 + 消息中间件 |

---

## 5. ARA 接口规范总览

ARA 是 **AA 唯一允许调用的接口面**。所有 FC 都通过 `ara::<fc>::<api>` 形式提供 C++ 接口。

```
ara::
├── core       → Future、Result、String、Vector、Map 等基础类型
├── com        → 服务通信（Method / Event / Field）
├── exec       → 执行管理（启动顺序、状态机）
├── phm        → Platform Health Management（监督/恢复）
├── per        → Persistent Storage（KV / 文件）
├── sm         → State Management（功能组状态）
├── diag       → 诊断服务（DID / DTC / Routine）
├── log        → 日志（DLT）
├── tracing    → 调用链追踪
├── iam        → Identity & Access Management
├── crypto     → 加解密、签名、HASH、随机数
├── time       → 全局时间同步
├── restart    → 应用内重启管理
├── ucm        → Update & Configuration Management（OTA）
├── rtable     → Routing Table（E2E 路由）
└── tsync      → Time Synchronization（TSN 时间同步）
```

**所有 ara::\* 接口必须遵循**：

- 错误用 `ara::core::Result<T, ErrorCode>` 返回，**禁止抛异常**
- 异步 API 返回 `ara::core::Future<T>`
- 不允许裸 `new/delete`，必须用智能指针
- 通过 Manifest 声明权限（不能"开了口子"调底层）

> **后端类比**：
> - `ara::core::Result<T,E>` ≈ Rust 的 `Result<T,E>` / Go 的 `(T, error)`
> - `ara::core::Future<T>` ≈ `std::future` / Java `CompletableFuture`
> - Manifest ≈ K8s RBAC + PodSpec，**没授权就跑不起来**

---

## 6. 核心 Functional Cluster 一览

| FC | 主要职责 | 经典类比 |
|---|---|---|
| **ara::com** | 服务通信（Method/Event/Field） | gRPC + 服务发现 |
| **ara::exec** | 进程启动、依赖解析、运行时模型 | systemd + K8s Pod 调度 |
| **ara::phm** | 进程监督、崩溃恢复、看门狗 | systemd watchdog + K8s livenessProbe |
| **ara::per** | 持久化 KV / 文件 | etcd / 本地数据库 |
| **ara::sm** | 整车状态机（PowerMode、驾驶模式） | 全局状态总线 |
| **ara::diag** | UDS 诊断 / 刷写 | 经典 CP 的 DCM |
| **ara::log** | DLT 日志 | Logback + Logstash |
| **ara::tracing** | 调用链追踪 | OpenTelemetry |
| **ara::iam** | 权限管理 | OAuth2 / IAM |
| **ara::crypto** | 加解密、签名、Hash、随机数 | OpenSSL / mbedTLS |
| **ara::time** | 全局同步时间 | NTP + PTP |
| **ara::ucm** | OTA / 软件更新 | K8s Deployment / Helm |
| **ara::restart** | 应用内可控重启 | Spring Boot Actuator restart |
| **ara::tsync** | 时间同步（TSN） | PTP gPTP |
| **ara::rtable** | 路由表 / 跨 ECU E2E | Service Mesh 路由表 |

下面挑最常用的几个 FC 详细拆解。

---

## 7. ara::com —— 面向服务的通信

`ara::com` 是 AP **最核心** 的 FC，相当于"车载 gRPC + 服务发现 + Pub/Sub"。

### 7.1 服务接口的三个元素

```
Service Interface = 一组 Method + Event + Field
                 = 一份 IDL（`.arxml` 描述）
                 = 自动生成 C++ 代理/骨架代码
```

| 元素 | 类比 | 典型用途 |
|---|---|---|
| **Method**（方法） | gRPC Unary RPC | 请求-响应（"查询车速"、"切换档位"） |
| **Event**（事件） | gRPC Server Streaming / ROS Topic | 状态推送（"速度变化"） |
| **Field**（字段） | gRPC + 状态缓存 | 属性读写（"当前车速 = getter/setter"） |

### 7.2 通信路径

```
AA-A (Skeleton/Provider)             AA-B (Proxy/Consumer)
        │                                     │
        │  ① 启动 → 注册服务到 ara::com       │
        │                                     │
        │  ──── SOME/IP / DDS / IPC  ───►     │
        │                                     │
        │                          ② FindService → 找到实例
        │                          ③ SubscribeEvent("SpeedChanged")
        │                                     │
        │  ④ 业务逻辑触发 SpeedChanged event  │
        │                                     │
        │  ──── SOME/IP 事件 ───►             │
        │                                     │
        │                          ⑤ 回调 Handler 收到事件
```

### 7.3 服务发现与传输

| 元素 | 说明 |
|---|---|
| **Service Discovery** | 基于 **SOME/IP-SD**（AUTOSAR 标准化），可换成 DDS |
| **传输协议** | SOME/IP over UDP/TCP、DDS/RTPS、可选共享内存（同核内进程） |
| **序列化** | AUTOSAR 自定义 CDR（类似 Protocol Buffers） |
| **绑定方式** | **Skeleton（Provider 端） / Proxy（Consumer 端）**，编译期由工具生成 |
| **E2E 保护** | ara::com 可选配 E2E Protection（CRC + Counter + 序列号） |

### 7.4 极简代码骨架

```cpp
// ============ Service Interface（由 ARXML 生成） ============
namespace ara::com::proxy {
    template <typename T>
    class SpeedProxy {
    public:
        ara::core::Future<ara::com::ServiceHandleContainer<SpeedProxy<T>>> 
            FindService(ara::com::InstanceIdentifier id);
        
        ara::core::Result<void> SubscribeEvent(size_t eventId);
        
        // ... Method / Field 调用
    };
}

// ============ AA-B（Consumer）订阅事件 ============
auto proxies = ara::com::proxy::SpeedProxy<uint32_t>::FindService(id);
if (proxies.HasValue()) {
    auto& proxy = proxies.Value()[0];
    proxy.SubscribeEvent(SpeedEventId::kSpeedChanged);
    proxy.SetEventHandler(SpeedEventId::kSpeedChanged, 
        [](auto msg) {
            LOG_INFO("Speed = ", msg.Speed);
        });
}

// ============ AA-A（Provider）发送事件 ============
void ReportSpeed(uint32_t speed) {
    SpeedDataType data{};
    data.Speed = speed;
    skeleton_.SpeedChanged.Send(data);   // 推送给所有订阅者
}
```

> **后端类比**：可以把 `ara::com` 类比为 **gRPC + Nacos/Consul + Protobuf** 的组合；Skeleton ≈ gRPC Server，Proxy ≈ gRPC Client Stub。

---

## 8. ara::exec —— 执行管理

`ara::exec` 负责 **Adaptive Application 的整个生命周期**。是 AP 的"systemd + K8s"。

### 8.1 Execution Manifest 描述

每个 AA 必须声明一个 **Execution Manifest**，类似 K8s 的 PodSpec：

```json
{
  "process": "ADAS_Perception",
  "executable": "/opt/adas/bin/perception",
  "dependencies": ["CameraDriver", "LidarDriver"],
  "startupOrder": 3,
  "restartOnFailure": true,
  "maxRestarts": 5,
  "resources": {
    "cpu": "2000m",
    "memory": "4Gi",
    "gpu": true
  },
  "functionGroups": ["VehicleMotion", "ADAS"]
}
```

### 8.2 启动顺序与依赖

```
arxml 里声明 Function Group = 启动顺序的"逻辑分组"
MachineState = "MachineStartup" / "Driving" / "Update"
               类似 systemd 的 target（graphical.target）
```

| 概念 | 后端类比 |
|---|---|
| Function Group | systemd target / K8s namespace |
| Execution Dependency | K8s init container / pod dependency |
| Process Startup Delay | K8s initialDelaySeconds |
| Reporting Mode | K8s livenessProbe + restartPolicy |

### 8.3 确定性执行（Deterministic Execution）

R22 之后 AP 引入 **Deterministic Execution**：

- 周期性任务在固定的"Execution Time Window"内执行
- 类似 Linux PREEMPT_RT + SCHED_DEADLINE
- 用于 ADAS 中 100Hz 实时融合任务

### 8.4 进程模型与资源隔离

每个 AA 是独立 **POSIX 进程**：

```
+----------------------+
| AA-1: Perception     |   cgroup: cpu=2 cores, mem=4GB
|   PID 1234            |   capabilities: CAP_NET_RAW (禁止)
+----------------------+
+----------------------+
| AA-2: Planning        |   cgroup: cpu=1 core,  mem=2GB
|   PID 1235            |   seccomp: 白名单 syscall
+----------------------+
+----------------------+
| AA-3: HMI Backend     |   cgroup: cpu=1 core,  mem=1GB
|   PID 1236            |   user namespace: 隔离
+----------------------+
```

**强制资源隔离**：Linux cgroups + namespaces，AP 标准要求 ASIL-B 级别隔离。

---

## 9. ara::phm —— 平台健康管理

`ara::phm` = 进程级 **健康监督 + 故障恢复**。类比 K8s 的 livenessProbe + 自愈。

### 9.1 三种监督机制

```
┌────────────────────────────────────────────────────┐
│ ① Alive Supervision（存活监督）                      │
│    AA 周期性调用 Phm::ReportAlive(Checkpoint)         │
│    超出 AliveBudget 阈值 → 平台认为进程死锁 → 重启    │
│    类比：K8s livenessProbe                           │
├────────────────────────────────────────────────────┤
│ ② Deadline Supervision（截止监督）                   │
│    一个执行块必须在 Deadline 内完成                    │
│    超时 → 通知 AA / 上报 SM / 重启                   │
│    类比：gRPC deadline、分布式超时控制                │
├────────────────────────────────────────────────────┤
│ ③ Logical Supervision（逻辑监督）                    │
│    应用自己定义"期望状态"和"实际状态"对比              │
│    不一致 → 平台恢复动作                              │
│    类比：业务自定义健康检查                            │
└────────────────────────────────────────────────────┘
```

### 9.2 恢复动作

| Recovery Action | 含义 | 类比 |
|---|---|---|
| `NoRecovery` | 仅记录 | 仅 metric |
| `RestartProcess` | 重启进程 | K8s Pod restart |
| `RestartMachine` | 重启整个 ECU | K8s Node reboot |
| `SwitchToRecoveryMode` | 进入恢复模式 | 维护模式 |

```cpp
#include <ara/phm/phm.h>

void PerceptionLoop() {
    ara::phm::Phm::ReportAlive(CheckpointId::kAfterLidar);   // 周期上报
    
    if (LidarTimeout()) {
        ara::phm::Phm::ReportCheckpoint(CheckpointId::kLidarFailed);
        // phm 内部根据配置决定是否恢复
    }
}
```

---

## 10. ara::per —— 持久化存储

`ara::per` 提供 **进程级持久化**（K-V / 文件），类似 **etcd 的本地版** 或 SQLite 客户端。

### 10.1 K-V 存储接口

```cpp
#include <ara/per/per.h>

// 打开一个键值数据库
ara::per::Storage kDb{ara::per::StorageIdentifier{"MyAppConfig"}};

// 写入
kDb.SetValue("cruise_speed_kph", uint32_t{120});

// 读取
auto value = kDb.GetValue<uint32_t>("cruise_speed_kph");
if (value.HasValue()) {
    LOG_INFO("Loaded cruise speed = ", value.Value());
}

// 事务
ara::per::Transaction tx{kDb};
tx.SetValue("odometer", uint64_t{12345});
tx.SetValue("trip_distance", uint32_t{120});
tx.Commit();   // 原子写入
```

### 10.2 文件存储

```cpp
ara::per::FileStorage fs{ara::per::StorageIdentifier{"ModelWeights"}};
fs.Write(buffer, size);          // 大文件（神经网络权重）
fs.Sync();                       // 强制落盘
```

### 10.3 后端类比与存储后端

| ara::per 类型 | 典型后端 | 类比 |
|---|---|---|
| K-V | SQLite / LevelDB / 自实现 | etcd / Redis |
| File | 普通文件系统（ext4） + journal | 本地挂载 PV |

---

## 11. ara::sm —— 状态管理

`ara::sm` 管理 **整车 / 机器 / 功能组状态机**。一个 Machine 上所有 AA 能观察到一致的全局状态。

### 11.1 状态层级

```
Machine State（机器级）
  ├── Vehicle State（整车）：Off / Acc / Run / Crank
  ├── Function Group State（功能组）
  │     ├── ADAS（Off / Standby / Active / Fault）
  │     ├── VehicleMotion
  │     └── Comfort
  └── Application State（应用级）
```

### 11.2 使用示例

```cpp
#include <ara/sm/sm.h>

ara::sm::StateClient client{ara::sm::FunctionGroupName{"ADAS"}};

// 请求把 ADAS 切到 Active
auto result = client.RequestStateChange(ara::sm::FunctionGroupState{"Active"});

// 订阅状态变化
client.SetStateChangeHandler([](auto newState){
    LOG_INFO("ADAS state = ", ara::sm::ToString(newState));
});
```

**后端类比**：ara::sm ≈ 全局配置中心（Nacos / Apollo）的 **namespace + 状态广播**，也类似 Kubernetes 的 Node Lifecycle。

---

## 12. ara::diag —— 诊断与刷写

`ara::diag` 是 AP 上的 **UDS 诊断栈**（ISO 14229），比 CP 的 DCM 更"现代"。

### 12.1 能力对比

| 维度 | CP DCM | AP ara::diag |
|---|---|---|
| 协议 | CAN DoIP | **DoIP**（以太网为主），支持 CAN |
| 配置 | 编译期静态 | Manifest 描述 + 运行时动态 |
| 并发 | 单一诊断会话 | 多客户端（通过 IAM 鉴权） |
| 刷写 | DCM + FlashDriver | ara::diag + **ara::ucm**（OTA/刷写） |

### 12.2 典型诊断场景

```cpp
#include <ara/diag/diag.h>

// 声明一个 DID（Data Identifier）读服务
ara::diag::DataIdentifier svBatteryVoltage{0x0100};

// 读 DID
auto future = svBatteryVoltage.Read();
future.GetValue().Then([](auto& response){
    LOG_INFO("BatteryVoltage = ", response.Value);
});

// 故障码（DTC）
ara::diag::DtcInfo dtc{0xC12345};
dtc.SetStatus(ara::diag::DtcStatus::kConfirmed);
```

---

## 13. ara::log & ara::tracing —— 日志与追踪

### 13.1 ara::log —— DLT 日志

`ara::log` 输出到 **DLT (Diagnostic Log and Trace)**，由 GENIVI 标准化，几乎所有车厂都用。

```cpp
#include <ara/log/logging.h>

// 声明 logger
ARA_LOGGER logger{"ADAS", "PerceptionModule"};

ARA_LOG_INFO(logger, "Camera frame received, id=", frameId);
ARA_LOG_WARN(logger, "Lidar timeout, frame skipped");
ARA_LOG_ERROR(logger, "Sensor fusion failed");
ARA_LOG_FATAL(logger, "Self-driving stack crashed");
```

**特性**：

- 多等级：D/I/W/E/F
- 多上下文：CTXID、AppID、Description
- 远程订阅：通过 DLT-Viewer / DoIP 上位机读取
- 高效：用户态无锁 buffer，异步落盘

### 13.2 ara::tracing —— 调用链追踪

类似 **OpenTelemetry**：

```cpp
ara::tracing::Span span{"FusionProcess"};
span.SetAttribute("frame_id", frameId);

// ... 业务逻辑
span.End();
```

可与车载网络追踪（DDS tracing / SOME/IP trace）联动。

---

## 14. ara::iam —— 身份与访问管理

`ara::iam` = **车内 IAM 服务**，类比云原生的 **OAuth2 / RBAC**。

### 14.1 能力

- 应用身份证书（AppID + 签名）
- 权限授予（Grant / Revoke）
- 跨域权限映射
- 与 OEM 后台 IAM 对接

### 14.2 应用

```cpp
#include <ara/iam/iam.h>

// 请求权限
ara::iam::IamClient iam{};
auto grant = iam.RequestAccess({
    ara::iam::Permission{"CameraRaw", ara::iam::Access::kRead},
    ara::iam::Permission{"VehicleSpeed", ara::iam::Access::kRead}
});

if (grant.HasValue()) {
    LOG_INFO("IAM grant success");
}
```

---

## 15. ara::crypto —— 加解密服务

`ara::crypto` 提供 **统一加解密接口**，底层可插拔 OpenSSL / mbedTLS / HSM。

```cpp
#include <ara/crypto/crypto.h>

// 创建 AES-GCM 加密上下文
auto ctx = ara::crypto::CipherCtx::Create("AES-256-GCM");

// 加密
ctx.SetKey(key);
ctx.SetIv(iv);
auto ciphertext = ctx.Encrypt(plaintext);

// 签名 / 验签
auto sig = ara::crypto::Sign::Create("ECDSA-P256");
auto signature = sig.Sign(digest, privateKey);
bool ok = sig.Verify(digest, signature, publicKey);
```

**典型场景**：

- V2X 消息签名
- OTA 包完整性校验
- 车内 TLS（DoIP、TLS-SOME/IP）

---

## 16. Manifest 与部署模型

Manifest 是 AP 的 **灵魂**——所有行为都通过 Manifest 声明，没有代码层面的"开后门"。

### 16.1 Manifest 分类

| Manifest | 作用 | 后端类比 |
|---|---|---|
| **Machine Manifest** | ECU 全局配置（IP、启动策略） | K8s Node 描述 |
| **Application Manifest** | AA 自身的进程模型、依赖、资源 | K8s PodSpec |
| **Service Instance Manifest** | 服务接口、传输、事件 topic | gRPC Service + Discovery |
| **Execution Manifest** | 可执行文件路径、启动参数 | Dockerfile CMD |
| **Software Cluster Manifest** | 软件包版本号、签名 | Docker Image Tag |

### 16.2 Service Interface Manifest 示例（ARXML）

```xml
<SERVICE-INTERFACE UUID="..." SHORT-NAME="SpeedService">
  <METHODS>
    <METHOD UUID="..." SHORT-NAME="GetSpeed">
      <ARGUMENTS>
        <ARGUMENT UUID="..." SHORT-NAME="speed" TYPE-TREF=".../uint32_t"/>
      </ARGUMENTS>
    </METHOD>
  </METHODS>
  <EVENTS>
    <EVENT UUID="..." SHORT-NAME="SpeedChanged">
      <ARGUMENTS>
        <ARGUMENT UUID="..." SHORT-NAME="newSpeed" TYPE-TREF=".../uint32_t"/>
      </ARGUMENTS>
    </EVENT>
  </EVENTS>
  <FIELDS>
    <FIELD UUID="..." SHORT-NAME="CurrentSpeed" ...>
      <PROPERTIES>
        <PROPERTY VALUE="GetCurrentSpeed"/>
        <PROPERTY VALUE="SetCurrentSpeed"/>
      </PROPERTIES>
    </FIELD>
  </FIELDS>
</SERVICE-INTERFACE>
```

工具链基于此生成 C++ Proxy / Skeleton。

### 16.3 部署流

```
ARXML（设计工具）   ──┐
                      ├─► RTE Generator ─► C++ Proxy/Skeleton ─┐
ARA Configuration ───┘                                          │
                                                                ▼
                                                       Adaptive App 源码
                                                                │
                                                                ▼
                                                  CMake / Bazel 编译
                                                                │
                                                                ▼
                                                    + Manifest 打包
                                                                │
                                                                ▼
                                                  ara::ucm 安装到 Machine
                                                                │
                                                                ▼
                                                       EM 启动应用
```

---

## 17. 开发流程与工具链

### 17.1 工具全景

| 工具 | 厂商 | 作用 |
|---|---|---|
| **Vector DaVinci Adaptive** | Vector | 全套 AP 工具链（Configurator + Developer + RTE Gen） |
| **EB tresos AutoCore** | Elektrobit | AP 配置 + 代码生成 |
| **ETAS RTA-VRTE** | ETAS / Bosch | RTE + BSW + AUTOSAR 工具 |
| **AOS Studio** | Apex.AI | ROS2 + AP 集成工具链 |
| **AUTOSAR Builder** | Artop / itemis | 开源建模环境 |
| **VSCode / CMake / Conan** | 社区 | C++ 编译调试基础环境 |
| **RTRT / Vector vTESTstudio** | Vector | 单元测试 + 集成测试 |
| **vADASdeveloper / vADASstim** | Vector | ADAS HIL 仿真 |

### 17.2 典型开发流水线

```
┌─────────────────────────────────────────────────────────────────┐
│ ① 架构设计                                                     │
│   Artop / DaVinci Adaptive：Service Interface、Machine、App    │
│   生成 .arxml 描述                                               │
└─────────────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────────┐
│ ② 代码生成                                                     │
│   ara::com RTE Generator 生成 Proxy / Skeleton / 类型定义        │
└─────────────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────────┐
│ ③ 应用开发                                                     │
│   C++17 / 20 实现业务逻辑（VSCode / CLion）                       │
│   集成第三方库（OpenCV、Eigen、TensorRT…）                       │
└─────────────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────────┐
│ ④ 编译打包                                                     │
│   CMake / Bazel → 可执行文件 + Manifest + 签名                   │
│   输出 SoftwareCluster（.ipk / .arxml.zip）                      │
└─────────────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────────┐
│ ⑤ 集成测试                                                     │
│   vADASdeveloper / vTESTstudio / gtest                         │
│   HIL（硬件在环） / SIL（软件在环）                              │
└─────────────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────────┐
│ ⑥ 部署                                                         │
│   ara::ucm 安装 SoftwareCluster 到 ECU                          │
│   EM 启动、PHM 监督、SM 管理状态、log 上传                       │
└─────────────────────────────────────────────────────────────────┘
```

---

## 18. 典型应用场景

### 18.1 智能驾驶域（Autonomous Driving Domain）

```
       Camera / Lidar / Radar
              │ (Sensor Driver AA)
              ▼
   ┌─────────────────────────┐
   │ 感知融合 AA (Perception) │ ──┐
   └─────────────────────────┘   │
                                 │  ara::com (ObjectList)
   ┌─────────────────────────┐   │
   │ 预测 AA (Prediction)    │ ◄─┘
   └─────────────────────────┘
              │
              ▼
   ┌─────────────────────────┐
   │ 规划 AA (Planning)      │ ──► ControlCommands
   └─────────────────────────┘
              │
              ▼
   ┌─────────────────────────┐
   │ 控制 AA (Control)       │ ──► 经典 CP ECU（线控）
   └─────────────────────────┘
```

### 18.2 智能座舱域（Cockpit Domain）

- QNX / Android + AP
- HMI 后端 AA + 仪表盘 AA + 导航 AA
- 跨域通信：与 ADAS 域 SOME/IP 互通

### 18.3 车身/网关域（Body/Connectivity）

- 中央网关 AA + T-Box（车云）AA
- OTA 中心：ara::ucm 协调全车 ECU 升级
- V2X：ara::crypto 签名 + ara::com 广播

### 18.4 跨域协同案例

```
经典 CP ECU (BCM/ESC/EMS)  ──CAN/CAN-FD──┐
                                          │
AP 中央计算单元                            ├──► 整车 SOA 总线（SOME/IP）
                                          │
AP 智驾域控                                │
                                          │
AP 座舱域控                                ┘
```

---

## 19. AP 与中间件选型决策

很多团队在选型时会问：**AP vs DDS vs SOME/IP vs gRPC 怎么选？**

### 19.1 一张决策表

| 需求 | 推荐方案 | 理由 |
|---|---|---|
| 车厂标准、跨 Tier1 协作 | **AUTOSAR AP + ara::com** | 行业强制规范 |
| 大规模 Pub/Sub、QoS 22+ 种 | DDS | DDS 原生强 |
| 自驾域内部、ROS2 生态 | DDS（Fast/Cyclone） | ROS2 默认中间件 |
| 跨域、车云一体 | **AP + gRPC/HTTP** | 云原生友好 |
| 资源极小（MCU 几 MB） | **CP + SOME/IP** | vSOMEIP 轻量 |
| 国内车厂 + AP 标准栈 | **AP + ara::com** | 国产化首选 |

### 19.2 实际项目混合方案

```
                  ┌────────────────────────────────────┐
                  │  中央计算单元                       │
                  │  Linux + AP（QNX 也可）             │
                  │                                    │
                  │  ┌────────────────────────────┐    │
                  │  │ AP AA (感知/融合/规划)      │    │
                  │  │   ↓ ara::com                │    │
                  │  │   Skeleton 自带 SOME/IP/DDS │    │
                  │  └────────────────────────────┘    │
                  └────────┬───────────────────────────┘
                           │ SOME/IP over TSN
        ┌──────────────────┼───────────────────┐
        ▼                  ▼                   ▼
   经典 CP ECU        经典 CP ECU        经典 CP ECU
   (BCM/ESC/EMS)      (ADAS Sensor)      (T-Box)
   CP 通信栈          CP 通信栈           CP + 4G/5G Modem
```

### 19.3 AP 的局限

虽然 AP 是"未来"，但它有短板：

| 局限 | 原因 | 应对 |
|---|---|---|
| 学习曲线陡 | 概念多、规范厚 | 跟随 Vector 等工具链教程 |
| 工具贵 | DaVinci/EB tresos 高授权费 | 社区开源（eProsima Fast DDS + 自实现） |
| 性能开销 | ARA 调用有间接层 | 对性能极敏感处绕过 ARA（不安全） |
| 文档分散 | 标准 + OEM 私有扩展 | 锁定一个项目再看标准 |
| 国内落地晚 | 2022 才开始量产 | 跟随车厂"白盒"平台 |

---

## 20. 学习路径与速查表

### 20.1 8 周学习路径建议

| 周次 | 主题 | 关键产出 |
|---|---|---|
| W1 | 基础概念：AP vs CP、ARA、FC、Manifest | 思维导图 |
| W2 | 写第一个 "Hello Service"：ara::com Method + Event | 可运行 demo |
| W3 | ara::exec / ara::phm：进程监督 | HMI 演示（崩溃后自动重启） |
| W4 | ara::per / ara::sm：持久化 + 状态机 | 状态切换 demo |
| W5 | ara::diag / ara::log / ara::tracing | 通过 DoIP 上位机读 DLT 日志 |
| W6 | Manifest 全套：Service Interface + Execution + App | ARXML 全套 |
| W7 | Vector DaVinci Adaptive 工具链实操 | Configurator 配置 |
| W8 | 综合项目：模拟智驾感知 AA + ara::com 上报 | 端到端 demo |

### 20.2 关键术语速查表

| 术语 | 全称 | 含义 |
|---|---|---|
| **AP** | Adaptive Platform | 高性能车载平台 |
| **AA** | Adaptive Application | 平台上的应用进程 |
| **ARA** | AUTOSAR Runtime for Adaptive Applications | AA 调用的 C++ API 总称 |
| **FC** | Functional Cluster | 一个个系统服务模块 |
| **EM** | Execution Management | ara::exec，负责启动/监控 |
| **PHM** | Platform Health Management | 健康监督与恢复 |
| **SM** | State Management | 整车/机器状态 |
| **UCM** | Update and Configuration Management | OTA 与配置更新 |
| **PHM/EM/SM** | 三大"基础设施类" FC | 决定整个机器的运转 |
| **ara::com** | Communication | 服务化通信 |
| **Skeleton/Proxy** | 服务端/客户端绑定 | 由 ara::com 工具生成 |
| **Method/Event/Field** | 服务接口三种元素 | RPC + Pub/Sub + 属性 |
| **Service Discovery** | 服务发现 | 默认 SOME/IP-SD |
| **Manifest** | 部署/接口描述 | ARXML |
| **SoftwareCluster** | 部署单元 | 一个可升级的软件包 |
| **Machine** | 一台 AP ECU | 类比 K8s Node |
| **Function Group** | 功能组 | 启动顺序单元 |

### 20.3 常用资源

| 资源 | 说明 |
|---|---|
| [AUTOSAR 官网](https://www.autosar.org/) | 标准文档（需注册） |
| AUTOSAR R22-11 / R24-11 EXP 文件 | FC 接口定义 |
| `autosar-dev` GitHub | 社区示例代码 |
| Vector 官方培训 | DaVinci Adaptive 课程 |
| ETAS / EB 培训 | RTA-VRTE / tresos 课程 |
| eProsima Fast DDS | 开源 DDS 实现，可与 ara::com 互通 |
| Apex.AI 文档 | ROS2 + AP 集成范式 |
| 《AUTOSAR AP 实战》（如有出版） | 中文实战参考 |

### 20.4 与本仓其他文档的衔接

| 想深入 | 看本仓哪篇 |
|---|---|
| Classic Platform 基础 | [`AUTOSAR-架构详解-后端转嵌入式.md`](./AUTOSAR-架构详解-后端转嵌入式.md) |
| 工具链实操 | [`Vector-DaVinci-工具链详解.md`](./Vector-DaVinci-工具链详解.md) |
| SOME/IP / DDS / gRPC 协议 | [`通信中间件DDS-SOMEIP-gRPC详解.md`](./通信中间件DDS-SOMEIP-gRPC详解.md) |

---

## 写在最后：AP 不是"换皮"的 Classic

最后想强调：**AP 不是 Classic 的简单升级，而是一套新的软件工程范式**。

- **CP**：汽车 ECU 时代（信号、静态、确定）
- **AP**：中央计算时代（服务、动态、自适应）

你看完本文后应该能在 30 秒内讲清楚：

> "AUTOSAR AP 是一套基于 POSIX OS 的、面向服务架构（SOA）的车载中间件。它通过 C++ 的 ARA 接口，把 AA 进程运行在多核 64 位 SoC 上，提供服务通信、进程监督、OTA 升级、诊断、加密等 FC 功能；用 Manifest 描述部署和服务接口；本质上是车规版的 'Linux + gRPC + K8s'。"

如果能再加一句："我们项目里把感知/规划放在 AP 的中央计算单元，CP 的 BCM/ESC 继续用 Classic，通过 SOME/IP over TSN 互通"，那就达到面试可用的水平了。

---

## 🤝 贡献

欢迎在 [autosar-learn](https://github.com/) 仓库提 Issue / PR 补充错误与示例。

> 📌 **持续更新中**：本文基于 AUTOSAR R22-11 / R24-11 撰写，后续将补充完整可编译的 ara::com demo。
