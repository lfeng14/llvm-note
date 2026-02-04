- 两个线程访问同一个变量，调度到同一个核上，共享缓存，减少同步开销；
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
  osScheduling/real_time         1.35 ms        0.124 ms          511
  threadAffinity/real_time      0.481 ms        0.071 ms         1389
  ```

  #### 附件
  - https://github.com/CoffeeBeforeArch/misc_code/blob/master/thread_affinity/thread_affinity.cpp
