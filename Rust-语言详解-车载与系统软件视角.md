# Rust 语言详解 —— 面向车载 / 嵌入式 / 系统软件工程师

> 面向人群：已掌握 C/C++，正在评估是否在 **车载 ECU、智驾域控、SDV 中间件、Linux 内核驱动、系统级工具链** 中引入 Rust 的工程师
> 目标：把 Rust 的语言特性、生态成熟度、安全关键认证、车载落地情况、与 C++ 的差异讲清楚，给出"什么时候用 Rust、什么时候还应该用 C++"的可操作判断
> 读者收益：能在面试、架构评审、技术选型 PPT 中讲清 "Rust 凭什么、到哪一步、还差什么"

---

## 写在前面：为什么 2026 年还要专门写一篇 Rust？

你可能已经看过 [`车企C-C++使用情况详解.md`](./车企C-C++使用情况详解.md)，里面给出一个冷峻的结论：

> "在可预见的 10 年内，C 与 C++ 仍是车载软件的'主语言'，Rust 只是在**新增模块**层面缓慢渗透。"

这句话在 2026 年依然成立——但 **"缓慢渗透"的速度曲线** 正在发生重要变化：

```
2015–2019：Mozilla 主导期，Rust 仅在 Firefox / Servo 内自用
2020–2022：Discord、Cloudflare、AWS、Facebook 工程团队公开 Rust 战绩
2023–2024：Linux 内核 6.1 引入 Rust 基础设施，Android 团队大规模迁移
2025      ：Rust 2024 Edition 稳定，Ferrocene 被 AdaCore 收购
2026      ：CISA "安全设计" 警报、安全关键 Rust 联盟扩张、白宫 ONCD 报告落地
未来 5 年 ：从 "试点" 走向 "Adaptive AUTOSAR 边缘组件、OTA、加密、IPC 中间件" 的稳态渗透
```

**核心矛盾**：C/C++ 给我们 30 年的工具链、MISRA 编码规范、ISO 26262 流程认证；但同时也带给我们 **70% 的安全漏洞**（Microsoft、Chrome、Android 三大统计都指向这个数字区间），而 Rust 在系统级编程领域**首次把"内存安全"做成语言级默认**。

本篇文章不鼓吹"用 Rust 重写一切"，而是把 Rust 作为 **C/C++ 旁边的另一种工程选项**，系统地讲清楚它能做什么、不能做什么、到 2026 年为止哪些工具链/认证已经够用。

---

## 目录

