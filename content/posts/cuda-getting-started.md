---
title: 'CUDA 入门指南：GPU 并行计算的世界'
date: 2026-04-07T14:00:00+08:00
draft: false
tags: ['cuda', 'gpu', 'performance-optimization', 'ai']
categories: ['tech']
description: 'CUDA 是 NVIDIA 推出的并行计算平台，本文以第三方视角介绍 CUDA 的核心概念、编程模型，并通过一个实战例子带你快速入门 GPU 开发。'
---

2006 年，NVIDIA 推出了 **CUDA（Compute Unified Device Architecture）**，这是一套针对自家 GPU 的并行计算平台和编程模型。在此之前，GPU 只能处理图形渲染；有了 CUDA，开发者终于能用熟悉的 C/C++ 语言调动 GPU 的超强算力。

大语言模型训练、深度学习推理、科学计算——这些动辄需要处理 TB 级数据的任务，几乎都离不开 CUDA。本文从一个中立视角出发，带你了解 CUDA 的核心概念，并通过一个实战例子让你快速上手。

<!--more-->

## 为什么需要 CUDA？

让我们先理解问题的本质：**CPU 和 GPU 的设计目标不同**。

| 设计目标 | CPU | GPU |
|----------|-----|-----|
| 核心思想 | 少量复杂计算，低延迟 | 大量简单计算，高吞吐 |
| 晶体管分配 | 复杂控制单元 + 大缓存 | 大量简单计算单元 |
| 适合场景 | 串行任务、分支判断 | 数据并行、矩阵运算 |
| 核心数 | 8-32 核（消费级） | 数千核（RTX 4090 有 16384 个 CUDA 核） |

打个比方：CPU 像一个由少数博士生组成的团队，每个人能力很强，能独立解决复杂问题；GPU 则像一个由数千名小学生组成的团队，单兵作战能力弱，但**团队协作**能做海量简单计算——比如数一亿颗豆子，GPU 能在几毫秒内完成，而 CPU 需要几十秒。

现代 AI 训练正是这样的场景：**权重矩阵乘法、反向传播**，本质都是可以高度并行的浮点运算。

## CUDA 核心概念

CUDA 程序分为 **Host（CPU）** 和 **Device（GPU）** 两部分。GPU 上的代码称为 **Kernel**，由 CPU 调用执行。

**Thread/Block/Grid 三级并行结构**：
- `Thread` 是最小执行单元
- `Block` 是一组线程（最多 1024 个），Block 内线程可通过 `__shared__` 共享内存通信
- `Grid` 是所有 Block 的集合

Kernel 启动时指定 Block 数和每个 Block 的线程数：

```cpp
my_kernel<<<256, 256>>>(d_data, size);  // 256个Block，每Block 256线程
```

线程通过 `blockIdx.x * blockDim.x + threadIdx.x` 计算全局 ID，访问对应数据。

## 实战：并行归约求和

我们来实现一个**并行归约（Parallel Reduction）**：对 1 亿个浮点数求和。

### 核心思路

CPU 串行累加需要 O(N) 次操作。GPU 让数千个线程同时工作，每个线程负责累加一部分，最后再合并结果。

### 关键代码

**Kernel 实现（使用共享内存 + 树形归约）：**

```cpp
__global__ void reduction_kernel(float *input, float *output, int n) {
    __shared__ float shared_data[256];

    unsigned int tid = threadIdx.x;
    unsigned int idx = blockIdx.x * blockDim.x + threadIdx.x;

    // 每个线程加载一个元素到共享内存
    shared_data[tid] = (idx < n) ? input[idx] : 0.0f;
    __syncthreads();

    // 树形归约：每轮减半
    for (unsigned int s = blockDim.x / 2; s > 0; s >>= 1) {
        if (tid < s) {
            shared_data[tid] += shared_data[tid + s];
        }
        __syncthreads();
    }

    // Block 结果写回全局内存
    if (tid == 0) {
        output[blockIdx.x] = shared_data[0];
    }
}
```

**主函数调用：**

```cpp
int main() {
    float *d_input, *d_intermediate, *d_output;

    // 分配GPU内存、拷贝数据...
    cudaMalloc(&d_input, N * sizeof(float));
    cudaMalloc(&d_intermediate, 256 * sizeof(float));
    cudaMalloc(&d_output, sizeof(float));
    cudaMemcpy(d_input, h_data, N * sizeof(float), cudaMemcpyHostToDevice);

    // 两阶段归约
    reduction_kernel<<<256, 256>>>(d_input, d_intermediate, N);  // 阶段1
    final_reduction<<<1, 1>>>(d_intermediate, d_output, 256);    // 阶段2

    // 拷贝结果回CPU...
}
```

**树形归约过程（8 个元素为例）：**

```
第1轮: [0+4] [1+5] [2+6] [3+7]   →  4个元素
第2轮: [0+2] [1+3]              →  2个元素
第3轮: [0+1]                    →  1个元素
```

`log2(256) = 8` 轮即可完成一个 Block 的归约。

### 编译运行

```bash
nvcc reduction.cu -o reduction -O3
./reduction
```

典型输出：

```
=== CUDA Parallel Reduction ===
Array size: 104857600 elements (400.00 MB)
CPU Result: 52486393.23
GPU Result: 52486393.23
GPU Time: 0.89 ms
Speedup: 160x
```

160 倍加速！这个例子展示了 GPU 并行计算的威力。

## CUDA 生态一览

学会基本概念后，你可以探索更多方向：

**深度学习**：cuDNN、TensorRT、CUDA-X AI（TensorFlow/PyTorch 底层加速）

**科学计算**：cuBLAS（矩阵运算）、cuFFT（傅里叶变换）、cuSOLVER（线性代数求解器）、Thrust（C++ STL 风格的 GPU 算法库）

**新兴工具**：CuPy/Numba（Python 调用 CUDA）、OpenAI Triton（更简单的 GPU 算子开发）

## 常见问题

**Q: 我的电脑没有 NVIDIA 显卡，能学 CUDA 吗？**
A: 可以用 Google Colab 等云端 GPU，或者先学习概念，不实际运行也能理解。

**Q: CUDA 只能用于 AI 吗？**
A: 不是。CUDA 适用于任何**数据并行**的计算密集型任务：物理仿真、图像处理、密码学、分子动力学等。

**Q: AMD 显卡能用 CUDA 吗？**
A: 不能。CUDA 是 NVIDIA 专有技术。AMD 显卡使用 **ROCm**（Radeon Open Compute Ecosystem）。

## 总结

CUDA 打开了一扇通向 GPU 并行计算的大门。核心概念简单明了：

1. **Host/Device** 分工明确
2. **Thread/Block/Grid** 三级并行结构
3. **Shared Memory** 实现 Block 内高效通信
4. **树形归约** 是并行计算的经典模式

掌握这些基础后，你可以继续探索深度学习优化、科学计算加速，或者转向更高级的工具链（如 TensorRT、cuDNN）。

大语言模型的训练和推理背后，CUDA 是最重要的基础设施之一。理解它，你就能更深入地理解 AI 系统的底层运行原理。
