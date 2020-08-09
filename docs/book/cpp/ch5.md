# 第五章：循环和关系表达式

前缀 `for (int i = 0; i < n; ++i)` 和后缀 `for (int i = 0; i < n; i++)` 的区别？

> 从逻辑上二者并没有任何区别。但是在实际使用中，前缀要比后缀效率更高，原因是前缀直接加一，而后缀需要提前复制保留一个副本然后加一在返回副本。

注意 ++ 的运算符要高于 * 。

> 例如：x = *pt++ 实际上等价于 x = *(pt++) 。

`while(name[i] != '\0')` 等价于 `while(name[i])`。

> 因为 name[i] 存在字符，其编码为非零值或则 true ，所以循环条件成立，反之当遇到空值字符编码值为  0 或 flase，而且后者不用判断速度更快。