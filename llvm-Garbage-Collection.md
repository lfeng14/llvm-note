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

#### 附件
- https://llvm.org/docs/GarbageCollection.html#plugin
- https://llvm.org/docs/Statepoints.html
- https://llvm.org/docs/StackMaps.html#stackmap-section
