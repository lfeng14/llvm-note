# LLVM 常用 C++ 接口手册（LLVM 22 / Jeandle 版）

> 面向阅读、调试和编写 LLVM IR Pass。本文依据本仓库 `jeandle-llvm` 中的
> LLVM 22.0.0git 源码整理。接口在其他 LLVM 大版本中可能略有差异。
>
> 打印建议：A4 双面、等宽代码字体 8.5–9 pt；先打印第 1、2、4、5、14、18 节，
> 日常使用频率最高。

## 目录

1. 一页总览
2. IR 对象模型与所有权
3. 类型、常量、DataLayout
4. Module / Function / BasicBlock / Instruction
5. 遍历 IR 与 CFG
6. 修改 IR 的安全规则
7. IRBuilder 常用接口
8. DominatorTree 与 PostDominatorTree
9. LoopInfo 与循环变换
10. 新 Pass Manager 与分析管理
11. AliasAnalysis、MemorySSA 与内存推理
12. ScalarEvolution 与 ValueTracking
13. 分支概率、频率及其他常用分析
14. 调试、打印、验证与 `opt`
15. LLVM ADT、错误处理和命令行
16. Call、Intrinsic、Metadata 和 Attribute
17. 常见任务配方
18. 高频接口速查表与踩坑清单

---

## 1. 一页总览

### 1.1 最重要的对象关系

```text
LLVMContext
  └─ Module                       一个编译单元
      ├─ Function                 函数；同时也是 GlobalValue 和 Value
      │   ├─ Argument             形式参数
      │   └─ BasicBlock           基本块；一串 Instruction
      │       └─ Instruction      指令；大多既是 User，又可能产生 Value
      │           └─ operand Use  对其他 Value 的引用
      ├─ GlobalVariable
      └─ GlobalAlias / GlobalIFunc
```

核心关系：

- `Value` 是“一个 SSA 值”；常量、参数、指令、基本块、函数都是 `Value`。
- `User` 是“使用其他 Value 的 Value”；指令和常量表达式通常是 `User`。
- `Use` 是一条精确的“使用边”，连接 `User` 与被使用的 `Value`。
- `Instruction` 由 `BasicBlock` 拥有，`BasicBlock` 由 `Function` 拥有，
  `Function` 由 `Module` 拥有。
- LLVM 广泛使用裸指针表达非拥有关系。不要对 IR 节点手工 `delete`。
- LLVM IR 使用 SSA：普通值单赋值；控制流汇合处由 `PHINode` 合并。

### 1.2 最常用 include

```cpp
#include "llvm/IR/BasicBlock.h"
#include "llvm/IR/Constants.h"
#include "llvm/IR/DataLayout.h"
#include "llvm/IR/Dominators.h"
#include "llvm/IR/Function.h"
#include "llvm/IR/IRBuilder.h"
#include "llvm/IR/Instructions.h"
#include "llvm/IR/IntrinsicInst.h"
#include "llvm/IR/Module.h"
#include "llvm/IR/PatternMatch.h"
#include "llvm/IR/Verifier.h"

#include "llvm/Analysis/AliasAnalysis.h"
#include "llvm/Analysis/AssumptionCache.h"
#include "llvm/Analysis/LoopInfo.h"
#include "llvm/Analysis/Loads.h"
#include "llvm/Analysis/MemorySSA.h"
#include "llvm/Analysis/PostDominators.h"
#include "llvm/Analysis/ScalarEvolution.h"
#include "llvm/Analysis/TargetLibraryInfo.h"
#include "llvm/Analysis/TargetTransformInfo.h"
#include "llvm/Analysis/ValueTracking.h"

#include "llvm/Passes/PassBuilder.h"
#include "llvm/Passes/PassPlugin.h"
#include "llvm/Support/Debug.h"
#include "llvm/Support/raw_ostream.h"
#include "llvm/Transforms/Utils/BasicBlockUtils.h"
#include "llvm/Transforms/Utils/Local.h"
```

### 1.3 高频操作十行版

```cpp
Function &F = ...;
Module *M = F.getParent();
LLVMContext &Ctx = F.getContext();
const DataLayout &DL = M->getDataLayout();

for (BasicBlock &BB : F)
  for (Instruction &I : BB)
    errs() << I << '\n';

auto &DT = FAM.getResult<DominatorTreeAnalysis>(F);
auto &LI = FAM.getResult<LoopAnalysis>(F);
bool ABeforeB = DT.dominates(A, B);
Loop *L = LI.getLoopFor(BB);
```

### 1.4 LLVM 风格的运行时类型判断

```cpp
if (auto *CI = dyn_cast<CallInst>(&I)) { /* I 是 CallInst */ }
if (isa<PHINode>(I)) { /* 只判断 */ }
auto *BO = cast<BinaryOperator>(&I);     // 断言：必须是该类型
auto *LI = dyn_cast_or_null<LoadInst>(MaybeNull);
```

经验：不能百分百确定时用 `dyn_cast`；逻辑上必然成立且希望断言暴露 bug 时用
`cast`。LLVM RTTI 不依赖 C++ `dynamic_cast`。

---

## 2. IR 对象模型与所有权

### 2.1 `Value`、`User`、`Use`

常用 `Value` API：

```cpp
Type *Ty = V->getType();
StringRef Name = V->getName();
V->setName("new.name");
bool HasName = V->hasName();

unsigned N = V->getNumUses();
bool None = V->use_empty();
bool One = V->hasOneUse();
bool NUses = V->hasNUses(2);

for (User *U : V->users()) { ... }
for (Use &U : V->uses()) {
  User *Owner = U.getUser();
  unsigned OperandNo = U.getOperandNo();
}
```

`users()` 只给使用者；`uses()` 还能区分同一使用者的具体 operand。后者对 PHI、
同一指令多次使用同一个值、支配关系判断尤其重要。

替换使用：

```cpp
Old->replaceAllUsesWith(New);                   // RAUW：替换所有 use
Old->replaceUsesWithIf(New, [](Use &U) { ... });
U.set(New);                                    // 只替换某条 use
```

约束：`Old` 与 `New` 的类型必须相同；RAUW 不删除 `Old`；不能造成非法自引用。

### 2.2 ownership 与生命周期

- 插入基本块后的指令由基本块持有。
- `IRBuilder::Create*` 通常立即插入到当前 insertion point。
- `Instruction::removeFromParent()` 只摘链，不销毁，调用者要负责后续归属。
- `Instruction::eraseFromParent()` 摘链并销毁，指针随即失效。
- `BasicBlock::eraseFromParent()` 和 `Function::eraseFromParent()` 同理。
- `dropAllReferences()` 只断开 operand 引用，常用于拆环，不等于删除。
- `Module`、`LLVMContext` 的生命周期必须覆盖其下所有 IR 对象。

### 2.3 SSA 和 PHI 的正确理解

```llvm
entry:
  br i1 %cond, label %then, label %else
then:
  %a = add i32 %x, 1
  br label %merge
else:
  %b = sub i32 %x, 1
  br label %merge
merge:
  %v = phi i32 [ %a, %then ], [ %b, %else ]
```

PHI 的 use 发生在对应前驱边末端，而不是概念上的 `merge` 块开头。因此判断
`%a` 是否支配 PHI 的某个 operand，应使用 `DT.dominates(Def, Use)`，不要简单写成
`DT.dominates(Def, Phi)`。

---

## 3. 类型、常量、DataLayout

### 3.1 常用类型

