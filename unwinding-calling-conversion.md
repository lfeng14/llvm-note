- pr: https://github.com/jeandle/jeandle-llvm/pull/15/changes#diff-3f05c00c1ba4ebe15ca72c030f39bc8257b80c0f0741068566249fd0c5d9f48e
- callee save vs caller save
  - 假如caller通过for循环调用callee，那么首位进行一次保存寄存器开销小；
- LR FP用于栈展开，哪时候需要栈展开 ？
  - 异常发生在调用链的末端，这使需要栈展开，逐层寻找每层栈帧是否有异常捕获逻辑
- 调用约定，参数顺序与寄存器编号,返回值是用哪个x0
  ```
  //  |--------------------------------------------------------------------|
  //  | X0       X1       X2      X3      X4      X5      X6      X7       |
  //  |--------------------------------------------------------------------|
  //  | c_rarg0  c_rarg1  c_rarg2 c_rarg3 c_rarg4 c_rarg5 c_rarg6 c_rarg7  |
  //  |--------------------------------------------------------------------|
  //  | j_rarg7  j_rarg0  j_rarg1 j_rarg2 j_rarg3 j_rarg4 j_rarg5 j_rarg6  |
  //  |--------------------------------------------------------------------|
  //
  
  let Entry = 1 in
  def CC_AArch64_Hotspot : CallingConv<[
    CCIfType<[i32], CCAssignToReg<[W1, W2, W3, W4, W5, W6, W7, W0]>>,
    CCIfType<[i64], CCAssignToReg<[X1, X2, X3, X4, X5, X6, X7, X0]>>,
  
    // Java floating args are passed as per C
    CCIfType<[f32], CCAssignToReg<[S0, S1, S2, S3, S4, S5, S6, S7]>>,
    CCIfType<[f64], CCAssignToReg<[D0, D1, D2, D3, D4, D5, D6, D7]>>,
  
    // Delegate to AAPCS for now
    CCDelegateTo<CC_AArch64_AAPCS>
  ]>;
  ```

  根据你提供的 TableGen 代码和搜索结果，HotSpot 在 AArch64 架构上的 Java 调用约定（Java Calling Convention）使用的寄存器分配规则如下：

### 一、整数参数寄存器（"旋转"分配）

这是 HotSpot Java 调用约定与标准 AAPCS（C 调用约定）最大的不同点：

| 参数位置 | Java 调用约定 (HotSpot) | C 调用约定 (AAPCS) |
|---------|------------------------|-------------------|
| 第 1 个参数 (j_rarg0) | **X1 / W1** | X0 / W0 |
| 第 2 个参数 (j_rarg1) | **X2 / W2** | X1 / W1 |
| 第 3 个参数 (j_rarg2) | **X3 / W3** | X2 / W2 |
| 第 4 个参数 (j_rarg3) | **X4 / W4** | X3 / W3 |
| 第 5 个参数 (j_rarg4) | **X5 / W5** | X4 / W4 |
| 第 6 个参数 (j_rarg5) | **X6 / W6** | X5 / W5 |
| 第 7 个参数 (j_rarg6) | **X7 / W7** | X6 / W6 |
| 第 8 个参数 (j_rarg7) | **X0 / W0** | X7 / W7 |

这种设计被称为"旋转"（rotation），将 `X0` 保留给第 8 个参数或特殊用途。如注释中表格所示：

```
| X0       X1       X2      X3      X4      X5      X6      X7       |
| c_rarg0  c_rarg1  c_rarg2 c_rarg3 c_rarg4 c_rarg5 c_rarg6 c_rarg7  |
| j_rarg7  j_rarg0  j_rarg1 j_rarg2 j_rarg3 j_rarg4 j_rarg5 j_rarg6  |
```

### 二、浮点参数寄存器

浮点参数的传递与 AAPCS 完全一致：

- **float 类型 (f32)**：依次使用 **S0, S1, S2, S3, S4, S5, S6, S7**
- **double 类型 (f64)**：依次使用 **D0, D1, D2, D3, D4, D5, D6, D7**

### 三、返回值寄存器

| 返回值类型 | Java 调用约定 |
|----------|--------------|
| int (32位) | **W0** |
| long (64位) | **X0**（低位在 X0，高位在 X1） |
| float | **S0** |
| double | **D0** |
| 对象引用 | **X0** |

### 四、特殊用途寄存器

HotSpot 还定义了一些专用寄存器：

| 寄存器 | 用途 |
|-------|------|
| **rthread (X28)** | 当前线程指针（thread-local storage） |
| **rfp (X29)** | 帧指针（frame pointer） |
| **lr (X30)** | 链接寄存器（返回地址） |
| **sp (X31)** | 栈指针 |
| **rscratch1 (X8)** | 临时暂存寄存器（scratch，分配器不可见） |
| **rscratch2 (X9)** | 临时暂存寄存器（scratch，分配器不可见） |
| **rbcp (X22)** | 字节码指针（bytecode pointer，解释器用） |
| **rlocals (X24)** | 局部变量基址（解释器用） |
| **rmonitors (X25)** | 监视器指针（解释器用） |
| **rcpool (X26)** | 常量池指针（解释器用） |

### 五、为什么这样设计？

从搜索结果和源码注释中可以推断出几个设计考量：

1. **`X0` 的特殊地位**：在 Java 中，`X0` 通常用于传递方法接收者（`this` 指针）或作为返回值寄存器。将其移到参数列表末尾，便于 JIT 生成的代码与解释器、JNI 调用协作。

2. **浮点寄存器保持一致**：浮点寄存器在解释器和 JNI 中的使用方式相对简单，因此直接沿用 AAPCS 规则，减少复杂性。

3. **委托 AAPCS 处理剩余情况**：当参数数量超过寄存器容量或涉及复杂类型时，通过 `CCDelegateTo<CC_AArch64_AAPCS>` 回退到标准 AAPCS 栈传递规则。

### 面试话术建议

如果面试官问到这个知识点，你可以这样回答：

> "HotSpot 在 AArch64 上的 Java 调用约定采用了**整数参数寄存器旋转**的设计：前七个整数参数使用 X1-X7，第八个才使用 X0。浮点参数则完全遵循 AAPCS。这种设计主要是为了让 X0 能够同时兼顾方法接收者（this）传递和返回值传递的角色，同时与解释器栈帧布局保持兼容。具体的寄存器定义可以在 OpenJDK 源码的 `src/hotspot/cpu/aarch64` 目录下的 `assembler_aarch64.hpp` 和 `aarch64.ad` 文件中找到。"
