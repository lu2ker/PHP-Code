# 以下内容 摘自 [挖掘暗藏ThinkPHP中的反序列利用链](https://blog.riskivy.com/%e6%8c%96%e6%8e%98%e6%9a%97%e8%97%8fthinkphp%e4%b8%ad%e7%9a%84%e5%8f%8d%e5%ba%8f%e5%88%97%e5%88%a9%e7%94%a8%e9%93%be/)

## 魔术方法
|方法名|调用条件|
| ---- | ---- |
|__call|调用不可访问或不存在的方法时被调用|
|__callStatic	|调用不可访问或不存在的静态方法时被调用
|__clone	|进行对象clone时被调用，用来调整对象的克隆行为
|__constuct	|构建对象的时被调用；
|__debuginfo	|当调用var_dump()打印对象时被调用（当你不想打印所有属性）适用于PHP5.6版本
|__destruct	|明确销毁对象或脚本结束时被调用；
|__get	|读取不可访问或不存在属性时被调用
|__invoke	|当以函数方式调用对象时被调用
|__isset	|对不可访问或不存在的属性调用isset()或empty()时被调用
|__set	|当给不可访问或不存在属性赋值时被调用
|__set_state	|当调用var_export()导出类时，此静态方法被调用。用__set_state的返回值做为var_export的返回值。
|__sleep	|当使用serialize时被调用，当你不需要保存大对象的所有数据时很有用
|__toString	|当一个类被转换成字符串时被调用
|__unset	|对不可访问或不存在的属性进行unset时被调用
|__wakeup	|当使用unserialize时被调用，可用于做些对象的初始化操作

## 反序列化的常见起点

|方法名|调用条件|
| ---- | ---- |
|__wakeup| 一定会调用
|__destruct |一定会调用
|__toString |当一个对象被反序列化后又被当做字符串使用

## 反序列化常见跳板

|方法名|调用条件|
| ---- | ---- |
|__toString |当一个对象被当做字符串使用
|__get |读取不可访问或不存在属性时被调用
|__set |当给不可访问或不存在属性赋值时被调用
|__isset |对不可访问或不存在的属性调用isset()或empty()时被调用

## 反序列化常见终点

|方法名|调用条件|
| ---- | ---- |
|__call |调用不可访问或不存在的方法时被调用
|call_user_func |一般php代码执行都会选择这里
|call_user_func_array| 一般php代码执行都会选择这里

## Phar反序列化原理以及特征

phar://伪协议会在多个函数中反序列化其metadata部分
受影响的函数包括不限于如下:
```
copy,file_exists,file_get_contents,file_put_contents,file,fileatime,filectime,filegroup,
fileinode,filemtime,fileowner,fileperms,
fopen,is_dir,is_executable,is_file,is_link,is_readable,is_writable,
is_writeable,parse_ini_file,readfile,stat,unlink,exif_thumbnailexif_imagetype,
imageloadfontimagecreatefrom,hash_hmac_filehash_filehash_update_filemd5_filesha1_file,
get_meta_tagsget_headers,getimagesizegetimagesizefromstring,extractTo
```
