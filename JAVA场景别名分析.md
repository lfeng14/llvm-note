#### 前言

1. 从Java语义看，所有heap对象最终都由`new`或`newarray`分配产生，但编译器当前处理的通常是对象引用，不一定能看到其原始分配点。

2. 如果两个引用可以在当前IR中分别追溯到两次独立的`new`/`newarray`，就能证明它们指向不同对象，访问相同field或array element也可以返回`NoAlias`。

3. 引用经过方法参数、field load、`phi/select`合流或未inline调用返回后，分配来源可能无法继续追踪。例如`f(obj, obj)`中两个参数是不同SSA值，但实际指向同一对象。

4. 因此别名分析只在能够建立完整provenance证明时返回`NoAlias`；来源不可见或分析复杂度超出边界时，必须保守返回`MayAlias`，以保证正确性。

当前实现分工如下：structured field/array和`gc.relocate`由`JavaHeapAA`直接分析；fresh allocation由frontend在`new`/`newarray`返回值上附加LLVM`noalias`，再由BasicAA等标准AA消费；safepoint和GC barrier属于`getModRefInfo`路径，回答调用是否读写指定`MemoryLocation`。

```
void f(Node a, Node b) { ... }       // 外部传入
Node x = holder.child;               // 从field读取
Node y = cond ? p : q;               // phi/select合流
Node z = getNode();                  // 未inline调用返回
```
#### Object field场景
**一句话总结：**`Root`是指向Java对象的引用值，不同SSA`Root`仍可能指向同一对象，因此当前仅凭root不同不判`NoAlias`，无法证明时保持保守。

针对两个Java object field访问，当前流程可以理解为：

```text
        ┌──────────────────────────────┐
        │ 输入 LocA、LocB              │
        └──────────────┬───────────────┘
                       │
                       v
        ┌──────────────────────────────┐
        │ 提取JavaHeapAccess            │
        │ Root、Field.Offset、Array等   │
        └──────────────┬───────────────┘
                       │
          Root/结构化信息完整？
                 ┌─────┴─────┐
                否           是
                │            │
                v            v
          ┌──────────┐   ┌──────────────────────┐
          │ MayAlias │   │ field范围是否重叠？ │
          └──────────┘   └──────────┬───────────┘
                                    │
                              否    │    是
                              │     │
                              v     v
                       ┌─────────┐  ┌────────────────┐
                       │ NoAlias │  │ RootA == RootB? │
                       └─────────┘  └───────┬────────┘
                                            │
                                      是    │    否
                                      │     │
                                      v     v
                               ┌──────────┐ ┌────────────────────┐
                               │ MayAlias │ │ 比较JavaType/klass │
                               └──────────┘ └─────────┬──────────┘
                                                       │
                                      类型是否可能相同？
                                             ┌─────────┴─────────┐
                                            否                    是
                                            │                     │
                                            v                     v
                                     ┌──────────┐           ┌──────────┐
                                     │ NoAlias  │           │ MayAlias │
                                     └──────────┘           └──────────┘
```

对应几种典型情况：

| 场景 | 处理 |
|---|---|
| 同一个root、不同field | 比较offset/range，不重叠则`NoAlias` |
| 不同root、不同field | 仍可比较相对range；不重叠则`NoAlias` |
| 不同root、相同field | 不能仅凭root不同判定，需要klass/provenance |
| 相同类、相同field | 可能是同一对象，`MayAlias` |
| 不同且不兼容的类 | 不可能是同一对象，`NoAlias` |
| 不同SSA值但类型相同 | 仍可能指向同一对象，`MayAlias` |

当前代码的实际判断顺序是：

```cpp
if (!AccessA.Root || !AccessB.Root)
  return AliasResult::MayAlias;

if (areDisjointStructuredFields(...))
  return AliasResult::NoAlias;

if (AccessA.Root == AccessB.Root)
  return AliasResult::MayAlias;

if (!areJavaTypesIncompatible(TypeA, TypeB))
  return AliasResult::MayAlias;

return AliasResult::NoAlias;
```

---

#### Array element场景

**一句话总结：**`Root`是指向Java数组对象的引用值，只有常量下标能提供可信的element offset；动态下标、范围不完整或无法证明不重叠时保持`MayAlias`。

针对两个Java array element访问：

