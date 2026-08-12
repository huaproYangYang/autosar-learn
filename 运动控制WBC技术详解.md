# 运动控制 WBC 技术详解 —— 全身控制从原理到车载落地

> 面向人群：负责 **人形机器人、四足机器人、移动机械臂底盘、域控制器运动控制算法** 的工程师，同时面向 **线控底盘、车辆动力学控制（ESC/转矩矢量分配）、智驾域运动规划** 的算法与系统架构师
> 目标：把 WBC（Whole-Body Control，全身控制）的**问题定义、数学基础、任务层级、零空间投影、接触约束、QP 求解、浮动基座、与 MPC 结合、在车端的类比**讲清楚，并回答"为什么 2024-2026 年这波人形机器人热潮让 WBC 重新成为显学"
> 配套阅读：[`运动控制FOC技术详解.md`](./运动控制FOC技术详解.md)（电机级）、[`AUTOSAR-Adaptive-平台详解.md`](./AUTOSAR-Adaptive-平台详解.md)（部署平台）、[`车企C-C++使用情况详解.md`](./车企C-C++使用情况详解.md)（语言生态）

---

## 写在前面：为什么 2026 年要写一篇 WBC？

2024-2026 年的具身智能浪潮把一个 30 年前由 Oussama Khatib 提出的概念从学术书架拽进了量产车间：

```
1986  ：Khatib 提出 Operational Space Formulation（任务空间控制）
2005  ：Sentis & Khatib 提出层级化 WBC（hierarchical WBC）
2010s ：IHMC、MIT、CMU 等持续迭代；ORC（Online Recalculation）走向实时
2018  ：Crocoddyl 开源，DDP + contact 进入工程工具链
2022  ：人形机器人 Optimus 公开；WBC 在工业界首次大规模落地
2023  ：Figure 01、1X Neo、Apptronik Apollo 量产路线图曝光
2024  ：国内人形（宇树、智元、银河通用、加速进化）密集首发
2025  ：Crocoddyl 2.x、Pinocchio 3.x、HPIPM 1.x 进入量产代码
2026  ：WBC 框架同时出现在 **人形机器人** 与 **线控底盘域控** 两端
```

一个老概念，为什么现在突然值钱？因为：

1. **算力到位**：SoC（NVIDIA Jetson Thor、Orin、高通 Ride 平台、芯驰 X9、地平线 J6）能在 1 kHz 把 30+ 关节的 QP 解出来
2. **动力学库成熟**：Pinocchio / Crocoddyl / Drake / CasADi 让"自动建模"成为可能
3. **接触场景普及**：双足/四足 + 操作臂的"全身协调"是过去简单栈式控制搞不定的
4. **车端类比成熟**：ESC、转矩矢量分配、线控转向的层级分配早就用了类似思想

本文按 **问题 → 数学 → 算法 → 实现 → 落地** 的顺序拆解 WBC，最后把它和车辆动力学控制做一张全景对照表。

---

## 目录