```cpp
LLVMContext &C = ...;
Type *VoidTy = Type::getVoidTy(C);
IntegerType *I1  = Type::getInt1Ty(C);
IntegerType *I8  = Type::getInt8Ty(C);
IntegerType *I32 = Type::getInt32Ty(C);
IntegerType *I64 = Type::getInt64Ty(C);
Type *F32 = Type::getFloatTy(C);
Type *F64 = Type::getDoubleTy(C);
PointerType *PtrTy = PointerType::get(C, 0); // address space 0
ArrayType *A10I8 = ArrayType::get(I8, 10);
VectorType *V4I32 = FixedVectorType::get(I32, 4);
StructType *Pair = StructType::get(C, {I32, I64});
FunctionType *FT = FunctionType::get(I32, {I32, I32}, false);
```

LLVM 新版本使用 opaque pointer：通常只有 `ptr` 和 address space，不再从指针类型
获得 pointee type。load/GEP/call 等接口必须显式提供被访问类型。

```cpp
bool IsInt = Ty->isIntegerTy();
bool IsI32 = Ty->isIntegerTy(32);
bool IsPtr = Ty->isPointerTy();
unsigned Bits = cast<IntegerType>(Ty)->getBitWidth();
```

### 3.2 常量与 `APInt` / `APFloat`

```cpp
ConstantInt *Zero = ConstantInt::get(I32, 0);
ConstantInt *MinusOne = ConstantInt::getSigned(I32, -1);
Constant *Null = ConstantPointerNull::get(PtrTy);
Constant *Undef = UndefValue::get(I32);
Constant *Poison = PoisonValue::get(I32);
Constant *ZeroInit = ConstantAggregateZero::get(Pair);

APInt Wide(128, "12345678901234567890", 10);
ConstantInt *CWide = ConstantInt::get(C, Wide);
```

不要混淆：

- `undef`：每次使用可任选一个值。
- `poison`：延迟传播的未定义行为，进入某些敏感操作会触发 UB。
- `freeze`：把 poison/undef 固定成某个任意但稳定的值。
- `null`：确定的零指针，不等于 undef。

读取整数常量：

```cpp
if (auto *CI = dyn_cast<ConstantInt>(V)) {
  const APInt &N = CI->getValue();
  uint64_t U = CI->getZExtValue(); // 仅在能放下时使用
  int64_t S = CI->getSExtValue();
  bool IsZero = CI->isZero();
  bool IsOne = CI->isOne();
}
```

### 3.3 `DataLayout`

不要根据宿主机 `sizeof(void *)` 推测目标布局。JIT 也应以 `Module` 的
`DataLayout` 为准。

```cpp
const DataLayout &DL = M.getDataLayout();
TypeSize Bytes = DL.getTypeStoreSize(Ty);
TypeSize AllocBytes = DL.getTypeAllocSize(Ty);
Align ABIAlign = DL.getABITypeAlign(Ty);
unsigned PtrBits = DL.getPointerSizeInBits(AddressSpace);
IntegerType *IntPtrTy = DL.getIntPtrType(C, AddressSpace);

StructLayout const *SL = DL.getStructLayout(ST);
uint64_t Offset = SL->getElementOffset(Index);
```

`TypeSize` 可能是 scalable，不能总是假设可直接转成普通整数。固定大小时先确认
`isScalable() == false`，或使用语义合适的接口。

---

## 4. Module / Function / BasicBlock / Instruction

### 4.1 `Module`

```cpp
Function *F = M.getFunction("foo");
GlobalVariable *G = M.getGlobalVariable("g");
GlobalValue *GV = M.getNamedValue("name");

for (Function &F : M) { ... }
for (GlobalVariable &G : M.globals()) { ... }

M.print(errs(), nullptr);
M.dump(); // 调试构建常用；正式代码优先 print
```

声明与定义：

```cpp
bool Decl = F->isDeclaration();
bool Empty = F->empty();
StringRef Name = F->getName();
CallingConv::ID CC = F->getCallingConv();
```

创建或取得函数声明：

```cpp
FunctionCallee Callee = M.getOrInsertFunction("foo", FT);
Value *CalleeValue = Callee.getCallee();
FunctionType *CalleeTy = Callee.getFunctionType();
```

不要无条件把 `getCallee()` `cast<Function>`：同名符号类型不一致时可能经过
bitcast，`FunctionCallee` 正是为了容纳这种情况。

### 4.2 `Function`

```cpp
Module *M = F.getParent();
LLVMContext &C = F.getContext();
FunctionType *FT = F.getFunctionType();
Type *RetTy = F.getReturnType();
unsigned ArgCount = F.arg_size();

for (Argument &A : F.args()) {
  unsigned No = A.getArgNo();
  Type *Ty = A.getType();
}

BasicBlock &Entry = F.getEntryBlock();
Instruction *EntryIP = &*Entry.getFirstInsertionPt();
```

`getEntryBlock()` 要求函数有定义。`getFirstInsertionPt()` 会跳过 PHI 和其他必须位于
块首的指令，是插入普通指令时比 `begin()` 更安全的选择。

### 4.3 `BasicBlock`

```cpp
Function *Parent = BB.getParent();
Instruction *Term = BB.getTerminator();
BasicBlock::iterator First = BB.getFirstNonPHIIt();
BasicBlock::iterator FirstNonPhiDbg = BB.getFirstNonPHIOrDbg();

bool IsEntry = BB.isEntryBlock();
bool HasNPred = BB.hasNPredecessors(2);
BasicBlock *SinglePred = BB.getSinglePredecessor();
BasicBlock *UniquePred = BB.getUniquePredecessor();
BasicBlock *SingleSucc = BB.getSingleSuccessor();
BasicBlock *UniqueSucc = BB.getUniqueSuccessor();
```

“single” 通常按 CFG edge 数量；“unique” 按不同块数量。一个 switch 可能从同一块
以多条边到达同一目标，二者并不总相同。

```cpp
for (BasicBlock *Pred : predecessors(&BB)) { ... }
for (BasicBlock *Succ : successors(&BB)) { ... }
unsigned NP = pred_size(&BB);
unsigned NS = succ_size(&BB);
```

### 4.4 `Instruction`

```cpp
BasicBlock *BB = I.getParent();
Function *F = I.getFunction();
Module *M = I.getModule();
unsigned Opcode = I.getOpcode();
StringRef OpcodeName = I.getOpcodeName();

Instruction *Prev = I.getPrevNode();
Instruction *Next = I.getNextNode();
bool MayRead = I.mayReadFromMemory();
bool MayWrite = I.mayWriteToMemory();
bool SideEffects = I.mayHaveSideEffects();
bool Terminator = I.isTerminator();
```

operand：

```cpp
for (Use &Op : I.operands()) {
  Value *V = Op.get();
}
Value *V0 = I.getOperand(0);
I.setOperand(0, NewV);
```

调试位置：

```cpp
DebugLoc DL = I.getDebugLoc();
if (DL) {
  unsigned Line = DL.getLine();
  unsigned Col = DL.getCol();
}
NewI->setDebugLoc(I.getDebugLoc());
```

---

## 5. 遍历 IR 与 CFG

### 5.1 基本遍历

```cpp
for (Function &F : M)
  for (BasicBlock &BB : F)
    for (Instruction &I : BB)
      visit(I);
```

只遍历指令：

```cpp
#include "llvm/IR/InstIterator.h"
for (Instruction &I : instructions(F)) { ... }
```

按类型过滤：

```cpp
for (Instruction &I : instructions(F)) {
  if (auto *LI = dyn_cast<LoadInst>(&I)) { ... }
  else if (auto *SI = dyn_cast<StoreInst>(&I)) { ... }
  else if (auto *CB = dyn_cast<CallBase>(&I)) { ... }
}
```

调用统一优先匹配 `CallBase`，它覆盖 `CallInst`、`InvokeInst`、`CallBrInst`。

### 5.2 CFG 的 DFS、逆后序

```cpp
#include "llvm/ADT/DepthFirstIterator.h"
#include "llvm/ADT/PostOrderIterator.h"

for (BasicBlock *BB : depth_first(&F.getEntryBlock())) { ... }

ReversePostOrderTraversal<Function *> RPOT(&F);
for (BasicBlock *BB : RPOT) { ... }
```

