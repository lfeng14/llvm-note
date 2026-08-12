> 基于本仓库 LLVM 22.0.0git，默认示例目标为 AArch64。
>
> 目标：不仅能背出流水线，还能解释每一层的输入、输出、合法性、成本模型、核心数据结构，
> 并能从 LLVM IR 一直追踪到 MIR、物理寄存器、MCInst 和最终机器码。

## 目录

1. 后端全景图
2. TargetMachine 与目标描述
3. Legalization：把 IR 操作变成目标能处理的形态
4. SelectionDAG 指令选择
5. GlobalISel 指令选择
6. Machine IR、MachineFunction 与 MachineInstr
7. 虚拟寄存器、物理寄存器与 Register Class
8. 活跃性、LiveInterval 与 SlotIndex
9. 寄存器分配、Spill、Split 与 Coalescing
10. 指令调度与处理器资源模型
11. Calling Convention 与 Call Lowering
12. 栈帧、FrameIndex 与 Prologue/Epilogue
13. Machine SSA、PHI Elimination 与 Two-Address
14. TableGen 高频知识
15. MC 层、汇编器、Fixup 与 Relocation
16. 分支、常量池、Jump Table 与代码布局
17. JIT 代码发射与运行时问题
18. 新增后端指令/优化的实现框架
19. 高频现场题与排障题
20. 面试前速背表

---

# 1. 后端全景图

## 1.1 从 LLVM IR 到机器码

```text
                         LLVM IR
                            │
             ┌──────────────┴──────────────┐
             │                             │
             ▼                             ▼
      SelectionDAG                    GlobalISel
   SelectionDAGBuilder               IRTranslator
         Legalize                     Legalizer
        DAGCombine                   RegBankSelect
     InstructionSelect             InstructionSelect
             └──────────────┬──────────────┘
                            ▼
               MachineInstr / Machine IR
                  （通常仍有虚拟寄存器）
                            │
            Machine SSA 优化、伪指令展开、调度
                            │
                 PHI Elimination / Two-Address
                            │
               LiveIntervals / Register Allocation
                            │
                   Spill / Reload / Rewrite
                            │
             Prologue/Epilogue、FrameIndex 消除
                            │
             Post-RA Scheduling / Branch Relaxation
                            │
                            ▼
                          MCInst
                            │
        ┌───────────────────┴───────────────────┐
        ▼                                       ▼
   Assembly Text                         Object / JIT Bytes
                                      fixup → relocation/link
```

这是一张概念图，不是所有 target、opt level 都严格按完全相同的 Pass 顺序执行。当前版本的
实际流水线应从 `TargetPassConfig`、目标 `TargetMachine` 和调试输出中确认。

## 1.2 一分钟标准回答

> LLVM IR 优化结束后，后端先根据目标合法类型和操作进行 legalization，再通过
> SelectionDAG 或 GlobalISel 完成指令选择，生成带虚拟寄存器的 Machine IR。机器级
> Pass 做 peephole、调度和 SSA 变换；随后活跃区间分析与寄存器分配把虚拟寄存器映射到
> 物理寄存器，必要时 spill、split 和 reload。FrameLowering 完成栈帧与序言/尾声，最后
> 降到 MCInst，经 code emitter、assembler backend 和 object writer 产生机器码、fixup
> 与 relocation。

## 1.3 后端面试最常考的边界

| 层次 | 主要对象 | 尚未决定的内容 |
|---|---|---|
| LLVM IR | `Function`、`Instruction` | 具体目标指令、物理寄存器、编码 |
| SelectionDAG/GISel | `SDNode` 或 generic MI | 最终寄存器、部分目标细节 |
| MIR pre-RA | `MachineInstr`、virtual register | 物理寄存器分配、spill |
| MIR post-RA | physical register 为主 | 最终地址/分支距离、编码细节 |
| MC | `MCInst`、`MCExpr`、`MCFixup` | 链接期符号地址 |
| Object/JIT memory | bytes + relocation | 加载地址与外部符号绑定 |

## 1.4 三条容易说错的话

- “指令选择之后就是机器码”——不对，之后还有大量 Machine IR Pass。
- “寄存器分配就是给 SSA value 随便选寄存器”——不对，要处理 live range 干涉、ABI、
  register class、subregister、regmask、spill 和 coalescing。
- “MCInst 就等于 MachineInstr”——不对，MachineInstr 是编译器内部机器级表示，仍可含
  pseudo、virtual register、frame index 和丰富 flags；MCInst 更接近可编码指令。

---

# 2. TargetMachine 与目标描述

## 2.1 核心组件

```text
TargetMachine
  ├─ TargetSubtargetInfo        CPU/features、调度模型
  ├─ TargetLowering             DAG 类型/操作合法化、调用 lowering
  ├─ TargetInstrInfo            指令描述、复制、分支、spill/reload hooks
  ├─ TargetRegisterInfo         寄存器、class、callee-saved、frame index
  ├─ TargetFrameLowering        栈增长、对齐、prologue/epilogue
  ├─ TargetSelectionDAGInfo     DAG 相关目标 hooks
  └─ TargetLoweringObjectFile   section/object format lowering
```

GlobalISel 还常见：

```text
CallLowering
LegalizerInfo
RegisterBankInfo
InstructionSelector
```

## 2.2 TargetMachine 做什么

`TargetMachine` 是 codegen 的目标配置入口，持有 triple、CPU、features、code model、
relocation model、optimization level、DataLayout 等，并负责构造目标相关 pass pipeline。

典型创建框架：

```cpp
std::string Error;
const Target *T = TargetRegistry::lookupTarget(TripleName, Error);
if (!T)
  report_fatal_error(Error);

TargetOptions Options;
std::unique_ptr<TargetMachine> TM(
    T->createTargetMachine(TripleName, CPU, Features, Options,
                           Reloc::PIC_, CodeModel::Small,
                           CodeGenOptLevel::Default));

M.setDataLayout(TM->createDataLayout());
M.setTargetTriple(Triple(TripleName));
```

真实接口的 optional 参数会随版本变化，重点是 Module 的 triple/DataLayout 必须与 TM 一致。

## 2.3 Subtarget 为什么通常按 Function 获取

同一 Module 中不同 Function 可能拥有不同 `target-cpu`、`target-features` 属性。例如一个
函数启用 SIMD 扩展，另一个保持基线 ISA。因此很多代码使用：

```cpp
const TargetSubtargetInfo &STI = MF.getSubtarget();
const TargetInstrInfo *TII = STI.getInstrInfo();
const TargetRegisterInfo *TRI = STI.getRegisterInfo();
```

不能把第一次取得的 subtarget 无条件缓存后用于所有函数。

## 2.4 高频问答

**问：Triple、DataLayout、CPU、Features 分别解决什么？**

- Triple：架构、厂商、OS、environment/object format 等平台身份。
- DataLayout：类型大小、对齐、pointer/address space 等 ABI 布局。
- CPU：具体微架构/处理器型号。
- Features：ISA 扩展和开关，如 SIMD、crypto 等。

**问：为什么同是 AArch64，生成代码仍可能不同？**

答：CPU 调度模型、features、code model、relocation model、ABI、opt level 和 profile 都会
影响指令选择、调度、布局和编码。

---

# 3. Legalization：把 IR 操作变成目标能处理的形态

## 3.1 为什么需要 Legalization

LLVM IR 支持很多位宽和操作，但硬件只直接支持有限集合：

```llvm
%x = add i137 %a, %b
%y = sdiv i8 %c, %d
%z = fadd fp128 %e, %f
```

目标可能没有 i137 寄存器、i8 division 或 fp128 指令。Legalizer 必须把它们转换成目标
支持的类型/操作，或 lower 成 runtime libcall。

## 3.2 三种典型动作

| 动作 | 含义 | 示例 |
|---|---|---|
| Legal | 目标可直接处理 | AArch64 i64 add |
| Promote/Widen | 提升到更宽合法类型 | i8 运算提升到 i32 |
| Expand/Lower | 拆成多条操作或调用 | 大整数拆分、division libcall |
| Custom | 交给 target 自定义 lowering | 特殊 intrinsic/addressing |

