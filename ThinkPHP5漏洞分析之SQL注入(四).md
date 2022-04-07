# ThinkPHP5漏洞分析之SQL注入(四).md

>   本次漏洞存在于所有 **Mysql** 聚合函数相关方法。由于程序没有对数据进行很好的过滤，直接将数据拼接进 **SQL** 语句，最终导致 **SQL注入漏洞** 的产生。漏洞影响版本： **5.0.0<=ThinkPHP<=5.0.21** 、 **5.1.3<=ThinkPHP5<=5.1.25** 。
>
>   不同版本 **payload** 需稍作调整：
>
>    **5.0.0~5.0.21** 、 **5.1.3～5.1.10** ： **id)%2bupdatexml(1,concat(0x7,user(),0x7e),1) from users%23**
>
>   **5.1.11～5.1.25** ： **id`)%2bupdatexml(1,concat(0x7,user(),0x7e),1) from users%23**
>
>   取自：https://github.com/Mochazz/ThinkPHP-Vuln/blob/master/ThinkPHP5/ThinkPHP5%E6%BC%8F%E6%B4%9E%E5%88%86%E6%9E%90%E4%B9%8BSQL%E6%B3%A8%E5%85%A56.md
>
>   想便捷获取漏洞环境也可参考上述链接

## 0x00 前言

代码分析，重在一个思考。写代码分析流程，重在记录思考。

共勉。

## 0x01 测试代码

```php
<?php
namespace app\index\controller;
class Index
{
    public function index()
    {
        $options = request()->get('options');
        $result = db('users')->max($options);
        var_dump($result);
    }
}
```

很明显，这次的主角是max聚合函数，与之前几个sqli漏洞不同的是它好像并没有调用select、update、insert、delete这种的（应该可以叫做SQL的关键字吧？）方法，也没有调用where方法。所以就在想，max在SQL语句中是做什么用途的？一个例子：

```
SELECT MAX(id) FROM world；
//从world表中取出id字段里最大的那个 值值值值值值值
```

依据这个结构，配合payload的样子不难发现，问题应该出用户可控的数据在max方法没有过滤的处理后直接拼接到了一个CRUD关键字后面

## 0x02 代码分析

-   首先要先看一下max方法的实现thinkphp\library\think\db\Query.php#697

```php
public function max($field, $force = true)
    {
        return $this->aggregate('MAX', $field, $force);
    }
```

注意到$force默认为true。它直接调用了aggregate方法，并传入了一个MAX字符串，估计是用于拼接的。

-   跟着去看一下aggregate的实现thinkphp\library\think\db\Query.php#619

```php
    public function aggregate($aggregate, $field, $force = false)
    {
        $this->parseOptions();
        $result = $this->connection->aggregate($this, $aggregate, $field);
        if (!empty($this->options['fetch_sql'])) {
            return $result;
        } elseif ($force) {
            $result = (float) $result;
        }
        // 查询完成后清空聚合字段信息
        $this->removeOption('field');
        return $result;
    }
```

怎么说呢，看到先执行了parseOptions这个方法，但是是用$this调用的，也就是整个Query类的‘自己’。这里简单看了一下parseOptions方法，应该是一些字段值的初始化置空的操作。不重要。

然后就是$this->connection->aggregate()了，connection代表着一个数据库连接对象，它的aggregate方法估计就是tp最底层的aggregate（译做聚合吧好理解一点）方法了。目前我们注意到，$field这个我们可控的变量还没有做什么实质性的处理。

进入到$this->connection->aggregate()

thinkphp\library\think\db\Connection.php#1315

```php
public function aggregate(Query $query, $aggregate, $field)
    {
        $field = $aggregate . '(' . $this->builder->parseKey($query, $field, true) . ') AS tp_' . strtolower($aggregate);
        return $this->value($query, $field, 0);
    }
```

 $aggregate此时是字符串“MAX”，这里还调用了builder的解析函数parseKey去解析我们可控的$field，看看它做了啥。000000看了下它啥也没做，解析解析，就仅仅判断了一下是不是个表达式而已。没有用。代码如下：

```php
public function parseKey(Query $query, $key, $strict = false)
    {
        return $key instanceof Expression ? $key->getValue() : $key;
    }
```

这就基本上是整个调用流程了，从parseKey返回之后，就会将可控字符串返回到$this->connection->aggregate()里拼接一些不重要的字符，（不重要是因为我们想看的只是输入是不是完全可控。）return时调用了本类的value方法，这个value方法实际上就是从数据库里面获取值的，所以它的代码就是生成sql语句和执行sql语句了。同时，因为这个漏洞不需要where、table等等等sql语句中常见的部分，生成SQL语句的时候也不会去进行这种特定的解析方法。最后就导致含有可控字符串的完整的select语句被下文执行。造成sql注入。

不过报错注入的话，再执行的时候就会报错了。但是程序还会将获取到的数据返回到Query->aggregate()方法中的`$result`，，如果这个数据会被打印的话：

（明天再看）