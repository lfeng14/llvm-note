# LLVM `dyn_cast` 类型每日记忆表：Operator、GEP 与常见 IR 节点

> 基于本仓库 LLVM 22.0.0git。适合每天用 10～15 分钟浏览。
>
> 第一遍只看“每日必背 20 个类型”；第二遍重点看 `Operator`；第三遍通过末尾自测题复习。

## 1. 先记住一个核心区别

LLVM 中有两种看起来相似但用途不同的类型：

```text
具体 Instruction 类型                         Operator 观察类型

GetElementPtrInst                             GEPOperator
BinaryOperator(add/sub/mul...)                AddOperator/SubOperator/...
BitCastInst                                   BitCastOperator
PtrToIntInst                                  PtrToIntOperator

只匹配真正位于 BasicBlock 中的指令             同时匹配 Instruction 和 ConstantExpr
```

例如下面两个值语义上都是 GEP：

```llvm
; Instruction：位于函数基本块中
%field = getelementptr %Obj, ptr %obj, i64 0, i32 1

; ConstantExpr：作为全局 initializer 的常量表达式
@field.addr = global ptr getelementptr (%Obj, ptr @global_obj, i64 0, i32 1)
```

它们的匹配结果：

| 查询 | GEP Instruction | GEP ConstantExpr |
|---|---:|---:|
| `isa<GetElementPtrInst>(V)` | 是 | 否 |
| `isa<ConstantExpr>(V)` | 否 | 是 |
| `isa<GEPOperator>(V)` | 是 | 是 |
| `isa<Operator>(V)` | 是 | 是 |

所以：

```cpp
// 只处理函数体内真正的 GEP 指令。
if (auto *GEP = dyn_cast<GetElementPtrInst>(V)) {
  BasicBlock *BB = GEP->getParent();
}

// 同时处理 GEP instruction 和 GEP ConstantExpr。
if (auto *GEP = dyn_cast<GEPOperator>(V)) {
  Value *Base = GEP->getPointerOperand();
  Type *SourceTy = GEP->getSourceElementType();
}
```

一句话记忆：

> `XxxInst` 强调“它是一条指令”；`XxxOperator` 强调“它具有某种运算语义”。

---

# 2. `Operator` 到底是什么

## 2.1 它是 utility view，不是普通 IR 节点

`Operator` 定义在：

```cpp
#include "llvm/IR/Operator.h"
```

它用于抽象 `Instruction` 和 `ConstantExpr` 的共同功能：

```text
                    Value / User
                       ▲
                       │ LLVM casting 观察
                    Operator
                    /      \
           Instruction    ConstantExpr
```

这张图表达的是“casting 语义”，不是普通 C++ 对象所有权层级。`Operator`：

- 不会被单独实例化；
- 构造、析构和 `operator new` 被删除；
- `classof()` 接受 `Instruction` 或 `ConstantExpr`；
- `getOpcode()` 对两种对象提供统一访问。

```cpp
if (auto *Op = dyn_cast<Operator>(V)) {
  unsigned Opcode = Op->getOpcode();
  // V 要么是 Instruction，要么是 ConstantExpr。
}
```

## 2.2 为什么需要它

如果没有 Operator，处理同一种运算需要写两遍：

```cpp
if (auto *GEP = dyn_cast<GetElementPtrInst>(V)) {
  ...
} else if (auto *CE = dyn_cast<ConstantExpr>(V)) {
  if (CE->getOpcode() == Instruction::GetElementPtr) {
    ...
  }
}
```

有 `GEPOperator` 后：

```cpp
if (auto *GEP = dyn_cast<GEPOperator>(V)) {
  ...
}
```

## 2.3 `Operator` 体系总图

```text
Operator
├── OverflowingBinaryOperator
│   ├── AddOperator
│   ├── SubOperator
│   ├── MulOperator
│   └── ShlOperator
├── PossiblyExactOperator
│   ├── AShrOperator
│   └── LShrOperator
├── FPMathOperator
├── GEPOperator
├── PtrToIntOperator
├── PtrToAddrOperator            LLVM 22 中可见
├── BitCastOperator
└── AddrSpaceCastOperator
```

注意：并非所有 LLVM opcode 都有一个同名 `XxxOperator`。例如普通 `and/or/xor` 通常通过
`BinaryOperator + opcode`，或者 `Operator + opcode` 处理。

---

# 3. Operator 每日必背表

## 3.1 `Operator`

| 项目 | 内容 |
|---|---|
| 匹配对象 | 任意 `Instruction` 或 `ConstantExpr` |
| 不匹配 | `Argument`、`BasicBlock`、普通 `ConstantInt` 等 |
| 核心 API | `getOpcode()`、`hasPoisonGeneratingFlags()` |
| 典型用途 | 不关心运算是 instruction 还是 constant expression |

```cpp
if (auto *Op = dyn_cast<Operator>(V)) {
  switch (Op->getOpcode()) {
  case Instruction::Add:
  case Instruction::Sub:
    break;
  }
}
```

## 3.2 `OverflowingBinaryOperator`

| 项目 | 内容 |
|---|---|
| 主要 opcode | `add`、`sub`、`mul`、`shl` |
| 关注 flags | `nuw`、`nsw` |
| 核心 API | `hasNoUnsignedWrap()`、`hasNoSignedWrap()` |
| 错误理解 | “可能溢出”不是说运行时一定溢出，而是该运算支持 nowrap flags |

