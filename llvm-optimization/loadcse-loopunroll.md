这个函数本质上是在某个循环里做「Load 的基于 SCEV 的 CSE」，沿着支配树 DFS 一路下去，维护一个“当前可复用的 load 集合”，遇到等价 load 就用之前的结果替换，遇到可能写内存的指令就把这一集合整体“失效”到新的世代（generation）。 [github](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Analysis/Loads.cpp)

我用「整体流程 → StackNode 的作用 → while 循环三种分支」三步帮你理一遍，你可以对照着源码一起看。

***

## 整体流程在干什么

- 输入：  
  - 一个循环 `L`，支配树 `DT`，标量演化 `SE`，LoopInfo `LI`，别名分析 `BAA`，以及 `GetMSSA()`（需要时拿 `MemorySSA`）。 [llvm](https://llvm.org/doxygen/classllvm_1_1ScalarEvolution.html)
- 维护的数据结构：  
  - `ScopedHashTable<const SCEV *, LoadValue> AvailableLoads;`  
    - 键：指向地址的 SCEV（`PtrSCEV`）  
    - 值：`LoadValue`，里面记录这个 load 的 `Value*` 和当时的世代号 `Generation`。  
  - `SmallVector<std::unique_ptr<StackNode>> NodesToProcess;`  
    - 实现一个“手写的 DFS 栈”，每个栈帧表示支配树上的一个节点 + 迭代位置。 [zhuanlan.zhihu](https://zhuanlan.zhihu.com/p/1996741886271066865)

核心思想：  

1. 从循环 header 对应的 DomTreeNode 开始，把它压栈。  
2. 用一个 `CurrentGeneration` 来表示“目前这个 basic block 上的内存环境版本号”。  
3. DFS 遍历支配树：  
   - 扫当前块里的指令：  
     - 如果是简单 load，就尝试用已有的可用 load 替换（CSE）；如果不能替换，就把它记到 `AvailableLoads` 中；  
     - 如果遇到「可能写内存」的指令，就把 `CurrentGeneration++`，表示之前存的 load 可能被 clobber，需要新的世代号。  
   - 处理完当前块后，记录下孩子的起始 generation，然后去递归处理支配树的子节点。  
4. 遍历结束，所有可以 CSE 的 load 都被替换掉了。 [github](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Analysis/Loads.cpp)

***

## StackNode 和 ScopedHashTable 的角色

你代码里的 `StackNode` 没展开，但从用法可以看出它负责三件事： [zhuanlan.zhihu](https://zhuanlan.zhihu.com/p/1996741886271066865)

- 保存当前支配树节点：`DomTreeNode *node()`  
- 保存本节点内对 `AvailableLoads` 的 scope（进入/离开节点时自动 push/pop scope）  
- 保存 DFS 状态：  
  - 是否已经处理过当前块 `isProcessed()`  
  - 孩子迭代器 `childIter()/end()/nextChild()`  
  - 进入子节点时应该继承的 `Generation`（`childGeneration()`）  
  - 当前节点刚处理完时的 `currentGeneration()`  

`ScopedHashTable` 的设计是：每当你新建一个 `StackNode` 时，就会 push 一个新的 scope；当这个 `StackNode` 出栈销毁时，相应的 scope 自动 pop，恢复上一层的可用 load 集合。这样保证 CSE 信息在支配树上是「带作用域层级的继承」。 [zhuanlan.zhihu](https://zhuanlan.zhihu.com/p/1996741886271066865)

***

## while 主循环的三种状态

主循环：

```c++
while (!NodesToProcess.empty()) {
  StackNode *NodeToProcess = &*NodesToProcess.back();

  CurrentGeneration = NodeToProcess->currentGeneration();

  if (!NodeToProcess->isProcessed()) {
    // 1. 第一次遇到这个支配树节点：处理当前 basic block
    ...
  } else if (NodeToProcess->childIter() != NodeToProcess->end()) {
    // 2. 当前块处理完了，还有孩子没处理：压栈处理一个子节点
    ...
  } else {
    // 3. 当前块和所有孩子都处理完了：弹栈
    NodesToProcess.pop_back();
  }
}
```

这就是一个典型的手写 DFS：  
- 分支 1：处理自己  
- 分支 2：处理一个子结点  
- 分支 3：回溯。 [github](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Analysis/Loads.cpp)

下面重点看「处理自己」那一段。

***

## 处理一个 DomTree 节点（一个 basic block）

进入第一个大 `if`：

```c++
if (!NodeToProcess->isProcessed()) {
  // Process the node.
```

### 1）根据前驱数决定是否 bump generation

```c++
  // If this block has a single predecessor, then the predecessor is the
  // parent of the domtree node and all of the live out memory values are
  // still current in this block.  If this block has multiple predecessors,
  // ... be conservative and invalidate memory ...
  if (!NodeToProcess->node()->getBlock()->getSinglePredecessor())
    ++CurrentGeneration;
```

- 若当前 BB 只有一个前驱，那么在支配树上，这个前驱就是父节点。父节点结束时的 memory state 可以无损地沿用到当前块，所以不需要马上 invalidation。  
- 若有多个前驱，则不同路径可能写了不同的内存，所以“父节点的 live-out load 结果”在这里不再可靠，于是通过 `++CurrentGeneration` 来「逻辑上」使之前记录的 load 全部变旧世代。 [zhuanlan.zhihu](https://zhuanlan.zhihu.com/p/1996741886271066865)

这里的 generation 相当于「全局版本号」：  
- 每当可能有写入发生而你不方便精确分析，就给 version++；  
- 每个 `LoadValue` 记录它当时的 version；  
- 后面做 CSE 时只有「version 相同」的 load 才被视为 still valid。

### 2）遍历块中指令

```c++
  for (auto &I : make_early_inc_range(*NodeToProcess->node()->getBlock())) {

    auto *Load = dyn_cast<LoadInst>(&I);
    if (!Load || !Load->isSimple()) {
      if (I.mayWriteToMemory())
        CurrentGeneration++;
      continue;
    }
```

- 用 `make_early_inc_range` 是因为可能在遍历过程中删掉 `Load`，需要安全地提前递增迭代器。 [zhuanlan.zhihu](https://zhuanlan.zhihu.com/p/1996741886271066865)
- 只关心「简单 load」：  
  - `LoadInst` 且 `isSimple()`（非 volatile、非原子）  
- 对非简单 load：  
  - 如果是 `mayWriteToMemory()` 的指令，就 `CurrentGeneration++`，表示内存可能被 clobber；  
  - 然后 `continue`。  

所以 generation 的变更发生在两种地方：  
- BB 入口，遇到多前驱  
- 块内部，每个可能写内存的指令之后。 [github](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Analysis/Loads.cpp)

### 3）对简单 Load 做基于 SCEV 的 CSE

```c++
    const SCEV *PtrSCEV = SE.getSCEV(Load->getPointerOperand());
    LoadValue LV = AvailableLoads.lookup(PtrSCEV);
    if (Value *M =
            getMatchingValue(LV, Load, CurrentGeneration, BAA, GetMSSA)) {
      if (LI.replacementPreservesLCSSAForm(Load, M)) {
        Load->replaceAllUsesWith(M);
        Load->eraseFromParent();
      }
    } else {
      AvailableLoads.insert(PtrSCEV, LoadValue(Load, CurrentGeneration));
    }
```

逻辑分成两步：

1. 用 ScalarEvolution 把 `Load->getPointerOperand()` 转成一个 SCEV 表达式作为 key：  
   - 即便指针表达式语法不同，只要 SE 认为是同一个地址（同一 AddRec/常量偏移等），就得到同一个 SCEV。  
2. 在 `AvailableLoads` 中查这个 SCEV：  
   - 如果找到 `LV`，就调用 `getMatchingValue` 看看这个旧的 load 在当前 `CurrentGeneration` 下是否仍然有效：  
     - 会考虑 generation（写内存是否在其后）、alias analysis（`BatchAAResults`）、可能还会用 `MemorySSA` 做更精准的检查。 [github](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Analysis/Loads.cpp)
   - `getMatchingValue` 返回一个可以复用的 `Value *M`：  
     - 再用 `LI.replacementPreservesLCSSAForm(Load, M)` 确保替换不会破坏 LCSSA。  
     - ok 的话，用 `replaceAllUsesWith` + `eraseFromParent` 删除冗余 load，实现 CSE。  
   - 如果没找到可用值：  
     - 把这次 load 作为新的「可复用源」，插入 `AvailableLoads`，记录当前的 generation。  

所以行间翻译就是：  
- 「看看当前这个地址以前有没有 load 过且仍然有效，如果有就直接用以前的结果；没有就把这次 load 记下来，后面兄弟/孩子节点可以用。」 [zhuanlan.zhihu](https://zhuanlan.zhihu.com/p/1996741886271066865)

### 4）记录 child generation 并标记已处理

```c++
  NodeToProcess->childGeneration(CurrentGeneration);
  NodeToProcess->process();
```

- `childGeneration(CurrentGeneration)`：  
  - 把「当前块结束后的最终 generation」存到这个 StackNode 里，后续创建子节点时会用它作为初始 generation。  
- `process()`：标记这个节点的「自己这一块」已经处理完了，接下来应该去处理 child 分支。 [zhuanlan.zhihu](https://zhuanlan.zhihu.com/p/1996741886271066865)

***

## 处理子节点 + 回溯

处理子节点的分支：

```c++
} else if (NodeToProcess->childIter() != NodeToProcess->end()) {
  // Push the next child onto the stack.
  DomTreeNode *Child = NodeToProcess->nextChild();
  if (!L->contains(Child->getBlock()))
    continue;
  NodesToProcess.emplace_back(
      new StackNode(AvailableLoads, NodeToProcess->childGeneration(), Child,
                    Child->begin(), Child->end()));
}
```

- `nextChild()` 拿一个支配树上的子节点。  
- 如果这个 child 的 basic block 不在当前 Loop `L` 内，就跳过。  
- 否则为 child 创建一个新的 `StackNode` 并压栈：  
  - 传入 `AvailableLoads`，让它为 child 开一个新的 scope，继承上一层的数据。  
  - 初始 generation 用 `NodeToProcess->childGeneration()`，也就是父节点处理完后记录的 generation。  
  - 再传入 child 的 `begin()/end()` 用来遍历指令。  

回溯分支就是简单地 `pop_back()`：  
- `StackNode` 析构时自动 pop 掉其 scope，恢复到父作用域的 `AvailableLoads` 状态。  

整个 DFS 就这样靠 `NodesToProcess` 栈维护，从 header 一路走完 Loop 里所有被 header 支配的块。 [zhuanlan.zhihu](https://zhuanlan.zhihu.com/p/1996741886271066865)

***

## 总结成一句话

`loadCSE` 做的是：在给定循环里，按支配树顺序遍历所有块，用 SCEV 把指针表达式归一化成等价类，再结合「版本号 + AA + MemorySSA」检测哪些 load 的值仍然可用，从而做一个「对 load 的、Loop 内的、支配树顺序的 CSE」。它用 `ScopedHashTable + StackNode` 维护作用域和 DFS 状态，用 `CurrentGeneration` 抽象地建模「这之后可能被写内存，所以旧的 load 不一定安全」。 [github](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Analysis/Loads.cpp)

如果你愿意，我可以下一步把 `getMatchingValue` 的大致逻辑（利用 generation、AA、MSSA）也拆一遍，并画一个简单的 IR 示例：给一个两层 if/loop 的例子，说明哪里会 CSE 掉 load，哪里不会。  
