#### Description
[linker](https://github.com/llvm/llvm-project/pull/192536)

#### Analysis
- 从白盒转向黑河测试
这个 PR #192536 是一次**代码清理和测试迁移**，其核心目的是**将 GlobalISel 的向量已知位（Known Bits）测试，从 C++ 单元测试迁移到更易于维护的 MIR (Machine IR) 文件形式**。

##### 🎯 PR 背景：为何要迁移测试？
这个 PR 并非修复 Bug，而是对测试基础设施的改进，属于非功能性变更（NFC）。在 LLVM 开发中，测试通常有两种形式：
*   **C++ 单元测试**：直接测试 C++ API，优点是测试粒度细，但编写和维护成本高，测试用例较长时会显得笨重。
*   **MIR 测试**：将代码表示为更接近后端的 MIR 格式，通过 `llc` 工具驱动，并使用 `FileCheck` 验证输出。这种方式更轻量，与后端代码的集成更紧密，也更容易通过脚本自动生成和更新。

这个 PR 就是将旧式 C++ 测试迁移到更现代、更易于维护的 MIR 测试体系的一部分工作。

##### 🛠️ 做了哪些修改？
PR 的修改非常直接，主要是文件的新增和删除。

###### 1. 新增了 MIR 测试文件
*   **新增文件**：`llvm/test/CodeGen/AArch64/GlobalISel/knownbits-vector.mir` (+1291 行)。
*   **内容**：包含大量 `G_BUILD_VECTOR`、`G_SHUFFLE_VECTOR` 等向量操作的已知位（Known Bits）和符号位（Sign Bits）传播测试用例。
*   **生成方式**：文件头部注释表明，其中的断言（CHECK 行）是由脚本 `utils/update_givaluetracking_test_checks.py` 自动生成的，这保证了测试与 LLVM 内部实现保持同步。

###### 2. 删除了 C++ 单元测试
*   **删除文件**：`llvm/unittests/CodeGen/GlobalISel/KnownBitsVectorTest.cpp` (-1370 行)。这个文件包含了与新增 MIR 测试对等的 C++ 测试代码，被完整地移除了。

##### 🧠 为什么要做这个迁移？
这个修改的动机，可以从 PR 描述和 LLVM 的整体发展趋势来理解：
*   **提高可维护性**：MIR 测试用例更简洁、专注，不依赖复杂的 C++ 测试框架，当 LLVM 的 API 发生变化时，维护成本更低。
*   **更好的集成性**：MIR 测试直接运行在 LLVM 的后端流水线上，能更真实地反映代码在实际编译过程中的行为。
*   **遵循最佳实践**：LLVM 社区正逐步将 GlobalISel 相关的测试从单元测试转向 MIR 测试，使其与 SelectionDAG 等其他后端组件保持一致。
*   **简化贡献流程**：贡献者在修改 GlobalISel 的已知位计算逻辑时，可以使用 `update_givaluetracking_test_checks.py` 这样的脚本自动更新测试输出，无需手动修改复杂的 C++ 断言。

##### 💎 总结
总的来说，这个 PR 是一次纯粹的质量提升工作（NFC）。它将一组关于向量操作的已知位（Known Bits）测试，从 C++ 单元测试的形式，完整地迁移到了更标准、更易于维护的 MIR 测试格式，紧跟 LLVM 项目的测试现代化趋势。

> `NFC` 是 "Non-Functional Change" 的缩写，指不改变软件功能、只改进代码结构或测试的修改。
