# ThinkPHP3.2.3_SQLi

刚好有朋友在测一个目标是tp3.2.3框架的站遇到了一些问题

顺手跟一下流程复现一下吧。

## 0x00 测试代码

```php
<?php
namespace Home\Controller;
use Think\Controller;
class IndexController extends Controller {
    public function index(){
        $id = $_GET['id'];
        $Admin = D("admin");
        $data = $Admin->where(['Id' => $id])->limit('2')->select();
        var_dump($data);
    }
}
```

## 0x02 paylaod

```
?id[0]=exp&id[1]==1 UNION ALL SELECT 1,2,3
```

通过paylaod和很久以前的经验猜测，这个漏洞应该也是~~代码中检查`exp`的时候多加了个空格导致无效了，数组`id`应该会是绕过了某些地方如`if(is_string($id)){xxxxx}`这样的判断~~（我居然在用tp5的经验猜tp3的洞，nt），两个等号的存在肯定是因为要 取的值是`=1 union xxxxx`而不是`1 union xxxx`，代码不给字段名后面加`=`，必须得自带一个`=`拼接到字段名后面。

想象完毕，开始实践分析

## 0x03 调用分析

看paylaod的样子，自然想到最后漏洞点应该是分析处理where的方法。

-   首先进入**ThinkPHP\Library\Think\Model.class.php#1798**

![image-20220403172700369](https://user-images.githubusercontent.com/68197734/161423938-4b0060dd-09a8-445d-9266-b214b533191f.png)

因为传入的`$where`是个数组，前面的所有条件都不满足，就直接返回了.....

limit方法不重要，这个洞又不是limit之后的注入，所以就走到测试代码中的`select()`方法了

-   **ThinkPHP\Mode\Lite\Model.class.php#509**

![image-20220403173950040](https://user-images.githubusercontent.com/68197734/161423946-c87c6531-7a19-4f02-ab59-08e785b3a683.png)

注意，因为测试代码中并没有给select传参，所以函数体内的`$options`也是一个数组，而且是一个空数组。540-571的三个条件判断都不满足，然后就走到`db->select()`方法，看代码看多了就知道这个db的出现，就意味着接下来就要走到核心的数据库操作代码了。

-   **在ThinkPHP\Library\Think\Db\Driver.class.php#943**

![image-20220403174501934](https://user-images.githubusercontent.com/68197734/161423953-930d2065-505c-4d79-8e38-727659c7feb7.png)

开始进行SQL语句的生成

-   **在ThinkPHP\Library\Think\Db\Driver.class.php#965**

![image-20220403174545292](https://user-images.githubusercontent.com/68197734/161423975-c23658d1-fd58-4316-8f1b-f8dd80e44b7d.png)

马上进入SQL语句的解析方法。

-   **ThinkPHP\Library\Think\Db\Driver.class.php#975**

![image-20220403174903090](https://user-images.githubusercontent.com/68197734/161423983-7d699d68-ecdc-4be8-87cc-df33943f01bc.png)

到了这里，别忘了我们刚开始提到的

![image-20220403174938067](https://user-images.githubusercontent.com/68197734/161423991-bd3a9498-a587-4fab-9c6e-499f83be6425.png)

所以说，造成漏洞的起点，就从983行的`parseWhere()`方法开始了。。

## 0x04 漏洞成因

直接略过上图979-982行的一些解析方法，进入到`parseWhere()`，注意传入给`parseWhere()`的参数是`$options['where']`项的值，本文虽然没有关注这个$options变量，但是也要清楚知道它的where项的值就是测试代码中`where()`中传入的。也就是

![image-20220403180019806](https://user-images.githubusercontent.com/68197734/161423999-9f79683b-e3e7-4d0f-aa53-8e37b6768781.png)

-   **ThinkPHP\Library\Think\Db\Driver.class.php#490**

![image-20220403175615201](https://user-images.githubusercontent.com/68197734/161424008-586a9ae2-e635-4c7d-8f71-4174b1d7978d.png)

我们刚才就知道了，`$where`是一个数组，所以503行之前的作用就是给`$operate`设置为了` ' AND '`这个默认的逻辑符。接着该504行开始正式解析`$where`数组了.

-   **ThinkPHP\Library\Think\Db\Driver.class.php#504**

![image-20220403180430205](https://user-images.githubusercontent.com/68197734/161424018-c56ab1b5-604e-49c3-85a5-1e45d527de77.png)

看看，where变量分成了key和value，那么key就是id（别忘了id是一个数组的名字哦），value是一个有两个元素的数组，分别为`exp`和`"=1 union select 1,2"`。前面的if都不满足。走到536行，就到了进入最深处的时候了。

-   **ThinkPHP\Library\Think\Db\Driver.class.php#547**---->`protected function parseWhereItem()`

这里，是解析where传入数据的最核心的功能代码。如果`parseWhereItem`能返回一个含有恶意sql语句的字符串，那么基本上就能确定是一个SQLi。

![image-20220403181227746](https://user-images.githubusercontent.com/68197734/161424024-6be114ea-f615-48a8-a8e3-6e27e58f3a19.png)

从549行开始，`$val`是一个数组。进入这个if块，`$val[0]`是字符串也没问题，然后给`$exp`设置为了`$val[0]`也就是`exp`。然后就到569行的`elseif('exp' == $exp )`满足条件，简单拼接上`$val[1]`，也就是paylaod 中最关键的放置恶意sql语句的地方。最后的`$whereStr`即为（Id =1 union select 1,2）直接return了。

## 0x05 总结

虽然是跟完了，但是如果能想一想这开发者为什么会这样写的话，会让我们对程序的代码逻辑更加亲切，也是一种提高审计能力的方法。我想：给where处理写一个解析数组的功能，肯定是为了迎合多个条件的where的情况，但是这里因为测试代码中只传入了一个条件所以没办法从这个漏洞跟一遍它的完整功能，但是看对$exp进行的相关操作，开发者的原意应该是可以让“输入”去控制where条件中的判断逻辑符，比如`>, <, !=, like, notlike`，然后用之前提到的`$operate`这个参数来拼接每一个判断式子，完整的效果应该类似于`where Id=1 AND name like 'admin' AND age != '14' `这个伢子。

而挖掘这个漏洞的人，应该就是从最后这个函数起步，关注`$whereStr`的处理过程，发现了一处简单拼接就return的语句，然后确定`$val`变量的来源，找怎么样能让`$val`是一个数组，往上分析即可。

PS：这个洞很简单。这么多篇幅有点不值。