注意：从 entry 出发的图遍历不会访问 unreachable blocks，而 `for (BB : F)` 会。

### 5.3 删除时的遍历模式

不要在 range-for 中直接删除当前指令。常用做法：

```cpp
for (BasicBlock &BB : F)
  for (Instruction &I : make_early_inc_range(BB))
    if (shouldDelete(I))
      I.eraseFromParent();
```

若判断逻辑可能访问刚删对象，先收集再删更稳妥：

```cpp
SmallVector<Instruction *> Dead;
for (Instruction &I : instructions(F))
  if (isInstructionTriviallyDead(&I))
    Dead.push_back(&I);
for (Instruction *I : Dead)
  RecursivelyDeleteTriviallyDeadInstructions(I);
```

### 5.4 遍历 use-def 链

向上看定义：读取 instruction operands。向下看使用：读取 `users()` / `uses()`。

```cpp
for (User *U : V->users())
  if (auto *I = dyn_cast<Instruction>(U)) { ... }

for (Use &U : V->uses()) {
  auto *I = dyn_cast<Instruction>(U.getUser());
  if (!I) continue;
  errs() << "operand #" << U.getOperandNo() << " of " << *I << '\n';
}
```

修改 use-list 时用 early increment 或先收集，避免迭代器失效。

---

## 6. 修改 IR 的安全规则

### 6.1 插入、移动、克隆、替换、删除

```cpp
NewI->insertBefore(&OldI);
NewI->insertAfter(&OldI);
I.moveBefore(InsertPos);
I.moveAfter(&Other);

Instruction *Copy = I.clone(); // clone 尚未插入；名称/调试信息按需处理
Copy->insertBefore(&I);

OldI.replaceAllUsesWith(NewV);
OldI.eraseFromParent();
```

删除前必须保证没有仍然存在的 use，或先 RAUW。`eraseFromParent()` 返回的下一个迭代器
可用于某些手写循环。

### 6.2 CFG 修改的连锁责任

增加、删除或重定向 CFG edge 时，至少检查：

1. 目标块每个 PHI 的 incoming block/value 是否同步。
2. `DominatorTree` / `PostDominatorTree` 是否更新或声明失效。
3. `LoopInfo` 是否更新或声明失效。
4. `MemorySSA` 是否更新或声明失效。
5. 分支权重、调试信息、profile metadata 是否仍有意义。

优先使用 `Transforms/Utils` 中的公共工具，它们通常能协助维护 PHI：

```cpp
DomTreeUpdater DTU(DT, DomTreeUpdater::UpdateStrategy::Lazy);
BasicBlock *NewBB = SplitBlock(BB, SplitBefore, &DTU, &LI);
BasicBlock *ThenBB = SplitEdge(From, To, &DT, &LI);
bool Changed = MergeBlockIntoPredecessor(BB, &DTU, &LI);
ReplaceInstWithInst(Old, New);
```

具体 overload 会随版本变化，使用前查看当前头文件
`llvm/Transforms/Utils/BasicBlockUtils.h`。

### 6.3 `PHINode`

```cpp
auto *PN = PHINode::Create(Ty, 2, "merged", InsertBefore);
PN->addIncoming(V1, Pred1);
PN->addIncoming(V2, Pred2);

unsigned N = PN->getNumIncomingValues();
Value *V = PN->getIncomingValue(i);
BasicBlock *B = PN->getIncomingBlock(i);
int Idx = PN->getBasicBlockIndex(Pred);
Value *PV = PN->getIncomingValueForBlock(Pred);
PN->setIncomingValue(i, NewV);
PN->setIncomingBlock(i, NewPred);
PN->removeIncomingValue(Pred, /*DeletePHIIfEmpty=*/false);
```

同一前驱块可能对应多条 CFG edge；维护 PHI 时不要天然假设 block 唯一。

### 6.4 常用局部化简

```cpp
#include "llvm/Transforms/Utils/Local.h"

if (isInstructionTriviallyDead(I))
  RecursivelyDeleteTriviallyDeadInstructions(I);

bool Changed = SimplifyInstructionsInBlock(&BB);
bool Removed = EliminateUnreachableBlocks(F);
```

`mayHaveSideEffects()` 为 false 并不自动等价于“现在可删”；最好使用 LLVM 已有的
deadness/simplify 工具，并提供需要的分析信息。

---

## 7. IRBuilder 常用接口

### 7.1 insertion point

```cpp
IRBuilder<> B(C);
B.SetInsertPoint(&BB);               // 插到 terminator 前/块尾的合法位置
B.SetInsertPoint(&I);                // 插到 I 前
B.SetInsertPoint(&BB, BB.begin());

IRBuilder<>::InsertPoint Saved = B.saveIP();
B.SetInsertPoint(OtherBB);
...
B.restoreIP(Saved);

IRBuilderBase::InsertPointGuard Guard(B); // 作用域退出自动恢复
```

复制调试位置：

```cpp
B.SetCurrentDebugLocation(I.getDebugLoc());
```

### 7.2 算术、比较、选择、转换

```cpp
Value *Sum = B.CreateAdd(X, Y, "sum");
Value *NSW = B.CreateNSWAdd(X, Y);
Value *UDiv = B.CreateUDiv(X, Y);
Value *And = B.CreateAnd(X, Mask);

Value *Eq = B.CreateICmpEQ(X, Y);
Value *SLt = B.CreateICmpSLT(X, Y);
Value *ULt = B.CreateICmpULT(X, Y);
Value *FOlt = B.CreateFCmpOLT(X, Y);
Value *Chosen = B.CreateSelect(Cond, TrueV, FalseV);

Value *Z = B.CreateZExt(V, I64);
Value *S = B.CreateSExt(V, I64);
Value *T = B.CreateTrunc(V, I8);
Value *P = B.CreateIntToPtr(Int, PtrTy);
Value *N = B.CreatePtrToInt(Ptr, I64);
Value *BC = B.CreateBitCast(V, DestTy);
```

有符号/无符号主要由操作语义决定，不是整数类型本身的属性。`i32` 没有“signed
i32”和“unsigned i32”之分。

### 7.3 内存与 GEP

```cpp
AllocaInst *Slot = B.CreateAlloca(I32, nullptr, "slot");
LoadInst *L = B.CreateLoad(I32, Slot, "v");
StoreInst *S = B.CreateStore(ValueToStore, Slot);
L->setAlignment(Align(4));
S->setVolatile(true);

Value *Elem = B.CreateGEP(ElementTy, BasePtr, Indices, "elem.ptr");
Value *InBounds = B.CreateInBoundsGEP(ElementTy, BasePtr, Indices);
Value *StructField = B.CreateStructGEP(StructTy, StructPtr, FieldNo);
```

GEP 只计算地址，不读取内存。`inbounds` 是强语义承诺，不能仅因为“看起来没有越界”
就随意添加；承诺不成立会产生 poison/UB 相关后果。

### 7.4 控制流、PHI、调用

```cpp
B.CreateBr(Dest);
B.CreateCondBr(Cond, TrueBB, FalseBB);
B.CreateRet(RetV);
B.CreateRetVoid();
B.CreateUnreachable();

PHINode *PN = B.CreatePHI(Ty, IncomingCount, "phi");
CallInst *Call = B.CreateCall(Callee, Args, "call");
InvokeInst *Inv = B.CreateInvoke(FT, CalleeV, NormalBB, UnwindBB, Args);
```

`IRBuilder` 可能常量折叠：例如两个常量相加返回 `Constant` 而不是
`BinaryOperator`，所以多数 `Create*` 返回 `Value *` 是有意的。

---

## 8. DominatorTree 与 PostDominatorTree

### 8.1 定义

