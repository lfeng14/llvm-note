- 默认数组排序；
- 计数排序；
  - unordered_map 使用的是 std::unordered_map<int, int>
  - flat_hash_map 使用的是 absl::flat_hash_map<int, int>

- 构建
  ```
  g++ sorting.cpp -g -O3 -flto -fuse-linker-plugin -march=native -mtune=native -lbenchmark -lpthread -labsl_hash -labsl_raw_hash_set -o sorting -std=c++2a
  ./sorting 
  2026-02-04T16:05:37+00:00
  Running ./sorting
  Run on (10 X 48 MHz CPU s)
  Load Average: 0.04, 0.03, 0.00
  ***WARNING*** Library was built as DEBUG. Timings may be affected.
  -----------------------------------------------------------------
  Benchmark                       Time             CPU   Iterations
  -----------------------------------------------------------------
  baseline/14/10               71.4 us         71.4 us         9011
  baseline/14/100               229 us          229 us         2699
  baseline/14/1000              371 us          371 us         2592
  baseline/14/10000             440 us          440 us         2347
  baseline/15/10                192 us          192 us         3532
  baseline/15/100               458 us          458 us         1230
  baseline/15/1000              812 us          812 us         1016
  baseline/15/10000             979 us          979 us          726
  baseline/16/10                574 us          574 us          991
  baseline/16/100              1147 us         1147 us          601
  baseline/16/1000             1597 us         1597 us          430
  baseline/16/10000            2038 us         2038 us          341
  unordered_map/14/10          76.7 us         76.7 us         9136
  unordered_map/14/100         77.8 us         77.8 us         9085
  unordered_map/14/1000        96.5 us         96.5 us         7234
  unordered_map/14/10000        349 us          349 us         1985
  unordered_map/15/10           153 us          153 us         4520
  unordered_map/15/100          155 us          155 us         4508
  unordered_map/15/1000         172 us          172 us         4065
  unordered_map/15/10000        539 us          539 us         1283
  unordered_map/16/10           310 us          310 us         2257
  unordered_map/16/100          309 us          309 us         2264
  unordered_map/16/1000         323 us          323 us         2157
  unordered_map/16/10000        736 us          736 us          955
  flat_hash_map/14/10          67.4 us         67.4 us        10501
  flat_hash_map/14/100         76.7 us         76.7 us         9074
  flat_hash_map/14/1000         104 us          104 us         6688
  flat_hash_map/14/10000        435 us          434 us         1606
  flat_hash_map/15/10           135 us          135 us         5198
  flat_hash_map/15/100          152 us          152 us         4583
  flat_hash_map/15/1000         175 us          175 us         3978
  flat_hash_map/15/10000        607 us          607 us         1114
  flat_hash_map/16/10           269 us          269 us         2606
  flat_hash_map/16/100          307 us          307 us         2289
  flat_hash_map/16/1000         319 us          319 us         2223
  flat_hash_map/16/10000        760 us          760 us          847 <-- 视频中这种效果更好
  ```
- 标准库排序 < unordered_map计数 < flat_hash_map计数
#### 附件
- https://github.com/CoffeeBeforeArch/misc_code/blob/master/sorting/sorting.cpp
