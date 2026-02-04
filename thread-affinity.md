- demo
  ```
  void os_scheduler() {
    AlignedAtomic a{0};
    AlignedAtomic b{0};
  
    // Create four threads and use lambda to launch work
    std::thread t1([&]() { work(a.val); });
    std::thread t2([&]() { work(a.val); });
    std::thread t3([&]() { work(b.val); });
    std::thread t4([&]() { work(b.val); });
  
    // Join the threads
    t1.join();
    t2.join();
    t3.join();
    t4.join();
  }
  ```

- 两个线程分别访问其中一个变量，调度到同一个核上，共享缓存，减少同步开销；
- 现象：
  ```
  ./thread_affinity 
  2026-02-04T14:38:35+00:00
  Running ./thread_affinity
  Run on (10 X 48 MHz CPU s)
  Load Average: 0.00, 0.00, 0.00
  ***WARNING*** Library was built as DEBUG. Timings may be affected.
  -------------------------------------------------------------------
  Benchmark                         Time             CPU   Iterations
  -------------------------------------------------------------------
  osScheduling/real_time         1.35 ms        0.124 ms          511  <- 同一个核
  threadAffinity/real_time      0.481 ms        0.071 ms         1389  <- 不同核
  ```
- 进一步通过perf c2c record可以看到命令更高，缓存没有实效，只是不用重新拉取（使用perf c2c求证）；如果不同核上且核上只有1个cacheline那么会出现挤占现象吧（待求证）
  ```
  perf c2c record ./thread_affinity --benchmark_affinity=threadAffinity
  perf c2c record ./thread_affinity --benchmark_affinity=osScheduling
  ```
  #### 附件
  - https://github.com/CoffeeBeforeArch/misc_code/blob/master/thread_affinity/thread_affinity.cpp
