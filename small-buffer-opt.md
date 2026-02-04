- 先给结论：libc++默认开辟24字节string；libstdc++默认开辟32字节; 而且两者对于将字符串数据放到栈上还是堆上策略也不同。
- libstdc++
```
/home/luofeng/src/BiShengCompiler-5.1.0-aarch64-linux/bin/clang++ demo.cpp -g -O2 -o demo   --stdlib=libstdc++

./demo
Size of string = 32
Characters: 0Address: 0xffffff284db8
Characters: 1Address: 0xffffff284db8
Characters: 2Address: 0xffffff284db8
Characters: 3Address: 0xffffff284db8
Characters: 4Address: 0xffffff284db8
Characters: 5Address: 0xffffff284db8
Characters: 6Address: 0xffffff284db8
Characters: 7Address: 0xffffff284db8
Characters: 8Address: 0xffffff284db8
Characters: 9Address: 0xffffff284db8
Characters: 10Address: 0xffffff284db8
Characters: 11Address: 0xffffff284db8
Characters: 12Address: 0xffffff284db8
Characters: 13Address: 0xffffff284db8
Characters: 14Address: 0xffffff284db8
Characters: 15Address: 0xffffff284db8 <== 栈上
Allocating 17bytes
Characters: 16Address: 0xaaaaf54e6280 <== 堆上，低地址
Allocating 18bytes
Characters: 17Address: 0xaaaaf54e6280
Allocating 19bytes
Characters: 18Address: 0xaaaaf54e6280
Allocating 20bytes
Characters: 19Address: 0xaaaaf54e6280
Allocating 21bytes
Characters: 20Address: 0xaaaaf54e6280
Allocating 22bytes
Characters: 21Address: 0xaaaaf54e6280
Allocating 23bytes
Characters: 22Address: 0xaaaaf54e6280
Allocating 24bytes
Characters: 23Address: 0xaaaaf54e6280
Allocating 25bytes
Characters: 24Address: 0xaaaaf54e62a0
Allocating 26bytes
Characters: 25Address: 0xaaaaf54e62a0
Allocating 27bytes
Characters: 26Address: 0xaaaaf54e62a0
Allocating 28bytes
Characters: 27Address: 0xaaaaf54e62a0
Allocating 29bytes
Characters: 28Address: 0xaaaaf54e62a0
Allocating 30bytes
Characters: 29Address: 0xaaaaf54e62a0
Allocating 31bytes
Characters: 30Address: 0xaaaaf54e62a0
Allocating 32bytes
Characters: 31Address: 0xaaaaf54e62a0
```
- libc++
```
clang++ demo.cpp -g -O2 -o demo   --stdlib=libc++

export LD_LIBRARY_PATH=/llvm/lib/:$LD_LIBRARY_PATH

ldd demo
	linux-vdso.so.1 (0x0000ffff9a84c000)
	libc++.so.1 => /home/luofeng/src/BiShengCompiler-5.1.0-aarch64-linux/lib/libc++.so.1 (0x0000ffff9a6d5000)
	libc++abi.so.1 => /home/luofeng/src/BiShengCompiler-5.1.0-aarch64-linux/lib/libc++abi.so.1 (0x0000ffff9a684000)
	libunwind.so.1 => /home/luofeng/src/BiShengCompiler-5.1.0-aarch64-linux/lib/libunwind.so.1 (0x0000ffff9a663000)
	libm.so.6 => /lib64/libm.so.6 (0x0000ffff9a592000)
	libgcc_s.so.1 => /lib64/libgcc_s.so.1 (0x0000ffff9a561000)
	libc.so.6 => /lib64/libc.so.6 (0x0000ffff9a3c9000)
	libatomic.so.1 => /lib64/libatomic.so.1 (0x0000ffff9a3a7000)
	libdl.so.2 => /lib64/libdl.so.2 (0x0000ffff9a386000)
	libpthread.so.0 => /lib64/libpthread.so.0 (0x0000ffff9a351000)
	/lib/ld-linux-aarch64.so.1 (0x0000ffff9a80e000)

./demo
Size of string = 24
Characters: 0Address: 0xfffff278f911
Characters: 1Address: 0xfffff278f911
Characters: 2Address: 0xfffff278f911
Characters: 3Address: 0xfffff278f911
Characters: 4Address: 0xfffff278f911
Characters: 5Address: 0xfffff278f911
Characters: 6Address: 0xfffff278f911
Characters: 7Address: 0xfffff278f911
Characters: 8Address: 0xfffff278f911
Characters: 9Address: 0xfffff278f911
Characters: 10Address: 0xfffff278f911
Characters: 11Address: 0xfffff278f911
Characters: 12Address: 0xfffff278f911
Characters: 13Address: 0xfffff278f911
Characters: 14Address: 0xfffff278f911
Characters: 15Address: 0xfffff278f911
Characters: 16Address: 0xfffff278f911
Characters: 17Address: 0xfffff278f911
Characters: 18Address: 0xfffff278f911
Characters: 19Address: 0xfffff278f911
Characters: 20Address: 0xfffff278f911
Characters: 21Address: 0xfffff278f911
Characters: 22Address: 0xfffff278f911
Allocating 26bytes
Characters: 23Address: 0xaaaadbaca670
Allocating 32bytes
Characters: 24Address: 0xaaaadbaca670
Allocating 32bytes
Characters: 25Address: 0xaaaadbaca670
Allocating 32bytes
Characters: 26Address: 0xaaaadbaca670
Allocating 32bytes
Characters: 27Address: 0xaaaadbaca670
Allocating 32bytes
Characters: 28Address: 0xaaaadbaca670
Allocating 32bytes
Characters: 29Address: 0xaaaadbaca670
Allocating 32bytes
Characters: 30Address: 0xaaaadbaca670
Allocating 32bytes
Characters: 31Address: 0xaaaadbaca670
```

#### 附件
- https://godbolt.org/z/xrn6Po74v