- A dominates B：从函数入口到 B 的每条路径都经过 A。
- A properly dominates B：A dominates B 且 A != B。
- immediate dominator（idom）：严格支配 B 的节点中离 B 最近者。
- A post-dominates B：从 B 到函数退出的每条路径都经过 A。

支配树是 CFG 上支配关系的树形表示，不是 CFG 本身。支配树节点的 parent 是 idom。

### 8.2 获取或独立构造

新 Pass Manager：

```cpp
DominatorTree &DT = FAM.getResult<DominatorTreeAnalysis>(F);
```

独立构造：

```cpp
DominatorTree DT(F);
// 或
DominatorTree DT;
DT.recalculate(F);
```

在 pass 中优先从 analysis manager 获取，避免重复计算，也能参与失效管理。

### 8.3 常用查询

```cpp
bool D1 = DT.dominates(BB1, BB2);
bool D2 = DT.properlyDominates(BB1, BB2);
bool D3 = DT.dominates(Inst1, Inst2);
bool D4 = DT.dominates(DefValue, UserInst);
bool D5 = DT.dominates(DefValue, OperandUse); // PHI 场景尤其重要

bool Reachable = DT.isReachableFromEntry(BB);
BasicBlock *NCD = DT.findNearestCommonDominator(A, B);

DomTreeNode *N = DT.getNode(BB);
BasicBlock *IDom = N && N->getIDom() ? N->getIDom()->getBlock() : nullptr;
unsigned Level = N ? N->getLevel() : 0;
```

遍历支配树：

```cpp
DomTreeNode *Root = DT.getRootNode();
for (DomTreeNode *Child : *Root) {
  BasicBlock *ChildBB = Child->getBlock();
}

SmallVector<BasicBlock *> Desc;
DT.getDescendants(BB, Desc);
```

### 8.4 指令级 dominance 的细节

同一基本块内通常按指令次序决定支配：定义在使用之前才支配该使用。跨块则先看基本块
支配关系。几个特殊点：

- 参数在函数入口可用。
- 常量和全局值不受普通局部定义点限制。
- PHI operand 的 use 位于 incoming edge。
- unreachable block 的支配语义容易违反直觉，先用
  `isReachableFromEntry()` 明确你想处理的范围。
- `dominates(A, A)` 为真；需要排除自身时用 `properlyDominates`。

```cpp
for (Use &U : Def->uses()) {
  if (!DT.dominates(Def, U))
    errs() << "definition does not dominate this exact use\n";
}
```

### 8.5 用支配树做典型判断

判断某检查是否覆盖某操作：

```cpp
if (DT.dominates(CheckInst, OperationInst)) {
  // 所有到达 Operation 的路径都经过 Check，且同块时 Check 在前。
}
```

寻找两个块最接近的共同控制祖先：

```cpp
BasicBlock *Where = DT.findNearestCommonDominator(BB1, BB2);
```

判断循环内一个定义是否可供循环内全部使用：对每条实际 `Use` 查询，而不是只看
user 所在块，才能正确覆盖 PHI edge。

### 8.6 修改 CFG 后维护 DT

最简单正确的策略是让本 pass 返回不保留 DT，由框架需要时重算。若变换频繁、重算昂贵，
使用 `DomTreeUpdater`：

```cpp
#include "llvm/Analysis/DomTreeUpdater.h"

DomTreeUpdater DTU(DT, DomTreeUpdater::UpdateStrategy::Lazy);

SmallVector<DominatorTree::UpdateType> Updates;
Updates.push_back({DominatorTree::Insert, From, To});
Updates.push_back({DominatorTree::Delete, OldFrom, OldTo});
DTU.applyUpdates(Updates);

// Lazy 模式下，需要最新结果前 flush。
DTU.flush();
```

原则：更新列表描述“实际 CFG 已经发生的 edge 变化”，二者必须一致。删除块可用
`DTU.deleteBB(BB)` 所提供的受控流程；具体前置条件请看当前版本头文件。

小变换通常首选 `BasicBlockUtils` 的 overload，将 `DT`/`DTU`、`LI` 传进去统一维护。

### 8.7 验证与打印 DT

```cpp
DT.print(errs());
bool Broken = DT.verify(DominatorTree::VerificationLevel::Full);
```

命令行：

```bash
opt -passes='print<domtree>' -disable-output input.ll
opt -passes='verify<domtree>' -disable-output input.ll
opt -passes='dot-dom' -disable-output input.ll
```

生成的 `.dot` 可用 Graphviz：

```bash
dot -Tpdf dom.foo.dot -o dom.foo.pdf
```

### 8.8 PostDominatorTree

```cpp
#include "llvm/Analysis/PostDominators.h"

PostDominatorTree &PDT =
    FAM.getResult<PostDominatorTreeAnalysis>(F);
bool MustPassLater = PDT.dominates(LaterBB, EarlierBB);
```

后支配树可能有虚拟根，函数也可能存在多个出口、无限循环或不可达区域；不要默认
`getRootNode()->getBlock()` 一定是实际基本块。常用于控制依赖、统一退出、判断某 cleanup
是否在所有离开路径上执行。

---

## 9. LoopInfo 与循环变换

### 9.1 获取与遍历

```cpp
LoopInfo &LI = FAM.getResult<LoopAnalysis>(F);

for (Loop *Top : LI) { ... }              // 顶层循环
for (Loop *L : LI.getLoopsInPreorder()) { ... }

Loop *L = LI.getLoopFor(BB);              // 包含 BB 的最内层循环
unsigned Depth = LI.getLoopDepth(BB);      // 不在循环中为 0
bool Header = LI.isLoopHeader(BB);
```

递归遍历：

```cpp
void visitLoop(Loop *L) {
  for (BasicBlock *BB : L->blocks()) { ... }
  for (Loop *Sub : L->getSubLoops())
    visitLoop(Sub);
}
```

### 9.2 `Loop` 高频接口

```cpp
BasicBlock *Header = L->getHeader();
BasicBlock *Preheader = L->getLoopPreheader();       // 可能为 null
BasicBlock *Latch = L->getLoopLatch();               // 不唯一时为 null
BasicBlock *Exiting = L->getExitingBlock();          // 唯一时才返回
BasicBlock *Exit = L->getExitBlock();                // 唯一 edge/条件相关
BasicBlock *UniqueExit = L->getUniqueExitBlock();
Loop *Parent = L->getParentLoop();
unsigned Depth = L->getLoopDepth();

bool HasBB = L->contains(BB);
bool HasInst = L->contains(I);
bool Invariant = L->isLoopInvariant(V);

SmallVector<BasicBlock *> ExitingBlocks;
L->getExitingBlocks(ExitingBlocks);
SmallVector<BasicBlock *> ExitBlocks;
L->getExitBlocks(ExitBlocks);
```

术语：

- header：支配整个循环的入口块。
- backedge：循环内指向 header 的边。
- latch：backedge 的源块。
- exiting block：循环内部、拥有通向循环外 edge 的块。
- exit block：循环外部、被 exiting edge 指向的块。
- preheader：循环外唯一前驱到 header 的规范化块。

### 9.3 LoopSimplify 与 LCSSA

很多 loop pass 假设：

- LoopSimplify form：有 preheader、唯一 backedge/latch、规范化 dedicated exits。
- LCSSA form：循环内定义且在循环外使用的值通过 exit PHI 暴露。

不要仅因拥有 `LoopInfo` 就认为这些性质成立。pipeline 中显式安排相应 pass，或查询：

```cpp
bool Simplified = L->isLoopSimplifyForm();
bool LCSSA = L->isLCSSAForm(DT);
```

### 9.4 修改循环

循环结构修改通常要同时维护 `LoopInfo`、`DominatorTree`、`ScalarEvolution`、
`MemorySSA`。可用工具包括：

