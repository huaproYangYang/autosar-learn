# 车企 C/C++ 使用情况详解

> 面向嵌入式 / 后端转车载的工程师，全景剖析全球主流车企对 C / C++ 的使用现状、编译工具链、合规标准、以及 Rust 替代趋势。

---

## 一、为什么车载软件几乎离不开 C/C++

汽车电子的发展史几乎等同于 C 语言在 ECU 上的扩散史。进入 21 世纪第二个十年，**C++（尤其是 C++11/14/17）** 又随着 **Adaptive AUTOSAR、ADAS 域控、中央计算单元** 的崛起而爆发。下面是几条不可替代的理由：

| 维度 | C / C++ 的优势 | 替代品的短板 |
|---|---|---|
| **实时性** | 无 GC、确定性的执行时间，毫秒级中断响应 | Java / Go / Python 受 GC 影响延迟抖动大 |
| **硬件控制** | 直接操作寄存器、内存映射、位操作 | 高级语言抽象层过厚 |
| **功耗与资源** | 裸跑 MCU 几十 KB 内存即可 | Linux 进程级应用动辄数十 MB |
| **生态成熟度** | GCC / IAR / GHS / Tasking 等全套工具链 30+ 年沉淀 | Rust 工具链车规级认证 2023 才起步 |
| **合规标准** | MISRA C / C++、AUTOSAR C++14、ISO 26262 都有对应映射 | Rust 仅有初步的 Ferrocene 等认证尝试 |
| **历史包袱** | 80 年代起 Bosch / Continental 等 Tier-1 累计 10 亿+ 行代码 | 推翻重来的代价太高 |

> **核心结论**：在可预见的 10 年内，C 与 C++ 仍是车载软件的"主语言"，Rust 只是在**新增模块**层面缓慢渗透。

---

## 二、传统车企（欧美日韩）

### 2.1 大众集团（Volkswagen Group）——CARIAD 转型

| 项 | 说明 |
|---|---|
| **代表品牌** | VW、Audi、Porsche、Bentley、SEAT、Škoda、Lamborghini |
| **软件子公司** | CARIAD（前身：Car.Software Organisation，2020 年成立） |
| **操作系统** | VW.OS（基于 Adaptive AUTOSAR） |
| **主语言** | **C++ 为主**（Adaptive AP、SOA 中间件），**C 保留** 在 Classic AP BSW 层 |
| **E3 架构** | 三个高性能域控制器（ICAS1 车载娱乐 / ICAS3 ADAS），统一运行 VW.OS |
| **关键事件** | 2024 年 CARIAD 大规模裁员重组，CEO 离职；2025 年与 Rivian 成立合资公司重启软件平台 |

* **代码量规模**：CARIAD 全球约 **5 000+ 工程师**，代码库累计估算 **> 5 000 万行 C/C++**
* **编译工具链**：Vector / EB tresos 配置 AUTOSAR BSW；C++ 部分使用 **Linux + GCC** 或 **Green Hills INTEGRITY**
* **典型问题**：跨域集成延迟、Porsche Macan Electric 上线时 VW.OS 仍未完全就绪
* **启示**：即便大众这样年销 900 万辆的车企巨头，自研 OS 与 C++ 重构也是高难度工程

### 2.2 丰田（Toyota）——保守 + 自研 RTOS

| 项 | 说明 |
|---|---|
| **软件子公司** | Woven by Toyota（2018 年成立，前身 Toyota Research Institute） |
| **操作系统** | 自研 **Arene**（C++）、混合 AUTOSAR |
| **主语言** | C（BSW/Classic AP）+ C++14/17（ADAS、HMI） |
| **关键事件** | 2022 年宣布与电装（Denso）整合软件部门；2024 年起 Arene 部分开源 |

* **代码规范**：丰田内部**自有的 TSSC（Toyota Software Coding Standard）**，远比 MISRA 严格
* **混合策略**：动力总成、底盘控制仍用 AUTOSAR Classic（C）；智能座舱、ADAS 用 Adaptive AUTOSAR（C++）
* **2024 转向**：
  * 与 Aisin、Bosch 联合开发 **Arene** 平台，目标是 2025 年量产
  * 与 AWS 合作开发云原生 SDV 平台