GlobalISel 中还会见到 narrowScalar、moreElements、fewerElements、libcall 等 legalization action。

## 3.3 类型合法化与操作合法化

这两个问题不同：

```text
类型是否合法？       i137 对寄存器类型不合法
操作是否合法？       i64 类型合法，但某 target 没有原生 i64 sdiv
```

可能类型合法而操作不合法，也可能反过来需要先改变类型才能选择目标操作。

## 3.4 大整数示例

概念上把 i128 加法降为两个 i64 加法：

```text
lo = add a.lo, b.lo
carry = lo < a.lo
hi = add a.hi, b.hi
hi = add hi, carry
```

在具有 add-with-carry 指令的目标上可选择更直接的指令序列。Legalization 暴露 carry 语义，
instruction selection 再匹配目标 opcode。

## 3.5 高频追问

**问：Legalization 为什么不能简单全部 lower 成 libc 调用？**

答：会丢失性能和进一步组合机会，且 runtime 函数未必存在；优先使用目标合法原语，必要
时才 libcall。

**问：Legalization 在 SelectionDAG 与 GlobalISel 中表示相同吗？**

答：目标相同但框架不同。DAG 通过 `TargetLowering` 的 action 和 DAG legalizer；GISel
通过 `LegalizerInfo` 和 generic MachineInstr 变换。

**问：Legalizer 之后为什么还需要 combine？**

答：拆分/扩展会产生冗余 pattern，combine 可折叠并为 instruction selection 建立更优形态。

---

# 4. SelectionDAG 指令选择

## 4.1 核心表示

SelectionDAG 是通常按基本块构造的有向无环图：

- 节点是操作（`SDNode`）。
- 边表示 value dependency。
- `SDValue` 是“节点 + result number”，因为节点可有多个结果。
- `MVT/EVT` 表示机器值类型。
- chain 表示内存/副作用顺序。
- glue 表示必须相邻或紧密绑定的调度关系。

## 4.2 为什么 load 节点需要 chain

纯 SSA data edge 只表达值依赖：

```text
load p ─value→ add
```

但两个 store 即使没有值依赖也不能随意交换。DAG 使用 chain token：

```text
EntryToken
    │ chain
    ▼
 store p
    │ chain
    ▼
 load p ─value→ add
```

chain 不是机器寄存器值，而是编译器内部对副作用顺序的建模。

## 4.3 Glue 是什么

Glue 用于要求节点在调度中保持紧邻或绑定，例如某些 compare/condition-code 与 consumer、
call sequence。它比普通数据依赖更强，但具体使用是 target-dependent。

## 4.4 典型流程

```text
LLVM IR BasicBlock
    │ SelectionDAGBuilder
    ▼
Initial DAG
    │ DAG Combine
    ▼
Type Legalization
    │ Legalize/Combine
    ▼
Operation Legalization / Vector Legalization
    │ DAG Combine
    ▼
Instruction Selection
    │ generic nodes → target-specific nodes
    ▼
DAG Scheduling
    │ linearize nodes
    ▼
MachineInstr
```

实际 combine/legalize 次数与顺序受版本、target 和 optimization level 影响。

## 4.5 Pattern matching

IR：

```llvm
%sum = add i64 %a, %b
%r = mul i64 %sum, 3
```

AArch64 可以把乘 3 组合成：

```asm
add x8, x0, x1
add x0, x8, x8, lsl #1
```

第二条相当于 `x8 + (x8 << 1)`，利用 shifted-register addressing form，避免通用 multiply。

## 4.6 `TargetLowering` 高频职责

- 设置 register type/count 和合法操作 action。
- custom lower 特定 DAG node。
- lower formal arguments、call、return。
- 提供 target combine。
- 选择 addressing mode、boolean content、setcc result type 等目标语义。
- 描述 target libcalls 和 operation properties。

## 4.7 DAG combine 与 IR InstCombine 的区别

| IR InstCombine | DAG Combine |
|---|---|
| 目标相对无关 LLVM IR | 已知更多目标合法类型/操作 |
| 产生 LLVM IR | 产生/重写 DAG nodes |
| 在 IR pipeline | instruction selection 内部阶段 |
| 不直接决定 machine opcode | 可为具体 target pattern 创造形态 |

## 4.8 高频问答

**问：SelectionDAG 为什么通常按基本块？**

答：便于保证 DAG 无环和控制复杂度；跨块信息主要由 IR 优化、Machine CFG 和后续 pass
处理。这也限制了一部分跨块 instruction selection 机会。

**问：节点为什么能有多个结果？**

答：操作可能同时产生 value 和 chain，或产生 value 与 flags。例如 load 返回 loaded value
及新的 chain。

**问：chain 是否完全决定机器指令顺序？**

答：它表达必须保持的副作用依赖，scheduler 仍可在不违反所有数据/chain/glue 依赖的范围
内重排。

---

# 5. GlobalISel 指令选择

## 5.1 四个核心阶段

```text
LLVM IR
   │
   ▼
IRTranslator
   │  生成 G_* generic MachineInstr
   ▼
Legalizer
   │  改成目标支持的 LLT/operation
   ▼
RegBankSelect
   │  为 operand/result 选择 register bank
   ▼
InstructionSelect
   │  G_* → target opcode / register constraints
   ▼
Target Machine IR
```

pipeline 中还可能穿插 Combiner、CSE、constant folding 和 target-specific pass。

## 5.2 Generic MIR 示例

概念上的 generic 指令：

```mir
%0:_(s64) = COPY $x0
%1:_(s64) = COPY $x1
%2:_(s64) = G_ADD %0, %1
%3:_(s64) = G_MUL %2, %const3
$x0 = COPY %3
```

这里 `_` 表示尚未确定传统 register class，`s64` 是 Low-Level Type（LLT）。经过
RegBankSelect/InstructionSelect 后可变成 AArch64 `gpr64` 和 `ADDXrr/ADDXrs`。

## 5.3 LLT、Register Bank、Register Class

| 概念 | 回答的问题 |
|---|---|
| LLT | value 的低层形状：scalar/vector/pointer 和位宽 |
| Register Bank | 大类硬件存储域：GPR、FPR/SIMD 等 |
| Register Class | 更精确的可分配物理寄存器集合及约束 |

同一个 `s32` 可能根据操作在不同 bank；同一个 bank 内又可能有多个 register class。

## 5.4 GISel Combiner

GlobalISel 的 generic/target combine 负责：

- canonicalization；
- 常量折叠；
- 消除 legalization 产生的冗余；
- 将多个 generic op 合成目标更容易选择的 pattern；
- 在 selection 前后做 target-specific combine。

## 5.5 SelectionDAG 与 GlobalISel 对照

| 维度 | SelectionDAG | GlobalISel |
|---|---|---|
| 中间表示 | 每基本块 DAG | MachineInstr-based generic MIR |
| 类型 | EVT/MVT | LLT |
| 副作用顺序 | chain/glue | MachineInstr/CFG 和 memory operands |
| Legalization | TargetLowering/DAG legalizer | LegalizerInfo |
| 寄存器域 | DAG value type 后续约束 | 显式 RegBankSelect |
| 可调试性 | DAG dump/view | MIR 可序列化、可用 `-run-pass` |
| 成熟度 | 历史长、目标覆盖广 | 目标支持程度依架构而异 |

不要回答“GlobalISel 一定比 SelectionDAG 更快/更好”。编译时间、代码质量和目标成熟度
依 target 与 workload 而异。

## 5.6 高频问答

**问：为什么叫 GlobalISel，它一定做全函数全局选择吗？**

答：名称强调基于全函数 Machine IR 的基础设施，而不是 SelectionDAG 的局部 DAG 表示；
具体 matcher 和 combine 是否跨块仍由实现决定，不能简单理解为全局最优选择。

**问：RegBankSelect 和寄存器分配有什么区别？**

