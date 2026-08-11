# LLVM 面试核心专题：Loop、mem2reg、SSA 与分析框架

> 基于本仓库 LLVM 22.0.0git。本文采用“知识框架 → 问答链 → IR 演化 → C++ 骨架 →
> 反例 → 手写题”的形式，适合围绕一个专题连续讲 10～20 分钟。
>
> LLVM 中正式的 Pass 名称是 `mem2reg`，面试中说的 “mem2register” 通常就是它。

## 目录

1. 总体知识地图
2. 专题一：SSA 与 mem2reg
3. 专题二：CFG 与 DominatorTree
4. 专题三：LoopInfo 与循环规范形
5. 专题四：ScalarEvolution 与归纳变量
6. 专题五：LICM 与循环代码移动
7. 专题六：AliasAnalysis 与 MemorySSA
8. 专题七：GVN、CSE 与值等价
9. 专题八：新 Pass Manager 与分析失效
10. 综合手写题
11. 面试速背框架

---

# 1. 总体知识地图

## 1.1 八个专题之间的关系

```text
源语言局部变量
      │
      ▼
alloca / load / store
      │
      │ SROA + mem2reg
      ▼
SSA value + PHI
      │
      ├──────────────→ DominatorTree ──→ 定义是否覆盖使用
      │                       │
      │                       └────────→ SSA 构造 / GVN / LICM
      │
      ▼
CFG 中的回边与单入口区域
      │
      │ LoopInfo
      ▼
Loop + preheader + latch + exits + LCSSA
      │
      ├──────────────→ ScalarEvolution ─→ IV / trip count / range
      │
      └──────────────→ AA + MemorySSA ──→ load/store 依赖
                                      │
                                      ▼
                          LICM / GVN / DSE / Vectorize

所有变换最终都受新 Pass Manager 的缓存与失效规则约束。
```

## 1.2 面试回答统一框架

回答任意分析或优化时，按下面顺序展开：

| 层次 | 要回答的问题 |
|---|---|
| 定义 | 它解决什么问题，输入和输出是什么？ |
| 表示 | 核心数据结构或 lattice/tree/SSA 节点是什么？ |
| 算法 | 如何构建、查询或达到不动点？ |
| 正确性 | dominance、memory、UB、异常等条件是什么？ |
| 工程接口 | 在新 PM 中如何获取，修改后如何维护？ |
| 复杂度 | 主要时间/空间成本在哪里？ |
| 反例 | 什么情况下必须保守拒绝？ |
| 协作 | 它为哪些 Pass 服务，又依赖哪些分析？ |

## 1.3 最容易被连续追问的主线

```text
什么是 SSA？
  → PHI 的 use 在哪里？
  → SSA 如何构造？
  → 为什么需要 dominance frontier？
  → mem2reg 哪些 alloca 不能提升？
  → 修改 CFG 后如何维护 SSA 和 DT？

什么是循环？
  → backedge 为什么要求 header 支配 latch？
  → preheader/latch/exiting/exit 分别是什么？
  → LoopSimplify 和 LCSSA 有什么用？
  → SCEV 如何表示 IV？
  → LICM 为什么不能只检查 operands invariant？
  → load hoist 为什么同时需要 AA 和 MemorySSA？
```

---

# 2. 专题一：SSA 与 mem2reg

## 2.1 先讲清 SSA

SSA（Static Single Assignment）要求一个 SSA value 只有一个定义，每个 use 直接引用其
definition。源语言变量可以更新多次，但会被重命名成不同 SSA value。

```text
源代码：                 SSA：
x = 1                    x.0 = 1
x = x + 1                x.1 = x.0 + 1
return x                 return x.1
```

### 为什么 SSA 有利于优化

- def-use 链显式，无需在每个程序点搜索“当前变量值”。
- 常量传播、DCE、CSE 可以沿 use-def/def-use 稀疏执行。
- 每个值的性质可以与其唯一 def 关联。
- 数据流汇合通过 PHI 明确表示。
- dominance 可以直接验证定义对使用是否合法。

### SSA 不代表什么

- 不代表源语言变量只能赋值一次。
- 不代表内存只写一次。
- 不代表所有程序状态都天然 SSA 化。
- 不代表 PHI 是真实机器指令。

## 2.2 PHI 的语义

```llvm
entry:
  br i1 %cond, label %left, label %right

left:
  %x.left = add i32 %a, 1
  br label %merge

right:
  %x.right = sub i32 %a, 1
  br label %merge

merge:
  %x = phi i32 [ %x.left, %left ], [ %x.right, %right ]
  ret i32 %x
```

```text
                entry
               /     \
           left       right
         x.left      x.right
               \     /
                merge
           x = φ(x.left, x.right)
```

PHI 根据实际进入 `merge` 的 edge 选择值。最重要的面试点：

> `%x.left` 的 use 位于 `%left -> %merge` 边，`%x.right` 的 use 位于
> `%right -> %merge` 边，而不是位于 `merge` 块开头。

因此精确 dominance 查询应使用：

```cpp
for (Use &U : Def->uses()) {
  if (!DT.dominates(Def, U))
    reportIllegalUse(U);
}
```

## 2.3 mem2reg 做什么

`mem2reg` 把满足要求的局部栈槽：

```text
alloca → store/load → memory variable
```

转换为：

```text
SSA values + 必要的 PHI
```

它不是通用逃逸分析，也不是任意内存对象的“寄存器分配”。`mem2reg` 的“register”是
抽象 SSA register，是否最终放入物理寄存器由后端决定。

## 2.4 最简单的 Before/After

### Before

```llvm
define i32 @simple(i32 %arg) {
entry:
  %x = alloca i32, align 4
  store i32 %arg, ptr %x, align 4
  %v = load i32, ptr %x, align 4
  %r = add i32 %v, 1
  ret i32 %r
}
```

### After

```llvm
define i32 @simple(i32 %arg) {
entry:
  %r = add i32 %arg, 1
  ret i32 %r
}
```

这里不需要 PHI，因为每个 load 到达时只有一个 store 定义。

## 2.5 分支汇合时的 Before/After

### Before

```llvm
define i32 @max(i32 %a, i32 %b) {
entry:
  %result = alloca i32, align 4
  %cmp = icmp sgt i32 %a, %b
  br i1 %cmp, label %take.a, label %take.b

take.a:
  store i32 %a, ptr %result, align 4
  br label %merge

take.b:
  store i32 %b, ptr %result, align 4
  br label %merge

merge:
  %v = load i32, ptr %result, align 4
  ret i32 %v
}
```

### After

```llvm
define i32 @max(i32 %a, i32 %b) {
entry:
  %cmp = icmp sgt i32 %a, %b
  br i1 %cmp, label %take.a, label %take.b

take.a:
  br label %merge

take.b:
  br label %merge

merge:
  %result.0 = phi i32 [ %a, %take.a ], [ %b, %take.b ]
  ret i32 %result.0
}
```

## 2.6 mem2reg 的经典算法框架

```text
步骤 1：筛选 promotable alloca
步骤 2：收集该变量的 store 定义块和 load 使用块
步骤 3：在 iterated dominance frontier 插入 PHI
步骤 4：沿 dominator tree DFS rename
步骤 5：用当前 SSA value 替换 load
步骤 6：store 更新当前值，不再生成内存写
步骤 7：删除 alloca/load/store
步骤 8：清理 trivial/dead PHI
```

### Rename 的栈式思路