* **典型特征**：**最小化 C++ 新特性使用**（禁用大部分模板元编程），追求"可审查、可静态分析"的代码

### 2.3 通用汽车（GM）——Global A / B 架构

| 项 | 说明 |
|---|---|
| **电子架构** | Global A、B（逐步过渡到 Ultifi） |
| **软件平台** | Ultifi（2022 起），基于 Linux + 自研中间件 |
| **主语言** | C（VCU、ECU）+ C++（车载娱乐、ADAS） |

* **开源贡献**：GM 是 **GENIVI / Automotive Grade Linux (AGL)** 的核心成员
* **关键事实**：GM Cruise Origin 自动驾驶项目因 2023 年事故召回，**暂停研发**；C++ 重写 L4 自动驾驶栈的工作暂缓
* **Ultifi 平台**：采用 SOA 架构，信号通信大量使用 SOME/IP（C++ 实现）

### 2.4 福特（Ford）

* **主语言**：C（车身控制）+ C++（SYNC 车载娱乐、ADAS）
* **平台**：SYNC 4 → SYNC 4A，搭载 QNX + Linux 双系统
* **战略动向**：2024 年起与 BlueCruise ADAS 系统持续投入，自研 L2+/L3

### 2.5 宝马（BMW）

| 项 | 说明 |
|---|---|
| **软件子公司** | 暂无独立子公司，但有 **BMW Group IT Research** 团队 |
| **操作系统** | 自研部分中间件 + QNX + AGL Linux |
| **主语言** | C（ECU）、C++14（Adaptive 平台、HMI）、少量 **Rust**（试验中） |
| **关键产品** | iDrive 8/8.5/9（基于 AGL），号称"全球最复杂的座舱" |

* **硬件**：BMW 与 Qualcomm 合作 Snapdragon Ride 平台（ADAS）+ 自研 ZCU 区域控制器
* **代码量**：估算 BMW 车载代码 **> 3 000 万行 C/C++**

### 2.6 梅赛德斯-奔驰（Mercedes-Benz）

| 项 | 说明 |
|---|---|
| **MBUX / MB.OS** | 自研操作系统（MB.OS，2024 起在 MMA 平台车型量产） |
| **主语言** | C++17/20 为主（座舱域），C 保留于动力底盘 |
| **合作伙伴** | NVIDIA DRIVE Orin（ADAS 主算力）、Google Cloud（云端） |

* **MB.OS 架构**：4 个域（车身、自动驾驶、座舱、驱动），统一 C++ 服务框架
* **特点**：相比 VW.OS 更加模块化，单一车型 OTA 失败不会波及其他车型

### 2.7 Stellantis

* **平台**：STLA Brain、STLA SmartCockpit、STLA AutoDrive（2024 起）
* **合作方**：亚马逊 AWS、宝马、富士康
* **主语言**：C++（主力），并积极与 **Mobileye**（C++ 栈）合作

### 2.8 现代 / 起亚（Hyundai / Kia）

* **子公司**：42dot（2021 年成立，自研 SDV 平台）
* **平台**：Pleos（2024 起），基于 Android Automotive + 自研中间件
* **主语言**：Java/Kotlin（信息娱乐层）+ C++（ADAS、VCU）+ C（ECU）

### 2.9 本田 / 日产 / 马自达 / 铃木

* **共同点**：**仍是 C/C++ 主导**，ADAS 用 C++，ECU 用 C
* **本田**：Honda Sensing Elite L3 自动驾驶系统用 C++ 编写
* **日产**：与 Renault 合作 SDV 平台，采用 C++17

### 2.10 沃尔沃 / 极星（Volvo / Polestar）

* **平台**：VolvoCars.OS（与 SPA2 架构同期）
* **合作**：与 Luminar（激光雷达）、NVIDIA Orin、Qualcomm 8155
* **主语言**：C++（VolvoCars.OS 核心）+ C（传统 ECU）
* **Polestar 3 / 4**：使用 VolvoCars.OS，部分功能"软件先行"策略

---

## 三、特斯拉 ——"软件定义汽车"的极端样本

特斯拉是 **唯一一家把 Rust 大规模引入车规关键路径的车企**，但 C++ 仍是绝对主力。

### 3.1 软件栈分层