```cpp
if (auto *OBO = dyn_cast<OverflowingBinaryOperator>(V)) {
  bool NUW = OBO->hasNoUnsignedWrap();
  bool NSW = OBO->hasNoSignedWrap();
}
```

IR：

```llvm
%x = add nuw nsw i32 %a, %b
%y = shl nsw i32 %x, 1
```

关键语义：违反 `nuw/nsw` 承诺会产生 poison；它们不是普通性能提示。

## 3.3 `AddOperator` / `SubOperator` / `MulOperator` / `ShlOperator`

| 类型 | 匹配 opcode | 常用原因 |
|---|---|---|
| `AddOperator` | `add` | 同时读取 instruction/ConstantExpr 的 add 与 nowrap flags |
| `SubOperator` | `sub` | 同上 |
| `MulOperator` | `mul` | 同上 |
| `ShlOperator` | `shl` | 同时检查 shift 与 nowrap flags |

```cpp
if (auto *Add = dyn_cast<AddOperator>(V)) {
  Value *LHS = Add->getOperand(0);
  Value *RHS = Add->getOperand(1);
  bool NSW = Add->hasNoSignedWrap();
}
```

与 `BinaryOperator` 的选择：

```cpp
// 只关心真正的 add Instruction，并可能修改/移动/删除它。
if (auto *BO = dyn_cast<BinaryOperator>(V)) {
  if (BO->getOpcode() == Instruction::Add)
    BO->eraseFromParent();
}

// 只读取 add 运算语义，来源可能是 Instruction 或 ConstantExpr。
if (auto *Add = dyn_cast<AddOperator>(V)) {
  inspect(Add);
}
```

## 3.4 `PossiblyExactOperator`

| 项目 | 内容 |
|---|---|
| 匹配 opcode | `udiv`、`sdiv`、`lshr`、`ashr` instruction |
| 关注 flag | `exact` |
| 核心 API | `isExact()` |
| exact 含义 | 除法无余数，或右移没有丢弃非零 bit |

```cpp
if (auto *PE = dyn_cast<PossiblyExactOperator>(V)) {
  if (PE->isExact()) {
    ...
  }
}
```

IR：

```llvm
%q = udiv exact i32 %x, 4
%s = lshr exact i32 %x, 2
```

违反 `exact` 承诺会产生 poison。

## 3.5 `AShrOperator` / `LShrOperator`

| 类型 | opcode | 语义 |
|---|---|---|
| `AShrOperator` | `ashr` | 算术右移，复制 sign bit |
| `LShrOperator` | `lshr` | 逻辑右移，高位补零 |

```cpp
if (auto *Shift = dyn_cast<LShrOperator>(V)) {
  Value *Input = Shift->getOperand(0);
  Value *Amount = Shift->getOperand(1);
  bool Exact = Shift->isExact();
}
```

为什么没有单独列 `UDivOperator` / `SDivOperator`：当前 `Operator.h` 提供通用的
`PossiblyExactOperator`，division 通常再检查 opcode。

```cpp
if (auto *PE = dyn_cast<PossiblyExactOperator>(V)) {
  if (PE->getOpcode() == Instruction::UDiv) {
    ...
  }
}
```

## 3.6 `FPMathOperator`

它用于统一读取 Fast-Math Flags（FMF），匹配范围比“浮点二元运算”更宽。

| 可能匹配 | 条件 |
|---|---|
| `fneg/fadd/fsub/fmul/fdiv/frem` | 浮点运算 instruction |
| `fptrunc/fpext/fcmp` | 对应 instruction |
| `phi/select/call` | 结果是支持的浮点、浮点向量或特定同质聚合类型 |
| `ConstantExpr` | 不匹配；当前实现只接受 Instruction |

核心 API：

```cpp
if (auto *FPOp = dyn_cast<FPMathOperator>(V)) {
  FastMathFlags FMF = FPOp->getFastMathFlags();
  bool Fast = FPOp->isFast();
  bool Reassoc = FPOp->hasAllowReassoc();
  bool NoNaNs = FPOp->hasNoNaNs();
  bool NoInfs = FPOp->hasNoInfs();
  bool NoSignedZeros = FPOp->hasNoSignedZeros();
  bool Reciprocal = FPOp->hasAllowReciprocal();
  bool Contract = FPOp->hasAllowContract();
  bool ApproxFunc = FPOp->hasApproxFunc();
}
```

记忆：

```text
reassoc   允许重结合
nnan      假定无 NaN
ninf      假定无 Infinity
nsz       忽略 signed zero 差异
arcp      允许 reciprocal transform
contract  允许 contraction，例如 FMA
afn       允许近似数学函数
fast      上述宽松条件的组合
```

## 3.7 `GEPOperator`

这是最值得熟悉的 Operator。

| 项目 | 内容 |
|---|---|
| 匹配 | `GetElementPtrInst` 或 GEP `ConstantExpr` |
| 不做什么 | 不读取内存，只计算地址 |
| operand 0 | base pointer |
| operand 1...N | indices |
| 类型来源 | `getSourceElementType()`，不能从 opaque pointer 推导 pointee type |

### 高频 API 表