```cpp
void rename(DomTreeNode *Node, Value *Incoming) {
  BasicBlock *BB = Node->getBlock();
  Value *Current = Incoming;

  if (PHINode *PN = PhiForBlock.lookup(BB))
    Current = PN;

  for (Instruction &I : make_early_inc_range(*BB)) {
    if (isStoreToPromotedAlloca(I)) {
      Current = cast<StoreInst>(I).getValueOperand();
      I.eraseFromParent();
    } else if (isLoadFromPromotedAlloca(I)) {
      I.replaceAllUsesWith(Current);
      I.eraseFromParent();
    }
  }

  for (BasicBlock *Succ : successors(BB))
    if (PHINode *PN = PhiForBlock.lookup(Succ))
      PN->addIncoming(Current, BB);

  for (DomTreeNode *Child : *Node)
    rename(Child, Current);
}
```

这只是算法骨架。LLVM 实现还要处理 undef、批量变量、高效 IDF、debug assignment tracking、
单 store/single block 快速路径等。

## 2.7 为什么是 dominance frontier

若定义块 `D` 支配某个 predecessor，却不严格支配汇合块 `Y`，说明存在另一条到 `Y` 的
路径未经过 `D`。`Y` 正是多个版本需要合并的位置。

```text
       entry
       /   \
   D1       D2
       \   /
         Y       ← D1、D2 的 dominance frontier
```

多个新 PHI 本身也构成定义，因此要计算 iterated dominance frontier，直到不再产生新
放置点。Pruned SSA 还结合变量活跃性，避免在值根本不会被读取的汇合点放 PHI。

## 2.8 哪些 alloca 不能直接提升

### 地址逃逸

```llvm
%x = alloca i32
call void @capture(ptr %x)
%v = load i32, ptr %x
```

未知调用可能保存地址或读写内存。

### 非直接访问

```llvm
%x = alloca { i32, i32 }
%field = getelementptr { i32, i32 }, ptr %x, i32 0, i32 1
store i32 7, ptr %field
```

这类聚合通常先交给 SROA 拆分，再产生 mem2reg 可提升的标量。

### Volatile

```llvm
%v = load volatile i32, ptr %x
```

volatile access 自身具有可观察行为，不能简单变成 SSA use。

## 2.9 新 PM 使用方式

命令行：

```bash
opt -S -passes='mem2reg' input.ll -o output.ll
```

C++ pipeline：

```cpp
#include "llvm/Transforms/Utils/Mem2Reg.h"

FunctionPassManager FPM;
FPM.addPass(PromotePass());
```

直接工具接口常见于更细粒度场景：

```cpp
SmallVector<AllocaInst *> Allocas;
for (Instruction &I : F.getEntryBlock())
  if (auto *AI = dyn_cast<AllocaInst>(&I))
    if (isAllocaPromotable(AI))
      Allocas.push_back(AI);

PromoteMemToReg(Allocas, DT, &AC);
```

具体 overload 以当前 `llvm/Transforms/Utils/PromoteMemToReg.h` 为准。

## 2.10 高频问答

**问：mem2reg 是否把变量放进物理寄存器？**

答：不是。它把内存表示提升为 SSA value。最终 location 由 instruction selection 和
register allocation 决定，也可能再次 spill 到栈。

**问：为什么只提升 alloca，而不提升 malloc？**

答：alloca 是函数局部对象，生命周期和 use 更容易闭合；malloc 对象可能逃逸、alias、
跨调用存在。更复杂的 allocation elimination 需要逃逸分析和语言运行时语义。

**问：mem2reg 是否需要完整活跃变量分析？**

答：基本 SSA 构造可通过定义块和 IDF 放 PHI；高质量实现会利用活跃性避免无用 PHI，
LLVM 实现还有多种快速路径。

**问：PHI 最终怎样变成机器码？**

答：后端把 Machine PHI 转为前驱边上的并行复制，critical edge 必要时拆分，再由寄存器
分配/coalescing 尽量消掉复制。

---

# 3. 专题二：CFG 与 DominatorTree

## 3.1 基本定义

| 概念 | 定义 |
|---|---|
| CFG | 基本块为节点、控制转移为边的有向图 |
| dominate | 从 entry 到 B 的所有路径都经过 A |
| properly dominate | A dominate B 且 A != B |
| immediate dominator | 离 B 最近的严格支配者 |
| dominator tree | 每个节点父亲为 idom 的树 |
| post-dominate | 从 B 到所有正常/建模出口的路径都经过 A |
| dominance frontier | A 的支配作用首次停止的汇合边界 |

## 3.2 CFG 与 DT 不是一回事

```text
CFG：                         Dominator Tree：

       entry                         entry
       /   \                         /   \
      a     b                       a     b
       \   /                              │
       merge                            merge
         │                                │
        exit                             exit
```

这里 `merge` 的 idom 是 `entry`，不是 CFG 中的某个直接 predecessor。DT edge 不要求是
CFG edge。

## 3.3 基本查询

```cpp
DominatorTree &DT = FAM.getResult<DominatorTreeAnalysis>(F);

bool BlockDom = DT.dominates(A, B);
bool Strict = DT.properlyDominates(A, B);
bool InstDom = DT.dominates(DefI, UserI);
bool UseDom = DT.dominates(DefV, U);
bool Reachable = DT.isReachableFromEntry(BB);

BasicBlock *NCD = DT.findNearestCommonDominator(A, B);
DomTreeNode *N = DT.getNode(BB);
BasicBlock *IDom =
    N && N->getIDom() ? N->getIDom()->getBlock() : nullptr;
```

## 3.4 指令级支配

跨块时主要由 block dominance 决定；同块内要看指令次序：

```llvm
%a = add i32 %x, 1
%b = mul i32 %a, 2   ; %a 支配这里的 use
```

将 `%a` 移到 `%b` 之后，类型仍然匹配，但 SSA 不合法。

特殊值：Argument、Constant、GlobalValue 的可用性不同于普通 Instruction；不要手写粗糙的
“比较块位置”逻辑，优先使用 DT 已有 overload。

## 3.5 PHI dominance 的经典陷阱

```llvm
a:
  %x = add i32 %v, 1
  br label %merge

merge:
  %p = phi i32 [ %x, %a ], [ %y, %b ]
```

`DT.dominates(%x, %p)` 不能完整表达目标问题，因为 `%p` 有多条 incoming use。正确方式：

```cpp
for (Use &U : X->uses()) {
  if (auto *PN = dyn_cast<PHINode>(U.getUser())) {
    bool Valid = DT.dominates(X, U);
    BasicBlock *Incoming = PN->getIncomingBlock(U.getOperandNo());
    (void)Incoming;
  }
}
```

## 3.6 DT 的典型用途

### 冗余检查消除

```text
check(p != null)
     │ success edge dominates
     ▼
... second check(p != null) ...  → 可能删除
```

必须由“第一个 check 的成功 edge”支配后一个位置，仅比较 `icmp` 指令是否支配是不够的。

### 公共表达式消除

已有表达式定义必须支配待替换定义的全部 use。

### 合法插入点

多个 use 的共同支配块可从最近公共支配者逐步求出：

```cpp
BasicBlock *Where = nullptr;
for (Instruction *UseI : Uses)
  Where = Where ? DT.findNearestCommonDominator(Where, UseI->getParent())
                : UseI->getParent();
```

这只确定 block；还要确认 operands 在具体 insertion point 可用，且指令可安全投机。

## 3.7 PostDominatorTree

```cpp
PostDominatorTree &PDT =
    FAM.getResult<PostDominatorTreeAnalysis>(F);

if (PDT.dominates(CleanupBB, WorkBB)) {
  // 从 WorkBB 出发的所有建模退出路径都经过 CleanupBB。
}
```

