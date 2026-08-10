
> 适用版本：本仓库 LLVM 22.0.0git。面向 LLVM IR 优化、编译器开发、JIT、
> JVM/LLVM 结合方向。本文不是接口字典，而是“面试时怎样回答”的训练材料。

## 使用方法

每道题分为四层：

- **30 秒回答**：先给结论，适合面试第一轮回答。
- **展开回答**：解释原理、正确性条件和工程含义。
- **追问链**：面试官可能逐层深入的问题。
- **易错点**：容易暴露理解不完整的回答。

推荐训练方式：遮住答案，先用 30 秒口述；随后任选两个追问各回答 1～2 分钟；最后用
LLVM IR 或 C++ API 举一个例子。

## 目录

1. 基础概念与 LLVM IR
2. SSA、CFG 与 Dominance
3. Pass Manager 与分析失效
4. 常用分析框架
5. 经典优化 Pass
6. 正确性、UB、Poison 与内存语义
7. LLVM 后端
8. JIT、ORC 与 Jeandle/JVM
9. 看 IR 现场题
10. 手写 Pass 与系统设计题
11. 面试前速背表

---

# 1. 基础概念与 LLVM IR

## Q1：LLVM 是什么？它只是一个编译器后端吗？

**30 秒回答**

LLVM 是一套模块化编译器基础设施。核心包括 LLVM IR、中端优化框架、多目标后端、
目标无关和目标相关分析，以及 ORC JIT、MC、调试信息等组件。Clang 是使用 LLVM 的
C/C++ 前端，但 LLVM 本身不等于 Clang，也不只是后端。

**展开回答**

典型流水线是：源语言前端生成 LLVM IR，中端对 IR 做目标相对无关的分析和优化，后端
完成合法化、指令选择、寄存器分配、指令调度和机器码/目标文件生成。JIT 场景也可以把
Module 交给 ORC，在运行时优化、编译、链接和解析符号。

**追问链**

1. Clang 与 LLVM 的边界是什么？
2. LLVM IR 为什么适合多语言、多目标？
3. MLIR 与 LLVM IR 的定位差异是什么？
4. 一个 JVM JIT 为什么还需要自己维护 Java 语义，而不能只依赖 LLVM？

**易错点**

- 把 LLVM 等同于 Clang。
- 认为 LLVM IR 完全与目标无关；实际上 `DataLayout`、address space、目标 intrinsic、
  TTI 等会携带目标信息。

## Q2：LLVM IR 有哪几种形态？

**30 秒回答**

LLVM IR 有内存中的 C++ 对象形式、文本 `.ll` 和 bitcode `.bc`。三者表达同一个 IR
模型：文本适合阅读和测试，bitcode 紧凑且便于序列化，内存形式供分析和变换使用。

**追问链**

1. `.ll` 和 `.bc` 是否保证跨任意 LLVM 版本双向兼容？
2. `llvm-as`、`llvm-dis`、`opt`、`llc` 分别做什么？
3. bitcode 与最终目标文件有什么区别？

**易错点**

- 把 bitcode 当作机器码。
- 认为文本 IR 的名字具有语义；SSA 名称主要用于引用和可读性。

## Q3：LLVM IR 的核心对象层级是什么？

**30 秒回答**

`LLVMContext` 管理上下文级唯一化数据，`Module` 表示编译单元，包含 `Function` 和
`GlobalValue`；函数包含 `BasicBlock`，基本块包含 `Instruction`。大多数 IR 实体继承
`Value`，指令同时通常也是 `User`，operand 关系由 `Use` 表示。

**追问链**

1. `Value`、`User`、`Use` 的区别？
2. `users()` 和 `uses()` 有何不同？
3. LLVM 裸指针通常是否表示所有权？
4. `removeFromParent()` 与 `eraseFromParent()` 的区别？

**优秀回答加分点**

`uses()` 能定位同一 User 中具体的 operand，PHI incoming edge 和 dominance 查询必须
经常精确到 `Use`。

## Q4：LLVM IR 为什么采用强类型？

**30 秒回答**

类型使 verifier、优化器和后端能够检查操作是否合法，并精确描述位宽、向量、聚合、
函数签名和 address space。但整数类型本身没有 signed/unsigned 属性，有符号语义由
`sdiv`、`icmp slt`、`sext` 等具体操作决定。

**追问链**

1. `i32` 如何表达有符号和无符号数？
2. `bitcast`、`ptrtoint`、`addrspacecast` 的区别？
3. opaque pointer 解决了什么问题？
4. 为什么 load/GEP 现在需要显式元素类型？

## Q5：什么是 opaque pointer？

**30 秒回答**

现代 LLVM 的指针类型通常只表示 `ptr` 和 address space，不再携带 pointee type。被
访问的数据类型由 load、store、GEP、call 等操作自己显式表达，减少无意义 pointer
bitcast，并避免把内存类型错误地当作指针自身属性。

**追问链**

1. 如何获得 load 的数据类型？——`LoadInst::getType()`。
2. 如何获得 store 的数据类型？——stored value 的类型。
3. 如何知道 GEP 按什么元素布局计算？——GEP 的 source element type。
4. address space 是否仍是指针类型的一部分？——是。

## Q6：GEP 是什么？它会访问内存吗？

**30 秒回答**

GEP 根据类型布局和索引计算派生地址，本身不读取也不写入内存。它不是简单的整数加法；
索引会按聚合布局和元素大小缩放。`inbounds` 是额外的语义承诺，不只是优化提示。

**展开回答**

GEP 结果仍是 pointer。第一个索引通常沿指针指向对象序列移动，后续索引进入数组、向量
或结构体。结构体索引必须是常量。目标大小和字段偏移来自 `DataLayout`。

**追问链**

1. GEP 能否改变 address space？——不能，通常需 `addrspacecast`。
2. 两个不同 GEP 是否可能 alias？——可能，应查询 AA。
3. `inbounds` 不成立有什么后果？——结果可能成为 poison，并影响后续语义。
4. 为什么不能用宿主机 `sizeof` 计算偏移？——编译目标可能不同。

## Q7：`DataLayout` 有什么作用？

**30 秒回答**

`DataLayout` 描述目标的指针位宽、类型大小和对齐、结构体布局、整数合法宽度等 ABI
布局信息。优化和代码生成必须依据目标 Module 的 DataLayout，而不是编译器宿主机器。

**追问链**

1. store size 与 alloc size 有什么差异？
2. ABI alignment 与 preferred alignment 的区别？
3. `TypeSize` 为什么可能是 scalable？
4. JIT 场景下 DataLayout 从哪里来？

## Q8：LLVM 的常量一定没有地址吗？

**30 秒回答**

不一定。`Constant` 表示编译期不可变的 IR 值类别，包括整数常量、聚合常量、全局值、
函数地址和常量表达式。`GlobalVariable` 是 `Constant` 派生体系中的全局地址，但它指向
的内存内容未必不可变，是否常量由 global 的属性决定。

**追问链**

1. `ConstantInt` 如何表示超过 64 位的整数？——`APInt`。
2. `ConstantExpr` 为什么常需要展开成 Instruction？
3. `GlobalVariable` 本身和它保存的 initializer 是什么关系？

## Q9：`CallInst` 能否代表所有调用？

**30 秒回答**

