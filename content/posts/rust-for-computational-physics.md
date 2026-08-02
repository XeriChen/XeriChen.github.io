---
title: "Rust 入门：面向计算物理的新手指南"
date: 2026-08-02T23:30:00+08:00
draft: false
tags: ["学习笔记", "工具链", "计算物理", "Rust"]
description: "面向计算物理场景的 Rust 新手教程：所有权模型、ndarray/nalgebra 库使用、常微分方程求解示例，以及与 C++ 和 Python 的对比。"
---

> **说明：** 本文写给有 Python 或 C++ 基础、想在计算物理任务中尝试 Rust 的研究者。撰写时 Rust 版本为 1.82，主要库：`ndarray 0.16`、`nalgebra 0.33`、`plotters 0.3`。

## 一、为什么计算物理需要 Rust

计算物理的核心需求：性能接近 C++，但希望内存安全、并发可靠、工具链现代。Rust 提供了这三者的组合。

**与 C++ 对比：** Rust 没有空指针（null pointer）、数据竞争（data race）、迭代器失效等常见错误——这些在大型数值程序中调试成本极高。编译器的借用检查器（borrow checker）在编译期排除这类问题，代价是需要适应所有权（ownership）模型。

**与 Python 对比：** Python 的 NumPy/SciPy 适合原型开发，但性能热点（内层循环、复杂逻辑）需要用 C 重写。Rust 用零成本抽象（zero-cost abstraction）直接写出高性能代码，无需"先用 Python 后转 C"的两阶段工作流。

**实际性能参考：** 在矩阵乘法、常微分方程求解等标准任务中，优化后的 Rust 代码通常比 Python 快 10–100 倍，与同等优化的 C++ 代码差距在 5% 以内。

**生态现状：** Rust 的科学计算生态仍在发展中。线性代数、常微分方程、FFT 都有可用库，但覆盖度不及 Python 的 SciPy 或 Fortran 的传统数值库。适合"性能关键但算法不稀奇"的场景。

---

## 二、环境配置

### 2.1 安装 Rust

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

安装后验证：

```bash
rustc --version
cargo --version
```

`rustc` 是编译器，`cargo` 是包管理器和构建工具。

### 2.2 创建项目

```bash
cargo new harmonic_oscillator
cd harmonic_oscillator
```

项目结构：

```
harmonic_oscillator/
├── Cargo.toml    # 依赖清单
└── src/
    └── main.rs   # 入口文件
```

在 `Cargo.toml` 添加依赖：

```toml
[dependencies]
ndarray = "0.16"
plotters = "0.3"
```

---

## 三、Rust 基础语法速览

### 3.1 变量与类型

```rust
fn main() {
    let n: usize = 1000;           // 不可变绑定（默认不可修改）
    let mut sum: f64 = 0.0;        // mut 表示可变
    let dt: f64 = 0.01;
    let x0: f64 = 1.0;

    // 类型推断
    let pi = 3.14159265358979;     // 自动推断为 f64
    let name = "harmonic";         // 字符串切片 &str
}
```

计算物理常用类型：`f64`（双精度浮点，必须使用）、`usize`（索引）、`i64`（大整数计数）。

### 3.2 数组与向量

```rust
use ndarray::Array1;

fn main() {
    // 创建长度为 1000 的零向量
    let mut y: Array1<f64> = Array1::zeros(1000);

    // 等差数列：linspace(0, 10, 1000)
    let t: Array1<f64> = Array1::linspace(0.0, 10.0, 1000);

    // 索引访问
    y[0] = 1.0;
    let val = y[[0]];              // 多维数组用 [[]]

    // 切片（slice）：获取连续子集
    let sub = y.slice(s![0..100]);
}
```

`ndarray` 是 Rust 中最接近 NumPy 的库。`Array1` 是一维数组，`Array2` 是二维。

### 3.3 循环与函数

```rust
// 显式循环（性能与 C 相当）
for i in 0..1000 {
    y[i] = (i as f64 * 0.01).sin();
}

// 迭代器风格（编译器会优化为同样快的代码）
let sum: f64 = y.iter().sum();

// 带索引的迭代
for (i, val) in y.iter().enumerate() {
    println!("y[{}] = {}", i, val);
}
```