| 层 | 操作系统 / 框架 | 主语言 |
|---|---|---|
| **信息娱乐** | 自研 Linux 发行版 | C++ / Python / JS |
| **Autopilot / FSD** | 自研 OS（基于 Linux），运行在自研 FSD 芯片 | **C++ + Rust** |
| **VCU / 底层 ECU** | 实时 OS | C |
| **OTA / 云端** | 自研 + Kafka / Python | 多语言 |

### 3.2 Rust 转型（2022 起）

* **官方公告**：2022 年 Tesla Autopilot 团队在博客解释**为何切换到 Rust**：
  * 编译期内存安全保证（避免 use-after-free）
  * 与 C++ 同等性能
  * 依赖管理与包管理现代化（Cargo）
  * 适配 `no_std` 嵌入式场景
* **典型应用**：
  * **FSD 网络通信层**
  * **车辆控制命令下发**
  * **OTA 升级模块**
* **教训**：早期 Rust 代码（2019 – 2021）部分已重写为 C++ 或重写回 Rust（早期 `unsafe` 用得过多）

### 3.3 软件规模

* **代码量估算**：> 1 亿行（含 C/C++/Rust/Python/JavaScript）
* **开发团队**：1 000+ 工程师（含前 SpaceX 团队）
* **OTA 频率**：平均 **每 2 周** 一次全车 OTA

---

## 四、中国车企（造车新势力 + 传统转型）

### 4.1 蔚来（NIO）—— SkyOS

| 项 | 说明 |
|---|---|
| **发布时间** | 2024 年 NIO IN 创新科技日 |
| **架构** | 1 + N（1 个整车 OS + N 个域控） |
| **主语言** | C++17/20（座舱、自动驾驶）+ C（VCU） |
| **关键技术** | SkyOS-M（车控）、SkyOS-L（中间件）、SkyOS-R（实时调度） |
| **预算** | 累计研发投入 > 50 亿元 |

### 4.2 理想（Li Auto）—— 理想星环 OS（开源）

| 项 | 说明 |
|---|---|
| **发布时间** | 2025 年 3 月，中关村论坛 |
| **开源时间** | 2025 年 4 月底登陆开源社区 |
| **团队规模** | 200 人 + 累计投入 **> 10 亿元** |
| **关键指标** | 芯片适配周期 5 个月 → 4 周，响应速度提升 1 倍，存储资源占用 -30% |

* **理想星环 OS 四大模块**：
  1. **车控操作系统**（替代部分 AUTOSAR Classic）
  2. **智能驾驶操作系统**
  3. **通信中间件**（类似 SOME/IP 自研实现）
  4. **虚拟化平台**（共享 AI 算力）
* **理念**："封闭只能放大系数，开源可以放大基数"

### 4.3 小鹏（Xpeng）—— Xmart OS

* **架构**：Android Automotive + 自研 Xmart OS
* **主语言**：Java / Kotlin（信息娱乐）+ C++（XPILOT ADAS）+ C（VCU）
* **核心自研**：XPILOT 3.0/3.5/4.0 全栈自研（C++ + Python 训练栈）

### 4.4 比亚迪（BYD）

* **策略**：**保守 + 渐进**
* **主语言**：C/C++ 主导，几乎不依赖 AUTOSAR（部分车型用 Classic AP）
* **e 平台 3.0**：自研整车控制器（VCU）+ 四电机分布式架构
* **代码规模**：估算车载代码 > 2 000 万行

### 4.5 上汽 / 广汽 / 长安 / 长城 / 奇瑞

* **共同特征**：仍以 **AUTOSAR Classic (C) + Adaptive (C++)** 为主
* **零束**（上汽子公司）：智能车解决方案，类 CARIAD 模式，C++14/17
* **广汽埃安**、**长安启源**：与博世、Vector 合作传统 AUTOSAR
* **长城**：与毫末智行合作 ADAS（C++）

### 4.6 小米（Xiaomi）

| 项 | 说明 |
|---|---|
| **入局时间** | 2021 年宣布造车，2024 年 SU7 上市 |
| **软件团队规模** | 2024 年披露约 **1 000 人** |
| **主语言** | C++（主力）+ C（VCU）+ 少量 Rust（试验） |
| **HyperOS** | 跨端 OS（车机 + 手机 + 平板 + IoT），底层 C/C++ |