不能。通用逻辑应优先使用 `CallBase`，它覆盖 `CallInst`、`InvokeInst` 和
`CallBrInst`。`getCalledFunction()` 对间接调用返回 null，调用目标也可能经过 pointer
cast。

**追问链**

1. `call` 与 `invoke` 的差别？——invoke 显式拥有 normal 和 unwind successor。
2. tail call 与 musttail 有什么差异？
3. 如何判断调用是否访问内存？
4. 函数 attribute 和 call-site attribute 冲突时怎么办？

## Q10：Intrinsic 与普通函数调用有什么区别？

**30 秒回答**

Intrinsic 是 LLVM 认识的内建操作，以 `llvm.*` 名称和 intrinsic ID 标识，优化器与
后端可以理解其特殊语义。部分 intrinsic 按类型重载。不能把任意 `llvm.*` 名称当作
普通外部函数。

**追问链**

1. intrinsic 与 target intrinsic 的区别？
2. `llvm.assume`、`llvm.trap`、lifetime intrinsic 的用途？
3. intrinsic 是否一定会生成机器指令？——不一定，可能优化掉、展开或影响分析。

---

# 2. SSA、CFG 与 Dominance

## Q11：什么是 SSA？为什么有利于优化？

**30 秒回答**

SSA 要求每个虚拟值只有一个定义，每个使用直接指向该定义；控制流汇合处通过 PHI
合并不同路径的值。它让 def-use 链显式化，使常量传播、死代码消除、值编号和数据流
分析更简单、更稀疏。

**展开回答**

传统数据流分析经常在每个程序点维护变量状态；SSA 将多数信息编码到值和 use-def 图中。
但内存通常仍不是普通 SSA 值，需要 MemorySSA 等辅助表示。后端在寄存器分配前后会通过
并行复制等方式消除 PHI。

**追问链**

1. SSA 是否意味着变量只能赋值一次？——IR value 单定义，不是源语言变量不能更新。
2. memory 是否天然是 SSA？——不是。
3. PHI 如何消除？——在前驱边插入并行复制，必要时拆 critical edge。
4. SSA 如何构造？——dominance frontier 插 PHI，随后按 dominator tree rename。

## Q12：PHI 的语义是什么？

**30 秒回答**

PHI 根据实际到达该块的 incoming edge 选择一个值。PHI 必须连续位于基本块开头，incoming
项要与 CFG 前驱边一致。PHI operand 的 use 发生在对应前驱边末端，而不是 PHI 指令的
文本位置。

**追问链**

1. 为什么 `DT.dominates(Def, Phi)` 可能不是正确查询？
2. 一个前驱块能否在 PHI 中出现多次？——可能有多条相同源/目标 CFG edge。
3. PHI 是并行执行还是顺序执行？——语义上并行选择。
4. predecessor 被替换后必须更新什么？——终结指令、目标 PHI 以及相关分析。

**易错点**

把 PHI 理解成普通的运行时条件表达式，或认为其 operand 都在 PHI 所在块求值。

## Q13：什么是基本块？

**30 秒回答**

基本块是单入口、控制流只从末尾退出的最大直线指令序列。LLVM 基本块通常以 terminator
结束，如 `br`、`switch`、`ret`、`invoke` 或 `unreachable`；PHI 必须放在块首。

**追问链**

1. 一个 block 能否没有 terminator？——构造过程中可以，合法完整 IR 不可以。
2. `landingpad` 对 insertion point 有何影响？
3. 块的文本顺序是否表示执行顺序？——不表示，CFG edge 才表示控制流。

## Q14：什么是 critical edge？为什么要拆？

**30 秒回答**

如果一条边的源块有多个 successor，目标块有多个 predecessor，它就是 critical edge。
无法在不影响其他路径的情况下把仅属于这条边的指令放到源或目标块，因此许多 PHI 消除、
profile instrumentation 和 CFG 变换要先拆边。

**追问链**

1. 拆边后 CFG 变成什么样？——中间增加单前驱单后继块。
2. 哪些信息必须更新？——PHI、DT、LI、MSSA、branch weights 等。
3. EH edge 是否能按普通 edge 拆？——通常有额外结构和 funclet 约束。

## Q15：什么是 dominance？

**30 秒回答**

A 支配 B 表示从函数入口到 B 的所有路径都经过 A。A 严格支配 B 还要求 A 不等于 B。
B 的 immediate dominator 是离 B 最近的严格支配者，所有 idom 关系组成 dominator tree。

**追问链**

1. `dominates(A, A)` 是否为真？——是。
2. 同一块两条指令如何判断支配？——看顺序。
3. PHI use 如何判断？——`DT.dominates(Def, Use)`。
4. unreachable block 有什么特殊性？——先明确可达性，语义可能不符合直觉。
5. 如何找最近公共支配者？——`findNearestCommonDominator`。

## Q16：什么是 post-dominance？

**30 秒回答**

A 后支配 B 表示从 B 到函数退出的所有路径都会经过 A。它常用于控制依赖、判断 cleanup
是否必经以及统一退出相关变换。多个退出、无限循环和不可达区域会让后支配树具有虚拟根
等特殊情况。

**追问链**

1. 为什么参数顺序看起来容易反？——`PDT.dominates(A, B)` 表示 A 后支配 B。
2. 无限循环里的块由哪个实际 exit 后支配？——未必存在。
3. control dependence 与 post-dominance 有什么关系？

## Q17：DominatorTree 和 CFG 是同一棵树吗？

**30 秒回答**

不是。CFG 是可能含环的有向图，边表示可能的控制转移；DominatorTree 是由支配关系
抽取出的树，父节点是 immediate dominator。DT edge 不等于 CFG edge。

**追问链**

1. DT 中父子块在 CFG 中是否必须直接相连？——不必须。
2. DT 遍历适合做什么？——按支配作用域维护事实、SSA rename 等。
3. CFG 修改后 DT 会自动更新吗？——不会，需 updater、工具函数或分析失效。

## Q18：什么是 dominance frontier？

**30 秒回答**

节点 X 的 dominance frontier 是这样一组节点 Y：X 支配 Y 的至少一个前驱，但不严格
支配 Y。它描述 X 的支配影响在控制流汇合处停止的位置，是经典 SSA 构造中放置 PHI 的
关键结构。

**追问链**

1. iterated dominance frontier 是什么？
2. 为什么只在定义块的 dominance frontier 放 PHI 还可能不够？
3. pruned SSA 如何避免无用 PHI？——结合变量活跃性。

## Q19：自然循环如何定义？

**30 秒回答**

如果 CFG edge `Latch -> Header` 中 Header 支配 Latch，该边是 backedge，并可形成以
Header 为入口的自然循环。LLVM `LoopInfo` 表示的是具有单入口 header 的循环层级，
irreducible CFG 需要特殊处理。

**追问链**

1. header、latch、preheader、exiting block、exit block 分别是什么？
2. loop 能否有多个 latch 或多个 exit？——可以。
3. irreducible loop 是什么？——循环区域存在多个入口，不能由单一 header 概括。

## Q20：LoopSimplify form 和 LCSSA form 是什么？

**30 秒回答**

LoopSimplify 通常提供唯一 preheader、规范化 latch/backedge 和 dedicated exits；LCSSA
要求循环内定义、循环外使用的值先通过 exit block PHI 暴露。二者让循环变换更局部、
更容易维护正确性。

**追问链**