| API | 含义 |
|---|---|
| `getPointerOperand()` | base pointer |
| `getPointerOperandType()` | base pointer 的 pointer type |
| `getPointerAddressSpace()` | address space |
| `getSourceElementType()` | GEP 开始索引的源元素类型 |
| `getResultElementType()` | 索引后指向的元素类型 |
| `getNumIndices()` | index 数量 |
| `indices()` | 遍历 index operands |
| `isInBounds()` | 是否有 `inbounds` 语义 |
| `getNoWrapFlags()` | LLVM 22 的 GEP nowrap flags |
| `hasAllZeroIndices()` | indices 是否全为 0 |
| `hasAllConstantIndices()` | indices 是否全是 `ConstantInt` |
| `countNonConstantIndices()` | 非常量 index 个数 |
| `accumulateConstantOffset()` | 尝试求固定 byte offset |
| `collectOffset()` | 分解 constant 与 variable offset |

### 读取 GEP

```cpp
if (auto *GEP = dyn_cast<GEPOperator>(V)) {
  errs() << "base=" << *GEP->getPointerOperand() << '\n';
  errs() << "source type=";
  GEP->getSourceElementType()->print(errs());
  errs() << '\n';

  for (const Use &Index : GEP->indices())
    errs() << "index=" << *Index.get() << '\n';
}
```

### 求常量 offset

```cpp
if (auto *GEP = dyn_cast<GEPOperator>(V)) {
  unsigned AS = GEP->getPointerAddressSpace();
  APInt Offset(DL.getIndexSizeInBits(AS), 0);

  if (GEP->accumulateConstantOffset(DL, Offset)) {
    errs() << "constant byte offset=" << Offset << '\n';
  }
}
```

不要自己用 `sizeof` 计算，必须使用目标 `DataLayout`。APInt 位宽也要与该 address space 的
index size 匹配。

### `inbounds` 不是普通标记

```cpp
if (GEP->isInBounds()) {
  ...
}
```

`inbounds` 是优化器可依赖的强语义承诺。错误添加可能产生 poison，不能因为“这个数组访问
看起来没有越界”就随便设置。

### `GEPOperator` 与相关工具区别

| 类型/工具 | 用途 |
|---|---|
| `GetElementPtrInst` | 需要 instruction parent、插入、移动、删除 |
| `GEPOperator` | 统一观察 Instruction 和 ConstantExpr 的 GEP 语义 |
| `GEPTypeIterator` | 按 index 遍历被索引的类型层级 |
| `DataLayout` | 计算 ABI size、alignment、struct offset |
| `getUnderlyingObject()` | 向上剥离 GEP/cast，寻找底层对象 |
| `stripPointerCasts()` | 剥离特定 pointer casts；语义比 underlying object 更窄 |

## 3.8 `PtrToIntOperator`

| 项目 | 内容 |
|---|---|
| 匹配 opcode | `ptrtoint` |
| 核心 API | `getPointerOperand()`、`getPointerAddressSpace()` |
| 用途 | 同时处理 instruction/ConstantExpr 形式的 pointer-to-integer conversion |

```cpp
if (auto *P2I = dyn_cast<PtrToIntOperator>(V)) {
  Value *Ptr = P2I->getPointerOperand();
  Type *IntegerTy = P2I->getType();
}
```

不要假设目标整数宽度一定等于宿主 `uintptr_t`；查询 `DataLayout`。

## 3.9 `PtrToAddrOperator`

LLVM 22 源码中存在该类型，用于 `ptrtoaddr` opcode 的统一观察：

```cpp
if (auto *P2A = dyn_cast<PtrToAddrOperator>(V)) {
  Value *Ptr = P2A->getPointerOperand();
  unsigned AS = P2A->getPointerAddressSpace();
}
```

它与 `ptrtoint` 的精确语义差异应以当前 LLVM LangRef/目标 address model 为准；阅读旧版本
资料时可能看不到该类型。

## 3.10 `BitCastOperator`

| 项目 | 内容 |
|---|---|
| 匹配 opcode | `bitcast` |
| 核心 API | `getSrcTy()`、`getDestTy()`、`getOperand(0)` |
| 特点 | 改变类型解释，不做普通数值转换 |

```cpp
if (auto *BC = dyn_cast<BitCastOperator>(V)) {
  Type *SrcTy = BC->getSrcTy();
  Type *DstTy = BC->getDestTy();
}
```

现代 opaque pointer 减少了很多 pointer-to-pointer bitcast，但向量/整数/浮点合法 bitcast
仍然存在。

## 3.11 `AddrSpaceCastOperator`

| 项目 | 内容 |
|---|---|
| 匹配 opcode | `addrspacecast` |
| 核心 API | `getPointerOperand()`、`getSrcAddressSpace()`、`getDestAddressSpace()` |
| 用途 | 跨 address space 的 pointer conversion |

```cpp
if (auto *ASC = dyn_cast<AddrSpaceCastOperator>(V)) {
  unsigned SrcAS = ASC->getSrcAddressSpace();
  unsigned DstAS = ASC->getDestAddressSpace();
}
```

address space cast 是否为 no-op、是否合法以及如何 codegen 都是 target-dependent，不能当成
普通 `bitcast`。

---

# 4. `Operator` 与具体 Instruction 对照总表

