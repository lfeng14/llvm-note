- Accurate garbage collection requires the ability to identify all pointers in the program at run-time (which requires that the source-language be type safe in most cases). Identifying pointers at run-time requires compiler support to locate all places that hold live pointer variables at run-time, including the processor stack and registers.

- LLVM provides support for generating stack maps at call sites, polling for a safepoint, and emitting load and store barriers. 
- The gc function attribute is used to specify the desired GC strategy to the compiler. Its programmatic equivalent is the setGC method of Function。
  Setting gc "name" on a function triggers a search for a matching subclass of GCStrategy. Some collector strategies are built in. You can add others using either the loadable plugin mechanism, or by patching your copy of LLVM. It is the selected GC strategy which defines the exact nature of the code generated to support GC. If none is found, the compiler will raise an error。
  ```
  define <returntype> @name(...) gc "name" { ... }
  ```
- Custom GC Strategies and built-in gc strategies:
  - 内建gc
    <img width="1000" height="250" alt="image" src="https://github.com/user-attachments/assets/8665a6f3-96aa-4840-904e-231e09ba42d3" />
- 概念解释：**Stack Map**（栈映射）正是垃圾回收器用来准确识别调用栈中各栈帧内**哪些位置存放了指向堆对象的指针**（即根引用）的元数据。
  - 在程序运行到**安全点**（safepoint）时，GC 需要遍历所有线程的栈，从每个栈帧中找出所有活跃的引用。
  - 编译器会为每个安全点生成对应的 stack map，记录当前栈帧中哪些寄存器或栈上偏移量存放着引用，以及可能的附加信息（如派生指针与对象基址的关系）。
  - 这样 GC 就能精确地遍历根集合，而不必保守地扫描整个栈，从而实现准确式垃圾回收。
  - 所以你的理解很到位：调用栈有多层，每一层都可能包含引用，stack map 就是告诉 GC 这些引用具体藏在哪里的“地图”。

- llvm.gcroot函数：作用：告知 LLVM 某栈变量引用堆对象，需被垃圾回收（GC）跟踪，具体代码影响由函数 GC 策略决定，且调用必须位于函数首个基本块内
  ```
    Entry:
       ;; In the entry block for the function, allocate the
       ;; stack space for X, which is an LLVM pointer.
       %X = alloca %Object*
    
       ;; Tell LLVM that the stack space is a stack root.
       ;; Java has type-tags on objects, so we pass null as metadata.
       %tmp = bitcast %Object** %X to i8**
       call void @llvm.gcroot(i8** %tmp, i8* null)
       ...
    
       ;; "CodeBlock" is the block corresponding to the start
       ;;  of the scope above.
    CodeBlock:
       ;; Java null-initializes pointers.
       store %Object* null, %Object** %X
    
       ...
    
       ;; As the pointer goes out of scope, store a null value into
       ;; it, to indicate that the value is no longer live.
       store %Object* null, %Object** %X
       ...
  ```
- Reading and writing references in the heap 要点总结:
  - 1. **读写屏障定义**：垃圾回收器需感知程序对堆对象字段指针的读写，在对应位置插入的代码分别为**读屏障**和**写屏障**。
  - 2. **性能影响**：屏障代码量小、非计算关键路径，对整体性能影响可接受。
  - 3. **指针要求**：屏障通常需**对象指针**和**派生指针**（对象内字段指针），LLVM 内置函数会分别传入这两类指针。
  - 4. **LLVM 约束**：LLVM 不强制两类指针的关联，特定回收策略可能有要求，违背该关系的回收器极少。
  - 5. **使用规则**：若目标垃圾回收器无需对应屏障，内置函数可选用；使用时应替换为对应加载/存储指令。
  - 6. **当前缺陷**：屏障内置函数未包含底层操作的大小与对齐信息，默认按指针大小和目标机器默认对齐处理。

- LLVM 不强制对象指针与派生指针的关联，特定垃圾回收（GC）策略可能会约束，但极少有 GC 会违背该关系。
  - 派生指针是指向堆对象内部某个字段或数组元素的指针，其值可通过对象基址加上偏移量得到
- 若目标 GC 无需对应屏障，相关内置函数可选用；GC 策略会将其替换为对应加载/存储指令。
- 当前设计不足：屏障内置函数未包含底层操作的大小与对齐信息，默认按指针大小和目标机器默认对齐处理。

在 LLVM 的垃圾回收框架中，LLVM 本身**只负责提供基础设施**，而实际的垃圾清理工作由**运行时系统（Runtime）** 完成。具体分工如下：

- 垃圾清理**LLVM 的作用**：
  - 在编译过程中插入**安全点（safepoint）** 指令（如 `gc.statepoint`）。
  - 为每个安全点生成**栈映射表（stack map）**，记录当前栈帧和寄存器中哪些位置存放了指向堆对象的指针（即根引用）。
  - 可选地插入读/写屏障（如果回收算法需要）。
- **运行时的作用**：
  - 当程序执行到安全点时，暂停线程（例如通过信号或主动轮询），将控制权交给垃圾回收器。
  - 利用 LLVM 提供的栈映射信息，**精确地枚举所有根引用**（全局变量、栈上的局部变量、寄存器中的指针等）。
  - 根据所选用的回收算法（标记-清扫、复制、分代、并发等）执行**实际的内存标记、清理、压缩**等操作。
  - 如果对象被移动（如复制式收集），运行时还需更新所有根引用（包括栈和寄存器中的指针）指向新地址。
- 简而言之，LLVM 是“工具提供者”，负责生成让垃圾回收器能够准确识别根所需的元数据；而真正的“清洁工”是运行时中的垃圾回收器实现。二者通过安全点和栈映射协作，实现精确式自动内存管理。

#### 附件
- https://llvm.org/docs/GarbageCollection.html
- https://llvm.org/docs/Statepoints.html
- https://llvm.org/docs/StackMaps.html#stackmap-section
- https://llvm.org/docs/LangRef.html有LLVM IR中和GC相关部分的介绍
- statepoint作者preames对statepoint的介绍：https://github.com/preames/public-notes/blob/master/llvm-gc-retrospective-2019)
- https://github.com/jeandle/document/blob/main/GC/Safepoint.md