1. 为什么 loop pass 常要求 preheader？——放置只执行一次的 hoisted 指令。
2. LCSSA 为什么方便 loop deletion/unroll？
3. 有 `LoopInfo` 是否就保证两种 form？——不保证。

---

# 3. Pass Manager 与分析失效

## Q21：LLVM Pass 有哪些 IR 层级？

**30 秒回答**

新 Pass Manager 常见层级包括 Module、CGSCC、Function 和 Loop。pass 的 `run` 接收
对应 IR unit 和 AnalysisManager，返回 `PreservedAnalyses`。不同层级通过 adaptor 和
proxy 组合、传递分析失效。

**追问链**

1. Function pass 如何加入 Module pipeline？——module-to-function adaptor。
2. 为什么 Inliner 常位于 CGSCC 层？——调用图强连通分量及递归处理。
3. Loop pass 为什么不能随意访问整个函数并修改其他循环？

## Q22：新旧 Pass Manager 有什么主要差异？

**30 秒回答**

新 PM 将 pass 与 analysis 组织为类型化的 manager，分析按需计算、缓存并通过明确的
`PreservedAnalyses` 失效；pipeline 组合与跨层 proxy 更系统。旧 PM 依赖继承体系和
`getAnalysisUsage`。LLVM 现代 IR pipeline 主要使用新 PM。

**追问链**

1. 为什么新 PM 更适合新的 inliner？
2. Analysis result 为什么可能有自己的 `invalidate`？
3. MachineFunction pass manager 的迁移状态是否与 IR PM 完全相同？

## Q23：`PreservedAnalyses` 为什么重要？

**30 秒回答**

它是 pass 对框架的正确性承诺：哪些缓存分析在变换后仍有效。保留过少主要损失性能，
保留过多会让后续 pass 使用陈旧结果，可能导致断言、崩溃或 miscompile。

**追问链**

1. 未修改 IR 返回什么？——`all()`。
2. 不确定时返回什么？——`none()`，优先正确。
3. 未改 CFG 是否能保留全部分析？——不能，SCEV、AA 等仍可能变化。
4. `CFGAnalyses` 表示什么？——一组只依赖 CFG 形状的分析保留集合。

## Q24：分析结果是如何获取和缓存的？

**30 秒回答**

通过对应 AnalysisManager 的 `getResult<Analysis>(IRUnit)` 获取。若缓存不存在就运行分析，
存在且未失效则复用。变换完成后 manager 根据 `PreservedAnalyses` 和 result 的
`invalidate` 逻辑清理缓存。

**追问链**

1. `getCachedResult` 与 `getResult` 的差别？
2. 为什么 Module pass 不能任意强制计算 Function analysis？
3. proxy 的作用是什么？——跨 IR 层访问 manager 并传播失效。

## Q25：写一个最小 Function Pass 需要什么？

**30 秒回答**

继承 `PassInfoMixin<MyPass>`，实现
`PreservedAnalyses run(Function &, FunctionAnalysisManager &)`；从 FAM 取分析，完成
变换，按修改范围返回 preserved analyses，并在 pipeline 中注册和加入 pass。

```cpp
class MyPass : public PassInfoMixin<MyPass> {
public:
  PreservedAnalyses run(Function &F, FunctionAnalysisManager &FAM) {
    DominatorTree &DT = FAM.getResult<DominatorTreeAnalysis>(F);
    bool Changed = transform(F, DT);
    return Changed ? PreservedAnalyses::none()
                   : PreservedAnalyses::all();
  }
};
```

**追问链**

1. 如何注册给 `opt -passes=`？
2. static plugin 和 pass plugin 有什么区别？
3. 为什么 pass 中保存跨函数裸指针危险？

## Q26：CFG 修改后怎样维护分析？

**30 秒回答**

有两种正确策略：使用 `BasicBlockUtils`、`DomTreeUpdater`、`MemorySSAUpdater` 等进行
增量维护，并精确 preserve；或者不保留相关分析，让框架下次重算。无论哪种策略都必须
同步目标块 PHI。

**追问链**

1. `DomTreeUpdater` eager/lazy 有何区别？
2. 什么时候重算可能比增量维护更划算？
3. 为什么只更新 DT 不够？——LI、MSSA、BPI/BFI 等也可能失效。
4. 如何验证维护正确？——verifier、DT/MSSA verification、`-verify-each`。

---

# 4. 常用分析框架

## Q27：AliasAnalysis 回答什么问题？

**30 秒回答**

AA 判断两个 `MemoryLocation` 是否可能引用重叠内存，结果通常是 NoAlias、MayAlias、
PartialAlias 或 MustAlias；还可查询 call 对位置的 Mod/Ref。它是保守分析，MayAlias
表示无法证明不重叠，不表示确定重叠。

**追问链**

1. 为什么不能只比较 pointer Value 是否相等？
2. `noalias` attribute 与 AA 有什么关系？
3. type-based AA 有什么风险？——依赖前端给出的语言 alias 规则 metadata。
4. `MemoryLocation` 为什么包含 size？

## Q28：MemorySSA 是什么？

**30 秒回答**

MemorySSA 为内存状态构造 SSA 形式：写操作对应 `MemoryDef`，读操作对应
`MemoryUse`，控制流汇合处是 `MemoryPhi`，入口是 `liveOnEntry`。它配合 AA 和 walker
高效查找某次内存访问可能被哪个定义 clobber。

**追问链**

1. MemoryUse 的 defining access 一定是真正写它的 store 吗？——不一定，需要 walker。
2. optimized use 与 defining access 的差别？
3. MemorySSA 是否精确表示每一个内存对象？——不是，通常是单一内存版本空间加 AA。
4. 修改 store/CFG 后怎么办？——MemorySSAUpdater 或失效。

## Q29：MemorySSA 和 AliasAnalysis 如何配合？

**30 秒回答**

MemorySSA 提供内存定义的 SSA/CFG 搜索结构，AA 用来判断候选定义是否真正可能影响当前
位置。前者减少需要检查的访问，后者过滤不相交的位置；二者解决的问题不同。

**追问链**

1. AA 回答 NoAlias 后 walker 能跳过什么？
2. 为什么 MemorySSA 不能替代语言级内存模型判断？
3. DSE 和 GVN 怎样利用 MemorySSA？

## Q30：ScalarEvolution 解决什么问题？

**30 秒回答**

SCEV 把整数和指针演化抽象成符号表达式，尤其用 AddRec 表示循环归纳关系，可推导
backedge taken count、trip count、范围、单调性和循环不变量，服务于 loop transform、
vectorization、bounds check elimination 等。

**追问链**

1. `{Start,+,Step}<L>` 表示什么？
2. backedge taken count 与 trip count 差多少？——通常 trip count 是前者加一，但要
   考虑无法执行、溢出和 exit 语义。
3. `SCEVCouldNotCompute` 怎么处理？——保守退出，不能猜测。
4. nowrap flags 对 SCEV 为什么重要？

## Q31：LoopInfo 能否告诉你循环执行次数？

**30 秒回答**

不能。LoopInfo 描述循环结构和嵌套关系，如 header、blocks、preheader、latch、exit；
执行次数通常由 ScalarEvolution 或 profile 分析。结构信息与数值演化信息应分开理解。

**追问链**

1. `getLoopFor(BB)` 返回最外层还是最内层？——最内层。
2. 一个块如何得到 loop depth？
3. LoopInfo 如何依赖 DominatorTree 构造自然循环？