```text
        ┌──────────────────────────────┐
        │ 输入 LocA、LocB              │
        └──────────────┬───────────────┘
                       │
                       v
        ┌──────────────────────────────┐
        │ 提取JavaHeapAccess            │
        │ Root、ArrayElement.Offset等  │
        └──────────────┬───────────────┘
                       │
             Root/element信息完整？
                 ┌─────┴─────┐
                否           是
                │            │
                v            v
          ┌──────────┐   ┌────────────────────────┐
          │ MayAlias │   │ element访问范围重叠？ │
          └──────────┘   └──────────┬─────────────┘
                                    │
                              否    │    是
                              │     │
                              v     v
                       ┌─────────┐  ┌────────────────┐
                       │ NoAlias │  │ RootA == RootB? │
                       └─────────┘  └───────┬────────┘
                                            │
                                      是    │    否
                                      │     │
                                      v     v
                               ┌──────────┐ ┌────────────────────┐
                               │ MayAlias │ │ 比较JavaType/klass │
                               └──────────┘ └─────────┬──────────┘
                                                       │
                                      类型是否可能相同？
                                             ┌─────────┴─────────┐
                                            否                    是
                                            │                     │
                                            v                     v
                                     ┌──────────┐           ┌──────────┐
                                     │ NoAlias  │           │ MayAlias │
                                     └──────────┘           └──────────┘
```

array element范围表示为：

```text
A = [ArrayOffsetA, ArrayOffsetA + AccessSizeA)
B = [ArrayOffsetB, ArrayOffsetB + AccessSizeB)
```

如果：

```cpp
ArrayOffsetA + AccessSizeA <= ArrayOffsetB ||
ArrayOffsetB + AccessSizeB <= ArrayOffsetA
```

则两个访问范围不重叠，可以返回`NoAlias`。

例如：

```java
int x = array[0]; // [16, 20)
array[1] = 1;     // [20, 24)
```

即使两个访问可能属于同一个数组，范围也不重叠。

对应情况：

| 场景 | 处理 |
|---|---|
| 同一个root、不同常量element | 比较offset/range，不重叠则`NoAlias` |
| 不同root、不同常量element | 仍可比较相对range，不重叠则`NoAlias` |
| 同一个root、相同element | 范围重叠，`MayAlias` |
| 不同root、相同element | 不能仅凭root不同判断，需要klass/provenance |
| 动态array index | 没有精确offset，`MayAlias` |
| 宽load跨越多个element | 范围重叠或无法证明，`MayAlias` |
| 不同且不兼容的array klass | 不可能是同一数组对象，`NoAlias` |
| 不同SSA值但可能指向同一数组 | 不能仅凭SSA值不同判断，`MayAlias` |

当前代码实际判断顺序：

```cpp
if (!AccessA.Root || !AccessB.Root)
  return AliasResult::MayAlias;

if (EnableJavaArrayAA &&
    areDisjointArrayElements(AccessA, LocA.Size,
                              AccessB, LocB.Size)) {
  return AliasResult::NoAlias;
}

if (AccessA.Root == AccessB.Root)
  return AliasResult::MayAlias;

if (!areJavaTypesIncompatible(TypeA, TypeB))
  return AliasResult::MayAlias;

return AliasResult::NoAlias;
```

其中`ArrayElement.Offset`是element相对于数组对象`Root`的偏移；它不是数组引用自身的地址。

---

#### Fresh allocation场景

**一句话总结：**`new`/`newarray`返回值带有LLVM`noalias`属性，表示该引用指向本次独立分配的Java对象；只要两个访问分别基于不同allocation，就无需比较field offset，直接返回`NoAlias`。

示例：

```java
Node a = new Node();
Node b = new Node();

a.value = 1;
b.value = 2;
```

流程：

```text
        ┌──────────────────────────────┐
        │ 输入LocA、LocB               │
        └──────────────┬───────────────┘
                       │
                       v
        ┌──────────────────────────────┐
        │ 提取两个访问的Root            │
        │ 及其allocation provenance     │
        └──────────────┬───────────────┘
                       │
              provenance信息完整？
                 ┌─────┴─────┐
                是           否
                │            │
                v            v
        ┌────────────────┐ ┌────────────────────┐
        │ 是否来自两次   │ │ 结构化field/array  │
        │ 独立fresh分配？ │ │ 信息是否完整？     │
        └───────┬────────┘ └─────────┬──────────┘
                │                    │
           是   │   否          是   │   否
                │                │   │
                v                v   v
          ┌──────────┐  ┌────────────────────┐
          │ NoAlias  │  │ range是否不重叠？   │
          └──────────┘  └─────────┬──────────┘
                                  │
                             是   │   否
                                  │
                                  v
                           ┌──────────┐
                           │ NoAlias  │
                           └──────────┘
                                  │
                                  v
                         ┌────────────────────┐
                         │ klass/provenance   │
                         │ 仍能证明不别名？   │
                         └─────────┬──────────┘
                                   │
                              是   │   否
                                   │
                                   v
                            ┌──────────┐ ┌──────────┐
                            │ NoAlias  │ │ MayAlias │
                            └──────────┘ └──────────┘
```

LLVM IR中，frontend在allocation返回值上添加`noalias`：