常见用途：control dependence、ADCE、统一 cleanup、判断某操作是否在所有退出路径执行。
多出口和无限循环可能引入虚拟 root，不要假设 root 必然对应一个真实 BasicBlock。

## 3.8 修改 CFG 后怎样更新 DT

### 策略 A：声明失效

```cpp
return PreservedAnalyses::none();
```

最稳妥，后续需要时重算。

### 策略 B：DomTreeUpdater

```cpp
DomTreeUpdater DTU(DT, DomTreeUpdater::UpdateStrategy::Lazy);

SmallVector<DominatorTree::UpdateType> Updates;
Updates.push_back({DominatorTree::Delete, OldFrom, OldTo});
Updates.push_back({DominatorTree::Insert, NewFrom, NewTo});

// 先让真实 CFG 与 Updates 描述一致。
DTU.applyUpdates(Updates);
DTU.flush();
```

### 策略 C：使用变换工具

```cpp
BasicBlock *NewBB = SplitBlock(BB, SplitBefore, &DTU, &LI);
bool Changed = MergeBlockIntoPredecessor(BB, &DTU, &LI);
```

优先使用 `BasicBlockUtils`，它能同时维护 PHI 以及传入的 DT/LI/MSSA updater。

## 3.9 验证

```cpp
assert(!verifyFunction(F, &errs()));
assert(!DT.verify(DominatorTree::VerificationLevel::Full));
```

注意这些 verifier 常使用“`true` 表示 broken”的返回约定。

```bash
opt -passes='verify<domtree>' -disable-output input.ll
opt -passes='print<domtree>' -disable-output input.ll
opt -passes='dot-dom' -disable-output input.ll
```

## 3.10 高频问答

**问：`dominates(A, A)` 是 true 吗？**

答：是。排除自身使用 `properlyDominates`。

**问：unreachable block 中谁支配谁？**

答：不可达区域的支配语义和实现细节容易违反从 entry 路径出发的直觉。分析前先调用
`isReachableFromEntry`，明确 pass 是否要处理不可达块。

**问：CFG 增加一条 edge 为什么会破坏 dominance？**

答：新路径可能绕过原支配者。例如原来 A 是到 B 的唯一路径，新增 Entry→B 后 A 不再
支配 B。

---

# 4. 专题三：LoopInfo 与循环规范形

## 4.1 LLVM 中的自然循环

若 edge `Latch -> Header` 满足 Header 支配 Latch，它是 backedge。Header 加上能够反向
到达 Latch 且不越过 Header 的节点构成自然循环。

```text
           preheader
               │
               ▼
             header ◄────────┐
               │             │ backedge
               ▼             │
              body           │
               │             │
               ▼             │
             latch ──────────┘
               │
               ▼
              exit
```

LLVM `LoopInfo` 主要表示单入口 header 的 reducible loop。Irreducible CFG 有多个入口，
不能简单当作普通自然循环处理。

## 4.2 术语表

| 术语 | 定义 | 是否唯一 |
|---|---|---|
| Header | 支配整个循环的入口块 | 对一个 `Loop` 唯一 |
| Backedge | 循环内指向 Header 的边 | 可多个 |
| Latch | Backedge 的源块 | 可多个 |
| Preheader | 循环外唯一进入 Header 的规范块 | 可能不存在 |
| Exiting block | 循环内含有通向外部 edge 的块 | 可多个 |
| Exit block | 循环外、由 exiting edge 到达的块 | 可多个 |
| Dedicated exit | exit 的所有 predecessor 都来自该循环 | 规范性质 |
| Parent loop | 直接包含当前循环的外层循环 | 最多一个 |

`getExitingBlock()`、`getLoopLatch()` 等单数接口在不唯一时通常返回 null。不要把 null 当作
“没有 exiting/latch”，它可能表示“有多个”。

`getExitBlock()` 按 `getExitBlocks()` 的结果判断单一出口，重复 CFG edge 也会影响结果；
`getUniqueExitBlock()` 先按目标块去重。同一个 switch 从 loop 内以多条 edge 跳到同一外部
块时，两者可能不同。

## 4.3 常用接口

```cpp
LoopInfo &LI = FAM.getResult<LoopAnalysis>(F);

Loop *L = LI.getLoopFor(BB);       // 最内层
unsigned D = LI.getLoopDepth(BB);  // 不在 loop 中为 0
bool IsHeader = LI.isLoopHeader(BB);

BasicBlock *Header = L->getHeader();
BasicBlock *Preheader = L->getLoopPreheader();
BasicBlock *Latch = L->getLoopLatch();
BasicBlock *Exiting = L->getExitingBlock();
BasicBlock *Exit = L->getExitBlock();
BasicBlock *UniqueExit = L->getUniqueExitBlock();

bool ContainsBB = L->contains(BB);
bool ContainsInst = L->contains(I);
bool Invariant = L->isLoopInvariant(V);

SmallVector<BasicBlock *> ExitingBlocks;
L->getExitingBlocks(ExitingBlocks);
SmallVector<BasicBlock *> ExitBlocks;
L->getExitBlocks(ExitBlocks);
```

遍历嵌套循环：

```cpp
void visitLoop(Loop *L) {
  process(*L);
  for (Loop *Sub : L->getSubLoops())
    visitLoop(Sub);
}

for (Loop *TopLevel : LI)
  visitLoop(TopLevel);
```

## 4.4 LoopSimplify form

常见保证包括：

- preheader；
- 单一 backedge/latch 形态；
- dedicated exits。

```cpp
if (!L->isLoopSimplifyForm())
  return false;
```

### 为什么 preheader 重要

```text
Without preheader:             With preheader:

outside1 ─┐                    outside1 ─┐
          ├→ header                      ├→ preheader → header
outside2 ─┘                    outside2 ─┘
```

LICM 要插入“一旦进入循环就执行一次”的代码。直接放在 `outside1` 或 `outside2` 会漏路径，
放在 header 又每轮执行；preheader 提供唯一位置。

## 4.5 LCSSA form

### Before

```llvm
loop:
  %v = ...
  br i1 %done, label %exit, label %loop

exit:
  %r = add i32 %v, 1
```

### After

```llvm
loop:
  %v = ...
  br i1 %done, label %exit, label %loop

exit:
  %v.lcssa = phi i32 [ %v, %loop ]
  %r = add i32 %v.lcssa, 1
```

LCSSA 要求循环内定义、循环外使用的值通过 exit PHI。单 incoming PHI 看似冗余，却把跨
循环边界的 use 集中起来。

### LCSSA 的价值

- loop pass 只需检查 exit PHI，就能找到所有外部 use。
- 删除、版本化、unroll 或 sink 循环时更容易修复 SSA。
- 嵌套 loop 的变换影响范围更局部。

```cpp
bool IsLCSSA = L->isLCSSAForm(DT);
```

## 4.6 Loop invariant 的不同层次

```cpp
bool OperandInvariant = L->isLoopInvariant(V);
```

这只说明 `V` 在循环迭代中不变，不自动说明使用 `V` 的指令可移出循环。

| 层次 | 问题 |
|---|---|
| Operand invariant | operands 是否来自 loop 外或 invariant def？ |
| Memory invariant | load 的内存是否不会被 loop 修改？ |
| Execution safe | 提前执行是否新增 trap/UB/exception？ |
| Placement legal | operands 是否在 preheader 可用，结果是否支配 uses？ |
| Profitable | 移动是否真的减少动态成本？ |

