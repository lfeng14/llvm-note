- 有些场景编译器会优化返回值；有些场景则优化不了；返回值拷贝到rdi指向内存
- 可以优化场景：
- 优化不了场景：
- [示例](https://godbolt.org/z/frb7GMrKs)
```
struct S {
    int data[6]; # data[1]
};

S getS(int cond) {
    S s1{};
    S s2{};
    return cond ? s1 :s2;
}
```
- 汇编
```
getS(int):
        mov     rax, rdi
        pxor    xmm0, xmm0
        movaps  XMMWORD PTR [rsp-40], xmm0
        mov     QWORD PTR [rsp-24], 0
        movaps  XMMWORD PTR [rsp-72], xmm0
        mov     QWORD PTR [rsp-56], 0
        lea     rcx, [rsp-40]
        lea     rdx, [rsp-72]
        test    esi, esi
        cmovne  rdx, rcx
        movdqa  xmm1, XMMWORD PTR [rdx] # XMMWORD双四字
        movups  XMMWORD PTR [rdi], xmm1
        mov     rdx, QWORD PTR [rdx+16]
        mov     QWORD PTR [rdi+16], rdx
        ret

```