```cpp
// new实例返回一个新的Java对象。
if (llvm::jeandle::isJavaAllocationAAEnabled())
  new_inst->addRetAttr(llvm::Attribute::NoAlias);
```

```cpp
// newarray返回一个新的Java数组对象。
if (llvm::jeandle::isJavaAllocationAAEnabled())
  result->addRetAttr(llvm::Attribute::NoAlias);
```

后续field访问只是从fresh object派生：

```llvm
%a = invoke noalias ptr addrspace(1) @jeandle.new_instance(...)
%b = invoke noalias ptr addrspace(1) @jeandle.new_instance(...)

%a.value = getelementptr inbounds i8,
           ptr addrspace(1) %a, i64 12

%b.value = getelementptr inbounds i8,
           ptr addrspace(1) %b, i64 12
```

虽然两个field的offset相同：

```text
a.value: objectA + 12
b.value: objectB + 12
```

但`objectA`和`objectB`来自两次独立allocation，因此：

```text
a.value 与 b.value => NoAlias
```

对应情况：

| 场景 | 处理 |
|---|---|
| 两次独立`new`访问相同field | `NoAlias` |
| 两次独立`newarray`访问相同element | `NoAlias` |
| 同一个fresh object的不同field | 继续比较offset/range |
| fresh object与普通参数对象 | LLVM`noalias`合同下可尝试判`NoAlias` |
| `b = a`后的两个引用 | 实际是同一对象，不能判`NoAlias` |
| 未inline函数返回新对象但无`noalias` summary | `MayAlias` |
| allocation来源经过无法解析的`phi/select/call` | `MayAlias` |
| `noalias`属性被关闭或未保留 | 回退普通AA，可能无法命中 |

当前实现中，fresh allocation主要由frontend添加LLVM`noalias`，再由LLVM默认AA消费；`JavaHeapAA::alias()`本身没有直接追溯`new`调用的独立分支。

---


#### `gc.relocate` provenance场景

**一句话总结：**`gc.relocate`可能改变对象的物理地址，但不改变Java对象身份；AA向relocate前的`derived pointer`回溯，继续保留原来的`Root`和field/array offset信息。

典型IR：

`statepoint`可能触发moving GC，GC期间对象被搬迁，因此原来的`%obj`可能变成旧地址；statepoint之后必须使用`gc.relocate`得到更新后的引用。

```llvm
; GC安全点，可能触发对象搬迁。
%statepoint = call token @llvm.experimental.gc.statepoint(...)

; 根据statepoint中的指针编号，取得搬迁后的对象引用。
; 这里的0、1是statepoint的base/derived索引，不是字节offset。
%relocated = call ptr addrspace(1)
    @llvm.experimental.gc.relocate.p1(%statepoint, i32 0, i32 1)

; 使用更新后的引用访问field。
%field = getelementptr inbounds i8,
    ptr addrspace(1) %relocated, i64 12
```

`gc.relocate`不是重新分配对象，而是获取**同一个Java对象的新地址**。因此AA需要把`%relocated`追溯回原来的对象root，避免GC搬迁后丢失别名关系。

流程：

```text
        ┌──────────────────────────────┐
        │ 输入LocA、LocB               │
        └──────────────┬───────────────┘
                       │
                       v
        ┌──────────────────────────────┐
        │ 回溯GEP、bitcast、addrspacecast │
        │ 和gc.relocate                 │
        └──────────────┬───────────────┘
                       │
              标准gc.relocate可追踪？
                 ┌─────┴─────┐
                是           否
                │            │
                v            v
        ┌────────────────┐ ┌────────────────────┐
        │ 取derived ptr   │ │ 按普通root继续分析 │
        │ 继续向前回溯    │ └─────────┬──────────┘
        └───────┬────────┘           │
                │                    v
                └──────────────┐ ┌──────────┐
                               │ │ MayAlias │
                               │ └──────────┘
                               v
                    ┌──────────────────────┐
                    │ Root和offset信息完整？ │
                    └──────────┬───────────┘
                               │
                          是   │   否
                               │
                               v
                    ┌──────────────────────┐
                    │ field/element范围是否 │
                    │ 不重叠？              │
                    └──────────┬───────────┘
                               │
                          是   │   否
                               │
                               v
                        ┌──────────┐ ┌──────────┐
                        │ NoAlias  │ │ MayAlias │
                        └──────────┘ └──────────┘
```

当前代码核心逻辑：

```cpp
// gc.relocate只改变物理地址，不改变Java对象身份。
// 使用relocate对应的derived pointer继续回溯。
if (EnableGCRelocateAA) {
  if (auto *Relocate = dyn_cast<GCRelocateInst>(Ptr)) {
    Ptr = Relocate->getDerivedPtr();
    continue;
  }
}
```

回溯完成后，仍使用原来的Java heap root和累计offset：

