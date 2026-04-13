#### 描述
[linker](https://github.com/llvm/llvm-project/pull/191790/files)

#### Analysis
该PR在**LLVM InstCombine 优化Pass**中，为单索引GEP（GetElementPtr）的规范化逻辑增加了**包含目标扩展类型（Target Extension Type）的类型强制跳过规范化**的保护机制，避免通用中端优化破坏带后端自定义语义的目标扩展类型，从根源上修复了非法类型重解释导致的后端编译失败、代码生成错误问题。

### 背景与问题根因
1. **原有GEP规范化逻辑**
    InstCombine中存在一个长期的GEP规范化优化：当GEP指令**只有1个索引**、且元素类型不是`i8`/`[N x i8]`时，会将其重写为基于`[sizeof(T) x i8]`的字节型GEP。
    例如：`getelementptr %struct.T, ptr %p, i64 %idx` 会被转换为 `getelementptr [16 x i8], ptr %p, i64 %idx`（假设`struct.T`大小为16字节）。
    该优化的目的是统一GEP的IR形式，方便后续的地址计算合并、别名分析等优化。

2. **目标扩展类型（TargetExtType）的语义约束**
    TargetExtType是LLVM IR中用于表达**后端自定义语义类型**的核心机制，典型场景包括SPIR-V的设备类型（如`target("spirv.DeviceEvent")`）、AMDGPU后端自定义类型、ARM SME/SVE架构扩展类型等。
    这类类型的核心约束是：**中端通用优化不得随意重解释、折叠、转换其类型，必须完整保留到后端，否则会破坏后端定义的语义，导致编译失败或运行时异常**。

3. **原有逻辑的致命缺陷**
    原有的GEP规范化逻辑没有对TargetExtType做任何豁免，会强行将包含TargetExtType的类型也重解释为字节数组，彻底抹除了后端自定义的类型语义，导致依赖该类型的后端无法正确处理IR，最终引发误编译。

### PR 具体实现的核心改动
1. **新增递归类型检查能力**
    在`visitGetElementPtrInst`函数中新增了递归检查函数`IsTargetExtType`，可以深度遍历类型树，判断一个类型是否**直接或间接包含TargetExtType**，覆盖的类型场景包括：
    - 基础场景：直接的TargetExtType（基例）
    - 向量类型：递归检查其标量元素类型
    - 数组类型：递归检查其数组元素类型
    - 结构体类型：遍历所有成员，只要有一个成员包含TargetExtType即返回true
    - 其他基础类型（整数、浮点、指针等）：返回false

2. **修改GEP规范化的触发条件**
    为原有的规范化触发条件新增了关键保护：
    ```cpp
    // 原有条件
    if (Indices.size() == 1 && !IsCanonicalType(GEPEltType))
    // 修改后条件
    if (Indices.size() == 1 && !IsCanonicalType(GEPEltType) && !IsTargetExtType(GEPEltType))
    ```
    核心效果：**只要GEP的元素类型包含TargetExtType，哪怕满足单索引、非规范化类型的条件，也绝对不会执行转i8数组的规范化操作**，完整保留原始类型语义。

3. **新增完整的验证测试用例**
    新增了`gep-target-ext-type.ll`测试文件，覆盖3个核心场景，验证InstCombine不会修改带TargetExtType的GEP：
    - 直接对TargetExtType执行GEP
    - 对TargetExtType的数组执行GEP
    - 对嵌套结构体中包含TargetExtType的类型执行GEP

### PR 解决的核心价值
- **正确性修复**：彻底避免了通用优化对TargetExtType的非法类型重解释，保证了SPIR-V、AMDGPU、ARM SME等架构后端的编译正确性。
- **语义合规**：严格遵循了LLVM对TargetExtType的设计约定——后端自定义类型必须被中端优化完整保留。
- **无侵入性**：仅对包含TargetExtType的类型做豁免，完全不影响普通类型的原有GEP规范化优化逻辑。

---

## 二、必须考虑的边界情况
### 一、类型系统检查的完整性边界
这是PR最核心的风险点，递归检查的覆盖度直接决定了修复的完整性。
1. **极端多层嵌套的复合类型**
    当前测试仅覆盖了一层嵌套结构体，未验证**结构体套数组、数组套结构体的多层深度嵌套**场景（如`struct A { struct B { [2 x struct C { target("xxx") }] } }`），需确认递归逻辑能无遗漏识别深层嵌套的TargetExtType。
2. **不透明结构体（Opaque Struct）**
    对于仅前向声明、无成员定义的不透明结构体，当前逻辑遍历`elements()`会得到空列表，返回`false`，但这类结构体可能在后端定义中包含TargetExtType，会出现漏检，导致非法规范化。
3. **可伸缩向量类型（Scalable Vector Type）**
    针对ARM SVE/RISC-V Vector等架构的可伸缩向量类型（如`vscale x 4 x target("xxx")`），需确认`isVectorTy()`能正确匹配、`getScalarType()`能正确拿到TargetExtType，避免漏检。
4. **指针类型的嵌套场景**
    当前逻辑未处理指针类型，若GEP元素类型是`ptr to target("xxx")`，检查会返回`false`。需明确：GEP规范化仅修改元素类型本身，不会修改指针指向的类型，该场景是否需要豁免？是否存在语义破坏风险？
5. **联合体类型与零长度数组**
    需验证：用结构体表示的联合体、包含零长度柔性数组成员的结构体，其内部的TargetExtType能否被正确识别。

### 二、GEP指令本身的边界场景
1. **多索引GEP转单索引的连锁优化**
    当前仅处理单索引GEP，但InstCombine中存在其他优化会将多索引GEP（如`gep %struct.T, ptr %p, i32 0, i32 1`）合并为单索引GEP。需确认：合并后的单索引GEP若包含TargetExtType，能否被正确豁免，不会被后续的规范化逻辑处理。
2. **带NoWrap标志的GEP**
    针对带`inbounds`/`nuw`/`nsw`标志的GEP，需验证跳过规范化后，标志位能否被完整保留，不会出现语义丢失；同时确认原有逻辑中，可伸缩类型的`assert(!Scale.isScalable())`与TargetExtType检查不会出现冲突。
3. **特殊索引场景**
    需覆盖验证：常量索引、零索引、负数索引的单索引GEP，在包含TargetExtType时，能否被正确跳过规范化；不同地址空间的指针基址（如GPU的私有/全局地址空间）是否会影响检查逻辑。

### 三、语义正确性与优化交互边界
1. **与其他GEP优化的兼容性**
    PR仅修复了规范化逻辑，需确认InstCombine中其他GEP优化（如GEP合并、常量折叠、地址计算简化）是否也存在对TargetExtType的非法转换，是否需要同步增加保护。
2. **别名分析的影响**
    原有规范化转i8的逻辑会改变别名分析的结果，跳过规范化后，需确认LLVM的别名分析能正确处理TargetExtType，不会出现过度保守或过度激进的别名判断，导致其他优化失效。
3. **TargetExtType的内存访问语义**
    需验证：若GEP被非法规范化为i8类型，后续的load/store会被优化为字节访问，破坏TargetExtType要求的特殊加载/存储指令（如SPIR-V的设备类型访问、ARM SME的矩阵寄存器访问），PR的修改是否完全规避了该风险。
4. **Undef/Poison语义**
    针对带undef/poison的索引、基址的GEP，需确认跳过规范化后，不会引入新的UB（未定义行为），语义与原有逻辑保持一致。

### 四、实现性能与工程边界
1. **编译时间开销**
    `visitGetElementPtrInst`是InstCombine中的高频调用函数，每个GEP指令都会触发执行。当前每次调用都会创建`std::function`并递归遍历类型树，对于大型项目中的复杂结构体、海量GEP指令，会带来显著的编译时间开销。
    优化方向：改为静态非递归的迭代实现、基于LLVM类型唯一化特性增加检查结果缓存，避免重复计算。
2. **递归栈溢出风险**
    极端场景下，超多层嵌套的类型会导致递归检查栈溢出，LLVM虽对类型嵌套深度有默认限制，但递归实现天然存在该风险，更健壮的方案是改为迭代遍历。

### 五、兼容性与下游适配边界
1. **原有逻辑的兼容性**
    需补充负向测试用例，验证**不包含TargetExtType的普通类型**，依然会被正常执行GEP规范化，确保PR没有破坏原有优化逻辑。
2. **下游后端的适配**
    需确认SPIR-V、AMDGPU、ARM等重度依赖TargetExtType的后端，是否存在适配原有规范化逻辑的hack代码，PR的修改是否会导致下游适配失效。
3. **LTO/MLIR交互场景**
    需验证在LTO/ThinLTO跨模块编译、MLIR生成LLVM IR的场景中，TargetExtType能被正确识别，豁免逻辑正常生效。

### 六、测试覆盖的缺失边界
当前测试仅覆盖了3个基础正向场景，需补充的关键测试：
1. 负向测试：普通类型的GEP依然会被正常规范化
2. 多层嵌套的复杂复合类型测试
3. 可伸缩向量类型、不透明结构体测试
4. 带`inbounds`等NoWrap标志的GEP测试
5. 常量索引、零索引、负数索引的GEP测试
6. 多索引转单索引的连锁优化场景测试