函数定义：

```rust
fn harmonic_force(x: f64, omega: f64) -> f64 {
    -omega * omega * x
}

fn rk4_step(
    x: f64,
    v: f64,
    dt: f64,
    omega: f64,
) -> (f64, f64) {
    // 四阶龙格-库塔法的单步
    let k1_x = v;
    let k1_v = harmonic_force(x, omega);

    let k2_x = v + 0.5 * dt * k1_v;
    let k2_v = harmonic_force(x + 0.5 * dt * k1_x, omega);

    let k3_x = v + 0.5 * dt * k2_v;
    let k3_v = harmonic_force(x + 0.5 * dt * k2_x, omega);

    let k4_x = v + dt * k3_v;
    let k4_v = harmonic_force(x + dt * k3_x, omega);

    let x_new = x + dt / 6.0 * (k1_x + 2.0 * k2_x + 2.0 * k3_x + k4_x);
    let v_new = v + dt / 6.0 * (k1_v + 2.0 * k2_v + 2.0 * k3_v + k4_v);

    (x_new, v_new)
}
```

Rust 的函数返回多个值用元组 `(T1, T2)`。

### 3.4 所有权模型

Rust 最独特的概念。核心规则：

1. 每个值有且只有一个"所有者"（owner）
2. 所有者离开作用域时，值被自动释放（drop）
3. 同一时刻，要么有一个可变引用，要么有多个不可变引用

数值计算中常见的借用问题：

```rust
// 错误示例：同时持有可变和不可变引用
let mut data = Array1::zeros(100);
let ref1 = &data[0..50];       // 不可变借用
data[50] = 1.0;                // 尝试可变借用 → 编译错误

// 正确做法：限定借用范围
{
    let ref1 = data.slice(s![0..50]);
    // 使用 ref1
} // ref1 的生命周期结束
data[50] = 1.0;                // 现在可以修改
```

初学时最常见的编译错误都来自所有权规则。编译器通常会给出具体修改建议，仔细阅读错误信息通常能直接修复。

---

## 四、计算物理常用库

| 库名 | 功能 | 等价 Python |
|------|------|-------------|
| `ndarray` | N 维数组 | NumPy |
| `nalgebra` | 线性代数（矩阵、向量、旋转） | NumPy + SciPy |
| `peroxide` | 数值计算（ODE、线性代数、统计） | SciPy（部分） |
| `plotters` | 2D 绘图 | Matplotlib |
| `rustfft` | FFT | NumPy FFT |

安装方式：在 `Cargo.toml` 中添加 `[dependencies]` 部分。

---

## 五、完整示例：谐振子求解与可视化

求解一维谐振子方程：

$$\frac{d^2 x}{dt^2} = -\omega^2 x$$

初始条件 $x(0) = 1, v(0) = 0$，解析解 $x(t) = \cos(\omega t)$。