答：前者选择寄存器银行/operand mapping，如 GPR 还是 FPR；后者把 virtual register 映射到
具体物理寄存器，如 `$x8`，并处理冲突与 spill。

---

# 6. Machine IR、MachineFunction 与 MachineInstr

## 6.1 对象层级

```text
MachineModuleInfo
  └─ MachineFunction
       ├─ MachineBasicBlock
       │    └─ MachineInstr
       │         └─ MachineOperand
       ├─ MachineRegisterInfo
       ├─ MachineFrameInfo
       ├─ MachineConstantPool
       └─ MachineJumpTableInfo
```

MachineFunction 通常与一个 IR Function 对应，但保存 target-specific codegen 状态。

## 6.2 MachineOperand 类型

常见 operand：

- register（virtual/physical，def/use，implicit，kill/dead 等）；
- immediate；
- frame index；
- basic block；
- global address / external symbol；
- constant pool index；
- jump table index；
- register mask；
- metadata / CFI index 等。

## 6.3 一份真实 MIR

下面由本仓库 LLVM 22 对简单函数使用 AArch64 `-stop-after=finalize-isel` 生成，省略 YAML
中的无关字段：

```mir
---
name:            addmul
tracksRegLiveness: true
isSSA:           true
registers:
  - { id: 0, class: gpr64 }
  - { id: 1, class: gpr64 }
  - { id: 2, class: gpr64 }
  - { id: 3, class: gpr64 }
liveins:
  - { reg: '$x0', virtual-reg: '%0' }
  - { reg: '$x1', virtual-reg: '%1' }
body:             |
  bb.0.entry:
    liveins: $x0, $x1

    %1:gpr64 = COPY $x1
    %0:gpr64 = COPY $x0
    %2:gpr64 = ADDXrr %0, %1
    %3:gpr64 = ADDXrs %2, %2, 1
    $x0 = COPY %3
    RET_ReallyLR implicit $x0
...
```

最终汇编：

```asm
addmul:
    add x8, x0, x1
    add x0, x8, x8, lsl #1
    ret
```

## 6.4 MIR 中值得逐项解释的字段

```mir
%2:gpr64 = ADDXrr %0, %1
```

- `%2`：virtual register。
- `gpr64`：register class。
- `=` 左侧：def operand。
- `ADDXrr`：AArch64 target opcode。
- `%0, %1`：use operands。

```mir
RET_ReallyLR implicit $x0
```

`$x0` 虽未写在显式 assembly operand 中，但 return 指令隐式使用它。隐式 operand 对
liveness、调度和 verifier 都很重要。

## 6.5 MachineMemOperand

load/store MachineInstr 会带 `MachineMemOperand`，描述：

- load/store 性质；
- size、alignment；
- pointer provenance / pseudo source value；
- volatile、atomic ordering；
- alias metadata、range 等。

机器级优化不能只看 opcode 和地址寄存器，内存语义通常还在 MMO 中。

## 6.6 构造 MachineInstr

```cpp
const TargetInstrInfo *TII = MF.getSubtarget().getInstrInfo();
DebugLoc DL = MI.getDebugLoc();

Register Dst = MRI.createVirtualRegister(&TargetRC);
BuildMI(MBB, InsertPt, DL, TII->get(TargetOpcode), Dst)
    .addReg(Src1)
    .addReg(Src2)
    .addImm(Shift);
```

删除时：

```cpp
MI.eraseFromParent();
```

插入/删除后要维护 liveness、slot indexes、live intervals 或让相应分析失效，具体取决于
Pass 所处 pipeline 阶段。

## 6.7 MachineFunctionPass 骨架

LLVM 22 同时存在 legacy 和新 Machine Pass Manager 基础设施，代码风格取决于当前 pipeline。
概念骨架：

```cpp
class MyMachinePass : public PassInfoMixin<MyMachinePass> {
public:
  PreservedAnalyses run(MachineFunction &MF,
                        MachineFunctionAnalysisManager &MFAM) {
    const TargetInstrInfo *TII = MF.getSubtarget().getInstrInfo();
    MachineRegisterInfo &MRI = MF.getRegInfo();

    bool Changed = false;
    for (MachineBasicBlock &MBB : MF)
      for (MachineInstr &MI : make_early_inc_range(MBB))
        Changed |= transform(MI, *TII, MRI);

    return Changed ? PreservedAnalyses::none()
                   : PreservedAnalyses::all();
  }
};
```

## 6.8 MIR 测试

```bash
# 停在某个 codegen pass 后并输出 MIR/YAML
llc -mtriple=aarch64-linux-gnu -O2 \
    -stop-after=finalize-isel input.ll -o output.mir

# 从 MIR 运行一个机器级 pass
llc -mtriple=aarch64-linux-gnu \
    -run-pass=machine-cp output.mir -o -

# 在每个机器 pass 后验证
llc -verify-machineinstrs input.ll -o /dev/null

# 打印特定 pass 前后（名字以当前版本为准）
llc -print-before-all -print-after-all input.ll -o /dev/null
```

---

# 7. 虚拟寄存器、物理寄存器与 Register Class

## 7.1 三类概念

| 概念 | 例子 | 含义 |
|---|---|---|
| Virtual Register | `%17` | 编译器创建，数量近似无限 |
| Physical Register | `$x0`、`$v1` | 硬件/ABI 真实寄存器 |
| Register Class | `gpr64` | 满足类型与指令约束的一组物理寄存器 |
| Register Bank | GPR/FPR | 更粗粒度的寄存器存储域 |

## 7.2 为什么需要 Virtual Register

- instruction selection 不必提前解决全函数资源冲突。
- 可保持 machine SSA，def-use 清晰。
- target instruction constraints 先限制 class，RA 再选择具体物理寄存器。
- 便于 machine CSE、copy propagation 和 scheduling。

## 7.3 物理寄存器 Alias 与 Subregister

AArch64 `$w0` 是 `$x0` 的低 32 位视图；x86 的 RAX/EAX/AX/AL 也存在重叠。给一个 live
range 分配 `$x0` 会影响所有 alias register 的可用性。

`TargetRegisterInfo` 描述：

- physical register 集合和 alias；
- subregister index；
- register class；
- callee-saved / reserved register；
- frame register；
- register pressure set 等。

## 7.4 Reserved Register

SP、某些 platform register、thread pointer 或 target-specific scratch 可能被保留，不参与
普通分配。JVM/JIT 还可能固定线程寄存器或 heap base，必须通过 target/register allocation
约束正确表达，不能只在代码生成后“约定不要用”。

## 7.5 Register Mask

call 指令通常携带 regmask，表示该调用会 clobber 哪些物理寄存器。live range 跨 call 时，
RA 必须选择 call-preserved register，或 spill/reload。

```text
vreg live across call
      │
      ├─ 分配 callee-saved register
      ├─ 在 call 前 spill、call 后 reload
      └─ 若可重算，call 后 rematerialize
```

## 7.6 高频问答

**问：virtual register 有固定位宽吗？**

答：其约束由 LLT、register bank/class、subregister 等阶段信息表达；不能只从 `%17` 名字
判断。Selection 后 register class 通常提供明确约束。

**问：为什么同一个物理寄存器有多个名字？**

答：硬件支持不同宽度/子寄存器视图，写某个视图还可能隐式清零或保留其他位，target 必须
精确建模 super/subregister 和 lane。

---

# 8. 活跃性、LiveInterval 与 SlotIndex

## 8.1 活跃性的定义

若某寄存器值在程序点之后可能沿某条路径被使用，且此前没有被重新定义，则该值在该点
live。两个 live ranges 同时活跃且不能共享同一物理资源时发生干涉。

```text
%a = def
   ...          ← %a live
%b = def
   ...          ← %a 与 %b 同时 live，可能 interfere
use %a
   ...          ← %a dead，%b 仍可能 live
use %b
```

## 8.2 Live-in / Live-out 数据流方程

经典 block-level 方程：

```text
LiveOut[B] = ⋃ LiveIn[S]，S ∈ Succ(B)
LiveIn[B]  = Use[B] ∪ (LiveOut[B] - Def[B])
```

