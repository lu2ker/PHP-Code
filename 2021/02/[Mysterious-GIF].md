# [Mysterious-GIF]

Time: 2021-2-28

Author: badbird

---

[题目](https://adworld.xctf.org.cn/task/answer?type=misc&number=1&grade=1&id=5557&page=6)

[wp](https://github.com/ctfs/write-ups-2017/tree/master/breakin-ctf-2017/misc/Mysterious-GIF)

答题环境：kali

题目资源只有一个GIF

上次学习vulnhub的靶场get到了**strings**命令的强大之处，所以决定先尝试用strings命令试一下看看能不能得到有用的信息

![image-20210228151144341](C:\Users\King\AppData\Roaming\Typora\typora-user-images\image-20210228151144341.png)

在元数据区发现了异常信息

![image-20210228151354575](C:\Users\King\AppData\Roaming\Typora\typora-user-images\image-20210228151354575.png)

很明显第三行是十六进制数据，转为字符串看下它是什么

![image-20210228151947517](C:\Users\King\AppData\Roaming\Typora\typora-user-images\image-20210228151947517.png)

目前看不出来是什么。

再回到文件分析的起点，使用**binwalk**命令分离提取这个GIF文件

![image-20210228152257364](C:\Users\King\AppData\Roaming\Typora\typora-user-images\image-20210228152257364.png)

出来个zip文件，继续binwalk

![image-20210228152442895](C:\Users\King\AppData\Roaming\Typora\typora-user-images\image-20210228152442895.png)

得到了256个256字节的.enc文件，那么接下来的任务就是解密了

回想起刚才在元数据区发现的诡异字符传，那个应该得是密钥了，但是看长度又不像。猜测strings命令应该只能获取到GIF的一帧数据，这肯定是密钥的一部分。肯定还有其余部分。

杀牛要用牛刀。这时候又用到一个很棒的命令**identify** 这个命令是用来获取一个或多个图像文件的格式和特性。

它有一个跟strings功能差不多的参数[**-format**](https://www.imagemagick.org/script/escape.php) 用来指定显示的信息

```
identify -format "%c" Question.gif > hex.txt
```

![image-20210228153437354](C:\Users\King\AppData\Roaming\Typora\typora-user-images\image-20210228153437354.png)

这次得到的16进制数据就比用strings命令得到的多多了还没有换行，很方便了。转换得到

![image-20210228153707515](C:\Users\King\AppData\Roaming\Typora\typora-user-images\image-20210228153707515.png)

很明显这就是整个RSA加密的密钥了。接下来就是用**openssl**进行解密。

此时需要注意，得到的整个密钥字符串是没有包含

所以会报错没有开始行

![image-20210228154344590](C:\Users\King\AppData\Roaming\Typora\typora-user-images\image-20210228154344590.png)

可以直接随便生成一个RSA的密钥看看应该有什么起始行

```shell
openssl genrsa -out rsa.key 2048
```

cat rsa.key

![image-20210228154706214](C:\Users\King\AppData\Roaming\Typora\typora-user-images\image-20210228154706214.png)

可以看到begin和end标志着开头和结尾

```
-----BEGIN RSA PRIVATE KEY-----
-----END RSA PRIVATE KEY-----
```

给我们的密钥文件加上这个头和尾即可

再次进行解密

```
openssl rsautl -decrypt -inkey key -in partaa.enc -out aa1
```

![image-20210228155130447](C:\Users\King\AppData\Roaming\Typora\typora-user-images\image-20210228155130447.png)

可以看到解密出来的文件是JPEG图片，且不完整（毕竟有256个.enc肯定是需要联合起来的）

但是256个文件命令修改执行256次实属愚蠢，然后从现在开始我开始了自己的第一次shell编程

```shell
#!/bin/bash
files=$(ls)
a="aa"
n=1
for f in $files; do
        openssl rsautl -decrypt -inkey key -in $f -out $a$n;
        ((n++));
done
```

```
chmod +x thisfile.sh
```

很low但是很实用...将其放到256个.enc文件的同级目录下执行，就可以得到一堆解密后的JPEG了,会报几个错误但是不影响。

提前把_temp.zip.extracted中的其他zip文件都删掉（都是一样的256个.enc）

```
rm -rf *.zip
```

最后使用cat拼接这些个文件最后会得到一张显示有flag的图片

```
cat aa* > flag
```

![image-20210228160343544](C:\Users\King\AppData\Roaming\Typora\typora-user-images\image-20210228160343544.png)

结束。