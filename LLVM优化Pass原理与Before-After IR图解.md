# LLVM 优化 Pass 原理与 Before/After IR 图解

> 适用版本：本仓库 LLVM 22.0.0git。示例使用 opaque pointer 和新 Pass Manager。
>
> 本文中的 After IR 主要表达“关键变化”，真实输出会受 LLVM 版本、DataLayout、属性、
> pipeline 顺序和成本模型影响，SSA 名称也可能不同。不要把示意输出当作固定字符串测试；
> 应用 `FileCheck` 检查关键结构。

## 阅读方法

每个 Pass 都回答六个问题：

1. 它消除什么成本？
2. Before IR 中识别什么模式？
3. After IR 为什么等价？
4. 需要哪些分析或规范形式？
5. 哪些反例必须拒绝？
6. 它为后续哪些 Pass 创造机会？

快速实验：

```bash
OPT=jeandle-llvm/build-release/bin/opt

$OPT -S -passes='mem2reg' input.ll -o -
$OPT -S -passes='instcombine,simplifycfg' input.ll -o -
$OPT -S -passes='default<O2>' input.ll -o output.ll
$OPT -passes='default<O2>' -print-pipeline-passes -disable-output input.ll
$OPT -passes='my-pass' -print-before=my-pass -print-after=my-pass \
  -verify-each -disable-output input.ll
```

## 目录

1. 优化的统一心智模型
2. mem2reg：内存局部变量转 SSA
3. SROA：聚合对象拆分
4. InstSimplify 与 InstCombine
5. Reassociate：表达式重结合
6. SimplifyCFG：控制流规范化
7. SCCP：稀疏条件常量传播
8. EarlyCSE 与 GVN：公共子表达式消除
9. DCE、ADCE 与 DSE
10. JumpThreading 与 CorrelatedValuePropagation
11. LICM：循环不变量代码移动
12. LoopRotate、IndVarSimplify 与 LSR
13. LoopUnswitch 与 LoopUnroll
14. Loop Vectorizer 与 SLP Vectorizer
15. Inlining 与跨过程优化
16. TailCallElim 与其他常见 Pass
17. Pass 之间如何形成优化链
18. 面试速查表与练习题

---

# 1. 优化的统一心智模型

## 1.1 一个优化必须同时满足三层条件

```text
                   ┌────────────────────┐
                   │ 语义等价/精化关系 │
                   └─────────┬──────────┘
                             │ 必须满足
                   ┌─────────▼──────────┐
                   │ 变换合法性 Legality│
                   └─────────┬──────────┘
                             │ 合法后再判断
                   ┌─────────▼──────────┐
                   │ 收益 Profitability │
                   └────────────────────┘
```

- **语义层**：不能改变任何原本有定义的可观察行为。
- **合法性层**：dominance、SSA、类型、CFG、异常、内存和目标约束均满足。
- **收益层**：运行时间、代码尺寸或功耗收益值得付出编译时间和代码膨胀。

合法不等于有收益。例如把 branch 转为 select 可能合法，却在可预测分支上变慢。TTI、BFI、
BPI、profile 和 size level 主要参与收益判断。

## 1.2 优化器经常做的五类动作

| 类型 | 目的 | 代表 Pass |
|---|---|---|
| 规范化 | 减少等价 IR 形态 | InstCombine、Reassociate、SimplifyCFG |
| 传播 | 把已知事实传到 use | SCCP、CorrelatedValuePropagation |
| 消除 | 删除重复或不可观察计算 | EarlyCSE、GVN、DCE、DSE |
| 移动 | 改变执行位置/频率 | LICM、GVN hoist/sink |
| 变形 | 改变循环、调用或向量粒度 | Unroll、Vectorize、Inlining |

## 1.3 为什么需要 pipeline，而不是“一个万能 Pass”

```text
SROA/mem2reg
    ↓ 暴露 SSA 值
InstCombine/Reassociate
    ↓ 建立 canonical form
SCCP/GVN
    ↓ 传播常量、消除重复值
SimplifyCFG
    ↓ 删除不可达边、合并块
Loop canonicalization
    ↓ 建立 preheader/LCSSA
LICM/IndVars/Vectorize/Unroll
    ↓ 产生新的局部冗余
InstCombine/DCE/SimplifyCFG
```

同一个 Pass 在默认 pipeline 中可能出现多次，因为前一个变换会为它重新创造机会。

## 1.4 看 Before/After 时先问什么

| 观察项 | 问题 |
|---|---|
| SSA | 新定义是否支配每条 use，尤其 PHI incoming edge？ |
| CFG | predecessor/successor 和 PHI 是否同步？ |
| Poison/UB | 是否新增 `nsw`、`nuw`、`exact`、`inbounds` 或投机执行？ |
| Memory | 中间是否有可能 clobber 的 store/call/atomic/volatile？ |
| Exception | 原本条件执行的 trap、throw、deopt 是否被提前？ |
| Analyses | DT、LI、SCEV、MSSA、BPI/BFI 是否更新或失效？ |
| Profit | 是否增加代码尺寸、寄存器压力或冷路径工作？ |

---

# 2. mem2reg：内存局部变量转 SSA

## 2.1 目标

`mem2reg` 即 `PromotePass`，把可提升的 scalar `alloca`、`load`、`store` 转成 SSA 值。
它消除栈槽访问，并让后续常量传播、CSE、DCE 直接沿 def-use 工作。

## 2.2 直线代码

### Before

```llvm
define i32 @straight(i32 %x) {
entry:
  %slot = alloca i32, align 4
  store i32 %x, ptr %slot, align 4
  %v = load i32, ptr %slot, align 4
  %r = add i32 %v, 1
  ret i32 %r
}
```

### After：`opt -S -passes='mem2reg'`

```llvm
define i32 @straight(i32 %x) {
entry:
  %r = add i32 %x, 1
  ret i32 %r
}
```

### 图解

```text
Before def-use:  %x → store → [stack slot] → load → add
After def-use:   %x ───────────────────────────→ add
```