| 想识别的语义 | 只匹配 Instruction | 同时兼容 ConstantExpr/抽象语义 |
|---|---|---|
| 任意运算 | `Instruction` | `Operator` |
| add | `BinaryOperator` + `Instruction::Add` | `AddOperator` |
| sub | `BinaryOperator` + `Instruction::Sub` | `SubOperator` |
| mul | `BinaryOperator` + `Instruction::Mul` | `MulOperator` |
| shl | `BinaryOperator` + `Instruction::Shl` | `ShlOperator` |
| udiv/sdiv exact | `BinaryOperator` + opcode | `PossiblyExactOperator` + opcode |
| ashr | `BinaryOperator` + opcode | `AShrOperator` |
| lshr | `BinaryOperator` + opcode | `LShrOperator` |
| GEP | `GetElementPtrInst` | `GEPOperator` |
| ptrtoint | `PtrToIntInst` | `PtrToIntOperator` |
| ptrtoaddr | `PtrToAddrInst`/当前版本具体类 | `PtrToAddrOperator` |
| bitcast | `BitCastInst` | `BitCastOperator` |
| addrspacecast | `AddrSpaceCastInst` | `AddrSpaceCastOperator` |
| fast-math flags | 多种具体 Instruction | `FPMathOperator`，但当前只匹配 Instruction |

选择口诀：

```text
需要 getParent / erase / move / insert？
    → 用 XxxInst 或 Instruction

只想读取统一运算语义，且 Value 可能来自全局 initializer？
    → 用 XxxOperator
```

---

# 5. Instruction 类型每日记忆表

## 5.1 算术与逻辑

LLVM 并不是每个二元 opcode 都有一个常用的 `AddInst`、`MulInst` 类。它们通常统一为
`BinaryOperator`。

| IR opcode | 具体 C++ 类型 | 分类 | 关键语义 |
|---|---|---|---|
| `add` | `BinaryOperator` | 整数二元运算 | 可有 `nuw/nsw` |
| `sub` | `BinaryOperator` | 整数二元运算 | 可有 `nuw/nsw` |
| `mul` | `BinaryOperator` | 整数二元运算 | 可有 `nuw/nsw` |
| `udiv` | `BinaryOperator` | 无符号除法 | 可有 `exact`；除零敏感 |
| `sdiv` | `BinaryOperator` | 有符号除法 | `INT_MIN/-1`、除零；可 exact |
| `urem` | `BinaryOperator` | 无符号余数 | 除零敏感 |
| `srem` | `BinaryOperator` | 有符号余数 | 除零敏感 |
| `shl` | `BinaryOperator` | 左移 | shift amount、nuw/nsw、poison |
| `lshr` | `BinaryOperator` | 逻辑右移 | 可 exact |
| `ashr` | `BinaryOperator` | 算术右移 | 可 exact |
| `and` | `BinaryOperator` | bitwise | 可交换 |
| `or` | `BinaryOperator` | bitwise | 可交换 |
| `xor` | `BinaryOperator` | bitwise | 可交换 |
| `fadd` | `BinaryOperator` | 浮点 | FMF、NaN、signed zero |
| `fsub` | `BinaryOperator` | 浮点 | FMF |
| `fmul` | `BinaryOperator` | 浮点 | FMF |
| `fdiv` | `BinaryOperator` | 浮点 | FMF |
| `frem` | `BinaryOperator` | 浮点 | FMF |
| `fneg` | `UnaryOperator` | 浮点一元 | FMF |

标准读取：

```cpp
if (auto *BO = dyn_cast<BinaryOperator>(V)) {
  unsigned Opc = BO->getOpcode();
  Value *LHS = BO->getOperand(0);
  Value *RHS = BO->getOperand(1);

  switch (Opc) {
  case Instruction::Add:
  case Instruction::Mul:
    break;
  }
}
```

## 5.2 比较、选择与 PHI

| 类型 | IR | 关键 API | 记忆点 |
|---|---|---|---|
| `CmpInst` | 所有 compare | `getPredicate()` | ICmp/FCmp 公共父类 |
| `ICmpInst` | `icmp` | signed/unsigned predicate | 整数和 pointer 比较 |
| `FCmpInst` | `fcmp` | ordered/unordered predicate | NaN 影响结果 |
| `SelectInst` | `select` | condition/true/false | 当前块按 i1 选值 |
| `PHINode` | `phi` | incoming value/block | 按 incoming edge 选值 |
| `FreezeInst` | `freeze` | operand | 固定 undef/poison 的一次选择 |

```cpp
if (auto *Cmp = dyn_cast<ICmpInst>(V)) {
  if (Cmp->getPredicate() == ICmpInst::ICMP_SLT)
    ...;
}

if (auto *PN = dyn_cast<PHINode>(V)) {
  for (unsigned I = 0; I < PN->getNumIncomingValues(); ++I) {
    Value *IncomingV = PN->getIncomingValue(I);
    BasicBlock *IncomingBB = PN->getIncomingBlock(I);
  }
}
```

## 5.3 内存与地址

| 类型 | IR | 是否访问内存 | 高频 API |
|---|---|---:|---|
| `AllocaInst` | `alloca` | 分配 stack object | `getAllocatedType()`、`getAlign()` |
| `LoadInst` | `load` | 读 | `getPointerOperand()`、`isVolatile()` |
| `StoreInst` | `store` | 写 | `getValueOperand()`、`getPointerOperand()` |
| `GetElementPtrInst` | `getelementptr` | 否 | 与 `GEPOperator` API 类似 |
| `FenceInst` | `fence` | 顺序约束 | `getOrdering()` |
| `AtomicCmpXchgInst` | `cmpxchg` | 原子读写 | compare/new value/orderings |
| `AtomicRMWInst` | `atomicrmw` | 原子读改写 | operation/value/ordering |

