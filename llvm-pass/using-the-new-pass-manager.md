- 处理完一个函数的 Pass1 和 Pass2 后，这个函数的中间数据就能及时释放，不用一直存在内存里。而如果先跑完全部函数的 Pass1，再跑 Pass2，那所有函数的 Pass1 结果都得缓存着，内存占用会高很多。编译器在设计时，这种 “局部性优先” 的原则能让它在有限内存下更高效地处理大型程序。
  ```
  # 差距在于执行范围和效率
  # 先用pass1循环一轮，然后再用pass2循环一轮；流水线有两个环节，先经过第一个环节，然后经过第二个环节
  ModulePassManager MPM;
  MPM.addPass(createModuleToFunctionPassAdaptor(FunctionPass1()));
  MPM.addPass(createModuleToFunctionPassAdaptor(FunctionPass2()));
  MPM.run();
  # 某个函数一次性执行所有function pass，可以释放不需要的分析信息；也就是把两个pass当作流水线一个环节；
  ModulePassManager MPM;
  FunctionPassManager FPM;
  FPM.addPass(FunctionPass1());
  FPM.addPass(FunctionPass2());
  MPM.addPass(createModuleToFunctionPassAdaptor(std::move(FPM)));
  ```

- Clang’s BackendUtil.cpp shows examples of a frontend adding (mostly sanitizer) passes to various parts of the pipeline. AMDGPUTargetMachine::registerPassBuilderCallbacks() is an example of a backend adding passes to various parts of the pipeline.
- Pass plugins can also add passes into default pipelines. Different tools have different ways of loading dynamic pass plugins. For example, opt -load-pass-plugin=path/to/plugin.so loads a pass plugin into opt. For information on writing a pass plugin
- 所以修改pass有几种方式，手动默认流水线源码add、通过default passbuilder钩子函数添加、opt load-pass-plugin 这几种方式
- 在 LLVM 流水线中，不同 Pass 通过 IR 传递数据，分析 Pass 的结果会被优化 Pass 依赖。当 IR 被修改时，LLVM 的 Pass 管理器会自动检测依赖关系，失效的分析信息会被标记为 “脏”，后续 Pass 需要时会重新计算，避免使用过时数据。新 Pass 管理器的缓存机制就是干这个的，确保分析信息的正确性
- llvm会评估分析结果后续是否会被用到，如果被用到则保留着，假如两个paas1 pass2绑定执行，那么分析结果就不需要一直保存着，减少内存使用。
- When a pass runs on some IR, it also receives an analysis manager which it can query for analyses. Querying for an analysis will cause the manager to check if it has already computed the result for the requested IR. If it already has and the result is still valid, it will return that. Otherwise it will construct a new result by calling the analysis’s run() method, cache it, and return it. You can also ask the analysis manager to only return an analysis if it’s already cached.
  ```
  // We've made no transformations that can affect any analyses.
  return PreservedAnalyses::all();
  
  // We've made transformations and don't want to bother to update any analyses.
  return PreservedAnalyses::none();
  
  // We've specifically updated the dominator tree alongside any transformations, but other analysis results may be invalid.
  PreservedAnalyses PA;
  PA.preserve<DominatorAnalysis>();
  return PA;
  
  // We haven't made any control flow changes, any analyses that only care about the control flow are still valid.
  PreservedAnalyses PA;
  PA.preserveSet<CFGAnalyses>();
  return PA;
  ```
- If it already has and the result is still valid, it will return that. Otherwise it will construct a new result by calling the analysis’s run() method, cache it, and return it. 
- 函数级 Pass 默认只能直接获取当前函数的分析结果。如果要获取其他函数的分析结果，或者更高层级的分析结果，就需要通过 Proxy 来间接访问，这样既能跨层级获取信息，又能保证分析缓存的正确管理。
- 这段代码是 LLVM 新 Pass 管理器环境下，在 CGSCC (Call Graph SCC) 级别的 Pass 中，尝试获取一个模块级别分析（FooAnalysis）的缓存结果。
  ```
  const auto &MAMProxy =
      AM.getResult<ModuleAnalysisManagerCGSCCProxy>(InitialC, CG);
  FooAnalysisResult *AR = MAMProxy.getCachedResult<FooAnalysis>(M);
  ```
  - 将失效机制与分析管理器代理结合会产生一定复杂性：
1. 模块级分析全部失效时，需同步失效内部代理可访问的函数级分析。内部代理先判断自身是否失效，若是则清空相关分析结果，否则将失效请求转发给内部分析管理器。
2. 外部代理的分析结果通常不可变，无需考虑失效；但部分内部分析会依赖外部分析，外部分析失效时，依赖它的内部分析也需失效（如别名分析）。
3. 当前仅 GlobalsAA 是模块级别名分析，且基本不会失效，该问题影响有限，可参考 `OuterAnalysisManagerProxy::Result::registerOuterAnalysisInvalidation()`。

需要我把这段总结再压缩成**更短的纯要点版**吗？
#### further reading
  - https://llvm.org/docs/NewPassManager.html
