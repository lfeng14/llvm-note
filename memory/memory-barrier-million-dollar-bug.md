- 编译器知道的假依赖；不会体现在汇编里。
- 想想如果没有加会导致什么问题，锁释放了，shared_value可能被意外覆盖。百万美元问题。
#### 附件
- https://godbolt.org/z/Yrv49vPGT