反向迭代直到不动点。LLVM 的 LiveIntervals 在此基础上构造更精细的指令位置区间。

## 8.3 SlotIndex

`SlotIndexes` 为 MachineInstr 分配稳定的有序位置。一个 instruction 附近可区分 base、early
clobber、register、dead 等 slot，使 def/use 和 early-clobber 时序可精确表示。

不要把 SlotIndex 当机器码 byte offset；它是编译器内部的程序点编号。

## 8.4 LiveInterval

```text
LiveInterval(vreg)
  ├─ Segment [start, end) → VNInfo #0
  ├─ Segment [start, end) → VNInfo #1
  └─ SubRange(lane mask) ...
```

- Segment 表示半开活跃区间。
- `VNInfo` 表示同一 register 不同 value definition。
- holes 表示区间中间不活跃。
- subranges/lane masks 描述部分寄存器 lane 的活跃性。

## 8.5 Kill/Dead flags

- kill：该 use 是沿当前表示上的最后一次使用，之后值不再 live。
- dead def：定义结果无后续 use。

这些 flags 是派生的 liveness 信息，不是程序语义源。修改 MachineInstr 后必须重算或正确
维护，不能把过时 kill flag 当成绝对真理。

## 8.6 高频问答

**问：LiveVariables 与 LiveIntervals 区别？**

答：前者偏离散的 block/instruction liveness，后者按 SlotIndex 表示区间和 value number，
直接服务于现代寄存器分配、split 和 spill。具体 pipeline 使用取决于 Pass。

**问：两个 vreg 的区间重叠就一定不能同寄存器吗？**

答：通常构成干涉，但还要考虑 register class、subregister lane、copy coalescing、early
clobber 和 target constraints。

---

# 9. 寄存器分配、Spill、Split 与 Coalescing

## 9.1 问题定义

输入：

- virtual registers 和 live intervals；
- physical registers、aliases、register classes；
- instruction operand constraints；
- calls 的 regmask；
- reserved/callee-saved registers；
- block frequency 与 spill cost。

输出：把 vreg 映射到 physical register，或插入 spill/reload/rematerialization，使所有时刻
资源约束满足。

## 9.2 图着色模型

```text
每个 live range = 图节点
两个 live range 干涉 = 图边
可用物理寄存器 = 颜色
```

一般图着色寄存器分配是 NP-hard。工程实现使用启发式，不追求全局最优证明。

## 9.3 LLVM Greedy RA 的高层流程

```text
计算 LiveIntervals / spill weights
             │
             ▼
按优先级选择一个 virtual live range
             │
             ▼
尝试分配可用 physical register
   ├─ 成功 → assign
   └─ 冲突
       ├─ evict 代价更低的已分配 interval
       ├─ split 当前 live range
       ├─ recolor / local repair
       └─ spill 到 stack
             │
             ▼
插入 reload/store 或 rematerialize，继续处理新 ranges
```

### 常见 allocator 对照

| Allocator | 特点 | 典型取舍 |
|---|---|---|
| Fast | 局部、编译速度优先 | O0/JIT 低层级常见，代码质量较保守 |
| Basic | 使用通用 live-range allocation 框架的基础实现 | 便于理解和测试，不代表默认最佳质量 |
| Greedy | eviction、splitting、hints、spill cost 等启发式 | 优化构建的主力，编译成本更高 |
| PBQP | 将部分约束建模为 Partitioned Boolean Quadratic Problem | 适合某些复杂寄存器约束，使用依 target/pipeline |

面试不要把 “greedy” 理解成遇到第一个空闲寄存器就结束。LLVM Greedy RA 还包含 allocation
order、live-range splitting、eviction、recoloring、hints 和多轮处理等机制。

## 9.4 Spill

### Before（概念 MIR）

```mir
%v:gpr64 = expensive_def
...
use %v
```

### After spill

```mir
%v:gpr64 = expensive_def
STORE %v, %stack.0
...
%reload:gpr64 = LOAD %stack.0
use %reload
```

spill 增加内存访问、栈空间和调度压力，因此 spill cost 会结合 use 次数、loop depth/block
frequency、rematerializability 等估算。

## 9.5 Live-range Splitting

不是把整个长区间都 spill，而是只在高压力区域外/内切分：

```text
长 live range： [==============================]
高压力区域：              [########]

split 后：      [register][spill][register]
```

这能保留热路径或多数区域的寄存器分配，但会引入 copy/spill 边界。

## 9.6 Rematerialization

如果值很便宜且无副作用，如小常量或简单地址，可在 use 附近重新计算，而不是 store/reload：

```text
spill:           def → store → ... → load → use
rematerialize:   ... → cheap-def → use
```

它用计算换内存，适合 cheap、safe、independent 的定义。

## 9.7 Coalescing

```mir
%b = COPY %a
```

若 `%a` 和 `%b` 的 live ranges 可安全合并且 class/constraint 允许，可分到同一物理寄存器，
COPY 随后消失。

过度 coalescing 会生成更长 live range、增加干涉和 spill 风险，所以不是“能合并就一定合并”。

## 9.8 Pre-colored / Fixed Register

调用约定、特殊指令可能要求 operand 位于固定物理寄存器。RA 要通过 copy、constraint 或
pre-colored interval 满足。例如 AArch64 整数返回值通常在 `$x0`。

## 9.9 Caller-saved 与 Callee-saved 决策

| 场景 | 可能偏好 |
|---|---|
| 短生命周期、不跨 call | caller-saved，避免函数保存恢复 |
| 长生命周期、跨多个 call | callee-saved，函数入口/出口保存一次可能更划算 |
| 叶子函数 | caller-saved 更充足，可能无需栈帧 |
| 冷路径跨 call | spill/reload 成本受频率影响 |

最终由 allocator、spill cost、hint 和 target constraints 综合决定。

## 9.10 高频问答

**问：寄存器不够时只会 spill 吗？**

答：还可 split、evict、recolor、coalesce/undo coalesce、rematerialize、改变 allocation order。

**问：为什么 RA 常在 instruction scheduling 之后？**

答：pre-RA scheduler 可重排以隐藏 latency 并控制 pressure；RA 需要相对确定的指令顺序
构造 live intervals。之后还可能有 post-RA scheduling，但自由度更低。

**问：spill slot 是普通 alloca 吗？**

答：不是 LLVM IR alloca，而是 `MachineFrameInfo` 中的 frame object，以 frame index 表示，
后续才解析成 SP/FP 相对地址。

---

# 10. 指令调度与处理器资源模型

## 10.1 调度目标

在不违反依赖的前提下安排指令顺序，以：

- 缩短关键路径；
- 隐藏 load/use latency；
- 提高 execution-unit throughput；
- 避免 pipeline hazard；
- 控制 register pressure；
- 利用 macro fusion / instruction pairing；
- 平衡代码尺寸和目标微架构偏好。

## 10.2 调度依赖

| 依赖 | 例子 | 能否交换 |
|---|---|---|
| RAW / true | A 定义 r，B 使用 r | 不能跨越 |
| WAR / anti | A 使用 r，B 重定义 r | 未重命名时受限 |
| WAW / output | A、B 都定义 r | 顺序受限 |
| Memory/order | aliasing load/store、barrier | 依 AA/MMO/ordering |
| Control/artificial | target 或 pass 添加约束 | 按边约束 |

虚拟寄存器 SSA 可消除许多 WAR/WAW，但 physical register、flags、implicit operands 和
post-RA 阶段仍会产生这些约束。

## 10.3 Latency 与 Throughput

- Latency：某指令结果到 consumer 可用需要多少周期。
- Reciprocal throughput：稳态下连续同类指令平均多久可发射一条。
- Resource occupancy：占用哪些执行资源多少周期。

低 latency 不一定高 throughput，二者不能混用。

## 10.4 一个调度例子

### Before

```asm
ldr x8, [x0]
add x9, x8, #1       // 立即依赖 load，可能等待
ldr x10, [x1]
add x11, x10, #1
```

