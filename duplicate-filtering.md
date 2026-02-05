  - uniq去重操作
  - 以unordered_set为例，若元素增加，耗时增加；元素个数增加一个数量级，耗时增加一倍；元素范围增加一倍，耗时变为6倍（想想为什么）；
  - 若从vector转为unordered_set，性能提升，因为使用hash表来插入；
  - c++标准库的性能还是最好的；
  
- 构建运行，控制数组大小（2^10）和取值范围(10000)
  ```
   g++ duplicate_removal.cpp -g -O3 -flto -fuse-linker-plugin -march=native -mtune=native -lbenchmark -lpthread -o duplicate_removal  -std=c++20
   ./duplicate_removal

  ----------------------------------------------------------------------
  Benchmark                            Time             CPU   Iterations
  ----------------------------------------------------------------------
  baseline/10（元素个数）/10（元素范围）                    2.48 us         2.48 us       280029
  baseline/10/100                   8.10 us         8.10 us        85010
  baseline/10/1000                  53.1 us         53.1 us        12245
  baseline/10/10000                 85.0 us         85.0 us         8268
  baseline/11/10                    4.90 us         4.90 us       142258
  baseline/11/100                   17.2 us         17.2 us        41052
  baseline/11/1000                   141 us          141 us         4854
  baseline/11/10000                  308 us          308 us         2274
  baseline/12/10                    9.88 us         9.88 us        69522
  baseline/12/100                   35.3 us         35.3 us        19845
  baseline/12/1000                   328 us          328 us         2079
  baseline/12/10000                 1074 us         1074 us          660
  unordered_set/10/10              0.757 us        0.757 us       912745
  unordered_set/10/100              1.86 us         1.86 us       377141
  unordered_set/10/1000             8.73 us         8.73 us        78322
  unordered_set/10/10000            12.5 us         12.5 us        55344
  unordered_set/11/10               1.40 us         1.40 us       491051
  unordered_set/11/100              2.50 us         2.50 us       286786
  unordered_set/11/1000             12.3 us         12.3 us        57597
  unordered_set/11/10000            23.7 us         23.7 us        28899
  unordered_set/12/10               2.69 us         2.69 us       261592
  unordered_set/12/100              3.92 us         3.92 us       178936
  unordered_set/12/1000             14.8 us         14.8 us        47705
  unordered_set/12/10000            48.8 us         48.8 us        13723
  unordered_set_copy/10/10         0.764 us        0.764 us       894928
  unordered_set_copy/10/100         1.95 us         1.95 us       356987
  unordered_set_copy/10/1000        9.58 us         9.58 us        76148
  unordered_set_copy/10/10000       13.5 us         13.5 us        51072
  unordered_set_copy/11/10          1.45 us         1.45 us       490077
  unordered_set_copy/11/100         2.62 us         2.62 us       268764
  unordered_set_copy/11/1000        13.1 us         13.1 us        54551
  unordered_set_copy/11/10000       26.1 us         26.1 us        26894
  unordered_set_copy/12/10          2.64 us         2.64 us       264813
  unordered_set_copy/12/100         3.76 us         3.76 us       186388
  unordered_set_copy/12/1000        15.5 us         15.5 us        44777
  unordered_set_copy/12/10000       52.0 us         52.0 us        13260
  sort_unique/10/10                 4.26 us         4.26 us       164735
  sort_unique/10/100                5.38 us         5.38 us       127074
  sort_unique/10/1000               5.66 us         5.66 us       122947
  sort_unique/10/10000              5.55 us         5.55 us       114346
  sort_unique/11/10                 8.59 us         8.59 us        82214
  sort_unique/11/100                11.7 us         11.7 us        60750
  sort_unique/11/1000               13.2 us         13.2 us        52622
  sort_unique/11/10000              12.5 us         12.5 us        54883
  sort_unique/12/10                 18.1 us         18.1 us        38179
  sort_unique/12/100                24.0 us         24.0 us        28516
  sort_unique/12/1000               32.9 us         32.9 us        25688
  sort_unique/12/10000              30.4 us         30.4 us        24814
  ```

#### 附件
- https://github.com/CoffeeBeforeArch/misc_code/blob/master/duplicate_removal/duplicate_removal.cpp