load 的值必然是最近 store 的 `%x`，栈槽地址没有逃逸，因此可直接替换。

## 2.3 控制流汇合：为什么需要 PHI

### Before

```llvm
define i32 @choose(i1 %c, i32 %a, i32 %b) {
entry:
  %slot = alloca i32, align 4
  br i1 %c, label %then, label %else

then:
  store i32 %a, ptr %slot, align 4
  br label %merge

else:
  store i32 %b, ptr %slot, align 4
  br label %merge

merge:
  %v = load i32, ptr %slot, align 4
  ret i32 %v
}
```

### After

```llvm
define i32 @choose(i1 %c, i32 %a, i32 %b) {
entry:
  br i1 %c, label %then, label %else

then:
  br label %merge

else:
  br label %merge

merge:
  %slot.0 = phi i32 [ %a, %then ], [ %b, %else ]
  ret i32 %slot.0
}
```

```text
                 entry
                /     \
          then: a     else: b
                \     /
                 merge
             phi [a],[b]
```

经典算法：

1. 找到 alloca 的定义块。
2. 在 iterated dominance frontier 放 PHI。
3. 沿 dominator tree 重命名，每条路径维护当前值。
4. 用当前值替换 load，删除 store、load 和 alloca。

## 2.4 不能提升的反例

```llvm
%slot = alloca i32
call void @capture(ptr %slot) ; 地址可能逃逸并被未知代码读写
%v = load i32, ptr %slot
```

常见限制：地址被传出、做了不受支持的 pointer 运算、以非直接 load/store 方式使用、
volatile 访问等。`mem2reg` 主要提升 entry block 中可识别的 allocas。

## 2.5 面试追问

- 为什么前端不直接生成 SSA？——复杂控制流下先生成 alloca 简单可靠，再统一提升。
- PHI 为什么放在 dominance frontier？——那里是不同定义路径首次汇合且单一定义不再
  支配全部路径的位置。
- mem2reg 和 SROA 谁先谁后？——SROA 可拆聚合，暴露更多可提升标量；pipeline 会组合。

---

# 3. SROA：聚合对象拆分

## 3.1 目标

SROA（Scalar Replacement of Aggregates）分析 alloca 的访问切片，把结构体、数组或宽对象
拆成更小的独立标量；随后可提升到 SSA，减少不必要的 load/store 和整体复制。

### Before

```llvm
%Pair = type { i32, i32 }

define i32 @pair(i32 %x, i32 %y) {
entry:
  %p = alloca %Pair, align 4
  %px = getelementptr inbounds %Pair, ptr %p, i32 0, i32 0
  %py = getelementptr inbounds %Pair, ptr %p, i32 0, i32 1
  store i32 %x, ptr %px, align 4
  store i32 %y, ptr %py, align 4
  %a = load i32, ptr %px, align 4
  ret i32 %a
}
```

### After（关键结果）

```llvm
define i32 @pair(i32 %x, i32 %y) {
entry:
  ret i32 %x
}
```

```text
聚合 alloca %p
  ├─ slice [0,4): field x ──→ scalar SSA %x
  └─ slice [4,8): field y ──→ 无 use，删除
```

## 3.2 SROA 比 mem2reg 多做了什么

mem2reg 要求 alloca 已经像一个标量变量；SROA 能理解 GEP、bit slice、部分访问、部分
memcpy 等，将一个对象切成若干 partition，再为每部分选择合适表示。

## 3.3 反例与边界

- 对象地址逃逸到未知调用。
- 访问区间动态且无法精确拆分。
- overlapping stores/loads 需要字节级重构。
- packed struct、非自然对齐、endianness 和 padding 不能凭直觉处理。
- atomic/volatile 和特殊 address space 限制变换。

## 3.4 面试追问

- SROA 为什么能促进向量化？——也可能把不合适聚合拆开，暴露规则 scalar access；反过来
  后续 vectorizer 再按成本打包。
- 为什么必须使用 DataLayout？——字段 offset、padding、store size 都由目标 ABI 决定。

---

# 4. InstSimplify 与 InstCombine

## 4.1 区别

| 项目 | InstSimplify | InstCombine |
|---|---|---|
| 形式 | 查询/工具及 simplification pass | worklist 驱动的变换 Pass |
| 是否创建复杂新 IR | 通常只返回已有值或常量 | 可以创建、替换指令 |
| 主要目标 | 快速证明表达式等于现有值 | 化简并建立 canonical form |
| 典型用途 | 自定义 Pass 中局部查询 | 默认优化 pipeline 反复运行 |

## 4.2 代数恒等式

### Before

```llvm
define i32 @identity(i32 %x) {
entry:
  %a = add i32 %x, 0
  %b = mul i32 %a, 1
  %c = xor i32 %b, 0
  ret i32 %c
}
```

### After

```llvm
define i32 @identity(i32 %x) {
entry:
  ret i32 %x
}
```

## 4.3 canonicalization 示例

### Before

```llvm
%cmp = icmp eq i32 %x, 0
%sel = select i1 %cmp, i1 false, i1 true
```

### After

```llvm
%sel = icmp ne i32 %x, 0
```

规范形式减少后续 pass 需要匹配的表达式数量。

## 4.4 Poison 陷阱

看似普通的代数式不一定可随意化简：

```llvm
%r = mul i32 %x, 0
```

若 `%x` 是 poison，结果仍可能是 poison，而常量 `0` 不是 poison。因此变换必须遵循 LLVM
对 poison 的精确规则，不能直接套数学恒等式。LLVM 公共 simplify/combine 工具已经编码
了许多此类约束，自定义 Pass 不应重复发明不完整规则。

## 4.5 flags 不能盲目复制

```llvm
%a = add nsw i32 %x, 1
```

重写表达式时，`nsw`、`nuw`、`exact`、fast-math flags 是语义承诺。新表达式只有在重新
证明相应性质后才能保留或添加 flags。

---

# 5. Reassociate：表达式重结合

## 5.1 目标

