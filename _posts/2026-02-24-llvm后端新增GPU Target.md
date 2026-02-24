---
layout:     post
title:      "LLVM后端新增GPU Target"
subtitle:   "llvm"
date:       2026-02-24 20:40:00
author:     "Claude"
catalog: false
published: true
header-style: text
tags:
  - llvm
---

在LLVM后端新增一个GPU目标（Target），需要完成以下几个主要阶段的工作：

## 1. 注册新目标

在 `llvm/lib/Target/` 下创建新目录（如 `MyGPU/`），并在以下位置注册：

- `llvm/include/llvm/BinaryFormat/ELF.h` — 添加 ELF machine type（如需要）
- `llvm/include/llvm/ADT/Triple.h` — 添加 ArchType 枚举
- `llvm/lib/Support/Triple.cpp` — 添加 arch 字符串解析
- `llvm/CMakeLists.txt` 和 `llvm/lib/Target/CMakeLists.txt` — 注册目标

---

## 2. 实现 Target Description（TableGen）

这是核心工作，写 `.td` 文件描述硬件：

- **`MyGPURegisterInfo.td`** — 定义寄存器、寄存器类（RegisterClass）
- **`MyGPUInstrInfo.td`** — 定义指令集（操作码、操作数、编码格式、调度信息）
- **`MyGPUCallingConv.td`** — 定义调用约定（参数传递、返回值）
- **`MyGPUSchedule.td`** — 定义调度模型（流水线、延迟）

---

## 3. 实现后端 C++ 代码

### TargetMachine

- `MyGPUTargetMachine.h / .cpp` — 目标机器描述，注册 Pass 流水线

### 寄存器信息

- `MyGPURegisterInfo.h / .cpp` — 帧指针、保留寄存器、寄存器消除（eliminateFrameIndex）

### 指令信息

- `MyGPUInstrInfo.h / .cpp` — 复制、跳转、分支分析等

### 指令选择

- `MyGPUISelDAGToDAG.cpp` — DAG 指令选择（SelectionDAG → MachineInstr）
- `MyGPUISelLowering.h / .cpp` — 各类 ISD 节点的 Lowering（call、ret、load/store、intrinsic 等）

### 帧/调用约定

- `MyGPUFrameLowering.h / .cpp` — 函数序言/尾声（prologue/epilogue），栈帧布局

### 汇编输出

- `MyGPUAsmPrinter.cpp` — MachineInstr → 汇编文本
- `MyGPUInstPrinter.h / .cpp` — 单条指令的打印格式

### 子目标特性

- `MyGPUSubtarget.h / .cpp` — 不同 GPU 型号的特性开关（Feature Flags）

---

## 4. MC 层（Machine Code）

如果需要汇编器/二进制输出：

- `MCTargetDesc/MyGPUMCTargetDesc.cpp` — MC 目标描述注册
- `MCTargetDesc/MyGPUMCCodeEmitter.cpp` — 指令编码为二进制
- `MCTargetDesc/MyGPUAsmBackend.cpp` — 重定位、fixup 处理
- `MCTargetDesc/MyGPUELFObjectWriter.cpp` — ELF 输出（如适用）

---

## 5. 特殊的 GPU 相关处理

GPU 后端通常还需要额外考虑：

- **地址空间**（Address Spaces）：区分 global / shared / local / constant memory
- **线程/波前模型**（SIMT/warp）的指令处理
- **内核函数（kernel）调用约定**与普通函数的区分
- **Barrier、同步指令**的处理
- **向量化与 SIMD**指令的支持
- **PTX / SPIR-V 等中间格式**的输出（如目标不直接生成二进制）
- **Intrinsic 扩展**（对应 CUDA/OpenCL 内置函数）

---

## 6. 测试

- 在 `llvm/test/CodeGen/MyGPU/` 下添加 `.ll` 测试用例
- 用 `llc -march=mygpu` 验证 IR → 汇编的正确性
- 添加 MC 层测试验证编码正确性

---

## 参考已有 GPU 后端

| 后端  | 参考价值 |
| --- | --- |
| `AMDGPU` | 最完整的开源 GPU 后端，架构复杂但功能全面 |
| `NVPTX` | 输出 PTX 文本而非二进制，相对简单 |
| `SPIRV` | 输出 SPIR-V 格式，适合 Vulkan/OpenCL |

建议从 **NVPTX** 开始参考，它结构相对清晰，是研究 GPU 后端的好起点。整体工作量在几万行代码量级，核心难点在于 ISelLowering 和 TableGen 指令描述的完整性。


使用ninja llc重新编译出新的llc程序

流程大致如下：

## 编译流程

**1. 配置 CMake（包含新目标）**

```bash
cmake -G Ninja \
  -DLLVM_TARGETS_TO_BUILD="X86;AMDGPU;MyGPU" \  # 加入新目标
  -DCMAKE_BUILD_TYPE=Release \
  ../llvm
```

**2. 用 Ninja 编译 llc**

```bash
ninja llc
# 或者只编译你的目标库加 llc
ninja LLVMMyGPUCodeGen llc
```

编译成功后，`bin/llc` 就包含了你新增的 GPU 后端。

---

## 验证新目标已注册

```bash
# 查看是否列出了新目标
./bin/llc --version

# 用新目标编译一个 IR 文件
./bin/llc -march=mygpu test.ll -o test.s
```

---

## 开发阶段的建议

- TableGen 文件改动后，Ninja 会自动重新运行 `llvm-tblgen` 生成对应的 `.inc` 文件，再编译 C++，**不需要手动干预**。
- 初期调试可以只 `ninja llc`，不必全量 `ninja`，节省时间。
- 如果只改了某个 `.cpp`，Ninja 的增量编译很快，改动 `.td` 文件则会触发较多重新编译。

所以整个开发循环就是：**改代码 → `ninja llc` → `llc -march=mygpu` 测试**，反复迭代。