- `formDedicatedExitBlocks`
- `InsertPreheaderForLoop`
- `SplitBlockPredecessors`
- `simplifyLoop`
- `formLCSSARecursively`
- `LoopInfo::changeLoopFor`、`addBasicBlockToLoop`（仅在理解层级后使用）

复杂变换若不能完整增量维护，返回不保留相关分析通常比留下陈旧结果更正确。

---

## 10. 新 Pass Manager 与分析管理

### 10.1 Function pass 模板

```cpp
class MyPass : public PassInfoMixin<MyPass> {
public:
  PreservedAnalyses run(Function &F, FunctionAnalysisManager &FAM) {
    DominatorTree &DT = FAM.getResult<DominatorTreeAnalysis>(F);
    LoopInfo &LI = FAM.getResult<LoopAnalysis>(F);

    bool Changed = transform(F, DT, LI);
    if (!Changed)
      return PreservedAnalyses::all();

    return PreservedAnalyses::none();
  }
};
```

只保留确实仍有效的分析：

```cpp
PreservedAnalyses PA;
PA.preserve<DominatorTreeAnalysis>();
PA.preserve<LoopAnalysis>();
PA.preserveSet<CFGAnalyses>(); // 仅当 CFG 未变且集合内分析确实有效
return PA;
```

“没有改 CFG”并不意味着所有分析都有效；改写算术也会让 SCEV、KnownBits、AA 等结果
过期。反之，只改不影响 CFG 的内容时通常可以保留 `CFGAnalyses`。

### 10.2 Module pass 访问 Function analysis

Module pass 不应随意触发任意 function analysis。通过代理读取已缓存结果，或使用
module-to-function adaptor 安排 function pass：

```cpp
auto &FAM = MAM.getResult<FunctionAnalysisManagerModuleProxy>(M).getManager();
```

构建 pipeline 时：

```cpp
FunctionPassManager FPM;
FPM.addPass(MyPass());

ModulePassManager MPM;
MPM.addPass(createModuleToFunctionPassAdaptor(std::move(FPM)));
```

### 10.3 Analysis pass 模板

```cpp
class MyAnalysis : public AnalysisInfoMixin<MyAnalysis> {
  friend AnalysisInfoMixin<MyAnalysis>;
  static AnalysisKey Key;

public:
  struct Result {
    unsigned Count;
  };

  Result run(Function &F, FunctionAnalysisManager &) {
    return {static_cast<unsigned>(F.size())};
  }
};

AnalysisKey MyAnalysis::Key;
```

取得结果：

```cpp
MyAnalysis::Result &R = FAM.getResult<MyAnalysis>(F);
```

### 10.4 常用标准分析获取方式

```cpp
auto &DT   = FAM.getResult<DominatorTreeAnalysis>(F);
auto &PDT  = FAM.getResult<PostDominatorTreeAnalysis>(F);
auto &LI   = FAM.getResult<LoopAnalysis>(F);
auto &SE   = FAM.getResult<ScalarEvolutionAnalysis>(F);
auto &AA   = FAM.getResult<AAManager>(F);
auto &MSSA = FAM.getResult<MemorySSAAnalysis>(F).getMSSA();
auto &AC   = FAM.getResult<AssumptionAnalysis>(F);
auto &TLI  = FAM.getResult<TargetLibraryAnalysis>(F);
auto &TTI  = FAM.getResult<TargetIRAnalysis>(F);
auto &BPI  = FAM.getResult<BranchProbabilityAnalysis>(F);
auto &BFI  = FAM.getResult<BlockFrequencyAnalysis>(F);
```

返回类型有的就是分析对象，有的是 wrapper result；以当前头文件的 `Result` typedef
和 `run()` 返回类型为准。

### 10.5 PassBuilder 的标准搭建

```cpp
LoopAnalysisManager LAM;
FunctionAnalysisManager FAM;
CGSCCAnalysisManager CGAM;
ModuleAnalysisManager MAM;

PassBuilder PB;
PB.registerModuleAnalyses(MAM);
PB.registerCGSCCAnalyses(CGAM);
PB.registerFunctionAnalyses(FAM);
PB.registerLoopAnalyses(LAM);
PB.crossRegisterProxies(LAM, FAM, CGAM, MAM);

ModulePassManager MPM =
    PB.buildPerModuleDefaultPipeline(OptimizationLevel::O2);
MPM.run(M, MAM);
```

注册顺序和 `crossRegisterProxies` 很重要，分析失效会跨 IR 层级传播。

---

## 11. AliasAnalysis、MemorySSA 与内存推理

### 11.1 AliasAnalysis（AA）

```cpp
AAResults &AA = FAM.getResult<AAManager>(F);

MemoryLocation A(PtrA, LocationSize::precise(4));
MemoryLocation B(PtrB, LocationSize::precise(4));
AliasResult R = AA.alias(A, B);

if (R == AliasResult::NoAlias) { ... }
if (R == AliasResult::MustAlias) { ... }
if (R == AliasResult::MayAlias) { ... }
if (R == AliasResult::PartialAlias) { ... }
```

便捷位置：

```cpp
MemoryLocation Loc = MemoryLocation::get(LI);
MemoryLocation Loc = MemoryLocation::get(SI);
```

Mod/Ref：

```cpp
ModRefInfo MRI = AA.getModRefInfo(Call, Loc);
if (isNoModRef(MRI)) { ... }
if (isModSet(MRI)) { ... }
if (isRefSet(MRI)) { ... }
```

`MayAlias` 不是“确定 alias”，而是分析无法证明不 alias；优化必须保守。

### 11.2 MemorySSA 心智模型

MemorySSA 给内存状态建立 SSA：

- `MemoryDef`：可能写内存的操作。
- `MemoryUse`：只读内存的操作。
- `MemoryPhi`：控制流汇合处的内存状态。
- `liveOnEntry`：函数入口时的初始内存状态。

```cpp
MemorySSA &MSSA = FAM.getResult<MemorySSAAnalysis>(F).getMSSA();

MemoryAccess *MA = MSSA.getMemoryAccess(I);
if (auto *MU = dyn_cast_or_null<MemoryUse>(MA)) { ... }
if (auto *MD = dyn_cast_or_null<MemoryDef>(MA)) { ... }
if (auto *MP = MSSA.getMemoryAccess(BB)) { /* block MemoryPhi */ }

MemorySSAWalker *Walker = MSSA.getWalker();
MemoryAccess *Clobber = Walker->getClobberingMemoryAccess(MA);
MSSA.print(errs());
```

MemorySSA 是分析加速结构，不等于完整内存语义证明；walker 会结合 AA。修改内存指令或
CFG 后需使用 `MemorySSAUpdater` 正确维护，或使 MemorySSA 失效。

### 11.3 何时选 AA、MemorySSA、MemoryDependence

- 两个具体内存位置是否可能重叠：AA。
- 某 load 往前最近可能被哪个写 clobber：MemorySSA walker。
- 大量 CFG 上内存依赖查询/变换：优先 MemorySSA。
- 不要只比较 pointer `Value *` 相等来判断内存是否相同；GEP、cast、不同表达式都可能
  指向同一地址。

---

## 12. ScalarEvolution 与 ValueTracking

### 12.1 ScalarEvolution（SCEV）

SCEV 用符号表达式描述整数值随循环迭代的变化：常量、加法、乘法、零/符号扩展、
AddRecurrence 等。

```cpp
ScalarEvolution &SE = FAM.getResult<ScalarEvolutionAnalysis>(F);

const SCEV *S = SE.getSCEV(V);
S->print(errs());

const SCEV *BTC = SE.getBackedgeTakenCount(L);
if (!isa<SCEVCouldNotCompute>(BTC)) { ... }

const SCEV *Trip = SE.getTripCountFromExitCount(BTC, /*...*/);
bool LoopInvariant = SE.isLoopInvariant(S, L);
```

识别归纳变量：