## Q32：ValueTracking 与 LazyValueInfo 有什么区别？

**30 秒回答**

ValueTracking 是一组按需、局部递归的值性质查询，如 KnownBits、non-zero、底层对象和
可投机执行；LVI 更侧重某值在特定 block/edge/program point 上的常量或范围，并利用
控制流条件。二者都可能结合 DT、AC、DL 等上下文。

**追问链**

1. 为什么查询要传 context instruction？
2. `KnownBits` 的 Zero/One 是否覆盖未知 bit？——不覆盖，未知位两边都是 0。
3. LVI 与 SCCP 有什么关系和区别？

## Q33：BPI 与 BFI 有什么区别？

**30 秒回答**

BPI 给 CFG edge 的相对分支概率；BFI 综合 CFG、BPI 和 profile，估计基本块相对执行
频率。frequency 不一定是真实次数，只有可用的 profile count 才有计数语义。

**追问链**

1. switch 到同一目标的多个 edge 如何取概率？——按 successor index，不只按 block。
2. 没有 profile 时信息来自哪里？——静态 heuristic、loop 结构等。
3. 热路径信息如何影响 inlining、布局和 vectorization？

## Q34：TLI 和 TTI 分别是什么？

**30 秒回答**

TLI 描述目标运行库函数的可用性和语义，例如某 libc 函数是否存在；TTI 提供目标机器
相关的成本与能力查询，例如指令吞吐成本、向量宽度和内存操作代价。优化决策应查询它们，
而不是写死某个架构假设。

**追问链**

1. TTI cost 是否永远是普通整数？——可能 invalid/scalable，要按 `InstructionCost` 处理。
2. 一个合法变换为什么仍可能因 TTI 成本而不做？——正确性与收益是两层问题。

## Q35：AssumptionCache 有什么用？

**30 秒回答**

它索引函数中的 `llvm.assume`，让 ValueTracking、SCEV 等分析快速利用在相应位置有效的
假设。新插入或删除 assume 后需更新缓存或使分析失效。

**追问链**

1. assume 为 false 有什么语义？
2. assume 能否无条件跨控制流使用？——要满足 dominance 和上下文条件。
3. 错误的 assume 为什么会 miscompile？——它是可被优化器信赖的事实。

---

# 5. 经典优化 Pass

## Q36：mem2reg 做什么？

**30 秒回答**

mem2reg 将满足条件的 entry-block scalar alloca 上的 load/store 提升为 SSA value：在
必要的 dominance frontier 插 PHI，然后沿 dominator tree rename 并删除内存操作。

**追问链**

1. 所有 alloca 都能提升吗？——不能，地址逃逸、复杂访问、非直接 load/store 等会阻止。
2. 为什么前端常先生成 alloca？——生成简单，随后统一提升到 SSA。
3. mem2reg 与 SROA 的区别？——SROA 先拆 aggregate/切片，也能促进后续提升。
4. pruned SSA 在这里如何减少 PHI？

## Q37：InstCombine 是代数优化 Pass 吗？

**30 秒回答**

它既做局部代数化简，也把 IR 规范化为其他 pass 更容易识别的 canonical form。它通常
结合 use 信息、KnownBits、DL 等，但不应被理解成完整全局优化器。很多变换可能暂时不
减少指令数，却改善后续优化机会。

**追问链**

1. InstSimplify 与 InstCombine 的区别？——前者只返回已有值/常量，后者可创建新 IR。
2. 为什么 canonicalization 要选择固定方向？——减少等价形式，避免 pass 循环打架。
3. 如何防止 combine 无限振荡？

## Q38：CSE、EarlyCSE 和 GVN 有什么关系？

**30 秒回答**

CSE 消除重复表达式。EarlyCSE 更轻量，常利用 dominator tree 和 scoped value table 做
早期局部/支配域消除；GVN 通过 value numbering 识别更广泛的等价值，也需要处理内存
依赖、PHI 翻译等复杂情况。

**追问链**

1. `a+b` 与 `b+a` 是否自动同值？——需要 canonicalization 或表达式处理。
2. 两个相同 load 何时能 CSE？——中间没有可能 clobber 的操作，并满足原子/volatile 等语义。
3. 为什么 dominance 是替换的必要条件？

## Q39：LICM 做什么？正确性条件是什么？

**30 秒回答**

LICM 将循环不变量移到 preheader，或把可下沉操作移到 exit。除了 operands invariant，
还要保证执行安全：不能新增 trap/UB/异常，内存操作要通过 AA/MemorySSA 证明不被循环中
clobber，并考虑 must-execute、atomic、volatile 和控制流。

**追问链**

1. `x/y` operands 都 invariant 就一定能 hoist 吗？——不一定，可能除零。
2. load 如何证明可 hoist？
3. may-throw call 为什么不能随意移动？
4. LCSSA 为什么方便 sink？

## Q40：SCCP 的核心思想是什么？

**30 秒回答**

Sparse Conditional Constant Propagation 同时在 SSA 值格和 CFG 可执行边上求不动点。
典型 lattice 是 unknown/undef、constant、overdefined；只沿可执行 edge 传播，因此比普通
常量传播还能删除不可达分支。

**追问链**

1. 为什么叫 sparse？——直接沿 SSA def-use，而不是每个程序点维护所有变量。
2. PHI 如何只合并可执行 incoming edge？
3. lattice 为什么需要 overdefined？
4. 算法何时达到不动点？

## Q41：DCE 与 ADCE 有什么区别？

**30 秒回答**

DCE 从已知无 use 且无副作用的指令出发删除；ADCE 更激进，通常先假定大部分代码死亡，
从有可观察行为的根反向标记活跃，还能利用控制依赖删除无用控制流。

**追问链**

1. 无 use 的 store 能否直接删除？——不能，可能有可观察内存效果。
2. 无 use 的 volatile load 能否删除？——不能。
3. 控制依赖为什么需要 post-dominance 信息？

## Q42：Dead Store Elimination 如何判断 store 已死？

**30 秒回答**

若某次 store 写入的内存在任何可能读取前必然被后续写完全覆盖，且删除不改变 volatile、
atomic、异常或可观察行为，则它可能是 dead store。现代实现常结合 MemorySSA、AA、
对象边界和写入区间分析。

**追问链**

1. 两个 store 地址相同但 size 不同怎么办？
2. 中间存在 unknown call 怎么办？——查询 Mod/Ref，无法证明则保守。
3. store 到新分配且从未逃逸对象是否可删？

## Q43：Inlining 为什么不只是复制函数体？

**30 秒回答**

内联需要克隆 CFG、映射参数和返回值、处理 PHI、异常边、属性、调试信息、调用图更新，
还要用 cost model 平衡后续优化收益、代码尺寸、热度和编译时间。递归和间接调用也增加
复杂性。

**追问链**

1. 为什么内联能促进常量传播和去虚拟化？
2. alwaysinline 是否绝对保证成功？——仍受结构合法性等硬约束。
3. 何为 inline threshold？
4. 为什么过度内联会降低性能？——I-cache、代码尺寸、编译时间、寄存器压力。

## Q44：Loop unroll 的收益与代价是什么？

**30 秒回答**

展开减少分支和归纳更新开销，暴露跨迭代优化与向量化机会；代价是代码膨胀、I-cache
压力、编译时间和寄存器压力。完全展开通常要求已知小 trip count，部分展开由成本模型
选择 factor，并处理 remainder。

