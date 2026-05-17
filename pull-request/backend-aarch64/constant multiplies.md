这个 PR 可以这样向面试官口头讲，大概 2–3 分钟就能讲清楚动机、改动点、权衡和测试覆盖。

## 动机：为什么要改

这次改动针对的是 AArch64 后端里“常量乘法只用来算内存地址”的场景。  
AArch64 的 load/store 指令本身支持比较强的寻址模式，比如 `base + (index << shift)`，可以把地址计算折到访存指令里，减少单独的 mul/add 指令。 [documentation-service.arm](https://documentation-service.arm.com/static/674d8b61c7fc0d1f211dc776?token=)
原来的实现里，如果一个乘法的唯一用户是 ADD 或 SUB，为了给 madd/msub 这样的乘加指令预留机会，后端会保守地不把这个 mul 拆成 shift+add，这样在算术路径上是安全的，但也会挡住一部分“其实只用于地址计算”的优化机会。

## 改动点：具体做了什么

这次 PR 主要做了两件事。  

第一，是在 AArch64ISelLowering 里新加了一个辅助函数 `isOnlyUsedAsMemoryAddress`。  
它专门检查一个 ADD 或 SUB 节点的所有 use，只有当它：  
- 只被 unindexed 的 load 用作地址操作数（operand 1），或者  
- 只被 unindexed 的 store 用作地址操作数（operand 2），或者  
- 被 prefetch 用作地址操作数（operand 2），  
并且没有任何其他用途时，才认为“这就是一个纯粹的地址”。只要有一个 use 不满足，比如被别的算术指令用掉，或者在 load/store 里不是地址槽位，这个函数就会返回 false。  

第二，是改了 `performMulCombine` 里面原来那段保护逻辑。  
原来是“只要 mul 的唯一 user 是 ADD/SUB，就直接 return，不做拆分”。  
现在改成：  
- 如果 mul 只有一个 user，并且这个 user 是 ADD/SUB，  
- 但是这个 ADD/SUB **不是** `isOnlyUsedAsMemoryAddress`，那还是像以前一样直接返回，继续保留 madd/msub 的机会；  
- 如果这个 ADD/SUB 被检测到是“纯地址”，那就允许后面的 mul 拆分逻辑继续执行，把 `常量乘法 + 加法` 变成更适合 AArch64 地址模式的形式，比如移位加加法，这样 load/store 有机会匹配到 `base + (index << shift)` 这一类寻址。

## 权衡：在什么场景下做、不做

这背后其实是一个很典型的后端权衡。  
对于算术路径，mul + add 更适合 fold 成 madd/msub，这种组合可以减少一条指令，而且没有访存参与，选择 madd/msub 更划算。 [github](https://github.com/llvm/llvm-project/issues/89430)
但对于纯粹的地址计算，尤其是后面紧跟 load/store 的场景，把偏移做成“加移位”形式，更有利于用上 AArch64 的复杂寻址模式，有时候甚至可以把 add 也吸进 load/store，只剩下一条访存指令，这对 pipeline 和功耗往往更好。 [documentation-service.arm](https://documentation-service.arm.com/static/674d8b61c7fc0d1f211dc776?token=)
所以这个 PR 的策略是：  
- 只要 ADD/SUB 有“非地址用途”，就继续走老的保守逻辑，坚决不拆，给 madd/msub 留空间；  
- 只有在 ADD/SUB 被证明“真的只当地址用”的时候，才敢把 mul 拆开放给地址模式优化。  

## 测试覆盖：怎么保证不踩雷

最后，PR 在 AArch64 的 codegen 测试里补充了一个 `mul-const-addressing-mode.ll` 测试。  
里面构造了一些“常量乘法只用于 load/store 地址”的 IR，用来验证：  
- 经过这个 pass 之后，生成的 AArch64 指令确实用上了带 shift 的 load/store 寻址，而不是保留独立的 mul；  
- 同时也覆盖了一些负例，确保在 ADD/SUB 有其他用途的时候，这个优化不会误触发，从而不破坏 madd/msub 这类现有优化。  

如果是跟面试官讲，我一般会用一句话收一下：  
“总结一下，这个 PR 的目标是：当常量乘法只用于地址计算时，把它往 AArch64 的 load/store 寻址模式上推，而在非地址路径上仍然优先保留 madd/msub 的机会，既提升了特定模式下的代码质量，又通过严格的 use 检查和测试保证了安全性。”  

你更想练下一版“更偏技术评审会议”的版本，还是我们先把一个具体 IR 例子拿出来，逐条想象下 pass 前后的 AArch64 指令差异？  

***
这个 PR 是在 AArch64 后端里，专门优化“只用于内存地址计算的常量乘法”，让它们可以被分解成更适合 AArch64 寻址模式的形态，同时避免破坏原有的 madd/msub 折叠机会。

## 修改背景

AArch64 的 load/store 支持多种带偏移的寻址模式，比如 base + (index << shift)，可以把地址计算合在访存指令里完成。  
当中间表示里出现 `index * 常量` 这种乘法，只要它最终只是用来计算 load/store/prefetch 的地址，理论上可以把乘法拆成移位加加法，让最终机器码用上“带 shift 的访存寻址”，减少显式 mul 指令。  
但旧代码里有一段“保守逻辑”：如果 mul 的唯一 user 是 ADD 或 SUB，就直接不做“mul -> shift+add” 的分解，目的是保留 mul+add 组合被折叠到 madd/msub 指令的机会。

问题在于，这种“一刀切”也挡住了“mul 只为地址服务”的场景：明明可以往地址模式方向继续 push，却被这段保护逻辑拦住了。  

## 核心修改逻辑

### 新增辅助函数 `isOnlyUsedAsMemoryAddress`

文件：`llvm/lib/Target/AArch64/AArch64ISelLowering.cpp`。  

新增静态函数：

```c++
static bool isOnlyUsedAsMemoryAddress(SDNode *N) {
  assert((N->getOpcode() == ISD::ADD || N->getOpcode() == ISD::SUB) &&
         "Expected add/sub node");

  for (SDUse &Use : N->uses()) {
    SDNode *User = Use.getUser();
    switch (User->getOpcode()) {
    case ISD::LOAD: {
      auto *Load = cast<LoadSDNode>(User);
      if (!Load->isUnindexed() || Use.getOperandNo() != 1)
        return false;
      break;
    }
    case ISD::STORE: {
      auto *Store = cast<StoreSDNode>(User);
      if (!Store->isUnindexed() || Use.getOperandNo() != 2)
        return false;
      break;
    }
    case AArch64ISD::PREFETCH:
      if (Use.getOperandNo() != 2)
        return false;
      break;
    default:
      return false;
    }
  }

  return true;
}
```

逻辑要点：  

- 只接受 `ISD::ADD` 或 `ISD::SUB` 节点。  
- 遍历这个 ADD/SUB 的所有 uses：  
  - 如果 user 是 `ISD::LOAD`：  
    - 必须 `isUnindexed()`，确保是未索引 load（不自增/不写回基址）。  
    - 当前 use 必须是 operand 1，也就是“地址”操作数（operand 0 是链）。  
  - 如果 user 是 `ISD::STORE`：  
    - 必须 `isUnindexed()`。  
    - 当前 use 必须是 operand 2，store 的地址在 operand 2。  
  - 如果 user 是 `AArch64ISD::PREFETCH`：  
    - 当前 use 必须是 operand 2（prefetch 地址）。  
  - 任何其他 opcode 或错误的 operand 位置，立即返回 false。  
- 所有 uses 都符合上面条件，才返回 true，表示“这个 add/sub 只用作内存地址”。  

这等价于一个非常严格的白名单：  
“只要这个 ADD/SUB 被用在别的算术/逻辑/比较上，或者在 load/store 中不是地址位，就不算纯地址”。  

### 调整 `performMulCombine` 的保护逻辑

同一文件里的 `performMulCombine` 本来有一段保护逻辑：

```c++
 // Conservatively do not lower to shift+add+shift if the mul might be
 // folded into madd or msub.
 if (N->hasOneUse() && (N->user_begin()->getOpcode() == ISD::ADD ||
                        N->user_begin()->getOpcode() == ISD::SUB))
   return SDValue();
```

这次修改为：  

```c++
if (N->hasOneUse()) {
  SDNode *User = *N->user_begin();
  if ((User->getOpcode() == ISD::ADD || User->getOpcode() == ISD::SUB) &&
      !isOnlyUsedAsMemoryAddress(User))
    return SDValue();
}
```

对比可以看到：  

- 仍然只在 `N->hasOneUse()` 时进入这段逻辑。  
- 取出唯一的 user `User`，如果它是 ADD/SUB 并且 **不是** “仅用于地址”（`!isOnlyUsedAsMemoryAddress(User)`），就继续保守：直接 `return SDValue()`，不拆分 mul。  
- 但如果 `User` 是 ADD/SUB 且 `isOnlyUsedAsMemoryAddress(User)` 为 true，就不会提前返回，从而允许后面的“mul -> shift+add(+shift)” 合并继续进行。  

这相当于在原来的“保护 madd/msub”逻辑上打了一个洞：  

- 非纯地址的 ADD/SUB：继续像以前一样保守，给 madd/msub 留空间。  
- 纯地址的 ADD/SUB：放行，让 mul 可以被拆成更适合地址模式的组合。  

### 测试补充

文件：`test/CodeGen/AArch64/mul-const-addressing-mode.ll`。  

PR 在这个测试里新增/调整了用例，用来覆盖：  

- 常量乘法只被用于某个 load/store 的地址。  
- 检查最终的 AArch64 汇编是否生成了类似 `ldr x?, [xbase, xindex, lsl #shift]` 这种 form，而不是单独的 mul 再 add。  

这确保 `isOnlyUsedAsMemoryAddress` 的逻辑有实际覆盖，同时验证不会破坏其他组合情况。  

## 相关知识点拆开讲

### 1. SelectionDAG / SDNode / SDUse 基本概念

- `SDNode`：SelectionDAG 里的节点，一个节点代表一个操作（例如 ISD::MUL、ISD::ADD、ISD::LOAD）。  
- `SDUse`：节点之间的边，每个 use 表示“某个节点在另一个节点的某个 operand 位置被使用”。  
- `Use.getOperandNo()`：指出这个 use 是 user 节点的第几个 operand，比如 load：  
  - operand 0：链（chain）；  
  - operand 1：地址；  
  - operand 2：内建 pointer info 或其他。  

这种“operand slot 精确匹配”是这次函数的核心：它要确保 N 只是被 plug 到那个“地址槽位”上，没有参与其他算术运算。  

### 2. unindexed load/store 的含义

- AArch64 有类似 post-indexed/pre-indexed 的 load/store，可以在访存同时更新基址。  
- 在 SelectionDAG 里，`isUnindexed()` 意味着这个 load/store 没有做额外的地址更新，而是单纯地用当前地址进行访问。  
- PR 要求 load/store 必须是 unindexed 才算“纯地址使用”；否则，这个 ADD/SUB 可能还是参与了地址更新语义，不再是纯粹的计算偏移。  

### 3. madd/msub vs “地址模式” 的权衡

- madd/msub：典型形态是 `dst = src0 * src1 + src2` 或 `dst = src2 - src0 * src1`，AArch64 有专门的 MAC（乘加）指令支持这种组合。  
- 地址模式（如 `ldr x0, [x1, x2, lsl #2]`）：把 base + (index << shift) 合在访存指令里完成。  
- 原先逻辑：看到 mul+add，就认为未来可能折叠成 madd/msub，于是拒绝把 mul 拆成 shift+add。  
- 现在的改动：  
  - 如果这个 add/sub 是算术路径的一部分，就继续倾向 madd/msub。  
  - 如果 add/sub 是纯地址路径，则倾向于“地址模式优化”：让 load/store 直接吃掉偏移，从而减少独立算术指令。  

这就是“针对用途不同，选择不同的合并方向”的典型后端 heuristics。  

## 一个直观的例子描述（逻辑层面）

假设 IR / DAG 上有这样的结构（简化想象）：  

- `idx_mul = idx * 4`  
- `addr = base + idx_mul`  
- `val = load addr`  

并且 `addr` 只被这一条 load 用来做地址，且为 unindexed。  

在老逻辑下：  
- `idx_mul` 唯一 user 是 `addr`（ADD）；  
- 遵守“mul+add 可能被折叠成 madd”的保守规则，不拆分 mul；  
- 后端可能生成：  
  - `mul xTmp, xIdx, #4`  
  - `add xAddr, xBase, xTmp`  
  - `ldr xVal, [xAddr]`  

在新逻辑下：  
- 检查 ADD `addr`：发现它只作为 load 的地址 operand1，并且 load 是 unindexed；  
- `isOnlyUsedAsMemoryAddress(addr) == true`，于是允许 `performMulCombine` 把 mul 拆成换成 shift+add 或其它形式，使后续 load 能选到带移位的地址模式；  
- 最终代码可能变成类似：  
  - `add xAddr, xBase, xIdx, lsl #2`  或  直接 `ldr xVal, [xBase, xIdx, lsl #2]`（具体 pattern 由后续匹配决定）。  

这样就节省了一条 mul 指令，甚至可能连 add 也被吸到 ldr 里去，只剩一条访存。  

## PR 信息与链接

- 标题：`[AArch64] Decompose constant multiplies used only by memory addresses`。  
- 仓库：`llvm/llvm-project`。  
- PR 编号：#194584。  
- 作者：JiangNingHX，合并者：efriedma-quic。  
- 变更文件：  
  - `llvm/lib/Target/AArch64/AArch64ISelLowering.cpp`  
  - `test/CodeGen/AArch64/mul-const-addressing-mode.ll`。  
- 链接：<https://github.com/llvm/llvm-project/pull/194584>  

如果你接下来想做“PR 自述 + 面试讲解”的练习，我可以帮你用英文组织一段 2–3 分钟的口头说明稿，重点围绕：动机、改动点、权衡和测试覆盖。你更希望我偏“口语化讲给面试官听”，还是偏“review meeting 上和 reviewer 讨论”的风格？  
