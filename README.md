# autosar-learn

> AUTOSAR 架构详解 —— 面向互联网后端开发转嵌入式的学习手册

## 📖 内容概览

### [`AUTOSAR-架构详解-后端转嵌入式.md`](./AUTOSAR-架构详解-后端转嵌入式.md)
面向后端转嵌入式的 AUTOSAR 入门文档，图文并茂、通俗易懂。

- **目标读者**：有互联网后端经验（Java/Go/Python 等），正在转向车载嵌入式开发的工程师
- **核心思路**：全程用后端熟悉的概念（Spring Cloud / Kafka / gRPC / systemd）类比 AUTOSAR 模块
- **覆盖内容**：
  - AUTOSAR 三层架构（ASW / RTE / BSW）
  - 关键 BSW 模块（COM 通信栈、OS、DCM/DEM、NVM、看门狗）
  - Classic vs Adaptive 对比
  - S/R 与 C/S 两种通信范式（含 C 代码示例）
  - 开发流程与 ARXML 配置
  - ECU 启动过程
  - 4 周学习路径 + 关键术语速查表

### [`Vector-DaVinci-工具链详解.md`](./Vector-DaVinci-工具链详解.md)
AUTOSAR 行业第一工具链 Vector DaVinci 全景解读，从工具家族、架构原理到端到端工作流。

- **目标读者**：即将进入 Vector 工具链项目或面试车载工具链岗位的工程师
- **核心思路**：用"全栈开发工具链"类比 DaVinci 三件套（Configurator Pro + Developer + RTE Generator）
- **覆盖内容**：
  - Vector 工具矩阵与市场地位
  - DaVenci Configurator Pro 内部结构（BSW 模块树、PBcfg）
  - DaVenci Developer 的 SWC 开发流程
  - ARXML 流转：从 System Description 到 RTE 生成的完整链路
  - 与第三方编译器（IAR / GHS / Tasking）的集成
  - DaVenci Adaptive 与 Classic 的差异
  - 与 ETAS ISOLAR、EB tresos 的对比
  - 6 周实战学习路径 + 关键术语速查

### [`通信中间件DDS-SOMEIP-gRPC详解.md`](./通信中间件DDS-SOMEIP-gRPC详解.md)
车载/嵌入式三大通信中间件 DDS、SOME/IP、gRPC 协议原理、QoS、API、MCU 移植与选型决策全景详解。

- **目标读者**：做车载 SOA、ADAS 域控、车云一体、机器人、可信云原生后端的工程师
- **核心思路**：横向对比 + 纵向源码级拆解，覆盖协议栈 / 序列化 / 发现机制 / QoS / RTOS 移植 / 桥接
- **覆盖内容**：
  - DDS：DCPS 模型、22+ QoS、RTPS wire protocol、Fast DDS / Cyclone DDS / Connext 对比
  - SOME/IP：消息结构、SOME/IP-SD 服务发现、ARXML 建模、vSOME/IP 开源实现
  - gRPC：HTTP/2 + Protobuf、四种通信模式、拦截器、xDS、TLS/mTLS、车载落地策略
  - MCU/RTOS 移植（lwIP / FreeRTOS）、共享内存传输、跨中间件桥接
  - 选型决策树、性能观测与速查表

### [`AUTOSAR-Adaptive-平台详解.md`](./AUTOSAR-Adaptive-平台详解.md)
AUTOSAR Adaptive Platform（AP）全景详解，面向域控/中央计算单元/智驾的高性能 SOA 中间件平台。

- **目标读者**：从 CP 转 AP、做智驾/座舱/车云域控、准备面试车载中间件岗位的工程师
- **核心思路**：用"Linux 进程 + gRPC + K8s"类比 AP，把 ARA/FC/Manifest/SOA 一线贯穿
- **覆盖内容**：
  - AP 出现的工程动机与 Classic vs Adaptive 深度对比
  - ARA 接口规范总览、AP 整体架构（AA / ARA / FC / OS）
  - 核心 Functional Cluster：ara::com / exec / phm / per / sm / diag / log / iam / crypto
  - ara::com 服务通信三要素（Method / Event / Field）+ 代码骨架
  - Manifest 模型（Machine / Application / Service Instance / SoftwareCluster）
  - 开发流水线与工具链（Vector DaVinci Adaptive / EB tresos / ETAS RTA-VRTE）
  - 典型应用场景（智驾域、座舱域、车身/网关域）+ 中间件选型决策
  - 8 周学习路径 + 关键术语速查表

### [`车企C-C++使用情况详解.md`](./车企C-C++使用情况详解.md)
全球主流车企（传统 + 新势力 + Tier-1）对 C / C++ 的使用现状、编译工具链、合规标准、Rust 替代趋势全景剖析。

