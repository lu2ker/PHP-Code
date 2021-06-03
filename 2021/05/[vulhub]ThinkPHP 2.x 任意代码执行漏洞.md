# [vulhub]TP2.x 任意代码执行复现

time：

---

### 0x01 关键函数preg_replace()

preg_replace()函数的作用是对字符串进行处理，执行一个正则表达式的搜索和替换。定义如下：

```php
preg_replace( mixed $pattern, mixed $replacement, mixed $subject[, int $limit = -1[, int &$count]] ) : mixed
    
#搜索subject中匹配pattern的部分，以replacement进行替换。
```

在**php5.5.0之前**，当$pattern处存在修饰符 e 时，$replacement的值会被当成php代码来执行。

而自5.5.0之后，传入 "\e" 修饰符的时候，会产生一个    **`E_DEPRECATED`** 错误； PHP 7.0.0 起，会产生    **`E_WARNING`** 错误，同时  "\e" 也无法起效。 

（这里用php5.4或者php5.3举个例子）

### 0x02 TP2.x产生RCE的原理

thinkphp框架的GET参数以index.php/a/b/c的形式传递，程序获取参数之前需要先解析URL，漏洞就发生在解析URL的地方，

