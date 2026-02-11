-
```
$ g++ mod_bench.cpp -lbenchmark -lpthread -O3 -march=native -mtune=native -flto -fuse-linker-plugin -fprofile-generate -fno-tree-vectorize -o mod_bench
$ ./mod_bench 
2026-02-11T01:29:25+00:00
Running ./mod_bench
Run on (10 X 48 MHz CPU s)
Load Average: 0.00, 0.00, 0.00
***WARNING*** Library was built as DEBUG. Timings may be affected.
-------------------------------------------------------
Benchmark             Time             CPU   Iterations
-------------------------------------------------------
baseMod/1245       7.22 us         7.22 us        77690
srMod              1.27 us         1.27 us       548681
$ ll
total 76
drwxr-xr-x  2 franz my_test_group  4096 Feb 11 01:29 ./
drwxr-xr-x 25 franz my_test_group  4096 Feb  4 14:34 ../
-rwxr-xr-x  1 franz my_test_group 83288 Feb 11 01:29 mod_bench*
-rw-r--r--  1 franz my_test_group   748 Feb  4 14:34 mod_bench.cpp
-rw-r--r--  1 franz my_test_group  2104 Feb 11 01:29 mod_bench.gcda
$ g++ mod_bench.cpp -lbenchmark -lpthread -O3 -march=native -mtune=native -flto -fuse-linker-plugin -fprofile-use -fno-tree-vectorize -o mod_bench
$ ./mod_bench 
2026-02-11T01:29:41+00:00
Running ./mod_bench
Run on (10 X 48 MHz CPU s)
Load Average: 0.00, 0.00, 0.00
***WARNING*** Library was built as DEBUG. Timings may be affected.
-------------------------------------------------------
Benchmark             Time             CPU   Iterations
-------------------------------------------------------
baseMod/1245       1.18 us         1.18 us       530786
srMod              1.17 us         1.17 us       588981
```

#### 附件
- 代理代码：https://github.com/CoffeeBeforeArch/misc_code/blob/master/strength_reduction/mod_bench.cpp
- [示例](https://godbolt.org/z/34ThzrhjY)
