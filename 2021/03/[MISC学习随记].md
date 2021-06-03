# [MISC学习随记]

#### base编码

- base64：

  - 6bit一组，3个字节需要4个可打印字符（相当于码元⑧）“ = ”代表补足的字节数，**最多两个**

    ![image-20210303123341746](C:\Users\King\AppData\Roaming\Typora\typora-user-images\image-20210303123341746.png)

  - **解码时补位的0不参与运算，可在此处隐藏信息**

- base32：

  - 5bit一组，5个字节需要8个可打印字符，“ = ”最多6个

    ![image-20210303124221060](C:\Users\King\AppData\Roaming\Typora\typora-user-images\image-20210303124221060.png)

#### 常用命令

- binwalk  多用于多个文件粘合在一起
  - -e 参数提取文件
  - 结合 dd 命令进行手动切割
- foremost 分离文件，与binwalk -e 的区别待考究
- strings 打印可打印的字符
- identify 获取一个或多个图像文件的格式和特性
  - -format 用来显示指定的信息
    - %c 注释元数据属性，这样就直接显示异常数据了
    - %s scene number (from input unless re-assigned)

#### 图片分析

- 元数据：又称中介数据、中继数据，为描述数据的数据（Data about data），主要是描述数据属性（property）的信息，用来支持如指示存储位置、历史数据、资源查找、文件记录等功能。

- PNG：文件头+3个以上的数据块

  - 文件头：89 50 4E 47 0D 0A 1A 0A

  - 数据块Chunk 每个Chunk都有统一的数据结构：

    | 名称                         | 字节数 | 说明                                                         |
    | ---------------------------- | ------ | ------------------------------------------------------------ |
    | Length 长度                  | 4bytes | 指定数据块中数据域的长度，其长度不超过（231－1）字节         |
    | Chunk Type code 数据库类型码 | 4bytes | 大小写字母混合4位 如IHDR、gAMA                               |
    | Chunk Data   数据            | 可变   | 前8字节：4字节宽度4字节高度，以像素为单位                    |
    | CRC                          | 4bytes | 校验码，是对 Chunk Type Code 域和 Chunk Data 域中的数据进行计算得到的。 |

  例：IHDR

  ```
  00000000  80 59 4e 47 0d 0a 1a 0a  00 00 00 0d 49 48 44 52  |.YNG........IHDR|
  00000010  00 00 00 00 00 00 02 f8  08 06 00 00 00 93 2f 8a  |............../.|
  00000020  6b 00 00 00 04 67 41 4d  41 00 00 9c 40 20 0d e4  |k....gAMA...@ ..|
  00000030  cb 00 00 00 20 63 48 52  4d 00 00 87 0f 00 00 8c  |.... cHRM.......|
  00000040  0f 00 00 fd 52 00 00 81  40 00 00 7d 79 00 00 e9  |....R...@..}y...|
  ...
  ```

  - 00 00 00 0d 代表数据块的数据长度为13

  - 49 48 44 52 即IHDR 是数据块的名称

  - 再往后数13个字节即为Chunk Data，这里很明显宽度被更改了，但是**我们不能任意修改文件宽度，如果不是真实值会报错**。后五个字节依次为：Bit depth、ColorType、Compression method、Filter method、Interlace method

  - 93 2f 8a 6b 即为CRC

    **可以通过CRC的值爆破得到宽度。**