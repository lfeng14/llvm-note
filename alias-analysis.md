- 默认两个相同引用类型时，编译器出于保守策略，会认为是同一别名，表现在重复的load；
- 若类型不同，则认为是不同的别名的，不会重复load：
<img width="2434" height="406" alt="image" src="https://github.com/user-attachments/assets/075fc022-e901-42e0-89e2-b255467457b0" />

- 可使用属性__restrict显式告诉编译器变量不是同一个别名

### 附件
- https://godbolt.org/z/Pchn6vnd1
