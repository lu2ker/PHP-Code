## 3. 数据库

相信跟完那几个SQLi漏洞的代码分析，就已经对TP种数据库操作有些了解了，故略。

## 4. 模板

### 4.1 变量输出

```php
// index.php 控制器
use think\Controller;
use think\View;
class Index extends Controller
{
    public function index()
    {
        $view = new View();
        $view->name = 'thinkphp';
        return $view->fetch();
    }
}

# index.html
<html>
    Hello,{$name}！
</html>
```

可以大致理解为：访问index方法，fetch渲染时会把name变量给传到index.html，而模板引擎解析的`{$name}`实际上就是`<?php echo($name);?>`

注意模板标签的`{`和`$`之间不能有任何的空格，否则标签无效。

输出数组变量的value用`{$array.key}`

输出对象属性value用`{$obj:property}`或者`${obj->property}`

输出其他变量（比如系统变量）以$Think开头，比如`{$Think.cookie.name}`

### 4.2 使用函数、默认值

```php
{$name|md5|strtoupper|substr=0,3}
# 解释：多个函数之间用“|”分割

编译后即为：
<?php echo (substr(strtoupper(md5($name)),0,3)); ?>
    
# 还可以简单的写法
{:substr(strtoupper(md5($name)),0,3)}

# 加个默认值
{$user.nickname|default="这家伙很懒，什么也没留下"}
```

### 4.3 TP内置标签

内置标签还是全一点把，遇到忘了的直接去[开发手册](https://www.kancloud.cn/manual/thinkphp5/125016)查



==PS：这儿有一个[简单的MVC代码案例](https://www.php.cn/php-weizijiaocheng-429302.html)==



## 5. 日志和错误

## 6. 杂项

### 6.1 缓存

![image-20220413195111418](https://user-images.githubusercontent.com/68197734/163178878-0d9f4e13-323d-4eeb-ab3c-01a2c2fac84b.png)

支持的缓存类型包括file、memcache、wincache、sqlite、redis和xcache。

驱动方式就是指什么形式存 缓存的数据

如果定义了多个缓存驱动：

```php
// 切换到file操作
Cache::store('file')->set('name','value');
Cache::get('name');
// 切换到redis操作
Cache::store('redis')->set('name','value');
Cache::get('name');
```

在TP类库里面thinkphp\library\think\db\Query.php这个数据库查询类调用Cache比较多。

Session 、Cookie 和 缓存差不多。

### 6.2 上传规则

默认情况下，会在上传目录下面生成以当前日期为子目录，以微秒时间的`md5`编码为文件名的文件。

```
/upload/20160510/42a79759f284b767dfcb2a0197904287.jpg
```

而且一般都在public/upload/目录下。

还有可能以32位哈希的前两位当子目录。

上传时可以用$file->rule()自定义文件名生成的规则，如`$file->rule('md5')->move('/home/www/upload/');`

File类的成员函数move()执行成功后文件成功上传，返回一个File类的对象，失败返回false。可以看看File类都有哪些成员，毕竟是文件相关的类，很有用。

`File`类继承了PHP的`SplFileObject`类，因此可以调用`SplFileObject`类所有的属性和方法。[SplFileObject](https://www.php.net/manual/en/class.splfileobject.php)手册解释
