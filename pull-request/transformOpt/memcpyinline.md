#### Description
[linker](https://atomgit.com/openeuler/llvm-project/pull/383/diffs)

#### Analysis
- 最主要的实现：[llvm::createMemCpyAsScalableLoop](https://atomgit.com/openeuler/llvm-project/blob/dev_17.0.6/llvm/lib/Transforms/Utils/LowerMemIntrinsics.cpp#L366)
- Patch 1: 移除 llvm.memcpy.inline 长度参数的常量限制
  - 背景
    - 原 llvm.memcpy.inline 内联函数要求第三个参数（长度）必须是编译时常量
    - 参考上游 PR #98281，原始作者 Alex Bradbury
    - 讨论背景：引入 llvm.memset.pattern.inline 内在函数的 RFC
  - 实现的功能
    1. 移除 ImmArg 约束：从 Intrinsics.td 中移除 llvm.memcpy.inline 第三个参数的 ImmArg<ArgIndex<2>> 属性
    2. 删除类型安全检查：移除 IntrinsicInst.h 中 MemCpyInlineInst::getLength() 强制转换为 ConstantInt* 的方法
    3. 统一 Lint 检查：将 memcpy_inline 和 memcpy 的 Lint 检查合并，不再要求长度为常量
    4. 文档更新：更新 LangRef，说明长度参数现在是普通整数而非常量整数
  ---
- Patch 2: MemCpyOpt - 将 memcpy 调用版本化为 SVE 循环
  - 背景
    - 在 MemCpyOpt pass 中，对 memcpy 进行版本化处理
    - 当 size 小于阈值时，转换为 llvm.memcpy.inline()
    - llvm.memcpy.inline() 在 PreISelIntrinsicLowering 阶段会被降低为 SVE load/store 循环
  - 实现的功能
    ```
     1. MemCpyOpt 中的版本化 (MemCpyOptimizer.cpp)
      - 新增 InlineMemCpyThreshold 选项控制阈值（默认 0 表示禁用）
      - 将一个 memcpy 拆分为控制流：
    原 BB
      ↓ (size ≤ threshold ?)
      ├─→ call.memcpy.inline: 调用 llvm.memcpy.inline
      └─→ call.memcpy.original: 调用原 memcpy
      ↓
    call.memcpy.merge: 合并块
    - 使用 SplitBlock 分割基本块，维护 MemorySSA
    2. PreISelIntrinsicLowering 中的扩展
    - 对非恒定长度的 llvm.memcpy.inline，调用 expandMemCpyAsLoop
    - 恒定长度的保持不变，留给 SelectionDAG 处理
    3. SVE 可扩展向量循环 (LowerMemIntrinsics.cpp)
    ```
  - createMemCpyAsScalableLoop 生成优化的 SVE 循环：
    - 主循环 (mem.exploop): 使用完整向量宽度的 load/store
    - Epilog (mem.epilog): 使用 get.active.lane.mask 进行掩蔽的 load/store 处理剩余字节
    - 使用 <vscale x 16 x i8> 可扩展向量类型

- Patch 1 的边界情况
  1. 长度为 0：需要处理零长度的 memcpy.inline（测试用例中有处理）
  2. PreISelIntrinsicLowering 未完全回传：注意 patch 说明中提到 "Changes in PreISelInstrinsicLowering is not backported"
  3. Verifier 变化：移除了对可变 size 的验证错误
- Patch 2 的边界情况
  1. 长度小于向量宽度 (Length ≤ vscale * 16)：直接走 epilog 路径
  2. 长度恰好是向量宽度的倍数：使用 select 在 epilog 中处理
  3. 内存重叠 (CanOverlap)：代码中通过 alias scope metadata 处理
  4. 基本块起始位置：代码专门处理 IsBBStart 情况，维护迭代器有效性
  5. MemorySSA 维护：需要为新创建的 memcpy.inline 创建 MemoryAccess
  6. 避免重复版本化：使用 VersionedMemCpy set 跟踪已处理的 memcpy
  7. Volatile 访问：同时处理 volatile 和非 volatile 情况
  8. 对齐处理：正确传递和使用 SrcAlign/DestAlign
  9. 64位索引：代码总是零扩展到 64 位迭代计数器
  10. 阈值比较：使用 icmp ule (无符号小于等于) 进行安全比较
