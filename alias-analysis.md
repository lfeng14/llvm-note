- 类型不同，会认为是不同的别名的，影响后续优化：
<img width="2434" height="406" alt="image" src="https://github.com/user-attachments/assets/075fc022-e901-42e0-89e2-b255467457b0" />

- 使用属性__restrict显式告诉编译器变量不是同一个别名

### 附件
- https://godbolt.org/z/Pchn6vnd1