### 4.7 华为（HiCar / 鸿蒙智行）

* **鸿蒙座舱 OS**：ArkTS / C++ 双栈
* **MDC 计算平台**（MDC 810 / 600 / 300）：C++ + 自研中间件
* **战略地位**：**Tier-1 软件供应商**，向赛力斯、奇瑞、江汽、北汽等多品牌供货

### 4.8 其它新势力

* **零跑（Leapmotor）**：C/C++ 主导；与 Stellantis 合资出海
* **极氪（Zeekr）**：基于 Mobileye + 自研，C++17
* **岚图 / 智己 / 阿维塔**：分别背靠东风 / 上汽 / 长安，C++ 中间件

---

## 五、Tier-1 供应商（车企背后的车企）

### 5.1 博世（Bosch）

* **AUTOSAR 发起者之一**
* **代码规模**：> **2 亿行** C/C++ 代码
* **编译器**：**Green Hills**（GHS）、**Tasking**（自研）
* **关键产品**：Bosch Radar / Camera ECU、ESP、ADAS

### 5.2 大陆集团（Continental）

* **主语言**：C（ECU）、C++（ADAS）
* **合作芯片**：NXP、Infineon、Qualcomm
* **代码规范**：内部 **MCG（Mission Critical Guidelines）** 比 MISRA 更严

### 5.3 电装（Denso）

* **丰田系 Tier-1**，主语言 C/C++
* **自研 RTOS**：DENSO Aurora
* **代码规范**：与丰田 TSSC 对齐

### 5.4 安波福（Aptiv）

* **风河 Wind River Diab** 编译器用户
* **C++ Adaptive 平台**：Wind River Helix Virtualization Platform

### 5.5 采埃孚（ZF）

* **域控制器 ProAI**：C++17 编写
* **Mobileye EyeQ 系列**：C++ 适配

### 5.6 Vector

* **C 语言"最大用户"**：AUTOSAR Classic BSW 几乎全部 C 实现
* **DaVinci 工具链**：ARXML → C 代码生成器
* **编译器合作**：Tasking、GHS、IAR

### 5.7 ETAS / EB tresos（Elektrobit）

* **RTA-OS / RTE**：C 实现
* **Adaptive AUTOSAR**：C++

---

## 六、关键工具链与编译生态

### 6.1 C 语言主要编译器

| 编译器 | 主要用户 | 关键特性 |
|---|---|---|
| **IAR Embedded Workbench** | 大众、博世、上汽 | MISRA C 严格度最高，瑞典产 |
| **Green Hills MULTI / INTEGRITY** | 博世、特斯拉 | 全球车规级最贵，安全认证完整 |
| **Tasking VX** | 大众、奥迪 | 荷兰产；AUTOSAR Classic 标配 |
| **Wind River Diab** | 安波福 | 老牌车规编译器 |
| **GCC** | 特斯拉、Mobileye | 开源，调试生态最强 |
| **Clang/LLVM** | 新势力、自研 OS | 现代化优化，正在追赶 IAR |

### 6.2 C++ 主要编译器

* **GCC**：所有新势力都用，编译速度 + 优化均衡
* **Clang/LLVM**：理想、蔚来、小米部分模块
* **GHS**：博世、安波福
* **MSVC**：罕见用于车载

### 6.3 静态分析工具

| 工具 | 厂商 | 标准支持 |
|---|---|---|
| **Helix QAC** | Perforce / PRQA | MISRA C:2012, MISRA C++:2008/2023, AUTOSAR C++14 |
| **Coverity** | Synopsys | C/C++ 深度分析 |
| **Cppcheck** | 开源 | 基础 MISRA 覆盖 |
| **clang-tidy** | LLVM | `bugprone-*`、`cert-*`、`misc-*` 系列 |
| **SonarQube** | SonarSource | 多语言，含 C/C++ |

---

## 七、合规编码标准（Coding Standards）

### 7.1 MISRA C / MISRA C++

* **MISRA C:2012**（包含 Amendment 1 – 4）：约 **160 条规则** + 22 条指令
* **MISRA C++:2023**（取代 2008 版）：约 **180 条规则**，适应 C++17
* **覆盖率**：几乎所有主机厂都强制 MISRA C，**新势力大多 MISRA C++**

