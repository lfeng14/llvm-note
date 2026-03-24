- 采样方式信息：
  - Sampled line represents the profile information of a source location. offsetN[.discriminator]: number_of_samples [fn5:num fn6:num ... ]
  - Callsite line represents the profile information of an inlined callsite. offsetA[.discriminator]: fnA:num_of_total_samples
  ```
  function1:total_samples:total_head_samples  # 函数为单位，被执行次数
   offset1[.discriminator]: number_of_samples [fn1:num fn2:num ... ]  # 函数定位位置为起始地址，所以如果代码修改了，这时导致profile信息失效；同一行也有可能调用fn1 fn2  比如 if (xx) fn1(); else fn2();
   offset2[.discriminator]: number_of_samples [fn3:num fn4:num ... ]
   ...
   offsetN[.discriminator]: number_of_samples [fn5:num fn6:num ... ]  
   offsetA[.discriminator]: fnA:num_of_total_samples
    offsetA1[.discriminator]: number_of_samples [fn7:num fn8:num ... ]  # 记录已经内联的调用；如果非内联则没有缩进
    offsetA1[.discriminator]: number_of_samples [fn9:num fn10:num ... ]
    offsetB[.discriminator]: fnB:num_of_total_samples
     offsetB1[.discriminator]: number_of_samples [fn11:num fn12:num ... ]
  ```
  - This profile indicates that there were a total of 35,504 samples collected in main. All of those were at line 1 (the call to foo). Of those, 31,977 were spent inside the body of bar. The last line of the profile (2: 0) corresponds to line 2 inside main. No samples were collected there.
  ```
  main:35504:0
  1: _Z3foov:35504
    2: _Z32bari:31977
    1.1: 31977
  2: 0
  ```
- 插桩方式信息：
  - 可以充当函数覆盖率信息
  - 多编译一次插桩版本，插入计数指令，产生运行开销；记录数据更完整，效果比采样方式好；
  - Clang supports two types of instrumentation: frontend-based and IR-based. Frontend-based instrumentation can be enabled with the option -fprofile-instr-generate, and IR-based instrumentation can be enabled with the option -fprofile-generate. For best performance with PGO, IR-based instrumentation should be used. It has the benefits of lower instrumentation overhead, smaller raw profile size, and better runtime performance. Frontend-based instrumentation, on the other hand, has better source correlation, so it should be used with source line-based coverage testing.
  - 如果代码修改了，中端插桩影响更小
  - The flag -fcs-profile-generate also instruments programs using the same instrumentation method as -fprofile-generate. However, it performs a post-inline late instrumentation and can produce context-sensitive profiles.
  - LLVM_PROFILE_FILE="code-%p.profraw" ./code
  - 不同的modifier 可以避免覆盖： %p, %h, %m, %b, %t, and %c
  - Both -fprofile-use and -fprofile-instr-use accept profiles in the indexed format, regardeless whether it is produced by frontend or the IR pass.
  - -fcs-profile-generate[=<dirname>]：The difference is that the instrumentation is performed after inlining so that the resulted profile has a better context sensitive information
  - -fprofile-update：Unless -fsanitize=thread is specified, the default is single, which uses non-atomic increments. The counters can be inaccurate under thread contention. atomic uses atomic increments which is accurate but has overhead. prefer-atomic will be transformed to atomic when supported by the target, or single otherwise.
  - -fprofile-continuous：Enables the continuous instrumentation profiling where profile counter updates are continuously synced to a file
  - -ftemporal-profile：Enables the temporal profiling extension for IRPGO to improve startup time by reducing .text section page faults. To do this, we instrument function timestamps to measure when each function is called for the first time and use this data to generate a function order to improve startup
  - Fine Tuning Profile Collection
    - void __llvm_profile_set_filename(const char *Name): changes the name of the profile file to Name.
    - void __llvm_profile_reset_counters(void): resets all counters to zero.
    - int __llvm_profile_dump(void): write the profile data to disk.
    ```
    int main() {
      initialize();
    
      // Reset all profile counters to 0 to omit profile collected during
      // initialize()'s execution.
      __llvm_profile_reset_counters();
      ... hot region 1
      // Dump the profile for hot region 1.
      __llvm_profile_set_filename("region1.profraw");
      __llvm_profile_dump();
    
      // Reset counters before proceeding to hot region 2.
      __llvm_profile_reset_counters();
      ... hot region 2
      // Dump the profile for hot region 2.
      __llvm_profile_set_filename("region2.profraw");
      __llvm_profile_dump();
    
      // Since the profile has been dumped, no further profile data
      // will be collected beyond the above __llvm_profile_dump().
      cleanup();
      return 0;
    }
    ```
    - \_\_LLVM_INSTR_PROFILE_GENERATE: defined when one of -fprofile[-instr]-generate/-fcs-profile-generate is in effect.
    - \_\_LLVM_INSTR_PROFILE_USE: defined when one of -fprofile-use/-fprofile-instr-use is in effect.    
    ```
    #if __LLVM_INSTR_PROFILE_GENERATE
    expensive_logging_of_full_program_state();
    #endif
    ```
    - Instrumenting only selected files or functions: clang++ -O2 -fprofile-instr-generate -fprofile-list=fun.list code.cc -o code
#### further reading
- https://llvm.org/devmtg/2020-09/slides/PGO_Instrumentation.pdf
- Patch: https://github.com/kpdev/llvm-project/tree/llvm-dev-mtg/callsite
- PGO Docs: https://clang.llvm.org/docs/UsersManual.html#profile-guided-optimization
- MST: https://llvm.org/pubs/2010-04-NeustifterProfiling.pdf
- Presentations:
  - LLVM Dev Mtg 2013 Presentation: http://llvm.org/devmtg/2013-11/slides/Carruth-PGO.pdf
  - MSVC team talk: https://channel9.msdn.com/Shows/C9-GoingNative/C9GoingNative-12-C-at-BUILD-2012-Inside-Profile-Guided-Optimization