### After（无 alias/顺序约束时）

```asm
ldr x8, [x0]
ldr x10, [x1]        // 用独立 load 填充 latency
add x9, x8, #1
add x11, x10, #1
```

但移动第二个 load 前必须确认 memory ordering、fault/exception 与 target constraints 允许。

## 10.5 Pre-RA 与 Post-RA Scheduling

| Pre-RA | Post-RA |
|---|---|
| virtual registers | physical registers |
| 重排自由度较大 | 受真实寄存器依赖限制 |
| 必须关注 register pressure | 可处理真实 hazard/latency |
| 可能影响后续 RA 难度 | 不能轻易增加新 vreg |

## 10.6 MachineScheduler 常见组件

```text
ScheduleDAG / SUnit dependency graph
Ready queues
Scheduling strategy
TargetSchedModel / ProcResource
RegisterPressureTracker
HazardRecognizer
Mutation（macro fusion、cluster 等）
```

## 10.7 高频问答

**问：为什么最短关键路径调度不一定最好？**

答：可能显著拉长 live ranges，导致 register pressure 和 spill；还可能破坏代码布局、fusion
或资源平衡。

**问：AA 为什么会影响 machine scheduling？**

答：若两个 memory operations 能证明不 alias，可移除不必要的 memory dependency，增加
调度自由度；MachineMemOperand 提供部分内存信息。

---

# 11. Calling Convention 与 Call Lowering

## 11.1 Calling Convention 决定什么

- 参数和返回值使用哪些寄存器/栈位置；
- caller-saved、callee-saved 集合；
- stack alignment 和 call frame；
- aggregate 如何拆分或间接传递；
- varargs 规则；
- sret/byval/inreg/extension 等 ABI 属性；
- tail call 的兼容条件。

## 11.2 AArch64 简化示例

```c
long add(long a, long b) { return a + b; }
```

典型 ABI：参数在 `x0`、`x1`，返回值在 `x0`：

```asm
add:
    add x0, x0, x1
    ret
```

这是 ABI 例子，不代表所有类型、平台变体和 calling convention 都相同。

## 11.3 Formal Arguments、Call、Return

后端需要分别 lower：

```text
Function entry:  外部 ABI locations → internal values
Call site:       internal values → callee ABI locations
Return:          internal return value → ABI return locations
```

SelectionDAG 常由 `TargetLowering::LowerFormalArguments/LowerCall/LowerReturn` 及 CC 分析处理；
GlobalISel 使用 `CallLowering` 体系。

## 11.4 `CCState` / `CCValAssign`

- `CCState` 追踪已分配寄存器、栈 offset 和调用约定状态。
- `CCValAssign` 描述一个 logical value 被放到哪个 physical register 或 stack slot，以及
  loc type、extension 等。
- TableGen 可生成 calling convention assignment functions，也可由 C++ custom logic 处理。

## 11.5 Tail Call 与 musttail

普通 tail call 优化需要保证：

- caller/callee ABI 兼容；
- outgoing arguments 不破坏仍需数据；
- stack frame 可复用/调整；
- return value 转发兼容；
- varargs、sret、特殊 attribute 等满足限制。

`musttail` 是 IR 的强语义要求并受到 verifier 的严格签名/ABI 限制；不是“优化器尽量做”。

## 11.6 JVM/JIT 额外关注

JIT 可能同时存在：

- Java compiled-to-compiled convention；
- compiled-to-interpreter adapter；
- runtime stub convention；
- native/JNI platform ABI；
- safepoint/deopt/exception 特殊入口。

不能默认所有调用都使用系统 C ABI。调用约定必须和栈扫描、oop map、unwind、callee-saved
约定共同设计。

## 11.7 高频问答

**问：调用点和函数定义 calling convention 不一致会怎样？**

答：参数、返回值、保存寄存器和栈清理规则可能不一致，属于严重错误；IR verifier 能捕获
部分情况，但 JIT 外部入口的 C++ function pointer cast 也必须严格匹配 ABI。

**问：小整数参数为什么可能需要 signext/zeroext？**

答：ABI 可能规定寄存器中的高位扩展责任。attribute 告诉 caller/callee 和优化器具体语义，
缺失或错误会导致跨边界值解释不一致。

---

# 12. 栈帧、FrameIndex 与 Prologue/Epilogue

## 12.1 MachineFrameInfo

栈对象在较早 MIR 阶段以抽象 frame index 表示：

```mir
%0:gpr64 = LDRXui %stack.0, 0
STRXui %1, %stack.1, 0
```

`MachineFrameInfo` 保存对象 size、alignment、fixed/local/spill 性质、offset、stack size、
callee-saved 信息、是否有 dynamic alloca 等。

## 12.2 为什么不早早确定 SP offset

在寄存器分配前还不知道：

- 有多少 spill slots；
- 保存哪些 callee-saved registers；
- 是否需要 stack realignment；
- outgoing call frame 多大；
- frame pointer/base pointer 是否需要；
- shrink wrapping 后 prologue 放在哪里。

因此先用 frame index，后期统一布局并消除。

## 12.3 Prologue/Epilogue Insertion

概念 prologue：

```asm
sub sp, sp, #frame_size
stp x29, x30, [sp, #offset]
mov x29, sp
```

概念 epilogue：

```asm
ldp x29, x30, [sp, #offset]
add sp, sp, #frame_size
ret
```

真实顺序、pre/post-index、CFI、PAC、shadow call stack 等取决于 target 和 features。

## 12.4 Frame Pointer 何时需要

- dynamic alloca 或动态 SP 调整；
- stack realignment；
- unwind/debug/profile 要求；
- frame address 被取；
- target/ABI 强制；
- 禁用 frame-pointer elimination；
- 函数结构使 SP-relative addressing 不稳定。

省略 FP 可多一个通用寄存器并减少指令，但调试、unwind 和 profiling 可能受影响。

## 12.5 Callee-saved Registers

如果 RA 使用 callee-saved physical register，函数通常在 prologue 保存、epilogue 恢复。
保存范围可以通过 shrink wrapping 移到只在需要该寄存器的路径周围，而不是统一入口/出口。

## 12.6 Stack Realignment

若局部对象要求高于 ABI 默认栈对齐，需要动态对齐 SP，并可能保留原始 SP/base pointer 以便
恢复。dynamic alloca 与 realignment 组合会显著增加 frame lowering 复杂度。

## 12.7 FrameIndex Elimination

目标 `TargetRegisterInfo`/frame lowering 根据最终对象 offset，把：

```mir
LOAD %stack.3
```

改成：

```asm
ldr x8, [sp, #offset]
```

若 offset 超出目标 addressing immediate 范围，还需 materialize 大 offset 或使用 scavenged
scratch register。

## 12.8 Register Scavenger

post-RA 阶段理论上已无自由 vreg，但某些 frame index elimination、large immediate expansion
需要临时物理寄存器。RegisterScavenger 在局部寻找不活跃寄存器，必要时紧急 spill。

## 12.9 高频问答

**问：spill slot 与局部 alloca 能共用栈空间吗？**

答：在生命周期、对齐和安全约束允许时可通过 stack slot coloring 等复用，但不是天然共用。

**问：叶子函数一定没有栈帧吗？**

答：不一定。它仍可能有 spill、local object、动态对齐、保存寄存器、安全机制或调试要求。

**问：red zone 是什么？**

答：某些 ABI 允许函数在 SP 下方固定区域临时使用而不调整 SP；是否存在、大小及中断语义
由 ABI 决定，不能跨平台假设。

---

# 13. Machine SSA、PHI Elimination 与 Two-Address

## 13.1 Machine SSA

Instruction selection 后通常以 virtual register SSA 表示：每个 vreg 一个 def，控制流汇合
用 Machine PHI。

```mir
bb.1:
  %1:gpr32 = ...
  G_BR %bb.3
bb.2:
  %2:gpr32 = ...
  G_BR %bb.3
bb.3:
  %3:gpr32 = PHI %1, %bb.1, %2, %bb.2
```

