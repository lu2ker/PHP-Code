# ThinkPHP5之文件包含审计分析(五)

>   参考链接：[Mochazz/ThinkPHP-Vuln/](https://github.com/Mochazz/ThinkPHP-Vuln/blob/master/ThinkPHP5/ThinkPHP5%E6%BC%8F%E6%B4%9E%E5%88%86%E6%9E%90%E4%B9%8B%E6%96%87%E4%BB%B6%E5%8C%85%E5%90%AB7.md)
>
>   影响版本：**5.0.0<=ThinkPHP5<=5.0.18 、5.1.0<=ThinkPHP<=5.1.10**
>
>   测试环境：PHP7.3.4、Mysql5.7.26、TP5.0.18

## 0x00 环境准备

关于tp默认的模板调用路径，全局搜了一下后发现应该是在这里：
**thinkphp\library\think\view\driver\Think.php#41**

```php
public function __construct($config = [])
{
	$this->config = array_merge($this->config, $config);
	if (empty($this->config['view_path'])) {
	$this->config['view_path'] = App::$modulePath . 'view' . DS;
}
	$this->template = new Template($this->config);
}
```

这里的构造函数生成的，默认位置就在application\index\view\。实际上按照习惯也应该是和模型（M），控制器（C）同级的。

之后呢，又在同类下的parseTemplate方法中132行，根据控制器名称使用默认规则（即同名）给`$template`赋值,然后又拼接上默认的模板文件后缀`.html`就好了。:

```php
$template = str_replace('.', DS, $controller) . $depr . (1 == $this->config['auto_rule'] ? Loader::parseName($request->action(true)) : $request->action());
........
return $path . ltrim($template, '/') . '.' . ltrim($this->config['view_suffix'], '.');
```

这就是为什么要创建创建 application/index/view/index/index.html文件

## 0x01 测试代码

```php
<?php
namespace app\index\controller;
use think\Controller;
class Index extends Controller
{
    public function index()
    {
        $this->assign(request()->get());
        return $this->fetch(); // 当前模块/默认视图目录/当前控制器（小写）/当前操作（小写）.html
    }
}
```

开始需要assign方法进行模板变量赋值，然后fetch方法渲染。应该是比较常见的写法。

正式开始之前，还要再public下放一张图片马，模拟已经上传好了。

访问：http://www.tp5018.qwe/index.php/index/index?cacheFile=1.jpg 即可

## 0x02 代码分析

request的get方法不是这次的重点，只是获取get传入的参数。所以直接忽略。首先会来到

**thinkphp\library\think\Controller.php#144**也就是Controller类的assign方法，

并直接调用**thinkphp\library\think\View.php#92**View类的的assign方法

```php
public function assign($name, $value = '')
    {
        if (is_array($name)) {
            $this->data = array_merge($this->data, $name);
        } else {
            $this->data[$name] = $value;
        }
        return $this;
    }
```

首先判断是不是数组，因为tp的request->get()返回的就是一个数组（$data变量），数组内每一个元素存放一个参数的键值对。所以会执行array_merge()方法，array_merge在手册中的解释是将一个或多个数组的单元合并起来，一个数组中的值附加在前一个数组的后面。返回作为结果的数组。

这些还只是正常的数据传递过程，和文件包含还没有什么关系，接着往下看fetch方法的渲染。

-   **在thinkphp\library\think\View.php#148**

进入了View类的fetch方法：其中，我们从158行开始：

```php
try {
            $method = $renderContent ? 'display' : 'fetch';
            // 允许用户自定义模板的字符串替换
            $replace = array_merge($this->replace, $replace, (array) $this->engine->config('tpl_replace_string'));
            $this->engine->config('tpl_replace_string', $replace);
            $this->engine->$method($template, $vars, $config);
        } catch (\Exception $e) {
            ob_end_clean();
            throw $e;
        }
```

这个try，首先给$method设置成了fetch（因为View类的fetch方法默认$renderContent是个false），然后执行的代码的作用就是取来一些渲染视图需要的文件位置，比如：

![image-20220409215551929](https://user-images.githubusercontent.com/68197734/162579100-e2e7c895-6208-4835-999f-f39bc4d155c3.png)

重要的是，注意到`$this->engine->$method($template, $vars, $config);`这条语句，因为$method已经被赋值过了，所以它是会去调用tp模板引擎的fetch方法（在Think类），而Think类下的fetch在最后又调用了template的fetch方法。也就是最后会走到这里：

-   **thinkphp\library\think\Template.php#160**

到这里先停一下，回想一下传入的cacheFile=1.jpg现在是个什么状态，在哪个变量里

1.   刚开始通过requets->get()获取，作为$name传入assign方法
2.   两层assign处理后，由Controller下的assign返回了$this，参数在$this->view->data数组里
3.   然后在View类中的fetch中赋值给了$vars（这里的fetch是第二层，层层深入）
4.   $vars 做为 Think类下fetch方法的$data[]参数传入（这里的fetch是第三层）
5.   $data 做为Template类下fetch方法的$vars[]传入（这里是最后一层fetch，模板引擎的核心）

变量的传递过程理清后来看最重要的代码：

![image-20220409221559355](https://user-images.githubusercontent.com/68197734/162579108-1d5067d4-3b4e-4f49-a4f6-33a6231473ef.png)

首先关注参数，$template在最开始我们就知道它存放的是模板文件的位置，$vars是我们的图片马文件参数，$config在调用过程中，一直是默认空的。

再经过了176行的模板文件解析之后进入了if代码块，首先把自动生存的缓存php文件路径给了$cacheFile变量，接着就调用了storage->read()，并将$data和$cacheFile一起传入了，跟入：

-   **来到thinkphp\library\think\template\driver\File.php#45**

```php
public function read($cacheFile, $vars = [])
    {
        if (!empty($vars) && is_array($vars)) {
            // 模板阵列变量分解成为独立变量
            extract($vars, EXTR_OVERWRITE);
        }
        //载入模版缓存文件
        include $cacheFile;
    }
```

注意到其中调用了extract函数来处理$vars，在官方手册中extract解释如下：

>   本函数用来将变量从数组中导入到当前的符号表中。
>
>   检查每个键名看是否可以作为一个合法的变量名，同时也检查和符号表中已有的变量名的冲突。 

意思就相当于全局是变量注册，可以看[我这里的记录](https://github.com/lu2ker/PHP-Code/blob/main/%E5%8F%AF%E8%83%BD%E8%A2%AB%E5%88%A9%E7%94%A8%E7%9A%84%E5%87%BD%E6%95%B0.md#extract)指出了对用户可控的输入使用该函数可能会造成风险。尤其是第二个参数：默认`EXTR_OVERWRITE`，如果有冲突，覆盖已有的变量。

了解了这个函数，就明白了在URL栏传入的`?cacheFile=1.jpg`实际上就是为了这里的变量覆盖！在变量覆盖之后，程序立马就include了`$cacheFile`，也就是1.jpg图片马。

至此完成文件包含。

## 0x03 总结

这个漏洞个人认为文件包含是小事，因为重点是在最后的变量覆盖。而且那条代码还是一个经典的变量覆盖漏洞案例。如果是从审计来看，定位到extract函数，注意到这里设置了EXTR_OVERWRITE，那么只要第一个参数可控就可以完成变量覆盖了，include只是完美利用了变量覆盖这个功能。全局搜索也是很容易搜到调用位置的，这里有两处，另外一处是display方法，试了下也是可以完美利用的，它在调用View类下的fetch方法的时候会传入`$renderContent=true`用来区分fetch方法：

![image-20220409223725596](https://user-images.githubusercontent.com/68197734/162579116-d4cdca72-548f-4b2a-b3f7-8d6ac1d0f9b3.png)

![image-20220409224352447](https://user-images.githubusercontent.com/68197734/162579118-4880e626-25f6-40d9-8a49-80a9c011460e.png)

