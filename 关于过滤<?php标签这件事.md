> 涉及到0day，暂不说明细节，只记录思路。

## 0x00 前言：
- 最近在审一套系统的时候，找到个sqli点
- 该系统搭建安装的时候默认mysql账号为root
- 对于into outfile写文件的条件也全部满足，遂想到用SQL注入写文件的方式来GetShell

## 0x01 问题
因为是用sqli写webshell，payload难免会需要绕过一些针对请求参数的过滤，这里就过滤了`<、>、"`等特殊字符，导致没有办法用类似
`select '<?php phpinfo()?>' into outfile '[webpath]'`的方式写入webshell，其中的尖括号会被html实体化编码，但是怎么能轻言放弃呢？

## 0x02 思路一
在该系统的web目录下有很多php空文件，但是留下了`<?php`，比如根目录下的phpinfo.php内容如下：
```php
<?php 
```
这也是我当时立马相到的思路：通过读取phpinfo.php的内容，得到`<?php`，使用select load_file()，这个方法需要如下条件：
- SQL注入点需要查询结果是两列的，因为`xxx union select load_file('phpinfo.php'),'phpinfo();' into outfile '[webpath]'`
- 需要支持联合查询
- 如果删除了这些毫无意义的空文件就立马shell不了了
随之分享给师傅们，集思广益

## 0x03 思路二
hex编码进行绕过
关于SQL注入中使用hex编码来绕过一些过滤的操作应该ctf会比较常见，当时也是没想起来，姿势用时方恨忘啊，遂决定写下此文。
使用方法如下：`select 0x3c3f70687020706870696e666f28293b3f3e into outfile '[webpath]'`，这样就不存在什么限制了。

## 0x04 思路三
师傅发来[先知一篇文章](https://xz.aliyun.com/t/11337)，其中John 师傅在评论区详细阐述了一种也很棒的方法：UTF-7编码绕过。
总结就是一句话：php的多字节编码并启用UTF-7。例如将<?php phpinfo();?>编码为utf-7，+ADw?php phpinfo()+ADs?+AD4-就能绕过<?检测。
可以通过上传

.htaccess
```
php_flag zend.multibyte 1
php_value zend.script_encoding UTF-7
```
或者

.user.ini
```
zend.multibyte=1
zend.script_encoding=utf-7
```
来开启这个配置。另外该师傅人真好，还讲了一个小tips，原文如下：
```
如果遇到图片文件头检测，`.user.ini`可以不加注释直接打上`GIF89a`然后换行，不会报错。`.htaccess`会报500错误，用#注释掉又无法绕过，
不过`0x00`也是`.htaccess`的注释符，所以可以用`0x00`开头的图片文件头，例如ico `00 00 01 00 01 00 20 20`，这样既能绕过文件头检测，也不会报错。
```
师傅好强，学习。