### 7.2 AUTOSAR C++14 Guidelines

* **发布**：2017 年（AUTOSAR 项目），2024 年最新修订
* **核心立场**：基于 MISRA C++ + Effective Modern C++，**禁用 raw 指针、new/delete、std::bind、std::shared_ptr**（部分）
* **与 MISRA 关系**：MISRA C++:2023 大量参考 AUTOSAR C++14

### 7.3 ISO 26262 与 C/C++

* **ASIL-D（最高安全等级）** 要求：MISRA C/C++ 几乎 100% 覆盖、避免动态内存分配
* **典型 ASIL 映射**：
  * **ASIL A/B**：可放宽 MISRA
  * **ASIL C/D**：必须 100% MISRA 合规 + 静态分析 0 warning

### 7.4 其它标准

* **CERT C / CERT C++**（SEI）：信息安全导向
* **CWE**：通用弱点枚举
* **High Integrity C++**：嵌入式 C++ 子集

---

## 八、C vs C++：车企的"分水岭"

### 8.1 决策矩阵

| 场景 | 推荐语言 | 理由 |
|---|---|---|
| 经典 AUTOSAR BSW | **C** | 工具链成熟、安全认证完整 |
| MCU 固件（VCU、BCM） | **C** | 资源受限、生态成熟 |
| Adaptive AUTOSAR 应用层 | **C++14/17** | 必须（C++ API） |
| ADAS 感知/规控 | **C++17** | 性能 + 模板元编程 + Eigen 等库 |
| 中央计算 SOA 中间件 | **C++17** | ara::com 必须 C++ |
| 信息娱乐 | **C++ + Java/Kotlin**（Android Auto）/ **Swift**（Apple CarPlay） |
| 工具链 / 调试器 | **C++ / Python** | 性能 + 脚本化 |
| AI 推理（边缘） | **C++** + Python（训练侧） | TensorRT / ONNX Runtime / 自研 |
| OTA 升级服务 | **C++** 或 **Go** | 网络 I/O 密集 |
| 车云协同 SDK | **C++**（车端）+ **Go / Java**（云端） | 双侧最优 |

### 8.2 C++ 在车端的"禁区"

即使是 C++ 阵营，以下特性**几乎所有车企禁用**：

| 禁用项 | 理由 |
|---|---|
| `new` / `delete` | 动态内存不可控，违反 ASIL |
| 异常（exception） | 中断 / OS 上下文无法安全展开 |
| RTTI（`dynamic_cast`、`typeid`） | 需要运行时类型信息，违反 MISRA |
| 多继承 | 复杂度过高，难静态分析 |
| 模板递归 / 元编程 | 编译期 + 二进制膨胀 |
| 裸指针 / `std::shared_ptr` 循环引用 | 内存安全风险 |

**车企 C++ 实战风格 = C++ 写法 + C 的内存纪律**

---

## 九、Rust 浪潮 —— 哪些车企在尝试？

### 9.1 已落地的车企

| 车企 | 用途 | 时间 |
|---|---|---|
| **特斯拉** | Autopilot 网络层、OTA | 2022 起 |
| **丰田** | 部分 HMI 模块 | 2023 |
| **宝马** | 试验性 ADAS 子模块 | 2024 |
| **梅赛德斯-奔驰** | E 级车 OTA 子模块 | 2024 |

### 9.2 Rust 仍未成为主流的原因

1. **工具链车规级认证不完整**：Ferrocene 编译器直到 2024 才获 ASIL-D 评估
2. **学习曲线陡**：车企 90% 工程师是 C/C++ 背景，转型成本高
3. **嵌入式生态不如 C 成熟**：HAL / Driver 几乎全部 C 接口
4. **AUTOSAR 框架未原生支持 Rust**：Adaptive AP API 仍是 C++
5. **MISRA 等价物未成熟**：Rust 缺乏像 MISRA 那样成熟的"车规编码规范"

### 9.3 Rust 在车端的适用场景

* **OTA 升级验证**
* **网络协议栈**（TCP/IP / QUIC / 自研协议）
* **加密 / 解密模块**（典型 use-after-free 高发区）
* **复杂状态机**（编译器帮助保证不变式）

