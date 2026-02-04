- perf c2c用来分析伪共享问题，伪共享问题包含：
  - 线程访问同一个变量；
  - 线程访问不同变量，但是变量在进程地址空间里是相邻的；
- 工具介绍
perf stat -d ./falsesharing
// add
perf c2c record ./falsesharing
// add
perf c2c report
perf c2c report --stdio # 文字报告
// 查看哪个线程访问了缓存哪一行
#### 附件
- https://github.com/CoffeeBeforeArch/spring_2020_tutorial/blob/master/false_sharing/vary_thread.cpp
