# 端侧 AI 模型框架性能优化详解 —— 从模型压缩到车载部署全栈

> 面向人群：**端侧推理引擎开发、模型压缩工程师、车载 AI 算法工程师、智驾域 SoC 系统架构师**；同时面向关心 **量化/稀疏化/编译器** 的算法与系统工程师
> 目标：把端侧 AI 模型框架的**全栈性能优化技术**讲清楚——从模型压缩（量化 / 剪枝 / 蒸馏）、推理引擎内核优化（算子融合 / 内存规划 / 硬件加速）、运行时调度（多流 / 异构）、主流框架对比（ONNX Runtime / TFLite / MNN / TVM / TensorRT / llama.cpp），到 **2026 年的 LLM 端侧化、车载智驾域 NPU 部署、与 AUTOSAR Adaptive 的集成**
> 配套阅读：[`运动控制WBC技术详解.md`](./运动控制WBC技术详解.md)（AI 控制栈）、[`AUTOSAR-Adaptive-平台详解.md`](./AUTOSAR-Adaptive-平台详解.md)（部署平台）、[`车企C-C++使用情况详解.md`](./车企C-C++使用情况详解.md)（语言生态）、[`Rust-语言详解-车载与系统软件视角.md`](./Rust-语言详解-车载与系统软件视角.md)（安全侧）

---

## 写在前面：为什么 2026 年要写一篇"端侧 AI 性能优化"？

2026 年是端侧 AI 的"分水岭年"——三个独立曲线同时越过临界点：

```
2017    ：MobileNet / TFLite 1.0，端侧 CNN 第一次工业可用
2019    ：量化感知训练（QAT）成熟，INT8 普及
2021    ：Apple ANE / 高通 Hexagon 把端侧 NN 推到 < 10 ms
2023    ：LLM 端侧化萌芽（MLC-LLM、llama.cpp）
2024    ：端侧 7B 模型在旗舰手机上跑通
2025    ：车端 NPU（J6 / Thor / Ride）算力 200+ TOPS，端侧 BEV / Occupancy 普及
2026    ：端侧 3B 多模态、车端 14B 视觉-语言模型（VLM）、量化感知 + 投机解码成为标配
```

**三个不可逆的趋势**：

1. **云推理成本失控**：LLM 单次推理云端成本 $0.001-0.01，每车每天调用 1000+ 次 → 一年几千元 / 车
2. **实时性要求爆炸**：智驾端到端模型要求 100 ms 内闭环，云端往返 RTT 不达标
3. **数据合规**：车端摄像头 / 麦克风数据出境受限 → 必须在车端 / 端侧完成处理

这三个趋势把"端侧 AI 性能优化"从"加分项"推到"核心能力"。本文按 **模型层 → 引擎层 → 运行时层 → 硬件层 → 部署层** 五层结构展开。

---

## 目录