通过排序和重新组织结合律运算，暴露常量合并、公共因子和 CSE 机会。整数普通 add/mul
与部分位运算较适合；浮点必须受 fast-math/reassociation 许可。

### Before

```llvm
%a = add i32 %x, 4
%b = add i32 %a, %y
%c = add i32 %b, 6
```

### After（示意）

```llvm
%a = add i32 %x, %y
%c = add i32 %a, 10
```

```text
Before tree: (((x + 4) + y) + 6)
After tree:  ((x + y) + 10)
```

## 5.2 公共因子机会

```llvm
; Before
%a = mul i32 %x, %z
%b = mul i32 %y, %z
%r = add i32 %a, %b

; 可能规范/组合为
%s = add i32 %x, %y
%r = mul i32 %s, %z
```

实际由哪个 Pass 完成最终折叠可能随版本和 pipeline 改变；重点是 Reassociate 把表达式
放入统一顺序，使 InstCombine/CSE 看见模式。

## 5.3 浮点为什么不同

IEEE 浮点加法通常不满足可自由使用的结合律：舍入、NaN、signed zero、异常状态都会让
`(a+b)+c` 与 `a+(b+c)` 不同。只有相应 fast-math 语义允许时才能重结合。

---

# 6. SimplifyCFG：控制流规范化

## 6.1 常量分支与不可达块

### Before

```llvm
define i32 @constant_branch() {
entry:
  br i1 true, label %yes, label %no

yes:
  ret i32 1

no:
  ret i32 2
}
```

### After

```llvm
define i32 @constant_branch() {
entry:
  ret i32 1
}
```

### CFG

```text
Before: entry ─true─→ yes → ret 1
          └──false─→ no  → ret 2

After:  entry → ret 1
```

## 6.2 空块转发与 PHI 更新

### Before

```llvm
a:
  br label %forward
forward:
  br label %merge
other:
  br label %merge
merge:
  %p = phi i32 [ 7, %forward ], [ 9, %other ]
```

### After

```llvm
a:
  br label %merge
other:
  br label %merge
merge:
  %p = phi i32 [ 7, %a ], [ 9, %other ]
```

删除转发块不仅改 branch，还必须把 PHI incoming block 从 `%forward` 改成 `%a`。

## 6.3 diamond 转 select

### Before

```llvm
entry:
  br i1 %c, label %t, label %f
t:
  %a = add i32 %x, 1
  br label %m
f:
  %b = add i32 %x, 2
  br label %m
m:
  %p = phi i32 [ %a, %t ], [ %b, %f ]
  ret i32 %p
```

### After（当两侧计算可安全投机且成本合适）

```llvm
entry:
  %a = add i32 %x, 1
  %b = add i32 %x, 2
  %p = select i1 %c, i32 %a, i32 %b
  ret i32 %p
```

如果某侧是可能 trap 的 load/div、会抛异常的 call 或昂贵计算，就不能仅为消除分支而
无条件执行两侧。

## 6.4 依赖与后续机会

SimplifyCFG 会使用 TTI、assumption、profile 等信息。CFG 变化会影响 DT、PDT、LI、
MemorySSA、BPI/BFI。它常在 SCCP、Inlining、Unroll 后运行，用来收尾新产生的常量分支。

---

# 7. SCCP：稀疏条件常量传播

## 7.1 两个同时求解的格

```text
Value lattice:  unknown/undef → constant → overdefined
CFG state:      edge not executable / executable
```

SCCP 只合并可执行 incoming edge 上的 PHI 值，因此能同时传播常量和发现不可达代码。

## 7.2 Before/After

### Before

```llvm
define i32 @sccp() {
entry:
  %x = add i32 40, 2
  %c = icmp eq i32 %x, 42
  br i1 %c, label %yes, label %no

yes:
  %r1 = mul i32 %x, 2
  ret i32 %r1

no:
  ret i32 0
}
```

### `sccp` 传播后的关键事实

```llvm
; %x = 42
; %c = true
; %no 不可执行
; %r1 = 84
```

### 再经 SimplifyCFG/DCE

```llvm
define i32 @sccp() {
entry:
  ret i32 84
}
```

这说明单个 Pass 的输出不一定是最终最简形式：SCCP 标记和替换，CFG/DCE 完成清理。

## 7.3 PHI 为什么必须忽略不可执行边

```llvm
entry:
  br i1 true, label %a, label %b
a:
  br label %m
b:
  %unknown = call i32 @unknown()
  br label %m
m:
  %p = phi i32 [ 7, %a ], [ %unknown, %b ]
```

普通“合并所有 incoming value”会认为 `%p` 非常量；SCCP 知道 `%b` edge 不可执行，因此
可把 `%p` 传播为 7。

## 7.4 面试算法回答

维护 SSA value worklist 和 CFG edge/block worklist；值 lattice 或可执行性单调向更保守
状态变化，反复处理 use、branch 和 PHI，直到无新变化，即达到不动点。

---

# 8. EarlyCSE 与 GVN：公共子表达式消除

## 8.1 EarlyCSE

EarlyCSE 是轻量早期清理，沿 dominator tree 维护作用域表达式表，消除明显重复计算和
部分重复 load。

### Before

```llvm
define i32 @cse(i32 %x, i32 %y) {
entry:
  %a = add i32 %x, %y
  %b = add i32 %x, %y
  %r = mul i32 %a, %b
  ret i32 %r
}
```

### After

```llvm
define i32 @cse(i32 %x, i32 %y) {
entry:
  %a = add i32 %x, %y
  %r = mul i32 %a, %a
  ret i32 %r
}
```

`%a` 必须支配 `%b` 的所有 use，表达式还必须具有相同 type、opcode、operands 和相关 flags。

## 8.2 GVN

GVN 为表达式或值分配 value number：若两个表达式在程序点上必然产生同一值，可以用已有
定义替换。它比简单文本匹配更全局，还要处理 dominance、PHI translation 和内存依赖。

### Load elimination Before

