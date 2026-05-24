- [linker](https://github.com/llvm/llvm-project/pull/177665/changes#diff-5c3903d5b2af46cfdf087e8c615e2bd02a1b2f2bff4cf5dc1e80aa5de8c9f45c)
- 总结来说，这个改动针对 GPU 场景对 PGO 做了这些修改：  
  一是为 GPU 函数插入 profiling 调用，收集执行数据；  
  二是调整符号的链接属性，弱符号使用 weak 或 linkonce_odr 来方便多文件合并；  
  三是在编译和运行时添加特定的支持文件和逻辑，保证 GPU 端的 profiling 数据也能被收集和合并。
  
