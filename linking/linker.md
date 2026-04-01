# 链接器相关知识

## 静态链接
- 静态链接时，输入文件顺序比较关键，否则容易出现符号找不到问题

## 重定位
- 重定位条目（relocation entry）

## 重定位公式
计算从下一条指令的 PC 值到目标符号的偏移量时，使用公式：

```
(S + A - P)
```

其中：
- S 为目标符号地址
- A 为修正值（addend）
- P 为当前指令的 PC 值

因此，需要加上 addend 作为修正项。

![image](https://github.com/user-attachments/assets/a8dad7de-d554-47f2-af01-3226a797a8c8)