```llvm
define i32 @load_cse(ptr %p) {
entry:
  %a = load i32, ptr %p, align 4
  %x = add i32 %a, 1
  %b = load i32, ptr %p, align 4
  %r = add i32 %x, %b
  ret i32 %r
}
```

### After（无中间 clobber 时）

```llvm
define i32 @load_cse(ptr %p) {
entry:
  %a = load i32, ptr %p, align 4
  %x = add i32 %a, 1
  %r = add i32 %x, %a
  ret i32 %r
}
```

## 8.3 阻止 load elimination 的反例

```llvm
%a = load i32, ptr %p
store i32 9, ptr %q       ; 若 %p 与 %q MayAlias，可能 clobber
%b = load i32, ptr %p
```

只有 AA/MemorySSA 等证明 `%q` 与 `%p` NoAlias，第二个 load 才能复用 `%a`。unknown call、
volatile、atomic ordering 也必须考虑。

## 8.4 EarlyCSE 与 GVN 对照

| 维度 | EarlyCSE | GVN |
|---|---|---|
| 目标 | 快速清理明显冗余 | 更强的全局值等价 |
| 成本 | 较低 | 较高 |
| 内存 | 有限、轻量处理 | 更深入的内存依赖处理 |
| Pipeline | 较早、可反复 | 中端主要优化阶段 |

---

# 9. DCE、ADCE 与 DSE

## 9.1 DCE：删除无 use 的纯计算

### Before

```llvm
define i32 @dce(i32 %x) {
entry:
  %dead1 = add i32 %x, 10
  %dead2 = mul i32 %dead1, 20
  %live = add i32 %x, 1
  ret i32 %live
}
```

### After

```llvm
define i32 @dce(i32 %x) {
entry:
  %live = add i32 %x, 1
  ret i32 %live
}
```

删除 `%dead2` 后 `%dead1` 才变死，因此 DCE/递归删除需要 worklist。

## 9.2 无 use 不等于 dead

```llvm
%v = load volatile i32, ptr %device
call void @may_have_side_effect()
store i32 1, ptr %visible
```

三者即使返回值无 use，也可能具有可观察行为。正确 deadness 判断必须考虑 memory、throw、
convergent、atomic、volatile 等语义。

## 9.3 ADCE：从活跃根反向标记

普通 DCE 常从“明显 dead”的指令向前删；ADCE 更像：

```text
先假定大多数代码 dead
        ↓
以 ret、可观察 store/call 等为 live roots
        ↓
沿 operand 和控制依赖反向标记 live
        ↓
删除未标记指令，并简化无用控制流
```

### Before（示意）

```llvm
entry:
  br i1 %c, label %work, label %exit
work:
  %unused = call i32 @readnone_calculation()
  br label %exit
exit:
  ret i32 0
```

### After（若 call 被证明无副作用且可删除）

```llvm
entry:
  ret i32 0
```

删除控制流需要理解 control dependence，常与 post-dominance 相关。

## 9.4 DSE：删除被覆盖的 store

### Before

```llvm
define void @dse(ptr %p) {
entry:
  store i32 1, ptr %p, align 4
  store i32 2, ptr %p, align 4
  ret void
}
```

### After

```llvm
define void @dse(ptr %p) {
entry:
  store i32 2, ptr %p, align 4
  ret void
}
```

第一个 store 的全部字节在任何读取前被第二个覆盖。

### 不能删除

```llvm
store i32 1, ptr %p
%v = load i32, ptr %p     ; 观察到 1
store i32 2, ptr %p
```

以及：

```llvm
store i32 1, ptr %p
call void @unknown()      ; 可能读取 %p
store i32 2, ptr %p
```

DSE 需要 MemorySSA、AA/ModRef、写入区间、对象边界等信息；部分覆盖比完整覆盖更复杂。

---

# 10. JumpThreading 与 CorrelatedValuePropagation

## 10.1 JumpThreading

若某个 predecessor 路径已经决定后续 branch 结果，可以复制/转发控制流，让该路径直接
跳向已知目标。

### Before

```llvm
entry:
  br i1 %c, label %a, label %b
a:
  br label %test
b:
  br label %test
test:
  %p = phi i1 [ true, %a ], [ %c, %b ]
  br i1 %p, label %yes, label %no
```

### After（关键 CFG）

```text
Before: entry → a ─┐
                    ├→ test → yes/no
              → b ─┘

After:  entry → a ─────────→ yes
              → b → test.b → yes/no
```

来自 `%a` 的路径上 `%p` 必为 true，因此无需再次测试。真实变换可能复制块或调整 PHI，
是否值得做受代码尺寸和 profile 影响。

## 10.2 CorrelatedValuePropagation

利用支配路径上的 branch 条件推导值关系。

### Before

```llvm
entry:
  %iszero = icmp eq i32 %x, 0
  br i1 %iszero, label %zero, label %nonzero
nonzero:
  %again = icmp ne i32 %x, 0
  br i1 %again, label %yes, label %impossible
```

### After（在 nonzero 中 `%again` 已知为 true）

```llvm
nonzero:
  br label %yes
```

事实只在被相应 edge 支配的区域有效。仅“比较指令支配 use”不够，必须确认具体分支结果
所代表的条件支配当前路径。

---

# 11. LICM：循环不变量代码移动

## 11.1 基本提升

### Before

```llvm
preheader:
  br label %loop

loop:
  %i = phi i32 [ 0, %preheader ], [ %next, %loop ]
  %limit2 = mul i32 %limit, 2
  %cmp = icmp slt i32 %i, %limit2
  %next = add nuw i32 %i, 1
  br i1 %cmp, label %loop, label %exit

exit:
  ret i32 %i
```

### After

```llvm
preheader:
  %limit2 = mul i32 %limit, 2
  br label %loop

loop:
  %i = phi i32 [ 0, %preheader ], [ %next, %loop ]
  %cmp = icmp slt i32 %i, %limit2
  %next = add nuw i32 %i, 1
  br i1 %cmp, label %loop, label %exit
```

```text
Before cost: mul 每次迭代执行
After cost:  mul 在 preheader 只执行一次
```

