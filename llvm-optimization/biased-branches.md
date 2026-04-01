
- 当我们在代码中添加likely（对应编译器内建函数__builtin_expect）或unlikely这类分支预测提示后，开启O1编译优化级别并查看生成的汇编代码时会发现，被标记为likely的分支代码会被编译器排布在条件判断指令的后续执行路径上（即“无跳转”的直连路径）。

<img width="2638" height="920" alt="image" src="https://github.com/user-attachments/assets/8e4ea675-c070-453a-af57-58debf149900" />
<img width="2564" height="936" alt="image" src="https://github.com/user-attachments/assets/87384c75-ac6b-43ea-af30-53f78d12153a" />

- demo
  ```
  #include <benchmark/benchmark.h>
  
  #include<algorithm>
  #include<random>
  #include<vector>
  
  static void custom_args(benchmark::internal::Benchmark *b) {
  	for (auto i : {14}) {
  		for (auto j : {0, 10, 20, 30, 40, 50, 60, 70, 80, 90, 100}) {
  			b = b->ArgPair(i, j);
  		}
  	}
  }
  
  // Benchmakr for using branches
  static void branchBenchRandom(benchmark::State &s) {
  	// Get the input vector size
  	auto N = 1 << s.range(0);
  
  	// Get the distribution
  	double probability = s.range(1) / 100.0;
  
  	// Create random number generator
  	// Bernolli distributio gives T/F outcomes
  	std::random_device rd;
  	std::mt19937 gen(rd());
  	std::bernoulli_distribution d(probability);
  
  	// Create a vector of randown booleans
  	std::vector<bool> v_in(N);
  	std::generate(begin(v_in), end(v_in), [&](){return d(gen);});
  
  	// Output element;
  	int sink = 0;
  
  	///Benchmark main loop
  	for (auto _ :s) {
  	   for (auto b : v_in) 
  	      if (b) benchmark::DoNotOptimize(sink += s.range(0));
  	}
  }
  
  BENCHMARK(branchBenchRandom)->Apply(custom_args)->Unit(benchmark::kMicrosecond);
  BENCHMARK_MAIN();
  ```
- 通过不同的概率查看分支预测情况: 中间耗时大，两头小；perf stat显示branch-miss跟耗时趋势一样；
  ```
   g++ random.cpp -O3 -march=native -mtune=native -flto -fuse-linker-plugin --std=c++2a -pthread -lbenchmark -o random
   sudo perf stat ./random --benchmark_filter=branchBenchRandom/14/0

  ./random 
  2026-02-04T13:35:00+00:00
  Running ./random
  Run on (10 X 48 MHz CPU s)
  Load Average: 0.00, 0.00, 0.00
  ***WARNING*** Library was built as DEBUG. Timings may be affected.
  -------------------------------------------------------------------
  Benchmark                         Time             CPU   Iterations
  -------------------------------------------------------------------
  branchBenchRandom/14/0         9.23 us         9.23 us        64400
  branchBenchRandom/14/10        9.62 us         9.62 us        72629
  branchBenchRandom/14/20        10.3 us         10.3 us        66517
  branchBenchRandom/14/30        11.4 us         11.4 us        61202
  branchBenchRandom/14/40        12.7 us         12.7 us        54823
  branchBenchRandom/14/50        13.8 us         13.8 us        49407
  branchBenchRandom/14/60        12.7 us         12.7 us        56184
  branchBenchRandom/14/70        10.2 us         10.2 us        68919
  branchBenchRandom/14/80        8.48 us         8.48 us        83314
  branchBenchRandom/14/90        8.12 us         8.12 us        85753
  branchBenchRandom/14/100       8.21 us         8.21 us        83141
  ```

#### 附件
- godbolt: https://godbolt.org/z/n1E4KP47d
- https://github.com/CoffeeBeforeArch/misc_code/blob/master/biased_branches/random.cpp
