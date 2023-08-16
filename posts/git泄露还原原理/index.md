# 021-Git泄露还原原理


## git源码泄露

## 自动化

利用脚本：

- [Githack](https://github.com/lijiejie/GitHack) 

遇到了一个git泄露，使用工具进行利用，失败。

手动获取 git 目录下文件成功，于是想手动利用下该漏洞，并了解下漏洞利用原理。

## 目录结构

首先 git 目录文件结构，通常包含以下内容：

```shell
COMMIT_EDITMSG
branches
config							# 项目的 Git 配置文件
description
HEAD								# 包含对当前分支的引用的文件
index
hooks/
info/
logs/
objects/						# 目录，存储 Git 对象
refs/
```

.git 文件夹存储项目的元数据和对象。这里的“对象”包括“blob”、“tree”、“commit”和“tag”。

- `blob` — 用来存储文件数据的一个文件
- `tree` — 引用了一堆其他tree 或 blob
- `commit` — 一个单独的tree，标记为项目在某个时间点的样子。
- `tag` — 以某种方式将特定提交进行O标记。

以下是 .git 目录中的一些标准文件和文件夹，它们对于重建项目源代码非常重要。

### /objects目录

/objects 目录用于存储 Git 对象。

该目录包含其他文件夹，每个文件夹都有两个角色名称。这些子目录以存储在其中的 git 对象的 SHA1 哈希值的前两个字符命名。

在这些子目录中，有一些以存储在其中的 git 对象的 SHA1 哈希值命名的文件。

例如，下面的命令将返回文件夹列表：

```shell
> ls .git/objects
00 0a 14 5a 64 6e 82 8c 96 a0 aa b4 be c8 d2 dc e6 f0 fa info pack
```

此命令将显示存储在该特定文件夹中的 git 对象：

```shell
> ls .git/objects/0a
082f2656a655c8b0a87956c7bcdc93dfda23f8 4a1ee2f3a3d406411a72e1bea63507560092bd 66452433322af3d319a377415a890c70bbd263 8c20ea4482c6d2b0c9cdaf73d4b05c2c8c44e9 ee44c60c73c5a622bb1733338d3fa964b333f0
0ec99d617a7b78c5466daa1e6317cbd8ee07cc 52113e4f248648117bc4511da04dd4634e6753 72e6850ef963c6aeee4121d38cf9de773865d8 
```

Git 对象根据其 SHA1 哈希值的前两个字符存储在 /objects 中。

例如，哈希值为`0a082f2656a655c8b0a87956c7bcdc93dfda23f8` 的 Git 对象将以文件名 `082f2656a655c8b0a87956c7bcdc93dfda23f8` 存储在目录 `.git/objects/0a` 中。

Git 在 .git/objects 中存储不同类型的对象。这里存储的对象可以是commit,、tree、blob 和带注释的tag。您可以使用以下命令确定对象的类型：

```shell
> git cat-file -t <OBJECT-HASH>
```

这个hash需要写完整的哈希而不是单纯的文件名

```shell
┌──(xavier💀Desktop-AQ550M)-[~/…/gitlab/git/objects/0a]
└─$ ls
66007207cbbbf3fc8e506ca4bc9e7317770060	a78bbdd307fd4839b26d61166b5c0532064bec

┌──(xavier💀Desktop-AQ550M)-[~/…/gitlab/git/objects/0a]
└─$ git cat-file -t 66007207cbbbf3fc8e506ca4bc9e7317770060
fatal: Not a valid object name 66007207cbbbf3fc8e506ca4bc9e7317770060

┌──(xavier💀Desktop-AQ550M)-[~/…/gitlab/git/objects/0a]
└─$ git cat-file -t 0a66007207cbbbf3fc8e506ca4bc9e7317770060
blob
```

提交对象存储有关提交的目录树对象哈希、父提交、作者、提交者、日期和提交消息的信息。

Tree对象包含提交的目录列表。 Blob 对象包含已提交的文件的副本（请阅读：实际源代码！）。而tag对象包含有关标记对象及其关联标记名称的信息。

您可以使用以下命令显示与 Git 对象关联的文件：

```
> git cat-file -p <OBJECT-HASH>
```

```shell
┌──(xavier💀Desktop-AQ550M)-[~/…/git/gitlab/git/objects]
└─$ git cat-file -t 06a67bade0fd6621314daadc38300c908e989dd0
blob

┌──(xavier💀Desktop-AQ550M)-[~/…/git/gitlab/git/objects]
└─$ git cat-file -p 06a67bade0fd6621314daadc38300c908e989dd0
# By Xavier
import sys
import requests
import urllib3
import time

urllib3.disable_warnings()
...
```

### /config文件

/config 文件是项目的 Git 配置文件。如果该文件可访问，也许能够下载 .git 目录的全部内容。

```
┌──(xavier💀Desktop-AQ550M)-[~/…/0727/git/gitlab/git]
└─$ cat config 
[core]
	repositoryformatversion = 0
	filemode = false
	bare = false
	logallrefupdates = true
	symlinks = false
	ignorecase = true
[remote "origin"]
	url = git@github.com:Username/Projectname.git
	fetch = +refs/heads/*:refs/remotes/origin/*
[branch "master"]
	remote = origin
	merge = refs/heads/master
[pull]
	rebase = false

```



### /head文件

/head文件是一个包含对当前分支的引用的文件

```shell
┌──(xavier💀Desktop-AQ550M)-[~/…/0727/git/gitlab/git]
└─$ cat HEAD 
ref: refs/heads/master
```



## .git泄露

尝试探测.git文件，可能有以下几种情况：

- 响应404，意味着应用程序的 .git 目录不向公众开放，无法通过这种方式泄露信息。
- 响应403，表示 .git 目录在服务器上可用，但您将无法直接访问该文件夹的根目录，因此无法列出该目录中包含的所有文件。
- 响应200，如果没有收到错误并且服务器响应 .git 目录的文档树，您可以直接浏览该文件夹的内容并检索其中包含的任何信息。

如果无法访问 /.git 文件夹的目录列表，即未开启目录遍历功能，则必须下载所需的每个文件，而不是从目录根目录递归下载。

但是，当目标文件具有复杂路径（例如“.git/objects/0a/72e6850ef963c6aeee4121d38cf9de773865d8”）时，如何找出服务器上的哪些文件可用？

这就需要从已知存在的文件路径开始，例如“.git/HEAD”！阅读此文件将为您提供当前分支的引用（例如，.git/refs/heads/master），您可以使用它来查找系统上的更多文件。

```shell
> cat .git/HEAD
ref: refs/heads/master
> cat .git/refs/heads/master
0a66452433322af3d319a377415a890c70bbd263
> git cat-file -t 0a66452433322af3d319a377415a890c70bbd263
commit
> git cat-file -p 0a66452433322af3d319a377415a890c70bbd263
tree 0a72e6850ef963c6aeee4121d38cf9de773865d8
```

从Web上就是：

```shell
> curl http://xxx.com/.git/HEAD
> curl http://xxx.com/.git/refs/heads/master
> wget http://xxx.com/.git/objects/0a/66452433322af3d319a377415a890c70bbd263
> git cat-file -t 0a66452433322af3d319a377415a890c70bbd263
commit
> git cat-file -p 0a66452433322af3d319a377415a890c70bbd263
tree 0a72e6850ef963c6aeee4121d38cf9de773865d8
```

.git/refs/heads/master 文件将指向您存储提交目录树的相应对象哈希。可以看到该对象是一个提交，并且与树对象 0a72e6850ef963c6aeee4121d38cf9de773865d8 关联。

继续检查存储在 0a72e6850ef963c6aeee4121d38cf9de773865d8 的树对象时：

```shell
> git cat-file -p 0a72e6850ef963c6aeee4121d38cf9de773865d8
100644 blob 6ad5fb6b9a351a77c396b5f1163cc3b0abcde895 .gitignore
040000 blob 4b66088945aab8b967da07ddd8d3cf8c47a3f53c source.py
040000 blob 9a3227dca45b3977423bb1296bbc312316c2aa0d README
040000 tree 3b1127d12ee43977423bb1296b8900a316c2ee32 resources
```

您会发现一些源代码文件和其他对象树来探索。

在远程服务器上，您发现不同文件的请求看起来更像是这样：

```shell
https://project.com/.git/HEAD 					# 判断 HEAD
https://project.com/.git/refs/heads/master 		# 找到 HEAD 中存储的对象
https://project.com/.git/objects/0a/72e6850ef963c6aeee4121d38cf9de773865d8  # 访问与commit相关联的tree
https://project.com/.git/objects/9a/3227dca45b3977423bb1296bbc312316c2aa0d # 下载 README 文件中存储的源代码
```

在这样的远程服务器上，您需要在读取下载的目标文件之前对其进行解压缩。这可以使用 Ruby 来完成：

```
ruby -rzlib -e 'print Zlib::Inflate.new.inflate(STDIN.read)' < OBJECT_FILE
```

## 敏感信息

之后就是从git泄露中去寻找有用的敏感信息，包括硬编码凭证、加密密钥和开发人员评论可快速了解系统。

## 参考文件

- [Hacking Git Directories](https://medium.com/swlh/hacking-git-directories-e0e60fa79a36)
- [Exposed .git Directory Exploitation](https://infosecwriteups.com/exposed-git-directory-exploitation-3e30481e8d75)



---

> 作者: Xavier  
> URL: https://www.bthoughts.top/posts/git%E6%B3%84%E9%9C%B2%E8%BF%98%E5%8E%9F%E5%8E%9F%E7%90%86/  

