# 001-渗透案例：巧用JS挖掘“宝藏”



&lt;!--more--&gt;
声明：本文纯属虚构，如有雷同纯属巧合。



## 背景

一个寻常的工作日，小X 收到任务对 A 集团某系统进行渗透测试，但是不能用自己电脑，需要远控客户电脑进行测试。（因此，以下内容配图都不是原图，~~尽量还原，~~都是虚构的）

连上客户电脑，简单看了下环境，还行，还是有Burpsuite的，不算天崩开局。BP一开，Link start！果然一个标准登录框：账户、密码、图形验证码。

![image-20250105225247971](resource/20250105-记一次渗透案例.assets/image-20250105225247971.png)

## 1 - 碰壁

常规思路是对系统进行路径扫描，客户电脑上没有工具，就用 Burp 自带的字典扫描一下，没什么发现。

同时对登录框进行常规测试：弱口令、用户名密码枚举、验证码缺陷等，未复习漏洞。



## 2 - JS的救赎

接下去对系统前端界面进行分析，通过抓包分析很神奇的事情，这个网站没有明确的.js和.css结尾的文件，怎么会呢，不科学。

分析数据包发现当访问系统域名时，会自动重定向到登录页面，但是这个过程中有个链接是这样的

```
/include.html?page=/path/login.jsp&amp;type=css&amp;xxx&amp;files=index.css;.../static/xxxx.css;
/include.html?page=/path/login.jsp&amp;type=js&amp;xxx&amp;files=index.js;.../static/xxxx.js;
```

对这个链接进行分析，分析相应功能网页的JS和CSS文件都是通过include.html进行统一加载。

那么这么一个链接，有哪些可能的漏洞呢？

我在这块测试了文件读取、SSRF等，未发现有效漏洞。

通过上述链接，当 `type=js` 时，会加载默认的JS文件，再加上所指定JS文件，对该JS进行分析，终于有所收获了。

![image-20250108001300569](resource/20250105-记一次渗透案例.assets/image-20250108001300569.png)

对JS文件进行扫描，发现了很多接口

![image-20250108003437130](resource/20250105-记一次渗透案例.assets/image-20250108003437130.png)

虽然有些接口访问是404，但还是有很多接口存在，不过下面这些接口访问401，无法利用。

```
/common/linkeditor.jsp?parentPage=pagedesigner&amp;url=
/admin/upload.htm
/admin/deleteTempFile.htm?fileName=
```

但是也成功找到了一些能够访问的接口，如下：

```
/runSafeScript.htm?sqlId=
/runScript.htm
/runStoredProcedure.htm
/runJavaMethod.htm
/runPython.htm
/runHttp.htm
```

对相应接口的JS代码进行分析，成功构造了`runScript` 和 `runHttp` 的请求包，并利用成功。

### SQL执行

```js
function runSQL(e, t) {
  …………
  $.ajax({
      url: getBasePath() &#43; &#34;/runScript.htm&#34;,
      type: &#34;post&#34;,
      async: !1,
      data: { script: encodeBase64Twice(i), dataSource: t || &#34;current&#34; },
      success: function (e) {  a = e; },
    })
	……
}
```

根据 JS 构造请求包：

```http
POST /runScript.htm HTTP/1.1
Host: xxxxxx

script={双重Base64的SQL语句}&amp;dataSource=current
```

使用建议SQL语句`select 1;` ，`select &#39;a&#39;` 探测都成功了，但是 `select database()`，`SELECT banner FROM v$version` 等都没执行成功。

在这块花了很多时间，进行漫长的探测，常见的SQL语句都无法成功执行，由于时间紧迫一度打算放弃了。最后想着再试一次，从头整理探测思路判断数据库，最终确定了系统的数据库类型为 Postgresql，而不是前期判断的 MySQL、Oracle、MSSQL、DMSQL。

![image-20250108012211964](resource/20250105-记一次渗透案例.assets/image-20250108012211964.png)

SQL语句执行成功。



### SSRF

再看另一个接口`runHttp.htm` 的 JS代码

```js
function runHttp(e, t, n, r, i) {
  $.post(
    getBasePath() &#43; &#34;/runHttp.htm&#34;,
    { url: e, method: t, data: n, header: r },
    function (e) {
      i &amp;&amp; i(e);
    }
  );
}
```

根据上述 JS 代码可构造如下HTTP请求

```http
POST /runHttp.htm HTTP/1.1
Host: xxxxxxx

url=http://xxxxx/statics/xxxx.js&amp;method=get&amp;data=&amp;header=
```

这块没什么好多说的，构造请求后runHttp 成功，能够实现SSRF探测，向目标URL进行请求并获取其响应内容。

![image-20250108021630165](resource/20250105-记一次渗透案例.assets/image-20250108021630165.png)

## 3 - 虚晃的 JWT

在JS中除了上述接口之外，还找到了一条 JWT Token泄露。对于JWT Token 的泄露，直接利用肯定是不行，一般都是过期了的。

这里对 JWT 的 secret Key 进行了枚举，还真是有弱 secret key，通过hashcat进行暴破，示例如下：

```shell
$ hashcat -m 16500 test.jwt.hash ./jwt.secrets.list --force --quiet --show
eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJuYmYiOjE2NzU5MTAwNjcsInVzZXJpZCI6ImZhMTIzMDZlLTEwYmMtNGYyNS1hNWU3LWY2NDIyNWJhMzAyNCIsImV4cCI6MTY3NTk1MDA2NywiaWF0IjoxNjc1OTEwMDY3LCJ1c2VybmFtZSI6Inh4eHhkZW1vMSJ9.xpqoFsaqwBcpYuslV5BvjWpWPH31fExNoDcE-gbryKs:123456
```

后续尝试利用该secret key进行 JWT 伪造，但是没有利用成功。说明当前系统的secret key很可能已经变更了。

不过通过这个JWT解码后，获取到了用户名xxxxdemo1

后续可以尝试用户名密码枚举，不过当前环境下没有这个条件。



## 后续

后续客户给我们提供了测试账户，后面的测试就比较正常了。不过不知道是不是因为测试账号权限比较低，反正所提供的账户是没有之前发现的SQL执行和SSRF的接口的。

所以还是要关注 JS文件中的信息，能有效扩展我们的攻击面，也许会有意外之喜。

最后祝大家工作顺利，Have a nice day! 


---

> 作者: Xavier  
> URL: http://localhost:1313/posts/001-case%E5%B7%A7%E7%94%A8js%E6%8C%96%E6%8E%98%E5%AE%9D%E8%97%8F/  

