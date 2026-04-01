# LLVM 学习笔记

LLVM 编译器基础设施、优化技术及系统编程相关知识的个人学习笔记集合。

## 目录结构

```
llvm-note/
├── llvm-pass/           # LLVM Pass 相关
├── llvm-optimization/   # LLVM 优化技术
├── pgo-tools/           # 性能分析引导优化工具
├── memory/              # 内存系统相关
├── concurrency/         # 并发与多线程
├── linking/             # 链接与调用约定
├── other/               # 其他主题
├── optimization-checklist.md
└── README.md
```

## 内容分类

### LLVM Pass

- [using-the-new-pass-manager.md](llvm-pass/using-the-new-pass-manager.md) - 新 Pass Manager 使用指南
- [add-a-pass.md](llvm-pass/add-a-pass.md) - 添加自定义 Pass
- [mem2reg.md](llvm-pass/mem2reg.md) - mem2reg  Pass 详解

### LLVM 优化技术

- [biased-branches.md](llvm-optimization/biased-branches.md) - 分支偏置优化
- [jump-table-scalarization-opt.md](llvm-optimization/jump-table-scalarization-opt.md) - 跳转表标量化优化
- [alias-analysis.md](llvm-optimization/alias-analysis.md) - 别名分析
- [func-ret-opt.md](llvm-optimization/func-ret-opt.md) - 函数返回值优化
- [loop-pealing-and-versioning.md](llvm-optimization/loop-pealing-and-versioning.md) - 循环剥离与多版本
- [small-buffer-opt.md](llvm-optimization/small-buffer-opt.md) - 小缓冲区优化
- [scalarization-opt.md](llvm-optimization/scalarization-opt.md) - 标量化优化
- [strength-reduction.md](llvm-optimization/strength-reduction.md) - 强度削减
- [duplicate-filtering.md](llvm-optimization/duplicate-filtering.md) - 重复过滤
- [sorting-with-filtering.md](llvm-optimization/sorting-with-filtering.md) - 带过滤的排序

### PGO 工具

- [llvm-pgo.md](pgo-tools/llvm-pgo.md) - LLVM 性能分析引导优化
- [profile-guided-optimization.md](pgo-tools/profile-guided-optimization.md) - PGO 概述
- [propeller.md](pgo-tools/propeller.md) - Propeller 优化工具
- [bolt.md](pgo-tools/bolt.md) - BOLT 二进制优化工具

### 内存系统

- [memory-system.md](memory/memory-system.md) - 内存系统概述
- [hardware-memory-barriers.md](memory/hardware-memory-barriers.md) - 硬件内存屏障
- [memory-barrier-million-dollar-bug.md](memory/memory-barrier-million-dollar-bug.md) - 内存屏障相关的"百万美元 bug"
- [false-sharing.md](memory/false-sharing.md) - 伪共享问题
- [garbage-collection-safepoints-in-llvm.md](memory/garbage-collection-safepoints-in-llvm.md) - LLVM 中的垃圾回收安全点
- [llvm-garbage-collection.md](memory/llvm-garbage-collection.md) - LLVM 垃圾收集

### 并发与多线程

- [thread-local-variable.md](concurrency/thread-local-variable.md) - 线程局部变量
- [thread-affinity.md](concurrency/thread-affinity.md) - 线程亲和性
- [peterson-algorithm-for-mutual-exclusion.md](concurrency/peterson-algorithm-for-mutual-exclusion.md) - Peterson 互斥算法

### 链接与调用约定

- [linker.md](linking/linker.md) - 链接器相关知识
- [unwinding-calling-conversion.md](linking/unwinding-calling-conversion.md) - 栈展开与调用约定

### 其他

- [assert.md](other/assert.md) - 断言相关
- [pointer-aliasing-in-cuda.md](other/pointer-aliasing-in-cuda.md) - CUDA 中的指针别名

### 优化检查清单

- [optimization-checklist.md](optimization-checklist.md) - 性能优化检查清单，按难易程度排序的优化方法

---

## 说明

本仓库为个人学习笔记，内容可能不完整或存在错误，仅供参考。
