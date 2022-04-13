## 2.请求相关

### 2.1 如何获取请求参数的？

首先要use一下TP写好的Request类，然后调用的话可以用很多种写法，下面是一种

```php
<?php
namespace app\index\controller;
use think\Request;

class Index 		// class Index extends Controller 解释如下：
//如果你继承了系统的控制器基类think\Controller的话，系统已经自动完成了请求对象的构造方法注入了，你可以直接使用$this->request属性调用当前的请求对象。
{
    /**
     * @var \think\Request Request实例
     */
    protected $request;
    /**
     * 构造方法
     * @param Request $request Request对象
     * @access public
     */
    public function __construct(Request $request)
    {
		$this->request = $request;
    }
    public function index()
    {
		return $this->request->param('name');
    }    
}
```

还可以这些样子：

```php
Request::instance=>param();//获取所有参数[ 结果类型数组],不分请求类型;
Request::instance=>param('name');//获取单个参数[即:直接填写变量名即可];
Request::instance=>get();//获取?后面的参数;
Request::instance=>route();//获取路由里面的参数; 
Request::instance=>post();//获取post请求参数
eg:
public function hello()
{
   $res=Request::instance()->param();
   var_dump($res);
}
//这是依赖注入方式，无论是否继承系统的控制器基类，都可以使用操作方法注入。
public function hello(Request $request)
{
   $res=$request->param();
   var_dump($res);
}
//也可以使用助手函数
$request = request();
```

### 2.2 检测变量是否设置

可以使用`has`方法来检测一个变量参数是否设置，如下：

```php
Request::instance()->has('id','get');
Request::instance()->has('name','post');
```

或者使用助手函数（助手函数很棒很简单）

```php
input('?get.id');
input('?post.name');
```

### 2.3 请求方法

可以通过`Request`对象完成全局输入变量的检测、获取和安全过滤，支持包括`$_GET`、`$_POST`、`$_REQUEST`、`$_SERVER`、`$_SESSION`、`$_COOKIE`、`$_ENV`等系统变量，以及文件上传信息。

| 方法    | 描述                           |
| :------ | :----------------------------- |
| param   | 获取当前请求的变量             |
| get     | 获取 $_GET 变量                |
| post    | 获取 $_POST 变量               |
| put     | 获取 PUT 变量                  |
| delete  | 获取 DELETE 变量               |
| session | 获取 $_SESSION 变量            |
| cookie  | 获取 $_COOKIE 变量             |
| request | 获取 $_REQUEST 变量            |
| server  | 获取 $_SERVER 变量             |
| env     | 获取 $_ENV 变量                |
| route   | 获取 路由（包括PATHINFO） 变量 |
| file    | 获取 $_FILES 变量              |

也可以在获取变量的时候添加过滤方法，例如：

```php
Request::instance()->get('name','','htmlspecialchars'); // 获取get变量 并用htmlspecialchars函数过滤
Request::instance()->param('username','','strip_tags'); // 获取param变量 并用strip_tags函数过滤
Request::instance()->post('name','','org\Filter::safeHtml'); // 获取post变量 并用org\Filter类的safeHtml方法过滤
//可以支持传入多个过滤规则，例如：
Request::instance()->param('username','','strip_tags,strtolower');// 获取param变量 并依次调用strip_tags、strtolower函数过滤
```

### 2.4 【▲】请求方法伪装

支持请求类型伪装，可以在`POST`表单里面提交`_method`变量，传入需要伪装的请求类型，例如：

```
<form method="post" action="">
    <input type="text" name="name" value="Hello">
    <input type="hidden" name="_method" value="PUT" >
    <input type="submit" value="提交">
</form>
```

提交后的请求类型会被系统识别为`PUT`请求。你可以设置为任何合法的请求类型，包括`GET`、`POST`、`PUT`和`DELETE`等。如果你需要改变伪装请求的变量名，可以修改应用配置文件：

```
// 表单请求类型伪装变量
'var_method'             => '_m',
```

### 2.5 伪静态配置效果

```
'url_html_suffix' => 'shtml'

http://serverName/Home/Blog/read/id/1
等价于
http://serverName/Home/Blog/read/id/1.shtml
如果'url_html_suffix' => ''
则
任何后缀都能正常访问

// 关闭伪静态后缀访问
'url_html_suffix' => false,
http://serverName/index/blog/read/id/3.html
id参数的值会被解析为3.html
```

### 2.6 参数绑定

就是给控制器下的方法（函数），给上参数，这样在url访问的时候就直接加上这个参数再给个值就能自动获取，不需要写Request。

![image-20220413134848111](https://user-images.githubusercontent.com/68197734/163171967-05eb5722-78d6-4ecc-94f4-82fca2381b3c.png)

### 2.7 请求缓存

例子：如果设置了这样一条路由，其中指定了cache字段。

```php
Route::get('bind/:id','Index/bind',['cache'=>3600]);
```

那么只有第一次访问时会走正常的C-M-V流程，也就是会真正去调用控制器下的操作方法。

之后再访问同样路由的话，检测到同样的路由就不会再去调用控制器下的操作方法了，而是直接从缓存中获取响应。很给力。
