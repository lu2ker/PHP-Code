## Request（5.0.18版本）

[前置PHP知识：PHP中self :: 和 this-> 的用法](https://www.cnblogs.com/wenzheshen/articles/9938820.html)

[前置PHP知识：new static 和 new self的区别](https://www.jb51.net/article/106103.htm)  -主要就是有继承的时候的区别

下面只列举部分重要方法

### 初始化和构造函数：instance和__construct

```php
public static function instance($options = [])
    {
        if (is_null(self::$instance)) {
            self::$instance = new static($options);
        }
        return self::$instance;
    }

protected function __construct($options = [])
    {
        foreach ($options as $name => $item) {
            if (property_exists($this, $name)) {		//类属性覆盖
                $this->$name = $item;
            }
        }
        if (is_null($this->filter)) {
            $this->filter = Config::get('default_filter');		//在应用目录下的config.php
        }
        // 保存 php://input
        $this->input = file_get_contents('php://input');
    }
```

instance就是Request的初始化方法，编程中这种是叫“单例模式”，目的是为了防止实例化的时候构造方法被多次调用，就先new了一下自己然后返回。

property_exists判断$this，也就是本类对象是否存在传入的$name属性，存在即赋值。接着还定义了过滤器，并获取了输入流（请求body）

这样初始化调用：`$req = Request::instance()->get('a');`

### 方法注入：hook和__call

```php
public static function hook($method, $callback = null)
    {
        if (is_array($method)) {
            self::$hook = array_merge(self::$hook, $method);
        } else {
            self::$hook[$method] = $callback;
        }
    }

public function __call($method, $args)
    {
        if (array_key_exists($method, self::$hook)) {
            array_unshift($args, $this);
            return call_user_func_array(self::$hook[$method], $args);
        } else {
            throw new Exception('method not exists:' . __CLASS__ . '->' . $method);
        }
    }
```

hook方法和`__call`魔术方法配合，使得可以在类外面自定义Request的方法。算是扩展性的实现。hook主要是来获取自定义函数的，然后因为这个函数本身不在Request中，所以会调用`__call`方法，检查这个函数是不是在hook数组里面，在的话就说明这是一个要注入的方法，就将Request类本身的$this对象作为自定义函数的参数传入，通过call_user_func_array进行调用。自定义函数使用闭包的写法：

```php
# application\index\controller\Index.php
public function index()
    {
        $req = Request::instance();
        $callback = function($RR){
            echo "匿名函数";
            $RR->test111();
        };
        Request::hook('invoke',$callback);
        $req->invoke();
    }
# thinkphp\library\think\Request.php添加一个如下方法：
public function test111()
    {
        echo "可以调用Request本身的方法";
    }

# 访问index.php/index/输出：匿名函数可以调用Request本身的方法
```

### 当前的请求类型：method

```php
public function method($method = false)
    {
        if (true === $method) {
            // 获取原始请求类型
            return IS_CLI ? 'GET' : (isset($this->server['REQUEST_METHOD']) ? $this->server['REQUEST_METHOD'] : $_SERVER['REQUEST_METHOD']);
        } elseif (!$this->method) {
            if (isset($_POST[Config::get('var_method')])) {
                $this->method = strtoupper($_POST[Config::get('var_method')]);
                $this->{$this->method}($_POST);
            } elseif (isset($_SERVER['HTTP_X_HTTP_METHOD_OVERRIDE'])) {
                $this->method = strtoupper($_SERVER['HTTP_X_HTTP_METHOD_OVERRIDE']);
            } else {
                $this->method = IS_CLI ? 'GET' : (isset($this->server['REQUEST_METHOD']) ? $this->server['REQUEST_METHOD'] : $_SERVER['REQUEST_METHOD']);
            }
        }
        return $this->method;
    }
```

IS_CLI是用来判断是使用命令行还是浏览器执行的，cli代表命令行。

$this->server和$_SERVER的区别*应该*是：前者提供了程序可定义的请求方法（在create方法中），后者是获取客户端请求的 请求方法。

如果调用method时没传入true，分三种情况：

① 如果配置了var_method，从`$_POST[Config::get('var_method')]`，即`$_POST['_method']`获取请求方法。

![image-20220414142214370](https://user-images.githubusercontent.com/68197734/163394294-aa793e54-4329-4cc0-b45d-826ae1bc7806.png)

有意思的是这条语句`$this->{$this->method}($_POST);`，用花括号包裹 先执行取值在以$this调用传入$_POST。

![image-20220414143045047](https://user-images.githubusercontent.com/68197734/163394308-d48d2f4c-f049-42ae-90d1-ab5e1fd4bb08.png)

② 通过请求包的HTTP头字段`X-HTTP-METHOD-OVERRIDE`来设置method

![image-20220414143840123](https://user-images.githubusercontent.com/68197734/163394322-f03b2d46-2e09-4680-96ac-c4a4c9035245.png)

③ 和true === $method时一样

### 获取当前请求的参数：param

该方法主体中是对请求参数的处理：包括针对不同的请求方法，选择调用Request类里不同的处理方法；还有获取文件上传信息的处理。获取post、get、put、file等请求参数的合并数据。但这些处理都不是方法内实现的，而是调用的Request类的其他成员方法。

### 获取GET数据：get

支持传入数组：

$this->get = array_merge($this->get, $name);

### 获取POST数据：post

```php
public function post($name = '', $default = null, $filter = '')
    {
        if (empty($this->post)) {
            $content = $this->input;
            if (empty($_POST) && false !== strpos($this->contentType(), 'application/json')) {
                $this->post = (array) json_decode($content, true);
            } else {
                $this->post = $_POST;
            }
        }
        if (is_array($name)) {
            $this->param       = [];
            return $this->post = array_merge($this->post, $name);
        }
        return $this->input($this->post, $name, $default, $filter);
    }
```

比get特别的地方在于，会从php://input输入流中获取数据，这里的`$content = $this->input;`是在构造函数中初始化的。然后还能用json_decode处理json数据并返回为数组格式的。

### 获取cookie参数：cookie

会调用filterValue方法过滤数据。而且如果cookie是个数组的话还会使用array_walk_recursive递归调用filterValue

### 获取上传的文件信息：file

会实例化File类，@return null|array|\think\File 返回这三种类型的数据。具体获取文件的什么信息再看。

### 被多处调用的方法：input

该方法有一个参数是$filter默认为空，说明支持可设置的过滤器。

处理流程在[ThinkPHP5之SQLI审计分析（一）](https://github.com/lu2ker/PHP-Code/blob/main/ThinkPHP5%E4%B9%8BSQLI%E5%AE%A1%E8%AE%A1%E5%88%86%E6%9E%90(%E4%B8%80).md)里面看过了。

### 递归过滤方法：filterValue

```php
private function filterValue(&$value, $key, $filters)
    {
        $default = array_pop($filters);
        foreach ($filters as $filter) {
            if (is_callable($filter)) {
                // 调用函数或者方法过滤
                $value = call_user_func($filter, $value);
            } elseif (is_scalar($value)) {
                if (false !== strpos($filter, '/')) {
                    // 正则过滤
                    if (!preg_match($filter, $value)) {
                        // 匹配不成功返回默认值
                        $value = $default;
                        break;
                    }
                } elseif (!empty($filter)) {
                    // filter函数不存在时, 则使用filter_var进行过滤
                    // filter为非整形值时, 调用filter_id取得过滤id
                    $value = filter_var($value, is_int($filter) ? $filter : filter_id($filter));
                    if (false === $value) {
                        $value = $default;
                        break;
                    }
                }
            }
        }
        return $this->filterExp($value);
    }
```

$filters是数组类型的，说明可以进行多个过滤器的处理。在判断了过滤器可以被调用后，就马上用call_user_func来调用过滤器处理数据了。

方法开头会用array_pop弹出默认过滤器，是在getFilter方法将所有接收到的过滤器放到数组里后，又在数组末尾加的那个$default

---

1.   利用method的任意方法调用，调用构造函数`__construct`，且调用时会传入`$_POST`数据，那么组合起来就是执行method方法可以控制类的成员变量的值。

2.   `filterValue中存在` `call_user_func`函数可以恶意利用，其参数$filters

3.   `getFilter`方法存在这样`$filter = $filter ?: $this->filter;`一条语句获取类成员变量filter（由1知可控）。
4.   `input`方法中存在：先调用`getFilter`获取到filter，再使用`array_walk_recursive`递归调用`filterValue`并传入filter的情况。

综上，如果找到一处先调用`method`再调用`input`且的调用点，就能够利用`call_user_func`造成RCE。

param方法就完美符合条件。