```cpp
if (auto *LI = dyn_cast<LoadInst>(V)) {
  Value *Ptr = LI->getPointerOperand();
  Type *LoadedTy = LI->getType();
  Align A = LI->getAlign();
}

if (auto *SI = dyn_cast<StoreInst>(V)) {
  Value *Stored = SI->getValueOperand();
  Value *Ptr = SI->getPointerOperand();
}
```

## 5.4 Cast 指令

| 类型 | IR | 核心含义 |
|---|---|---|
| `CastInst` | 所有 cast | 公共父类 |
| `TruncInst` | `trunc` | 整数截断 |
| `ZExtInst` | `zext` | 零扩展 |
| `SExtInst` | `sext` | 符号扩展 |
| `FPTruncInst` | `fptrunc` | 浮点精度截断 |
| `FPExtInst` | `fpext` | 浮点精度扩展 |
| `FPToUIInst` | `fptoui` | FP → unsigned integer |
| `FPToSIInst` | `fptosi` | FP → signed integer |
| `UIToFPInst` | `uitofp` | unsigned integer → FP |
| `SIToFPInst` | `sitofp` | signed integer → FP |
| `PtrToIntInst` | `ptrtoint` | pointer representation → integer |
| `IntToPtrInst` | `inttoptr` | integer → pointer |
| `BitCastInst` | `bitcast` | 保持 bit pattern 的合法类型重解释 |
| `AddrSpaceCastInst` | `addrspacecast` | 跨 address space pointer cast |

```cpp
if (auto *CI = dyn_cast<CastInst>(V)) {
  Value *Src = CI->getOperand(0);
  Type *SrcTy = Src->getType();
  Type *DstTy = CI->getType();
  unsigned CastOpcode = CI->getOpcode();
}
```

## 5.5 Call-like 指令

```text
CallBase
├── CallInst
├── InvokeInst
└── CallBrInst
```

| 类型 | 什么时候用 |
|---|---|
| `CallBase` | 逻辑适用于所有调用形式时，默认首选 |
| `CallInst` | 只处理普通 `call` |
| `InvokeInst` | 需要 normal/unwind CFG edge |
| `CallBrInst` | inline asm 等带多个间接目标的 callbr |
| `IntrinsicInst` | 只处理 LLVM intrinsic call |

```cpp
if (auto *CB = dyn_cast<CallBase>(V)) {
  Function *Direct = CB->getCalledFunction(); // 间接调用时 null
  Value *Called = CB->getCalledOperand();
  for (Use &Arg : CB->args())
    inspect(Arg.get());
}
```

Intrinsic 常用子类：

| 类型 | 覆盖内容 |
|---|---|
| `IntrinsicInst` | 所有 LLVM intrinsic calls |
| `DbgInfoIntrinsic` | debug intrinsic |
| `MemIntrinsic` | memcpy/memmove/memset 公共类 |
| `MemTransferInst` | memcpy/memmove 公共类 |
| `MemCpyInst` | memcpy |
| `MemMoveInst` | memmove |
| `MemSetInst` | memset |
| `LifetimeIntrinsic` | lifetime.start/end |
| `AssumeInst` | llvm.assume |

## 5.6 Terminator 与异常控制流

| 类型 | IR | 关键点 |
|---|---|---|
| `BranchInst` | `br` | conditional/unconditional |
| `SwitchInst` | `switch` | default + case successors |
| `IndirectBrInst` | `indirectbr` | address-selected destination |
| `ReturnInst` | `ret` | return value 可能为 null |
| `UnreachableInst` | `unreachable` | 到达即 UB/不可达语义 |
| `ResumeInst` | `resume` | 继续异常展开 |
| `CatchSwitchInst` | `catchswitch` | Windows EH dispatch |
| `CatchReturnInst` | `catchret` | catch funclet return |
| `CleanupReturnInst` | `cleanupret` | cleanup funclet return |

通用判断：

```cpp
if (I->isTerminator()) {
  for (BasicBlock *Succ : successors(I->getParent()))
    ...;
}
```

需要 successor API 时也可处理 `TerminatorInst` 相关接口，但 LLVM 22 代码中通用逻辑常直接
使用 `Instruction::isTerminator()`、`BasicBlock::getTerminator()` 和 CFG iterator。

## 5.7 Vector 与 Aggregate 指令

| 类型 | IR | 作用 |
|---|---|---|
| `ExtractElementInst` | `extractelement` | 从 vector 取 lane |
| `InsertElementInst` | `insertelement` | 向 vector 写 lane，产生新 vector |
| `ShuffleVectorInst` | `shufflevector` | 按 mask 重排/组合 vector |
| `ExtractValueInst` | `extractvalue` | 从 struct/array aggregate 取字段 |
| `InsertValueInst` | `insertvalue` | 向 aggregate 写字段，产生新 aggregate |

不要混淆：

```text
extractelement / insertelement → vector，index 是运行时 Value
extractvalue / insertvalue     → aggregate，indices 是常量路径
```

## 5.8 EH 与特殊指令

