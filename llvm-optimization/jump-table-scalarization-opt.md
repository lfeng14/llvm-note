- [ llvm issue 链接](https://github.com/llvm/llvm-project/issues/177903)
- 测试示例：
  ```
  cat test-switch.c
  #include <stdio.h>
  
  void test_large_range(int n) {
      
      switch (n) {
          case 0:  printf("Zero\n"); break;
          case 10: printf("Ten\n"); break;
          case 20: printf("Twenty\n"); break;
          case 30: printf("Thirty\n"); break;
          case 40: printf("Forty\n"); break;
          case 50: printf("Fifty\n"); break;
          case 60: printf("Sixty\n"); break;
          case 70: printf("Seventy\n"); break;
          case 80: printf("Eighty\n"); break;
          case 90: printf("Ninety\n"); break;
          case 100: printf("One Hundred\n"); break;
          default: printf("Other: %d\n", n); break;
      }
   }
  ```
- 构建命令
  ```
  # generates IR
  clang -S -emit-llvm test-switch.c -o test-switch.ll

  # pic mode 
  llc --relocation-model=pic test-switch.ll  -o pic-test-switch.s
  llc --relocation-model=pic test-switch.ll --filetype=obj -o pic-test-switch.o  

  # no pic mode
  llc  test-switch.ll  -o nopic-test-switch.s
  llc  test-switch.ll --filetype=obj -o nopic-test-switch.o
  ```
- x86-64汇编介绍
   ```
           # %bb.0:                                 # 基本块 0（入口）
               pushq   %rbp                         # 保存旧的基址指针
               .cfi_def_cfa_offset 16               # CFI: 调用帧信息，栈偏移16
               .cfi_offset %rbp, -16                # CFI: rbp保存在-16位置
               movq    %rsp, %rbp                   # 建立新栈帧
               .cfi_def_cfa_register %rbp           # CFI: 使用rbp作为帧寄存器
               subq    $16, %rsp                     # 分配16字节局部变量空间
               movl    %edi, -4(%rbp)                # 将第一个参数（int n）存入栈
               movl    -4(%rbp), %eax                # 读回eax（用于后续计算）
               movq    %rax, %rcx                     # 读取n，rcx = n（64位扩展）
               subq    $100, %rcx                     # rcx = n - 100 编译器的scalarization optimization
               ja      .LBB0_13                       # 如果 n-100 > 0（ja 是根据无符号比较结果进行跳转，它检查的标志位是 CF=0（无借位）且 ZF=0（结果不为零）），跳转到default .LBB0_13
         # %bb.1:
             leaq    .LJTI0_0(%rip), %rcx            # rcx = 跳转表基地址（RIP相对寻址）
             movslq  (%rcx,%rax,4), %rax             # rax = 跳转表[rax]（符号扩展32->64位）
             addq    %rcx, %rax                      # rax = 跳转表基址 + 偏移量（n） = 目标地址
             jmpq    *%rax                           # 间接跳转：跳转到计算出的地址
       .LBB0_2:                                      # case handler，跳转表 跳转到这里
               leaq    .L.str(%rip), %rdi 
               movb    $0, %al
               callq   printf@PLT
               jmp     .LBB0_14
       .LBB0_3:                                       # case handler，跳转表 跳转到这里
               leaq    .L.str.1(%rip), %rdi
               movb    $0, %al
               callq   printf@PLT
               jmp     .LBB0_14
       .LBB0_4:                                       # case handler，跳转表 跳转到这里
               leaq    .L.str.2(%rip), %rdi
               movb    $0, %al
               callq   printf@PLT
               jmp     .LBB0_14
       .LBB0_5:                                       # case handler，跳转表 跳转到这里
               leaq    .L.str.3(%rip), %rdi
               movb    $0, %al
               callq   printf@PLT
               jmp     .LBB0_14
   ```
- 该例子同样出现addend超过.text大小
   ```
    # pic
    Relocation section '.rela.rodata' at offset 0x7a8 contains 101 entries:
      Offset          Info           Type           Sym. Value    Sym. Name + Addend
    000000000000  000200000002 R_X86_64_PC32     0000000000000000 .text + 2b
    000000000004  000200000002 R_X86_64_PC32     0000000000000000 .text + eb
    000000000008  000200000002 R_X86_64_PC32     0000000000000000 .text + ef
    00000000000c  000200000002 R_X86_64_PC32     0000000000000000 .text + f3
    ...
    000000000178  000200000002 R_X86_64_PC32     0000000000000000 .text + 25f
    00000000017c  000200000002 R_X86_64_PC32     0000000000000000 .text + 263
    000000000180  000200000002 R_X86_64_PC32     0000000000000000 .text + 267
    000000000184  000200000002 R_X86_64_PC32     0000000000000000 .text + 26b # 注意差距4
    000000000188  000200000002 R_X86_64_PC32     0000000000000000 .text + 26f  # 注意差距4
    00000000018c  000200000002 R_X86_64_PC32     0000000000000000 .text + 273  # 注意差距4
    000000000190  000200000002 R_X86_64_PC32     0000000000000000 .text + 267
    
    # no pic
    Relocation section '.rela.rodata' at offset 0x858 contains 101 entries:
      Offset          Info           Type           Sym. Value    Sym. Name + Addend
    000000000000  000200000001 R_X86_64_64       0000000000000000 .text + 25
    000000000008  000200000001 R_X86_64_64       0000000000000000 .text + 105
    ...
    000000000308  000200000001 R_X86_64_64       0000000000000000 .text + 105  # 很多addend值相同
    000000000310  000200000001 R_X86_64_64       0000000000000000 .text + 105
    000000000318  000200000001 R_X86_64_64       0000000000000000 .text + 105
    000000000320  000200000001 R_X86_64_64       0000000000000000 .text + f2
   ```
- 原因分析：
  - 通过观察nopic-test-switch.s pic-test-switch.s，这两个文件都会生成跳转表，且跳转表长度一样
  - 跳转表差异点：
     - pic模式下：
        ```
        .LJTI0_0:
            .long   .LBB0_2-.LJTI0_0     # 可以看到这里填写目标符号采用相对寻址，基地址为跳转表头的偏移
            .long   .LBB0_13-.LJTI0_0   # 发现很多占位的，因为0 ~ 100，其中1需要走到default .LBB0_13执行
            .long   .LBB0_13-.LJTI0_0
            .long   .LBB0_13-.LJTI0_0
            .long   .LBB0_13-.LJTI0_0
        ```
      - 非pic模式下
         ```
         .LJTI0_0:
              .quad   .LBB0_2   #  可以看到这里填写目标符号采用绝对寻址，基地址为.text
              .quad   .LBB0_13
              .quad   .LBB0_13
              .quad   .LBB0_13
              .quad   .LBB0_13
              .quad   .LBB0_13
              .quad   .LBB0_13
          ```
  - PIC模式时，如何计算偏移量，也就是计算上面的.LBB0_2-.LJTI0_0，注意这里的相对寻址是相对跳转表头
  - PC32重定位采用S+A-P公式
  - S表示目标跳转地址，也就是.text内的 .LBB0_2、 .LBB0_3、 .LBB0_4
  - A表示Addend，是个修正值
  - P表示需要重定位的地址，也就是只读段 跳转表内的.LJTI0_0、.LJTI0_0+4、.LJTI0_0+8
  - 比如n = 0时，需要重定位到.LBB0_2处，那么偏移量通过 .LBB0_2 + Addend - (.LJTI0_0 + 0)
  - 比如n = 1时，需要重定位到.LBB0_3处，那么偏移量通过 .LBB0_13 + Addend - (.LJTI0_0 + 4)
  - 如果走到同样的default分支，那么偏移量应该是一样的.LBB0_13-.LJTI0_0，看到上面.rela.rodata段也有+4，用于抵消跳转表元素偏移的+4，所以只要跳转表长度（还需要乘4）够长就有可能超过.text长度。
  - 如果进程地址空间里，.LBB0_2 在前，.LJTI0_0跳转表地址在后，所以.LBB0_2 - .LJTI0_0 应该小于0
  - 还有个问题，上面的2b偏移是啥呢 