**追问链**

1. runtime unroll 如何处理未知余数？
2. unroll-and-jam 是什么？
3. 为什么展开后要重新简化和维护 LCSSA？

## Q45：Loop vectorizer 和 SLP vectorizer 的区别？

**30 秒回答**

Loop vectorizer 把不同迭代打包为向量，关注循环依赖、trip count、remainder 和 runtime
alias check；SLP 在基本块或区域内把多条独立、同构的 scalar 操作打包，常从 stores 或
归约等 seed 构建向量树。

**追问链**

1. vectorization legality 与 profitability 的区别？
2. runtime alias check 解决什么问题？
3. reduction 如何向量化？
4. scalable vector 与 fixed vector 的差别？

## Q46：SimplifyCFG 常做什么？

**30 秒回答**

它折叠常量分支、合并简单块、转发空块、简化 PHI/switch、合并返回等，使 CFG 更规范。
但每次 CFG 简化都可能影响 DT、LI、profile metadata 和代码布局，不能只看语义等价。

**追问链**

1. 为什么两个相邻块不一定能合并？——多前驱、多后继、PHI、地址被取、EH 等限制。
2. 分支转 select 总是更快吗？——不是，要看 speculation safety 和目标成本。

---

# 6. 正确性、UB、Poison 与内存语义

## Q47：`undef` 和 `poison` 有什么区别？

**30 秒回答**

`undef` 的每次 use 可以选择任意允许的值；`poison` 是由非法优化承诺或某些操作产生的
延迟错误值，会传播，并在进入分支条件、内存地址等敏感位置时导致 UB。现代优化更强调
poison 语义。

**追问链**

1. `add nsw` 溢出产生什么？——poison。
2. poison 是否产生后立刻 UB？——通常不是，会先传播。
3. 为什么 `select` 对 poison 的传播有特殊性？——未选择 operand 的 poison 不必传播。
4. 什么情况下需要 `freeze`？

## Q48：`freeze` 做什么？

**30 秒回答**

`freeze` 把 undef 或 poison 变成某个任意但对该 freeze 结果稳定的普通值；对已有正常值
保持原值。它允许优化器在不把 poison 扩散成 UB 的情况下固定一次非确定选择。

**追问链**

1. 两个独立 freeze 是否必须选择相同值？——不必须。
2. freeze 是否是运行时随机数？——不是，这是语义模型，不要求随机实现。
3. 为什么循环变换和 select-to-branch 可能需要考虑 freeze？

## Q49：`nsw`、`nuw`、`exact` 是优化提示吗？

**30 秒回答**

它们是语义 flags。`nsw`/`nuw` 承诺有符号/无符号意义下不溢出，`exact` 承诺除法或移位
没有被丢弃的非零部分。承诺违反通常产生 poison，因此优化器可据此做更激进推理。

**追问链**

1. 为什么不能为了性能随意加 flag？——可能 miscompile。
2. 变换 `add` 时 flags 能否原样复制？——必须重新证明新操作也满足。
3. SCEV 如何利用 nowrap 信息？

## Q50：为什么 `isSafeToSpeculativelyExecute` 很重要？

**30 秒回答**

把只在某分支执行的指令提前，可能让原本不执行的 trap、UB、异常或内存访问发生。
该查询综合 opcode、operand、attribute 和上下文，判断投机执行是否安全；但对读取内存的
指令，即使不 trap，移动后取值也可能改变，还需依赖分析。

**追问链**

1. 整数除法为何通常不能无条件 speculate？
2. load dereferenceable 是否就能随意 hoist？——还要保证值未被 clobber。
3. convergent call 为什么限制代码移动？

## Q51：volatile 与 atomic 有什么区别？

**30 秒回答**

volatile 表示该内存访问本身具有必须保留的可观察行为，常用于设备或特殊内存；atomic
描述多线程同步、原子性和 memory ordering。volatile 不自动提供线程同步，atomic 也不
等于 volatile。

**追问链**

1. 两次 volatile load 能否 CSE？——通常不能。
2. unordered、monotonic、acquire、release、acq_rel、seq_cst 有何层级？
3. atomic load/store 的对齐和目标合法性如何处理？

## Q52：LLVM IR 的 UB 与源语言 UB 相同吗？

**30 秒回答**

不完全相同。前端把源语言语义编码成 LLVM IR 的操作、flags、attribute 和 metadata；
进入 IR 后优化器依据 LLVM LangRef 的 UB/poison 规则。错误编码会让原本合法源程序在
IR 层获得过强假设并被错误优化。

**追问链**

1. Java 没有 C 式 signed overflow UB，应如何 lower？——不能随意加 `nsw`。
2. null、越界、异常语义由谁保留？——前端/JIT 必须显式检查、控制流或 runtime call。
3. LLVM 是否理解 Java class loading、deoptimization？——默认不理解。

## Q53：attribute 和 metadata 的区别？

**30 秒回答**

attribute 通常附着于函数、返回值、参数或 call site，表达优化器可依赖的语义契约；
metadata 附着范围更广，既有必须正确维护的语义信息，也有 profile/debug 等提示信息。
不能一概认为 metadata 都可随意丢弃或复制。

**追问链**

1. 错误 `nonnull`、`noundef`、`noalias` 有什么后果？——可能 miscompile。
2. TBAA metadata 如何影响 alias？
3. branch weights 在 CFG 重写后怎样维护？
4. debug metadata 是否影响程序运行语义？——通常不影响，但影响调试质量。

## Q54：RAUW 后为什么还不能立即认为变换正确？

**30 秒回答**

`replaceAllUsesWith` 只做同类型 use 替换，不验证新定义是否支配每个 use，也不自动维护
分析、debug info、metadata、CFG 或内存语义。RAUW 后还需验证 dominance、生命周期、
poison/attribute 和删除旧指令的条件。

**追问链**

1. RAUW 是否删除旧 Value？——不删除。
2. PHI use 应怎样验证？——逐 `Use` 查询 dominance。
3. 修改 use-list 时怎样避免迭代器失效？——early increment 或先收集。

---

# 7. LLVM 后端

## Q55：LLVM 从 IR 到机器码的大致流程是什么？

**30 秒回答**

IR 优化后进入后端，依次经历目标/类型合法化、指令选择，形成 Machine IR；然后做机器级
SSA 优化、寄存器分配、PHI 消除、prologue/epilogue、指令调度、分支和布局优化，最后
降到 MCInst 并编码成目标文件或内存中的机器码。

**追问链**

1. SelectionDAG 与 GlobalISel 分别位于哪里？
2. MachineInstr 是否仍是 LLVM IR Instruction？——不是，是后端 MIR 对象。
3. MC 层是否仍有虚拟寄存器？——通常已经是可编码的目标表达。

## Q56：SelectionDAG 与 GlobalISel 有什么区别？

**30 秒回答**

SelectionDAG 将一个基本块的操作表示为含数据和 chain/glue 依赖的 DAG，经过 legalization、
DAG combine 和 pattern selection。GlobalISel 使用较通用的 MachineInstr 流水线，典型
阶段是 IRTranslator、Legalizer、RegBankSelect 和 InstructionSelect，更易扩展全局机器
表示和复用基础设施。

**追问链**

