#### Description

- [linker](https://github.com/llvm/llvm-project/pull/192052)
- For SVE vectors：     
     ```
     sext(icmp slt a, b) --> asr (shsub a, b), bitwidth-1
     sext(icmp ult a, b) --> asr (uhsub a, b), bitwidth-1
     zext(icmp slt a, b) --> lsr (shsub a, b), bitwidth-1
     zext(icmp ult a, b) --> lsr (uhsub a, b), bitwidth-1
     sext(icmp sgt a, b) --> asr (shsub b, a), bitwidth-1
     sext(icmp ugt a, b) --> asr (uhsub b, a), bitwidth-1
     zext(icmp sgt a, b) --> lsr (shsub b, a), bitwidth-1
     zext(icmp ugt a, b) --> lsr (uhsub b, a), bitwidth-1
     ```
- 数学等价替换，使用开销更小的硬件指令
#### Analysis
- 做了什么优化
  我**完全不用硬件知识**，只用**小学数学+逻辑**给你讲透！
  一步一步拆，你瞬间就懂，这根本不是玄学，就是**数学等价替换**！
  
  #### 第一步：先看懂左边 `sext(icmp slt a, b)` 是干嘛
  把它拆成两个动作：
  1. **icmp slt a, b**
     有符号比较：`a 小于 b 吗？`
     ✅ 是 → 输出 **1**
     ❌ 否 → 输出 **0**
  
  2. **sext（符号扩展）**
     把 1 变成 **全1的数字**（比如32位就是 -1）
     把 0 变成 **全0的数字**
  
  👉 **左边最终结果**：
  a < b → **全1**
  a ≥ b → **全0**
  
  #### 第二步：再看懂右边 `asr (shsub a, b), bitwidth-1`
  我直接翻译成**纯数学计算**，**完全不讲硬件指令**：
  1. **shsub a, b = (a - b) ÷ 2** （有符号减半减法）
  2. **asr(结果, 位宽-1)** = 把结果**带符号右移 31位**（32位数字）
     👉 这个移位只有两个结果：
     - 如果数字是**负数** → 移位后 = **全1**
     - 如果数字是**正数/0** → 移位后 = **全0**
  
  #### 第三步：核心！为什么两边**完全相等**？
  我用**两个真实数字例子**证明，一看就懂！
  
  ##### 例子1：a=3，b=5 （a < b，左边输出全1）
  左边：
  icmp slt 3,5 → 1 → sext → **全1**
  
  右边：
  shsub(3,5) = (3-5)/2 = **-1**（负数）
  asr(-1, 31) → **全1**
  
  ✅ **两边结果一模一样！**
  
  ##### 例子2：a=5，b=3 （a > b，左边输出全0）
  左边：
  icmp slt5,3 → 0 → sext → **全0**
  
  右边：
  shsub(5,3) = (5-3)/2 = **1**（正数）
  asr(1,31) → **全0**
  
  ✅ **两边结果还是一模一样！**
  
  #### 终极大白话总结（你记这个就够了）
  `sext(icmp slt a,b)`
  和
  `asr(shsub a,b, 位宽-1)`
  
  **数学上完全相等，输出一模一样！**
  
  ##### 那为什么这叫优化？
  因为：
  - 左边：需要 **比较指令 + 扩展指令**（2条，甚至更多）
  - 右边：只需要 **减法指令 + 移位指令**（2条，**更快、更简单、不挑芯片**）
  
  ##### 安抚你一下：
  这真的不难！
  你只是**不知道背后是数学等价**，以为是硬件黑魔法。
  一旦知道「两边算出来的数完全一样」，这个优化就彻底通透了！

- 为什么有优化：
  - 在AArch64架构中，shsub/uhsub（减半减法）和asr/lsr（移位）都是**单周期、低延迟**的指令，而原来的icmp（比较）+ sext/zext（扩展）组合，虽然也是两条指令，但比较指令的结果需要通过标志位或中间向量传递，可能引入额外的依赖延迟。更重要的是，旧方式可能依赖SCLAMP等特殊指令，这些指令的延迟通常比减法+移位更高。所以整体来看，减法+移位的指令组合在**延迟和资源占用上都更优**，尤其是在SVE向量场景下，硬件对这类简单算术指令的并行处理效率更高。
- 注释：
  ```
  // 1. 第一步：判断！是不是【无符号数小于比较】？
  // SETULT = Set Unsigned Less Than（无符号比较 a < b）
  if (CC == ISD::SETULT) {
          // 注释：就是我们聊的数学公式！一字不差
          // sext(icmp ult a, b) --> asr (uhsub a, b), bitwidth-1
          
          // 2. 第二步：生成 SVE 硬件专属的 无符号减半减法 指令
          SDValue Sub = DAG.getNode(
              ISD::INTRINSIC_WO_CHAIN, DL, N->getValueType(0),
              // 核心：调用 ARM SVE 芯片自带的硬件指令：uhsub（无符号减半减法）
              DAG.getConstant(Intrinsic::aarch64_sve_uhsub_u, DL, MVT::i64),
              AllTruePred,  // SVE向量：对所有元素都执行这个计算
              A, B);        // 就是我们的两个数 a 和 b
          
          // 3. 第三步：生成 算术右移 指令（就是之前的asr）
          // 右移 位宽-1 位，直接得到 全0 或 全1
          return DAG.getNode(ISD::SRA, DL, N->getValueType(0), Sub, ShiftAmt);
  }
  ```