## 4.7 修改 LoopInfo

拆块、合并块、改变 loop membership 后不能只更新 CFG：

- 新块属于哪个最内层 loop？
- 新 edge 是否形成/删除 backedge？
- header/latch/exit 是否改变？
- 父子 loop 层级是否仍正确？
- DT、SCEV、LCSSA、MemorySSA 是否仍有效？

小修改使用接收 `DTU`、`LI`、`MSSAU` 的公共 utility；复杂变换无法可靠维护时使分析失效。

## 4.8 高频问答

**问：一个 Loop 可以有多个 exit 吗？**

答：可以。还要区分多个 exiting blocks、多个 exit edges 和多个 unique exit blocks。

**问：`getLoopFor(BB)` 返回外层还是内层？**

答：包含该块的最内层 Loop，可用 `getParentLoop()` 向外遍历。

**问：有 LoopInfo 是否意味着有 preheader 和 LCSSA？**

答：不意味着。这些是 pipeline 通过 LoopSimplify/LCSSA pass 建立的附加规范形式。

**问：不可约循环是什么？**

答：循环区域有多个入口，没有单一 header 支配整个区域；许多自然循环算法和优化需要先
结构化或保守放弃。

---

# 5. 专题四：ScalarEvolution 与归纳变量

## 5.1 SCEV 的定位

LoopInfo 回答“循环结构是什么”，ScalarEvolution 回答“值如何随迭代演化”。SCEV 不把
每条 IR 指令原样复制，而是建立唯一化的符号表达式。

| SCEV 类型 | 含义示例 |
|---|---|
| `SCEVConstant` | 常量 4 |
| `SCEVUnknown` | 无法继续分解的 IR value |
| `SCEVAddExpr` | `A + B` |
| `SCEVMulExpr` | `A * B` |
| `SCEVUDivExpr` | unsigned division |
| `SCEVAddRecExpr` | 某 loop 内的递推 `{Start,+,Step}` |
| `SCEVCouldNotCompute` | 无法求得 trip/exit 信息 |

## 5.2 AddRecurrence

```llvm
loop:
  %i = phi i32 [ 0, %preheader ], [ %next, %latch ]
  ...
latch:
  %next = add i32 %i, 2
  br label %loop
```

SCEV 可表示为：

```text
{0,+,2}<loop>

iteration 0: 0
iteration 1: 2
iteration 2: 4
iteration k: 0 + k*2
```

嵌套循环可能产生嵌套 AddRec，每一层关联特定 `Loop *`。

## 5.3 常用接口

```cpp
ScalarEvolution &SE = FAM.getResult<ScalarEvolutionAnalysis>(F);

const SCEV *S = SE.getSCEV(V);
S->print(errs());

if (auto *AR = dyn_cast<SCEVAddRecExpr>(S)) {
  const SCEV *Start = AR->getStart();
  const SCEV *Step = AR->getStepRecurrence(SE);
  Loop *Scope = AR->getLoop();
}

const SCEV *BTC = SE.getBackedgeTakenCount(L);
if (!isa<SCEVCouldNotCompute>(BTC)) {
  const SCEV *Trip = SE.getTripCountFromExitCount(BTC);
}

bool Inv = SE.isLoopInvariant(S, L);
bool NonNegative = SE.isKnownNonNegative(S);
```

## 5.4 Backedge taken count 与 trip count

对于一个正常执行 10 次、每次末尾判断是否回边的循环：

```text
header 执行次数：10      → trip count
backedge 被走次数：9     → backedge taken count
```

通常 `TripCount = BackedgeTakenCount + 1`，但面试中必须补充：

- 循环可能零次进入；
- exit count 可能针对某个 exit，而非整个 loop；
- 加一可能在表达式位宽下溢出；
- multiple exits、undefined behavior、不可计算条件会影响结果。

所以应使用 SCEV 提供的转换接口，而不是手工无条件加一。

## 5.5 SCEV 如何帮助 bounds-check elimination

源模式：

```text
for i = 0; i < len; i++:
    check(i >= 0 && i < len)
    use(array[i])
```

证明框架：

1. 识别 `%i` 是 `{0,+,1}<L>`。
2. 从 loop guard 推导迭代域。
3. 证明该域内 `0 <= i < len`。
4. 确认 `len` 在循环中不变。
5. 确认检查失败的异常、deopt 和副作用语义可被前置/消除。
6. 使用 dominance 确认 guard 覆盖所有数组访问。

SCEV 只解决数值关系，不能单独证明 Java exception ordering、array identity、GC 或内存安全。

## 5.6 nowrap flags

`{0,+,1}<nuw>` 等 flags 允许 SCEV 假设递推不发生相应 wrap，从而证明单调性和范围。错误
添加 flags 会把合法 wraparound 程序变为 poison/UB 相关行为。

对 Java `int` 普通加法，不能因为它被当作有符号数使用就自动添加 `nsw`。

## 5.7 修改 IR 后的失效

SCEV 会缓存 IR value 到表达式的映射。改变 induction、exit、loop structure 后，旧表达式
可能无效。

两种选择：

- 调用恰当的 `forgetValue`、`forgetLoop` 等精细接口；
- Function Pass 返回不保留 `ScalarEvolutionAnalysis`。

若不完全理解依赖闭包，后者更安全。

## 5.8 高频问答

**问：SCEV 能否计算所有循环次数？**

答：不能。复杂控制流、非仿射递推、multiple exits、不可预测 load/call 都可能返回
`SCEVCouldNotCompute`，优化必须保守。

**问：SCEV 是范围分析吗？**

答：它能推导 signed/unsigned range 和 loop evolution，但定位不等同于通用 range analysis；
LVI、ConstantRange、KnownBits 等也解决不同层面的范围问题。

**问：SCEV 表达式与原 IR 一一对应吗？**

答：不一一对应。多个 IR 值可以映射到同一个 canonical SCEV，一个 SCEV 也可能没有直接
对应的单条 IR 指令。

---

# 6. 专题五：LICM 与循环代码移动

## 6.1 LICM 的两个方向

| 方向 | 行为 | 目的 |
|---|---|---|
| Hoist | 从 loop 移到 preheader | 每轮执行变一次执行 |
| Sink | 从 loop 移到 exit | 只在真正使用的退出路径执行 |

LLVM LICM 还可能做 scalar promotion 等更复杂的内存优化，但面试先讲清 hoist/sink。

## 6.2 基础算术提升

### Before

```llvm
preheader:
  br label %loop

loop:
  %i = phi i32 [ 0, %preheader ], [ %next, %latch ]
  %scale = mul i32 %factor, 4
  %v = add i32 %i, %scale
  br label %latch

latch:
  %next = add i32 %i, 1
  %cond = icmp slt i32 %next, %n
  br i1 %cond, label %loop, label %exit

exit:
  ret void
```

### After

```llvm
preheader:
  %scale = mul i32 %factor, 4
  br label %loop

loop:
  %i = phi i32 [ 0, %preheader ], [ %next, %latch ]
  %v = add i32 %i, %scale
  br label %latch
```

`%factor` 定义在循环外，mul 无副作用且可安全执行，结果支配所有原 uses。

## 6.3 LICM 合法性决策树