| 类型 | 作用 |
|---|---|
| `LandingPadInst` | Itanium-style EH landing pad |
| `CatchPadInst` | catch funclet pad |
| `CleanupPadInst` | cleanup funclet pad |
| `VAArgInst` | 从 `va_list` 取下一个参数 |

这些类型带有严格 CFG/placement 规则，不能像普通纯计算指令一样移动或克隆。

---

# 6. Constant 与 GlobalValue 每日记忆表

## 6.1 Constant 体系

```text
Constant
├── ConstantInt
├── ConstantFP
├── ConstantPointerNull
├── ConstantAggregateZero
├── ConstantDataSequential
│   ├── ConstantDataArray
│   └── ConstantDataVector
├── ConstantAggregate
│   ├── ConstantArray
│   ├── ConstantStruct
│   └── ConstantVector
├── ConstantExpr
├── UndefValue
├── PoisonValue
├── BlockAddress
└── GlobalValue
```

| 类型 | 先这样理解 | 高频 API |
|---|---|---|
| `ConstantInt` | 任意位宽整数常量 | `getValue()`、`isZero()`、`getZExtValue()` |
| `ConstantFP` | 任意 FP 语义常量 | `getValueAPF()` |
| `ConstantPointerNull` | 某 pointer type 的 null | `getType()` |
| `ConstantAggregateZero` | aggregate 全零初始化 | element type |
| `ConstantDataArray` | 紧凑常量数组，常用于字符串 | `isString()`、`getAsString()` |
| `ConstantExpr` | 常量组成的运算表达式 | `getOpcode()`、`getAsInstruction()` |
| `UndefValue` | 每个 use 可取任意允许值 | 不等于 poison |
| `PoisonValue` | 延迟传播的错误值 | 敏感 use 可导致 UB |
| `BlockAddress` | 某函数内 BasicBlock 的地址 | indirectbr 等 |

## 6.2 `ConstantExpr`

```cpp
if (auto *CE = dyn_cast<ConstantExpr>(V)) {
  unsigned Opc = CE->getOpcode();

  // 如果某算法只会处理 Instruction，可临时转换成未插入的 Instruction。
  std::unique_ptr<Instruction> Temp(CE->getAsInstruction());
  inspect(*Temp);
}
```

注意 ownership：`getAsInstruction()` 创建新 instruction，调用者负责其生命周期；若插入 IR，
则改由 parent 拥有。能直接使用 `GEPOperator` 等统一接口时，不必为了读取信息而转换。

## 6.3 GlobalValue

```text
GlobalValue
├── Function
├── GlobalVariable
├── GlobalAlias
└── GlobalIFunc
```

| 类型 | 含义 |
|---|---|
| `Function` | 函数符号，可能是 declaration 或 definition |
| `GlobalVariable` | 全局对象的地址，initializer 是另一个 Constant |
| `GlobalAlias` | 指向另一个 global symbol 的别名 |
| `GlobalIFunc` | 间接函数，由 resolver 选择实现 |

函数和全局变量也是 `Constant`/`Value` 体系的一部分，但“GlobalVariable 是 Constant”不表示
它指向的内存内容不可变。

---

# 7. Analysis 类型体系每日记忆表

`dyn_cast` 不只用于 IR。看到陌生类型时先判断当前指针属于哪套体系。

## 7.1 SCEV

```text
SCEV
├── SCEVConstant
├── SCEVUnknown
├── SCEVAddExpr
├── SCEVMulExpr
├── SCEVUDivExpr
├── SCEVAddRecExpr
├── SCEVTruncateExpr
├── SCEVZeroExtendExpr
├── SCEVSignExtendExpr
├── SCEVPtrToIntExpr
├── SCEVMinMaxExpr 的具体类型
└── SCEVCouldNotCompute
```

| 类型 | 含义 | 高频 API |
|---|---|---|
| `SCEVConstant` | 常量 SCEV | `getAPInt()`/constant value |
| `SCEVUnknown` | 叶子 IR Value | `getValue()` |
| `SCEVAddExpr` | canonical 加法表达式 | `operands()` |
| `SCEVMulExpr` | canonical 乘法表达式 | `operands()` |
| `SCEVAddRecExpr` | loop recurrence | `getStart()`、`getStepRecurrence()`、`getLoop()` |
| `SCEVCouldNotCompute` | 分析无法计算 | 必须保守退出 |

```cpp
const SCEV *S = SE.getSCEV(V);

if (auto *AR = dyn_cast<SCEVAddRecExpr>(S)) {
  const SCEV *Start = AR->getStart();
  const SCEV *Step = AR->getStepRecurrence(SE);
}
```

## 7.2 MemorySSA

```text
MemoryAccess
├── MemoryUseOrDef
│   ├── MemoryUse
│   └── MemoryDef
└── MemoryPhi
```

| 类型 | 含义 |
|---|---|
| `MemoryUse` | 只读 memory access 的分析节点 |
| `MemoryDef` | 写或可能写 memory 的分析节点 |
| `MemoryPhi` | CFG 汇合处的 memory state |
| `MemoryUseOrDef` | Use/Def 的公共操作接口 |

```cpp
MemoryAccess *MA = MSSA.getMemoryAccess(I);

if (auto *MU = dyn_cast_or_null<MemoryUse>(MA)) {
  ...
} else if (auto *MD = dyn_cast_or_null<MemoryDef>(MA)) {
  ...
} else if (auto *MP = dyn_cast_or_null<MemoryPhi>(MA)) {
  ...
}
```

