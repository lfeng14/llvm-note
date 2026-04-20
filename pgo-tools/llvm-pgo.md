#### LLVM PGO 流程（可通过小例子学习）
- 如何插桩编译
- 如何运行采集
- 如何 合并数据
- 如何反馈重编

#### Analysis
- 采样方式信息：
  - Sampled line represents the profile information of a source location. offsetN[.discriminator]: number_of_samples [fn5:num fn6:num ... ]
  - Callsite line represents the profile information of an inlined callsite. offsetA[.discriminator]: fnA:num_of_total_samples
  ```
  function1:total_samples:total_head_samples  # 函数为单位，被执行次数
   offset1[.discriminator]: number_of_samples [fn1:num fn2:num ... ]  # 函数定位位置为起始地址，所以如果代码修改了，这时导致profile信息失效；同一行也有可能调用fn1 fn2  比如 if (xx) fn1(); else fn2();
   offset2[.discriminator]: number_of_samples [fn3:num fn4:num ... ]
   ...
   offsetN[.discriminator]: number_of_samples [fn5:num fn6:num ... ]  
   offsetA[.discriminator]: fnA:num_of_total_samples
    offsetA1[.discriminator]: number_of_samples [fn7:num fn8:num ... ]  # 记录已经内联的调用；如果非内联则没有缩进
    offsetA1[.discriminator]: number_of_samples [fn9:num fn10:num ... ]
    offsetB[.discriminator]: fnB:num_of_total_samples
     offsetB1[.discriminator]: number_of_samples [fn11:num fn12:num ... ]
  ```
  - This profile indicates that there were a total of 35,504 samples collected in main. All of those were at line 1 (the call to foo). Of those, 31,977 were spent inside the body of bar. The last line of the profile (2: 0) corresponds to line 2 inside main. No samples were collected there.
  ```
  main:35504:0
  1: _Z3foov:35504
    2: _Z32bari:31977
    1.1: 31977
  2: 0
  ```
- 插桩方式信息：
  - 可以充当函数覆盖率信息
  - 多编译一次插桩版本，插入计数指令，产生运行开销；记录数据更完整，效果比采样方式好；
  - Clang supports two types of instrumentation: frontend-based and IR-based. Frontend-based instrumentation can be enabled with the option -fprofile-instr-generate, and IR-based instrumentation can be enabled with the option -fprofile-generate. For best performance with PGO, IR-based instrumentation should be used. It has the benefits of lower instrumentation overhead, smaller raw profile size, and better runtime performance. Frontend-based instrumentation, on the other hand, has better source correlation, so it should be used with source line-based coverage testing.
  - 如果代码修改了，中端插桩影响更小
  - The flag -fcs-profile-generate also instruments programs using the same instrumentation method as -fprofile-generate. However, it performs a post-inline late instrumentation and can produce context-sensitive profiles.
  - LLVM_PROFILE_FILE="code-%p.profraw" ./code
  - 不同的modifier 可以避免覆盖： %p, %h, %m, %b, %t, and %c
  - Both -fprofile-use and -fprofile-instr-use accept profiles in the indexed format, regardeless whether it is produced by frontend or the IR pass.
  - -fcs-profile-generate[=<dirname>]：The difference is that the instrumentation is performed after inlining so that the resulted profile has a better context sensitive information
  - -fprofile-update：Unless -fsanitize=thread is specified, the default is single, which uses non-atomic increments. The counters can be inaccurate under thread contention. atomic uses atomic increments which is accurate but has overhead. prefer-atomic will be transformed to atomic when supported by the target, or single otherwise.
  - -fprofile-continuous：Enables the continuous instrumentation profiling where profile counter updates are continuously synced to a file
  - -ftemporal-profile：Enables the temporal profiling extension for IRPGO to improve startup time by reducing .text section page faults. To do this, we instrument function timestamps to measure when each function is called for the first time and use this data to generate a function order to improve startup
  - Fine Tuning Profile Collection
    - void __llvm_profile_set_filename(const char *Name): changes the name of the profile file to Name.
    - void __llvm_profile_reset_counters(void): resets all counters to zero.
    - int __llvm_profile_dump(void): write the profile data to disk.
    ```
    int main() {
      initialize();
    
      // Reset all profile counters to 0 to omit profile collected during
      // initialize()'s execution.
      __llvm_profile_reset_counters();
      ... hot region 1
      // Dump the profile for hot region 1.
      __llvm_profile_set_filename("region1.profraw");
      __llvm_profile_dump();
    
      // Reset counters before proceeding to hot region 2.
      __llvm_profile_reset_counters();
      ... hot region 2
      // Dump the profile for hot region 2.
      __llvm_profile_set_filename("region2.profraw");
      __llvm_profile_dump();
    
      // Since the profile has been dumped, no further profile data
      // will be collected beyond the above __llvm_profile_dump().
      cleanup();
      return 0;
    }
    ```
    - \_\_LLVM_INSTR_PROFILE_GENERATE: defined when one of -fprofile[-instr]-generate/-fcs-profile-generate is in effect.
    - \_\_LLVM_INSTR_PROFILE_USE: defined when one of -fprofile-use/-fprofile-instr-use is in effect.    
    ```
    #if __LLVM_INSTR_PROFILE_GENERATE
    expensive_logging_of_full_program_state();
    #endif
    ```
    - Instrumenting only selected files or functions: clang++ -O2 -fprofile-instr-generate -fprofile-list=fun.list code.cc -o code
