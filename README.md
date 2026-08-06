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

## 🚀 快速开始

按顺序阅读效果最佳：

1. 先看 [`AUTOSAR-架构详解-后端转嵌入式.md`](./AUTOSAR-架构详解-后端转嵌入式.md) 建立 AUTOSAR 全景
2. 再看 [`Vector-DaVinci-工具链详解.md`](./Vector-DaVinci-工具链详解.md) 了解工具如何落地

## 🗂️ 仓库结构

```
.
├── README.md
├── AUTOSAR-架构详解-后端转嵌入式.md
└── Vector-DaVinci-工具链详解.md
```

## 📝 后续计划

- [ ] 增加实战 demo：完整 SWC + ARXML 示例
- [ ] 拆分章节为多个独立 MD
- [ ] 补充 Adaptive AUTOSAR (ara::com) 实战

## 🤝 贡献

欢迎通过 Issue / PR 补充错误、提出建议。