合法性：`%limit` 在循环外定义，mul 无副作用，且放到 preheader 不会引入原本不存在的
observable behavior。

## 11.2 operands invariant 仍不够

```llvm
loop:
  %q = sdiv i32 %x, %y
  br i1 %execute_body, label %body, label %exit
```

即使 `%x/%y` 都 invariant，循环可能零次执行或 `%q` 原本只在受保护路径执行；hoist 会
让 `%y == 0` 时新增 UB。必须证明 safety 或 must-execute。

## 11.3 Load hoisting

```llvm
loop:
  %v = load i32, ptr %p
  ...
```

至少要证明：

1. 地址在新位置可安全解引用，提升不会新增 trap。
2. 循环中没有可能修改 `%p` 所指位置的操作。
3. 不是禁止移动的 volatile/atomic 情形。
4. 提升不跨越不允许的同步、异常、GC/JIT 语义边界。

AA 回答地址重叠，MemorySSA walker 帮助查 clobber，dereferenceability/must-execute 解决
投机安全；多个分析缺一不可。

## 11.4 Sinking

若一个循环内定义只在 exit 使用，可尝试下沉到 exit，减少未走该 exit 路径的执行。但多个
exit 可能需要克隆指令，LCSSA 让外部 use 集中在 exit PHI 附近。

## 11.5 面试追问

- preheader 为什么重要？——保证进入循环时执行一次，且不影响其他到 header 的路径。
- LICM 是否只移动算术？——也处理部分内存、控制和 promotion，但合法性更复杂。
- safepoint poll 能否当普通 readnone call hoist？——JIT 必须用正确属性/语义阻止破坏
  安全点可达性。

---

# 12. LoopRotate、IndVarSimplify 与 LSR

## 12.1 LoopRotate

把条件位于 header 的 while 形循环旋转为带 precondition 的 do-while 风格，使循环主体在
主循环路径上更直线，并便于后续 simplify、vectorize 和 profile 布局。

### Before CFG

```text
       preheader
           │
           ▼
      header: test ─false─→ exit
           │true
           ▼
          body
           │
           └──────────────→ header
```

### After CFG（示意）

```text
      preheader: test ─false─→ exit
           │true
           ▼
          body
           │
           ▼
       latch: test ─true────→ body
           │false
           ▼
          exit
```

代价是可能复制 header 中部分指令；profile/debug、PHI、LCSSA 都需正确维护。

## 12.2 IndVarSimplify

目标是把归纳变量、exit condition 和循环外使用规范到 SCEV 更易理解的形式。

### Before（多个派生 IV）

```llvm
%i = phi i32 [ 0, %pre ], [ %i.next, %latch ]
%j = phi i32 [ 10, %pre ], [ %j.next, %latch ]
%i.next = add i32 %i, 1
%j.next = add i32 %j, 2
```

### After（示意：从 canonical IV 推导）

```llvm
%i = phi i32 [ 0, %pre ], [ %i.next, %latch ]
%j = shl i32 %i, 1
%j.with.start = add i32 %j, 10
%i.next = add i32 %i, 1
```

真实结果由 use、成本、溢出和 SCEV 表达决定。重点不是“永远只剩一个 PHI”，而是规范
归纳关系并消除冗余 IV。

## 12.3 Loop Strength Reduction（LSR）

把循环中的昂贵派生表达式变成廉价递增，并结合目标 addressing mode 降低成本。

### Before

```llvm
loop:
  %offset = mul i64 %i, 4
  %addr = getelementptr i8, ptr %base, i64 %offset
  %v = load i32, ptr %addr
  %i.next = add i64 %i, 1
```

### After（概念上）

```llvm
loop:
  %ptr = phi ptr [ %base, %pre ], [ %ptr.next, %loop ]
  %v = load i32, ptr %ptr
  %ptr.next = getelementptr i8, ptr %ptr, i64 4
```

这不是单纯“乘法一定比加法慢”；后端 addressing mode、寄存器压力和多个 use 共同决定最优
形式，LSR 会使用 TTI 和 SCEV。

---

# 13. LoopUnswitch 与 LoopUnroll

## 13.1 LoopUnswitch

将循环内不随迭代变化的条件移到循环外，通过复制/版本化循环，使每个版本内部条件固定。

### Before

```text
loop N 次:
  if (invariant_cond)
    A(i)
  else
    B(i)
```

### After

```text
if (invariant_cond)
  loop N 次: A(i)
else
  loop N 次: B(i)
```

### IR/CFG 示意

```text
Before: pre → loop.header → cond → A/B → latch ↺

After:  pre → cond ─true → loop.A.header → A → latch.A ↺
                   └false→ loop.B.header → B → latch.B ↺
```

收益是循环内部少一次分支并暴露常量传播；代价是复制循环造成 code size、I-cache 和编译
时间增长。partial/non-trivial unswitch 还要处理 exits、LCSSA 和 profile。

## 13.2 LoopUnroll

### Before

```llvm
; 概念循环：for i=0..3, sum += a[i]
loop:
  %i = phi i64 [ 0, %pre ], [ %next, %loop ]
  %p = getelementptr i32, ptr %a, i64 %i
  %v = load i32, ptr %p
  %sum.next = add i32 %sum, %v
  %next = add i64 %i, 1
  %done = icmp eq i64 %next, 4
  br i1 %done, label %exit, label %loop
```

### After（完全展开的概念结果）

```llvm
%p0 = getelementptr i32, ptr %a, i64 0
%v0 = load i32, ptr %p0
%s0 = add i32 %sum, %v0
%p1 = getelementptr i32, ptr %a, i64 1
%v1 = load i32, ptr %p1
%s1 = add i32 %s0, %v1
%p2 = getelementptr i32, ptr %a, i64 2
%v2 = load i32, ptr %p2
%s2 = add i32 %s1, %v2
%p3 = getelementptr i32, ptr %a, i64 3
%v3 = load i32, ptr %p3
%s3 = add i32 %s2, %v3
```

## 13.3 收益与代价