- **目标读者**：嵌入式工程师、车载软件岗位求职者、车企技术选型决策者
- **核心思路**：分车企逐一拆解语言栈 + 横向对比工具链/标准 + 行业趋势预判
- **覆盖内容**：
  - C/C++ 在车载不可替代的 6 大理由（实时性、硬件控制、MISRA、ISO 26262…）
  - VW CARIAD、丰田 Woven、GM Ultifi、Ford SYNC、BMW iDrive、Mercedes MB.OS
  - 特斯拉 Rust 转型（2022 起，C++ 仍为绝对主力）
  - 中国新势力：蔚来 SkyOS、理想星环 OS（开源）、小鹏 Xmart、华为 MDC、小米 HyperOS
  - Tier-1：博世、大陆、电装、安波福、采埃孚、Vector、ETAS
  - 编译工具链：IAR / GHS / Tasking / Wind River / GCC / Clang
  - 合规标准：MISRA C/C++、AUTOSAR C++14、ISO 26262 ASIL 映射
  - C vs C++ 决策矩阵 + C++ 在车端的"禁区清单"
  - Rust 浪潮：已落地的车企与未能成为主流的 5 大原因
  - SDV、中央计算 + 区域控制器、端到端自动驾驶等趋势

### [`Rust-语言详解-车载与系统软件视角.md`](./Rust-语言详解-车载与系统软件视角.md)
Rust 语言全景详解：从语言特性、安全关键认证、车载落地情况、与 C++ 对比到学习路径。

- **目标读者**：已掌握 C/C++，正在评估在 **车载 ECU、智驾域控、SDV 中间件、Linux 内核驱动、系统级工具链** 中引入 Rust 的工程师
- **核心思路**：把 Rust 作为 C/C++ 旁边的另一种工程选项，**不替代**而是**并存**
- **覆盖内容**：
  - Rust 历史与版本演进（2024 Edition、1.85–1.88 关键特性）
  - 核心语言特性：所有权、借用、生命周期、Trait、泛型、错误处理、智能指针
  - 异步运行时生态（Tokio / embassy）与线程安全（Send/Sync）
  - Unsafe、FFI、C++ 互操作（cxx / autocxx / crubit）
  - 宏系统与编译期编程（macro_rules! / 过程宏 / const fn）
  - 嵌入式生态（embedded-hal、embassy、RTIC、probe-rs） + MCU 支持矩阵
  - 安全关键认证：Ferrocene（ASIL B）+ Safety-Critical Rust 联盟
  - AUTOSAR Rust 工作组 + 车企落地（Tesla / Volvo / Woven / Bosch / CARIAD）
  - Linux 内核、Android、AWS / Cloudflare / Discord 等互联网基础设施
  - 白宫 ONCD、CISA、EU CRA 三大政策驱动的"内存安全"浪潮
  - Rust vs C++ 17 维度对比 + 批评与局限 + 8 周学习路径

### [`C++标准演进详解-C++98到C++26.md`](./C++标准演进详解-C++98到C++26.md)
从 C++98/03 到 C++26 每一个标准版本的语言特性、标准库变化，以及在 AUTOSAR / 功能安全语境下的取舍。

- **目标读者**：写车载 C++ 的嵌入式工程师、需要做语言标准/工具链选型的架构师
- **核心思路**：按版本纵向罗列特性，再横向对照 AUTOSAR C++14 / MISRA C++:2023 / `ara::core`
- **覆盖内容**：
  - 版本时间线速览与 ISO 编号对照
  - C++98/03 基线与 MISRA C++:2008
  - C++11 断代升级（移动语义、lambda、并发内存模型、智能指针）
  - C++14（AUTOSAR 指南基线）为何被选中，342 条规则方向
  - C++17（`optional`/`variant`/`string_view`/`constexpr if`）与 MISRA C++:2023
  - C++20 四大特性（Concepts / Ranges / Coroutines / Modules）+ `std::span`
  - C++23（`expected`/`mdspan`/`flat_map`/`generator`/`print`）
  - C++26 进行中（Reflection / Contracts / std::execution / inplace_vector）
  - `ara::core` 与标准库映射表、CP vs AP 特性禁区表
  - 编译器与认证工具链现实支持度 + 一页速查代码

### [`Linux驱动开发详解.md`](./Linux驱动开发详解.md)
Linux 内核驱动开发全景详解，从内核模块机制到字符/块/网络三大类驱动 + 车载 AP 落地实战。

- **目标读者**：从 CP/AP 应用层下沉到 Linux 内核的工程师、智驾/座舱域控 BSP 开发者
- **核心思路**：**原理 + 实战并重**——每章先讲内核机制，再给可编译运行的代码，最后回到 AP/车载场景
- **覆盖内容**：
  - 内核全景、模块机制、字符设备（含 misc/cdev/ioctl/poll/mmap/iter）
  - Linux 设备模型（bus/device/driver）与 sysfs/uevent
  - 平台设备 + 设备树（含 DTS 语法、overlay、of API）
  - 中断（GIC、threaded IRQ、硬中断/下半部）+ IRQ 亲和性
  - 并发同步（spinlock/mutex/RCU/seqlock/atomic/completion + lockdep）
  - 内存与 DMA（streaming/coherent/SG、SMMU、CMA）
  - 块设备 blk-mq 实战、IO scheduler
  - 网络设备 NAPI + ethtool、车载以太网 MAC/TSN
  - 调试与可观测（printk/dynamic debug/ftrace/bpftrace/perf/KASAN/kmemleak）
  - 车载 / AP 落地：SocketCAN、车载以太网、I2C/SPI 外设、DRM/KMS 座舱、PCIe、PREEMPT_RT、安全启动
  - DKMS / 内核模块签名 / 内核树内 Kconfig 编写
  - 调试工具速查 + 故障排查清单 + 性能优化清单
  - 写驱动前 8 个问题 + 学习路径与延伸阅读