```text
Operands 都 loop invariant？
  ├─ 否 → 不能 hoist
  └─ 是
      │
      ▼
指令有 side effect / convergent / volatile / forbidden atomic？
  ├─ 是 → 通常不能 hoist
  └─ 否
      │
      ▼
指令可能 trap、throw、产生立即不可接受行为？
  ├─ 是 → 必须证明原位置 must-execute，或额外证明安全
  └─ 否
      │
      ▼
若读内存，循环中可能 clobber 同一位置？
  ├─ 是/未知 → 不能 hoist
  └─ 否
      │
      ▼
新位置 operands 可用且结果支配全部 use？
  ├─ 否 → 不能
  └─ 是 → 合法，再做 profitability 判断
```

## 6.4 除法反例

### Before

```llvm
entry:
  br i1 %enter, label %loop, label %exit

loop:
  %q = sdiv i32 %x, %y
  ...
```

如果 `%enter` 为 false，原程序不执行除法。把 `%q` 放到 entry 会在 `%y == 0` 时新增 UB。
operands invariant 只回答“不随迭代变化”，没有回答“提前执行安全”。

## 6.5 Load hoist 的完整推理

### 候选

```llvm
loop:
  %v = load i32, ptr %p, align 4
  ...
```

### 需要分别证明

```text
Address invariant:       %p 在 loop 中不变
Dereferenceable:         在 preheader 执行 load 不会新增 trap
No clobber:              loop 内 store/call 不修改该 location
Ordering legal:          非 volatile，atomic ordering 允许
Exception/GC legal:      不跨越语言运行时禁止边界
Use dominance:           提升后的 %v 支配全部 use
```

### AA 与 MemorySSA 的分工

```text
MemorySSA：沿 memory def-use 找到候选 clobber
AA：判断该 clobber 的 location 与 %p 是否可能重叠
```

仅有 AA 时需要在 CFG/循环中搜索大量 memory instructions；仅有 MemorySSA 时仍需 AA 区分
不同地址。

## 6.6 MustExecute

某指令若每次进入循环都保证执行，再把可能 trap 的指令移到 preheader，有时不会增加执行
次数。判断不能只看它位于 header：异常 edge、提前 exit、不可达和 guaranteed-to-transfer
语义都会影响 must-execute。

优先使用 LLVM `MustExecute` / `ValueTracking` 相关工具，不要自创“块支配 latch 就必执行”
这种不完整规则。

## 6.7 LICM-lite 骨架

```cpp
bool runLICMLite(Loop &L, DominatorTree &DT, LoopInfo &LI,
                 AssumptionCache &AC, TargetLibraryInfo &TLI) {
  BasicBlock *Preheader = L.getLoopPreheader();
  if (!Preheader)
    return false;

  Instruction *InsertBefore = Preheader->getTerminator();
  SmallVector<Instruction *> Candidates;

  for (BasicBlock *BB : L.blocks()) {
    for (Instruction &I : *BB) {
      if (isa<PHINode>(I) || I.isTerminator())
        continue;
      if (!all_of(I.operands(), [&](Use &U) {
            return L.isLoopInvariant(U.get());
          }))
        continue;
      if (I.mayReadOrWriteMemory() || I.mayHaveSideEffects())
        continue; // 第一版保守排除 memory/side effect
      if (!isSafeToSpeculativelyExecute(&I, InsertBefore, &AC, &DT, &TLI))
        continue;
      Candidates.push_back(&I);
    }
  }

  for (Instruction *I : Candidates)
    I->moveBefore(InsertBefore);

  return !Candidates.empty();
}
```

这是面试骨架而不是生产版 LICM。它未处理候选之间的新 invariant 发现顺序、guaranteed
execution、memory promotion、LCSSA、debug records 和 profitability。

## 6.8 高频问答

**问：LICM 为什么需要 preheader？**

答：它是只要进入循环就执行一次、同时支配 header 的唯一规范插入点。

**问：readonly call 可以提升吗？**

答：readonly 只表示不写内存，仍可能读取被循环修改的内存、抛异常、依赖环境或具有其他
限制，需要更完整的 memory 与 execution-safety 证明。

**问：为什么 LICM 后常需 InstCombine/DCE？**

答：移动会暴露常量、重复表达式和 dead PHI，后续清理使最终 IR 简洁。

---

# 7. 专题六：AliasAnalysis 与 MemorySSA

## 7.1 AliasAnalysis 的问题形式

不是问“两个 Value 是否相同”，而是问：

> 两个 pointer 加上各自访问 size 后，代表的内存区间是否可能重叠？

```cpp
MemoryLocation A(PtrA, LocationSize::precise(4));
MemoryLocation B(PtrB, LocationSize::precise(8));
AliasResult R = AA.alias(A, B);
```

| 结果 | 含义 |
|---|---|
| `NoAlias` | 可以证明不重叠 |
| `MayAlias` | 无法证明不重叠 |
| `PartialAlias` | 部分区间重叠 |
| `MustAlias` | 确定指向同一起始位置/按接口定义必 alias |

最重要的面试纠错：`MayAlias` 不是“确定相同”，而是保守的“不知道”。

## 7.2 为什么 pointer identity 不够

```llvm
%p1 = getelementptr i32, ptr %base, i64 %i
%p2 = getelementptr i8, ptr %base, i64 %byte_offset
```

`%p1 != %p2` 这两个 SSA Value 指针不同，但运行时地址可能相同。反过来，相同 base 的两个
固定不重叠字段可能 NoAlias。

## 7.3 Mod/Ref

对 call 与 memory location：

```cpp
ModRefInfo MRI = AA.getModRefInfo(Call, Loc);

if (isNoModRef(MRI)) { /* 不读也不写该位置 */ }
if (isRefSet(MRI))   { /* 可能读 */ }
if (isModSet(MRI))   { /* 可能写 */ }
```

| 结果 | 对 load CSE | 对 DSE |
|---|---|---|
| NoModRef | 不构成 clobber/观察 | 可越过该 call |
| Ref only | 不修改 load 值 | 可能观察 earlier store |
| Mod only | 可能修改 load 值 | 不一定观察 earlier store |
| ModRef | 两者都要保守 |

## 7.4 MemorySSA 模型

```llvm
entry:
  store i32 1, ptr %p
  %a = load i32, ptr %p
  store i32 2, ptr %q
  %b = load i32, ptr %p
```

概念 MemorySSA：

```text
liveOnEntry
    │
    ▼
1 = MemoryDef(store 1 → p)
    │
    ├── MemoryUse(load a ← p)
    │
    ▼
2 = MemoryDef(store 2 → q)
    │
    └── MemoryUse(load b ← p)
```

如果 AA 证明 `p` 与 `q` NoAlias，walker 查 `%b` 的 clobber 时可以越过 MemoryDef 2，找到
MemoryDef 1，因此 `%b` 可能复用 `%a`。

## 7.5 MemoryPhi

```llvm
entry:
  br i1 %c, label %a, label %b
a:
  store i32 1, ptr %p
  br label %merge
b:
  store i32 2, ptr %p
  br label %merge
merge:
  %v = load i32, ptr %p
```

```text
MemoryDef(store 1) ─┐
                    ├→ MemoryPhi(merge) → MemoryUse(load)
MemoryDef(store 2) ─┘
```

MemoryPhi 合并的是抽象内存状态，不产生普通 LLVM IR 指令。

## 7.6 常用接口

```cpp
AAResults &AA = FAM.getResult<AAManager>(F);
MemorySSA &MSSA = FAM.getResult<MemorySSAAnalysis>(F).getMSSA();

MemoryAccess *MA = MSSA.getMemoryAccess(I);
MemorySSAWalker *Walker = MSSA.getWalker();
MemoryAccess *Clobber = Walker->getClobberingMemoryAccess(MA);

MSSA.print(errs());
```