```rust
// src/main.rs
use ndarray::{Array1, s};
use plotters::prelude::*;

fn harmonic_force(x: f64, omega: f64) -> f64 {
    -omega * omega * x
}

fn rk4_step(x: f64, v: f64, dt: f64, omega: f64) -> (f64, f64) {
    let k1_x = v;
    let k1_v = harmonic_force(x, omega);
    let k2_x = v + 0.5 * dt * k1_v;
    let k2_v = harmonic_force(x + 0.5 * dt * k1_x, omega);
    let k3_x = v + 0.5 * dt * k2_v;
    let k3_v = harmonic_force(x + 0.5 * dt * k2_x, omega);
    let k4_x = v + dt * k3_v;
    let k4_v = harmonic_force(x + dt * k3_x, omega);

    let x_new = x + dt / 6.0 * (k1_x + 2.0 * k2_x + 2.0 * k3_x + k4_x);
    let v_new = v + dt / 6.0 * (k1_v + 2.0 * k2_v + 2.0 * k3_v + k4_v);
    (x_new, v_new)
}

fn main() -> Result<(), Box<dyn std::error::Error>> {
    let omega = 2.0 * std::f64::consts::PI;
    let dt = 0.0001;
    let t_max = 10.0;
    let n_steps = (t_max / dt) as usize;

    let mut t = Array1::zeros(n_steps);
    let mut x = Array1::zeros(n_steps);
    let mut v = Array1::zeros(n_steps);

    x[0] = 1.0;
    v[0] = 0.0;

    for i in 0..n_steps - 1 {
        t[i + 1] = t[i] + dt;
        let (x_new, v_new) = rk4_step(x[i], v[i], dt, omega);
        x[i + 1] = x_new;
        v[i + 1] = v_new;
    }

    // 计算误差
    let analytical: Array1<f64> = t.mapv(|ti| (omega * ti).cos());
    let error = &x - &analytical;
    let max_error = error.iter().map(|e| e.abs()).fold(0.0, f64::max);
    println!("最大误差: {:.2e}", max_error);

    // 绘图
    let root = BitMapBackend::new("harmonic_oscillator.png", (800, 600)).into_drawing_area();
    root.fill(&WHITE)?;

    let x_min = *x.iter().min_by(|a, b| a.partial_cmp(b).unwrap()).unwrap();
    let x_max = *x.iter().max_by(|a, b| a.partial_cmp(b).unwrap()).unwrap();

    let mut chart = ChartBuilder::on(&root)
        .caption("Rust: Harmonic Oscillator", ("sans-serif", 30))
        .margin(10)
        .x_label_area_size(40)
        .y_label_area_size(40)
        .build_cartesian_2d(0.0..t_max, x_min..x_max)?;

    chart.configure_mesh().draw()?;

    let points: Vec<(f64, f64)> = (0..n_steps).step_by(100)
        .map(|i| (t[i], x[i]))
        .collect();

    chart.draw_series(LineSeries::new(points, &RED))?
        .label("RK4 numerical")
        .legend(|(x, y)| PathElement::new(vec![(x, y), (x + 20, y)], &RED));

    let analytical_points: Vec<(f64, f64)> = (0..n_steps).step_by(100)
        .map(|i| (t[i], analytical[i]))
        .collect();

    chart.draw_series(LineSeries::new(analytical_points, &BLUE))?
        .label("analytical cos(2πt)")
        .legend(|(x, y)| PathElement::new(vec![(x, y), (x + 20, y)], &BLUE));

    chart.configure_series_labels().draw()?;
    root.present()?;

    println!("图像已保存至 harmonic_oscillator.png");
    Ok(())
}
```

运行：

```bash
cargo run --release
```

`--release` 开启优化，Debug 模式比 Release 慢 10–50 倍。计算物理模拟始终用 `--release` 编译。

**性能参考：** 上述代码（10 万步，dt=0.0001）在 Release 模式下约 5–15 毫秒完成，同等 Python/NumPy 实现约 200–500 毫秒。

---

## 六、进阶方向

**常微分方程：** `peroxide` 库提供与 SciPy `solve_ivp` 类似的接口。

**线性代数：** `nalgebra` 适合中小规模稠密矩阵；大规模稀疏矩阵可用 `sprs`。

**并行计算：** `rayon` 库提供数据并行迭代器，把 `iter()` 改成 `par_iter()` 即可多线程并行，几乎不需要改代码。

**与 Python 互操作：** `pyo3` 库允许你用 Rust 写 Python 扩展模块，在保持 Python 调用接口的同时用 Rust 加速内层循环。这是当前计算物理中 Rust 最实用的切入方式——不用重写整个项目，只替换性能瓶颈。

---

## 七、何时选择 Rust

| 适合 Rust | 不适合 Rust |
|-----------|-------------|
| 性能热点需要 C++ 级别速度 | 需要现成的高级数值算法（FFT 变体、特殊函数） |
| 大型长期维护的数值代码库 | 快速原型、一次性脚本 |
| 需要并发/并行且担心数据竞争 | 团队无 Rust 学习预算 |
| 与 Python 混合使用（PyO3） | 需要丰富的交互式可视化 |

建议工作流：Python 写原型 → 用 `cProfile` 找到瓶颈 → 用 Rust + PyO3 重写瓶颈 → 保持 Python 顶层逻辑。
