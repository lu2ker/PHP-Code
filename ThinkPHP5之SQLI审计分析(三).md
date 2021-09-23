# ThinkPHP 之 SQLI审计分析(三)

**Time：9-23**

**影响版本：ThinkPHP=5.1.22**

**Payload：**

```
/public/index.php/index/index?orderby[id`|updatexml(1,concat(0x7,user(),0x7e),1)%23]=1
```

这是一篇由已知漏洞寻找利用过程的文章，跟着**[参考链接](https://github.com/hongriSec/PHP-Audit-Labs/blob/master/Part2/ThinkPHP5/ThinkPHP5%E6%BC%8F%E6%B4%9E%E5%88%86%E6%9E%90%E4%B9%8BSQL%E6%B3%A8%E5%85%A55.md)**学习分析。以下是收获记录。

感谢七月火的高质量文章。

---

### 0x00 测试代码做了什么

```php
<?php
namespace app\index\controller;

class Index
{
    public function index()
    {
        $orderby = request()->get('orderby');
        $result = db('users')->where(['username' => 'mochazz'])->order($orderby)->find();
        var_dump($result);
    }
}
```

以GET方式从`orderby`中获取参数值，查询`users`表中`username=mochazz`，再用order方法添加排序语句最后find获取结果。

`request`类是在`thinkphp/library/think/Request.php`中定义的，其get方法：

![image-20210923105334245](ThinkPHP5之SQLI审计分析(三)_images/image-20210923105334245.png)

并没有什么比较关键的东西，之前在研究`（username/a）`的`a`是什么意思时，对`get`到`input`方法的调用处理流程也分析过，最终的过滤代码是在`thinkphp/library/think/Request.php`的**1408行**的`filterValue`函数，但是由于仅仅是一个框架，并没有添加自定义的过滤器进行过滤。所以这边基本上是传入什么值就获取什么值。未来如果没有特殊情况就不用分析测试代码的作用了。

### 0x01 分析调用链

- where()方法

emm，这个版本的`parseWhereExp`方法与之前分析过的5.0.10版本的不太一样，所以还是学习一下吧。

![image-20210923112437413](ThinkPHP5之SQLI审计分析(三)_images/image-20210923112437413.png)

**1474-1477行**像是修了个小BUG的样子，之所以会这么想是因为getOptions方法是这么写的：

![image-20210923112620567](ThinkPHP5之SQLI审计分析(三)_images/image-20210923112620567.png)

总的来说感觉大概是防止为空时出错吧。不重要。

但有了5.0.10版本分析的经验，知道`parseWhereExp`方法中主要看的是if-else语句块。这段代码如下：

![image-20210923113247824](ThinkPHP5之SQLI审计分析(三)_images/image-20210923113247824.png)

因为`$field`是一个数组（`$field`就是测试代码中`where`中的参数值）所以直接跳到**1495行**看起，进入`parseArrayWhereItems`方法。并且，在`parseArrayWhereItems`处理完后就会直接return了。

![image-20210923114535150](ThinkPHP5之SQLI审计分析(三)_images/image-20210923114535150.png)

**1564行**`if`条件满足，foreach拆分`username`和`mochazz`，之后会进入**1571行**的else中对`$where`数组进行赋值，这里三元运算因为mochazz不是数组，所以第二个元素最终为'='，然后到**1580行**的if，也是简单的取值赋值操作。

where方法分析完毕，好像也没什么不同。区别在于5.0.10中没有看`parseArrayWhereItems`方法，这次补上了。下断点看一下处理后的最终结果：

![image-20210923120020929](ThinkPHP5之SQLI审计分析(三)_images/image-20210923120020929.png)

- order()方法，本次的主角

`thinkphp/library/think/db/Query.php`的**1823-1864行**，因为测试代码给定参数的特殊性，程序不会进入部分代码，也为了舍去不必要的篇幅，直接来到**1853行**开始看起：

![image-20210923120807266](ThinkPHP5之SQLI审计分析(三)_images/image-20210923120807266.png)

**1854行**，既然`where`在`options`中是个数组，那么`order`在`options`也应是个数组，很合理。

**1857行**，判断为数组，之后也是用array_merge进行简单的赋值操作。还是下断点看看执行完的情况：

![image-20210923124524845](ThinkPHP5之SQLI审计分析(三)_images/image-20210923124524845.png)

自始至终没有任何有效的过滤存在意味着`where`和`order`值都是可控的。但是具体怎么触发漏洞还是没有个清晰的逻辑，往后看find的执行流程吧。

。。。

实际上，也没什么看头，基本上是在解析`$this->options`中的值，然后在**3041行**调用另一处的`find`方法，而后执行的代码与前几次分析SQLI漏洞的流程基本一致。em先下个断点看一下SQL语句构造完成后是什么：
最终在`thinkphp/library/think/db/Connection.php`中`find`方法的**826行**生成的SQL语句。

![image-20210923123045070](ThinkPHP5之SQLI审计分析(三)_images/image-20210923123045070.png)

明显可以看到是通过反引号闭合id前一个反引号导致的注入攻击。

既然是order by之后的注入，那么漏洞点一定和`select`方法中调用的`parseOrder`这个方法有关系：

有了以往的经验，知道具体代码都在`thinkphp/library/think/db/Builder.php`中，来到**847行**的`parseOrder`

![image-20210923123938698](ThinkPHP5之SQLI审计分析(三)_images/image-20210923123938698.png)

阅读**855-871行**，暂时不看程序不会进入的代码直接来到最后一个else，**863-869**都没有什么特殊的地方，870会把`$sort`置为空，**871行**的`parseKey`方法很关键，跟进。

![image-20210923125813142](ThinkPHP5之SQLI审计分析(三)_images/image-20210923125813142.png)

它的作用是获取key的值。但是下断点步进发现，调用的实际上不是Builder.php中的`parseKey`而是Mysql.php中的`parseKey`，是因为自始至终Mysql都是extends于Builder的，前面一直在Builder中分析代码是因为Mysql中没有写相应的函数，自然就向父类查找了，但是与以往不同的是，`parseKey`是在Mysql中定义好的。（涨记性，有经验了也得根据实际情况来。）

来到实际执行的`parseKey`：

`thinkphp/library/think/db/builder/Mysql.php`的**113行**：

![image-20210923131035953](ThinkPHP5之SQLI审计分析(三)_images/image-20210923131035953.png)

![image-20210923131104960](ThinkPHP5之SQLI审计分析(三)_images/image-20210923131104960.png)

已知的payload并不满足**123行**的if，直接会到**143行**处的if，因为`$strict`是写死的`true`，这个if必然会进入，然后执行的操作是往`$key`的两边<u>直接拼接</u>加反引号，这个时候想到payload的形式，毫无疑问**漏洞点**就在这里。

最后，将return的`$key`拼接上`$sort`，再在`parseOrder`中的875行拼接前缀`ORDER BY`，SQL注入就构造好了。

### 0x03 总结

这次分析并没有那么细，也没有想太多，只是根据payload和测试代码的流程过了一遍。并不是严格意义上的白盒审计，很多程序没有进入的代码并没有认真去分析，不知道其他代码做了什么就很可能还存在漏洞点。还是一样的感觉：目前所学习的代码审计就是一个跳过某段代码和进入某段代码的过程，也就是所谓的“链”。至于绕过过滤这些，感觉不是代码审计的重点，只能算是一种奇巧的思路吧。

加油。