1. [端侧 AI 的特殊性：与云端推理的 5 大差异](#1-端侧-ai-的特殊性与云端推理的-5-大差异)
2. [模型层优化 1：量化（Quantization）](#2-模型层优化-1量化quantization)
3. [模型层优化 2：剪枝（Pruning）与稀疏化](#3-模型层优化-2剪枝pruning与稀疏化)
4. [模型层优化 3：知识蒸馏（KD）与架构改造](#4-模型层优化-3知识蒸馏kd与架构改造)
5. [引擎层优化 1：算子融合（Operator Fusion）](#5-引擎层优化-1算子融合operator-fusion)
6. [引擎层优化 2：内存规划与张量重用](#6-引擎层优化-2内存规划与张量重用)
7. [引擎层优化 3：内核（Kernel）极致优化](#7-引擎层优化-3内核kernel极致优化)
8. [运行时优化 1：多流与异构调度](#8-运行时优化-1多流与异构调度)
9. [运行时优化 2：动态 shape 与 KV cache 优化](#9-运行时优化-2动态-shape-与-kv-cache-优化)
10. [硬件层：CPU SIMD、NPU、GPU、专用加速器](#10-硬件层cpu-simdnpugpu专用加速器)
11. [主流端侧推理框架全景对比](#11-主流端侧推理框架全景对比)
12. [LLM 端侧化：2026 年的最大变量](#12-llm-端侧化2026-年的最大变量)
13. [车载智驾域 NPU 部署：Orin / Thor / Ride / J6 / X9](#13-车载智驾域-npu-部署orin--thor--ride--j6--x9)
14. [与 AUTOSAR Adaptive 的集成形态](#14-与-autosar-adaptive-的集成形态)
15. [性能评估方法论：怎么测、怎么比、怎么信](#15-性能评估方法论怎么测怎么比怎么信)
16. [2026 趋势与未解问题](#16-2026-趋势与未解问题)
17. [学习路径与速查表](#17-学习路径与速查表)

---

## 1. 端侧 AI 的特殊性：与云端推理的 5 大差异

| 维度 | 云端推理 | 端侧推理 |
|---|---|---|
| 算力 | H100 / A100 单卡 1000+ TFLOPS | 手机 NPU 30-50 TOPS；车端 NPU 100-2000 TOPS |
| 内存 | 80 GB HBM | 手机 8-16 GB 统一内存；车端 16-64 GB LPDDR5 |
| 功耗 | 服务器 700 W | 手机芯片 5-8 W；车载 30-100 W |
| 延迟 | 网络 RTT 主导（50-200 ms） | 本地 5-50 ms |
| 模型规模 | 70B+ 几乎无压力 | 通常 0.5-7B（手机）、3-14B（车端） |
| 部署频率 | 实时更新 | OTA 升级数月一次 |
| 量化要求 | 几乎不量化（FP16 / BF16） | 必须量化（INT8/INT4） |
| 关键优化目标 | 吞吐 (img/s) | **延迟 + 功耗 + 内存** |

**核心权衡公式**：

$$
\text{端侧模型选择} = \arg\min_{\text{model}} \left( \alpha \cdot \text{latency} + \beta \cdot \text{memory} + \gamma \cdot \text{accuracy\_loss} + \delta \cdot \text{power} \right)
$$

α/β/γ/δ 权重随场景剧烈变化：智驾感知看重 α（延迟）、嵌入式看重 β（内存）、手机相机看重 δ（功耗）。

---

## 2. 模型层优化 1：量化（Quantization）

### 2.1 量化基本概念

把 FP32 → INT8/INT4/FP8，用更少 bit 表达同一数值范围：

| 类型 | bit 数 | 范围 | 典型用途 |
|---|---|---|---|
| FP32 | 32 | ±3.4×10³⁸ | 训练基线 |
| FP16 / BF16 | 16 | ±65504 / ±3.4×10³⁸ | 训练 + 高端推理 |
| FP8 (E4M3 / E5M2) | 8 | ±448 / ±57344 | H100 / 端侧 NPU 新兴 |
| INT16 | 16 | -32768~32767 | 早期端侧 |
| **INT8** | 8 | -128~127 | **端侧主流** |
| UINT8 | 8 | 0~255 | 端侧主流（对称/非对称） |
| **INT4** | 4 | -8~7 | LLM 端侧主流 |
| INT2 / Binary | ≤2 | ±1/±2 | 极端压缩（学术界） |

### 2.2 量化粒度

| 粒度 | 含义 | 精度 | 推理开销 |
|---|---|---|---|
| Per-tensor | 整层共享一个 scale/zero | 最差 | 最低 |
| Per-channel | 每个输出通道一个 scale | 较好 | 略高（向量化友好） |
| Per-token | 每个 token 一个 scale（LLM） | 好（weight 仍 per-channel） | 高 |
| Per-group | 每 32/64/128 个元素一个 scale | 极好 | 中（GPTQ 风格） |

LLM 量化常用 **per-channel weight + per-token activation + per-group KV cache** 三件套。

### 2.3 训练后量化（PTQ）vs 量化感知训练（QAT）

| 维度 | PTQ | QAT |
|---|---|---|
| 校准数据 | 少量（100-1000 张） | 全量训练数据 |
| 训练成本 | 几乎为零 | 需重新训练 |
| 精度损失 | 较大（尤其 INT4） | 接近 FP32 |
| 适用 | 端侧快速上线 | 量产 / 关键任务 |

PTQ 三步流程：

```
[FP32 模型] → ① 准备校准集 → ② 统计激活分布 → ③ 计算 scale/zero → [INT8 模型]
```

### 2.4 INT4 / INT8 混合量化策略

LLM 端侧化的事实标准：

```
embedding / output  → FP16  （精度敏感）
attention QKV     → INT8  （中间稳定）
attention proj    → INT8
MLP up / down     → INT4  （参数量大、可压）
KV cache          → INT4 / FP8
LayerNorm        → FP16
```

**SmoothQuant**（Xiao et al. 2022）解决"激活 outlier 难量化"问题：把激活的难度按 1/α 转移到 weight 上，让 INT8 量化跑得通。

**GPTQ**（Frantar et al. 2022）用二阶 Hessian 信息分组量化 INT4，精度损失 < 0.5 PPL。

**AWQ**（Lin et al. 2023）观察"权重中 1% 的 outlier 占大部分误差"，把这些权重保留 FP16，其余 INT4——比 GPTQ 推理更快（不需要反量化矩阵）。

### 2.5 量化精度损失的可接受范围

| 任务 | 可接受精度损失 |
|---|---|
| 分类（ImageNet） | < 1% top-1 |
| 检测（VOC/COCO） | < 1 mAP |
| 分割（Cityscapes） | < 1 mIoU |
| LLM 困惑度（PPL） | < 0.5 |
| ASR（语音识别） | < 1% WER |
| 智驾感知（BEV / Occupancy） | **< 0.5 mIoU**（安全敏感） |

---

## 3. 模型层优化 2：剪枝（Pruning）与稀疏化

### 3.1 三大剪枝维度

| 维度 | 非结构化 | 结构化 | 模式化 |
|---|---|---|---|
| 粒度 | 单个权重 | 整个 channel / filter | 整组（如 4x4 块） |
| 硬件友好 | ❌（要稀疏计算单元） | ✅（直接变瘦） | ✅（NPU 高效） |
| 加速比 | 理论高、实际难兑现 | 接近线性 | 中等 |
| 典型算法 | Magnitude, SNIP | Channel Prune, AMC | N:M 稀疏（如 2:4） |

### 3.2 N:M 稀疏：硬件的事实标准

NVIDIA Ampere 引入 2:4 稀疏——每 4 个连续权重中至少 2 个为 0：

```text
[0.1, 0,    0.2, 0  ]
[0,    0.3, 0,    0.4]   ←  2:4 模式
```

- 矩阵乘法硬件加速 2×
- 模型大小减少 ~50%
- 端侧 NPU（高通 Hexagon V73、苹果 ANE）逐步支持

### 3.3 剪枝工作流

```
[训练好的 FP32 模型]
        ↓ ① 重要性评分（权重 / 激活 / 梯度）
[稀疏化掩码]
        ↓ ② 微调（恢复精度）
[稀疏 INT8 模型]
        ↓ ③ 硬件部署
```

---

## 4. 模型层优化 3：知识蒸馏（KD）与架构改造

### 4.1 知识蒸馏三件套

| 类型 | 教师输出 | 学生学习什么 |
|---|---|---|
| Logit 蒸馏 | Softmax 概率 | 类别间暗含的相似度关系 |
| Feature 蒸馏 | 中间层特征图 | 中间表示 |
| Relation 蒸馏 | 层间 / 样本间关系 | 数据结构 |

经典损失：

$$
L_{\text{KD}} = \alpha \cdot L_{\text{CE}}(y, \sigma(z_s)) + \beta \cdot L_{\text{KL}}(\sigma(z_t/T), \sigma(z_s/T))
$$

### 4.2 自蒸馏（Self-Distillation）

让学生 = 教师，只通过数据增强 / 多次训练"自己教自己"。代表作：

- **MobileBERT**：BERT 大模型 → 24 层小模型
- **TinyLlama**：Llama 2 → 1.1B 学生
- **DistilBERT**、**MiniLMv2**、**MobileLLM**

### 4.3 架构改造：让小模型跑得更快

不是简单压参数，而是**重设计**：

| 替代 | 原版 | 优势 |
|---|---|---|
| **Depthwise Separable Conv** | 标准 Conv | FLOPs ↓ 8-9× |
| **Inverted Bottleneck** | 残差块 | 移动端友好 |
| **Group Conv** | 标准 Conv | 参数量 ↓ |
| **ReLU6 / h-swish** | ReLU | 量化友好 |
| **SE / ECA 注意力** | FC | 轻量通道注意力 |
| **MQA / GQA** | MHA | LLM KV cache ↓ 4-8× |
| **滑动窗口 Attention** | 全局 Attention | LLM 长序列加速 |
| **SSM（Mamba）/ RWKV** | Transformer | O(n) 复杂度 |

代表架构：

- **MobileNetV3** / **V4**（手机 CNN 标杆）
- **EfficientNetV2** / **EfficientFormerV2**（混合 CNN-Transformer）
- **MobileLLM** / **MiniCPM** / **Phi-3-mini**（手机 LLM）
- **YOLOv8-Nano** / **YOLOv9-Tiny**（实时检测）

---

## 5. 引擎层优化 1：算子融合（Operator Fusion）

### 5.1 为什么需要融合

未融合的执行模式：

```
Conv → 写中间张量到内存 → BN → 读 → 计算 → 写
                                   ↓
                              ReLU → 读 → 计算 → 写
```

每次"读写中间张量"都触发一次 **DDR 访问**——这是端侧最大的延迟源。算子融合把多个算子合并为一个 kernel：

```
Conv + BN + ReLU = 一个 kernel，只读一次输入、只写一次输出
```

### 5.2 融合类型

| 融合类型 | 例子 | 收益 |
|---|---|---|
| 横向融合（horizontal） | 同种 op 多输入并行 | 减少 kernel launch |
| 纵向融合（vertical） | Conv → BN → ReLU | 减少内存往返 |
| 拆分融合（split） | 一个大 kernel 拆 N 个小核 | 适配异构硬件 |
| 算子替换（rewriter） | BN → Conv 折叠 | 减少 op 数 |

### 5.3 实战案例

**Conv-BN-ReLU 折叠**：

```python
# 训练时：BN 有自己的 γ/β/μ/σ²
# 推理时：等价为新的 Conv 权重
W_fused = W * γ / sqrt(σ² + ε)
b_fused = (b - μ) * γ / sqrt(σ² + ε) + β
# 一个 Conv 顶三个算子
```

**ONNX Runtime / TFLite / MNN 都默认做 Conv-BN 折叠**，无需用户干预。

### 5.4 融合的极限

不是所有算子都该融合：

- **巨型 Conv**（参数量 > 100 MB）：融合其他小算子收益小，反而占 register
- **激活张量复用**：Eltwise + Mul 的输入被多次使用 → 融合反而吃亏
- **不同硬件亲和**：CPU 跑 Conv、NPU 跑 Softmax → 强制融合可能拖慢

**TVM / Ansor / TensorRT** 自动搜索最优融合策略，典型加速 1.5-3×。

---

## 6. 引擎层优化 2：内存规划与张量重用

### 6.1 内存是端侧的"第一约束"

一个 ResNet-50 FP32 模型中间特征图峰值：

```
输入 224×224×3    =    600 KB
第 1 层输出       =   3.2 MB
第 10 层输出      =  6.4 MB
第 30 层输出      = 12.8 MB
峰值             ≈ 50 MB
```

INT8 后变成 12.5 MB——这还只是 1 张图。多 batch / 多 stream 一上来就吃满移动端内存。

### 6.2 静态内存规划

推理前 **分析整张计算图** 的张量生命周期，找到可重用的内存块：

```python
# 伪代码：liveness analysis
for each tensor:
    birth = first use
    death = last use
    if tensor_A.death < tensor_B.birth:
        reuse same memory block
```

经典实现：

- **MNN** 的静态规划器
- **TensorFlow Lite** 的 arena allocator
- **TVM** 的 memory planning pass

### 6.3 内存复用率

| 框架 | 复用率（典型模型） |
|---|---|
| PyTorch Eager | 30-50% |
| ONNX Runtime | 60-80% |
| MNN | 75-90% |
| TFLite (XNNPACK) | 70-85% |
| TensorRT | 80-95% |
| 手写汇编 | 90%+（极端） |

### 6.4 动态内存策略

模型有动态 shape（NLP、检测）时，静态规划不够用：

- **预分配池子**（memory pool）：按峰值预留，运行时不释放
- **动态重规划**：每 N 次推理重新分配
- **offload**：暂时不用的中间张量换到 SSD（极端嵌入式）

---

## 7. 引擎层优化 3：内核（Kernel）极致优化

### 7.1 卷积优化的三个层次

| 层次 | 方法 | 典型收益 |
|---|---|---|
| 算法层 | Winograd、FFT、Direct Conv | 2-4× |
| 数据排布层 | NCHW → NHWC → NCHW4/NHWC8（NPU 友好） | 1.5-2× |
| 微架构层 | 向量化、tiling、prefetch、unroll | 1.5-3× |

### 7.2 Winograd 卷积

标准 Conv F(2×2, 3×3)：每个输出点 9 乘法
Winograd F(2×2, 3×3)：每个输出点 4 乘法（理论 2.25×）

```text
输入 4×4  ──transform──> 4×4
权重 3×3  ──transform──> 4×4
element-wise mul ──> 4×4 ──inverse transform──> 2×2 输出
```

代价：数值精度变差 → 量化后需配合 INT8 重校准。

### 7.3 GEMM-based 卷积

把 Conv 展开成 **im2col → GEMM**：

```
input [N, C, H, W] → im2col → [N, C*Hf*Wf, OH*OW]
weight [K, C, Hf, Wf] → reshape → [K, C*Hf*Wf]
output = weight @ im2col_im
```

优势：直接复用 BLAS / cuBLAS / 厂商优化库

代价：im2col 内存翻 9 倍 → **NHWC + 直接卷积** 成为端侧主流

### 7.4 端侧 SIMD 指令集

| 架构 | SIMD | 典型用法 |
|---|---|---|
| ARMv7 | NEON 128-bit | float32 ×4、int8 ×16 |
| ARMv8 (AArch64) | NEON 128-bit / SVE / SME | SVE 可变长、SME 矩阵扩展 |
| ARM Cortex-M | Helium (MVE) | 128-bit 矢量 |
| x86 AVX-512 | 512-bit VNNI | INT8/INT4 矩阵点积加速 |
| Apple M-series | AMX | 矩阵 tile |
| RISC-V V | Vector 扩展 | 嵌入式新兴 |
| 国产芯 | 自研向量 | 寒武纪 MLU、地平线 BPU、芯驰 NPU |

**VNNI（Vector Neural Network Instructions）** 是 x86 INT8 推理的关键：

```
VPDPBUSD  zmm, zmm, mem    ; 一条指令：INT8 * INT8 + INT32 累加
```

### 7.5 Kernel 优化的隐藏地雷

- **Cache miss**：tile 大小 > L1 cache → 性能崩盘
- **Bank conflict**：共享内存同 bank 访问 → 串行化
- **未对齐访问**：ARM 上 128-bit 跨 16 字节边界 → 慢 50%
- **分支预测**：kernel 里 if 越少越好（SIMD lane 同步）

端侧 kernel 通常要 **手写汇编** + **大量 benchmark**（如 XNNPACK、QNNPACK、gemmlowp、NNPACK）。

---

## 8. 运行时优化 1：多流与异构调度

### 8.1 多流（Multi-Stream）

端侧常同时跑多个模型：

```
Stream A: 摄像头预览（30 FPS，CPU）
Stream B: 检测模型 YOLO（30 FPS，NPU）
Stream C: 跟踪模型（30 FPS，NPU）
Stream D: 语音识别（10 FPS，CPU DSP）
```

调度策略：

- **静态 partition**：预先给每个 stream 分算力
- **优先级抢占**：A/D 高优先级，B/C 低优先级
- **共享池**：空闲 stream 把 NPU 让给忙碌 stream

### 8.2 异构调度（Heterogeneous）

CPU + NPU + GPU + DSP 各自擅长：

| 算子类型 | 最优硬件 |
|---|---|
| 大矩阵乘 (GEMM) | NPU / GPU |
| Element-wise | CPU |
| 小 Conv、Pooling | CPU / DSP |
| 控制流（if / loop） | CPU |
| Softmax / LayerNorm | CPU（复杂算子） |
| Embedding lookup | CPU / 专用 SRAM |

### 8.3 调度器实现

| 框架 | 调度器 |
|---|---|
| TFLite | XNNPACK + delegate（CPU/GPU/NPU） |
| ONNX Runtime | Execution Provider（CUDA/TensorRT/CPU/NNAPI/CoreML） |
| MNN | OpenCL / Metal / NPU backend |
| TVM | Heterogeneous scheduling |
| llama.cpp | CPU + Metal + CUDA 多 backend |

### 8.4 PipeDream / 流水线并行

多 stage 模型并行推理：

```
Frame 1: Camera → Preprocess → NPU (stage 1)
Frame 2: Camera → Preprocess → NPU (stage 2)
Frame 3: Camera → Preprocess → NPU (stage 3)
```

吞吐提升 2-3×，但延迟不变。常用于车端多摄像头的融合感知。

---

## 9. 运行时优化 2：动态 shape 与 KV cache 优化

### 9.1 动态 shape 是端侧的新常态

早期：固定 batch = 1、固定 input size
现在：NLP（变长序列）、检测（变目标数）、BEV（变 batch）

挑战：每次 shape 变就要重新做内存规划 → **占大头**

### 9.2 KV cache：LLM 端侧的最大开销

LLM 自回归推理每生成一个 token 都要重新算一次 attention：

$$
\text{Attention}(Q, K, V) = \text{softmax}\left(\frac{QK^\top}{\sqrt{d}}\right)V
$$

把历史 $K, V$ 缓存起来 → **KV cache**：

| 模型 | 7B 模型 KV cache（FP16） | INT4 量化后 |
|---|---|---|
| Llama 2 7B（2k context） | 1.0 GB | 250 MB |
| Llama 2 7B（8k context） | 4.0 GB | 1.0 GB |
| Llama 2 7B（32k context） | 16 GB | 4 GB |

### 9.3 KV cache 优化 5 把斧

| 技术 | 节省 |
|---|---|
| **GQA / MQA** | KV cache ↓ 4-8×（共享 KV） |
| **PagedAttention（vLLM 风格）** | 内存碎片 ↓ 50-70% |
| **KV cache 量化（INT4/INT8）** | 内存 ↓ 2-4× |
| **Sliding Window Attention** | 内存有界 |
| **Flash Attention** | 不存中间注意力矩阵 |

**PagedAttention** 借鉴操作系统虚拟内存的页表思想，把 KV cache 切成固定大小页，按需分配 → 端侧 / 车端 7B 模型长上下文必备。

### 9.4 投机解码（Speculative Decoding）

端侧 LLM 的"加速神器"：

```
Draft 模型（小、快）  →  生成 K 个候选 token（每步 ~5ms）
Target 模型（大、准）  →  一次性验证 K 个 token（~15ms）

收益：1.5-3× 加速（小模型能猜对大部分 token）
```

代表：

- **SpecDec** / **SpecInfer**
- **Medusa**（多头并行预测）
- **EAGLE**（特征级 draft）

---

## 10. 硬件层：CPU SIMD、NPU、GPU、专用加速器

### 10.1 硬件全景

| 类型 | 代表 | 算力 | 适用 |
|---|---|---|---|
| **CPU SIMD** | ARM NEON、x86 AVX-512 | 几 GFLOPS-100 GFLOPS | 控制流、小算子 |
| **DSP** | Hexagon V73、CEVA XM | 几十 TOPS | 语音、信号处理 |
| **GPU** | Mali、Adreno、Apple GPU | 100-500 GFLOPS | 并行算子、INT8 矩阵乘 |
| **NPU（通用）** | 高通 Hexagon NPU、Apple ANE、MediaTek APU | 30-100 TOPS | 主流 CNN/Transformer |
| **NPU（大算力）** | NVIDIA Orin/Thor、华为 MDC | 100-2000 TOPS | 车端大模型 |
| **专用加速器** | Groq LPU、Cerebras | 极高 | 数据中心（端侧罕见） |

### 10.2 主流车端 NPU

| SoC | NPU 算力 | 工艺 | 代表车型 / 平台 |
|---|---|---|---|
| NVIDIA Orin | 275 TOPS | 8nm | 蔚来 ET7、小鹏 G9、理想 L9 |
| NVIDIA Thor | 2000 TOPS | 4nm | 2025-2026 量产 |
| 高通 Ride 8775 | 700 TOPS | 5nm | 智己、极氪 |
| 高通 Ride 8797 | 1300 TOPS | 4nm | 2026 量产 |
| 地平线 J5 | 128 TOPS | 16nm | 理想 L7 早期 |
| 地平线 J6 | 560 TOPS | 7nm | 2025-2026 量产 |
| 芯驰 X9 | 50 TOPS | 12nm | 入门级智驾 |
| 黑芝麻华山 A1000 | 196 TOPS | 16nm | 吉利、东风 |
| 华为 MDC 610 | 200+ TOPS | 7nm | 问界、阿维塔 |
| 寒武纪 MLU 370 | 256 TOPS | 7nm | 比亚迪（实验） |
| TI TDA4VH | 32 TOPS | — | L2 入门 |

### 10.3 NPU 编程模型

| NPU | 编程方式 |
|---|---|
| 高通 QNN | QNN SDK + 量化工具 |
| Apple ANE | Core ML 编译器自动映射 |
| NVIDIA Orin | TensorRT / Triton |
| 地平线 | 天工开物工具链 + 量化感知 |
| 华为 MDC | CANN + MindSpore / TensorFlow |
| 黑芝麻 | 瀚海工具链 |

**行业现实**：每家 NPU 都要求用自家编译器 + 量化工具链，模型要在每家重做量化。这是端侧 AI 工程化的最大痛点之一。

### 10.4 硬件亲和的模型设计

| 硬件偏好 | 模型设计建议 |
|---|---|
| Apple ANE | Conv 通道对齐 16/32 |
| 高通 Hexagon | 通道对齐 32、INT8 严格 |
| NVIDIA TensorRT | FP16/INT8 自由，attention 优化 |
| 地平线 BPU | 算子白名单限制 |
| 华为 CANN | Ascend 自有算子集 |

---

## 11. 主流端侧推理框架全景对比

### 11.1 总览表

| 框架 | 来源 | 优点 | 缺点 | 适用 |
|---|---|---|---|---|
| **ONNX Runtime** | Microsoft | 跨平台、EP 多 | 编译产物大 | 通用 |
| **TensorFlow Lite** | Google | 生态成熟 | TF 绑定深 | 手机、Android |
| **PyTorch Mobile** | Meta | PyTorch 直出 | 性能一般 | 原型 |
| **Core ML** | Apple | ANE 自动加速 | 仅 Apple | iOS / macOS |
| **TensorRT** | NVIDIA | 极致性能 | 仅 NVIDIA | Orin/Thor |
| **MNN** | 阿里 | 端侧性能强 | 文档弱 | 手机 |
| **NCNN** | 腾讯 | 极致轻量、CPU 强 | NPU 支持弱 | 老旧手机 |
| **TVM** | Apache | 自动编译优化 | 学习曲线陡 | 跨硬件 |
| **OpenVINO** | Intel | x86 强 | 仅 Intel 优化 | 工控 |
| **llama.cpp** | 开源 | LLM 端侧标杆 | 仅 LLM | LLM 部署 |
| **MediaPipe** | Google | 多模态 | 灵活性差 | 视觉 |
| **Tengine** | OPEN AI LAB | 国产端侧 | 生态小 | 国产芯片 |

### 11.2 选型决策

```
                              ┌─ 仅 Apple？→ Core ML
                              │
你是哪种硬件？ ──────────────├─ NVIDIA？→ TensorRT / ONNX Runtime CUDA EP
                              │
                              ├─ 国产 NPU？→ 厂商 SDK（MNN/Tengine/天工开物）
                              │
                              └─ 通用 CPU？→ ONNX Runtime / TFLite / MNN / NCNN
                                          │
                                          ├─ 主要跑 LLM？→ llama.cpp
                                          │
                                          └─ 自动优化？→ TVM / Ansor
```

### 11.3 性能基准（粗略）

YOLOv8-N INT8，端侧 640×640，单 batch：

| 框架 | Snapdragon 8 Gen 3 | Apple A17 Pro |
|---|---|---|
| TFLite + XNNPACK | 8 ms | 6 ms |
| ONNX Runtime | 9 ms | 7 ms |
| MNN | 7 ms | 5 ms |
| NCNN | 11 ms | — |
| 厂商 NPU SDK | 4-5 ms | 3-4 ms (ANE) |

---

## 12. LLM 端侧化：2026 年的最大变量

### 12.1 端侧 LLM 现状

```
2023 ：MLC-LLM 首发，iPhone 跑 7B（5-10 token/s）
2024 ：llama.cpp + GGUF 格式成为端侧事实标准
2025 ：量化（INT4/INT8）+ 投机解码让 7B 模型在手机 30+ token/s
2026 ：3B 多模态、7B 中文、车载 14B VLM 进入量产
```

### 12.2 端侧 LLM 关键技术

| 技术 | 作用 |
|---|---|
| INT4 量化 | 7B 模型从 13 GB → 4 GB |
| Q4_K_M / Q5_K_M | 量化方案（llama.cpp 系列） |
| Speculative Decoding | 1.5-3× 加速 |
| PagedAttention | 长上下文必备 |
| Flash Attention | attention 计算省内存省时间 |
| Continuous Batching | 高吞吐推理 |
| Prefix Caching | 共享 prompt 的 prefix |

### 12.3 主流端侧 LLM 模型

| 模型 | 大小 | 端侧推理速度（手机） |
|---|---|---|
| **Phi-3-mini** 3.8B | 2.3 GB (Q4) | 25-40 token/s |
| **Gemma 2 2B** | 1.6 GB (Q4) | 50-80 token/s |
| **Llama 3.2 1B/3B** | 0.8/2.0 GB (Q4) | 60-100 / 30-50 token/s |
| **Qwen2 1.5B/7B** | 1.0/4.5 GB (Q4) | 60-80 / 25-35 token/s |
| **MiniCPM 2B** | 1.5 GB (Q4) | 40-60 token/s |
| **Llama 3.1 8B** | 5 GB (Q4) | 15-25 token/s |
| **Llama 3.1 70B** | 40 GB | 需云端或工作站 |

### 12.4 车端 LLM

2026 年的车端 VLM：

- **理想 MindGPT**（车机 LLM，3B 端云协同）
- **斑马智行元神AI**（智己汽车 VLM）
- **小鹏 XOS 5.x**（端云双模型）
- **NVIDIA Alpamayo-R1**（车端 VLM 研发中）
- **DriveGPT**（特斯拉端侧 FSD LLM）

车端与手机的关键差异：

- 算力更多（200-1000 TOPS NPU）
- 内存更大（16-64 GB）
- 功耗预算宽（30-100 W）
- 模型规模可上探到 7-14B

---

## 13. 车载智驾域 NPU 部署：Orin / Thor / Ride / J6 / X9

### 13.1 典型车载感知模型

| 模型 | 输入 | 输出 | 帧率 | 算力需求 |
|---|---|---|---|---|
| YOLOv8-N | 摄像头 640×640 | 检测框 | 30 Hz | ~5 TOPS |
| BEVFusion | 6 cam + 1 LiDAR | BEV 特征 | 20 Hz | ~100 TOPS |
| Occupancy Network | 6 cam | 3D 占用 | 20 Hz | ~80 TOPS |
| LaneNet | 前视摄像头 | 车道线 | 30 Hz | ~10 TOPS |
| SLABFusion | 6 cam + LiDAR + Radar | 4D 占用 | 10 Hz | ~150 TOPS |
| End-to-End Planner | BEV + 导航 | 轨迹 | 10 Hz | ~100 TOPS |
| VLM Driver | 6 cam + 导航 | 自然语言 / 决策 | 1-5 Hz | ~200 TOPS |

### 13.2 模型量化 + 编译流程

```
[PyTorch / ONNX 模型]
        ↓ ① PTQ / QAT 量化 (INT8 / FP16)
[量化 ONNX]
        ↓ ② 厂商编译器 (TensorRT / 天工开物 / CANN)
[NPU 可执行模型 (.engine / .bin / .om)]
        ↓ ③ 运行时部署 (TensorRT Runtime / 自研)
[车载 SoC 上跑]
```

### 13.3 多模型部署的工程挑战

- **算力分时复用**：BEVFusion 跑 20 Hz，VLM 跑 1 Hz，总算力 ≈ 150 TOPS / 周期
- **内存峰值管理**：6 路摄像头预处理 + BEV 特征图 + KV cache（VLM）→ 需 8-12 GB
- **功耗管理**：NPU 全速 30 W+ → 需温控策略（限频 / 降帧）
- **延迟 SLO**：感知 50 ms、规划 100 ms、决策 200 ms → 端到端 < 500 ms

### 13.4 端云协同

```
┌─────────────────────────────────────────────────────┐
│                    端云协同                            │
│                                                     │
│   [车端 NPU]                  [云端 GPU]             │
│   实时感知（YOLO、BEV）       大模型（VLM、规划）     │
│   紧急决策（AEB）             复杂预测（路径生成）    │
│                                                     │
│   ↑                       ↓                         │
│   └─── 4G/5G ─── 关键场景回传 ───────────────────   │
└─────────────────────────────────────────────────────┘
```

---

## 14. 与 AUTOSAR Adaptive 的集成形态

### 14.1 集成架构

```
┌────────────────────────────────────────────────────────────┐
│                   AUTOSAR Adaptive (域控)                  │
│                                                            │
│   [Sensor AA] ──→ [Preprocess AA] ──→ [Inference AA]      │
│       │                                       │            │
│       │            [Memory Pool] ←────────────┤            │
│       │                                       ↓            │
│       │                              [Postprocess AA]      │
│       │                                       ↓            │
│       │                              [Fusion AA]           │
│       │                                       ↓            │
│       └──────────────────────────────→ [Planner AA]       │
│                                                            │
└────────────────────────────────────────────────────────────┘
                            ↑↓
                   ┌─────────────────────────────────┐
                   │   Classic AUTOSAR (ECU)         │
                   │   - 电机控制、制动、转向         │
                   └─────────────────────────────────┘
```

### 14.2 Inference AA 的内部结构

```cpp
class InferenceApp : public ara::core::Application {
    void OnInitialize() {
        // 1. 加载模型
        model_ = LoadModel("bevfusion_int8.engine");
        
        // 2. 分配 NPU 张量内存
        input_buffers_ = AllocateGpuMemory(...);
        
        // 3. 注册周期性执行
        timer_ = ara::exec::CreateTimer(20ms, [this]() { RunInference(); });
    }
    
    void RunInference() {
        // 1. 拷贝传感器数据 → NPU 输入
        // 2. 触发推理
        // 3. 拷贝 NPU 输出 → CPU
        // 4. 通过 ara::com 发布结果
        publisher_->Send(perception_result_);
    }
};
```

### 14.3 关键集成点

| 集成点 | 说明 |
|---|---|
| 模型 OTA | 通过 `ara::per` 持久化、通过 `ara::com` 触发更新 |
| 模型签名 | 加密签名 + 安全启动（AUTOSAR 标准要求） |
| 推理确定性 | 异构调度 → 严格周期保证（部分 SoC 支持） |
| 失败处理 | NPU 故障 → fallback 到 CPU 推理（慢但稳） |
| 功能安全 | SOTIF + ASIL B/D 分解，感知部分 ASIL B |

### 14.4 与 POSIX 实时对比

| 维度 | POSIX 实时（裸 Linux） | AUTOSAR Adaptive |
|---|---|---|
| 实时性 | RT-PREEMPT 可达 100 µs | 1-10 ms（取决于调度） |
| 调度 | 自定义 | 标准调度策略 |
| 通信 | 自定义 IPC | ara::com SOME/IP / DDS |
| 安全认证 | 难 | 有完整认证路径 |
| AI 适配 | 灵活 | 受 EE 架构约束 |

**主流车厂选择**：AI 部分在 POSIX / 自定义 Adaptive 域，底盘控制在 Classic AUTOSAR。

---

## 15. 性能评估方法论：怎么测、怎么比、怎么信

### 15.1 核心指标

| 指标 | 含义 | 测量方式 |
|---|---|---|
| **Latency** | 单次推理时间 | `std::chrono`、nsys、TensorRT profiler |
| **Throughput** | QPS（queries per second） | benchmark 跑满 |
| **Memory Peak** | 峰值内存 | valgrind / Massif / Android Profiler |
| **Power** | 平均功耗 | 示波器、内置 PMU |
| **Accuracy** | 模型精度 | 验证集 |
| **Cold-start** | 首次推理时间 | 包含模型加载 |
| **Warm Latency** | 热启动后稳定延迟 | 跑 100 次后取 P99 |

### 15.2 Latency 分布关注

- **P50**（中位数）：常见场景
- **P95 / P99**（长尾）：决定 SLA
- **Max**：决定最差用户体验

LLM 端侧常用 **TTFT（Time To First Token）** 和 **TPOT（Time Per Output Token）**。

### 15.3 基准测试工具

| 工具 | 用途 |
|---|---|
| **MLPerf Mobile / Tiny** | 端侧基准（业界标准） |
| **AITemplate** | NVIDIA 推理基准 |
| **ONNX Runtime benchmark** | ONNX 跨平台基准 |
| **TensorRT trtexec** | NVIDIA 详细 profile |
| **Qualcomm AI Engine SDK** | 高通端侧基准 |
| **Apple Neural Engine benchmark** | Apple ANE 基准 |

### 15.4 评估陷阱

| 陷阱 | 真相 |
|---|---|
| "TFLite 比 ONNX Runtime 快" | 在 GPU 上 TFLite 优化好，在 CPU 上 ONNX 更好 |
| "FP16 比 INT8 快" | 通常是的，但有时 INT8 + 大 batch 反而快 |
| "NPU 一定比 CPU 快" | 小模型 < 1ms 任务 NPU 启动开销可能更大 |
| "我的模型跑 50 ms" | 1 ms × 50 = 50 ms，但加上输入拷贝 + 后处理可能 80 ms |

### 15.5 一致性保证

- **固定硬件**（同款 SoC、同款频率）
- **固定温升**（冷机 vs 热机可能差 30%）
- **固定后台**（关闭系统更新推送等）
- **多次取 P99**（单次测量不可信）

---

## 16. 2026 趋势与未解问题

### 16.1 趋势

1. **端侧 LLM 成为标配**：
   - 手机/车机 NPU 算力继续翻倍
   - 3B 多模态模型在端侧 100+ token/s
2. **量化进入 INT3 / FP4 时代**：
   - NVIDIA Blackwell 已支持 FP4，2026 年端侧开始跟进
3. **端云协同成为标准架构**：
   - 端侧小模型 + 云端大模型协同
4. **编译器取代手工优化**：
   - TVM / Ansor / Modular 持续吃掉手工 kernel
5. **车端 NPU + 域控 + AUTOSAR 深度融合**：
   - "感知-规控-决策"全部端侧化
6. **Rust 进入端侧推理引擎**：
   - 安全性 + 性能兼顾（Hugging Face candle-rs、tch-rs）

### 16.2 未解问题

| 问题 | 现状 |
|---|---|
| 长上下文（128k+）端侧 LLM | 内存仍是瓶颈，PagedAttention 部分缓解 |
| 多模态实时融合（视频 + 音频 + IMU） | 模型架构未定型 |
| 端侧训练（on-device training） | 内存、算力都不够，仅联邦学习小步前进 |
| NPU 编程标准化 | 每家厂商 SDK 各异，碎片化严重 |
| 量化 + 功能安全 | INT8 模型如何通过 ASIL 认证，无成熟路径 |
| 模型 IP 保护 | 端侧模型易被 dump，硬件级加密是方向 |

---

## 17. 学习路径与速查表

### 17.1 推荐学习顺序

1. **深度学习基础**（花书 / CS231n）→ CNN / Transformer / 损失函数
2. **模型部署入门**（《深度学习嵌入式部署》）→ ONNX / TFLite
3. **量化与剪枝**（Quantization and Training of Neural Networks for Efficient Integer-Arithmetic-Only Inference）
4. **推理引擎源码**（MNN / NCNN / llama.cpp）→ 理解算子融合、内存规划
5. **TVM / Ansor** → 自动编译优化
6. **车端 NPU 工具链**（TensorRT / 天工开物）→ 实战部署

### 17.2 关键论文

- Jacob et al., "Quantization and Training of Neural Networks for Efficient Integer-Arithmetic-Only Inference" (2018)
- Lin et al., "AWQ: Activation-aware Weight Quantization" (2023)
- Frantar et al., "GPTQ: Accurate Post-Training Quantization for Generative Pre-trained Transformers" (2022)
- Rombach et al., "High-Resolution Image Synthesis with Latent Diffusion Models" (2022) (Stable Diffusion)
- Kwon et al., "PagedAttention: Virtual Memory for LLM Serving" (2023)
- Touvron et al., "Llama 2: Open Foundation and Fine-Tuned Chat Models" (2023)
- Dao et al., "FlashAttention: Fast and Memory-Efficient Exact Attention" (2022)

### 17.3 速查公式

```
模型大小估计：
  size(MB) ≈ params × bits_per_weight / 8 / 1024²

INT8 量化推理加速比（理论）：
  speedup = 4 (计算密度) × memory_bandwidth_improvement

LLM 端侧推理速度估算（经验）：
  tokens_per_second ≈ NPU_TOPS × utilization / (model_params × bits_per_weight / 8)
                     ≈ 100 × 0.4 / (7e9 × 4 / 8) ≈ 11 tokens/s
```

### 17.4 一句话总结

> **端侧 AI 性能优化 = 模型压缩（量化 / 剪枝 / 蒸馏）+ 引擎优化（融合 / 内存规划 / Kernel）+ 运行时调度（异构 / KV cache / 投机解码）+ 硬件亲和**——它是 2026 年 AI 落地的"硬功夫"，从手机到车机都离不开这套技术栈。

---

## 附录：与本仓库其他文档的交叉引用

- [`运动控制WBC技术详解.md`](./运动控制WBC技术详解.md) — 端侧 AI 与 WBC 协同（上层决策 + 下层控制）
- [`运动控制FOC技术详解.md`](./运动控制FOC技术详解.md) — WBC 输出 → FOC 输入，控制栈最底层
- [`AUTOSAR-Adaptive-平台详解.md`](./AUTOSAR-Adaptive-平台详解.md) — 端侧 AI 在车端的部署形态
- [`AUTOSAR-Adaptive-vs-Classic-对比详解.md`](./AUTOSAR-Adaptive-vs-Classic-对比详解.md) — 域控选择
- [`车企C-C++使用情况详解.md`](./车企C-C++使用情况详解.md) — 端侧推理引擎的语言选择（C++ 主流、Rust 新兴）
- [`Rust-语言详解-车载与系统软件视角.md`](./Rust-语言详解-车载与系统软件视角.md) — Rust 在端侧推理的尝试
- [`Rust-内存安全机制详解.md`](./Rust-内存安全机制详解.md) — Rust 推理引擎的安全保证
- [`通信中间件DDS-SOMEIP-gRPC详解.md`](./通信中间件DDS-SOMEIP-gRPC详解.md) — 域间 AI 推理结果传输
- [`C++标准演进详解-C++98到C++26.md`](./C++标准演进详解-C++98到C++26.md) — C++17/20 在推理引擎中的实战