```cpp
if (Access.Field.isValid())
  Access.Field.Offset = CumulativeGEPOffset;

if (Access.ArrayElement.isValid())
  Access.ArrayElement.Offset = CumulativeGEPOffset;

Access.Root = Root;
```

对应场景：

| 场景 | 处理 |
|---|---|
| relocation前后访问同一个field | 保留同一对象身份，通常为`MayAlias` |
| relocation前后访问不同field | 继续比较field range，不重叠则`NoAlias` |
| relocation前后访问不同常量array element | 继续比较element range，不重叠则`NoAlias` |
| relocation结果经过`bitcast`或`GEP` | 继续剥离并保留provenance |
| 普通函数伪装成relocate | 不特殊处理，保持`MayAlias` |
| 非`inbounds GEP` | 不继承Java对象provenance，保持保守 |
| relocation链超过追踪深度或来源不完整 | `MayAlias` |

关键点是：

```text
gc.relocate不提供“对象不同”证明，
只负责让同一个Java对象在GC搬迁前后的身份保持可追踪。
```

---

#### Primitive field跨safepoint场景

**一句话总结：** frontend为primitive field附加`safepoint-invariant`和field size metadata，AA确认查询范围完整落在该field内后，对`safepoint_handler`返回`NoModRef`，使LICM将循环内load移出。

```text
之前：比较两个MemoryLocation是否别名
      alias(LocA, LocB) -> NoAlias/MayAlias

当前：判断safepoint调用是否影响某个MemoryLocation
      getModRefInfo(Call, Loc) -> NoModRef/ModRef
```

当前并不是证明两个field不重叠，而是证明：

```text
safepoint_handler可能处理其他内存，
但不会修改这个已标记的primitive field。
```

因此主要消费者也不同：

```text
NoAlias    -> GVN/DSE
NoModRef   -> LICM跨safepoint hoist load
```

示例：

```java
int sum(Cell obj, int n) {
    int result = 0;
    for (int i = 0; i < n; i++) {
        result += obj.value;
        safepoint();
    }
    return result;
}
```

流程：

```text
        ┌──────────────────────────────┐
        │ 输入safepoint call和MemoryLocation │
        └──────────────┬───────────────┘
                       │
                       v
        ┌──────────────────────────────┐
        │ 判断调用目标是否为             │
        │ safepoint_handler             │
        └──────────────┬───────────────┘
                       │
                 是    │    否
                       │
                       v
        ┌──────────────────────────────┐
        │ safepoint field AA开关已开启？ │
        └──────────────┬───────────────┘
                       │
                 是    │    否
                       │
                       v
        ┌──────────────────────────────┐
        │ Java field provenance完整？   │
        │ Root、offset、field size存在？ │
        └──────────────┬───────────────┘
                       │
                 是    │    否
                       │
                       v
        ┌──────────────────────────────┐
        │ field是否为primitive且标记为   │
        │ safepoint-invariant？         │
        └──────────────┬───────────────┘
                       │
                 是    │    否
                       │
                       v
        ┌──────────────────────────────┐
        │ 查询范围是否固定且完全位于     │
        │ primitive field范围内？       │
        └──────────────┬───────────────┘
                       │
                 是    │    否
                       │
                       v
                 ┌──────────┐
                 │ NoModRef │
                 └────┬─────┘
                      │
                      v
             LICM可hoist field load
```

核心代码：

```cpp
// 只有safepoint field AA和Java field AA同时开启时，
// 才对primitive field进行特殊处理。
const bool FieldSafepointAAEnabled =
    EnableSafepointFieldAA && EnableJavaFieldAA;

const Function *Callee = Call->getCalledFunction();
if (!Callee || Callee->getName() != "safepoint_handler")
  return AAResultBase::getModRefInfo(Call, Loc, AAQI);

JavaHeapAccess Access = getJavaHeapAccess(Loc.Ptr);

// 要求：
// 1. 有safepoint-invariant标记；
// 2. 有精确field范围；
// 3. 本次查询是固定大小；
// 4. 查询范围完全落在该primitive field内。
bool PrimitiveField =
    FieldSafepointAAEnabled &&
    Access.SafepointInvariant &&
    isAccessWithinProvenanceRange(Access.Field, Loc.Size);

if (Access.Root && PrimitiveField)
  return ModRefInfo::NoModRef;

return AAResultBase::getModRefInfo(Call, Loc, AAQI);
```

对应场景：

| 场景 | 处理 |
|---|---|
| primitive field跨`safepoint_handler` | `NoModRef`，允许LICM hoist |
| reference field跨safepoint | 保持默认ModRef |
| field范围内的精确`i32` load | `NoModRef` |
| 从field起点进行`i64`/vector宽load | 可能越界，保持保守 |
| 外层GEP改变了field地址 | 清除invariant，保持保守 |
| 动态或未知大小访问 | 保持保守 |
| Unsafe、VarHandle、JNI地址 | 无可信metadata，保持保守 |
| safepoint之后primitive field被显式修改 | 由其他内存访问和依赖关系处理，不能仅凭该标记删除真实修改 |