---

## 十、几个值得关注的行业趋势

### 10.1 软件定义汽车（SDV）

* **传统**：**100+ ECU**（每个 100 万行代码）
* **新趋势**：**4 – 6 个域控制器**（每个 5 000 万行代码）
* **核心语言**：C++（Adaptive AUTOSAR）+ C（少量底层）
* **代码量爆发**：2020 年平均 1 亿行；2030 年预计 **> 3 亿行**

### 10.2 中央计算 + 区域控制器

* **特斯拉 Hardware 4.0**：单 FSD 芯片（HW4.0 算力 ~ 720 TOPS）
* **小鹏 / 蔚来 / 理想**：单 Orin / 双 Orin 域控
* **VW / Mercedes**：3 个高算力域控制器 + 区域控制器

### 10.3 OTA 的安全与升级

* **车企 OTA 挑战**：
  * 多控制器（ECU / 域控 / 中央）协同升级
  * 失败回滚（A/B 分区）
  * 安全签名与 SOTA（软件 OTA）+ FOTA（固件 OTA）
* **典型栈**：C/C++ 编写升级逻辑 + Rust 重写敏感模块

### 10.4 自动驾驶的算力爆炸

* **NVIDIA Drive Thor**：2 000 TOPS（2025 量产）
* **地平线 J6 / 黑芝麻 A2000**：国产替代
* **车企自研芯片**：特斯拉 FSD、蔚来神玑、小鹏图灵、理想自研

### 10.5 端到端自动驾驶（E2E）

* **2024 起**：端到端 NN 模型落地（特斯拉 FSD v12、小鹏 XNGP、华为 ADS 3.0）
* **编程语言**：
  * **车端推理**：C++ + CUDA
  * **训练侧**：Python + PyTorch
  * **数据闭环**：Python + Scala

---

## 十一、对求职 / 转岗的建议

| 目标岗位 | 必学技术栈 |
|---|---|
| **Tier-1 嵌入式开发** | C、MISRA C、FreeRTOS、CAN、AUTOSAR Classic |
| **主机厂 ECU 开发** | C、AUTOSAR Classic、ISO 14229（UDS） |
| **域控制器 / 中央计算** | C++14/17、Adaptive AUTOSAR、Linux、CMake |
| **ADAS 算法工程师** | C++、CUDA、TensorRT、Python |
| **智能座舱** | C++ / Qt、Android Automotive、Kotlin |
| **OTA / 安全** | C++ / Rust、加密、TLS、TEE |
| **SOA / 中间件** | C++17、ara::com、SOME/IP、DDS |
| **AI 训练（车云协同）** | Python、PyTorch、CUDA |

> **C/C++ 是 5 年内最稳妥的车载技术栈投资**。Rust 加分但非必需。

---

## 十二、结语

* **C 是车规的"母语"**——30 年遗产、工具链完整、合规标准完备
* **C++ 是域控时代的"主语言"**——Adaptive AUTOSAR、ADAS、中央计算都离不开它
* **Rust 是增量而非替代**——5 年内不会动摇 C/C++ 根基
* **中国新势力正在"绕过 AUTOSAR"**——理想星环 OS、蔚来 SkyOS 都在用 C++ 自研 OS 与中间件
* **未来 5 年**：C++17/20 + 少量 Rust + Python（训练侧）将是主流组合

---

## 附：参考资料

1. **AUTOSAR** – Classic Platform R20-11 / Adaptive Platform R22-11
2. **MISRA** – C:2012 Amendment 1 / C++:2023
3. **ISO 26262:2018** – Road vehicles — Functional safety
4. **SAE J3061** – Cybersecurity Guidebook for Cyber-Physical Systems
5. **Tesla AI Day 2022** – FSD chip & software stack
6. **理想星环 OS 开源公告** – 2025 中关村论坛
7. **CARIAD** – Volkswagen Group Software
8. **Renesas / NXP / Infineon** – 车规 MCU 与 MPU 文档
9. **Green Hills Software** – INTEGRITY RTOS & MULTI Compiler
10. **Perforce / PRQA** – Helix QAC 静态分析

> 本文档遵循 CC BY-SA 4.0 协议，欢迎补充与勘误。