1. DAG 中 chain 解决什么问题？——表达内存/副作用顺序。
2. GlobalISel 为什么需要 register bank selection？
3. 两条流水线是否所有目标支持程度相同？——不一定。

## Q57：虚拟寄存器与物理寄存器有什么区别？

**30 秒回答**

虚拟寄存器数量理论上不受硬件限制，便于保持 machine SSA；物理寄存器是目标真实资源，
有 register class、alias、subregister、calling convention 等约束。寄存器分配将虚拟
live ranges 映射到物理寄存器，必要时 spill。

**追问链**

1. register class 与 register bank 有何区别？
2. 两个物理寄存器为何可能 alias？——例如不同宽度子寄存器。
3. machine SSA 在什么时候被破坏/消除？

## Q58：寄存器分配为什么困难？

**30 秒回答**

本质上要给相互干涉的 live range 分配有限物理寄存器，同时满足 register class、固定
寄存器、调用约定和 coalescing 约束；一般图着色形式是 NP-hard。工程实现使用 greedy、
PBQP 或启发式，并在压力过大时 spill、split 或 rematerialize。

**追问链**

1. 什么是 live interval？
2. spill、live-range splitting 和 rematerialization 的区别？
3. coalescing 有什么收益和风险？
4. caller-saved 与 callee-saved 如何影响分配？

## Q59：指令调度考虑什么？

**30 秒回答**

调度在保持数据、内存和控制依赖的前提下，考虑指令 latency、处理器资源、吞吐、寄存器
压力和关键路径安排顺序。pre-RA 与 post-RA 调度的自由度和成本不同。

**追问链**

1. dependency DAG 有哪几类边？
2. 为什么降低 latency 可能增加寄存器压力？
3. scheduler model 从哪里来？——目标描述和调度模型。

## Q60：TableGen 是什么？

**30 秒回答**

TableGen 是 LLVM 的声明式记录语言和代码生成工具。后端用 `.td` 描述寄存器、指令、
operand、calling convention、pattern、subtarget feature 等，再生成大量匹配表和 C++
辅助代码。它不是通用编译器前端。

**追问链**

1. `def`、`class`、`multiclass`、`defm` 大致是什么？
2. pattern 为什么可能匹配失败？——类型、predicate、legalization、复杂 pattern 等。
3. 新增一条指令只改 `.td` 就够吗？——未必，还可能需要 lowering、encoding、asm parser 等。

## Q61：calling convention 在 LLVM 中体现在哪些地方？

**30 秒回答**

IR Function/CallBase 有 calling convention 和参数 attribute；后端按目标 ABI 决定参数与
返回值放在寄存器还是栈、栈对齐、caller/callee-saved、varargs 和特殊返回规则，并生成
call lowering 与 prologue/epilogue。

**追问链**

1. caller 和 callee 的 calling convention 不一致会怎样？
2. tail call 为什么受 ABI 约束？
3. JIT 调用生成代码时为什么必须匹配宿主 ABI？

---

# 8. JIT、ORC 与 Jeandle/JVM

## Q62：JIT 与 AOT 的主要差异是什么？

**30 秒回答**

AOT 在运行前完成编译，信息稳定、可投入较多编译时间；JIT 在运行时编译，可利用实际
类型、热点和 profile，但受启动延迟、编译线程、code cache、并发、符号生命周期和
deoptimization 等约束。

**追问链**

1. tiered compilation 为什么有用？
2. JIT 如何平衡编译时间和代码质量？
3. speculative optimization 失败后如何恢复？——guard、deopt 或 fallback。

## Q63：ORC JIT 的核心对象是什么？

**30 秒回答**

高层常用 `LLJIT`；底层核心包括 `ExecutionSession`、`JITDylib`、symbol string pool、
MaterializationUnit/Responsibility，以及 IR compile、object linking 等 layer。它支持按需
物化、并发编译、符号搜索和资源跟踪。

**追问链**

1. JITDylib 是否等同于操作系统动态库？——语义类似命名空间，但不是同一对象。
2. materialization 是什么？——将符号责任转化为已解析/已发射定义的过程。
3. ResourceTracker 用于什么？——成组管理和移除 JIT 资源。
4. `ThreadSafeModule` 为什么需要独立 context 管理？

## Q64：JIT 如何解析运行时符号？

**30 秒回答**

ORC 在 `ExecutionSession`/`JITDylib` 搜索顺序中查找 symbol，可通过 definition generator
暴露当前进程或动态库符号，也可显式 define absolute symbol。符号名称需考虑目标 mangling，
生命周期和并发解析也必须受控。

**追问链**

1. 为什么源码函数名不一定等于链接符号名？
2. JIT 代码怎样调用 JVM runtime stub？
3. 卸载代码前为什么要保证没有执行线程和悬空入口？

## Q65：LLVM 为什么不会自动保证 Java 语义？

**30 秒回答**

LLVM 只理解 IR 明确表达的语义，不理解 Java class loading、精确异常、GC、安全点、
deoptimization、对象移动、内存屏障或 signed overflow 规则。JIT 前端必须通过控制流、
runtime call、statepoint、barrier、attribute 和 metadata 正确编码这些约束。

**追问链**

1. Java 整数溢出为什么通常不能加 `nsw`？
2. null check 能否无条件删除？——需要语言语义和 dominance/implicit null check 证明。
3. GC 移动对象后普通 LLVM pointer 为什么可能失效？
4. safepoint 前后哪些 value 需要 relocation？

## Q66：什么是去虚拟化？

**30 秒回答**

去虚拟化把间接的 virtual/interface call 转成一个或多个可验证的直接调用。依据可来自
CHA、精确类型、profile 或 inline cache；若假设可能失效，需要 class guard、fallback、
dependency 或 deoptimization 机制保持正确性。

**追问链**

1. CHA 在动态类加载环境为什么需要依赖管理？
2. monomorphic、bimorphic、megamorphic call site 是什么？
3. 去虚拟化为什么促进 inlining？
4. guard 应放在哪里？——必须支配被保护的直接调用路径。

## Q67：逃逸分析怎样帮助 JIT？

**30 秒回答**

逃逸分析判断对象是否可能被当前作用域之外观察。未逃逸对象可进行 scalar replacement、
栈上分配、锁消除或整个 allocation elimination；但必须保留异常、identity、finalizer、
GC、deopt materialization 和内存模型语义。

**追问链**

1. `NoEscape`、`ArgEscape`、`GlobalEscape` 如何理解？
2. 对象被标量替换后 deopt 怎么恢复？
3. store 到未逃逸对象为什么仍不能盲目删？——可能随后被读取或影响异常/初始化语义。

## Q68：safepoint 为什么限制优化？

**30 秒回答**

安全点是运行时可暂停线程、扫描栈、移动对象或执行 deopt 的位置。编译器必须提供准确的
live reference 和 frame state；对象地址在 GC 后可能改变，所以跨 safepoint 的引用需要
受 statepoint/relocation 或 JVM 自己的机制约束，普通代码移动也不能破坏 poll 可达性。

**追问链**

1. safepoint poll 能否被 LICM 移出循环？——通常受不可删除/不可移动语义保护。
2. derived pointer 如何在 GC 后恢复？
3. 为什么无限循环仍需 safepoint？

---

# 9. 看 IR 现场题

## 题 1：定义是否支配使用？