| 收益 | 代价 |
|---|---|
| 减少 branch/IV update | 代码膨胀 |
| 暴露跨迭代 CSE/constant folding | I-cache 压力 |
| 增强 instruction-level parallelism | 寄存器压力 |
| 为 SLP/vectorization 创造模式 | 编译时间增加 |

未知 trip count 时可部分展开，并生成 epilogue/remainder loop 处理余数。完全展开一般要求
小而已知的 trip count 或 profile/pragma 强信号。

---

# 14. Loop Vectorizer 与 SLP Vectorizer

## 14.1 Loop Vectorizer：跨迭代打包

### Scalar Before

```llvm
loop:
  %i = phi i64 [ 0, %pre ], [ %next, %loop ]
  %pa = getelementptr float, ptr %a, i64 %i
  %pb = getelementptr float, ptr %b, i64 %i
  %x = load float, ptr %pa, align 4
  %y = fmul float %x, 2.000000e+00
  store float %y, ptr %pb, align 4
  %next = add i64 %i, 1
  %cmp = icmp ult i64 %next, %n
  br i1 %cmp, label %loop, label %exit
```

### Vector After（VF=4 的关键形态，示意）

```llvm
vector.body:
  %index = phi i64 [ 0, %vector.ph ], [ %index.next, %vector.body ]
  %pa = getelementptr float, ptr %a, i64 %index
  %wide = load <4 x float>, ptr %pa, align 4
  %scaled = fmul <4 x float> %wide,
                  <float 2.0, float 2.0, float 2.0, float 2.0>
  %pb = getelementptr float, ptr %b, i64 %index
  store <4 x float> %scaled, ptr %pb, align 4
  %index.next = add i64 %index, 4
```

真实输出还可能包含：

```text
vector.ph            计算 vector trip count
memcheck             runtime alias check
vector.body          宽向量主循环
middle.block         判断是否还有余数
scalar.ph            准备 scalar remainder
scalar.body          处理尾部元素
```

## 14.2 Legality 与 Profitability

**合法性**关注：

- 是否存在 loop-carried dependence；
- AA 能否证明数组访问不冲突，或能否生成 runtime check；
- 操作是否有向量形式；
- reduction、induction、异常和 memory ordering 能否合法转换。

**收益性**关注：

- TTI 给出的 vector/scalar cost；
- vectorization factor（VF）和 interleave factor（UF）；
- trip count、profile、epilogue 成本；
- register pressure、gather/scatter 和 mask 成本。

## 14.3 Reduction

```text
scalar: sum = (((0 + a0) + a1) + a2) + a3
vector: lanes = <a0,a1,a2,a3>
        sum = vector.reduce.add(lanes)
```

整数 reduction 要考虑 overflow flags；浮点 reduction 若改变结合顺序，通常需要 fast-math
许可或使用保持严格顺序的策略。

## 14.4 SLP：同一块内打包同构操作

### Before

```llvm
%a0 = load float, ptr %p0
%a1 = load float, ptr %p1
%a2 = load float, ptr %p2
%a3 = load float, ptr %p3
%r0 = fadd float %a0, %x0
%r1 = fadd float %a1, %x1
%r2 = fadd float %a2, %x2
%r3 = fadd float %a3, %x3
```

### After（示意）

```llvm
%va = load <4 x float>, ptr %p0
%vx = insertelement/shufflevector ...
%vr = fadd <4 x float> %va, %vx
```

SLP 从 stores、运算或 reduction 等 seed 构建 isomorphic operation tree，计算打包、shuffle、
extract 和 memory cost。相邻源码语句不代表一定能打包，对齐、布局和 use graph 都重要。

## 14.5 两种 Vectorizer 对照

| 维度 | Loop Vectorizer | SLP Vectorizer |
|---|---|---|
| 打包来源 | 不同 loop iteration | 同一 block/region 的多条 scalar op |
| 主要依赖 | loop structure、dependence、SCEV | isomorphic expression tree、store seeds |
| 尾部处理 | remainder、mask、epilogue | 通常无 trip remainder |
| 典型输入 | 数组循环 | 展开的标量语句、结构体字段运算 |

---

# 15. Inlining 与跨过程优化

## 15.1 Inlining

### Before

```llvm
define i32 @inc(i32 %x) {
entry:
  %r = add i32 %x, 1
  ret i32 %r
}

define i32 @caller(i32 %a) {
entry:
  %v = call i32 @inc(i32 %a)
  %r = mul i32 %v, 2
  ret i32 %r
}
```

### After

```llvm
define i32 @caller(i32 %a) {
entry:
  %v.i = add i32 %a, 1
  %r = mul i32 %v.i, 2
  ret i32 %r
}
```

## 15.2 为什么内联能产生二次收益

```text
call boundary 消失
   ├─ 实参常量进入 callee → SCCP/InstCombine
   ├─ caller/callee 内存关系可见 → AA/GVN/DSE
   ├─ 间接调用可能变直接 → further inlining
   └─ 返回值和分支可见 → SimplifyCFG/DCE
```

## 15.3 内联成本

| 正向因素 | 负向因素 |
|---|---|
| 删除 call/return 开销 | 代码尺寸增加 |
| 常量和属性传播 | I-cache 压力 |
| 暴露去虚拟化/标量化 | 编译时间增加 |
| 热调用点收益大 | 寄存器压力可能增加 |

递归、冷调用点、巨大 callee、异常 CFG、动态 alloca 等会影响决策。`alwaysinline` 是强要求，
但仍可能有结构上无法内联的硬限制；`noinline` 则禁止普通内联决策。

## 15.4 IPSCCP

跨过程稀疏常量传播可以从所有 call sites 推导函数参数/返回值事实。

### Before

```llvm
define internal i32 @f(i32 %x) {
  %r = add i32 %x, 1
  ret i32 %r
}

define i32 @g() {
  %r = call i32 @f(i32 41)
  ret i32 %r
}
```

### After（概念结果）

