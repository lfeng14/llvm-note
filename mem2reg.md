```
  V ← 4
    ← V + 5
  V ← 6
    ← V + 7

  ; V ← 4
  store i32 4, i32* %V.addr
  ;   ← V + 5
  %tmp1 = load i32* %V.addr
  %tmp2 = add i32 %tmp1, 5
  store i32 %tmp2, i32* %V.addr
  ; V ← 6
  store i32 6, i32* %V.addr
  ;   ← V + 7
  %tmp3 = load i32* %V.addr
  %tmp4 = add i32 %tmp3, i32 7
  store i32 %tmp4, i32* %V.addr
```
- 局部变量都是在函数的入口基本块通过 alloca 来“声明”的，并且后续对局部变量的赋值都是通过 store 指令，获取局部变量的值都是通过 load 指令，正是前面所说的 alloca/load/store 形式的 LLVM IR。为什么这么保守呢？
  - 因为可能存在别名，别名对该内存进行store，所以需要重新load，而不是从寄存器获取值；重新store也是一样，考虑存在别名的情况。
- 如果知道 %V.addr 是 alloca（栈上局部变量），优化就简单很多：
能看到所有使用位置
不会和别的指针别名（alias）
知道对齐、不会访存出错
没有类似 free 的操作
函数结束时栈变量生命周期自动结束，没用的 store 可以直接删
所有访问都删掉后，alloca 本身也能删

- **在不知道 %V.addr 到底是什么指针的情况下，编译器必须保守假设：它指向的内存可能在函数外部、其他地方被使用。**
  - %V.addr 只是一个指针，编译器**没看到它是怎么来的**（不知道是不是全局变量、是不是被别的函数传入、是不是和别的指针重叠）。
  - 既然来源不明，**就不能随便删 store**、不能随便优化，因为万一外面还在用这块内存，优化就会出错。
  - 只有当确定它是 `alloca` 分配的栈变量时，编译器才能放心大胆地做各种优化（比如删无用 store、直接用寄存器代替内存、插 phi 指令等）。
  - 一句话总结：信息不足 → 保守；信息足够（知道是 alloca） → 激进优化。


- step 7: RenamePass
  ```
  While (worklist != NULL)
      Remove block B from worklist and mark B as visited.
      For each instruction in B:
          If instruction is a load instruction from location L (where L is a promotable candidate) to value V, delete load instruction, replace all uses of V with most recent value of L i.e, IncomingVals[L].
          If instruction is a store instruction to location L (where L is a promotable candidate) with value V, delete store instruction, set most recent name of L i.e, IncomingVals[L] = V.
          For each PHI-node corresponding to a alloca — L , in each successor of B, fill the corresponding PHI-node argument with most recent name for that location i.e, IncomingVals[L].
      Add each unvisited successor of B to worklist.
  ```
    - **While (worklist != NULL)**
       用**工作列表**遍历控制流图 CFG，从函数入口块开始。
    
    - **Remove block B from worklist，标记已访问**
       每次取一个基本块 B 处理，避免重复遍历。
    
    - **遍历 B 里每条指令**
       只关心两种：**load** 和 **store**。
    
    - **遇到 load（读内存）**
       - 删掉这条 load
       - 把用到这个 load 值的地方，直接换成 **IncomingVals[L]**
         （L 是被提升的 alloca 变量，IncomingVals 存它当前最新的 SSA 值）
    
    - **遇到 store（写内存）**
       - 删掉这条 store
       - 把写入的值 V，记到 **IncomingVals[L] = V**
         表示：这个变量现在最新值就是 V。
    
    - **给后继块的 PHI 节点填值**
       走到块 B 末尾时，把当前 L 的最新值，填进**后继块里对应 L 的 phi 节点**。
       这一步是**构建正确 SSA**的关键。
    
    - **把没访问过的后继块加入 worklist**
       继续往下遍历 CFG，直到所有可达块都处理完。

### 最终效果
- 所有 **alloca / load / store** 全部删掉
- 变量直接用 **SSA 值 + phi** 表示
- 代码从「内存风格」变成**纯寄存器 SSA 形式**

#### further reading
- https://dl.acm.org/doi/abs/10.1145/115372.115320
- https://llvm-study-notes.readthedocs.io/en/latest/ssa/Mem2Reg.html