```cpp
if (auto *AR = dyn_cast<SCEVAddRecExpr>(S)) {
  const SCEV *Start = AR->getStart();
  const SCEV *Step = AR->getStepRecurrence(SE);
  Loop *Scope = AR->getLoop();
}
```

SCEV 的 nowrap flags 是推理承诺；不要未经证明手工添加。修改 SCEV 依赖的值/循环后，
若直接操作 `SE`，需要调用恰当的 forget 接口；在 pass 层面更简单的是不 preserve。

### 12.2 ValueTracking 高频工具

```cpp
#include "llvm/Analysis/ValueTracking.h"

SimplifyQuery SQ(DL, &TLI, &DT, &AC, CtxI);
bool KnownNonZero = isKnownNonZero(V, SQ);
bool KnownPositive = isKnownPositive(V, SQ);
bool PowerOfTwo = isKnownToBeAPowerOfTwo(V, /*OrZero=*/false, SQ);

// 此接口声明在 llvm/Analysis/Loads.h。
bool Deref = isDereferenceablePointer(V, Ty, DL, CtxI, &AC, &DT, &TLI);

KnownBits KB(BitWidth);
computeKnownBits(V, KB, SQ);
APInt KnownZero = KB.Zero;
APInt KnownOne = KB.One;

Value *Underlying = getUnderlyingObject(Ptr);
```

这些函数的 overload 参数在版本间变化明显，应以当前
`llvm/Analysis/ValueTracking.h` 为准。关键概念是传入上下文指令 `CtxI`、
`DataLayout`、`AssumptionCache` 和 `DominatorTree`，让查询利用当前位置有效的信息。

### 12.3 `SimplifyQuery` / InstructionSimplify

```cpp
#include "llvm/Analysis/InstructionSimplify.h"

SimplifyQuery Q(DL, &TLI, &DT, &AC);
if (Value *V = simplifyInstruction(&I, Q)) {
  I.replaceAllUsesWith(V);
  I.eraseFromParent();
}
```

InstructionSimplify 只返回已有值或常量，不创建新的复杂指令；InstCombine 则可重写成
新指令序列。

---

## 13. 分支概率、频率及其他常用分析

### 13.1 BPI / BFI

```cpp
BranchProbabilityInfo &BPI =
    FAM.getResult<BranchProbabilityAnalysis>(F);
BlockFrequencyInfo &BFI =
    FAM.getResult<BlockFrequencyAnalysis>(F);

BranchProbability P = BPI.getEdgeProbability(From, SuccIndex);
BlockFrequency Freq = BFI.getBlockFreq(BB);
std::optional<uint64_t> Count = BFI.getBlockProfileCount(BB);
```

概率是 edge 级别；同一 successor block 可能对应多个 successor index。不要把 heuristic
frequency 当作真实采样次数，只有 profile count 才具有计数语义，且也可能不存在。

### 13.2 AssumptionCache

`AssumptionCache` 索引 `llvm.assume`，供 ValueTracking、SCEV 等利用。取得方式：

```cpp
AssumptionCache &AC = FAM.getResult<AssumptionAnalysis>(F);
```

插入/删除 assume 后若要继续使用同一缓存，应调用缓存提供的登记/注销接口，或让分析
失效。

### 13.3 TLI 与 TTI

- `TargetLibraryInfo`：目标 libc/运行库函数是否存在、语义是什么。
- `TargetTransformInfo`：目标相关成本、合法性、寄存器/向量偏好等。

```cpp
TargetLibraryInfo &TLI = FAM.getResult<TargetLibraryAnalysis>(F);
TargetTransformInfo &TTI = FAM.getResult<TargetIRAnalysis>(F);

bool Available = TLI.has(LibFunc_malloc);
InstructionCost Cost = TTI.getInstructionCost(
    &I, TargetTransformInfo::TCK_RecipThroughput);
```

不要把 cost 强转成整数后假设永远有效；先处理 `InstructionCost` 的 valid/invalid 状态。

### 13.4 LazyValueInfo

LVI 回答某值在某程序点/边上的常量或范围信息，适合分支条件传播：

```cpp
LazyValueInfo &LVI = FAM.getResult<LazyValueAnalysis>(F);
Constant *C = LVI.getConstant(V, CtxI);
ConstantRange R = LVI.getConstantRange(V, CtxI);
```

### 13.5 OptimizationRemarkEmitter

优化为何做/没做，最好用 remark 而不是无条件 `errs()`：

```cpp
OptimizationRemarkEmitter &ORE =
    FAM.getResult<OptimizationRemarkEmitterAnalysis>(F);

ORE.emit([&]() {
  return OptimizationRemark(DEBUG_TYPE, "Transform", &I)
         << "optimized " << ore::NV("Value", V);
});
```

配合 `-pass-remarks`、`-pass-remarks-missed`、`-pass-remarks-analysis` 使用。

---

## 14. 调试、打印、验证与 `opt`

### 14.1 打印对象

```cpp
errs() << "value: " << *V << '\n';
errs() << "type: "; V->getType()->print(errs()); errs() << '\n';
I.print(errs());
BB.print(errs());
F.print(errs());
M.print(errs(), nullptr);
```

只有指针可能为 null 时不要直接 `*V`。`dump()` 适合 GDB/临时调试，但可能只在 assertions
构建可用，提交代码更适合 `print(raw_ostream&)` 或 LLVM debug 宏。

### 14.2 `LLVM_DEBUG` 与 `DEBUG_TYPE`

```cpp
#define DEBUG_TYPE "my-pass"
#include "llvm/Support/Debug.h"

LLVM_DEBUG(dbgs() << "visiting: " << I << '\n');
```

运行：

```bash
opt -debug-only=my-pass -passes='my-pass' input.ll -disable-output
```

LLVM 通常需 assertions/debug 支持才能使用这些开关。不要提交无条件刷屏的 `errs()`。

### 14.3 验证 IR

```cpp
bool BrokenFunction = verifyFunction(F, &errs());
bool BrokenModule = verifyModule(M, &errs());
```

注意返回值：`true` 表示 broken，而不是验证成功。

在 pipeline 中：

```bash
opt -passes='my-pass,verify' input.ll -S -o output.ll
opt -verify-each -passes='pass1,pass2,my-pass' input.ll -disable-output
```

开发 CFG/SSA 变换时强烈建议使用 `-verify-each`。

### 14.4 `opt` 高频命令

```bash
# 看 pass 后 IR
opt -S -passes='my-pass' input.ll -o -

# 默认 O2 pipeline
opt -S -passes='default<O2>' input.ll -o output.ll

# 打印 pipeline
opt -passes='default<O2>' -print-pipeline-passes -disable-output input.ll

# 前后 IR 和 pass 调试
opt -passes='my-pass' -print-before=my-pass -print-after=my-pass \
  -disable-output input.ll

# 只对特定函数打印（常与 print-before/after 配合）
opt -filter-print-funcs=foo -passes='my-pass' -disable-output input.ll

# CFG / domtree 图
opt -passes='dot-cfg' -disable-output input.ll
opt -passes='dot-dom' -disable-output input.ll

# 加载外部 pass plugin
opt -load-pass-plugin=./libMyPass.so -passes='my-pass' \
  -S input.ll -o output.ll
```

当前 Jeandle 有自定义 pipeline/option，实际名字可结合
`llvm/lib/Passes/PassBuilder.cpp` 和项目文档查询。

### 14.5 GDB 实用表达式

```gdb
(gdb) p I->dump()
(gdb) p BB->dump()
(gdb) p F->dump()
(gdb) call llvm::errs().flush()
```

断点经常设在 `MyPass::run`、`verifyFunction`、某个具体变换工具或断言位置。优化构建中
变量可能 optimized out，调 LLVM pass 更适合 assertions + debug info 构建。

---

## 15. LLVM ADT、错误处理和命令行