关键区别：

```text
primitive field的数值不会因GC搬迁而改变；
reference field的值可能因对象搬迁而被GC更新。
```

**LICM hoist修改点**，但当前修改`LICM.cpp`是为了配合AA：

1. **保留Java provenance metadata**

LLVM原有LICM在hoist后可能执行：

```cpp
I.dropUBImplyingAttrsAndMetadata();
```

这会丢掉`java-field-offset`、`java-field-size`、`java-field-safepoint-invariant`等信息，后续`JavaHeapAA`就无法识别该访问是primitive field。

因此增加了保留逻辑：

```cpp
I.dropUBImplyingAttrsAndMetadata({
    JavaFieldOffsetKind,
    JavaFieldSafepointInvariantKind,
    JavaFieldSizeKind,
    JavaArrayElementOffsetKind,
    JavaArrayElementSizeKind,
    JavaArrayElementPrimitiveKind
});
```

2. **记录实际消费情况**

```cpp
++NumJavaFieldLoadsHoistedAcrossSafepoints;
jeandle::recordJavaFieldLoadHoistedAcrossSafepoint(...);
```

这只是统计/诊断，用来证明LICM确实跨safepoint hoist了Java field load，不参与正确性判断。

3. **真正的安全判断仍由AA完成**

```text
JavaHeapAA::getModRefInfo(...)
    -> NoModRef
LICM现有逻辑
    -> 判断load可移动并执行hoist
```

所以：

```text
AA负责证明能不能移动；
LICM改动负责保留provenance并记录hoist结果。
```
---

#### Primitive array element跨safepoint场景

**一句话总结：** frontend为可精确定位的primitive array element附加element offset、size和primitive metadata；AA确认查询范围完全位于该元素内后，对`safepoint_handler`返回`NoModRef`，现有LICM即可将循环内load移出。

示例：

```java
int sum(int[] values, int n) {
    int result = 0;
    for (int i = 0; i < n; i++) {
        result += values[0];
        // 循环回边可能包含safepoint
    }
    return result;
}
```

对应IR大致为：

```llvm
%element = getelementptr inbounds i8,
    ptr addrspace(1) %array, i64 16,
    !java-array-element-offset !0,
    !java-array-element-size !1,
    !java-array-element-primitive !2

%value = load atomic i32, ptr addrspace(1) %element
call void @safepoint_handler(ptr null)
```

流程：

```text
        ┌──────────────────────────────┐
        │ 输入safepoint call和Loc       │
        └──────────────┬───────────────┘
                       │
                       v
        ┌──────────────────────────────┐
        │ 调用目标是safepoint_handler？ │
        └──────────────┬───────────────┘
                       │
                 是    │    否
                       │     └──────────> 默认ModRef
                       v
        ┌──────────────────────────────┐
        │ array safepoint AA和          │
        │ Java array AA均已开启？       │
        └──────────────┬───────────────┘
                       │
                 是    │    否
                       │     └──────────> 默认ModRef
                       v
        ┌──────────────────────────────┐
        │ array element provenance完整？│
        │ Root、offset、size均存在？    │
        └──────────────┬───────────────┘
                       │
                 是    │    否
                       │     └──────────> 默认ModRef
                       v
        ┌──────────────────────────────┐
        │ element带有primitive标记？    │
        └──────────────┬───────────────┘
                       │
                 是    │    否
                       │     └──────────> 默认ModRef
                       v
        ┌──────────────────────────────┐
        │ 查询大小固定且访问范围完全位于 │
        │ 该array element范围内？       │
        └──────────────┬───────────────┘
                       │
                 是    │    否
                       │     └──────────> 默认ModRef
                       v
                ┌───────────┐
                │ NoModRef  │
                └─────┬─────┘
                      │
                      v
          地址和值满足循环不变等条件？
                      │
                      v
             LICM自动hoist load
```

核心代码：

```cpp
const bool ArraySafepointAAEnabled =
    EnableSafepointArrayAA && EnableJavaArrayAA;

JavaHeapAccess Access = getJavaHeapAccess(Loc.Ptr);

// 要求：
// 1. 这是可信的primitive array element；
// 2. element offset和size完整；
// 3. Loc.Size固定且精确；
// 4. 整个访问没有越过当前element。
bool PrimitiveElement =
    ArraySafepointAAEnabled &&
    Access.PrimitiveArrayElement &&
    isAccessWithinProvenanceRange(Access.ArrayElement, Loc.Size);

if (Access.Root && PrimitiveElement) {
  recordSafepointArrayNoModRef(*Call->getFunction());
  return ModRefInfo::NoModRef;
}

return AAResultBase::getModRefInfo(Call, Loc, AAQI);
```

