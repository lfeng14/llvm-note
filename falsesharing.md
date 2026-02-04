<img width="1620" height="190" alt="image" src="https://github.com/user-attachments/assets/32bc62c1-659a-4104-b7c6-b409c725794d" /><img width="1620" height="190" alt="image" src="https://github.com/user-attachments/assets/32bc62c1-659a-4104-b7c6-b409c725794d" />- 通过打印发现四个变量内存地址是相邻的
  ```
    g++ vary_thread.cpp -o falsesharing -lbenchmark
  
    void bench4() {
    std::atomic<int> a{0};
    std::atomic<int> b{0};
    std::atomic<int> c{0};
    std::atomic<int> d{0};
  
    std::cout << "Address of a: " << &a << std::endl;
    std::cout << "Address of b: " << &b << std::endl;
    std::cout << "Address of c: " << &c << std::endl;
    std::cout << "Address of d: " << &d << std::endl;
  
    // Creat four threads and use lambda to launch work
    std::thread t1([&]() { work(a, 4); });
    std::thread t2([&]() { work(b, 4); });
    std::thread t3([&]() { work(c, 4); });
    std::thread t4([&]() { work(d, 4); });
  
    // Join the threads
    t1.join();
    t2.join();
    t3.join();
    t4.join();
  }

  Address of a: 0xffffc5121b60
  Address of b: 0xffffc5121b68
  Address of c: 0xffffc5121b70
  Address of d: 0xffffc5121b78
  ```
- 通过增加Align对齐属性
  ```
  g++ aligned_type.cpp -lbenchmark -o aligned_type
  $ ./aligned_type 
  Address of AlignedType a - 0xfffffff02300
  Address of AlignedType b - 0xfffffff022c0
  Address of AlignedType c - 0xfffffff02280
  Address of AlignedType d - 0xfffffff02240
  
  // Our aligned atomic
  struct alignas(64) AlignedType {
    AlignedType() { val = 0; }
    std::atomic<int> val;
  };
  ```
- 比较nosharing singsharing directsharing falsesharing
  - singThread: 1 atomic used by 1 threads
  - directSharing: 1 atomic shared by 4 threads
  - falseSharing: 4 atomics on 1 cache line shared by 4 threads
  - noSharing: 4 atomics each on their own cache line shared by 4 thread
  ```
  g++ false_sharing.cpp -o falsesharing -lbenchmark
  # ./falsesharing --benchmark_filter=noSharing --benchmark_min_time=3
  ./falsesharing 
  2026-02-04T11:06:40+00:00
  Running ./falsesharing
  Run on (10 X 48 MHz CPU s)
  Load Average: 0.00, 0.03, 0.01
  ***WARNING*** Library was built as DEBUG. Timings may be affected.
  ------------------------------------------------------------------
  Benchmark                        Time             CPU   Iterations
  ------------------------------------------------------------------
  singleThread                 0.703 ms        0.703 ms          783  <- second
  directSharing/real_time       8.02 ms        0.127 ms           85  <- fourth
  falseSharing/real_time        2.51 ms        0.123 ms          275  <- third
  noSharing/real_time           1.86 ms        0.118 ms          391  <- best
  ```
- 如何检测：perf c2c用来分析伪共享问题，伪共享问题包含：
  - 线程访问同一个变量；
  - 线程访问不同变量，但是变量在进程地址空间里是相邻的；
- 工具介绍
  ```
  perf stat -d -d -d ./falsesharing --benchmark_filter=directSharing --benchmark_min_time=3
  perf stat -d ./falsesharing
  // add
  perf c2c record  ./falsesharing --benchmark_filter=directSharing --benchmark_min_time=3
  // add
  perf c2c report
  # 可以看到程序debug信息
  perf c2c report --stdio # 文字报告
  # 查看哪个线程访问了缓存哪一行
  ```
<img width="1620" height="190" alt="image" src="https://github.com/user-attachments/assets/b8783e91-23f8-4f6f-98a7-cbb623108b97" />
<img width="1016" height="630" alt="image" src="https://github.com/user-attachments/assets/03af92b8-548b-4815-b4e2-f911fbbf2dad" />

#### 附件
- https://github.com/CoffeeBeforeArch/spring_2020_tutorial/blob/master/false_sharing/false_sharing.cpp