这些不是 `Instruction`，不能调用 `getParent()` 后就假定取得 BasicBlock 中的普通指令；使用
MemoryAccess/MemorySSA 自己的接口。

---

# 8. PatternMatch 与 `dyn_cast` 的关系

简单类型识别适合 `dyn_cast`：

```cpp
if (auto *BO = dyn_cast<BinaryOperator>(V)) {
  if (BO->getOpcode() == Instruction::Add) {
    ...
  }
}
```

表达式树匹配适合 `PatternMatch`：

```cpp
using namespace llvm::PatternMatch;

Value *X;
ConstantInt *C;
if (match(V, m_Add(m_Value(X), m_ConstantInt(C)))) {
  ...
}
```

| 工具 | 最适合的问题 |
|---|---|
| `dyn_cast<T>` | “这个单个对象实际是什么类型？” |
| opcode switch | “这个 instruction/operator 是哪种 opcode？” |
| `PatternMatch` | “这棵表达式是否具有某种结构？” |
| `InstVisitor` | “我要为许多 Instruction 类型定义 visitor 回调” |

复杂代码出现大量嵌套 `dyn_cast` 时，可以考虑 PatternMatch 或 InstVisitor；但它们不会自动
替你处理 dominance、poison、memory 和 flags。

---

# 9. 每日必背 20 个类型

| # | 类型 | 一句话记忆 |
|---:|---|---|
| 1 | `Value` | 任意可被引用的 LLVM 值 |
| 2 | `User` | 使用其他 Value 的 Value |
| 3 | `Instruction` | BasicBlock 中的 IR 指令 |
| 4 | `Operator` | Instruction/ConstantExpr 的统一运算观察层 |
| 5 | `BinaryOperator` | add/sub/mul/shift/bitwise/FP 二元 instruction |
| 6 | `OverflowingBinaryOperator` | 可携带 nuw/nsw 的 add/sub/mul/shl 语义 |
| 7 | `PossiblyExactOperator` | 可携带 exact 的 div/right-shift 语义 |
| 8 | `FPMathOperator` | 统一读取 Fast-Math Flags |
| 9 | `GetElementPtrInst` | 函数体内的 GEP instruction |
| 10 | `GEPOperator` | Instruction/ConstantExpr 的统一 GEP 语义 |
| 11 | `LoadInst` | 读取内存 |
| 12 | `StoreInst` | 写内存 |
| 13 | `PHINode` | 按 incoming edge 选择 SSA value |
| 14 | `ICmpInst` | 整数或 pointer 比较 |
| 15 | `CallBase` | call/invoke/callbr 公共基类 |
| 16 | `CastInst` | 所有 cast instruction 公共基类 |
| 17 | `ConstantInt` | APInt 整数常量 |
| 18 | `ConstantExpr` | 常量组成的运算表达式 |
| 19 | `SCEVAddRecExpr` | loop 中 `{Start,+,Step}` 递推 |
| 20 | `MemoryDef/Use/Phi` | MemorySSA 写、读、汇合节点 |

---

# 10. 按代码意图选择类型

| 我想做什么 | 首选类型/写法 |
|---|---|
| 判断是否为函数体内 GEP | `dyn_cast<GetElementPtrInst>(V)` |
| 读取任意 GEP，包括 ConstantExpr | `dyn_cast<GEPOperator>(V)` |
| 删除一条 add instruction | `BinaryOperator` + opcode |
| 读取任意 add 的 nowrap flags | `dyn_cast<AddOperator>(V)` |
| 读取 div/shift 的 exact flag | `dyn_cast<PossiblyExactOperator>(V)` |
| 读取浮点运算的 FMF | `dyn_cast<FPMathOperator>(V)` |
| 处理所有 call-like 指令 | `dyn_cast<CallBase>(V)` |
| 只处理普通 call | `dyn_cast<CallInst>(V)` |
| 处理所有 cast instruction | `dyn_cast<CastInst>(V)` |
| 读取 instruction/ConstantExpr bitcast | `dyn_cast<BitCastOperator>(V)` |
| 判断整数常量 0 | `match(V, m_Zero())` 或 `ConstantInt::isZero()` |
| 识别 add 表达式树 | `PatternMatch::m_Add(...)` |
| 判断 loop recurrence | `dyn_cast<SCEVAddRecExpr>(S)` |
| 判断 MemorySSA 节点类型 | `dyn_cast<MemoryUse/Def/Phi>(MA)` |

---

# 11. 阅读陌生 `dyn_cast<T>` 的固定动作

看到：

```cpp
if (auto *X = dyn_cast<SomeOperator>(V)) {
  ...
}
```

按五步读：

```text
1. V 的静态类型是什么？Value、Instruction、SCEV 还是 MemoryAccess？
2. SomeOperator 定义在哪个头文件？
3. 它的 classof() 接受哪些 opcode/节点？
4. 转换后代码调用了哪些专属 API？
5. 代码只读取语义，还是会修改/删除真实 Instruction？
```

仓库搜索：

```bash
# 找定义和 classof
rg -n "class GEPOperator|classof" \
  jeandle-llvm/llvm/include/llvm/IR/Operator.h

# 找生产代码中的用法
rg -n "dyn_cast<GEPOperator>|isa<GEPOperator>|cast<GEPOperator>" \
  jeandle-llvm/llvm/lib

# 找单元测试
rg -n "GEPOperator" jeandle-llvm/llvm/unittests
```

