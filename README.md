# autosar-learn

> AUTOSAR 架构详解 —— 面向互联网后端开发转嵌入式的学习手册

## 📖 内容概览

[`AUTOSAR-架构详解-后端转嵌入式.md`](./AUTOSAR-架构详解-后端转嵌入式.md) —— 一份图文并茂、通俗易懂的 AUTOSAR 入门文档。

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

## 🚀 快速开始

直接阅读 [`AUTOSAR-架构详解-后端转嵌入式.md`](./AUTOSAR-架构详解-后端转嵌入式.md) 即可。

推荐阅读顺序：

1. 先看**整体架构**那节，建立三层模型的心智地图
2. 再看**BSW 关键模块**，结合你自己的后端中间件经验类比
3. 最后看**学习路径**，按 4 周计划实践

## 🗂️ 仓库结构

```
.
├── README.md
└── AUTOSAR-架构详解-后端转嵌入式.md
```

## 📝 后续计划

- [ ] 增加实战 demo：完整 SWC + ARXML 示例
- [ ] 拆分章节为多个独立 MD
- [ ] 补充 Adaptive AUTOSAR (ara::com) 实战

## 🤝 贡献

欢迎通过 Issue / PR 补充错误、提出建议。