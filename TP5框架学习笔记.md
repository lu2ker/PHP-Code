# TP5框架简单理解

（PS：只做粗略、关键知识的记录，TP程序的开始。详情请阅读[官方手册](https://www.kancloud.cn/manual/thinkphp5/118003)）

## 1. 架构总览

TP程序的开始

>   PHP >=5.3.0， PHP7
>
>   `ThinkPHP5.0`应用基于`MVC`（模型-视图-控制器）的方式来组织。
>
>   MVC是一个设计模式，它强制性的使应用程序的输入、处理和输出分开。使用MVC应用程序被分成三个核心部件：模型（M）、视图（V）、控制器（C），它们各自处理自己的任务。
>
>   5.0的URL访问受路由决定，如果关闭路由或者没有匹配路由的情况下，则是基于：
>
>   http://serverName/index.php/模块/控制器/操作/参数/值…

### 1.1 控制器/操作

5.0的控制器类比较灵活，可以无需继承任何基础类库。控制器不应该过多的介入业务逻辑处理

一个典型的`Index`控制器类如下：==控制器就是一个类，类里的不同的方法就是不同的操作==

```php
namespace app\index\controller;
use think\Request;

class Index 
{
    public function index()
    {
        return 'hello,thinkphp!';
    }
    public function test(){
        $name = Request::instance()->param('name');
        echo "My name is " . $name;
    }
}
```

操作方法可以不使用任何参数，如果定义了一个非可选参数，则该参数必须通过用户请求传入，如果是URL请求，则通常是`$_GET`或者`$_POST`方式传入。

-   如上代码第一行是确定了命名空间。什么是命名空间？

>   在PHP中，命名空间用来解决在编写类库或应用程序时创建可重用的代码如类或函数时碰到的两类问题。
>
>   1.   用户编写的代码与PHP内部的类/函数/常量或第三方类/函数/常量之间的名字==冲突==
>   2.   为很长的标识符名称(通常是为了缓解第一类问题而定义的)创建一个别名（或简短）的名称，提高源代码的可读性。

而TP5的思想是

>   `ThinkPHP5`采用命名空间方式定义和自动加载类库文件，有效的解决了多模块和`Composer`类库之间的命名空间冲突问题，并且实现了更加高效的类库自动加载机制。

-   如上代码第二行use就是加载类库，调用TP5写好的Request类，其中实现了一些请求方法。

![image-20220411191418823](TP5框架学习笔记.assets/image-20220411191418823.png)

相对应的就是上面代码11行的`Request::instance()->param('name');`即获取客户端传入的参数name

### 1.2 MVC模式流程

通常一个基于TP开发的CMS，可能有很多个模块，而每个模块都是控制器---模型---视图完整的。控制器根据要求选择调用模型，模型进行业务逻辑处理，交给视图渲染。这个流程。

>   一个模块下面有多个`控制器`负责响应请求，而每个`控制器`其实就是一个独立的`控制器类`。`控制器`主要负责请求的接收，并调用相关的`模型`处理，并最终通过`视图`输出。严格来说，`控制器`不应该过多的介入业务逻辑处理。
>
>   `模型类`通常完成实际的业务逻辑和数据封装，并返回和格式无关的`数据`。
>
>   控制器调用模型类后返回的`数据`通过`视图`组装成不同格式的输出。`视图`根据不同的需求，来决定调用`模板引擎`进行内容解析后输出还是直接输出。

### 1.3 类库自动加载

命名空间的路径与类库文件的目录一致，那么就可以实现`类的自动加载`。

如果想调用不同命名空间下的类库的方法，就要先用`use`给加载上。

### 1.4 URL访问检测

应用初始化完成后，就会进行URL的访问检测，包括`PATH_INFO`检测和URL后缀检测。

5.0的URL访问必须是`PATH_INFO`方式（包括兼容方式）的URL地址，例如：

```
http://serverName/index.php/index/index/hello/val/value
```

所以，如果你的环境只能支持普通方式的URL参数访问，那么必须使用

```
http://serverName/index.php?s=/index/index/hello&val=value
```

但是，这样也是行的：

```
http://serverName/index.php/index/index/hello/?val=value
```

我的环境可以用上面三种方式，都是默认配置。

### 1.5 路由模式

#### 1.5.1 普通模式

'url_route_on'  =>  false,

关闭路由，使用默认的PATH_INFO模式访问URL：如上面 1.4URL访问检测 的第一个
http://serverName/index.php/module/controller/action/param/value/...

#### 1.5.2 混合模式

'url_route_on'  =>  true,

'url_route_must'=>  false,

开启路由，也就是允许了自定义路由，同时对没有定义的路由使用默认的普通模式

#### 1.5.4 强制路由

'url_route_on'  =>  true,

'url_route_must'=>  true,

这种方式下面必须严格给每一个访问地址定义路由规则（包括首页），否则将抛出异常。

首页的路由规则采用`/`定义即可，例如下面把网站首页路由输出`Hello,world!`

```php
Route::get('/',function(){
    return 'Hello,world!';
});
```

### 1.6 路由定义

**ThinkPHP5** 中支持 **5种** 路由地址方式定义：

| 定义方式                  | 定义格式                                                     |
| ------------------------- | :----------------------------------------------------------- |
| 方式1：路由到模块/控制器  | '[模块/控制器/操作]?额外参数1=值1&额外参数2=值2...'          |
| 方式2：路由到重定向地址   | '外部地址'（默认301重定向） 或者 ['外部地址','重定向代码']   |
| 方式3：路由到控制器的方法 | '@[模块/控制器/]操作'                                        |
| 方式4：路由到类的方法     | '\完整的命名空间类::静态方法' 或者 '\完整的命名空间类@动态方法' |
| 方式5：路由到闭包函数     | 闭包函数定义（支持参数传入）                                 |

#### 1.6.1方式 1：路由到模块/控制器

```php
// 路由到默认或者绑定模块
'blog/:id'=>'blog/read', 				# 意思是如果访问blog/5，就会调用blog/read并传入参数id=5,下面同理
// 路由到index模块
'blog/:id'=>'index/blog/read',
----------------------------------------------
namespace app\index\controller;
class Blog {
    public function read($id){
        return 'read:'.$id;
    }
}
```

还支持多级控制器：

```php
//多级控制器方式
'blog/:id'=>'index/group.blog/read'
----------------------------------------------
namespace app\index\controller\group;
class Blog {
    public function read($id){
        return 'read:'.$id;
    }
}
```

#### 1.6.2 方式2：路由到重定向地址

举个例子，如果我们希望`avatar/123`重定向到/member/avatar/id/123_small的话，只能使用：

```php
'avatar/:id'=>'/member/avatar/id/:id_small'

# 区别于 'avatar/:id'=>'avatar/read' ，这个不是采用301重定向的。重定向的外部地址必须以“/”或者http开头的地址。
```

路由地址采用重定向地址的话，如果要引用动态变量，直接使用动态变量即可。

采用重定向到外部地址通常对网站改版后的URL迁移过程非常有用，例如：

```
'blog/:id'=>'http://blog.thinkphp.cn/read/:id'
```

表示当前网站（可能是http://thinkphp.cn ）的 blog/123地址会直接重定向到 http://blog.thinkphp.cn/read/123。

#### 1.6.3 方式3：路由到控制器的方法

```php
'blog/:id'=>'@index/blog/read',
-------------------------------------
会执行 Loader::action('index/blog/read');
相当于直接调用 \app\index\controller\blog类的read方法。
namespace app\index\controller;
class Blog {
    public function read($id){
        return 'read:'.$id;
    }
}
```

#### 1.6.4 方式4：路由到类的方法

```php
'blog/:id'=>'\app\index\service\Blog@read'（动态方法）
或
'blog/:id'=>'\app\index\service\Blog::read',（静态方法）

执行的是 \app\index\service\Blog类的read方法。
```

### 1.7 路由使用

application目录是这个应用的目录，index目录是这个应用的一个模块，其内的controller目录存放的是这个模块的控制器，同理一般modle目录就存放的模型，view目录就放的视图。（这里是单纯的框架所以没有）

![image-20220412183539545](TP5框架学习笔记.assets/image-20220412183539545.png)

![image-20220412184814572](TP5框架学习笔记.assets/image-20220412184814572.png)

```php
#index.php
<?php
namespace app\index\controller;
use think\Request;
class Index
{
    public function index()
    {
        echo "hahaha";
    }
    public function test(){
        $name = Request::instance()->param('name');
        echo "My name is " . $name;
    }
}
```

### 1.8 路由的其他知识

```php
// 使用注解路由
'route_annotation'       => true,

class Index
{
    /**
     * @param  string  $name 数据名称
     * @return mixed
     * @route('hello/:name','get')
     */
	public function hello($name)
    {
    	return 'hello,'.$name;
    }
}
```

其余的直接看[这里](https://www.kancloud.cn/manual/thinkphp5_1/469333)(TP5.1的)就好

### 1.9 其他东西

`tarit`关键字关于这个东西，看PHP手册就行，我的目录下有CHM格式的手册；

`'default_return_type'=>'json'`修改config.php下的这个配置，可以使输出自动进行数据转换处理。`Response`类会统一处理。