```llvm
define i32 @f(i1 %c) {
entry:
  br i1 %c, label %a, label %b
a:
  %x = add i32 1, 2
  br label %merge
b:
  br label %merge
merge:
  %y = add i32 %x, 3
  ret i32 %y
}
```

**答案**

不合法。`%x` 只在 `%a` 路径定义，不支配 `merge` 中的 use。不能简单给 `%b` 路径上的
`%x` 随便选择一个值；应根据源语言语义增加另一 incoming value 和 PHI，或重构控制流。

**追问**

如果 `%b` 是 `unreachable` 呢？需要结合 CFG 可达性和 verifier 的精确规则重新判断，不能
仅凭文本猜测。

## 题 2：PHI use 在哪里？

```llvm
a:
  %x = add i32 %v, 1
  br label %merge
b:
  %y = sub i32 %v, 1
  br label %merge
merge:
  %p = phi i32 [ %x, %a ], [ %y, %b ]
```

**答案**

`%x` 的 use 在 `%a -> %merge` edge，`%y` 的 use 在 `%b -> %merge` edge。检查合法性应对
每条 `Use` 调用 dominance 查询。PHI 指令在块首的文本顺序不是 operand 的执行位置。

## 题 3：这条优化正确吗？

```llvm
%q = sdiv i32 %x, %y
br i1 %cond, label %use, label %exit
```

假设原先 `%q` 只在 `%use` 中计算，现在把它提前到分支前。

**答案**

不一定。若 `%cond` 为 false 时 `%y` 可能为 0，提前执行 `sdiv` 会引入原本不存在的 UB。
需要证明 divisor 非零并满足其他 poison/overflow 条件，或由
`isSafeToSpeculativelyExecute` 等可靠查询证明。

## 题 4：两个 load 可以合并吗？

```llvm
%a = load i32, ptr %p
call void @unknown()
%b = load i32, ptr %p
%r = add i32 %a, %b
```

**答案**

默认不能。unknown call 可能修改 `%p` 指向的内存。只有 attribute、AA Mod/Ref 或
MemorySSA 等证明 call 不 clobber 该位置时，第二个 load 才可能复用第一个。若 load 是
volatile，则即使值相同也不能这样 CSE。

## 题 5：`add nsw` 能否替换为普通 add？

```llvm
%x = add nsw i32 %a, %b
```

**答案**

把 `add nsw` 弱化为普通 add 通常不会缩小正常结果集合，但可能改变 poison 语义，是否
允许取决于周围变换方向。反向给普通 add 添加 `nsw` 必须证明永不有符号溢出，否则会
引入 poison 并可能 miscompile。

## 题 6：这个循环一定有 preheader 吗？

```llvm
entry:
  br i1 %c, label %header, label %other
other:
  br label %header
header:
  ...
```

**答案**

不一定。header 有多个循环外前驱，未形成唯一规范 preheader。运行 LoopSimplify 或用
`InsertPreheaderForLoop` 等工具规范化后才能依赖 preheader。

## 题 7：下面的删除安全吗？

```llvm
%v = load volatile i32, ptr %device
ret void
```

**答案**

不安全。即使 `%v` 没有 SSA use，volatile load 本身具有可观察行为。判断 deadness 不能
只看 `use_empty()`。

## 题 8：Java 加法可以生成 `add nsw` 吗？

**答案**

Java `int`/`long` 普通加法按二进制补码截断，不因有符号溢出产生 UB，因此通常应生成
普通 `add`，不能仅因为源值“看起来有符号”就加 `nsw`。只有额外分析已证明不溢出时才
能安全增加相关 flag。

---

# 10. 手写 Pass 与系统设计题

## 题 9：手写一个“删除重复 null check”的思路

**期待回答框架**

1. 先定义要识别的 null check IR 形态和失败行为。
2. 沿 dominator tree 维护当前支配域中已验证非空的 value 集合。
3. 两个 pointer 表达式是否等价不能只靠文本名称，可先 canonicalize/strip 合法 cast，
   必要时结合 value numbering 或 AA，但注意 alias 不等于值相等。
4. 后一个 check 只有在前一个成功路径支配它、失败路径语义一致且中间没有使假设失效的
   runtime 事件时才能删。
5. 重写 CFG 后维护 PHI、DT、LI、MSSA 或让分析失效。
6. 用 `-verify-each` 和正反例测试：同块、分支、循环、PHI、异常边、safepoint。

**典型追问**

- 前一个 check 的比较指令支配后一个 check 就够吗？——还需确认支配的是“成功事实”
  所在 edge，而不是仅比较本身。
- Java implicit null check 如何影响判断？
- safepoint 后对象引用是否仍是同一 SSA pointer？取决于 GC relocation 表示。

## 题 10：设计一个简化版局部 CSE

**期待回答框架**

- 为纯表达式构造 key：opcode、type、operands、flags、predicate 等。
- 对 commutative 操作规范 operand 顺序。
- 按 dominator tree scope 维护 expression table，离开子树时恢复。
- replacement definition 必须支配所有 use。
- load CSE 需要额外 MemorySSA/AA；第一版可以排除所有 memory/side-effect 指令。
- 不能忽略 fast-math、nsw/nuw/exact、constrained FP、metadata 语义。

**复杂度回答**

哈希表平均查询 O(1)，总体近似 O(N)；dominance 已由 DT 提供。最坏哈希和复杂表达式 key
成本另计。

## 题 11：实现 LICM-lite 要考虑什么？

**期待回答框架**

1. 要求或建立 LoopSimplify form，取得 preheader。
2. 迭代找 operands 都 invariant 的候选。
3. 排除 terminator、PHI、有副作用、convergent、volatile/atomic 等。
4. 用 speculation safety 或 must-execute 判断 trap/UB。
5. 第一版排除 memory；高级版结合 AA/MemorySSA 证明 load invariant。
6. 移动时保持 debug location，并维护/失效 SCEV、MSSA 等分析。

## 题 12：怎样测试一个 LLVM Pass？

**30 秒回答**

用最小 `.ll` 输入配合 `opt` 和 `FileCheck` 检查目标结构；同时写 SHOULD-CHANGE 和
SHOULD-NOT-CHANGE，启用 `-verify-each`。覆盖 CFG、PHI、unreachable、poison、atomic、
volatile、异常边和分析失效。对 miscompile 使用 `llvm-reduce` 缩减。

**追问链**

1. 为什么不能只检查输出中出现某字符串？——要约束顺序、变量捕获和负例。
2. unit test 与 lit test 如何选择？
3. 如何验证优化前后语义等价？——差分执行、Alive2 类工具、随机测试等；它们各有边界。

## 题 13：Pass 只在 O2 崩溃，怎样定位？

**期待回答顺序**

1. 保存输入 IR、完整命令、LLVM build 和随机环境。
2. 使用 `-verify-each` 确认第一个产生非法 IR 的 pass。
3. `-print-before/after` 或 pass instrumentation 比较变换。
4. 用 `-filter-print-funcs` 缩小函数。
5. 缩减 pipeline，再用 `llvm-reduce` 缩减 IR。
6. 检查 iterator/use-list 失效、分析过度 preserve、CFG/PHI 不同步、poison flags、
   debug-only 差异和 C++ UB。
7. 加 DT/MSSA verification 与 assertions 复现。

## 题 14：设计一个 profile-guided devirtualization pass

**期待回答框架**