如果 `@f` 的全部可见调用都传 41，且 linkage/地址逃逸允许推理，可传播参数或克隆/简化，
最终 `@g` 可能直接返回 42，并让 `@f` 进一步内联或删除。

外部 linkage、函数地址逃逸、动态链接/interposition 会限制跨过程假设。

## 15.5 Dead Argument Elimination

```llvm
; Before
define internal i32 @f(i32 %used, i32 %unused) {
  ret i32 %used
}
call i32 @f(i32 7, i32 99)

; After（概念）
define internal i32 @f(i32 %used) {
  ret i32 %used
}
call i32 @f(i32 7)
```

只能在所有调用点和函数可见性允许修改 ABI 时进行；外部可调用函数不能随意改变签名。

## 15.6 GlobalOpt / GlobalDCE

- GlobalOpt 简化 global 初始化、可见性允许的 global 状态和调用关系。
- GlobalDCE 从外部可达符号根出发，删除不可达函数和 global。
- linkage、comdat、used/compiler.used、动态链接和反射式地址使用都影响可删除性。

---

# 16. TailCallElim 与其他常见 Pass

## 16.1 TailCallElim

### Tail recursion Before

```llvm
define i32 @sum(i32 %n, i32 %acc) {
entry:
  %done = icmp eq i32 %n, 0
  br i1 %done, label %ret, label %recur
ret:
  ret i32 %acc
recur:
  %n1 = sub i32 %n, 1
  %acc1 = add i32 %acc, %n
  %r = call i32 @sum(i32 %n1, i32 %acc1)
  ret i32 %r
}
```

### After（循环化示意）

```llvm
entry:
  br label %loop
loop:
  %n.cur = phi i32 [ %n, %entry ], [ %n1, %recur ]
  %acc.cur = phi i32 [ %acc, %entry ], [ %acc1, %recur ]
  %done = icmp eq i32 %n.cur, 0
  br i1 %done, label %ret, label %recur
ret:
  ret i32 %acc.cur
recur:
  %n1 = sub i32 %n.cur, 1
  %acc1 = add i32 %acc.cur, %n.cur
  br label %loop
```

一般 tail call 优化还受 calling convention、ABI、stack layout、varargs、返回值和平台支持
限制。IR 的 `tail` 是允许/期望，`musttail` 具有更强的验证和语义约束。

## 16.2 LowerSwitch

将 `switch` 降成 branch tree、jump table 或其他目标适合形式。纯 IR 层 lower-switch 与
后端真正选择 jump table/bit test 不是完全同一个决策点；case 密度、范围、profile 和目标
成本会影响结果。

## 16.3 Attributor

Attributor 用抽象属性框架跨函数推导 `nonnull`、`nocapture`、memory behavior、willreturn
等事实，并让属性之间迭代到不动点。推导出的 attribute 是可依赖的语义事实，必须建立在
可见性、调用图和 IR 语义上。

## 16.4 MergedLoadStoreMotion / GVN Hoist/Sink

这些变换在 diamond 或公共控制流区域提升/下沉相似内存操作，减少重复，但必须处理 alias、
speculation、异常、PHI/select 和写入可观察顺序。面试时不要只回答“代码一样就移动”。

---

# 17. Pass 之间如何形成优化链

## 17.1 示例：一个函数如何逐步变成常量

### 初始 IR

```llvm
define i32 @pipeline() {
entry:
  %slot = alloca i32
  store i32 40, ptr %slot
  %x = load i32, ptr %slot
  %a = add i32 %x, 2
  %c = icmp eq i32 %a, 42
  br i1 %c, label %yes, label %no
yes:
  %r = mul i32 %a, 2
  ret i32 %r
no:
  ret i32 0
}
```

### 第一步：mem2reg

```llvm
%a = add i32 40, 2
%c = icmp eq i32 %a, 42
br i1 %c, label %yes, label %no
```

### 第二步：InstCombine / SCCP

```llvm
br i1 true, label %yes, label %no
yes:
  ret i32 84
```

### 第三步：SimplifyCFG

```llvm
define i32 @pipeline() {
entry:
  ret i32 84
}
```

一个 pass 往往只负责一类事实：mem2reg 暴露 SSA，SCCP 发现常量与可执行边，SimplifyCFG
清理 CFG。模块化能控制复杂度，也方便复用分析。

## 17.2 示例：循环优化协作

```text
LoopSimplify + LCSSA
        ↓ 建立 preheader、dedicated exits
LoopRotate
        ↓ 形成适合分析的主循环
LICM
        ↓ 移走 invariant，简化 loop body
IndVarSimplify + LSR
        ↓ 规范 IV/地址演化
LoopVectorize 或 Unroll
        ↓ 批量处理迭代
InstCombine + DCE + SimplifyCFG
        ↓ 清理变换遗留冗余
```

## 17.3 O1/O2/O3/Os/Oz 的方向

| 级别 | 大致目标 |
|---|---|
| O0 | 保留可调试性与低编译成本，通常不跑主要优化 pipeline |
| O1 | 较低编译成本下做基础优化 |
| O2 | 综合运行性能、代码尺寸和编译时间的常用平衡 |
| O3 | 更积极的循环、向量化、内联等性能优化，可能增大代码 |
| Os | 在优化性能的同时更重视代码尺寸 |
| Oz | 更激进地最小化代码尺寸 |

具体 pipeline 会随 LLVM 版本变化，面试时应回答设计方向，并用
`-print-pipeline-passes` 查看当前版本，而不是死背某个旧版本的 pass 顺序。

## 17.4 为什么 Pass 顺序会导致结果不同

- Inlining 太早：暴露机会，但代码量上升、分析成本增加。
- Inlining 太晚：错过跨调用边界常量传播。
- Unroll 前 LICM：先把循环体变小，降低展开膨胀。
- Unroll 后 InstCombine：清理每份复制体暴露的常量。
- Vectorize 前 canonicalize：让 induction、reduction、memory pattern 易识别。
- DCE/SimplifyCFG 穿插：控制 IR 规模，避免后续分析浪费时间。

---