1. [全景：Rust 是什么？](#1-全景rust-是什么)
2. [历史与版本演进](#2-历史与版本演进)
3. [核心语言特性](#3-核心语言特性)
4. [异步、并发与运行时](#4-异步并发与运行时)
5. [Unsafe、FFI 与 C/C++ 互操作](#5-unsafeffi-与-cc-互操作)
6. [宏系统与编译期编程](#6-宏系统与编译期编程)
7. [嵌入式与 RTOS 生态](#7-嵌入式与-rtos-生态)
8. [安全关键认证：Ferrocene 与 Safety-Critical Rust 联盟](#8-安全关键认证ferrocene-与-safety-critical-rust-联盟)
9. [AUTOSAR 与车规生态](#9-autosar-与车规生态)
10. [Linux 内核、Android 与互联网基础设施](#10-linux-内核android-与互联网基础设施)
11. [政府/法规驱动的内存安全浪潮](#11-政府法规驱动的内存安全浪潮)
12. [Rust vs C++：一份冷静的对比](#12-rust-vs-c一份冷静的对比)
13. [批评、局限与未解决问题](#13-批评局限与未解决问题)
14. [什么时候该用 Rust、什么时候不要](#14-什么时候该用-rust什么时候不要)
15. [学习路径与速查表](#15-学习路径与速查表)

---

## 1. 全景：Rust 是什么？

**Rust** 是一门由 Mozilla 员工 Graydon Hoare 个人项目起步、2010 年首次公开、2015 年发布 1.0 稳定版、由 Rust 基金会（2015 年成立，2021 年独立为非营利组织）治理的系统级编程语言。它的设计目标可以一句话概括：

> **零成本抽象 + 内存安全 + 线程安全 + 实用主义**

| 维度 | Rust 的位置 |
|---|---|
| **抽象层级** | 与 C++ 相当（零成本抽象），高于 Go/Rust 早期 |
| **执行模型** | 编译为原生机器码，**无 GC、无运行时**（除 `alloc` / `std` 可选组件） |
| **内存安全** | **编译期保证**（借用检查器 + 类型系统），非运行时检查 |
| **线程安全** | 通过 `Send` / `Sync` trait 在编译期阻止数据竞争 |
| **生态策略** | **Cargo** + **crates.io** 集中式包管理（学习曲线陡，但一致性高） |
| **工具链** | rustc + cargo + clippy + rustfmt + rust-analyzer 是官方"全家桶" |
| **社区治理** | Rust 基金会 + 核心团队 + 工作组模式（RFC 流程透明） |

Rust 不是"另一种 C++"——它在**内存管理哲学**上做了根本性切换：

```
C/C++  ：程序员手动管内存 → 编译期只检查语法，运行时崩溃 / 漏洞
Java/Go：垃圾回收器自动管   → 编译期宽松，但 GC 停顿无法用于硬实时
Rust   ：所有权系统在编译期证明内存安全 → 零运行时开销、零 GC、零数据竞争
```

这就是为什么 Rust 同时吸引 **系统程序员**（"我想要 C 的速度但不要 C 的坑"）和 **应用程序员**（"我想要 Java 的安全但不要 GC"）。

---

## 2. 历史与版本演进

### 2.1 关键时间线

| 年份 | 事件 |
|---|---|
| 2006 | Graydon Hoare 开始个人项目 |
| 2009 | Mozilla 接手赞助 |
| 2010 | 首次公开（0.1 发布） |
| 2015-05-15 | **Rust 1.0 稳定版发布** |
| 2015-12 | **Rust 2015 Edition**（首个 Edition） |
| 2018-12 | **Rust 2018 Edition**（引入 `?` 操作符、NLL 等） |
| 2021-10 | **Rust 2021 Edition**（引入闭包捕获改进、`IntoIterator` for arrays 等） |
| 2024-11-12 | **Rust 2024 Edition** 公告 |
| 2025-02-20 | **Rust 1.85.0**：2024 Edition 稳定 + Async Closures |
| 2025-04-03 | Rust 1.86.0：Trait Upcasting + `extract_if` |
| 2025-06-26 | Rust 1.88.0：`let`-chains |
| 2026-08（当前） | 最新稳定版 1.8x 系列，6 周一个版本节奏不变 |

### 2.2 Edition 机制

Rust 的 **Edition** 是其独创的"语法版本号"机制：

```rust
// Cargo.toml
[package]
edition = "2024"   // 可选：2015 / 2018 / 2021 / 2024
```

* Edition 不是 **语言版本**——同一份源码可以用不同 edition 编译
* 同一 crate 可以混合不同 edition 的依赖
* Edition 的作用是引入**破坏性语法变化**而不影响向后兼容（旧的 crate 继续按老 edition 编译）
* 2024 Edition 主要变化：`unsafe extern` 块、`gen` 块（夜间）、`unsafe` 属性宏、`trait` 中的 RPITIT 改进等

**为什么这很重要？** 与 C++ 不同（每 3 年一个新标准，导致 ABI 不兼容），Rust 可以在不破坏生态的前提下稳步演进——这是治理层面的创新。

### 2.3 最近稳定的关键特性（2025–2026）

| 版本 | 日期 | 关键特性 |
|---|---|---|
| **1.85** | 2025-02-20 | 2024 Edition 稳定、**Async Closures**（`AsyncFn`/`AsyncFnMut`/`AsyncFnOnce` trait） |
| **1.86** | 2025-04-03 | **Trait Upcasting**（`&dyn Sub` → `&dyn Super`）、`HashMap::extract_if`、`slice::concat_into`、`core::error::Error` v2 |
| **1.87** | 2025-05-15 | 文档改进、库 API 稳定化 |
| **1.88** | 2025-06-26 | **`let`-chains**（`if let Some(x) = a && let Some(y) = b`）、i586 tier-1 host 移除、wasm64 tier-2 |

仍处于 **nightly** 或 **未稳定** 的关键特性（2026 年 8 月现状）：

* **`gen` blocks（`gen {}`）**：协程/生成器，仍在 nightly `#![feature(gen_blocks)]`
* **Polonius / 下一代借用检查器**：独立仓库开发，**未默认开启**
* **完整 const generics 扩展**（`generic_const_exprs`）：部分场景仍需 nightly

---

## 3. 核心语言特性

### 3.1 所有权（Ownership）

Rust 抛弃了"手动 `malloc/free`"和"GC 自动回收"两条老路，独创**编译期证明**的第三条路：

```rust
fn main() {
    let s = String::from("hello");   // s 拥有堆上的 "hello"
    takes_ownership(s);              // 所有权 move 进函数，s 在此处之后失效
    // println!("{s}");              // ❌ 编译错误：value borrowed here after move

    let x = 5;
    makes_copy(x);                   // i32 是 Copy，原变量仍可用
    println!("{x}");                 // ✅
}

fn takes_ownership(s: String) {      // s 进入作用域
    println!("{s}");
}                                    // s 离开作用域，drop 自动调用

fn makes_copy(n: i32) {              // n 是 Copy
    println!("{n}");
}
```

**核心规则**（背诵这三条，能解 90% 的借用检查错误）：

1. **每个值有且仅有一个所有者**（owner）
2. **所有者离开作用域时，值被自动 `drop`**（RAII）
3. **要么 move、要么 borrow，要么 Copy**——没有"浅拷贝"这个选项

### 3.2 借用与生命周期（Borrowing & Lifetimes）

```rust
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() { x } else { y }
}
```

* `&T` 是不可变借用，`&mut T` 是可变借用
* **同时只能有一个 `&mut T`**，**`&T` 和 `&mut T` 不能同时存在**——这是编译期数据竞争防护的核心
* 生命周期参数 `'a` 让编译器证明 "返回的引用不会比输入活得更久"
* 生命周期省略规则（Lifetime Elision）让 90% 的代码不需要手写 `'a`

**对 C++ 程序员的直觉映射**：

| Rust 概念 | C++ 对应物 | 差异 |
|---|---|---|
| `&T` | `const T&` | Rust 在编译期强制排他 |
| `&mut T` | `T&` | Rust 禁止别名 + 可变同时存在 |
| `'a` | （无显式对应，靠程序员自律） | 编译器静态证明 |
| `Box<T>` | `std::unique_ptr<T>` | Rust 自动 drop，C++ 也要 RAII |
| `Rc<T>` | `std::shared_ptr<T>` | Rust 强制 `Rc::clone()` 显式 |
| `Arc<T>` | `std::shared_ptr<T>`（线程安全版） | 内建线程安全 |

### 3.3 类型系统：代数数据类型 + 模式匹配

```rust
enum VehicleState {
    Parked,
    Driving { speed_kmh: f32, gear: Gear },
    Fault(FaultCode),
    Off,
}

struct Ecu {
    id: u16,
    sw_version: Version,
    state: VehicleState,
}

fn handle(state: &VehicleState) {
    match state {
        VehicleState::Parked => println!("停车"),
        VehicleState::Driving { speed_kmh, .. } if *speed_kmh > 120.0 => {
            println!("超速！{speed_kmh} km/h");
        }
        VehicleState::Driving { speed_kmh, gear } => {
            println!("行驶中 {speed_kmh} km/h, gear={gear:?}");
        }
        VehicleState::Fault(code) => eprintln!("故障 {code:?}"),
        VehicleState::Off => {}
    }
    // 编译器保证：穷尽所有分支，否则警告
}
```

* `enum` 是真正的 **sum type**（C++ 的 `enum class` 是 product type 的退化）
* `match` 必须**穷尽所有分支**——编译器保证无遗漏
* **`if let`、`while let`、`let .. else`** 让模式匹配无处不在
* **`?` 操作符**把错误传播写成线性流

### 3.4 Trait 系统（Rust 的"接口"）

```rust
trait CanBus {
    fn send(&mut self, frame: &Frame) -> Result<(), BusError>;
    fn recv(&mut self) -> Result<Frame, BusError>;
}

struct VectorCanFd { /* ... */ }

impl CanBus for VectorCanFd {
    fn send(&mut self, frame: &Frame) -> Result<(), BusError> { /* ... */ }
    fn recv(&mut self) -> Result<Frame, BusError> { /* ... */ }
}

// 静态分派（零成本）
fn transmit<T: CanBus>(bus: &mut T, frame: &Frame) -> Result<(), BusError> {
    bus.send(frame)
}

// 动态分派（vtable，运行时多态）
fn transmit_dyn(bus: &mut dyn CanBus, frame: &Frame) -> Result<(), BusError> {
    bus.send(frame)
}
```

* **没有继承**——Rust 用 **trait + 默认实现 + 组合** 替代 C++ 的 OOP 继承
* **孤儿规则**（Orphan Rule）：不能为外部类型实现外部 trait——防止生态碎片化
* **关联类型**（`type Item`）让 trait 表达"集合"这种抽象
* **Trait Object**（`dyn Trait`）提供运行时多态，但有虚表开销

### 3.5 泛型与零成本抽象

```rust
fn largest<T: PartialOrd>(list: &[T]) -> &T {
    let mut largest = &list[0];
    for item in &list[1..] {
        if item > largest {
            largest = item;
        }
    }
    largest
}

// 单态化（monomorphization）：每个 T 实例化一份机器码
let n = largest(&[1, 5, 3, 2]);     // largest::<i32>
let s = largest(&["a", "bb", "c"]); // largest::<&str>
```

* 与 C++ 模板相同，泛型在编译期**单态化**，没有运行时开销
* 与 C++ 不同，**Rust 泛型有显式 trait 约束**，编译错误信息友好

### 3.6 错误处理：`Result<T, E>` 而非异常

```rust
use std::fs::File;
use std::io::{self, Read};

fn read_username(path: &str) -> Result<String, io::Error> {
    let mut file = File::open(path)?;          // ? 自动传播 Err
    let mut name = String::new();
    file.read_to_string(&mut name)?;
    Ok(name)
}

fn main() {
    match read_username("config.txt") {
        Ok(name) => println!("user = {name}"),
        Err(e) => eprintln!("读取失败: {e}"),
    }
}
```

* 没有异常，**没有栈展开的运行时开销**（`panic = abort` 时连 unwind 表都不生成）
* `Result<T, E>` 强制调用者处理错误——编译期保证
* `?` 操作符让错误传播如丝般顺滑
* `panic!` 仍然存在，但只用于"不可恢复错误"——社区惯例是库代码返回 `Result`

### 3.7 智能指针与堆分配

| 类型 | 用途 | 类比 |
|---|---|---|
| `Box<T>` | 堆上单所有权 | `unique_ptr<T>` |
| `Rc<T>` | 单线程共享所有权 | `shared_ptr<T>` |
| `Arc<T>` | 多线程共享所有权 | `shared_ptr<T>` + atomic refcount |
| `Cow<'a, T>` | 写时克隆 | 手动 `if_owned_else_borrowed` |
| `Cell<T>` / `RefCell<T>` | 内部可变性 | C++ `mutable` |
| `Mutex<T>` / `RwLock<T>` | 线程同步 | `std::mutex` / `std::shared_mutex` |

### 3.8 模块系统

```rust
// lib.rs
mod vehicle;          // 声明子模块
pub use vehicle::ecu::Ecu;   // 重导出

// vehicle/mod.rs 或 vehicle.rs
pub mod ecu;
pub mod can;

// vehicle/ecu.rs
pub struct Ecu { /* ... */ }
impl Ecu { /* ... */ }
```

* 模块路径**显式**（`use crate::vehicle::ecu::Ecu`）——没有 C++ 的 `#include` 隐式魔法
* **默认私有**，必须 `pub` 才可见——可见性粒度比 C++ 更细

---

## 4. 异步、并发与运行时

### 4.1 async/await：编译期状态机

```rust
async fn fetch_vehicle_telemetry(id: u32) -> Result<Telemetry, NetError> {
    let req = Request::get(format!("/api/v1/vehicles/{id}/telemetry"));
    let resp = req.send().await?;
    let body = resp.bytes().await?;
    Ok(serde_json::from_slice(&body)?)
}
```

* `async fn` 返回实现 `Future` 的状态机，**无栈协程**
* `.await` 让出执行权，**不阻塞 OS 线程**
* 编译期生成状态机，**零运行时调度器**（除非你选用 Tokio 等运行时）

### 4.2 异步运行时分裂（2026 年仍是 Rust 最大的"内部矛盾"）

| 运行时 | 适用场景 | 代表用户 |
|---|---|---|
| **Tokio** | 服务器、网络、高并发 I/O | AWS、Discord、Cloudflare |
| **async-std** | 通用异步（已萎缩） | 历史项目 |
| **smol** | 轻量级、可嵌入 | 小型服务 |
| **embassy** | 嵌入式裸跑 | 嵌入式 / RTOS 替代 |
| **glommio** | 线程绑定的核间共享 | 高性能存储 |
| **monoio**（字节跳动） | io_uring 优先 | 字节跳动生产环境 |

* 2025–2026 趋势：**Tokio 主导服务器侧**、**embassy 主导嵌入式侧**
* 标准库至今**不提供** async runtime——这是有意的设计，但也带来生态碎片化

### 4.3 无锁并发：`Send` / `Sync`

```rust
auto trait Send {}     // 可以跨线程 move
auto trait Sync {}     // 可以跨线程共享 &T

// 编译器自动推导；用 !Send / !Sync 显式禁止
struct MyHandle { /* ... */ }
impl !Send for MyHandle {}  // 禁止跨线程（持有线程局部资源）
```

* 数据竞争（data race）**编译期阻止**——不需要 `-fsanitize=thread`
* 这比 `Arc<Mutex<T>>` 更优雅，比 `unsafe` 更安全

### 4.4 线程池与并行

```rust
use rayon::prelude::*;

let results: Vec<i32> = (0..1_000_000)
    .into_par_iter()                      // 自动并行
    .map(|x| expensive_compute(x))
    .collect();
```

* `rayon` 提供工作窃取线程池，**API 与标准迭代器无缝集成**
* 比手写 `std::thread::spawn` 简单一个数量级

---

## 5. Unsafe、FFI 与 C/C++ 互操作

### 5.1 Unsafe 的边界

```rust
fn get_unchecked(arr: &[i32], idx: usize) -> i32 {
    unsafe {
        *arr.get_unchecked(idx)    // 跳过边界检查，调用者保证 idx < arr.len()
    }
}
```

* **`unsafe` 不关闭借用检查器**——只解锁 5 个能力：
  1. 解引用裸指针 `*const T` / `*mut T`
  2. 调用 `unsafe fn` / `unsafe trait`
  3. 访问/修改 `unsafe` 静态变量
  4. 实现 `unsafe trait`
  5. 访问 `union` 字段
* 良好的实践：**unsafe 块尽量小，外面包一层 safe 抽象**

```rust
// 推荐写法：safe API + 内部 unsafe
pub struct Vec<T> { /* ... */ }
impl<T> Vec<T> {
    pub fn get(&self, idx: usize) -> Option<&T> {
        if idx < self.len() {
            // SAFETY: idx < len 已由 if 保证
            Some(unsafe { self.get_unchecked(idx) })
        } else {
            None
        }
    }
}
```

### 5.2 FFI：调用 C 库

```rust
extern "C" {
    fn abs(input: i32) -> i32;
}

#[no_mangle]
pub extern "C" fn rust_callback(x: i32) -> i32 {
    x.abs()
}
```

* `extern "C"` 块声明外部 C 函数
* `#[no_mangle]` + `extern "C"` 导出 Rust 函数给 C 调用
* **`bindgen`** 自动从 C 头文件生成绑定（`cargo install bindgen`）

### 5.3 C++ 互操作（2026 年仍在演进）

| 工具 | 状态 | 适用场景 |
|---|---|---|
| **cxx** | **生产可用** | 安全 FFI 桥（Rust ↔ C++） |
| **autocxx** | 实验性 | 自动从 C++ 头文件生成绑定 |
| **crubit**（Google） | 实验性（2026） | 自动 C++ ↔ Rust 互操作 |
| **diplomat** | 实验性 | 多语言绑定生成（Rust → C/JS/WASM） |
| **c++ 子集手写** | **最稳** | 关键路径仍推荐手写 FFI |

**实战策略**：

```
整个车控系统 = C++ AUTOSAR AP + Rust 安全模块 + C 驱动

         ┌────────────────────────────────�
         │   Adaptive AUTOSAR (C++ ARA)   │
         └─────────────┬──────────────────┘
                       │ cxx FFI 边界
         ┌─────────────▼──────────────────┐
         │   Rust 安全模块                │
         │   - 加密 / TLS                 │
         │   - 序列化 (Protobuf)          │
         │   - 状态机                     │
         │   - 解析器                     │
         └─────────────┬──────────────────┘
                       │ extern "C" FFI
         ┌─────────────▼──────────────────┐
         │   C 驱动 / MCAL / 寄存器       │
         └────────────────────────────────┘
```

---

## 6. 宏系统与编译期编程

### 6.1 声明宏（`macro_rules!`）

```rust
macro_rules! vehicle_log {
    ($level:ident, $($arg:tt)*) => {
        match $level {
            LogLevel::Info  => println!("[INFO]  {}", format!($($arg)*)),
            LogLevel::Warn  => eprintln!("[WARN]  {}", format!($($arg)*)),
            LogLevel::Error => eprintln!("[ERROR] {}", format!($($arg)*)),
        }
    };
}

vehicle_log!(LogLevel::Info, "ECU {} online", ecu_id);
```

### 6.2 过程宏（Procedural Macros）

```rust
// derive 宏：自动实现 trait
#[derive(Debug, Clone, PartialEq, Eq, Hash)]
struct CanFrame { id: u32, data: Vec<u8> }

// 属性宏：修改代码
#[inline]
fn hot_path() { /* ... */ }

#[test]
fn test_can_frame() { /* ... */ }

// 函数式宏：自定义 DSL
let sql = sql_select!(users => id, name WHERE age > 18);
```

* 过程宏在编译期运行，可以 **改写 AST**——这是 Rust 实现 ORM、序列化框架的核心
* `serde`、`tokio`、`clap` 等明星库都重度依赖过程宏

### 6.3 const fn 与编译期计算

```rust
const fn factorial(n: u32) -> u32 {
    if n <= 1 { 1 } else { n * factorial(n - 1) }
}

const FACT_10: u32 = factorial(10);  // 编译期计算，无运行时开销
```

* `const fn` 在编译期求值，可用于数组长度、全局配置等
* 仍在持续扩展（const generics、const 浮点、const mut）

---

## 7. 嵌入式与 RTOS 生态

### 7.1 三层架构

```
┌─────────────────────────────────────────────────────────────┐
│  应用层   embassy-task / RTIC task                          │
├─────────────────────────────────────────────────────────────┤
│  框架层   embassy-executor / RTIC scheduler                 │
├─────────────────────────────────────────────────────────────┤
│  HAL 层   embedded-hal 1.0 trait（统一接口）                │
├─────────────────────────────────────────────────────────────┤
│  PAC 层   svd2rust 生成的寄存器定义                          │
├─────────────────────────────────────────────────────────────┤
│  硬件     Cortex-M / RISC-V / AVR / ESP32                   │
└─────────────────────────────────────────────────────────────┘
```

### 7.2 关键嵌入式 crate

| crate | 作用 | 状态 |
|---|---|---|
| **embedded-hal** | 统一硬件抽象 trait | 1.0 稳定 |
| **embassy** | 异步嵌入式框架 | 活跃 |
| **RTIC** | 实时中断驱动并发 | v2.x 活跃 |
| **probe-rs** | 烧录/调试 | 主流替代 OpenOCD |
| **defmt** | 高效日志框架 | RTT 传输 |
| **cargo-embed** | 一键烧录 | probe-rs 官方 |

### 7.3 支持的 MCU 矩阵（2026 年）

| 厂商 | MCU | Rust 支持 |
|---|---|---|
| **ST** | STM32F/G/L/H/U/WB/WL | embassy / stm32-hal 完善 |
| **NXP** | i.MX RT、LPC | 社区 + NXP 官方样例 |
| **Infineon** | AURIX TC3xx | 官方 [github.com/infineon/aurix-rs](https://github.com/infineon/aurix-rs) + HighTec 编译器 |
| **Renesas** | RA / RX | 社区 |
| **Renesas** | RH850 | 社区 [rh850-rust](https://github.com/rh850/rh850-rust) |
| **Espressif** | ESP32 | esp-rs 官方支持 |
| **Nordic** | nRF52/53/91 | embassy-nrf 完善 |
| **Raspberry Pi** | RP2040 | embassy-rp 完善 |
| **Microchip** | PIC32 / SAM | 社区 |

---

## 8. 安全关键认证：Ferrocene 与 Safety-Critical Rust 联盟

这是 Rust 在车载领域**最关键**的章节——没有合规认证，就上不了量产 ECU。

### 8.1 Ferrocene（Ferrous Systems → AdaCore）

| 项 | 状态（2026-08） |
|---|---|
| 提供方 | Ferrous Systems（2024 年被 **AdaCore 收购**） |
| 网址 | https://ferrocene.dev |
| 认证等级 | **ISO 26262 ASIL B**（ASIL D 暂无公开认证）、IEC 61508 SIL 2、IEC 62304 |
| 开源 | 工具链本身开源 [github.com/ferrocene/ferrocene](https://github.com/ferrocene/ferrocene) |
| 商业支持 | AdaCore 销售（与 GNAT Pro Ada 同一家） |
| 目标硬件 | ARM Cortex-M/R/A、RISC-V、x86_64（覆盖范围持续扩展） |

**对比 MISRA-C / C++ 工具链**：

```
                    认证历史    工具链成熟度   文档完整度   客户基数
IAR / GHS / Tasking   30+ 年     ★★★★★         ★★★★★        ★★★★★
Ferrocene            2-3 年     ★★★★          ★★★★         ★★
```

**关键认知**：Ferrocene 现在是 AdaCore 的产品——这意味着它**与汽车行业 30 年来使用 GNAT Pro Ada 的客户基础绑定**，这是 Rust 在车规生态中最重要的"靠山"。

### 8.2 Safety-Critical Rust 联盟（Rust 基金会下属）

**成立**：2024 年 6 月  
**初始成员**：AdaCore、Arm、Ferrous Systems、OxidOS、HighTec EDV-Systeme、TrustInSoft、Veecle、**Woven by Toyota**

**2025 年新增**：CARIAD（VW 集团）、Bosch、LG Electronics、Nordic Semiconductor、NXP Semiconductors、KDAB、Raptor Scientific

**核心工作**：

1. **编码指南**：`rustfoundation/safety-critical-rust-coding-guidelines`（GitHub 公开）
   * 映射 CERT C 规则到 Rust 类别
   * 截至 2026 年仍在积极更新（Issue #427 等）
   * **不是 MISRA 发布的"MISRA Rust"**——但已经是事实上的"车规 Rust 编码指南"

2. **形式化验证**：与 AWS 等合作探索 Rust 标准库的形式化规约（Ferrocene 也参与）

3. **培训与认证**：推动 Rust 安全关键工程师认证体系

### 8.3 与 MISRA-C / C++ 的"对标关系"

| 维度 | MISRA-C / C++ | Safety-Critical Rust 联盟 |
|---|---|---|
| **组织** | MISRA（汽车行业软件可靠性协会） | Rust Foundation 下属工作组 |
| **历史** | 1998 年起（C），2008 年起（C++） | 2024 年起 |
| **成熟度** | 工业标准，OEM 强约束 | 草案阶段，OEM 谨慎采纳 |
| **强制力** | 合同级要求 | 内部指南，逐年成熟 |
| **覆盖范围** | C / C++ 全语言子集 | Rust safe + 限定 unsafe 子集 |

**给工程师的实操建议**：如果你 2026 年要给客户提案 "用 Rust 做 ASIL B 模块"，可以说：

> "我们将基于 **Ferrocene 工具链**（ASIL B 已认证）+ **Safety-Critical Rust 联盟编码指南**（覆盖 ~80% MISRA-C 等价规则），结合项目级补充规则，达成 ASIL B 目标。ASIL D 模块仍以 C/C++ + MISRA + ISO 26262 流程为主。"

---

## 9. AUTOSAR 与车规生态

### 9.1 AUTOSAR Rust 工作组

* **存在**：AUTOSAR 已成立 Rust 工作组（前身 Classic-Rust WG），探索 Rust 在 AP/CP 中的角色
* **状态（2026-08）**：**提案/讨论阶段**，有早期实现，但**Adaptive AP 官方 Rust API 尚未正式发布**
* **方向**：在 ARA::com / ARA::exec 等核心接口提供 Rust 绑定（类似 C++ 的 `ara::com::Skeleton`）

### 9.2 主要 Tier-1 / OEM 的 Rust 落地（2026 年 8 月）

| 厂商 | 落地场景 | 来源 |
|---|---|---|
| **Tesla** | Autopilot 部分组件（2022 起公开转型） | Tesla 官方博客 |
| **Volvo / Zenseact** | EX90 自动驾驶栈（Zenseact 提供） | 公开访谈 |
| **Woven by Toyota** | Arene 平台试验、联盟成员 | Woven 技术博客 |
| **Bosch** | 联盟成员、内部试点 | 公告 |
| **CARIAD（VW）** | 联盟成员（2025 加入） | 公告 |
| **Renesas** | RH850 社区 Rust 移植 | GitHub |
| **Infineon** | AURIX TC3xx 官方 Rust 仓库 | GitHub |
| **HighTec EDV** | AURIX Rust 商业编译器 | 商业产品 |

### 9.3 车载 Rust 的典型应用层

```
┌──────────────────────────────────────────┐
│   应用层（OEM 业务逻辑）                  │
│   - Rust 写：状态机、解析器、加密         │
│   - C++ 写：Adaptive AP ARA 接口         │
├──────────────────────────────────────────┤
│   中间件层                                │
│   - SOME/IP、DDS、gRPC（混合 C++/Rust）  │
│   - Protobuf 序列化（Rust 生态强）        │
├──────────────────────────────────────────┤
│   驱动 / OS 层                           │
│   - C 驱动 + Rust 内核模块（Linux 6.x）  │
│   - QNX / VxWorks Rust 集成（探索中）    │
└──────────────────────────────────────────�
```

### 9.4 一个具体的 AUTOSAR AP + Rust 混合栈示例

```rust
// 业务逻辑层（Rust，safe）
mod business_logic {
    pub fn validate_ota_image(image: &[u8]) -> Result<ImageMeta, OtaError> {
        // 签名验证、版本检查、回滚条件评估
        // 全部 safe Rust，编译期保证无缓冲区溢出
        // ...
    }
}

// FFI 桥（Rust → C++ AUTOSAR ARA）
#[cxx::bridge]
mod ffi {
    unsafe extern "C++" {
        include!("ara/com/skeleton.h");
        type SomeIpSkeleton = ara::com::Skeleton;
        fn send_event(self: &SomeIpSkeleton, payload: &CxxVector<u8>);
    }
    extern "Rust" {
        fn validate_ota_image(image: &[u8]) -> Result<RustString>;
    }
}

// C++ AUTOSAR AP 侧（保持原有架构）
// 调用 Rust 业务逻辑做 OTA 验证，然后通过 SOME/IP 通知其他 ECU
```

**结论**：在 2026 年，**Adaptive AP 仍然是 C++**——但 **Rust 作为"业务逻辑层 + 安全模块"** 已经可以嵌入到 C++ AP 的边界内。这与 20 年前 "C 业务 + C++ 框架" 的混合模式同构。

---

## 10. Linux 内核、Android 与互联网基础设施

Rust 在车规之外已经大规模落地，这是判断它"会不会继续存在"的最强信号。

### 10.1 Linux 内核

| 里程碑 | 时间 |
|---|---|
| Linux 6.1 引入 Rust 基础设施 | 2022-12 |
| Linux 6.14：Rust drivers 支持完善 | 2025-Q1 |
| Kernel Maintainers Summit：Rust "不再是实验性" | 2025 末 |
| Linus Torvalds 公开接受 Rust 代码 | 2025 末 |

**已落地的 Rust 内核子系统/驱动**：

* Asahi Linux Apple GPU 驱动（已生产可用）
* NVMe 驱动（进行中）
* 9p / 网络抽象层
* 平台设备 / PCI 子系统绑定

### 10.2 Google Android

| 指标 | 2022 | 2024 | 2025–2026 |
|---|---|---|---|
| Android 内存安全漏洞占比 | ~20% | <5% | 持续下降 |
| 新增原生代码中 Rust 比例 | 0% | ~30%（Android 15） | **>50%** |

* **Google Security Blog（2025-05）**：Android 平台内存安全漏洞从 2022 年的 ~20% 降至 2024 年的 <5%
* **2025 起**：Android 新增原生代码中 Rust 占比已超 **50%**

### 10.3 其他互联网基础设施

| 公司 | 项目 | 规模 |
|---|---|---|
| **AWS** | Firecracker（Lambda/Fargate 微 VMM）、Bottlerocket（容器 OS） | 关键基础设施 |
| **Cloudflare** | Pingora（HTTP 代理）、Workers 边缘 | 全球边缘网络 |
| **Discord** | 服务端（从 Go 迁移） | 亿级用户 |
| **Dropbox** | 文件同步引擎（2015 起） | PB 级存储 |
| **Figma** | 渲染引擎 → WebAssembly | 设计工具头部 |
| **Microsoft** | Windows 内核组件、Azure 服务 | 关键基础设施 |
| **Meta** | Monorepo 工具链（buck2/buck）、广告服务 | 部分生产 |
| **Cloudflare** | R2 存储、Workers Runtime | 全球生产 |

### 10.4 操作系统 / 工具链替代

| 项目 | 状态（2026） |
|---|---|
| **uutils/coreutils**（GNU coreutils Rust 重写） | **未**成为 Ubuntu/Debian 默认；Alpine/Fedora 可选 |
| **Redox OS** | 研究型 Rust 操作系统 |
| **SerenityOS** | 桌面 OS，部分 Rust |
| **Deno** | JS/TS 运行时，Rust 实现（GitHub ~100k+ star） |
| **Rustls** | TLS 库，已被 curl、BoringSSL 替代评估 |

---

## 11. 政府/法规驱动的内存安全浪潮

这是 2024–2026 年 **Rust 在 ToB 市场最大的增长引擎**——不是因为"程序员喜欢"，而是因为"政府要求"。

### 11.1 关键政策时间线

| 时间 | 文件 | 关键内容 |
|---|---|---|
| **2023-12** | NSA / CISA / FBI 联合指南 | 呼吁采用内存安全语言 |
| **2024-02-26** | **白宫 ONCD 报告**："Back to the Building Blocks" | 明确推荐 Rust/Java/Go/Swift 等替代 C/C++ |
| **2024-06** | CISA + 5 国盟友联合更新 | 国际版本内存安全指南 |
| **2025-02-11** | CISA "消除缓冲区溢出" 安全设计警报 | 把内存安全漏洞列为"产品坏实践" |
| **2024-10-10** | EU CRA 通过 | 强制漏洞处理、SBOM、安全设计 |
| **2026-09-11** | EU CRA 漏洞报告生效 | ENISA 强制 SBOM + 漏洞报告 |
| **2027-12-11** | EU CRA 全面适用 | 所有联网产品强制合规 |

### 11.2 OEM / 企业的内部"内存安全路线图"

* **Microsoft**：2024 年起要求 **新代码默认 Rust / C#**，C/C++ 新项目需特批
* **Google**：Android 团队 2026 年 Rust 占比 >50%
* **Amazon**（AWS）：Rust 是 Lambda / 关键基础设施首选
* **Apple**：Swift + 内部 Swift-Rust 互操作
* **Linux 基金会 / CNCF**：多个项目（linkerd2-proxy、TikV 等）从 Go 迁向 Rust 以减少 GC 抖动

**对车规的传导路径**：

```
2024：白宫 ONCD → 美国 OEM 内部压力
2025：CISA 警报   → OEM 安全团队内部要求
2026：EU CRA 报告 → 出口欧洲的车载产品受影响
2027：EU CRA 全效 → 强制的"安全设计"间接要求内存安全
        ↓
OEM 反向施压 Tier-1：在新增模块、OTA、安全栈上引入 Rust
        ↓
Tier-1 施压编译器供应商：要求提供车规级 Rust 工具链
        ↓
这就是 Ferrocene、HighTec Rust、AdaCore GNAT Pro for Rust 2024–2026 集体出现的原因
```

---

## 12. Rust vs C++：一份冷静的对比

| 维度 | C++ (C++17/20) | Rust (2024 edition) |
|---|---|---|
| **内存安全** | 程序员自律 + 工具链检测（ASan/MSan/Coverity） | **编译期强制**（借用检查器） |
| **线程安全** | 程序员自律 + TSan | 编译期（Send/Sync trait） |
| **零成本抽象** | ✅ 模板 / inline | ✅ 泛型单态化 / trait 静态分派 |
| **学习曲线** | 陡（10+ 年掌握） | **更陡**（所有权 + 生命周期是全新心智模型） |
| **编译时间** | 中等（模板实例化慢） | **慢**（cargo 增量编译已大幅改进，但首次仍慢） |
| **二进制大小** | 小（手工优化空间大） | 较大（默认含 unwind 表，可关闭） |
| **运行时** | 无（贴近硬件） | 无（除 `alloc`/`std`） |
| **元编程** | 模板 + SFINAE + concepts | 过程宏 + const fn + trait |
| **错误处理** | 异常 / error code（分裂） | `Result<T,E>` + `?`（统一） |
| **生态治理** | 委员会（ISO C++ WG21） | 基金会 + 团队（RFC 流程） |
| **ABI 稳定性** | Itanium ABI（稳定但复杂） | **每 crate 不稳定**（需 `cdylib` + C ABI 互通） |
| **车规工具链** | IAR / GHS / Tasking / Wind River（30+ 年） | Ferrocene / HighTec（2-3 年） |
| **车规编码规范** | MISRA C/C++、AUTOSAR C++14 | Safety-Critical Rust 联盟指南（草案） |
| **ASIL D 认证** | ✅ 多家编译器有完整认证 | ⚠️ Ferrocene 当前 ASIL B（ASIL D 暂无公开认证） |
| **现役代码基数** | 数十亿行（汽车行业） | < 1 亿行（含车企内部） |
| **工程师池** | 充裕 | 稀缺（车规方向尤其） |
| **社区评价（2025 SO 调查）** | 第 8 名受喜爱语言 | **第 1 名受喜爱语言（连续 9 年）** |

### 一个心智模型

> **C++** 是 "给你一把瑞士军刀，所有刀刃都开着"——能力大、责任重  
> **Rust** 是 "同把瑞士军刀，但有安全锁"——能力相当、责任更可控

---

## 13. 批评、局限与未解决问题

不写缺点就不是好文档——以下是 Rust **当下确实存在**的问题：

### 13.1 编译时间

* 第一次完整构建大型项目（如 `rustc` 自身、Chromium 部分）需要 10-30 分钟
* 增量编译已大幅优化（2025–2026 持续改进）
* 缓解工具：**sccache**（远程编译缓存）、**mold / lld 链接器**、**miri**（UB 检测）

### 13.2 异步生态碎片化

* Tokio / async-std / smol / embassy / glommio / monoio 多家并存
* 标准库至今不提供官方运行时
* 2025–2026 趋势是 **Tokio 主导服务器、embassy 主导嵌入式**，但仍是社区痛点

### 13.3 二进制大小

```toml
# Cargo.toml 优化二进制大小
[profile.release]
opt-level = "z"      # 体积优化
lto = true           # 链接时优化
panic = "abort"      # 去掉 unwind 表
strip = true         # 去掉符号
codegen-units = 1    # 减少并行编译但更好优化
```

* 默认配置下，hello world 静态二进制约 **200KB**（带 std）
* 经过优化可降到 **10–50KB**（裸 `no_std`）
* 对 MCU 上的几十 KB 内存仍是挑战

### 13.4 学习曲线

* 所有权 + 借用 + 生命周期是**全新心智模型**——C++ 程序员转 Rust 平均需要 3-6 个月才能写出自然代码
* 高级特性（trait object、`dyn` vs `impl`、HRTB、协变/逆变）需要 1-2 年才能掌握

### 13.5 缺少稳定 ABI

* Rust crate 之间没有标准 ABI——升级 rustc 版本可能导致二进制不兼容
* 对**长期嵌入式部署**是真实问题
* 缓解：`cdylib` + C ABI 是当前事实标准

### 13.6 Unsafe 在生态中的占比

* Rust 标准库本身使用 `unsafe`（不可避免——它要调用 OS）
* 流行 crate 中仍有不少 `unsafe` 块
* **Rustsec Advisory DB** 持续跟踪安全漏洞
* 2026 年趋势：越来越多的库开始提供 `audit` 友好接口（`cargo audit` / `cargo deny`）

### 13.7 车规工程师池子太小

* 全球具备 ASIL B/D Rust 经验的工程师估计 < 5000 人（粗略估算）
* Safety-Critical Rust 联盟的培训计划正在扩张
* 短期内，"Rust + 车规"人才招聘成本可能比 C++ 高 30-50%

### 13.8 AUTOSAR 原生支持缺位

* Adaptive AP ARA 接口仍是 C++ 优先
* Rust 工作组在推进，但 2026 年还没有正式发布的 Rust API
* 短期仍需 FFI 桥接

---

## 14. 什么时候该用 Rust，什么时候不要

### 14.1 ✅ 推荐用 Rust 的场景

| 场景 | 理由 |
|---|---|
| **新启动的 SDV 中间件 / SOA 服务** | 安全 + 性能 + 并发模型全占优 |
| **OTA / 安全模块 / 加密栈** | 内存安全直接降低 CVE 数量 |
| **解析器 / 序列化**（Protobuf、JSON、自定义协议） | 类型系统 + 性能双优 |
| **网络服务 / IPC / gRPC 客户端** | Tokio 生态成熟 |
| **嵌入式驱动 + 业务逻辑混合** | embassy + RTIC 提供硬实时 |
| **Linux 用户态工具链**（替代 C 重写） | uutils/coreutils 模式 |
| **Linux 内核新驱动** | 6.14+ Rust 支持成熟 |
| **WebAssembly 模块** | Figma 模式，Rust → WASM 工具链最佳 |

### 14.2 ❌ 仍推荐用 C/C++ 的场景

| 场景 | 理由 |
|---|---|
| **ASIL D 安全关键路径**（刹车、转向、气囊） | Ferrocene 当前最高 ASIL B，工具链覆盖不足 |
| **既有 C 代码热路径**（10+ 年验证的 ECU BSW） | 重写风险大于收益 |
| **超紧资源约束 MCU**（< 32KB Flash） | Rust 运行时（即使 `core`）仍占空间 |
| **硬实时中断处理**（< 1μs） | Rust 可达但工具链调优经验少 |
| **需要稳定 ABI 的长期库** | Rust ABI 不稳定 |
| **团队完全无 Rust 经验 + 量产压力** | 学习成本不划算 |

### 14.3 混合策略（最现实的 2026 落地方式）

```
┌────────────────────────────────────────────────┐
│  应用层 / 新模块（Rust）                        │
│  - 业务逻辑、状态机、解析、加密、序列化          │
├────────────────────────────────────────────────┤
│  适配层（Rust FFI / cxx）                      │
├────────────────────────────────────────────────┤
│  Adaptive AP / ARA（C++ 既有）                 │
├────────────────────────────────────────────────┤
│  Classic AP / BSW（C 既有）                    │
├────────────────────────────────────────────────┤
│  MCAL / 寄存器层（C / 汇编）                    │
└────────────────────────────────────────────────┘
```

**口诀**：**C/C++ 守底线、Rust 打增量、FFI 当胶水**。

---

## 15. 学习路径与速查表

### 15.1 8 周学习路径（针对 C/C++ 背景）

| 周 | 内容 | 资源 |
|---|---|---|
| **1** | 安装 rustup、Hello World、Cargo、所有权基础 | [The Rust Book](https://doc.rust-lang.org/book/) 前 10 章 |
| **2** | 借用、生命周期、引用 vs 指针 | Rust Book 第 10 章 + [Rustlings](https://github.com/rust-lang/rustlings) |
| **3** | 结构体、Enum、模式匹配、错误处理 | Rust Book 第 6/9 章 |
| **4** | 泛型、Trait、Trait Object | Rust Book 第 10 章 |
| **5** | 模块系统、Crate 生态、cargo | [Cargo Book](https://doc.rust-lang.org/cargo/) |
| **6** | 并发：线程、Mutex、Channel | Rust Book 第 16 章 |
| **7** | 异步：async/await、Tokio | [Tokio Tutorial](https://tokio.rs/tokio/tutorial) |
| **8** | Unsafe、FFI、宏 | [The Rustonomicon](https://doc.rust-lang.org/nomicon/) |

### 15.2 关键速查表

#### 所有权 / 借用
```rust
let s = String::from("hi");   // s 拥有
let t = s;                    // move
let r = &t;                   // 不可变借用
let m = &mut t;               // 可变借用（独占）
```

#### 错误处理
```rust
fn f() -> Result<T, E> { ... }
let x = f()?;                 // 错误传播
match x { Ok(v) => ..., Err(e) => ... }
```

#### 并发
```rust
let h = std::thread::spawn(|| { ... });
h.join().unwrap();

let data = Arc::new(Mutex::new(0));
```

#### 异步
```rust
async fn f() -> Result<T, E> { ... }
let fut = f();
let v = fut.await?;
tokio::spawn(async move { ... });
```

#### Unsafe 边界
```rust
unsafe {
    *ptr;                     // 解引用裸指针
    unsafe_fn();
}
```

### 15.3 关键术语速查表

| 术语 | 含义 |
|---|---|
| **所有权** Ownership | 每个值有且仅有一个所有者 |
| **借用** Borrowing | `&T`（共享）或 `&mut T`（独占） |
| **生命周期** Lifetime | 引用有效的作用域，`'a` |
| **Trait** | Rust 的接口机制 |
| **泛型** Generic | 编译期单态化 |
| **闭包** Closure | 可捕获环境的匿名函数 |
| **宏** Macro | 编译期代码生成（`macro_rules!` / 过程宏） |
| **crate** | Rust 的编译/分发单元（类比 npm 包） |
| **Cargo** | 官方构建系统 + 包管理器 |
| **edition** | 语法版本号（2015/2018/2021/2024） |
| **MSRV** | Minimum Supported Rust Version |
| **`#[derive(...)]`** | 自动派生 trait 实现 |
| **`dyn Trait`** | 运行时多态（trait object） |
| **`impl Trait`** | 静态分派的匿名类型 |
| **`Send` / `Sync`** | 线程安全标记 trait |
| **`'static`** | 生命周期等同于整个程序 |

---

## 写在最后：Rust 是什么"级别"的语言？

回到开篇的问题——**Rust 在车载软件栈中应该被放在什么位置**？

```
                  软件工程语言光谱

安全 ←───────────────────────────→ 控制力
Java / C# / Kotlin     C++ / Rust     C / 汇编

       （应用层）   （系统层）   （硬件层）
```

**Rust 占据了 C++ 的生态位**——它**没有替代 C++，而是为 C++ 提供了一个更安全的同位选项**。在 2026 年的车载软件栈里：

* **新增模块、安全关键逻辑、新中间件**：Rust 是合理选择
* **既有代码、ASIL D 路径、紧资源 MCU**：C/C++ 仍是首选
* **混合架构（Rust + C++ via FFI）**：是 2026 年最现实的落地方式

**一句话总结**：

> **Rust 不是 C++ 的掘墓人，而是 C++ 的安全补丁——它给系统软件世界多了一个"零成本 + 内存安全"的选项。5 年后看 Rust 在车端的渗透率，会跟今天 Go 在云端的渗透率类似：不会替代 C/C++，但会成为每个工程师工具箱里的常驻选项。**

---

## 参考资料

* [Rust 官方](https://www.rust-lang.org/)
* [The Rust Programming Language（官方书）](https://doc.rust-lang.org/book/)
* [Rust 2024 Edition 发布博客](https://blog.rust-lang.org/2024/11/12/Rust-2024-edition.html)
* [Rust 1.85.0 发布博客](https://blog.rust-lang.org/2025/02/20/Rust-1.85.0/)
* [Ferrocene（AdaCore）](https://ferrocene.dev/)
* [Safety-Critical Rust Coding Guidelines](https://github.com/rustfoundation/safety-critical-rust-coding-guidelines)
* [Google Security Blog: Rust in Android](https://security.googleblog.com/2024/09/rust-in-android-move-slowly-and-fix-things.html)
* [Google Security Blog: Memory Safety in Android Platform](https://security.googleblog.com/2025/05/memory-safety-in-android-platform.html)
* [White House ONCD Report（2024-02）](https://www.whitehouse.gov/wp-content/uploads/2024/02/Final-ONCD-Technical-Report.pdf)
* [CISA Product Security Bad Practices](https://www.cisa.gov/news-events/directives/product-security-bad-practices)
* [EU Cyber Resilience Act 概览](https://bootlin.com/blog/cyber-resilience-act-cra-overview/)
* [Stack Overflow Developer Survey 2025](https://survey.stackoverflow.co/2025/technology)
* [Embassy Embedded Framework](https://github.com/embassy-rs/embassy)
* [Infineon AURIX Rust](https://github.com/infineon/aurix-rs)
* [Autosar-learn 目录配套文档](./车企C-C++使用情况详解.md)

---

**版权与维护**：本文档为 autosar-learn 系列学习手册之一，遵循仓库内的贡献方式，欢迎通过 Issue / PR 补充错误、提出建议。