### 15.1 高频 ADT

```cpp
StringRef S = "abc";                 // 非拥有字符串视图
ArrayRef<Value *> Args = Values;      // 非拥有连续数组视图
MutableArrayRef<int> Mutable = Data;

SmallVector<Value *, 8> Values;       // 小数据内联，超出后堆分配
SmallPtrSet<Value *, 16> Seen;         // 指针集合
DenseMap<Value *, unsigned> IDs;       // 高性能哈希表
DenseSet<BasicBlock *> Blocks;
BitVector Bits(100);
SmallBitVector SmallBits(16);
```

生命周期陷阱：`StringRef`/`ArrayRef` 不拥有底层存储，不能引用已经销毁的临时
`std::string`/`SmallVector`。

```cpp
StringRef Name = F.getName();
std::string Owned = Name.str();

for (auto [Index, V] : enumerate(Values)) { ... }
for (auto [A, B] : zip(As, Bs)) { ... }
for (Instruction &I : reverse(BB)) { ... }
```

### 15.2 `Error` / `Expected<T>`

```cpp
Expected<Result> R = compute();
if (!R)
  return R.takeError();
use(*R);

if (Error Err = doWork())
  return Err;

return createStringError(inconvertibleErrorCode(),
                         "bad value: %s", Name.str().c_str());
```

顶层工具常用：

```cpp
ExitOnError ExitOnErr;
Result R = ExitOnErr(compute());
```

每个 `Error` 必须被消费：返回、`consumeError`、`handleErrors` 或交给
`ExitOnError`；静默丢弃会触发检查。

### 15.3 `raw_ostream`

```cpp
outs() << "normal output\n";
errs() << "diagnostic\n";
dbgs() << "debug\n";

std::string Buffer;
raw_string_ostream OS(Buffer);
V->print(OS);
OS.flush();
```

### 15.4 命令行选项

```cpp
#include "llvm/Support/CommandLine.h"

static cl::opt<bool> EnableFoo(
    "enable-foo", cl::desc("Enable foo transform"), cl::init(false),
    cl::Hidden);
```

库代码谨慎增加全局 option，避免命名冲突；pass plugin 更适合通过 pipeline 参数或回调
配置。

---

## 16. Call、Intrinsic、Metadata 和 Attribute

### 16.1 `CallBase`

```cpp
if (auto *CB = dyn_cast<CallBase>(&I)) {
  Function *Direct = CB->getCalledFunction(); // 间接调用时为 null
  Value *CalledOperand = CB->getCalledOperand();
  FunctionType *FT = CB->getFunctionType();
  unsigned N = CB->arg_size();

  for (Use &Arg : CB->args()) {
    Value *V = Arg.get();
  }

  bool Tail = CB->isTailCall();
  bool NoThrow = CB->doesNotThrow();
  bool ReadNone = CB->doesNotAccessMemory();
  bool OnlyReads = CB->onlyReadsMemory();
}
```

调用目标可能经过 pointer cast；需要识别直接函数时可按任务考虑
`getCalledOperand()->stripPointerCasts()`，但不要因此忽略 alias/interposition 等语义。

### 16.2 Intrinsic

```cpp
Function *Trap = Intrinsic::getDeclaration(&M, Intrinsic::trap);
B.CreateCall(Trap);

if (auto *II = dyn_cast<IntrinsicInst>(&I)) {
  Intrinsic::ID ID = II->getIntrinsicID();
}
```

重载 intrinsic（如依赖元素类型）需向 `getDeclaration` 传 overload types。

### 16.3 Attribute

```cpp
F.addFnAttr(Attribute::NoUnwind);
bool NoUnwind = F.hasFnAttribute(Attribute::NoUnwind);
F.removeFnAttr(Attribute::NoUnwind);

CB->addFnAttr(Attribute::NoUnwind);
CB->addParamAttr(0, Attribute::NonNull);
bool NonNull = CB->paramHasAttr(0, Attribute::NonNull);
```

attribute 是优化器可依赖的语义承诺，错误的 `nonnull`、`dereferenceable`、`noundef`、
`noalias` 等可能直接导致错误代码，不只是“提示不准确”。

### 16.4 Metadata

```cpp
MDNode *N = MDNode::get(C, MDString::get(C, "payload"));
I.setMetadata("my.tag", N);
MDNode *Found = I.getMetadata("my.tag");
I.setMetadata("my.tag", nullptr); // 删除
```

复制/替换指令时，调试位置、alias scope、TBAA、range、branch weights 等 metadata 是否应该
保留必须逐项判断；盲目复制可能把旧指令的错误事实附到新指令上。

---

## 17. 常见任务配方

### 17.1 找函数内所有直接调用

```cpp
for (Instruction &I : instructions(F)) {
  auto *CB = dyn_cast<CallBase>(&I);
  if (!CB) continue;

  Function *Callee = CB->getCalledFunction();
  if (!Callee) continue; // indirect call
  errs() << F.getName() << " -> " << Callee->getName() << '\n';
}
```

### 17.2 判断值是否只在某循环内使用

```cpp
bool allUsesInside(Value *V, Loop *L) {
  for (Use &U : V->uses()) {
    auto *I = dyn_cast<Instruction>(U.getUser());
    if (!I || !L->contains(I))
      return false;
  }
  return true;
}
```

若涉及 PHI edge，业务语义可能要求检查 incoming block，而不只是 PHI 所在块：

```cpp
if (auto *PN = dyn_cast<PHINode>(U.getUser())) {
  BasicBlock *Incoming = PN->getIncomingBlock(U.getOperandNo());
  ...
}
```

### 17.3 在所有 return 前插入调用

```cpp
SmallVector<ReturnInst *> Returns;
for (BasicBlock &BB : F)
  if (auto *RI = dyn_cast<ReturnInst>(BB.getTerminator()))
    Returns.push_back(RI);

for (ReturnInst *RI : Returns) {
  IRBuilder<> B(RI);
  B.SetCurrentDebugLocation(RI->getDebugLoc());
  B.CreateCall(Hook, {});
}
```

如果函数可通过 `resume`、cleanupret、异常路径等离开，“所有 return”不等于“所有退出”。

### 17.4 用 PatternMatch 匹配表达式

```cpp
using namespace llvm::PatternMatch;

Value *X;
ConstantInt *C;
if (match(V, m_Add(m_Value(X), m_ConstantInt(C)))) { ... }

// 匹配 X & (2^k - 1)
APInt Mask;
if (match(V, m_And(m_Value(X), m_APInt(Mask)))) { ... }
```

PatternMatch 会隐藏部分 operand 顺序细节。需要保留 flags、精确指令类型或 use 时，匹配后
仍要检查原 `Instruction`。

### 17.5 安全替换一条纯计算指令

```cpp
if (Value *Replacement = simplifyInstruction(&I, Q)) {
  I.replaceAllUsesWith(Replacement);
  I.eraseFromParent();
  Changed = true;
}
```

若 replacement 是新创建的指令：先设置 debug loc/必要 metadata，插入到支配所有 use 的
合法位置，再 RAUW。用 `DT.dominates(Replacement, U)` 验证每条 use 是很好的调试手段。

### 17.6 新建 if-then 控制流

优先查看并使用：

```cpp
#include "llvm/Transforms/Utils/BasicBlockUtils.h"

Instruction *ThenTerm = SplitBlockAndInsertIfThen(
    Cond, SplitBefore, /*Unreachable=*/false, /*BranchWeights=*/nullptr,
    &DT, &LI);
```

比手工拆块更不容易漏掉 PHI、DT、LI 更新。具体 overload 以 LLVM 22 当前头文件为准。

### 17.7 查找支配全部目标的位置

```cpp
BasicBlock *InsertBB = nullptr;
for (BasicBlock *BB : Targets)
  InsertBB = InsertBB ? DT.findNearestCommonDominator(InsertBB, BB) : BB;

if (!InsertBB) return;
Instruction *IP = &*InsertBB->getFirstInsertionPt();
```