# 18. 面试速查表与练习题

## 18.1 Pass 速查表

| Pass | 核心输入特征 | 核心输出 | 关键分析/条件 |
|---|---|---|---|
| mem2reg | promotable scalar alloca | SSA + PHI | DT、dominance frontier |
| SROA | aggregate alloca/slices | scalar partitions | DataLayout、escape/use |
| InstCombine | 非规范局部表达式 | canonical IR | KnownBits、DL、poison 规则 |
| Reassociate | 可结合运算树 | 统一顺序/常量聚集 | flags、浮点语义 |
| SimplifyCFG | 常量/冗余分支和块 | 更简单 CFG | TTI、profile、speculation |
| SCCP | SSA + 条件 CFG | 常量、不可执行边 | value lattice、worklist |
| EarlyCSE | 支配域重复表达式 | 复用已有值 | DT、轻量内存分析 |
| GVN | 全局等价表达式/load | value replacement | DT、AA/内存依赖 |
| DCE | dead pure instructions | 删除死 def 链 | side-effect 判断 |
| ADCE | 大范围无可观察代码 | 删除值和控制流 | control dependence/PDT |
| DSE | 被覆盖 store | 删除旧写入 | MemorySSA、AA、写区间 |
| JumpThreading | 路径上条件已知 | edge/threaded CFG | LVI、profile、DT |
| LICM | loop invariant op | hoist/sink | LI、DT、AA、MSSA、safety |
| LoopRotate | header-tested loop | latch-tested canonical loop | LI、DT、LCSSA |
| IndVarSimplify | 多个/复杂 IV | canonical IV 和 exit | SCEV、LI |
| LSR | 循环内昂贵地址/乘法 | 增量 recurrence | SCEV、TTI |
| LoopUnswitch | invariant branch | 版本化循环 | LI、DT、成本模型 |
| LoopUnroll | 小/热循环 | 复制迭代体 | SCEV、BFI、TTI |
| LoopVectorize | 跨迭代同构操作 | vector loop | dependence、AA、SCEV、TTI |
| SLPVectorize | 同区域同构 scalar tree | vector operation tree | TTI、memory layout |
| Inliner | call edge | cloned callee CFG | cost、profile、call graph |
| IPSCCP | 跨调用常量事实 | 参数/返回传播 | linkage、call graph |
| TailCallElim | tail recursion/call | loop 或 tail marker | ABI、calling convention |

## 18.2 高频“为什么没优化”排查表

| 症状 | 可能原因 |
|---|---|
| alloca 没被 mem2reg | 地址逃逸、非 entry alloca、volatile、复杂 use |
| load 没被 CSE/hoist | MayAlias clobber、unknown call、atomic/volatile、可能 trap |
| 循环没向量化 | dependence、未知 trip、成本不划算、unsupported op、无 fast-math |
| 函数没内联 | cost 太高、cold、noinline、递归、结构/ABI 限制 |
| 分支没转 select | 两侧不可安全投机、成本高、profile 强偏向 |
| 循环没展开 | trip count 未知、body 太大、size level、收益不足 |
| store 没被 DSE | 中间可能读取、只部分覆盖、地址 MayAlias、volatile/atomic |
| 算术没折叠 | poison/overflow flags、类型不匹配、非 canonical form |

## 18.3 回答任意 Pass 的通用模板

> 这个 Pass 的目标是 **消除某种运行时成本或建立规范形式**。它识别 **某种 IR/CFG
> 模式**，并在 **dominance、memory、poison/UB、异常和目标约束** 满足时变换为 **目标
> IR 形态**。合法性依赖 **相关分析**，收益由 **TTI/profile/size cost** 判断。变换后要
> 更新或失效 **DT/LI/SCEV/MSSA/BPI/BFI 等分析**，并通过 `-verify-each` 和正反例验证。

## 18.4 练习题

1. 给出一个 operands 都 invariant、但不能 LICM hoist 的整数指令例子。
2. 给出一个第二次 load 不能复用第一次 load 的最小 IR。
3. 手画 `if-then-else` 经 mem2reg 后 PHI 的 incoming edge。
4. 解释为什么 LoopUnswitch 通常需要成本模型。
5. 写出一个 SCCP 能证明 PHI 为常量、普通合并所有 incoming 却不能的 CFG。
6. 比较“先 inline 再 SCCP”和“先 SCCP 再 inline”可能产生的结果。
7. 给出普通 `add` 不能随意添加 `nsw` 的 Java 例子。
8. 解释 vector legality 与 profitability 各自失败时诊断重点。
9. 设计 DSE 的三个负例：中间 load、unknown call、partial overwrite。
10. 一个自定义 CFG Pass 返回 `PreservedAnalyses::all()` 有什么风险？

## 18.5 实验建议

对每个 Pass 准备两个 `.ll`：

```text
positive.ll       应当发生变换
negative.ll       只差一个关键条件，必须拒绝变换
```

运行：

```bash
OPT=jeandle-llvm/build-release/bin/opt

$OPT -S -passes='pass-name' positive.ll -o - | less
$OPT -S -passes='pass-name' negative.ll -o - | less
$OPT -passes='pass-name,verify' positive.ll -disable-output
$OPT -passes='default<O2>' -debug-pass-manager positive.ll -disable-output
```

真正掌握一个优化的标志不是能背出 After IR，而是能构造一个只改变一项条件、从“允许优化”
变成“必须拒绝”的反例。

---

## 附录：与前两本手册的配合

```text
llvm-common-api-handbook.zh-CN.md
    解决：C++ 接口怎样调用

llvm-interview-handbook.zh-CN.md
    解决：面试问题怎样回答、怎样应对追问

llvm-optimization-passes-before-after.zh-CN.md
    解决：优化前后 IR 为什么变化、何时不能变化
```

推荐顺序：先读本文的 mem2reg、SCCP、GVN、LICM、Vectorization 五章，再回到接口手册查
DT、LI、SCEV、AA、MemorySSA 的具体 API，最后使用面试手册的追问链进行口述训练。