`getMemoryAccess(I)` 可能返回 null，因为纯计算没有 MemorySSA 节点。block 上的查询主要用于
取得该块的 MemoryPhi。

## 7.7 MemorySSA 不是什么

- 不是每个内存对象各自一套完整 SSA。
- 不是 alias analysis 的替代品。
- 不是 Java/C++ memory model 的完整证明器。
- 不是修改 IR 后会自动更新的结构。
- defining access 不一定就是最终精确 clobber，walker 才会进一步优化查询。

## 7.8 MemorySSAUpdater

插入/删除 memory instruction 或修改 CFG 后：

```cpp
MemorySSAUpdater MSSAU(&MSSA);
```

然后使用对应的 create/insert/remove/move/update APIs，或把 `&MSSAU` 传给
`BasicBlockUtils`。接口前置条件较多，应以当前
`llvm/Analysis/MemorySSAUpdater.h` 为准。

如果不能证明维护完整，返回不保留 `MemorySSAAnalysis`。

## 7.9 高频问答

**问：为什么 MemorySSA 只有一个 memory version 链仍然有用？**

答：它先用 SSA/CFG 结构快速缩小候选定义，再由 AA 跳过不 alias 的 clobber；比每次从
load 沿 CFG 扫描所有 store 更高效。

**问：MemoryUse 的 defining access 是 MemoryDef 2，但 AA 证明 NoAlias，怎么办？**

答：使用 walker 继续向上找实际 clobber，不能把初始 defining access 直接当最终答案。

**问：volatile load 可以用 MemorySSA 消除吗？**

答：不能仅凭值相同消除。volatile access 自身是可观察事件，MemorySSA 结构不能覆盖其
全部语言语义限制。

---

# 8. 专题七：GVN、CSE 与值等价

## 8.1 CSE 的基本条件

```llvm
%a = add i32 %x, %y
...
%b = add i32 %x, %y
```

用 `%a` 替换 `%b` 通常要求：

- opcode、type、operands 和语义 flags 相容；
- `%a` 支配 `%b` 的 uses；
- 指令没有不可重复/不可消除的副作用；
- 如果读内存，中间无 clobber；
- 替换不改变 poison、异常、convergent 等语义。

## 8.2 EarlyCSE 框架

```text
沿 DominatorTree DFS
  进入节点：push scope
  遍历指令：
    计算 expression key
    若表中已有且 existing dominates current：RAUW
    否则插入表
  遍历 DT children
  离开节点：pop scope
```

简化骨架：

```cpp
struct ExprKey {
  unsigned Opcode;
  Type *Ty;
  SmallVector<Value *, 2> Operands;
  // 真实实现还需 predicate、flags 等。
};

// 算法伪代码：生产实现还需为 ExprKey 提供 DenseMapInfo/哈希与相等判断。
void visit(DomTreeNode *N, ScopedHashTable<ExprKey, Value *> &Table) {
  ScopedHashTableScope Scope(Table);

  for (Instruction &I : make_early_inc_range(*N->getBlock())) {
    if (!isPureCandidate(I))
      continue;
    ExprKey K = makeKey(I);
    if (Value *Old = Table.lookup(K)) {
      I.replaceAllUsesWith(Old);
      I.eraseFromParent();
    } else {
      Table.insert(K, &I);
    }
  }

  for (DomTreeNode *Child : *N)
    visit(Child, Table);
}
```

真实 LLVM 代码使用经过优化的 value numbering、simple values、memory generation 等机制；
该骨架用于面试阐述 dominance scope。

## 8.3 表达式 key 容易漏什么

```text
icmp predicate
fast-math flags
nsw / nuw / exact
GEP inbounds 和 source element type
call target、arguments、attributes、operand bundles
load type、address、alignment、atomic/volatile ordering
constrained floating-point environment
```

只用 opcode + operands 可能把语义不同的指令错误合并。

## 8.4 GVN 与 PHI translation

某表达式的 operand 在 merge 块是 PHI，沿不同 predecessor 反向看时可以翻译成不同值：

```llvm
a:
  %xa = ...
  br label %merge
b:
  %xb = ...
  br label %merge
merge:
  %x = phi i32 [ %xa, %a ], [ %xb, %b ]
  %r = add i32 %x, 1
```

沿 `%a -> merge` 边，`add %x, 1` 对应 `add %xa, 1`；沿 `%b` 边则对应
`add %xb, 1`。这使 PRE/available value reasoning 能识别路径上的已有计算。

## 8.5 PRE

Partial Redundancy Elimination 处理“某些路径已有、某些路径没有”的表达式。

```text
Before:
             entry
             /   \
     a: t=x+y     b
             \   /
      merge: r=x+y     ← 在 a 路径冗余，在 b 路径不冗余

After:
             entry
             /   \
     a: t=x+y     b: u=x+y
             \   /
       merge: p=phi(t,u)
```

PRE 用增加一条路径计算换取 merge 中公共计算消除，需要考虑代码大小、异常/投机安全和
critical edge。

## 8.6 高频问答

**问：`add x,y` 与 `add y,x` 是同一表达式吗？**

答：数学上对整数普通 add 可交换，但 CSE key 未必自行交换 operands；通常由 canonicalization
或 key 规则排序。浮点、flags 和特殊语义必须另外判断。

**问：两个完全相同的 call 能 CSE 吗？**

答：只有在 call 的内存、异常、convergent、willreturn 等属性证明可重复/可消除，且参数、
callee、operand bundles 等语义相同时才可能，不能按文本相同直接合并。

---

# 9. 专题八：新 Pass Manager 与分析失效

## 9.1 四个主要 IR 层级

```text
Module
  └─ LazyCallGraph SCC（CGSCC）
       └─ Function
            └─ Loop
```

每层有自己的 PassManager 和 AnalysisManager。Adaptor 负责把外层 pipeline 映射到内层，
Proxy 负责跨层访问和失效传播。

## 9.2 Function Pass 标准骨架

```cpp
class InterviewPass : public PassInfoMixin<InterviewPass> {
public:
  PreservedAnalyses run(Function &F, FunctionAnalysisManager &FAM) {
    if (F.isDeclaration())
      return PreservedAnalyses::all();

    DominatorTree &DT = FAM.getResult<DominatorTreeAnalysis>(F);
    LoopInfo &LI = FAM.getResult<LoopAnalysis>(F);
    ScalarEvolution &SE = FAM.getResult<ScalarEvolutionAnalysis>(F);
    AAResults &AA = FAM.getResult<AAManager>(F);
    MemorySSA &MSSA = FAM.getResult<MemorySSAAnalysis>(F).getMSSA();

    bool Changed = transform(F, DT, LI, SE, AA, MSSA);
    if (!Changed)
      return PreservedAnalyses::all();

    // 初始实现保守失效，正确后再精确 preserve。
    return PreservedAnalyses::none();
  }
};
```

## 9.3 Loop Pass 标准分析集合

Loop Pass 常通过 `LoopStandardAnalysisResults` 得到：

```cpp
class MyLoopPass : public PassInfoMixin<MyLoopPass> {
public:
  PreservedAnalyses run(Loop &L, LoopAnalysisManager &LAM,
                        LoopStandardAnalysisResults &AR,
                        LPMUpdater &U) {
    DominatorTree &DT = AR.DT;
    LoopInfo &LI = AR.LI;
    ScalarEvolution &SE = AR.SE;
    AAResults &AA = AR.AA;
    TargetTransformInfo &TTI = AR.TTI;
    AssumptionCache &AC = AR.AC;
    MemorySSA *MSSA = AR.MSSA;
    ...
  }
};
```