### [`AUTOSAR-Adaptive-vs-Classic-对比详解.md`](./AUTOSAR-Adaptive-vs-Classic-对比详解.md)
AUTOSAR Classic Platform（CP）与 Adaptive Platform（AP）的全方位对比，从动机、架构、OS、通信、应用模型、安全、工具链到选型决策一文穿透。

- **目标读者**：需要做 ECU 平台选型的架构师、面试候选人、从 CP 转 AP 或反之的工程师
- **核心思路**：横纵交织——纵向沿"硬件 → OS → 运行时 → 通信 → 安全 → 工具链"主线深度对比，横向按"目标场景、关键术语、面试高频题"列表化呈现
- **覆盖内容**：
  - AP 出现的工程动机与 CP 局限（4 项核心硬约束）
  - 定位与目标场景、典型 ECU 分类与芯片对照
  - 整体架构差异（CP 三层 vs AP 四层 + 进程/线程/内存保护）
  - 操作系统与运行时（AUTOSAR OS vs POSIX OS + 启动流程对比）
  - 通信模型（RTE/COM 栈 vs ara::com Method/Event/Field + SOME/IP/DDS）
  - 应用模型与 Manifest 四类（Machine/Application/ServiceInstance/SoftwareCluster）
  - 安全与功能安全对比（DEM/WDG vs PHM/SM + ISO 26262 / 21434）
  - Functional Cluster ↔ BSW 模块对照表 + AP 新增能力
  - 开发流程与工具链对照 + CP/AP 共存模式
  - 选型决策树、面试高频题与后端类比对照表
  - 一页速查表 + 关键术语速查 + 延伸阅读

## 🚀 快速开始

按顺序阅读效果最佳：

1. 先看 [`AUTOSAR-架构详解-后端转嵌入式.md`](./AUTOSAR-架构详解-后端转嵌入式.md) 建立 AUTOSAR 全景
2. 接触域控/智驾/座舱/中央计算时，看 [`AUTOSAR-Adaptive-平台详解.md`](./AUTOSAR-Adaptive-平台详解.md) 学习 AP
3. 做平台选型/对比时看 [`AUTOSAR-Adaptive-vs-Classic-对比详解.md`](./AUTOSAR-Adaptive-vs-Classic-对比详解.md)
4. 再看 [`Vector-DaVinci-工具链详解.md`](./Vector-DaVinci-工具链详解.md) 了解工具如何落地
5. 通信中间件选型时看 [`通信中间件DDS-SOMEIP-gRPC详解.md`](./通信中间件DDS-SOMEIP-gRPC详解.md)
6. 想了解车企语言栈现状时看 [`车企C-C++使用情况详解.md`](./车企C-C++使用情况详解.md)
7. 写代码前/做语言标准选型时看 [`C++标准演进详解-C++98到C++26.md`](./C++标准演进详解-C++98到C++26.md)
8. 进入 AP 底层 / BSP / 内核驱动时看 [`Linux驱动开发详解.md`](./Linux驱动开发详解.md)

## 🗂️ 仓库结构

```
.
├── README.md
├── AUTOSAR-架构详解-后端转嵌入式.md
├── AUTOSAR-Adaptive-平台详解.md
├── AUTOSAR-Adaptive-vs-Classic-对比详解.md
├── Vector-DaVinci-工具链详解.md
├── 通信中间件DDS-SOMEIP-gRPC详解.md
├── 车企C-C++使用情况详解.md
├── C++标准演进详解-C++98到C++26.md
├── Rust-语言详解-车载与系统软件视角.md
├── Linux驱动开发详解.md
└── AUTOSAR-Adaptive-vs-Classic-对比详解.md
```

## 📝 后续计划

- [ ] 增加实战 demo：完整 SWC + ARXML 示例
- [ ] 拆分章节为多个独立 MD
- [x] 补充 Adaptive AUTOSAR (ara::com) 与通信中间件对比 → 见 `通信中间件DDS-SOMEIP-gRPC详解.md`
- [x] 新增 AUTOSAR Adaptive Platform（AP）专题文档 → 见 `AUTOSAR-Adaptive-平台详解.md`
- [x] 新增全球车企 C/C++ 使用情况专题 → 见 `车企C-C++使用情况详解.md`
- [x] 新增 Rust 语言详解（车载与系统软件视角） → 见 `Rust-语言详解-车载与系统软件视角.md`
- [ ] 增加主流车规 MCU（NXP S32K、Infineon AURIX、Renesas RH850）对比专题
- [ ] 增加端到端自动驾驶（E2E AD）算法栈专题

## 🤝 贡献

欢迎通过 Issue / PR 补充错误、提出建议。