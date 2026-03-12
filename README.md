# generateBrittleFractureDataset

[English version below](#english)

---

## 简介

本项目使用**物质点法（Material Point Method，MPM）**自动生成脆性断裂数据集。给定一个封闭的三角网格（watertight triangle mesh），程序会将其归一化到单位尺度，然后模拟一个刚性小球从随机方向撞击物体。每次碰撞后，程序将输出：

- 碰撞接触力与碰撞方向
- 断裂表面（裂缝面）
- 碰撞后产生的各个碎片

---

## 目录

- [功能说明](#功能说明)
- [项目结构](#项目结构)
- [依赖环境](#依赖环境)
- [编译流程](#编译流程)
- [运行说明](#运行说明)
- [输出格式](#输出格式)
- [算法原理](#算法原理)
- [已知问题](#已知问题)

---

## 功能说明

### 核心功能

1. **自动化脆性断裂仿真**：对输入的三维网格物体进行多次随机方向的刚性球撞击，每次撞击均为一组独立的仿真实验（trial）。
2. **物质点法（MPM）仿真**：采用 MPM 方法模拟弹性固体的大变形动力学过程，使用二次 B 样条核函数进行粒子—网格插值。
3. **损伤演化模型**：基于局部连续损伤力学，通过应力特征值判断材料是否进入损伤状态，损伤变量 `d ∈ [0,1]` 控制材料刚度退化。当 `d ≥ 0.97` 时视为完全断裂。
4. **裂缝面提取**：利用 **Voronoi 图（Voro++ 库）** 从损伤粒子中提取完整的裂缝表面，并对碎片进行连通性分析，输出各碎片的几何形状。
5. **数据集生成**：自动遍历多组随机方向（由 `randomSeeds.txt` 控制），批量生成包含碰撞信息与几何结果的数据集。

### 材料参数（以脆性材料为例）

| 参数 | 值 | 说明 |
|------|-----|------|
| 杨氏模量 E | 3.2e12 Pa | 弹性刚度 |
| 泊松比 ν | 0.2 | 横向变形比 |
| 断裂能 Gf | 3.2e5 J/m² | 裂缝扩展能量 |
| 应力阈值 θf | 8.0e9 Pa | 触发损伤的临界应力 |
| 损伤阈值 | 0.97 | 完全断裂判据 |

---

## 项目结构

```
generateBrittleFractureDataset/
├── CMakeLists.txt                          # 主构建配置文件
├── README.md                               # 项目说明文档
├── .gitmodules                             # Git 子模块配置
├── src/                                    # 源文件
│   ├── main.cpp                            # 主程序入口，仿真主循环
│   ├── particles.cpp                       # 粒子初始化（从网格读取）
│   ├── weights.cpp                         # B 样条权函数计算
│   ├── advance.cpp                         # MPM 单步仿真推进
│   ├── utils.cpp                           # 工具函数（文件读写、矩阵运算等）
│   └── damageGradient.cpp                  # 损伤场梯度计算
├── include/generateBrittleFractureDataset/ # 头文件
│   ├── particles.h                         # MPM 粒子结构体
│   ├── weights.h                           # 权函数结构体
│   ├── grid.h                              # 背景网格节点结构体
│   ├── materials.h                         # 材料参数结构体
│   ├── utils.h                             # 工具函数声明与参数结构体
│   ├── advance.h                           # MPM 推进函数声明
│   ├── damageGradient.h                    # 损伤梯度函数声明
│   └── extractCrack.h                      # 裂缝面提取（仅头文件实现）
├── extern/                                 # 第三方依赖（Git 子模块）
│   ├── CMakeLists.txt
│   ├── eigen/                              # Eigen 线性代数库
│   └── voro/                               # Voro++ Voronoi 图库
├── input/                                  # 输入数据
│   ├── lion.obj                            # 示例输入网格（狮子模型）
│   ├── lionParticles_20000.obj             # 预计算的 20000 个内部粒子
│   ├── sphereParticles.obj                 # 刚性小球粒子
│   └── randomSeeds.txt                     # 随机方向种子文件
├── output/                                 # 仿真结果输出目录
└── scripts/
    └── TODO.txt                            # 已知问题记录
```

---

## 依赖环境

### 必要依赖

| 依赖 | 版本要求 | 说明 |
|------|---------|------|
| C++ 编译器 | 支持 C++14 | GCC 5+、Clang 3.5+、MSVC 2015+ |
| CMake | ≥ 3.11 | 构建系统 |
| Eigen | 随仓库附带 | 线性代数库（子模块） |
| Voro++ | 随仓库附带 | Voronoi 图计算库（子模块） |

### 可选依赖

| 依赖 | 说明 |
|------|------|
| OpenMP | 多线程并行加速（强烈推荐，默认启用） |

### 平台说明

- **Linux / macOS**：推荐使用 GCC 或 Clang，OpenMP 支持良好。
- **Windows**：需要 Visual Studio 2015 及以上版本，OpenMP 通过 MSVC 编译器支持。

---

## 编译流程

### 第一步：克隆仓库并初始化子模块

```bash
git clone --recursive https://github.com/marksweli/generateBrittleFractureDataset.git
cd generateBrittleFractureDataset
```

> 如果已经克隆但忘记 `--recursive`，可以运行：
>
> ```bash
> git submodule update --init --recursive
> ```

### 第二步：创建构建目录

```bash
mkdir build
cd build
```

### 第三步：配置 CMake

**Linux / macOS（推荐 Release 模式）：**

```bash
cmake .. -DCMAKE_BUILD_TYPE=Release
```

**禁用 OpenMP（不推荐，仿真速度会大幅降低）：**

```bash
cmake .. -DCMAKE_BUILD_TYPE=Release -DUSE_OpenMP=OFF
```

**Windows（Visual Studio）：**

```bash
cmake .. -G "Visual Studio 16 2019" -A x64
```

### 第四步：编译

**Linux / macOS：**

```bash
cmake --build . --config Release
# 或使用 make（利用全部 CPU 核心）：
make -j$(nproc)
```

**Windows（命令行）：**

```bash
cmake --build . --config Release
```

或在 Visual Studio 中打开生成的 `.sln` 文件，选择 **Release** 配置后按 **F7** 编译。

### 编译成功标志

编译完成后，可执行文件位于：

- Linux / macOS：`build/generateBrittleFractureDataset`
- Windows：`build/Release/generateBrittleFractureDataset.exe`

---

## 运行说明

### 前提条件

确保以下输入文件存在于 `input/` 目录下：

| 文件 | 说明 |
|------|------|
| `lion.obj` | 输入网格（狮子模型） |
| `lionParticles_20000.obj` | 对应的 20000 个内部粒子位置 |
| `sphereParticles.obj` | 刚性球粒子 |
| `randomSeeds.txt` | 随机方向种子数据 |

### 运行命令

在项目根目录（`generateBrittleFractureDataset/`）下运行：

```bash
./build/generateBrittleFractureDataset
```

> **注意**：程序通过 CMake 宏 `ROOT_DIR` 定位输入文件，因此**必须在项目根目录下运行**，否则无法找到 `input/` 目录中的文件。

### 主要仿真参数

以下参数在 `src/main.cpp` 中硬编码，如需修改请直接编辑源代码后重新编译：

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `numBunnyInteriorPar` | 20000 | 物体内部 MPM 粒子数量 |
| `sphereRadius` | 0.02 | 刚性球半径（归一化单位） |
| `sphereVelMag` | 100 | 刚性球碰撞初速度大小 |
| `dt` | 1.0×10⁻⁶ s | 仿真时间步长 |
| `dx` | `(2/20000)^(1/3) / 2` | 背景网格间距 |
| `numOfThreads` | 40 | OpenMP 线程数 |
| 仿真轮数 | 371 次（k=329 至 700） | 总碰撞试验次数 |
| 最大时间步数 | 2000 步/轮 | 每次碰撞的最大仿真步数 |

### 运行时间估计

每一轮碰撞仿真约需数分钟（取决于硬件和线程数），全部 371 轮完整运行需要较长时间。建议在多核服务器上运行，并根据实际核心数调整 `numOfThreads`。

---

## 输出格式

所有输出文件保存在 `output/` 目录下。

### 全局文件

| 文件 | 说明 |
|------|------|
| `output/lion_mesh.obj` | 原始输入网格 |
| `output/lion.obj` | 最后一帧的变形网格 |

### 每轮仿真输出（`output/VT_k/` 目录，k 为轮次编号）

| 文件 | 说明 |
|------|------|
| `crackSurfaceFull.obj` | 完整裂缝表面（三角网格） |
| `fragment_0.obj` | 第 0 号碎片网格 |
| `fragment_1.obj` | 第 1 号碎片网格 |
| `fragment_N.obj` | 第 N 号碎片网格（数量因撞击而异） |
| `contact.txt` | 碰撞点位置与方向信息 |

---

## 算法原理

### 1. 物质点法（MPM）

MPM 将连续体离散为一组携带物理量（位置、速度、变形梯度、应力等）的粒子，并在欧拉背景网格上求解动量方程。每个时间步的流程如下：

```
粒子 → 网格（P2G）：
  将粒子的质量、动量通过二次 B 样条权函数插值到背景网格节点

应力计算：
  基于 Neo-Hookean 超弹性本构模型计算每个粒子的 Cauchy 应力
  引入损伤变量 d 对拉伸应力进行退化

网格力计算与更新：
  由粒子应力计算网格节点的内力
  通过动量方程更新网格节点速度

网格 → 粒子（G2P）：
  将网格节点速度插值回粒子，更新粒子速度与变形梯度
```

### 2. 损伤演化模型

损伤变量 `d` 根据应力最大特征值 `σ_max` 演化：

```
d = 0,                                    若 σ_max ≤ θ_f
d = (1 + H_s)(1 - θ_f / σ_max),          若 θ_f < σ_max < (1 + 1/H_s)·θ_f
d = 1,                                    若 σ_max ≥ (1 + 1/H_s)·θ_f
```

其中 `θ_f` 为应力阈值，`H_s` 为硬化模量（由断裂能 `Gf` 和特征长度计算）。当 `d ≥ 0.97` 时，该粒子被标记为"完全断裂"。

### 3. 裂缝面提取（Voronoi 图方法）

1. 将完全损伤粒子（`d ≥ 0.97`）从系统中分离。
2. 对损伤边界节点使用 **Voro++ 库**计算 Voronoi 图。
3. 对每个 Voronoi 单元，检查其面两侧的粒子损伤状态——若一侧已损伤、另一侧未损伤，则该面属于裂缝表面。
4. 对未损伤的 Voronoi 单元进行连通性分析，将相互独立的连通区域分别输出为碎片。

### 4. 刚性球碰撞

刚性球由一组不可断裂的粒子构成，以指定速度向物体施加集中力（`applyPointForce()`）。碰撞方向由 `randomSeeds.txt` 中预先生成的随机向量决定。

---

## 已知问题

- 若粒子坐标出现负值，程序将在短时间内崩溃。请确保输入网格经过正确归一化，使所有粒子坐标均为正值。

---

<a name="english"></a>

---

# English

## Overview

This project uses the **Material Point Method (MPM)** to automatically generate a brittle fracture dataset. Given a watertight triangle mesh, the code normalizes it to unit scale and simulates a small rigid sphere striking the object from random directions. Outputs include contact force/direction, crack surfaces, and resulting fragments.

## Dependencies

- C++14 compiler (GCC 5+, Clang 3.5+, MSVC 2015+)
- CMake ≥ 3.11
- Eigen (included as submodule)
- Voro++ (included as submodule)
- OpenMP (optional but recommended)

## Build Instructions

```bash
# Clone with submodules
git clone --recursive https://github.com/marksweli/generateBrittleFractureDataset.git
cd generateBrittleFractureDataset

# Configure and build
mkdir build && cd build
cmake .. -DCMAKE_BUILD_TYPE=Release
cmake --build . --config Release
```

## Running

Run from the project root directory:

```bash
./build/generateBrittleFractureDataset
```

The program reads input meshes from `input/` and writes results to `output/VT_k/` for each trial `k`.

## Output

Each trial `k` produces files in `output/VT_k/`:

- `crackSurfaceFull.obj` — full crack surface mesh
- `fragment_N.obj` — individual fragment meshes
- `contact.txt` — contact point and impact direction

## Known Issues

- The program crashes if any particle has a negative coordinate. Ensure the input mesh is properly normalized.