若删除、增加或重新访问 loop，要通过 `LPMUpdater` 告知 loop pass manager，不能只改
`LoopInfo` 就假设调度器自动知道。

## 9.4 `PreservedAnalyses` 决策表

| 修改 | 通常可考虑保留 | 通常要失效/更新 |
|---|---|---|
| 完全未修改 IR | all | 无 |
| 仅改名称/debug info | 多数语义分析 | 依具体 debug 维护规则 |
| 改算术 operand，CFG 不变 | CFG analyses | SCEV、value/AA 类分析 |
| 插入纯指令，CFG 不变 | 部分 CFG analyses | value-dependent analyses |
| 修改 load/store/call | 可能保留 DT/LI | AA cache、MSSA、SCEV 等 |
| 增删 CFG edge | 很少 | DT、PDT、LI、MSSA、BPI/BFI |
| 内联/克隆整个 CFG | 需专门维护 | 大部分函数/调用图分析 |

“CFG 不变”只足以讨论保留 `CFGAnalyses` 集合，不代表 `PreservedAnalyses::all()`。

## 9.5 过度 preserve 为什么可能 miscompile

```text
Pass A：把 CFG edge E 删除
        但错误 preserve DominatorTree
              │
              ▼
Pass B：读取缓存 DT，以为旧支配关系成立
              │
              ▼
把定义移动/替换到错误位置
              │
              ▼
非法 IR、崩溃，甚至 Release 下 miscompile
```

少 preserve 的主要代价是重算和编译时间；多 preserve 是正确性问题。

## 9.6 自定义 Analysis 骨架

```cpp
class BlockCountAnalysis : public AnalysisInfoMixin<BlockCountAnalysis> {
  friend AnalysisInfoMixin<BlockCountAnalysis>;
  static AnalysisKey Key;

public:
  struct Result {
    unsigned Count;
  };

  Result run(Function &F, FunctionAnalysisManager &) {
    return {static_cast<unsigned>(F.size())};
  }
};

AnalysisKey BlockCountAnalysis::Key;
```

使用：

```cpp
auto &R = FAM.getResult<BlockCountAnalysis>(F);
errs() << "blocks=" << R.Count << '\n';
```

## 9.7 Pipeline 注册骨架

```cpp
PassBuilder PB;
LoopAnalysisManager LAM;
FunctionAnalysisManager FAM;
CGSCCAnalysisManager CGAM;
ModuleAnalysisManager MAM;

PB.registerModuleAnalyses(MAM);
PB.registerCGSCCAnalyses(CGAM);
PB.registerFunctionAnalyses(FAM);
PB.registerLoopAnalyses(LAM);
PB.crossRegisterProxies(LAM, FAM, CGAM, MAM);

FunctionPassManager FPM;
FPM.addPass(InterviewPass());

ModulePassManager MPM;
MPM.addPass(createModuleToFunctionPassAdaptor(std::move(FPM)));
MPM.run(M, MAM);
```

## 9.8 调试框架

```bash
# 每个 pass 后验证 IR
opt -verify-each -passes='my-pass,instcombine' input.ll -disable-output

# 查看 pass 前后 IR
opt -print-before=my-pass -print-after=my-pass \
    -passes='my-pass' input.ll -disable-output

# 查看 manager 调度和分析失效
opt -debug-pass-manager -passes='my-pass' input.ll -disable-output

# 只打印目标函数
opt -filter-print-funcs=foo -print-after=my-pass \
    -passes='my-pass' input.ll -disable-output

# 验证和打印支配树
opt -passes='verify<domtree>,print<domtree>' input.ll -disable-output
```

C++ 内部：

```cpp
#define DEBUG_TYPE "interview-pass"
LLVM_DEBUG(dbgs() << "candidate: " << I << '\n');

assert(!verifyFunction(F, &errs()) && "invalid IR after transform");
```

## 9.9 高频问答

**问：为什么分析由 manager 缓存，而不是 pass 自己 new？**

答：多个 pass 能共享昂贵结果，manager 统一按 IR unit 和 preserved information 管理生命
周期、按需计算和失效。

**问：Module Pass 如何访问 FunctionAnalysisManager？**

答：通过 `FunctionAnalysisManagerModuleProxy`；更常见设计是把 function transform 放到
Function Pass，并用 adaptor 加入 Module pipeline。

**问：`getResult` 和 `getCachedResult` 区别？**

答：前者必要时计算分析，后者只返回已有缓存，不主动运行；跨层访问时尤其要遵守 manager
关于是否允许触发计算的规则。

---

# 10. 综合手写题

## 10.1 题一：统计循环并打印结构

### 要求

打印每个 loop 的 depth、header、preheader、latch、exiting blocks 和 exit blocks。

### 骨架

```cpp
static void printLoop(raw_ostream &OS, Loop &L) {
  OS << "depth=" << L.getLoopDepth()
     << " header=" << L.getHeader()->getName() << '\n';

  if (BasicBlock *PH = L.getLoopPreheader())
    OS << "  preheader=" << PH->getName() << '\n';
  else
    OS << "  preheader=<none>\n";

  if (BasicBlock *Latch = L.getLoopLatch())
    OS << "  unique latch=" << Latch->getName() << '\n';
  else
    OS << "  unique latch=<none or multiple>\n";

  SmallVector<BasicBlock *> Exiting;
  L.getExitingBlocks(Exiting);
  for (BasicBlock *BB : Exiting)
    OS << "  exiting=" << BB->getName() << '\n';

  SmallVector<BasicBlock *> Exits;
  L.getExitBlocks(Exits);
  for (BasicBlock *BB : Exits)
    OS << "  exit=" << BB->getName() << '\n';

  for (Loop *Sub : L.getSubLoops())
    printLoop(OS, *Sub);
}
```

### 追问

- 为什么 `getLoopLatch()` 返回 null 不等于无 latch？
- 如何按最内层到最外层遍历？
- 打印型 Pass 应返回什么 preserved analyses？——`all()`。

## 10.2 题二：验证所有 use 的 dominance

```cpp
static bool verifyUses(Value *Def, DominatorTree &DT) {
  bool Valid = true;
  for (Use &U : Def->uses()) {
    if (!DT.dominates(Def, U)) {
      errs() << "non-dominated use: " << *U.getUser() << '\n';
      Valid = false;
    }
  }
  return Valid;
}
```

### 追问

- 为什么参数/常量也要考虑？
- 为什么不能只用 `DT.dominates(DefI, UserI)`？
- 如果 User 是 ConstantExpr 而不是 Instruction 怎么办？

## 10.3 题三：写 LICM-lite

最低合格回答：

1. 检查 `getLoopPreheader()`。
2. 找 operands invariant 的纯指令。
3. 检查 `isSafeToSpeculativelyExecute`。
4. 第一版拒绝所有 memory instruction。
5. 收集后移动，避免遍历失效。
6. 说明生产版本还需 must-execute、MSSA/AA、LCSSA、profitability。

### 测试矩阵

| 用例 | 是否移动 |
|---|---|
| invariant `add` | 是 |
| variant operand `add` | 否 |
| 可能除零的 `sdiv` | 否，除非额外证明 |
| invariant pointer load、loop 中有 MayAlias store | 否 |
| volatile load | 否 |
| loop 可能零次、可能 trap 指令 | 否 |
| 无 preheader | 先规范化或拒绝 |

## 10.4 题四：基于 MemorySSA 判断重复 load