1. [WBC 是什么：定义、动机、与传统控制的关系](#1-wbc-是什么定义动机与传统控制的关系)
2. [核心数学基础：刚体动力学 + 任务空间](#2-核心数学基础刚体动力学--任务空间)
3. [任务层级与优先级：HBC、任务栈](#3-任务层级与优先级hbctask-栈)
4. [零空间投影：把"次要任务"塞进"主要任务"的余量里](#4-零空间投影把次要任务塞进主要任务的余量里)
5. [接触约束与力分布：WBC 的"刚性底线"](#5-接触约束与力分布wbc-的刚性底线)
6. [QP 求解：从公式到每秒 1000 次决策](#6-qp-求解从公式到每秒-1000-次决策)
7. [浮动基座：人形机器人最棘手的一环](#7-浮动基座人形机器人最棘手的一环)
8. [接触一致性 WBC（CC-WBC）与动态行走](#8-接触一致性-wbccc-wbc与动态行走)
9. [WBC × MPC：把"未来 1 秒"喂进 QP](#9-wbc--mpc把未来-1-秒喂进-qp)
10. [车辆动力学中的"类 WBC"思想](#10-车辆动力学中的类-wbc思想)
11. [软件栈：Pinocchio / Crocoddyl / HPIPM / Drake / CasADi](#11-软件栈pinocchio--crocoddyl--hpipm--drake--casadi)
12. [人形机器人典型 WBC 架构](#12-人形机器人典型-wbc-架构)
13. [在 AUTOSAR / ROS 2 上的部署形态](#13-在-autosar--ros-2-上的部署形态)
14. [工程化挑战：实时性、数值稳定性、模型-实物差距](#14-工程化挑战实时性数值稳定性模型-实物差距)
15. [2026 年趋势与未解问题](#15-2026-年趋势与未解问题)
16. [学习路径与速查表](#16-学习路径与速查表)

---

## 1. WBC 是什么：定义、动机、与传统控制的关系

### 1.1 一句话定义

> **WBC（Whole-Body Control，全身控制）**：把一个有 N 个驱动关节（人形 30+、四足 12+、机械臂 7+）的机械系统视为一个整体，在 **同时满足多个任务 + 接触约束 + 动力学可行性** 的前提下，求出每个关节的 **广义力 / 期望加速度 / 期望位置-速度**。

它不是单一算法，而是一类**框架**——具体实现可以是：

| 流派 | 代表 | 一句话思路 |
|---|---|---|
| 解析 WBC | Khatib、Sentis | 任务层级 + 零空间投影 + 闭式解 |
| 优化 WBC | NCI、CC-WBC | 把约束写成 QP / NLP，求数值解 |
| 优先级 WBC | Kanoun、De Lasa | 严格按优先级分层求解（strict hierarchy） |
| 预测 WBC | MPC-WBC | 用 MPC 预测轨迹，WBC 跟踪 + 力分配 |

### 1.2 为什么需要 WBC？

对比传统单任务控制：

| 场景 | 传统做法 | WBC 做法 |
|---|---|---|
| 机械臂抓杯子 | 单独 6-DoF 关节空间控制 | 抓杯（任务 1）+ 保持末端姿态（任务 2）+ 关节限位（约束）+ 避障（任务 3） 同一时间解 |
| 人形行走 | 上半身姿态 PD + 下半身 ZMP 规划分开 | 浮动基座 + 双脚接触约束 + 质心轨迹 + 上肢协作 **同一个 QP** |
| 线控底盘 | 横摆稳定性 ESC + 转向 EPS + 制动 IBC 各自闭环 | 整车 6-DoF（纵/横/垂 + 横摆/俯仰/侧倾） **统一协调**，带优先级 |
| 四足+机械臂 | 步态 Gait Controller + 臂 IK 独立 | 基体平衡 + 步态 + 操作任务 **同一框架**，自然利用零空间互不干扰 |

### 1.3 WBC ≠ 单一控制器

WBC 通常 **不是一个 PD 控制器**，而是一个**轨迹/加速度生成器**：把"我想要让某任务变量达到某值"翻译成"每个关节应该输出什么扭矩/位置/速度"。

```
           ┌────────────────────────────────────┐
   上层    │   任务指令（位置/姿态/力）        │   ← 规划层（Planner）
           └────────────────────────────────────┘
                          ↓
           ┌────────────────────────────────────┐
   中层    │   WBC（核心）：任务 + 约束 → q̈*   │   ← 本文的重点
           └────────────────────────────────────┘
                          ↓
           ┌────────────────────────────────────┐
   下层    │   关节伺服（FOC、扭矩环）         │   ← [`运动控制FOC技术详解.md`](./运动控制FOC技术详解.md)
           └────────────────────────────────────┘
```

---

## 2. 核心数学基础：刚体动力学 + 任务空间

### 2.1 浮动基座下的刚体动力学

对 N 自由度的机器人（人形约 30+ 关节 = 6 浮动基座 + 25 关节），广义坐标 $\mathbf{q} \in \mathbb{R}^n$ 满足：

$$
M(\mathbf{q})\,\ddot{\mathbf{q}} + h(\mathbf{q}, \dot{\mathbf{q}}) = S^\top \boldsymbol{\tau} + J_c^\top \boldsymbol{\lambda}_c
$$

| 符号 | 含义 |
|---|---|
| $M(\mathbf{q})$ | $n\times n$ 质量矩阵（对称正定） |
| $h(\mathbf{q}, \dot{\mathbf{q}})$ | 科氏 + 重力 + 离心项 |
| $S^\top \boldsymbol{\tau}$ | 关节扭矩（$S$ 是驱动器选择矩阵，浮动基座处 $S=0$） |
| $J_c^\top \boldsymbol{\lambda}_c$ | 接触力虚功项（$J_c$ 接触雅可比，$\lambda_c$ 接触力偶） |

人形的关键差异：**基座是被动的**（没有驱动器），只能通过接触力 + 关节力矩的耦合来间接控制——这就是"浮动基座"的全部复杂性来源。

### 2.2 任务空间映射

第 $i$ 个任务的"任务变量" $\mathbf{x}_i \in \mathbb{R}^{m_i}$ 与广义坐标 $\mathbf{q}$ 的关系：

$$
\mathbf{x}_i = f_i(\mathbf{q}), \qquad \dot{\mathbf{x}}_i = J_i(\mathbf{q})\,\dot{\mathbf{q}}, \qquad \ddot{\mathbf{x}}_i = J_i(\mathbf{q})\,\ddot{\mathbf{q}} + \dot{J}_i(\mathbf{q}, \dot{\mathbf{q}})\,\dot{\mathbf{q}}
$$

其中 $J_i = \partial f_i / \partial \mathbf{q}$ 叫 **任务雅可比**。

### 2.3 接触雅可比与力约束

如果末端 / 脚底与环境接触，对应接触点速度为零：

$$
J_c(\mathbf{q})\,\dot{\mathbf{q}} = 0
$$

对偶地，接触力 $\boldsymbol{\lambda}_c$ 通过 $J_c^\top$ 进入动力学方程。WBC 的硬约束就是这条等式 + 摩擦锥不等式：

$$
\boldsymbol{\lambda}_c \in \mathcal{K} \quad (\text{摩擦锥，例如 4 面线性近似})
$$

### 2.4 质心动力学（Centroidal Dynamics）

把人/机器人整体看成一个刚体，质心 $c$ 和角动量 $L$ 由接触力唯一决定：

$$
m\,\ddot{\mathbf{c}} = \sum_k \boldsymbol{\lambda}_k + m\mathbf{g}
$$
$$
\dot{L} = \sum_k (\mathbf{p}_k - \mathbf{c}) \times \boldsymbol{\lambda}_k
$$

这套动力学是 **简化降阶模型**：6 个自由度（质心 3 + 角动量 3），但和完整动力学在接触力分配上等价。

它在 WBC 里扮演什么角色？

- **降阶预测**：MPC 用质心动力学做 1-2 s 预测，QP 用全阶动力学做 ms 级跟踪
- **可行性检查**：质心轨迹是否在支撑多边形内（ZMP 条件）
- **接触力一致性**：让 QP 解出的 $\lambda_c$ 满足质心动力学

---

## 3. 任务层级与优先级：HBC、Task 栈

### 3.1 任务的分类

| 类别 | 例子 | 优先级约定（人形） |
|---|---|---|
| 接触一致性 | 双脚不能离地/打滑 | 最高（约束，不是任务） |
| 平衡 | 质心跟踪、ZMP | 高 |
| 基础任务 | 基体姿态保持 | 高 |
| 主操作任务 | 末端跟踪某轨迹 | 中 |
| 次操作任务 | 头部看某方向、手臂避障 | 低 |
| 关节限位 / 平滑 | 关节范围、jerk 最小 | 软约束（最低） |

### 3.2 严格优先级 WBC（Strict Hierarchy）

求解 N 个 QP，按优先级串行：

```
Task 1（最高）:  min ‖A1 q̈ − b1‖²        s.t.  contact constraints
            ↓  求出 q̈₁* ∈ null space of J1
Task 2:        min ‖A2 q̈ − b2‖²        s.t.  J1·q̈ = J1·q̈₁*  ,  contact
            ↓
...
Task k:        min ‖Ak q̈ − bk‖²        s.t.  J<i·q̈ = J<i·q̈ᵢ*  for all i<k
```

关键：**每层都把高优先级任务的解"锁死"**，后续任务只能在其零空间内优化。这就是零空间投影的核心。

### 3.3 软优先级 WBC（Weighted Stack）

把所有任务打包成一个 QP，用权重 $\mathbf{W}_i$ 区分优先级：

$$
\min_{\ddot{\mathbf{q}}} \sum_i \mathbf{w}_i \| A_i \ddot{\mathbf{q}} - \mathbf{b}_i \|^2
$$

**优点**：实现简单、求解快、调参方便
**缺点**：高优先级任务可能被低优先级任务"偷走"资源（如果权重差不够大）

### 3.4 二者对比

| 维度 | 严格优先级 | 加权软优先级 |
|---|---|---|
| 安全性 | 高（高优先级任务绝对不会被挤掉） | 取决于权重差 |
| 实现复杂度 | 高（N 个 QP 嵌套） | 低（单 QP） |
| 计算成本 | 高 | 低 |
| 调参难度 | 低（按顺序声明） | 高（权重需大量实验） |
| 工程主流 | IHMC、某些人形 | 多数车端控制、QP-WBC |

**工程经验**：当硬安全要求（不摔倒、不越界）重要时，用严格优先级；当只是性能优化时，加权栈更实用。

---

## 4. 零空间投影：把"次要任务"塞进"主要任务"的余量里

### 4.1 直观比喻

> 主任务是"把 A 路口走到 B 路口"。次任务是"路上避开水坑、顺路买杯咖啡"。你不会为了咖啡绕远路到走不到 B；也不会因为赶路就掉进水坑。WBC 的零空间投影就是这个"在主任务允许的范围内尽量满足次任务"。

### 4.2 数学

设主任务雅可比为 $J_1$，次任务雅可比为 $J_2$。最小二乘意义下，次任务的最优加速度指令是：

$$
\ddot{\mathbf{q}}_{2,\text{opt}} = J_2^\top (J_2 J_2^\top)^{-1} (\ddot{\mathbf{x}}_2^d - \dot{J}_2 \dot{\mathbf{q}})
$$

但这个解可能破坏主任务。把它投影到主任务的零空间：

$$
\ddot{\mathbf{q}}^* = \ddot{\mathbf{q}}_{1}^* + N_1 \ddot{\mathbf{q}}_{2,\text{opt}}
$$

其中 $N_1 = I - J_1^\top (J_1 M^{-1} J_1^\top)^{-1} J_1 M^{-1}$（考虑惯性的零空间投影，比 $I - J_1^+ J_1$ 更适合动力学系统）。

### 4.3 为什么用 $M^{-1}$ 加权？

经典零空间投影 $I - J_1^+ J_1$ 在任务空间上是合理的，但在关节空间会"乱动"某些关节。考虑惯性的零空间投影最小化 **动能二次型**——结果更平滑、更省能、更像人类动作。

### 4.4 例子：机械臂边跟踪边避障

```text
主任务：末端沿直线 → x_d(t)
次任务：手肘远离障碍物 → d_obs(q) > d_min

求解：
  q̈₁* = IK 主任务加速度
  q̈₂* = 障碍梯度方向加速（远离障碍）
  N₁   = I - J₁ᵀ(J₁M⁻¹J₁ᵀ)⁻¹ J₁M⁻¹
  q̈*  = q̈₁* + N₁ · q̈₂*

结果：末端严格走直线；肘部在零空间里"自然"绕开障碍
```

---

## 5. 接触约束与力分布：WBC 的"刚性底线"

### 5.1 接触的两面

WBC 里"接触"既约束运动也参与施力：

| 方向 | 数学表达 | WBC 里的处理 |
|---|---|---|
| 运动学 | $J_c \ddot{\mathbf{q}} + \dot{J}_c \dot{\mathbf{q}} = -\dot{J}_c \dot{\mathbf{q}}$ （无相对加速度） | 等式约束 |
| 动力学 | $J_c^\top \boldsymbol{\lambda}_c$ 进入动力学方程 | 求解变量 |
| 摩擦 | $\boldsymbol{\lambda}_c \in \mathcal{K}$ | 线性摩擦锥不等式 |
| 单向接触 | 法向力 $\lambda^n \geq 0$ | 不等式约束 |

### 5.2 接触力分配（Force Distribution）

双脚站立：双脚各 3-DoF 力 + 3-DoF 力矩 → 共 12 个接触力变量，但只需要满足 6 个质心动力学方程。冗余自由度怎么解？

> **典型策略**：在 WBC QP 里加入最小范数力项 $\min \|\boldsymbol{\lambda}_c\|^2$，并以质心动力学为等式约束。

求解后得到一组"最省力"的接触力，但仍满足：

- 质心期望轨迹
- 摩擦锥约束
- ZMP 在支撑多边形内

### 5.3 接触序列切换（Hybrid Dynamics）

步态本质上是一连串"双脚接触 / 单脚接触 / 飞行相"的切换。每次切换是一次**混合动力学切换**——刚度矩阵的秩变、约束集变。

WBC 在切换瞬间需要：

1. **事件检测**：GRF 估计 → 触地状态机
2. **约束集切换**：接触约束从 N 变 N-1
3. **轨迹重新规划**：MPC 在线更新质心轨迹
4. **避免冲击**：切换瞬间冲击力 $\to 0$

这是 WBC 工程化最棘手的部分之一。

---

## 6. QP 求解：从公式到每秒 1000 次决策

### 6.1 WBC-QP 标准形式

把"任务跟踪 + 接触约束 + 力分配 + 关节限位"打包成一个二次规划：

$$
\begin{aligned}
\min_{\ddot{\mathbf{q}},\, \boldsymbol{\lambda}} \quad & \sum_i \|A_i \ddot{\mathbf{q}} - \mathbf{b}_i\|_{W_i}^2 + \alpha \|\boldsymbol{\lambda}\|^2 \\
\text{s.t.} \quad & M \ddot{\mathbf{q}} + h = S^\top \boldsymbol{\tau}_{\text{cmd}} + J_c^\top \boldsymbol{\lambda} \\
                   & J_c \ddot{\mathbf{q}} + \dot{J}_c \dot{\mathbf{q}} = -\dot{J}_c \dot{\mathbf{q}} \quad (\text{接触不动}) \\
                   & \boldsymbol{\lambda} \in \mathcal{K} \quad (\text{摩擦锥}) \\
                   & \ddot{\mathbf{q}}_{\min} \leq \ddot{\mathbf{q}} \leq \ddot{\mathbf{q}}_{\max}
\end{aligned}
$$

### 6.2 求解器选择

| 求解器 | 类型 | 优点 | 缺点 |
|---|---|---|---|
| **HPIPM** | 内点法 QP | 高速、热启动友好、Crocoddyl 默认 | 装第三方依赖 |
| **OSQP** | 一阶分裂法 | 易部署、嵌入式友好 | 收敛较慢，迭代多 |
| **qpOASES** | 有效集 | 严格可行 | 大规模慢 |
| **ProxQP** | 近端 ADMM | 大规模稀疏 QP 快 | 较新 |
| **ECOS** | 内点法 | 凸锥规划 | 通用 QP 不一定最快 |
| **CUAD** / **Clarabel** | Rust 实现 | 内存安全 | 性能接近 OSQP |

人形机器人 30+ 自由度 + 12 接触力变量 → 单次 QP 大约 **0.3-1 ms**（HPIPM，Jetson Orin / Thor）。

### 6.3 热启动与抖动抑制

WBC 每 1-2 ms 解一次 QP，但变量维度不变、结构稳定——这是**热启动**（warm start）的天然场景：

- 本帧 $\ddot{\mathbf{q}}^*$ 作为下一帧的初值
- 求解器内部 IPM 迭代次数下降 50-70%

抖动（chattering）来自：

- 接触状态机抖动
- QP 解在约束边界跳来跳去
- 离散化噪声

工程上常用：

- **松弛变量**（slack variables）允许轻微约束违反换取平滑
- **死区 / 滞回**：接触状态机加 50 ms 滞回
- **一阶低通**：$\ddot{\mathbf{q}}_{\text{cmd}} = \alpha \ddot{\mathbf{q}}_{\text{QP}}^* + (1-\alpha)\ddot{\mathbf{q}}_{\text{prev}}$

---

## 7. 浮动基座：人形机器人最棘手的一环

### 7.1 什么是浮动基座

机器人"基座"是躯干（pelvis 或 trunk），它在世界坐标系下 **没有驱动器**——它的 6 个位姿（3 平移 + 3 旋转）只能由接触力 + 关节扭矩间接驱动。

```
[世界系] ——重力和惯性——→ [基座]  ←  (被动, 靠下边驱动)
                                  │
                       [腰部关节] → [髋] → [腿] → [脚]（接触地面）
                                  │
                       [肩] → [肘] → [腕]（末端）
```

### 7.2 后果

- **欠驱动**：6 个基座自由度没有直接控制输入
- **耦合非线性**：腰一动全身跟着动
- **积分漂移**：基座位姿靠里程计融合，估计误差会累积

WBC 的应对：

- 把基座位姿视为**求解变量**，从接触力 + 动力学约束反解
- 用 IMU + 视觉 / 腿足里程计在线校正
- QP 解中加入质心跟踪项，把基座漂移"压"在可控范围内

### 7.3 浮动基座下的精简坐标选择

| 方式 | 表示 | 优点 | 缺点 |
|---|---|---|---|
| 全 6+ n 坐标 | 欧拉角 + 关节角 | 完整 | 万向锁、奇异 |
| 全 6+ n 坐标 | 四元数 + 关节角 | 无奇异 | 4 维表达 + 1 约束 |
| 浮动基座 + 加速度层 | $\ddot{\mathbf{q}} \in \mathbb{R}^{n+6}$ | WBC 主流 | 不显式表达基座速度漂移 |
| 任务空间 | $\mathbf{x}_{\text{CoM}}, \mathbf{x}_{\text{feet}}, \mathbf{x}_{\text{hands}}$ | 直接操控任务 | 关节空轨迹需二次映射 |

工业主流：**四元数表达 + 加速度层 QP**。

---

## 8. 接触一致性 WBC（CC-WBC）与动态行走

### 8.1 什么叫"接触一致"

动态行走（跑步、跳跃、芭蕾式旋转）时，接触点会"打滑"——如果 WBC 解出的接触力与真实接触不一致，机器人会瞬间失衡。

**Contact-Consistent WBC（CC-WBC）** 的核心：

> 解出的接触力 $\boldsymbol{\lambda}$ 必须与期望接触加速度（通常为 0）一致，即不产生期望外的滑动。

数学上等价于：

$$
J_c \ddot{\mathbf{q}} + \dot{J}_c \dot{\mathbf{q}} = 0 \quad \text{(无相对加速度)}
$$
$$
J_c^\top \boldsymbol{\lambda} \in \text{range}(J_c^\top) \quad \text{(力只能来自接触)}
$$

这条看似显然的约束，是 WBC 从"静态站立"扩展到"跑步 / 单脚跳"的关键。

### 8.2 代表性工作

- **Audren et al. (2014)**：CC-WBC 早期工作
- **Righetti & Schaal (2016)**：iLQG + 接触一致
- **Caron et al. (2020)**：Stability-Consistent WBC
- **Mittal et al. (2023)**：人形跳跃 CC-WBC

---

## 9. WBC × MPC：把"未来 1 秒"喂进 QP

### 9.1 为什么要结合

WBC 是 **反馈式**（仅看当前状态），但行走/抓取需要 **前瞻**（提前规划下一步）。MPC（Model Predictive Control）提供 1-2 s 的预测：

```
[Planner] → 1-2s 质心/足轨迹 → [WBC] → 关节扭矩 → [关节伺服]
   ↑                                        ↑
   └──── IMU + 关节反馈 + 视觉 ─────────────┘
```

### 9.2 典型架构：双层 MPC-WBC

| 层级 | 周期 | 模型 | 输出 |
|---|---|---|---|
| 高层（Slow MPC） | 50-100 ms | 质心动力学 + 简化接触 | 1-2 s 质心轨迹 + ZMP |
| 中层（WBC-QP） | 1-2 ms | 全阶动力学 | 关节加速度 + 接触力 |
| 低层（关节 FOC） | 0.1-0.5 ms | 单关节电机模型 | PWM |

中间层 WBC 把"高层指令"转化为"每关节扭矩"，同时满足动力学可行性 + 摩擦锥约束。

### 9.3 经典库

- **Crocoddyl**：基于 Pinocchio 的 DDP / iLQR + 接触约束；非常适合 Slow MPC
- **OCS2**：ETH 的 MPC 库；支持接触序列切换
- **Towr**：足式轨迹优化（腿式步行专用）
- **Drake**：MIT 的全套仿真 + 优化 + 控制

---

## 10. 车辆动力学中的"类 WBC"思想

虽然"WBC"这个词在车端几乎不用，但**车端的多目标协调控制**早就走了相似的层级化路径。

### 10.1 线控底盘的协调控制

现代线控底盘（SBW、线控制动、线控驱动）需要在 6-DoF（纵/横/垂 + 横摆/俯仰/侧倾）上同时满足：

| 任务 | 优先级 |
|---|---|
| 防抱死（ABS） | 最高 |
| 横摆稳定（ESC） | 很高 |
| 牵引力控制（TCS） | 高 |
| 转矩矢量分配（TV） | 中 |
| 主动悬架 | 中 |
| 节能优化 | 低 |
| 舒适（垂向、噪声） | 低 |

传统做法是 **分布式**（每个控制器独立 + 上层仲裁），但 2020 年后越来越多车企转向 **集中式 WBC 风格**：

```
                        ┌──────────────────────────────┐
[驾驶员/自动驾驶指令] →  │ 6-DoF WBC（整车动力学 QP）   │ → 各执行器命令
                        └──────────────────────────────┘
                          ↑
              IMU + 转向角 + 轮速 + 视觉/雷达
```

### 10.2 横向对比

| 维度 | 人形 WBC | 车辆 WBC |
|---|---|---|
| 自由度 | 30+ | 6-DoF + 4 轮独立扭矩/制动 |
| 接触 | 双/单脚 | 4 轮 |
| 摩擦锥 | 双脚各 12 维 | 4 轮各 3 维 |
| 冗余自由度 | 大量（手/胳膊/头） | 较少（4 轮过驱动） |
| 决策频率 | 1 kHz | 100-200 Hz |
| 安全要求 | ISO 13849 / IEC 61508 | ISO 26262 ASIL D |

**车端"类 WBC" 三大优势**：
1. **统一可行性检查**：所有指令先经过 6-DoF 动力学 QP，可行才下发
2. **避免控制器打架**：ABS / ESC / TCS 不再需要复杂仲裁协议
3. **更容易实现 L3+ 接管**：自动驾驶层 → WBC → 执行器，单一接口

### 10.3 典型工作

- 比亚迪"易四方"、特斯拉 Cybertruck 后轮转向、奔驰 ESP 10、博世 iBooster+ESP hev——本质都是分布式接近 WBC 的工程实现
- 学术界：Stanford Dynamic Design Lab、Ohio State、清华大学、北航在车辆 WBC 上有 10+ 年积累

---

## 11. 软件栈：Pinocchio / Crocoddyl / HPIPM / Drake / CasADi

### 11.1 全景图

```
┌─────────────────────────────────────────────────────────────┐
│                       仿真 / 训练                              │
│  MuJoCo  · Isaac Gym/Lab  · Drake  · Bullet  · Genesis     │
└─────────────────────────────────────────────────────────────┘
                            ↑↓
┌─────────────────────────────────────────────────────────────┐
│                     建模 + 自动微分                            │
│        Pinocchio  ·  Drake  ·  CasADi  ·  SymPy              │
│        （URDF → 动力学方程 → 雅可比 / Hessian）                │
└─────────────────────────────────────────────────────────────┘
                            ↑↓
┌─────────────────────────────────────────────────────────────┐
│                       优化求解                                │
│  WBC QP : HPIPM  ·  OSQP  ·  qpOASES  ·  ProxQP              │
│  MPC NLP: Crocoddyl (DDP/iLQR)  ·  OCS2  ·  Acados          │
└─────────────────────────────────────────────────────────────┘
                            ↑↓
┌─────────────────────────────────────────────────────────────┐
│                       控制框架                                │
│   ROS 2 control  ·  mc_rtc  ·  ocs2_ros  ·  custom          │
│   (任务接口、参数服务、可视化、回放)                            │
└─────────────────────────────────────────────────────────────┘
                            ↑↓
┌─────────────────────────────────────────────────────────────┐
│                   部署：AUTOSAR Adaptive / ROS 2 / Bare-Metal │
│   (功能安全认证、确定性通信、实时调度)                          │
└─────────────────────────────────────────────────────────────┘
```

### 11.2 Pinocchio：刚体动力学的事实标准

Pinocchio（法国 CNRS / INRIA / LAAS 开发）是目前 WBC 工程几乎必装的库：

- URDF / SDF → 自动构建 $M(\mathbf{q})$、$h(\mathbf{q}, \dot{\mathbf{q}})$、$J_i$、$J_c$
- 解析自动微分（代码生成）+ 数值微分（fallback）
- 支持四足、双足、机械臂、固定翼、车辆
- C++ / Python 双 API

```python
import pinocchio as pin
model = pin.buildModelFromUrdf("humanoid.urdf")
data = model.createData()
q = pin.randomConfiguration(model)
pin.forwardKinematics(model, data, q)
pin.computeJointJacobians(model, data, q)
pin.framesForwardKinematics(model, data, q)
M = pin.crba(model, data, q)        # 质量矩阵
h = pin.nonLinearEffects(model, data, q, v)  # 科氏+重力
J = pin.getFrameJacobian(model, data, model.getFrameId("L_ankle"), pin.LOCAL_WORLD_ALIGNED)
```

### 11.3 Crocoddyl：带接触的 DDP

Crocoddyl = Pinocchio + iLQR / DDP + 多接触序列：

```python
import crocoddyl
state = crocoddyl.StateMultibody(model)
actuation = crocoddyl.ActuationModelFull(state)
running = crocoddyl.ActionModelContactFwdDynamics(state, actuation, cost, contacts)
solver = crocoddyl.SolverDDP(problem)
solver.solve()
```

输出是 **整段轨迹**（每个时刻的 $\mathbf{q}, \dot{\mathbf{q}}, \ddot{\mathbf{q}}, \boldsymbol{\lambda}$），非常适合 Slow MPC。

### 11.4 HPIPM：工业级 QP

HPIPM（High-Performance Interior-Point Method）：

- 专为机器人设计、稀疏 QP 优化
- C/C++ 内核，单核 0.3-1 ms 可解 30 自由度 WBC QP
- 提供热启动、矩阵预处理
- Crocoddyl 默认后端

### 11.5 Drake：MIT 全家桶

- 仿真 + 优化 + 控制一站式
- 优化建模语言（Symbolic + MathematicalProgram）
- 适合 research，但工业部署偏重

### 11.6 CasADl：符号 + 数值自动微分

- 把优化问题用数学语言直接表达
- 支持 IPOPT / SNOPT / WORHP 等 NLP 求解器
- 在 WBC-MPC 学术论文里出现频率最高

---

## 12. 人形机器人典型 WBC 架构

### 12.1 三大主流流派对比

| 公司 | 上层 | WBC 实现 | 部署平台 |
|---|---|---|---|
| Tesla Optimus | 行为树 + RL | 自研 QP-WBC | 自研 FSD-on-chip |
| Figure 01 | BT + LLM | Helix 模型（端到端）+ WBC | 自研 SoC |
| 1X Neo | 模仿学习 | 自研 WBC | Jetson Orin |
| Agility Digit | Model-based + RL | 自研 WBC | custom |
| 国内（宇树/智元/银河） | 强化学习 + 模型 | 多为 Pinocchio + 自研 QP | Jetson Orin / 地平线 |
| IHMC 学术 | 严格优先级 WBC | mc_rtc + HPIPM | Jetson / 桌面 |

### 12.2 简化参考架构

```
┌──────────────────────────────────────────────────────────────┐
│                  行为层（10-100 Hz）                          │
│  任务调度、目标物体识别、路线规划、动作选择                      │
└──────────────────────────────────────────────────────────────┘
                              ↓
┌──────────────────────────────────────────────────────────────┐
│                 MPC / 轨迹优化（50-100 Hz）                    │
│  Crocoddyl / 自研 DDP：质心 + 双足轨迹 + 末端轨迹               │
└──────────────────────────────────────────────────────────────┘
                              ↓
┌──────────────────────────────────────────────────────────────┐
│              WBC QP（500-1000 Hz）                            │
│  任务优先级：                                                  │
│    P1: 接触一致性 / ZMP 跟踪                                  │
│    P2: 基体姿态跟踪                                            │
│    P3: 末端跟踪                                                │
│    P4: 关节限位 / 平滑                                          │
└──────────────────────────────────────────────────────────────┘
                              ↓
┌──────────────────────────────────────────────────────────────┐
│              关节伺服 / FOC（1-10 kHz）                       │
│  每个关节电流环 / 速度环 / 位置环                              │
└──────────────────────────────────────────────────────────────┘
```

### 12.3 控制器分工

| 层级 | 关注什么 | 不关心什么 |
|---|---|---|
| 行为层 | "用户让我去厨房倒水" | 关节扭矩 |
| MPC | "接下来 1 s 双脚走哪、基体姿态怎样" | 单关节电流 |
| WBC | "本时刻各关节期望加速度 + 接触力" | 1 秒后的事 |
| FOC | "本时刻 i_q 应该是多少安" | 整体姿态 |

---

## 13. 在 AUTOSAR / ROS 2 上的部署形态

### 13.1 ROS 2：研究 / 原型首选

- **ros2_control**：标准硬件抽象 + 管理器
- **controller_manager**：加载多个控制器（关节 + WBC）
- **topic 通信**：WBC 节点输出 `JointCommand`，关节控制器订阅

**优势**：开发快、生态完整、可视化工具齐全（RViz、Foxglove）
**劣势**：实时性需配 RT 核 / RT-PREEMPT；不易做 ISO 26262

### 13.2 AUTOSAR Adaptive：量产域控首选

WBC 作为 **AA（Adaptive Application）** 部署：

| 元素 | AUTOSAR Adaptive 对应物 |
|---|---|
| WBC QP 求解 | AA + ara::com（输入/输出） |
| 状态估计 | AA + 订阅 IMU / 编码器 |
| 配置参数 | ara::per（持久化）/ ara::com |
| 周期性执行 | 调度器（10 ms / 1 ms） |
| 日志 / 监控 | ara::diag / ara::log |
| 安全监控 | ara::saf（如果做功能安全） |
| 部署平台 | 高通 8775 / 芯驰 X9 / 地平线 J6 |

**优势**：与车辆其他域（底盘、动力、智驾）共享中间件；功能安全可认证
**劣势**：起步慢、调试体验差、对 WBC 这种高频算法栈不友好

### 13.3 混合部署：ROS 2 + AUTOSAR Adaptive

```
[Linux / ROS 2]  ←─ 规划、机器学习、可视化、调试
       │  ara::com / SOME/IP（跨域通信）
       ↓
[AUTOSAR Adaptive] ←─ WBC、状态估计、关节指令、A/B 采样
       │  CAN-FD / FlexRay / EtherCAT
       ↓
[经典 AUTOSAR + 电机 ECU] ←─ FOC、扭矩环、传感器采集
```

这是 2026 年主流的"人形机器人在汽车供应链"部署形态：

- 高频 WBC 跑在 Adaptive 域控（Linux/POSIX 实时核）
- 单关节 FOC 在 Classic ECU（确定性更强）
- 用以太网/SOME/IP 做域间桥接

### 13.4 实时性预算

| 层级 | 周期 | 端到端延迟预算 |
|---|---|---|
| 上层规划 | 50 ms | 100 ms |
| MPC | 10 ms | 20 ms |
| WBC | 1-2 ms | 5 ms |
| 关节 FOC | 0.1 ms | 0.5 ms |
| 传感器采集 | 0.5 ms | 1 ms |

人形机器人 1 ms WBC 周期 + 0.5 ms FOC → 整体控制 1 kHz 闭环，对应端到端 < 3 ms。

---

## 14. 工程化挑战：实时性、数值稳定性、模型-实物差距

### 14.1 实时性

| 挑战 | 应对 |
|---|---|
| QP 求解耗时不稳定 | 热启动、HPIPM 选合适 IPM 迭代上限 |
| 自动微分首次编译慢 | 模型编译期生成代码（pinocchio / drake 都支持） |
| 调度抖动 | RT-PREEMPT、Xenomai、或 VxWorks/QNX |
| 多线程竞争 | WBC 单线程跑、传感融合独立核 |

### 14.2 数值稳定性

| 问题 | 表现 | 解决 |
|---|---|---|
| 质量矩阵 $M$ 接近奇异 | 特定构型下求解爆炸 | Tikhonov 正则化 |
| 雅可比秩亏 | 奇异位姿附近解跳变 | SVD 截断 / 阻尼伪逆 |
| 摩擦锥线性化误差 | 边缘接触处打滑 | 4 面 → 8 面线性化或 SOCP |
| 接触状态机抖动 | 力估计在零附近震荡 | 滞回 + 平滑 |

### 14.3 模型-实物差距（Sim-to-Real）

- **质量 / 质心偏差**：加工公差 → 模型不准 → WBC 解出"理论正确"的扭矩，实物跟不住
- **关节摩擦非线性**：库仑摩擦 + Stribeck → 速度死区 → 低速段 WBC 失效
- **接触形变**：脚底垫、足底变形 → 接触点偏移
- **驱动器带宽差异**：谐波减速器 vs 行星 → WBC 解的扭矩环跟不上

**工程应对**：

1. **系统辨识**：每个关节做扫频 → 拟合惯量 + 摩擦参数
2. **在线估计**：扩展卡尔曼 / 粒子滤波 估计质量、质心、摩擦
3. **鲁棒控制**：在 WBC 指令上加阻抗 / 力反馈外环
4. **Sim-to-Real RL**：用域随机化训练 policy，与 WBC 并行兜底

---

## 15. 2026 年趋势与未解问题

### 15.1 趋势

1. **WBC × 端到端学习**：
   - 端到端 RL 输出"任务指令" + WBC 做"可行性过滤"
   - 或直接把 WBC 的解作为 reward signal 训练 RL
2. **GPU 加速 WBC**：
   - 30+ 自由度 WBC QP 正在被 GPU 内点法 / ADMM 加速
   - NVIDIA cuAD、Clarabel GPU 版都在路上
3. **WBC × 全身触觉**：
   - 全身皮肤传感器 → 在线更新外力扰动 → WBC 在线补偿
4. **WBC × 车端域融合**：
   - "人形机器人在车上下棋、开车门、倒水"的场景推动车端 WBC 与机器人 WBC 共享中间件

### 15.2 未解问题

- **抓取与接触动力学耦合**：手部抓物体后系统动力学跳变 → WBC 如何切换约束集
- **多机协作 WBC**：双足 + 四足 + 机械臂协同，约束共享
- **长程任务下的 WBC 漂移**：连续运动 30 min，积分漂移如何抑制
- **认证路径**：ISO 26262 ASIL D 是否能完全容纳 WBC 的"自适应"行为（机器学习部分）

---

## 16. 学习路径与速查表

### 16.1 推荐学习顺序

1. **机器人学基础**（Siciliano 教材）→ 任务空间控制、雅可比
2. **刚体动力学**（Featherstone）→ $M(\mathbf{q})$ 计算
3. **Khatib 的 Operational Space** → 任务空间 PD + 零空间
4. **Sentis 的 Hierarchical WBC** → 任务优先级 + 投影
5. **Caron 的 CC-WBC / Stability WBC** → 接触一致 + 稳定约束
6. **Crocoddyl 教程** → DDP + 接触
7. **人形机器人开源项目**（OCS2、mc_rtc、bipedal-locomotion-framework）

### 16.2 关键论文清单

- Khatib, "A unified approach for motion and force control" (1987)
- Sentis & Khatib, "Synthesis of whole-body behaviors through hierarchical control" (2005)
- Bouyarmane & Kheddar, "Using task redundancy for gait generation" (2011)
- Audren et al., "Whole-body planning and control on humanoid robots" (2014)
- Caron et al., "Stability-preserving and contact-consistent WBC" (2020)
- Fernbach et al., "MPC and WBC for legged robots" (2018)

### 16.3 开源项目速查

| 项目 | 用途 | 语言 |
|---|---|---|
| **Pinocchio** | 刚体动力学 | C++ / Python |
| **Crocoddyl** | 带接触的最优控制 | C++ / Python |
| **OCS2** | MPC（含 WBC） | C++ / ROS |
| **mc_rtc** | IHMC WBC 框架 | C++ / ROS |
| **HPIPM** | 高速 QP | C |
| **Drake** | 全套仿真 + 控制 | C++ / Python |
| **CasADi** | 符号 + 数值优化 | C++ / Python |
| **Towr** | 足式步态优化 | C++ |
| **MuJoCo / Genesis** | 仿真 | C / Python |

### 16.4 一句话总结

> **WBC = 任务优先级 + 零空间投影 + 接触约束 + QP 求解**——它把"我要让机器人做很多事"翻译成"每个关节此刻该出什么扭矩"，同时保证动力学上走得通、不摔倒、不打滑。它在人形机器人里是核心，在线控底盘里是趋势，在 AUTOSAR Adaptive 里是部署形态。

---

## 附录：与本仓库其他文档的交叉引用

- [`运动控制FOC技术详解.md`](./运动控制FOC技术详解.md) — WBC 输出 → 关节 FOC 输入
- [`AUTOSAR-Adaptive-平台详解.md`](./AUTOSAR-Adaptive-平台详解.md) — WBC 部署平台
- [`AUTOSAR-架构详解-后端转嵌入式.md`](./AUTOSAR-架构详解-后端转嵌入式.md) — 域控视角
- [`车企C-C++使用情况详解.md`](./车企C-C++使用情况详解.md) — WBC 实现语言（C++ / Rust 趋势）
- [`Rust-语言详解-车载与系统软件视角.md`](./Rust-语言详解-车载与系统软件视角.md) — Rust 在 WBC 中的尝试
- [`通信中间件DDS-SOMEIP-gRPC详解.md`](./通信中间件DDS-SOMEIP-gRPC详解.md) — 域间通信：SOME/IP 用于车端 WBC 与底盘域对接