## 13.2 PHI Elimination

概念变换：

```mir
bb.1:
  %phi.tmp = COPY %1
  BR bb.3
bb.2:
  %phi.tmp = COPY %2
  BR bb.3
bb.3:
  ; 使用 %phi.tmp
```

PHI 是并行复制语义，存在 copy cycle 时需要临时寄存器或更复杂处理。critical edge 可能要
拆分，避免一条 edge 专属 copy 影响源块其他 successor。

## 13.3 Two-Address 指令

x86 等目标常有：

```text
two-address:  dst = dst + src
three-address MIR: out = lhs + rhs
```

需要插入 COPY 并约束 def/use tied：

```mir
%out = COPY %lhs
%out = ADD %out(tied-def 0), %rhs
```

Coalescing 和 RA 尽量让 `%out` 与 `%lhs` 分到同一物理寄存器，消除 COPY。

## 13.4 Early-clobber

某些指令在读完所有 inputs 前就写 destination，因此 destination 不能与特定 input 分配到
同一物理寄存器。early-clobber operand flag 向 RA 表达该约束。

## 13.5 高频问答

**问：为什么不保留 PHI 到最终机器码？**

答：普通 ISA 没有“根据前驱边选择寄存器值”的 PHI 指令，必须 lower 成边上的 copy 或通过
coalescing 消除。

**问：PHI elimination 后仍是 SSA 吗？**

答：通常会破坏严格 machine SSA，MachineFunction properties 会反映当前状态；后续 pass
不能继续假设每个 register 只有单 def。

---

# 14. TableGen 高频知识

## 14.1 TableGen 是什么

TableGen 是声明式 record language 和生成器框架。后端 `.td` 通常描述：

- registers、subregisters、register classes；
- instructions、operands、encoding；
- SelectionDAG/GISel patterns；
- calling conventions；
- processor resources、itinerary/scheduling model；
- assembler parser/matcher、disassembler decoder；
- target features 和 predicates。

## 14.2 常见语法概念

| 语法 | 用途 |
|---|---|
| `class` | 可参数化记录模板 |
| `def` | 定义一个具体 record |
| `multiclass` | 批量生成多组记录的模板 |
| `defm` | 实例化 multiclass |
| `let` | 覆盖字段 |
| `foreach` | 生成重复定义 |
| `dag` | pattern/operand 结构表示 |

## 14.3 指令定义概念示例

```tablegen
def MYADDrr : MyInst<...> {
  let OutOperandList = (outs GPR64:$dst);
  let InOperandList  = (ins GPR64:$lhs, GPR64:$rhs);
  let AsmString = "myadd $dst, $lhs, $rhs";
  let Pattern = [(set GPR64:$dst,
                      (add GPR64:$lhs, GPR64:$rhs))];
}
```

这是结构示意，不是某个 target 可直接编译的完整定义；真实基类还定义编码、namespace、
size、TSFlags、decoder 等。

## 14.4 Pattern 匹配失败的常见原因

- legalization 后类型/操作已改变；
- operand/register class 不匹配；
- immediate predicate 不满足；
- feature predicate 未启用；
- pattern root/canonical form 不一致；
- chain/glue 或 memory node 结构不同；
- 更早 combine 已改写表达式；
- complex pattern 或 target lowering 未实现。

## 14.5 Scheduling Model

TableGen 可描述：

```text
SchedMachineModel
ProcResource / ProcResGroup
WriteRes / ReadAdvance
latency / release cycles
micro-op 数量
unsupported features
```

MachineScheduler 和 `llvm-mca` 可利用这些信息估计资源压力、latency 和 throughput。

## 14.6 高频问答

**问：新增指令只写一条 TableGen pattern 就够吗？**

答：通常不够。还可能需要寄存器/operand 定义、encoding、asm parser/printer、disassembler、
legalization、custom lowering、scheduler model、MC code emitter 和测试。

**问：TableGen 是运行时解释的吗？**

答：不是。构建 LLVM 时由 `llvm-tblgen` 读取 `.td`，生成 `.inc`/C++ 表和 matcher，再编译进
LLVM 工具或库。

---

# 15. MC 层、汇编器、Fixup 与 Relocation

## 15.1 MachineInstr 到 MCInst

```text
MachineInstr
  target pseudo expansion / lowering
          │
          ▼
MCInst(opcode + MCOperand)
          │
          ├─ AsmPrinter → assembly text
          │
          └─ MCCodeEmitter → instruction bytes + fixups
                                  │
                                  ▼
                         MCAssembler / ObjectWriter
                          sections + symbols + relocations
```

## 15.2 MCInst 的特点

- target opcode；
- physical register、immediate、`MCExpr` 等 operands；
- 不承载完整 MachineFunction 分析状态；
- 接近编码和汇编语法层；
- 可被 integrated assembler/disassembler 使用。

## 15.3 Fixup 与 Relocation

```asm
bl external_function
```

编译时不知道 `external_function` 最终地址：

1. CodeEmitter 先在指令相关位域留下占位。
2. 产生 `MCFixup`，记录 offset、kind 和 expression。
3. 若 assembly 阶段能解析本地符号距离，直接应用 fixup。
4. 否则 ObjectWriter 生成目标格式 relocation。
5. linker/JIT linker 知道最终地址后修补。

```text
Fixup：MC 组装过程中“这里以后需要修”的内部描述
Relocation：写入目标文件、交给 linker/loader 的重定位记录
```

不是每个 fixup 最终都会成为 relocation。

## 15.4 Object File 组成

常见 ELF 内容：

- `.text`、`.rodata`、`.data`、`.bss`；
- symbol table、string table；
- relocation sections；
- unwind/CFI、debug sections；
- COMDAT/group、TLS、GNU notes 等。

LLVM 通过 `MCStreamer` 抽象输出事件，具体 ELF/COFF/Mach-O streamer/writer 产生不同格式。

## 15.5 AsmPrinter 为什么名字容易误导

AsmPrinter 不仅打印文本汇编。在 object emission 路径中，它把 MachineInstr lower 为 MC
层事件/MCInst，并通过 MCStreamer 发往 object writer。因此它是 Machine → MC 的关键桥梁。

## 15.6 高频问答

**问：编码是 TableGen 自动完成的吗？**

答：很多固定字段和 decoder/encoder 表可生成，但 complex operands、fixups、pseudo、target
特殊编码仍需要 C++ hooks 和 MCCodeEmitter/AsmBackend 配合。

**问：静态链接 relocation 与 JIT relocation 有何共同点？**

答：都在符号/地址确定后把表达式应用到代码或数据位置；区别是 object linker 写文件，JIT
linker 通常直接在分配的内存中修补并处理权限/cache。

---

# 16. 分支、常量池、Jump Table 与代码布局

## 16.1 Branch Relaxation

某些 branch encoding 只能覆盖有限距离。早期布局时目标可能在范围内，后续代码扩张后可能
超范围，需要：

- 换成长分支形式；
- 插入 branch island/veneer；
- 条件反转 + 无条件长跳；
- target-specific expansion。

因此最终 branch encoding 往往必须在接近布局确定时处理。

## 16.2 Constant Pool

无法直接编码的大常量、浮点常量或地址可能放入 per-function/module constant pool，通过
PC-relative load 等方式访问。目标要处理：

- alignment；
- reachability/range；
- duplication/island；
- relocation；
- section/layout。

## 16.3 Jump Table

密集 switch 常可变为：

```text
range check
index = value - min
target = jump_table[index]
indirect branch target
```

稀疏 case 可能更适合 binary search、bit test 或 compare chain。决策取决于 case density、
profile、code size、PIC 和目标间接分支成本/安全机制。

## 16.4 Block Placement

根据 CFG probability/frequency 把热路径放成 fallthrough、减少 taken branches、改善 I-cache
局部性，并将冷块/outlining 放远。布局改变 block 地址，进一步影响 branch relaxation、
jump table 和 EH range。

## 16.5 Tail Duplication

复制小块到 predecessor 可消除 branch、暴露 fallthrough 和局部调度机会，但增加 code size。
它在机器级尤其能利用真实指令成本和布局信息。

