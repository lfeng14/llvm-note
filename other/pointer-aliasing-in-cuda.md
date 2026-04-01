- 编译器对于别名分析采用保守策略，如果没有加__restrict则认为两个变量有可能overlap，所以修改了另外一个变量就需要重新load当前变量；
  <img width="1836" height="1534" alt="image" src="https://github.com/user-attachments/assets/fa0c2aad-4e5b-4e64-88bf-58f8f9fdc41d" />

### 附件
- https://developer.nvidia.com/blog/cuda-pro-tip-optimize-pointer-aliasing/
- https://github.com/CoffeeBeforeArch/cuda_programming/blob/master/02_matrix_mul/noAliasing/restrict/mmul.cu
