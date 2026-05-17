# LLVM PR #194833 修改记录

## PR 概述
这个 PR 的标题是 **[AArch64] Match vector neg(and X, 1) as CMTST**，修改位置在 `llvm/lib/Target/AArch64/AArch64ISelLowering.cpp`，并新增了对应测试 `llvm/test/CodeGen/AArch64/cmtst-neg-and-one.ll`。 [github](https://github.com/llvm/llvm-project/issues/107088)

这个补丁已经合入 `llvm:main`，属于 AArch64 后端的 DAG combine / 指令选择阶段优化，而不是中端通用 IR combine。 [github](https://github.com/llvm/llvm-project/issues/107088)

## 修改背景
这个 PR 的背景是 AArch64 后端存在一个已知的 missed optimization：某些向量按位测试语义本来可以用 `CMTST` 指令表达，但实际代码生成没有折叠到该指令。 [lists.llvm](https://lists.llvm.org/pipermail/all-commits/Week-of-Mon-20260420/297680.html)

相关 issue #107088 给出的例子里，源码语义本质上是对向量执行按位与，再判断结果是否不等于零，最后把布尔结果作为掩码参与 `select`/`bsl` 类操作；实际生成的汇编是 `and` 加 `cmeq`，而 issue 认为更理想的形式应当是 `cmtst` 加 `bit`/`bif` 一类按位选择指令。 [lists.llvm](https://lists.llvm.org/pipermail/all-commits/Week-of-Mon-20260420/297680.html)

从 PR 的实现方式看，这个补丁处理的不是一般形式的 `(and X, C) != 0`，而是更窄的特例：把向量上的 `sub 0, (and X, 1)` 或等价的 `neg (and X, 1)` 识别出来，再改写为 `setcc (and X, 1), 0, ne`，从而为后续选择 `CMTST` 创造条件。 [github](https://github.com/llvm/llvm-project/issues/107088)

之所以只匹配 `1` 或 splat(1)，是因为这个模式依赖“与 1 后非零”能够自然映射为布尔掩码语义；PR 中新增的辅助函数 `isOneVector` 也明确只接受 one / one-splat / `DUP` 出来的常量 1 向量。 [github](https://github.com/llvm/llvm-project/issues/107088)

## 修改逻辑
这个 PR 在 `AArch64ISelLowering.cpp` 中新增了 `isOneVector(SDValue V)`，用于识别“元素全为 1”的固定长度向量常量，或者由 `AArch64ISD::DUP` 复制出来的 1 常量向量。 [github](https://github.com/llvm/llvm-project/issues/107088)

核心变换函数是 `performSubNegAndOneCombine(SDNode *N, SelectionDAG &DAG)`。 [github](https://github.com/llvm/llvm-project/issues/107088)

它的匹配条件包括：
- 当前节点必须是 `ISD::SUB`。 [github](https://github.com/llvm/llvm-project/issues/107088)
- 结果类型必须是 fixed-length vector。 [github](https://github.com/llvm/llvm-project/issues/107088)
- 左操作数必须是零向量。 [github](https://github.com/llvm/llvm-project/issues/107088)
- 右操作数必须是 `ISD::AND`。 [github](https://github.com/llvm/llvm-project/issues/107088)
- `and` 的第二个操作数必须是 one-vector，也就是 1 的向量 splat 或等价形式。 [github](https://github.com/llvm/llvm-project/issues/107088)

当这些条件满足时，原始形式：

```text
sub zerovector, (and X, 1)
```

会被改写成：

```text
setcc (and X, 1), 0, ne
```

也就是逐元素判断 `(and X, 1)` 是否不等于 0。 [github](https://github.com/llvm/llvm-project/issues/107088)

这个改写的关键点在于：对 `and X, 1` 来说，结果按元素只可能是 0 或 1，因此“与零比较是否非零”正好对应布尔真值；而在 AArch64 向量后端里，这类“test bits and form all-ones/all-zeros mask”的语义可以进一步映射到 `CMTST`。 [lists.llvm](https://lists.llvm.org/pipermail/all-commits/Week-of-Mon-20260420/297680.html)

这也是它被放在 AArch64 target combine 中而不是 InstCombine 的原因：这里不是做平台无关 canonicalization，而是在目标后端把一个已经被 canonicalize 成 `sub 0, (and X, 1)` 的形式重新识别回更适合 AArch64 指令选择的语义模式。 [github](https://github.com/llvm/llvm-project/issues/107088)

## 测试用例
PR 新增了专门的代码生成测试文件 `llvm/test/CodeGen/AArch64/cmtst-neg-and-one.ll`，用于覆盖这个新 combine。 [github](https://github.com/llvm/llvm-project/issues/107088)

从改动规模看，测试文件是本次 PR 的主要新增内容之一；整个 PR 总共新增 134 行，没有删除代码，其中后端实现新增 25 行，其余主要来自测试。 [github](https://github.com/llvm/llvm-project/issues/107088)

结合补丁逻辑，这个测试的目的应当是验证以下几类要点：
- 固定长度向量上的 `sub 0, (and X, splat(1))` 或等价 `neg(and X, 1)` 能被识别。 [github](https://github.com/llvm/llvm-project/issues/107088)
- 识别后不会继续生成单独的 `and` 加 `neg` 低效序列，而是走向 `CMTST` 相关选择路径。 [github](https://github.com/llvm/llvm-project/issues/107088)
- 仅在常量为 1 向量时触发，避免把更一般的 `and X, 2`、`and X, 4` 等情况错误折叠成同一模式。 [github](https://github.com/llvm/llvm-project/issues/107088)

issue #107088 中展示的 IR 也能帮助理解测试意图：它先形成 `%3 = and <16 x i8> %2, <2,2,...>`，再做 `icmp ne %3, zeroinitializer`，最后用比较结果做 `select`；这类“bit test -> mask -> select”的链条正是 `CMTST` 适合承接的目标语义。 [lists.llvm](https://lists.llvm.org/pipermail/all-commits/Week-of-Mon-20260420/297680.html)

不过需要注意，issue 中举的常量示例是 `2`，而本 PR 实际落地的是更保守的 `1` 特例匹配，因此测试范围应围绕 `and X, 1` 展开，而不是泛化到任意单 bit 常量。 [lists.llvm](https://lists.llvm.org/pipermail/all-commits/Week-of-Mon-20260420/297680.html)

## 影响与意义
这个修改修复的是 AArch64 后端一个非常具体但真实存在的 missed optimization：当前端/中端把布尔掩码构造成 `sub 0, (and X, 1)` 之后，后端现在可以把它重新恢复成“按位测试非零”的目标语义。 [lists.llvm](https://lists.llvm.org/pipermail/all-commits/Week-of-Mon-20260420/297680.html)

对于依赖向量 mask 的代码路径，这有助于生成更贴近 AArch64 NEON 指令集语义的代码，并减少无谓的指令序列拆分。 [lists.llvm](https://lists.llvm.org/pipermail/all-commits/Week-of-Mon-20260420/297680.html)

从 LLVM 分层上看，这类改动也很典型：中端负责 canonicalize，目标后端再针对本架构的专有指令做模式回收和定向匹配。 [github](https://github.com/llvm/llvm-project/issues/107088)

## 链接
- PR: [https://github.com/llvm/llvm-project/pull/194833/changes](https://github.com/llvm/llvm-project/pull/194833/changes) [github](https://github.com/llvm/llvm-project/issues/107088)
- Related issue: [https://github.com/llvm/llvm-project/issues/107088](https://github.com/llvm/llvm-project/issues/107088) [lists.llvm](https://lists.llvm.org/pipermail/all-commits/Week-of-Mon-20260420/297680.html)
