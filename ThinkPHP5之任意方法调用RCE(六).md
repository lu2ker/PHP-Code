## [书接上文](https://github.com/lu2ker/PHP-Code/blob/main/TP5%E6%A1%86%E6%9E%B6%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%E8%BF%9B%E9%98%B6%E4%B9%8BRequest.md)
1.   利用`method`的任意方法调用，调用构造函数`__construct`，且调用时会传入`$_POST`数据，那么组合起来就是执行method方法可以控制类的成员变量的值。

2.   `filterValue`中存在 `call_user_func`函数可以恶意利用，关注其参数`$filters`是否可控。

3.   `getFilter`方法存在这样`$filter = $filter ?: $this->filter;`一条语句获取类成员变量filter（由1知可控）。
4.   `input`方法中存在：先调用`getFilter`获取到filter，再使用`array_walk_recursive`递归调用`filterValue`并传入filter的情况。

综上，如果找到一处先调用`method`再调用`input`的调用点，就能够利用`call_user_func`造成RCE。

----

### method方法调用链：

(调用时必须要$this->method为空或false)

#### 程序启动时：

-   run()::thinkphp\library\think\App.php#116->routeCheck
-   routeCheck::thinkphp\library\think\App.php#639->check
-   check::thinkphp\library\think\Route.php#848->$request->method()

在开启debug模式时，这条method调用链会在调用任意方法后，执行$request->param()方法，而这个方法里就有调用input()。然后就有：

-   调用`__construct`，需传入两个参数filter[]=system（filter=system也行）和xxx=whoami即可

![image-20220416131554673](https://user-images.githubusercontent.com/68197734/163669227-d349becd-0195-4088-a76e-89e7609c20b2.png)

-   不调用`__construct`，直接调用`filter`方法直接设置过滤器

![image-20220415174257578](https://user-images.githubusercontent.com/68197734/163669230-1058fffa-66ca-4d7d-a578-d04f926733f5.png)

注意第二种payload必须要按照上面的顺序，不然filterValue会进入我们不期望的流程，==直接break了。==

**为什么第一种payload不需要考虑传参顺序呢？**

因为两种payload注册过滤器的方法不一样：

1.   构造函数注册过滤器使用类属性覆盖，在传入的参数中只有类中之前声明过的filter会被覆盖掉（因为有`property_exists($this, $name)`）。
2.   而filter方法注册过滤器的时候是直接将传入的`$_POST`赋值给`$this->filter`导致filterValue遍历取过滤器名的时候，不仅仅是可以通过is_callable判断的system，比如上图例子中的filter传入后会经过is_callable('filter')，结果为false。


### 后门技巧

1.   允许'app_debug'  => false

     修改application\config.php#default_filter

     设置默认过滤器  system，直接请求，参数名xx=whoami即可执行命令（业务影响未知，大概率影响）

2.   5.x的RCE漏洞修复较简单，只是用if校验了一下方法白名单。直接删掉不影响业务。

3.   手写一个有漏洞的控制器代码。可以利用TP自带的很多处回调函数比如Request::filterValue中的，怎么写可以参考P牛[老文](https://www.leavesongs.com/PENETRATION/php-callback-backdoor.html)。也是不影响业务。

4.   ...

---

不开debug也能调用到input方法。

-   [路由报错](https://xz.aliyun.com/t/11189#toc-3)：[ThinkPHP V5.0.22](https://github.com/top-think/framework/releases/tag/v5.0.22) 改进Log类支持`json`日志格式，添加了parseLog方法。该方法的处理过程会调用到filterValue。（需要构造错误，例如在route.php中不use think\Route 直接写路由规则）


```
调用堆栈:
think\Request->filterValue
think\Request->input
think\Request->server
think\Request->host
think\log\driver\File->parseLog 
think\log\driver\File->write 
think\log\driver\File->save 
think\Log::save
think\Error::appShutdown
```