## 阅读记录模板

| 字段 | 示例：GEPOperator |
|---|---|
| 所属体系 | IR Operator utility view |
| 输入父类型 | 通常 `Value *` / `User *` |
| 匹配范围 | GEP Instruction + GEP ConstantExpr |
| IR 语义 | 只计算派生地址，不访问内存 |
| 高频 API | base、source type、indices、inbounds、offset |
| 能否 erase | 不能仅凭 GEPOperator；先确认是 Instruction |
| 易错点 | opaque pointer、DataLayout、inbounds poison |

---

# 12. 自测题

## 第一组：判断该用哪个类型

1. 希望同时处理函数中的 GEP 和全局 initializer 中的 GEP。
2. 希望把匹配到的 GEP 从 BasicBlock 删除。
3. 希望处理 `call` 和 `invoke`。
4. 希望读取 `add` 的 `nsw`，输入可能是 ConstantExpr。
5. 希望判断一个 SCEV 是否为 loop recurrence。

答案：

```text
1. GEPOperator
2. GetElementPtrInst（或先 dyn_cast<Instruction> 再删除）
3. CallBase
4. AddOperator
5. SCEVAddRecExpr
```

## 第二组：下面代码有什么问题

```cpp
if (auto *GEP = dyn_cast<GEPOperator>(V))
  GEP->eraseFromParent();
```

答案：`GEPOperator` 可能观察的是 `ConstantExpr`，也不提供普通 instruction mutation API。
如果确实要删除指令：

```cpp
if (auto *GEP = dyn_cast<GetElementPtrInst>(V))
  GEP->eraseFromParent();
```

删除前还要保证 uses、debug information 和分析维护正确。

## 第三组：为什么下面代码可能漏匹配

```cpp
if (auto *GEP = dyn_cast<GetElementPtrInst>(Initializer)) {
  ...
}
```

答案：Global initializer 是 Constant；其中的 GEP 通常是 `ConstantExpr`，不是
`GetElementPtrInst`。应使用 `GEPOperator` 或先处理 `ConstantExpr`。

## 第四组：为什么不能统一使用 Operator 修改 IR

答案：Operator 是跨 `Instruction`/`ConstantExpr` 的只读语义视图。Instruction 属于
BasicBlock，可以移动和删除；ConstantExpr 属于常量图，修改方式、唯一化和 ownership 不同。
需要 mutation 时必须先区分真实对象类型。

## 第五组：口述检查

看到下面代码，用一句话解释每行在缩小什么语义范围：

```cpp
if (auto *Op = dyn_cast<Operator>(V))
  if (auto *GEP = dyn_cast<GEPOperator>(Op))
    if (auto *I = dyn_cast<Instruction>(GEP))
      if (DT.dominates(I, UseI))
        ...;
```

参考答案：

```text
V 首先必须是 Instruction 或 ConstantExpr；
然后必须具有 GEP opcode；
再限制为真正位于函数体中的 GEP instruction；
最后检查它是否支配目标 use instruction。
```

---

# 13. 一页打印速查表

```text
LLVM dyn_cast 三问
────────────────────────────────────────────────────────────
1. 输入属于哪套体系？ IR / SCEV / MemorySSA / Machine IR
2. 我要具体节点，还是统一运算语义？ XxxInst / XxxOperator
3. 我要只读，还是修改 IR？ Operator 适合观察，Instruction 才能移动删除

Operator 核心
────────────────────────────────────────────────────────────
Operator                    Instruction 或 ConstantExpr
OverflowingBinaryOperator   add/sub/mul/shl；nuw/nsw
PossiblyExactOperator       udiv/sdiv/lshr/ashr；exact
FPMathOperator              Fast-Math Flags；当前匹配 Instruction
GEPOperator                 GEP Inst + GEP ConstantExpr
PtrToIntOperator            ptrtoint Inst + ConstantExpr
PtrToAddrOperator           ptrtoaddr Inst + ConstantExpr（LLVM 22）
BitCastOperator             bitcast Inst + ConstantExpr
AddrSpaceCastOperator       addrspacecast Inst + ConstantExpr

最常用公共父类
────────────────────────────────────────────────────────────
BinaryOperator              所有二元算术/逻辑 instruction
CmpInst                     ICmpInst + FCmpInst
CastInst                    所有 cast instruction
CallBase                    CallInst + InvokeInst + CallBrInst
MemIntrinsic                memcpy + memmove + memset
MemoryUseOrDef              MemoryUse + MemoryDef

GEP 必背
────────────────────────────────────────────────────────────
GEP 只算地址，不访问内存
operand 0 是 base，后续 operands 是 indices
opaque pointer 下用 getSourceElementType，不查 pointee type
offset 使用 DataLayout + APInt
inbounds/nowrap 是语义承诺，错误时可能产生 poison
需要删除/移动时使用 GetElementPtrInst，不要只持有 GEPOperator
```

每天复习时，只需确认自己能解释三个问题：

1. `GEPOperator` 为什么能匹配 `GetElementPtrInst` 和 `ConstantExpr`？
2. `BinaryOperator` 与 `AddOperator` 的使用边界是什么？
3. 为什么 `Operator` 适合读取语义，却不适合作为修改 IR 的统一接口？