primitive标记只有在element provenance可信时才保留：

```cpp
Access.PrimitiveArrayElement =
    Facts.OuterOffset == 0 &&
    Facts.ArrayElementSize &&
    *Facts.ArrayElementSize != 0 &&
    Facts.PrimitiveArrayElement;
```

对应场景：

| 场景 | 处理 |
|---|---|
| `int[]`常量元素跨safepoint | `NoModRef`，允许LICM hoist |
| `long[]`等primitive元素且范围精确 | `NoModRef` |
| `Object[]`reference元素 | 默认ModRef |
| 动态下标且没有精确offset | 默认ModRef |
| 宽load跨越相邻元素 | 默认ModRef |
| element外层存在非零GEP | provenance失效，保持保守 |
| Unsafe、VarHandle或JNI地址 | 无可信metadata，保持保守 |
| 未知或scalable访问大小 | 默认ModRef |

关键原因：

```text
GC可能更新reference array中的对象引用，
但不会改变int[]、long[]等primitive array element的数值。
```

`LICM.cpp`的联动仍只负责保留array provenance metadata和记录hoist结果；是否允许跨safepoint移动由`JavaHeapAA`的`NoModRef`结论决定。

---

#### GC barrier Mod/Ref场景

**一句话总结：** pre-barrier只读取被覆盖的reference slot，post-barrier只更新card table/TLS而不访问Java heap payload；AA根据barrier的真实内存行为，对指定`MemoryLocation`精确返回`Ref`或`NoModRef`。

当前实现已经将barrier Mod/Ref与`safepoint-invariant`解耦：barrier是否可信由runtime wrapper的函数名、HotSpot calling convention、函数attribute和签名共同决定，不要求查询field带`safepoint-invariant`。

示例：

```java
class Cell {
    int value;
    Object reference;
}

int test(Cell obj, Object newValue) {
    int first = obj.value;
    obj.reference = newValue; // 产生pre/post GC barrier
    return first + obj.value;
}
```

reference store大致lowering为：

```text
pre_barrier(&obj.reference)  // 读取旧reference
store newValue -> obj.reference
post_barrier(&obj.reference) // 更新card table/TLS queue
```

流程：

```text
        ┌──────────────────────────────┐
        │ 输入barrier Call和查询Loc     │
        └──────────────┬───────────────┘
                       │
                       v
        ┌──────────────────────────────┐
        │ 是签名、calling convention和 │
        │ attribute均匹配的可信barrier？│
        └──────────────┬───────────────┘
                       │
                 是    │    否
                       │     └──────────> 默认ModRef
                       v
        ┌──────────────────────────────┐
        │ 查询Loc属于Java heap？        │
        └──────────────┬───────────────┘
                       │
                 是    │    否
                       │     └──────────> 默认ModRef
                       v
              ┌─────────────────┐
              │ barrier类型？    │
              └───────┬─────────┘
                 pre  │  post
          ┌───────────┘   └────────────┐
          v                            v
┌──────────────────────┐      ┌─────────────────────┐
│ 用参数0和reference大小│      │ 不读取Java heap     │
│ 构造实际slot Loc      │      │ payload             │
└──────────┬───────────┘      └──────────┬──────────┘
           │                             │
           v                             v
┌──────────────────────┐           ┌──────────┐
│ query Loc与slot确定   │           │ NoModRef │
│ 不相交？              │           └──────────┘
└──────────┬───────────┘
      是   │   否
           │
           v
    ┌──────────┐  ┌──────────┐
    │ NoModRef │  │   Ref    │
    └──────────┘  └──────────┘
```

核心逻辑：

```cpp
if (!EnableBarrierFieldAA)
  return AAResultBase::getModRefInfo(Call, Loc, AAQI);

JavaBarrierKind Kind = getTrustedJavaBarrier(*Call);

// 同名函数、错误签名、错误calling convention或没有可信attribute，
// 都不能套用GC barrier语义。
if (Kind == JavaBarrierKind::None)
  return AAResultBase::getModRefInfo(Call, Loc, AAQI);

// 只精确回答barrier对Java heap payload的影响。
// 对card table、TLS、C heap仍保持默认结果。
if (!isJavaHeapLocation(Loc))
  return AAResultBase::getModRefInfo(Call, Loc, AAQI);

if (Kind == JavaBarrierKind::Post) {
  // post-barrier只使用Java heap地址计算card地址，
  // 不读取或写入Java heap payload。
  return ModRefInfo::NoModRef;
}

// pre-barrier的参数0是将被覆盖的reference slot。
uint64_t OopBytes = getPreBarrierSlotBytes(*Call);
MemoryLocation Slot(
    Call->getArgOperand(0),
    LocationSize::precise(OopBytes));

AliasResult R = AAQI.AAR.alias(Loc, Slot, AAQI, Call);

if (R == AliasResult::NoAlias)
  return ModRefInfo::NoModRef;

// pre-barrier可能读取该slot，但不会修改Java heap payload。
return ModRefInfo::Ref;
```