这只得到共同支配块；仍需确保新指令的 operands 在 insertion point 可用，并处理 PHI、
异常边、不可达块和 speculation safety。

### 17.8 判断能否投机执行

```cpp
#include "llvm/Analysis/ValueTracking.h"

if (isSafeToSpeculativelyExecute(&I, CtxI, &AC, &DT, &TLI)) {
  ...
}
```

不要以“无 store”替代该判断：除零、非法 shift、可能 trap 的 load、convergent call、
poison/UB 都会影响能否移动或投机执行。

---

## 18. 高频接口速查表与踩坑清单

### 18.1 按问题找接口

| 问题 | 首选接口/分析 |
|---|---|
| A 是否保证在 B 之前执行 | `DominatorTree::dominates` |
| A 是否保证在 B 之后经过 | `PostDominatorTree::dominates` |
| 块属于哪个最内层循环 | `LoopInfo::getLoopFor` |
| 两个地址是否可能重叠 | `AAResults::alias` |
| 某 load 可能被哪个写覆盖 | `MemorySSA` + walker |
| 循环次数/归纳表达式 | `ScalarEvolution` |
| 一个值哪些 bit 已知 | `computeKnownBits` |
| 值在某点是否为常量/范围 | `LazyValueInfo` |
| 指令是否可投机执行 | `isSafeToSpeculativelyExecute` |
| 指令是否平凡死代码 | `isInstructionTriviallyDead` |
| 获取目标 ABI 大小/对齐 | `DataLayout` |
| 获取目标机器成本 | `TargetTransformInfo` |
| 获取 libc 语义 | `TargetLibraryInfo` |
| 拆块、拆边、合并块 | `BasicBlockUtils` |
| 检查生成 IR 是否合法 | `verifyFunction` / `verifyModule` |

### 18.2 `DominatorTree` 速查

| 目的 | 写法 |
|---|---|
| 获取 | `FAM.getResult<DominatorTreeAnalysis>(F)` |
| 块支配 | `DT.dominates(A, B)` |
| 严格支配 | `DT.properlyDominates(A, B)` |
| 指令支配 | `DT.dominates(DefI, UserI)` |
| 精确 use 支配 | `DT.dominates(Def, U)` |
| 是否从 entry 可达 | `DT.isReachableFromEntry(BB)` |
| 最近公共支配者 | `DT.findNearestCommonDominator(A, B)` |
| idom | `DT.getNode(BB)->getIDom()` |
| 重算 | `DT.recalculate(F)` |
| 增量维护 | `DomTreeUpdater` |
| 验证 | `DT.verify(...)` / `verify<domtree>` |
| 打印 | `DT.print(errs())` / `print<domtree>` |

### 18.3 提交 pass 前逐项检查

- [ ] 是否处理 declaration、empty function 和 unreachable blocks？
- [ ] 是否把 `CallInst` 错当成所有调用，而漏掉 invoke/callbr？
- [ ] 是否用目标 `DataLayout`，而不是宿主 `sizeof`？
- [ ] 是否正确区分 signed/unsigned compare、extend、division？
- [ ] 是否理解 poison、undef、freeze 和 inbounds/nowrap flags？
- [ ] 新定义是否支配每一条 use，尤其 PHI incoming edge？
- [ ] CFG edge 变化后 PHI incoming 是否同步？
- [ ] DT、PDT、LI、SCEV、MSSA、BPI/BFI 是否被更新或失效？
- [ ] 删除指令时是否避免迭代器和裸指针失效？
- [ ] RAUW 前后类型是否完全一致？
- [ ] 移动/投机执行是否考虑 trap、异常、convergent、原子和 volatile？
- [ ] 新指令是否继承恰当的 debug location？
- [ ] metadata/attribute 是经过证明的语义事实，而不是猜测？
- [ ] `verifyFunction` / `verifyModule` 的返回值是否按“true = broken”理解？
- [ ] 测试是否使用 `-verify-each`，并覆盖负例和异常 CFG？
- [ ] `PreservedAnalyses` 是否只保留确实仍然有效的结果？

### 18.4 推荐源码阅读入口

本仓库中最值得随手翻的头文件：

```text
jeandle-llvm/llvm/include/llvm/IR/Value.h
jeandle-llvm/llvm/include/llvm/IR/Instructions.h
jeandle-llvm/llvm/include/llvm/IR/IRBuilder.h
jeandle-llvm/llvm/include/llvm/IR/Dominators.h
jeandle-llvm/llvm/include/llvm/Support/GenericDomTree.h
jeandle-llvm/llvm/include/llvm/Analysis/DomTreeUpdater.h
jeandle-llvm/llvm/include/llvm/Analysis/LoopInfo.h
jeandle-llvm/llvm/include/llvm/Analysis/ScalarEvolution.h
jeandle-llvm/llvm/include/llvm/Analysis/ValueTracking.h
jeandle-llvm/llvm/include/llvm/Analysis/AliasAnalysis.h
jeandle-llvm/llvm/include/llvm/Analysis/MemorySSA.h
jeandle-llvm/llvm/include/llvm/IR/PassManager.h
jeandle-llvm/llvm/include/llvm/Passes/PassBuilder.h
jeandle-llvm/llvm/include/llvm/Transforms/Utils/BasicBlockUtils.h
jeandle-llvm/llvm/include/llvm/Transforms/Utils/Local.h
```

读 LLVM 接口最快的方法通常是：先从头文件注释确认契约，再在 `llvm/lib` 搜实现，在
`llvm/unittests` 和 `llvm/test` 搜调用范例。例如：

```bash
rg -n "findNearestCommonDominator" jeandle-llvm/llvm/{include,lib,unittests}
rg -n "getResult<DominatorTreeAnalysis>" jeandle-llvm/llvm/{lib,examples}
rg -n "MemorySSAUpdater" jeandle-llvm/llvm/{lib,unittests}
```

---

## 附录：一个可直接改造的 Function Pass 骨架

```cpp
#include "llvm/Analysis/LoopInfo.h"
#include "llvm/IR/Dominators.h"
#include "llvm/IR/Function.h"
#include "llvm/IR/InstIterator.h"
#include "llvm/IR/PassManager.h"
#include "llvm/IR/Verifier.h"
#include "llvm/Support/Debug.h"

#define DEBUG_TYPE "example-pass"

using namespace llvm;

namespace {

class ExamplePass : public PassInfoMixin<ExamplePass> {
public:
  PreservedAnalyses run(Function &F, FunctionAnalysisManager &FAM) {
    if (F.isDeclaration())
      return PreservedAnalyses::all();

    DominatorTree &DT = FAM.getResult<DominatorTreeAnalysis>(F);
    LoopInfo &LI = FAM.getResult<LoopAnalysis>(F);

    bool Changed = false;
    for (Instruction &I : instructions(F)) {
      LLVM_DEBUG(dbgs() << "visit " << I << '\n');

      BasicBlock *BB = I.getParent();
      Loop *L = LI.getLoopFor(BB);
      LLVM_DEBUG(if (L) dbgs() << "  loop depth=" << L->getLoopDepth()
                               << '\n');

      // 在这里查询分析并完成变换。
      (void)DT;
    }

#ifndef NDEBUG
    if (Changed)
      assert(!verifyFunction(F, &errs()) && "ExamplePass produced invalid IR");
#endif

    if (!Changed)
      return PreservedAnalyses::all();

    // 示例没有实现分析的增量维护，因此保守失效全部分析。
    return PreservedAnalyses::none();
  }
};

} // namespace
```

这份骨架故意采用保守失效策略。变换正确并稳定后，再根据实际修改范围精确声明
`PreservedAnalyses`，不要一开始就过度保留分析结果。
