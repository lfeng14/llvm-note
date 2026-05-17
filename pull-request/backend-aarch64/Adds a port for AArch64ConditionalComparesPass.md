#### Description
[linker](https://github.com/llvm/llvm-project/pull/192755/changes)

#### Analysis
- 刚刚打开的链接显示，它其实是关于 **将 `AArch64ConditionalComparesPass` 迁移到新的 Pass Manager 架构（NewPM）**的，与你之前提到的 `FPR16` 子寄存器逻辑无关。
  ```cpp
  bool AArch64ConditionalComparesLegacy::runOnMachineFunction(
      MachineFunction &MF) {
    if (skipFunction(MF.getFunction()))
      return false;
  
    auto *MBPI =
        &getAnalysis<MachineBranchProbabilityInfoWrapperPass>().getMBPI();
    auto *DomTree = &getAnalysis<MachineDominatorTreeWrapperPass>().getDomTree();
    auto *Loops = &getAnalysis<MachineLoopInfoWrapperPass>().getLI();
    auto *Traces = &getAnalysis<MachineTraceMetricsWrapperPass>().getMTM();
  
    AArch64ConditionalComparesImpl Impl(MBPI, DomTree, Loops, Traces);
    return Impl.run(MF);
  }
  
  PreservedAnalyses
  AArch64ConditionalComparesPass::run(MachineFunction &MF,
                                      MachineFunctionAnalysisManager &MFAM) {
    auto *MBPI = &MFAM.getResult<MachineBranchProbabilityAnalysis>(MF);
    auto *DomTree = &MFAM.getResult<MachineDominatorTreeAnalysis>(MF);
    auto *Loops = &MFAM.getResult<MachineLoopAnalysis>(MF);
    auto *Traces = &MFAM.getResult<MachineTraceMetricsAnalysis>(MF);
  
    AArch64ConditionalComparesImpl Impl(MBPI, DomTree, Loops, Traces);
    bool Changed = Impl.run(MF);
    if (!Changed)
      return PreservedAnalyses::all();
  
    PreservedAnalyses PA = getMachineFunctionPassPreservedAnalyses();
    PA.preserve<MachineDominatorTreeAnalysis>();
    PA.preserve<MachineLoopAnalysis>();
    PA.preserve<MachineTraceMetricsAnalysis>();
    return PA;
  }
  ```
- 🎯 PR 的核心背景与目标
这个 PR 是在 LLVM 全面推行 **NewPM** 的背景下进行的，它的目标是**代码架构的现代化重构，而非修复具体的指令选择 Bug**。

*   **背景**：LLVM 正在从“传统 Pass 管理器”迁移到功能更强大、结构更清晰的“NewPM”。为了让 AArch64 后端能全面适配 NewPM，需要逐一为所有后端优化 Pass 编写 NewPM 接口。
*   **目标**：为 `AArch64ConditionalCompares` 这个 Pass 创建 NewPM 版本，同时保持其 Legacy PM 接口的可用性。

- 🛠️ 主要修改内容
为了实现这一目标，PR 主要进行了以下代码层面的调整：
*   **拆分实现逻辑**：将 Pass 的核心优化代码从 Legacy PM 类中抽离，放入一个新类 `AArch64ConditionalComparesImpl`。这样，新旧两种 Pass Manager 可以共享同一套优化算法。
*   **保留旧接口**：创建了 `AArch64ConditionalComparesLegacy` 类作为“包装器”。它继承自 `MachineFunctionPass`，其核心的 `runOnMachineFunction` 函数会去调用 `Impl` 类的逻辑，从而保留 Legacy PM 下通过 `opt` 命令行直接调用的能力。
*   **增加新接口**：新增了 `AArch64ConditionalComparesPass` 类，它遵循 NewPM 规范。其 `run` 方法通过 `MachineFunctionAnalysisManager` 获取所需分析结果，并调用 `Impl` 完成优化。
*   **注册更新**：在 `AArch64PassRegistry.def` 中添加了新 Pass 的声明，确保 NewPM 能识别它。

- 🧪 测试用例更新
为了验证 NewPM 版本能正常工作，PR 也更新了相关的测试用例：
*   **新增 NewPM 测试**：在 `ccmp-look-through-copy.mir` 和 `ccmp-successor-probs.mir` 这两个测试中，**增加了**使用 `-passes=aarch64-ccmp`（NewPM 方式）的运行指令。
*   **保留旧测试**：原有的 `-run-pass=aarch64-ccmp`（Legacy PM 方式）指令被**保留**，以确保新旧两种接口均能正常工作。

- 💎 总结
总而言之，这个 PR 的重点是 **`AArch64ConditionalCompares` 这个特定 Pass 的架构迁移，确保其能在 LLVM 的新 Pass Manager 框架下运行**。