`getTrustedJavaBarrier`只接受以下runtime wrapper：

| 检查项 | 要求 |
|---|---|
| callee | `jeandle.pre_barrier`或`jeandle.post_barrier` |
| calling convention | call和callee均为`HotSpot_JIT` |
| function attribute | `jeandle-java-barrier-kind="pre"/"post"` |
| 签名 | pre为`void(JavaHeapPtr)`；post为`void(JavaHeapPtr,JavaHeapPtr)` |

因此同名伪barrier、错误签名、缺少attribute或普通calling convention都会回退LLVM基类AA。

#### Compressed oop的slot范围

pre-barrier的参数0是reference slot地址，不是slot中保存的oop值。Java heap地址本身通常使用宽指针，但slot可能保存4字节compressed oop。`InsertGCBarriers`在函数启用`use-compressed-oops`且store值类型明确时，在pre-barrier call site附加：

即使查询地址与参数0是同一个SSA值，也不能只凭“参数相同”直接过滤：查询的`LocationSize`可能覆盖slot之外的字节，仍必须按slot宽度做范围判断；不同SSA值也不自动代表不同slot。

```text
jeandle-java-barrier-oop-kind="narrow"  // compressed oop
jeandle-java-barrier-oop-kind="wide"    // wide oop
```

AA仅在以下条件同时满足时采用窄slot宽度：

1. 编译函数有`use-compressed-oops`attribute；
2. call site明确标记`narrow`；
3. `DataLayout`中的`NarrowOopAddrSpace`宽度严格小于Java heap地址空间宽度。

否则使用Java heap指针宽度，宁可漏掉相邻field的优化，也不缩小真实slot范围。例如真实compressed slot为`[16,20)`，相邻primitive field为`[20,24)`；只有可信的`narrow`标记才能返回`NoModRef`，缺少标记时按宽slot`[16,24)`保守返回`Ref`。

正例——查询相邻primitive field：

```text
obj.value:     [12,16)
obj.reference: [24,32)

pre_barrier(&obj.reference)对obj.value
=> 两个range不相交
=> NoModRef
```

反例——查询同一个reference slot：

```text
query Loc:     obj.reference [24,32)
barrier slot:  obj.reference [24,32)

pre_barrier会读取该slot
=> Ref
```

对应场景：

| 场景 | 处理 |
|---|---|
| pre-barrier与相邻primitive field不重叠 | `NoModRef` |
| pre-barrier与同一reference slot重叠 | `Ref` |
| pre-barrier与跨字段wide load可能重叠 | `Ref` |
| pre-barrier地址来源不明，无法证明不相交 | `Ref` |
| post-barrier查询primitive field | `NoModRef` |
| post-barrier查询reference field | `NoModRef` |
| post-barrier查询C heap/card table/TLS | 默认ModRef |
| 错误签名的同名函数 | 默认ModRef |
| 会读取Java heap的同名伪barrier | 默认ModRef |
| G1/Serial pre-barrier wrapper | 按实际reference-slot读取建模 |
| G1/Serial post-barrier wrapper | 对Java heap payload返回`NoModRef` |

关键区别：

```text
safepoint AA依赖field/element是否具有GC不变性；
barrier AA依赖barrier自身明确、可信的内存行为，
不应依赖SafepointInvariant。
```

**fresh allocation barrier消除与当前场景功能差异**

两者都围绕GC barrier，属于互补优化。

| 对比 | 当前GC barrier Mod/Ref AA | `dev_3`的`ReduceInitialCardMarks` |
|---|---|---|
| 核心问题 | 已存在的barrier会不会读写某个`MemoryLocation` | fresh object初始化时是否根本不需要barrier |
| 分析对象 | `barrier call + Loc` | `allocation + reference store + escape/safepoint路径` |
| 结果 | 返回`NoModRef`或`Ref` | 不插入pre/post barrier |
| 主要收益 | 让GVN/LICM跨barrier复用或移动load | 直接消除barrier运行开销 |
| 适用范围 | 普通对象和fresh object中仍存在的barrier | 仅fresh、未发布且路径安全的对象初始化 |

例如：

```java
Node n = new Node();
n.ref = value;
```

`dev_3`证明`n`是fresh且未发布，直接不生成barrier。

而：

```java
int x = obj.value;
obj.ref = value;       // barrier必须保留
return x + obj.value;
```

