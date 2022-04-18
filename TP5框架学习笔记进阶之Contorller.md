很烂，不建议看。建议略过
## Controller

大多是一些模板渲染的方法。从构造函数中就可以看出来，

```php
public function __construct(Request $request = null)
    {
        $this->view    = View::instance(Config::get('template'), Config::get('view_replace_str'));
        $this->request = is_null($request) ? Request::instance() : $request;

        // 控制器初始化
        $this->_initialize();

        // 前置操作方法
        if ($this->beforeActionList) {
            foreach ($this->beforeActionList as $method => $options) {
                is_numeric($method) ?
                $this->beforeAction($options) :
                $this->beforeAction($method, $options);
            }
        }
    }
```

刚开始就实例化了View类。想到之前复现的ThinkPHP的SQL注入漏洞，有些测试代码就会调用fetch、assign、display这些方法。而模块开发的开始就是从 application\模块\controller 下面开始的。应用模块的控制器如果继承了Controller类，就能直接调用这些方法进行视图渲染。

这个类还有两个方法：beforeAction和validate

beforeAction是[前置操作](https://www.kancloud.cn/manual/thinkphp5/118050)：设置 `beforeActionList`属性可以指定某个方法为其他方法的前置操作，数组键名为需要调用的前置方法名，无值的话为当前控制器下所有方法的前置方法。

```php
['except' => '方法名,方法名']
#表示这些方法不使用前置方法，
['only' => '方法名,方法名']
#表示只有这些方法使用前置方法。

# demo
protected $beforeActionList = [
        'first',
        'second' =>  ['except'=>'hello'],
        'three'  =>  ['only'=>'hello,data'],
    ];

# 类内所有方法调用前都会调用first方法
# 类内hello方法不会调用second这个前置方法
# 只有hello，data方法会前置调用three方法。
```

如果你在自己继承了Controller的控制器代码中定义了`protected $beforeActionList`，那么它就会在构造函数中foreach处理后，调用beforeAction进行only还是except的判断，最后用call_user_func调用。

```php
call_user_func([$this, $method])
```

这种写法是对类内方法的调用，这里的$this指向的是继承Controller类的Index类。也就是说回调的函数必须在Index类中定义好了。

validate验证器方法主要还是调用的Validate类，应该是为了方便控制器对数据的验证。

具体流程就是用 加载验证器 的 加载器，来加载验证器。。。。。

如果传入的$validate是个数组，就先用thinkphp\library\think\Validate.php#rule赋到$this里面。不是数组的话就直接调用。

