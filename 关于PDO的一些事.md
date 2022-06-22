PDO主要是这个选项

```php
PDO::ATTR_EMULATE_PREPARES  => false
```

![image-20220622131618003](https://user-images.githubusercontent.com/68197734/174957301-42de2cc2-6dc0-4727-8630-7fd6495346e0.png)

这个选项涉及到PDO的“预处理”机制：因为不是所有数据库驱动都支持SQL预编译，所以PDO存在“模拟预处理机制”。如果说开启了模拟预处理，那么PDO内部会模拟参数绑定的过程，SQL语句是在最后`execute()`的时候才发送给数据库执行；如果我这里设置了`PDO::ATTR_EMULATE_PREPARES => false`，那么PDO不会模拟预处理，参数化绑定的整个过程都是和Mysql交互进行的。

非模拟预处理的情况下，参数化绑定过程分两步：第一步是prepare阶段，发送带有占位符的sql语句到mysql服务器（parsing->resolution），第二步是通过execute()函数多次发送占位符参数给mysql服务器进行执行（多次执行optimization->execution）。

这时，假设在第一步执行`prepare($SQL)`的时候我的SQL语句就出现错误了，那么就会直接由mysql那边抛出异常，不会再执行第二步。[（From phith0n）](https://www.leavesongs.com/PENETRATION/thinkphp5-in-sqlinjection.html)



PDO默认支持多语句查询，如果php版本小于5.5.21或者创建PDO实例时未设置`PDO::MYSQL_ATTR_MULTI_STATEMENTS`为false时可能会造成堆叠注入[（From haby0）](https://xz.aliyun.com/t/3950)

![image-20220622101510813](https://user-images.githubusercontent.com/68197734/174957319-02c7f511-10bd-4636-8498-1a0eef7bceba.png)

翻译过来就是当它设置为False时PDO::prepare() 和 PDO::query() 就会禁用多查询，

PDO的配置放在实例化时的第四个参数位置，类型为数组。

```php
new PDO($dsn, $user, $pass, array( PDO::MYSQL_ATTR_MULTI_STATEMENTS => false))
```

_PS：有个小问题，非模拟预处理下PDO::prepare()才会和数据库交互，而非模拟预处理是不存在堆叠注入可能性的，官方手册里却还写上了这是为啥？感觉这里应该写成PDO::execute()才对。_

## 总结

-   并不是说使用了PDO就是安全的
-   pdo的模拟预处理和非模拟预处理
    -   模拟预处理：在PDO内部模拟参数绑定的过程。`PDO::ATTR_EMULATE_PREPARES  => true`
    -   非模拟预处理：参数化绑定的整个过程都与数据库交互，就是交给数据库去预编译，`PDO::ATTR_EMULATE_PREPARES  => false`
    -   认为可以理解为模拟预处理和预处理....
-   可控变量直接拼接的前提下：
    -   模拟预处理下，execute阶段会造成注入，可以使用堆叠注入
    -   非模拟预处理下，不存在堆叠注入，会在prepare阶段报错，但还可以用其他注入类型（毕竟是直接拼接了）。
    -   比如既然会在prepare阶段报错，那么理所应当可以使用报错注入，但是要回显错误信息的话还需要设置`PDO::ATTR_ERRMODE => PDO::ERRMODE_EXCEPTION`
-   综上再copy一下[haby0](https://xz.aliyun.com/t/3950)给的安全建议：
    -   使用PDO时尽量使用非模拟预处理
    -   PDO最好配置`PDO::MYSQL_ATTR_MULTI_STATEMENTS => false`，禁止多语句查询(默认是true)
    -   不要直接拼接可控变量，占位符多好啊

案例的话，去看提到的两位师傅的文章吧。
