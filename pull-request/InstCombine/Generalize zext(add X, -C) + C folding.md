##### 描述
[linker](https://github.com/llvm/llvm-project/pull/191723/files) 
zext(add X, -C) + C folding --> zext(X)

#### Analysis
- 首先注意这里C是常量，编译器能够解析出来，非常量则不做优化；
- 符号常量、常量都会处理吗 ？
  - 经过**常量传播和常量折叠后**，在LLVM IR层面，所有常量都是具体的数值，无论你在C++中是用变量名表示还是直接字面量。但用户可能混淆了“符号常量”和“数值常量”的区别，或者是在问代码是否能够处理通过变量引用传递的常量（即由前端生成的IR中的常量表达式）。
  - 在 LLVM IR 的语境下，不存在“符号常量”与“数值常量”的区分。无论是你在 C++ 中写死的字面量 10，还是定义了一个 const int z = 10 后使用变量名 z，前端（Clang）都会在生成 IR 时将其全部折叠求值为具体的 ConstantInt。
- 逻辑确实主要针对 zext (X + (-C)) + C 的情况（外层常量为正数）。对于你提出的对称情况 zext (X + 10) + -10，它在当前这段代码中不会被触发。
- 这里不固定x和C的数据类型，可能是i8 i32，llvm如何处理 ?
  - 针对不同位宽应该如何处理(在 LLVM IR 中，不同位宽的常量不能直接用 == 比较)，所以llvm这边使用函数getBitWidth来

  ```
  const APInt *InnerC;
  if (match(Op0, m_ZExt(m_Add(m_Value(X), m_APIntAllowPoison(InnerC))))) {
    unsigned NarrowBW = InnerC->getBitWidth();
    if (C->getActiveBits() <= NarrowBW) {
      APInt NarrowC = C->trunc(NarrowBW);  // 处理不同位宽，先获取有效位宽，然后trunc成相同位宽，最后再比较。例如i32 0x80 00 00 00 getActiveBits返回32;
      const SimplifyQuery Q = SQ.getWithInstruction(&Add);
      if (*InnerC == -NarrowC &&
          (NarrowC.isOne() ? llvm::isKnownNonZero(X, Q)
                           : computeKnownBits(X, &Add).getMinValue().uge(NarrowC)))
        return new ZExtInst(X, Ty);
    }
  }

  %inner = add i8 %or, -4     
  %z = zext i8 %inner to i32  # inner:-4 i8; 
  %r = add i32 %z, 4          # outer:4 i32; 
  ```
- 所以下回可以这么问AI：pr实现了什么功能，需要考虑哪些**边界情况**，