- 论文《[Efficient Profiling in the LLVM Compiler Infrastructure](https://llvm.org/pubs/2010-04-NeustifterProfiling.pdf)》：
  - 最优计数器放置 - 基于 Ball & Larus (1994),只在 CFG 的非生成树边上放置计数器,平均从 93.5% 减少到 47.5%
  - 虚拟边处理 - 处理只有单个基本块的函数等特殊情况
  - 将 profiling 信息传递给后端 - 寄存器分配器等可以利用这些信息
  - 实现了静态估算器 - 当没有动态 profiling 数据时使用
- 论文《[PGO and LLVM: Status and Current Work](http://llvm.org/devmtg/2013-11/slides/Carruth-PGO.pdf)》

  - 一、关键设计
  	- 1. **基于 AST 的插桩与 Profile 关联**
  	  - **设计**：将计数器直接与 Clang 的抽象语法树（AST）节点关联，而不是与 LLVM IR 指令绑定。
  	  - **目的**：使 Profile 数据对编译器版本和 IR 变换不敏感；只要源代码的控制流结构不变，Profile 仍然有效。
  
  	- 2. **最小化插桩开销**
  	  - **设计**：不必要为每个基本块都插桩。通过计算控制流图（CFG）的生成树（spanning tree）或利用 AST 结构，仅对必要的控制流边插入计数器，减少运行时和编译时的开销。
  
  	- 3. **外部采样 Profile 支持**
  	  - **设计**：支持使用 Linux `perf` 等硬件计数器采样工具，无需插桩构建。通过转换工具将采样数据映射到源代码位置，再转换为 LLVM IR 中的分支权重（branch weights）。
  	  - **优点**：运行时开销极低（<1%），可在生产环境收集 Profile，数据更具代表性。
  
  	- 4. **统一的 IR 元数据表示**
  	  - **设计**：所有 Profile 信息最终都以 `!prof` 元数据的形式附加在 LLVM IR 的分支指令上。分析 pass（如 `BranchProbabilityInfo`、`BlockFrequencyInfo`）通过统一 API 读取这些元数据，若缺失则回退到静态启发式。
  
  	- 5. **应对代码和编译器的变化**
  	  - **设计**：通过检测源文件修改，仅丢弃发生变化的函数的 Profile 数据，保留其他函数的数据。Profile 不依赖于特定的编译器版本或 IR 布局，提升了复用性。
  
  	- 6. **分析 API 与优化集成**
  	  - **设计**：所有优化 pass 通过 `BranchProbabilityInfo` 和 `BlockFrequencyInfo` 获取分支概率和块频率，从而利用 Profile 指导决策，如寄存器分配（spill placement）、代码布局（MachineBlockPlacement）、内联等。
  - 二、所解决的核心问题
      | 问题 | 解决方案 / 设计点 |
      |------|------------------|
      | **传统插桩 PGO 开销大、编译慢** | 减少计数器数量、AST 级插桩、提供外部采样方案 |
      | **Profile 与编译器版本强耦合，易失效** | 将 Profile 关联到源代码（AST），而非 IR 指令 |
      | **代码变更导致 Profile 完全不可用** | 检测变更函数，仅丢弃相关部分，其他函数仍可用 |
      | **采样 Profile 精度低、难以映射到源码** | 利用调试信息（行号、列号、discriminators）精确映射，转换工具将其转为分支权重 |
      | **优化器未充分利用 Profile** | 统一分析 API，使 inliner、loop unroll、block placement 等都能使用 Profile |
      | **Profile 与静态启发式冲突** | 设计谨慎的冲突处理机制（警告、静默覆盖），避免性能倒退 |
      | **IR 变换后 Profile 元数据失效** | 需要 pass 维护 `!prof` 元数据（如反转分支时交换权重），保证后续优化仍能使用 |
      | **冷热代码分离不彻底** | 利用 Profile 进行函数冷热分区，改善指令缓存和代码布局 |
  
  - 三、你可以向 LLVM 社区提交的贡献方向
    基于文章提到的“Status”和“In the works”，结合当前 LLVM PGO 的实际情况，以下是一些具体且可行的贡献建议：
    - 1. **改进采样 Profile 转换工具**
      - 目前 LLVM 已有 `llvm-profgen` 等工具，但仍可提升精度（如支持更多事件、处理 inline 上下文）。
      - **贡献点**：增强 `llvm-profdata` 对采样 Profile 的合并/过滤功能；实现将 `perf` 数据更准确地映射到 LLVM IR 的算法，特别是处理多行指令、内联函数等场景。
  
    - 2. **完善上下文敏感 PGO（CSPGO）在 ThinLTO 中的支持**
      - 文章虽未详述 CSPGO，但它是当前 PGO 的重要方向。CSPGO 在内联后收集上下文信息，但在 ThinLTO 场景下，跨模块的上下文传递仍有优化空间。
      - **贡献点**：修复 ThinLTO 中上下文 Profile 丢失的问题；优化 Profile 合并算法以减少构建时间。
  
    - 3. **提升优化 pass 对 Profile 的利用率**
      - **Inliner**：虽然现在 inliner 已使用 Profile，但决策模型仍可改进，例如结合调用链的上下文热度，避免将冷调用内联到热函数中。
      - **Loop Unroll / Vectorizer**：当前这些优化对 Profile 的利用有限，可以增加基于块频率的展开因子决策。
      - **贡献点**：为某个优化 pass 添加基于 Profile 的启发式，并通过 SPEC 等基准测试验证效果。
  
    - 4. **处理 Profile 元数据在 IR 变换中的维护**
      - 许多优化 pass 会修改 CFG，但没有更新 `!prof` 元数据（例如将 `br` 转换为 `select`，或将条件分支拆分为多个分支）。
      - **贡献点**：实现一个通用的元数据更新机制，或在具体 pass（如 SimplifyCFG、InstCombine）中添加分支权重迁移逻辑，确保 Profile 信息不丢失。
  
    - 5. **增强冷热分区与函数拆分**
      - 当前 LLVM 有 `-fsplit-machine-functions` 等选项，但主要作用于机器码层。中端（如 `CodeGenPrepare`）的函数拆分仍有待完善。
      - **贡献点**：实现一个中间端函数拆分 pass，利用 Profile 将冷代码块提取为单独函数，并接入 LTO 的跨模块优化。
  
    - 6. **添加 Profile 诊断工具**
      - 当 Profile 过时或与代码严重不符时，用户往往难以察觉。
      - **贡献点**：在 `llvm-profdata` 或编译器内部添加校验功能，输出警告（如“函数 XXX 的 CFG 与 Profile 不匹配，将忽略其 Profile”），帮助用户优化训练流程。
  
    - 7. **支持更丰富的 Profile 类型**
      - 文章提到未来扩展：值 Profile、间接调用目标 Profile 等。这些在 LLVM 中已有部分实现（如 `vp` 元数据），但尚未被所有优化充分利用。
      - **贡献点**：为 Indirect Call Promotion（ICP）或 Value Profiling 增加新的用途，例如利用值 Profile 指导常量传播或特殊化。
  
    - 8. **性能与内存优化**
      - 大型应用的 Profile 文件可能非常大，合并和加载时占用大量内存。
      - **贡献点**：优化 `llvm-profdata` 的合并算法，减少内存占用；研究更紧凑的 Profile 编码格式（如基于 bitcode 的序列化）。
  
    - 9. **文档与测试**
      - 增加 PGO 相关文档，特别是 CSPGO、采样 PGO 的使用指南和最佳实践。
      - **贡献点**：编写或更新 `llvm/docs/ProfileGuidedOptimization.rst`，补充示例；添加测试用例覆盖不同 Profile 场景（如分支反转、冷热分区）。

#### further reading
- https://llvm.org/devmtg/2020-09/slides/PGO_Instrumentation.pdf
- Patch: https://github.com/kpdev/llvm-project/tree/llvm-dev-mtg/callsite
- PGO Docs: https://clang.llvm.org/docs/UsersManual.html#profile-guided-optimization
- MST: https://llvm.org/pubs/2010-04-NeustifterProfiling.pdf
- Presentations:
  - LLVM Dev Mtg 2013 Presentation: http://llvm.org/devmtg/2013-11/slides/Carruth-PGO.pdf
  - MSVC team talk: https://channel9.msdn.com/Shows/C9-GoingNative/C9GoingNative-12-C-at-BUILD-2012-Inside-Profile-Guided-Optimization