当前Mod/Ref AA保留barrier，但证明它不影响`obj.value`，从而消除第二次load。

所以可以理解为：

```text
dev_3：能删barrier就删；
GC barrier Mod/Ref AA：不能删barrier时，减少它对其他内存优化的阻碍。
```

#### Barrier的插入、降低与消费

`InsertGCBarriers`只处理Java heap中的reference store（包括wide/narrow oop）：在store前插入pre-barrier，在非null store后插入post-barrier。runtime template为两个wrapper添加`jeandle-java-barrier-kind`；G1或Serial的具体实现由wrapper在后续JavaOperationLower阶段展开。

当前Jeandle流水线中的相关顺序为：

```text
InsertGCBarriers
  -> JavaOperationLower(1)
  -> default O3
  -> ExpandNarrowOopCast
  -> RewriteStatepointsForGC
  -> JavaOperationLower(9)   // 展开pre/post wrapper
  -> JavaOperationDeletion/TLSPointerRewrite
  -> GVN
  -> DSE
```

因此barrier AA是否带来实际优化，不能只看query命中，还要确认query发生在wrapper仍存在的阶段，并且GVN/DSE前后的load/store确实减少。`JavaOperationLower(9)`之后wrapper通常已经展开，末端GVN/DSE可能无法再查询`jeandle.pre/post_barrier`。

可用诊断开关：

```text
-jeandle-trace-barrier-pipeline
-jeandle-trace-barrier-field-aa
-stats
```

其中`-jeandle-trace-barrier-pipeline`在`after-insert-gc-barriers`、`after-o3`、`after-rs4gc`、`after-lower-9`和`after-gvn-dse`输出barrier、load、store数量；`-jeandle-trace-barrier-field-aa`输出trusted query、`NoModRef`和`Ref`统计。只有AA结果和IR变化能够对应起来时，才能宣称barrier AA被流水线实际消费。

#### 当前开关

| 开关 | 默认值 | 作用 |
|---|---:|---|
| `-jeandle-enable-java-heap-aa` | `false` | 将`JavaHeapAA`注册到default AA pipeline；也可通过`-aa-pipeline=...,java-heap-aa`显式使用 |
| `-jeandle-enable-java-field-aa` | `true` | structured object field `NoAlias`；也是field safepoint AA的provenance前置开关 |
| `-jeandle-enable-java-array-aa` | `true` | 常量index array element `NoAlias`；也是array safepoint AA的provenance前置开关 |
| `-jeandle-enable-java-allocation-aa` | `true` | frontend为`new`/`newarray`返回值添加`noalias` |
| `-jeandle-enable-gc-relocate-aa` | `true` | 回溯`gc.relocate`的derived pointer |
| `-jeandle-enable-safepoint-field-aa` | `false` | primitive field跨`safepoint_handler`的`NoModRef`，experimental |
| `-jeandle-enable-safepoint-array-aa` | `true` | primitive array element跨`safepoint_handler`的`NoModRef`，与field开关独立 |
| `-jeandle-enable-barrier-field-aa` | `true` | trusted GC barrier的Java heap Mod/Ref |
| `-jeandle-safepoint-field-aa-method-filter` | 空 | 仅在函数名包含指定子串时启用field safepoint AA |

所有范围、metadata、root、klass或barrier contract无法证明的情况都回退LLVM基类结果；`NoAlias`/`NoModRef`不是默认假设。

诊断开关`-jeandle-trace-safepoint-field-aa`、`-jeandle-trace-barrier-field-aa`、`-jeandle-trace-barrier-pipeline`和LLVM通用的`-stats`用于统计query、`NoModRef`结果、LICM hoist以及各Pipeline阶段的load/store数量，不改变分析语义。

#### 当前验证结论

- `structured-fields.ll`：field range、metadata offset校验、外层GEP、header/raw地址、safepoint和barrier消费；
- `structured-arrays.ll`：常量element range、动态/宽访问、primitive array safepoint以及field/array独立开关；
- `allocation-provenance.ll`、`gc-relocate-provenance.ll`：fresh allocation和relocation后root/provenance；
- `gc-barrier-modref.ll`、`gc-barrier-compressed-oop.ll`：pre同slot/相邻field/宽访问、post Java/C heap、untrusted wrapper和compressed oop宽度；
- `exact-klass.ll`、`incompatible-classes.ll`：exact/incompatible Java klass的保守性边界。

已有针对性JMH结果保持不变：不同object field load约`+0.86%`、primitive field跨safepoint约`+12%`、不同常量array element load约`+14.2%`、array element dead store约`+22.8%`、fresh allocation约`+3.6%`；相同element等负例中性。GC barrier Mod/Ref目前有lit和pipeline诊断覆盖，但暂无独立、稳定的端到端收益，因此不宣称barrier专项性能收益。