---

# 17. JIT 代码发射与运行时问题

## 17.1 JIT 后端路径

```text
ThreadSafeModule
    │ IRTransformLayer
    ▼
IRCompileLayer / TargetMachine
    ▼
object buffer
    │ JITLink / RuntimeDyld 类链接层
    ▼
分配 code/data memory
    │ resolve symbols + apply relocations
    ▼
设置内存权限、flush instruction cache
    ▼
发布 callable symbol address
```

具体 ORC layer 组合因项目配置而异。

## 17.2 W^X 与内存权限

安全系统通常不允许页面同时 Writable + Executable：

```text
分配 RW memory
  → 写入代码并应用 relocation
  → 切换为 RX
  → flush I-cache（目标需要时）
  → 发布入口
```

并发 JIT 必须保证代码完全链接、权限和 cache 同步完成后，其他线程才看到入口。

## 17.3 符号与 Stub

远距离调用、lazy compilation、重定义或地址超范围可能使用 stub/GOT/PLT 类间接层：

```text
caller → stable stub → current implementation
```

更新 stub target 可以不修改所有 caller，但要处理原子发布、代码卸载和并发执行。

## 17.4 Code Cache 与卸载

卸载 JIT code 之前需保证：

- 没有线程正在执行该代码；
- 没有 return address、inline cache 或函数指针仍指向它；
- unwind/debug/profiler registration 已撤销；
- relocation 指向的 data/stub 生命周期仍合法；
- GC/deopt metadata 与 code 一起管理。

ORC 的 `ResourceTracker` 可帮助分组管理资源，但语言运行时可达性仍需 JVM/JIT 自己协调。

## 17.5 JVM 特有后端元数据

生成机器码不等于 JIT 编译完成，还可能要生成：

- oop maps / stack maps；
- safepoint PCs；
- deoptimization frame state；
- exception handler table；
- implicit null-check table；
- patching/inline-cache sites；
- source/bytecode PC mapping；
- unwind/debug/profiler information。

机器级 Pass 若移动、删除或复制相关 instruction，必须同步这些 metadata 或使用稳健的标记
机制追踪最终 PC。

---

# 18. 新增后端指令/优化的实现框架

## 18.1 新增一条目标指令

```text
1. ISA 语义与 encoding
2. Register operands / immediate constraints
3. TableGen instruction definition
4. AsmString、parser/printer、aliases
5. Disassembler decoder
6. Selection pattern 或 custom selector
7. Legalizer/TargetLowering（若 generic op 尚不合法）
8. Scheduling model 和 side-effect flags
9. MCCodeEmitter / fixup（若需要）
10. MIR、asm、encoding、negative tests
```

### 测试层次

| 层次 | 工具/检查 |
|---|---|
| IR → asm | `llc` + FileCheck |
| MIR transform | `llc -run-pass` |
| asm → encoding | `llvm-mc -show-encoding` |
| bytes → asm | `llvm-mc --disassemble` / `llvm-objdump` |
| feature gating | `+feature` 成功、`-feature` 报错 |
| invalid operands | parser diagnostic negative test |

## 18.2 Machine peephole Pass

例：

```mir
%dst = ADDri %src, 0
```

改成 copy 或直接替换。检查框架：

```text
匹配 opcode 与 operands
  → 检查 flags/implicit operands/MMO
  → 检查当前 pre-RA/post-RA 阶段允许的 register 形式
  → 检查 subregister/tied/early-clobber constraints
  → 创建替代 MI，复制必要 debug/memory info
  → 更新 MRI/liveness/LIS/SlotIndexes
  → 删除旧 MI
  → MachineVerifier
```

### 简化骨架

```cpp
bool foldAddZero(MachineInstr &MI, const TargetInstrInfo &TII,
                 MachineRegisterInfo &MRI) {
  if (MI.getOpcode() != MyTarget::ADDri)
    return false;
  if (!MI.getOperand(2).isImm() || MI.getOperand(2).getImm() != 0)
    return false;

  Register Dst = MI.getOperand(0).getReg();
  Register Src = MI.getOperand(1).getReg();
  if (!canReplaceReg(Dst, Src, MRI))
    return false;

  MRI.replaceRegWith(Dst, Src);
  MI.eraseFromParent();
  return true;
}
```

这是面试骨架；生产实现还要处理 register classes、subregs、constraints、debug uses、liveness
和 pass stage。

## 18.3 新增 SelectionDAG custom lowering

高层步骤：

1. 在 target lowering 构造中为 `ISD::Opcode + MVT` 设置 `Custom` action。
2. 在 `LowerOperation` 分派到目标 helper。
3. 用 `DAG.getNode` 构造合法 DAG nodes。
4. 保留 chain/glue 和多结果语义。
5. 提供 TableGen pattern 或 target node selection。
6. 添加 legalize/isel tests。

错误地丢失 chain 是严重后端 bug，可能重排有副作用的 memory/call 操作。

## 18.4 新增 GlobalISel 支持

1. `LegalizerInfo` 声明 generic op/type action。
2. `CallLowering` 处理参数/返回（若涉及）。
3. `RegisterBankInfo` 提供 mapping/cost。
4. TableGen `GICombineRule`/selection patterns 或 C++ selector。
5. 检查 legalized、regBankSelected、selected properties。
6. 分阶段 MIR tests，定位失败发生在哪一层。

---

# 19. 高频现场题与排障题

## 19.1 题一：为什么乘 3 没有生成 `mul`

IR：

```llvm
%r = mul i64 %x, 3
```

AArch64：

```asm
add x0, x0, x0, lsl #1
```

**回答**：指令选择/DAG combine/target combine 识别常数乘法，可利用 shifted-register add
完成 `x + 2*x`。是否合法由整数 wrap 语义支持，是否有收益由目标指令成本、latency、size
决定。不能笼统说“add 永远比 mul 快”。

## 19.2 题二：为什么出现很多 COPY，最终汇编却没有

MIR 中 COPY 用于：

- ABI physical register 与 vreg 之间传值；
- PHI elimination；
- two-address constraint；
- register class/bank crossing；
- split live range 边界。

RA/coalescer 若把两侧分配到同一物理寄存器，COPY 成为 identity 并删除。无法 coalesce 时
才保留为真实 move，或由 target copy hook 展开。

## 19.3 题三：为什么换了指令顺序后反而 spill 更多

调度把 definition 提前或 use 推后，拉长 live range，使同时活跃值增加，超过可用物理寄存器。
这是 latency 优化与 register pressure 的典型权衡。应查看 pressure sets、LiveIntervals、
block frequency，并让 scheduler strategy 同时考虑关键路径与压力。

## 19.4 题四：只在 Release 出现错误机器码，怎样查

```text
1. 保存 LLVM IR、triple、CPU/features、llc 完整参数
2. 判断是 IR 优化错误还是 codegen 错误：比较 -O0/-O2、解释执行/其他 target
3. 使用 -verify-machineinstrs
4. 用 -stop-before/-stop-after 导出 MIR
5. 二分 machine pass，使用 -run-pass 最小复现
6. llvm-reduce 缩减 IR/MIR（按工具支持）
7. 检查 stale kill/dead、LIS/SlotIndex 未更新、implicit operand、regmask
8. 检查 TableGen side-effect/constraints/encoding 声明
9. 检查 poison/undef 是否在 IR 层已允许该结果
10. 建立 MIR + encoding 回归测试
```

## 19.5 题五：如何分析一次 spill

回答顺序：

1. 找被 spill 的 vreg、register class、live interval。
2. 看是否跨 call、跨热 loop、是否有固定寄存器约束。
3. 查同一 pressure set 上的竞争 intervals。
4. 看 spill weight、allocation order、hints 和 eviction。
5. 判断调度是否把 range 拉长。
6. 判断能否 rematerialize、split 或缩短 live range。
7. 看最终 spill/reload 是否位于热路径。
8. 用 benchmark 验证，避免只凭静态指令数判断。