```text
间接调用
  ├─ 检查 profile 是否可靠、是否单态/双态
  ├─ 生成 receiver/class guard
  ├─ 命中：直接调用，可继续 inline
  └─ 未命中：保留原间接调用或 deopt
```

需要处理：

- profile key 与调用点稳定性；
- class loading/unloading 和 dependency；
- null receiver、异常、calling convention；
- guard 必须支配 direct call；
- fallback 合并返回值的 PHI；
- branch weights 和 debug info；
- CFG、DT、LI、MSSA、call graph 更新；
- code size 与收益模型。

---

# 11. 面试前速背表

## 11.1 一句话概念表

| 概念 | 一句话回答 |
|---|---|
| SSA | 每个值单定义，use 直接连到 def，汇合处用 PHI |
| PHI | 按实际 incoming edge 选择值，use 位于前驱边 |
| Dominance | 从 entry 到目标的所有路径都经过该节点 |
| Post-dominance | 从目标到 exit 的所有路径都经过该节点 |
| IDom | 最近的严格支配者 |
| Dominance frontier | 支配作用在控制流汇合处停止的位置 |
| Critical edge | 源多后继且目标多前驱的 CFG edge |
| LoopInfo | 循环结构和嵌套，不负责推导 trip count |
| SCEV | 值随循环演化的符号表达式分析 |
| AA | 判断内存位置是否可能重叠及 call 的 Mod/Ref |
| MemorySSA | 内存访问的 SSA/def-use 搜索结构 |
| LVI | 值在特定控制流位置的常量或范围 |
| BPI | edge 概率 |
| BFI | block 相对执行频率 |
| TLI | 目标运行库语义 |
| TTI | 目标机器成本和能力 |
| mem2reg | promotable alloca 转 SSA |
| SROA | 聚合内存对象拆成 scalar/切片 |
| GVN | 用 value numbering 消除等价计算 |
| LICM | 安全地提升/下沉循环不变量 |
| SCCP | 在 SSA 值格和可执行 CFG edge 上做稀疏常量传播 |
| DCE | 删除无可观察效果且无有效 use 的代码 |
| LCSSA | 循环内定义通过 exit PHI 暴露给循环外 |
| Poison | 可传播的延迟错误值，触及敏感操作可导致 UB |
| Freeze | 把 undef/poison 固定为一个任意但稳定的普通值 |
| Opaque pointer | 指针只保留 address space，访问类型由操作表达 |

## 11.2 分析选择表

| 面试场景 | 首选工具 |
|---|---|
| 定义是否覆盖使用 | DominatorTree |
| cleanup 是否所有退出路径必经 | PostDominatorTree |
| 块属于哪个循环 | LoopInfo |
| 推导归纳变量和循环次数 | ScalarEvolution |
| 两地址是否重叠 | AliasAnalysis |
| load 向前被谁写过 | MemorySSA walker |
| 某 bit 是否确定为 0/1 | KnownBits / ValueTracking |
| 某位置的值范围 | LazyValueInfo |
| 指令能否提前投机执行 | ValueTracking speculation query |
| 目标上变换是否划算 | TargetTransformInfo |
| libc 调用可否识别替换 | TargetLibraryInfo |

## 11.3 “做优化前”六问

1. **等价性**：所有 defined behavior 下结果是否相同？
2. **可执行性**：是否让原来不执行的 trap、UB、异常或内存访问提前发生？
3. **支配性**：新定义是否支配每一条实际 use，尤其 PHI edge？
4. **内存性**：AA、Mod/Ref、atomic、volatile 和 MemorySSA 是否允许？
5. **维护性**：CFG、PHI、DT、LI、SCEV、MSSA、profile 是否更新或失效？
6. **收益性**：目标成本、代码尺寸、热度、编译时间是否值得？

## 11.4 高频陷阱表

| 错误回答 | 正确补充 |
|---|---|
| “无 use 就能删” | 还要确认无 side effect、volatile/atomic 等可观察行为 |
| “operands invariant 就能 hoist” | 还要证明 speculation、memory 和异常安全 |
| “两个 pointer 不相等就 NoAlias” | 不同 SSA pointer 仍可能指向同一内存 |
| “MayAlias 表示一定 alias” | 它表示无法证明 NoAlias |
| “PHI 在块开头读取所有 operand” | operand use 位于各 incoming edge |
| “没改 CFG 就 preserve all” | 值、memory、SCEV、AA 等分析仍可能失效 |
| “RAUW 会自动保证正确” | 它不维护 dominance、分析、metadata 或删除旧值 |
| “`inbounds`/`nsw` 是性能提示” | 它们是可产生 poison 的语义承诺 |
| “volatile 等于线程安全” | volatile 不提供 atomic ordering |
| “LLVM 会保留 Java 语义” | JIT 必须显式编码异常、GC、deopt、barrier 等约束 |

## 11.5 自测评分

| 水平 | 能力表现 |
|---|---|
| 基础 | 能准确解释 IR 对象、SSA、PHI、CFG、dominance |
| 合格 | 能解释 DT/LI/AA/SCEV/MSSA 的职责和经典优化原理 |
| 良好 | 能分析 poison、speculation、分析失效和内存正确性 |
| 高级 | 能设计非平凡 pass，覆盖 CFG/异常/profile/JIT 生命周期 |
| 专家 | 能讨论算法复杂度、目标成本、编译时间和 miscompile 定位方法 |

建议达到以下标准再参加技术面试：

- 60 秒内讲清 SSA、PHI、dominance 和 MemorySSA；
- 能在纸上画出一个循环的 CFG、DT、preheader、latch 和 exits；
- 能解释为什么一个“看起来正确”的 LICM/CSE 会因 trap 或 alias 产生 miscompile；
- 能写出新 PM Function Pass 骨架并正确返回 `PreservedAnalyses`；
- 能说出用 `-verify-each`、print-before/after、pipeline reduction、`llvm-reduce`
  定位错误的完整顺序；
- 能把自己在 Jeandle 中做过的一个优化，按“问题—分析—正确性—收益—测试”讲清楚。

---

## 附录：项目经历的 STAR 回答模板

不要只说“我写了一个 LLVM Pass”，建议使用下面结构：

| 部分 | 回答内容 |
|---|---|
| Situation | 原 pipeline 的性能、代码质量或正确性问题是什么 |
| Task | 你的 pass 要识别和变换什么，不能破坏哪些 JVM 语义 |
| Analysis | 使用了 DT、LI、AA、MSSA、SCEV 或 profile 中的哪些信息 |
| Correctness | dominance、alias、异常、GC、deopt、poison 如何证明安全 |
| Implementation | pass 层级、核心数据结构、CFG/分析怎样维护 |
| Testing | lit/FileCheck、jtreg、benchmark、负例、`-verify-each` |
| Result | 编译时间、运行时间、代码尺寸、命中率和回归数据 |
| Reflection | 遇到的 miscompile、如何缩减定位、下一步怎样改进 |

一个好的结尾示例：

> 这个变换最初只在普通分支上正确，但异常边和 PHI incoming use 暴露了 dominance 判断
> 粒度不足的问题。我把判断从 user instruction 改为逐 `Use`，CFG 修改统一通过 updater
> 维护，并加入 `-verify-each` 负例。最终在不增加明显编译时间的前提下提高了目标调用点
> 的直接调用比例。这个过程让我更重视 LLVM 语义契约，而不只是 pattern matching。
