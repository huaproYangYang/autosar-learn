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

## 🚀 快速开始

按顺序阅读效果最佳：

1. 先看 [`AUTOSAR-架构详解-后端转嵌入式.md`](./AUTOSAR-架构详解-后端转嵌入式.md) 建立 AUTOSAR 全景
2. 再看 [`Vector-DaVinci-工具链详解.md`](./Vector-DaVinci-工具链详解.md) 了解工具如何落地
3. 通信中间件选型时看 [`通信中间件DDS-SOMEIP-gRPC详解.md`](./通信中间件DDS-SOMEIP-gRPC详解.md)

## 🗂️ 仓库结构

```
.
├── README.md
├── AUTOSAR-架构详解-后端转嵌入式.md
├── Vector-DaVinci-工具链详解.md
└── 通信中间件DDS-SOMEIP-gRPC详解.md
```

## 📝 后续计划

- [ ] 增加实战 demo：完整 SWC + ARXML 示例
- [ ] 拆分章节为多个独立 MD
- [x] 补充 Adaptive AUTOSAR (ara::com) 与通信中间件对比 → 见 `通信中间件DDS-SOMEIP-gRPC详解.md`

## 🤝 贡献

欢迎通过 Issue / PR 补充错误、提出建议。