## 19.6 题六：如何判断是 selection 还是 encoding 的 bug

```text
错误 opcode/operand/register：      多半在 lowering/isel/MIR pass
MIR 正确但汇编文本错误：           AsmPrinter/InstPrinter
汇编文本正确但 bytes 错：          MCCodeEmitter/fixup
bytes 正确但链接地址错：            relocation/object writer/JITLink
disassemble 与 ISA 不符：           decoder table/disassembler
```

通过 MIR、`llvm-mc -show-encoding`、`llvm-objdump -d` 分层定位。

## 19.7 题七：手画一个跨 call 的 live range

```text
def %v
  │
  │ %v live
  ▼
call @foo      ← regmask clobbers caller-saved registers
  │
  │ %v still live
  ▼
use %v
```

可选方案：callee-saved register、call 前 spill/后 reload、在 call 后 rematerialize，或重构
代码缩短 lifetime。选择由 cost model 决定。

## 19.8 题八：Machine Pass 删除指令前检查什么

- 显式与隐式 defs/uses；
- side effects、mayLoad/mayStore、unmodeled side effects；
- terminator、call、barrier、convergent/target flags；
- MachineMemOperands；
- bundled instructions；
- debug instruction/reference；
- liveness、kill/dead、LiveIntervals、SlotIndexes；
- CFI、stack map、patchpoint、statepoint 类特殊语义。

“destination 无 use”远远不足以证明 MachineInstr 可删除。

---

# 20. 面试前速背表

## 20.1 一句话回答

| 主题 | 一句话 |
|---|---|
| Legalization | 把非法类型/操作变成目标可选择的合法形态 |
| SelectionDAG | 用 DAG data/chain/glue 依赖完成 legalize、combine、selection、schedule |
| GlobalISel | 在 generic MIR 上经过 IRTranslator、Legalizer、RegBankSelect、ISel |
| MachineInstr | 仍可含 vreg、pseudo、frame index 和丰富机器语义的后端指令 |
| MCInst | 接近编码层的 target opcode + operands |
| Register Bank | GPR/FPR 等粗粒度寄存器域 |
| Register Class | 一组满足指令/类型约束的可分配物理寄存器 |
| LiveInterval | 按 SlotIndex 表示的 vreg 活跃 segments 与 value numbers |
| Register Allocation | vreg → physical register，冲突时 split/spill/remat |
| Coalescing | 合并 copy 两侧 live ranges，使 move 消失 |
| Rematerialization | use 附近重算便宜值，替代 spill/reload |
| Pre-RA Schedule | 在 vreg 阶段平衡 latency、资源和 register pressure |
| Calling Convention | 定义参数、返回、保存寄存器和 stack ABI |
| FrameIndex | 栈对象的抽象编号，后期解析为 SP/FP offset |
| PHI Elimination | 将 Machine PHI lower 成前驱 edge 上的并行复制 |
| TableGen | 构建时生成指令、寄存器、matcher、encoding 等 C++ 表 |
| Fixup | MC 层尚待解析的位域修补描述 |
| Relocation | 交给 linker/loader/JIT linker 的地址修补记录 |

## 20.2 高频对比表

| A | B | 核心区别 |
|---|---|---|
| LLVM IR | MIR | 目标相对无关 SSA vs target-specific machine representation |
| MachineInstr | MCInst | 优化/RA 信息丰富 vs 接近编码 |
| DAG chain | data edge | 副作用顺序 vs 值依赖 |
| Register Bank | Register Class | 粗存储域 vs 精确可分配集合 |
| Spill | Rematerialize | 保存到内存再加载 vs use 附近重算 |
| Caller-saved | Callee-saved | caller 跨调用保存 vs callee 使用时保存 |
| Pre-RA schedule | Post-RA schedule | 自由度大且关注 pressure vs 真实寄存器/hazard |
| Fixup | Relocation | MC 内部待修位置 vs 输出给链接器的记录 |
| SelectionDAG | GlobalISel | DAG/EVT/chain vs generic MIR/LLT/reg bank |
| Legal | Profitable | 能否正确生成 vs 是否值得采用 |

## 20.3 回答后端问题的八步框架

```text
1. 输入 IR/MIR 是什么
2. 当前 pipeline 阶段有什么 invariants
3. target 提供哪些 legality/constraint
4. 变换生成什么 node/opcode/register form
5. 数据、内存和 implicit dependencies 如何保持
6. liveness、register pressure、ABI、frame 如何影响
7. 如何 lower 到 MC encoding/relocation
8. 用什么 MIR/asm/encoding 测试验证
```

## 20.4 高频错误回答

| 错误说法 | 正确补充 |
|---|---|
| isel 后已经是机器码 | 仍有 MIR 优化、调度、RA、frame、MC emission |
| vreg 数量无限所以没有成本 | vreg 最终竞争有限物理寄存器，过多 live values 导致 spill |
| RegBankSelect 就是 RA | 它只选 bank，RA 才选具体 physical register |
| kill flag 是程序语义 | 它是派生 liveness，修改 MIR 后可能过时 |
| spill 只是多一次 store/load | 还影响栈帧、调度、cache、code size 和后续 ranges |
| callee-saved 一定优于 spill | 保存/恢复也有成本，取决于 calls、热度和 lifetime |
| TableGen 自动完成整个后端 | custom lowering、MC、frame、scheduler 等常需 C++ |
| Fixup 都会成为 relocation | 本地可解析 fixup 可在 assembler 内直接应用 |
| 汇编正确说明机器码正确 | 仍需检查 encoding、relocation 和最终链接地址 |
| JIT 编译完 bytes 就可执行 | 还需 relocation、权限、I-cache、原子发布和生命周期 |

## 20.5 自测题

1. 从 LLVM `mul i64 %x, 3` 解释到 AArch64 shifted add。
2. 解释 DAG load 为什么同时返回 value 和 chain。
3. 画出 GlobalISel 四阶段，并区分 LLT、bank、class。
4. 从真实 MIR 中找出 def、use、implicit use、physical live-in。
5. 画两个相互干涉的 LiveIntervals，并说明为何不能同色。
6. 对跨 call live range 给出三种处理方案。
7. 解释一次调度如何降低 latency 却增加 spill。
8. 描述 frame index 从创建到 SP/FP offset 的全过程。
9. 解释 Machine PHI 和 two-address copy 为什么可能被 coalesce。
10. 讲清 MCCodeEmitter、AsmBackend、ObjectWriter 的分工。
11. 给出一个 fixup 不需要输出 relocation 的例子。
12. 设计新增 target instruction 的完整测试矩阵。

---

## 附录：配套命令

```bash
LLC=jeandle-llvm/build-release/bin/llc
MC=jeandle-llvm/build-release/bin/llvm-mc
OBJDUMP=jeandle-llvm/build-release/bin/llvm-objdump

# IR → AArch64 汇编
$LLC -mtriple=aarch64-linux-gnu -O2 input.ll -o -

# IR → MIR，停在 instruction selection 后
$LLC -mtriple=aarch64-linux-gnu -O2 \
  -stop-after=finalize-isel input.ll -o output.mir

# 从 MIR 运行单个机器 Pass
$LLC -mtriple=aarch64-linux-gnu \
  -run-pass=machine-cp output.mir -o -

# 验证 MachineInstr
$LLC -verify-machineinstrs input.ll -o /dev/null

# 汇编并查看 encoding
echo 'add x0, x1, x2' | \
  $MC -triple=aarch64-linux-gnu -show-encoding

# 生成 object 并反汇编
$LLC -mtriple=aarch64-linux-gnu -filetype=obj input.ll -o output.o
$OBJDUMP -d -r output.o

# 查看隐藏 codegen 选项与 stop point
$LLC --help-hidden | less
```

建议把同一个 10 行 LLVM IR 分别保存为：instruction selection 后 MIR、register allocation
前后 MIR、最终 assembly 和 object disassembly。能沿这四层解释一次 value 如何变成 opcode、
virtual register、physical register 和 bytes，后端面试的主干就建立起来了。