回答流程：

```text
取得 load 对应 MemoryUse
       ↓
使用 walker 找 clobbering access
       ↓
找到支配当前 load 的 earlier load/value？
       ↓
比较 MemoryLocation 和类型/atomic/volatile 属性
       ↓
AA 证明中间 defs 不 clobber
       ↓
earlier value 支配精确 use
       ↓
RAUW + 删除 + 更新/失效 MemorySSA
```

必须主动说出：MemorySSA 节点相同不自动保证两个 load 的全部 IR 语义相同。

## 10.5 题五：删除重复 bounds check

### 识别框架

```text
Check 1: 0 <= i && i < len
        success edge
             │ dominates
             ▼
Check 2: 0 <= i && i < len
```

### 证明清单

- `i` 和 `len` 在两个位置是否为同一值或可证明同范围？
- Check 1 的 success edge 是否支配 Check 2？
- 中间是否有改变 array length/identity 解释的运行时事件？
- signed/unsigned compare 与 Java index 语义是否一致？
- check failure 的异常类型和先后顺序是否被改变？
- 若循环内消除，SCEV 能否证明整个 iteration domain？
- 修改 CFG 后 PHI、DT、LI、MSSA 是否维护？

### 高级追问

若在 loop preheader 合并成一次 range check，需要证明整个 SCEV recurrence 的最小/最大值，
还要处理 overflow。仅检查第一轮和最后一轮可能因整数 wraparound 不正确。

## 10.6 题六：分析 O2-only miscompile

```text
1. 固定输入 IR、命令、LLVM build、目标 triple/DataLayout
2. 加 -verify-each，找第一个破坏 IR 的 pass
3. 若 verifier 不报错但结果错，缩小 pipeline 并比较 before/after
4. 用 -filter-print-funcs 限定函数
5. 使用 llvm-reduce 缩减 IR
6. 开 assertions，验证 DT/MSSA/LCSSA
7. 检查分析过度 preserve
8. 检查 poison/nowrap/inbounds、speculation、alias、异常和迭代器失效
9. 写最小正例与负例 FileCheck 回归
```

---

# 11. 面试速背框架

## 11.1 一分钟回答表

| 专题 | 一分钟必须说出的关键词 |
|---|---|
| SSA | 单定义、def-use、PHI、dominance、memory 不天然 SSA |
| mem2reg | promotable alloca、IDF 放 PHI、DT rename、地址逃逸拒绝 |
| DominatorTree | all paths、idom、PHI edge use、CFG 修改后更新 |
| LoopInfo | natural loop、header/latch/preheader、exit、嵌套层级 |
| LoopSimplify | preheader、规范 latch、dedicated exits |
| LCSSA | loop 内定义经 exit PHI 给外部使用 |
| SCEV | AddRec、backedge count、trip count、nowrap、CouldNotCompute |
| LICM | invariant、preheader、speculation/must-execute、AA/MSSA |
| AA | MemoryLocation、No/May/Partial/MustAlias、ModRef |
| MemorySSA | MemoryDef/Use/Phi、walker、AA、Updater |
| GVN | value number、dominance、memory clobber、PHI translation/PRE |
| 新 PM | IR levels、AnalysisManager、proxy/adaptor、PreservedAnalyses |

## 11.2 专题之间的“桥接句”

面试中主动使用这些句子，能把零散知识连成体系：

- “mem2reg 的 PHI 放置依赖 dominance frontier，rename 则沿 dominator tree 完成。”
- “LoopInfo 提供结构，SCEV 提供值随迭代演化，二者职责不同。”
- “operand invariant 只是 LICM 的第一层条件，memory invariant 和 execution safety 需要
  AA、MemorySSA、MustExecute/ValueTracking。”
- “MemorySSA 缩小 CFG 上的 clobber 搜索，AA 判断具体 location 是否重叠。”
- “GVN 找到等价值后仍需 dominance 保证 replacement 对每条 use 可用。”
- “所有增量维护最终都要通过 PreservedAnalyses 向新 PM 做出正确承诺。”

## 11.3 最常见错误回答

| 错误说法 | 正确说法 |
|---|---|
| mem2reg 把变量分配到 CPU 寄存器 | 它把栈槽提升成 SSA value |
| PHI 在 merge 块读取全部输入 | 每个 incoming use 位于相应 edge |
| DT 就是把 CFG 去环 | DT 表示支配关系，edge 是 idom，不等于 CFG edge |
| 有 LoopInfo 就有 preheader | preheader 是 LoopSimplify form 的额外性质 |
| SCEV 一定能算出 trip count | 复杂循环会 CouldNotCompute |
| operands invariant 就能 LICM | 还需投机、内存、异常、放置和收益证明 |
| 两个指针值不同所以 NoAlias | 不同 SSA pointer 仍可能指向重叠内存 |
| MemorySSA 可以替代 AA | 两者配合，职责不同 |
| 没改 CFG 就 preserve all | SCEV、AA 等仍可能因值/内存修改失效 |
| verifier 通过就说明优化等价 | verifier 只验证 IR 结构/规则，不证明变换语义等价 |

## 11.4 白板画图顺序

被要求解释循环优化时，建议按这个顺序画：

```text
1. CFG：preheader → header → body → latch ↺，并画 exits
2. DT：说明 header 支配 latch，确认 backedge/natural loop
3. SSA：在 header 画 IV PHI
4. SCEV：写 {Start,+,Step}<L>
5. MemorySSA：为 loop 中 load/store 画 Def/Use/Phi
6. LICM：分别说明 invariant、no-clobber、must-execute
7. PM：说明移动后哪些分析保留，CFG 变化时怎样更新
```

## 11.5 最终自测题

1. 不看资料画出 diamond CFG 的 dominance frontier 和 mem2reg PHI。
2. 解释为什么 PHI operand dominance 必须查询 `Use`。
3. 写出一个 `getLoopLatch() == nullptr` 但循环确实有 backedge 的 CFG。
4. 区分 exiting block、exit block 和 dedicated exit。
5. 把 `i = 3; i += 2` 写成 SCEV AddRec，并说明第 k 次值。
6. 给出 operands invariant 但不能 hoist 的 `sdiv` 和 load 反例。
7. 解释 MemorySSA walker 为什么仍需 AA。
8. 写出 CSE expression key 至少要包含的五类语义信息。
9. 给出一个“CFG 不变但 SCEV 失效”的 IR 修改。
10. 用五分钟完整讲述一个 bounds-check elimination Pass 的合法性证明。

---

## 附录：配套实验命令

```bash
OPT=jeandle-llvm/build-release/bin/opt

# SSA / mem2reg
$OPT -S -passes='mem2reg' input.ll -o -

# Loop、DT、SCEV、MemorySSA
$OPT -passes='print<loops>' input.ll -disable-output
$OPT -passes='print<domtree>' input.ll -disable-output
$OPT -passes='print<scalar-evolution>' input.ll -disable-output
$OPT -passes='print<memoryssa>' input.ll -disable-output

# LICM 前后
$OPT -passes='loop-simplify,lcssa,loop(licm)' \
  -print-before=licm -print-after=licm input.ll -disable-output

# 完整验证
$OPT -verify-each -passes='default<O2>' input.ll -disable-output

# 当前 LLVM 版本支持的 pass 名称
$OPT --print-passes
```

建议每个专题自己写一个 positive case 和一个只改变单一条件的 negative case。能解释
“为什么负例必须拒绝”，比只记住某个成功优化的 After IR 更能体现编译器理解深度。
