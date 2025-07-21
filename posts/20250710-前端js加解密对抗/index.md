# 前端JS加解密对抗




本文主要介绍常见的应对前端JS加解密的方法

## 零、简述场景

常见需要前端JS调试完成加解密对抗的情况有：

1. 参数值加密：如登录过程中对用户名、密码进行了加密；
2. 键值加密：对参数的键值都进行了加密；
3. 数据包全加密：对数据包请求体进行了全加密；
4. 数据包签名：对数据包内容进行了某种签名算法以防篡改；
5. 组合利用

**常见加解密方法**有：

1. 对称加密：AES、SM4(国密)、 DES、3DES、SM1、SM7、RC5、RC6、RC4、ZUC、SSF446等

   特点：加解密使用同一个密钥，可逆

2. 非对称加密：RSA, SM2. SM9, Rabin, DH, DSA, ECC

   特点：密钥是成对的，分为公钥和私钥；

3. 散列算法：MD5、 SM3、 MAC, HMAC, SHA-1, SHA-2 (SHA-224. SHA-256, SHA-512/224. SHA-512/256) 等

   特点：无密钥，不可逆，固定长度

**问题解决思路**：

1. 分析加密算法，确认加密方式，还原加解密流程

2. 分析密钥来源：

   若为客户端生成密钥：密钥固定（硬编码密钥）/密钥随机（分析密钥生成算法）

   若为服务器返回密钥：则重新建立连接并进行抓包，获取密钥，再加解密

3. 若无法还原加解密流程：通常是存在魔改算法或混淆场景，通过定位明文将其转发，篡改明文后再转发回去走加密流程

## 一、手动加解密

常见的渗透测试场景包括web/小程序/公众号/app，在不同场景下的分析方法不同。

本文主要以Web端为例，其他场景不过多讨论。

### web端

定位加密算法：

**1、全局搜索关键字**

- 参数名、“参数名=、参数名=、参数名：

- URI、API接口、加密算法关键词

```
encrypt, decrypt, JSON.stringify, JSON.parse, secret，secretkey，publickey, privatekey，padding，
key, aes, sm4, rsa, sm2，des, iv, pkcs
```

搜索的关键词越短，出来的误报也就越多，所以要选好关键词

**2、断点调试**

找到一些关键词或关键动作打上断点，比如疑似加解密函数部分，鼠标点击特定按钮事件，网络请求等。

- 调用堆栈：可在浏览器点击对应请求后，右侧面板查看请求调用堆栈，然后打断点跟踪定位
- XHR断点：在source资源面板右下角，可以添加URL关键字，在发起网络请求时触发
- 事件监听断点：可以在发生相关事件时触发断点，如登陆时触发click点击事件
- DOM断点：可以在子元素改变时、属性改变时和元素被移除时触发断点，如滑块验证码

**3、hook**

在找不到关键字时（如混淆），可以通过hook某些通用方法寻找突破口，如 JSON.stringify、JSON.parse

还可以hook，如 cookie、header、URL、eval、Function、绕debugger等



### APP端

定位加密算法：

**1、静态分析**

需要先对APP进行逆向，通过脱壳反编译得到源码；

然后也是jadx全局搜索关键字，类似Web思路，搜索如sign，加引号的“sign”等。

**2、hook**

也可以通过frida来hook一些java层的常见类/方法，如

- HashMap的put方法

```java
var hashMap = Java.use(&#34;java.util.HashMap&#34;);
hashMap.put.implementation = function(a, b){
	if(a.equals(&#34;username&#34;)){
		showStacks();
		console.log(&#34;hashMap.put:&#34;, a, b);
	}
	return this.put(a, b);
}
```

- Log日志

```java
var log = Java.use(&#34;android.util.Log&#34;);
log.w.overload(&#39;java.lang.String&#39;, &#39;java.lang.String&#39;).implementation = function (tag, message) {
	showStacks();
	console.log(&#34;log.w: &#34;, tag, message);
	return this.w(tag, message);
}
```

java层常见方法hook，如

- JSONObject的Jput方法

```java
var jSONObject = Java.use(&#34;org.json.JSONObject&#34;);
jSONObject.put.overload(&#39;java.lang.String&#39;, &#39;java.lang.Object&#39;).implementation = function(a, b) {
  showStacks();
	//var result = Java.cast(a, Java.use(&#34;java.util.ArrayList&#34;));
  console.log(&#34;jSONObject.put: &#34;, a, b);
  return this.put(a, b);
}
```

- JSONObject的JgetString方法

```java
var jSONObject = Java.use(&#34;org.json.JSONObject&#34;);
jSONObject.getString.implementation = function (a) { 
  //showStacks;
  //var result = Java.cast(a, Java.use(*java.util.ArrayList&#34;));
  console.log(&#34;jSONObject.getString:&#34;, a);
  var result = this.getString(a);
  console.log(&#34;jSONObject.getString result: &#34; , result);
  return result;
}
```

java层常见方法hook：

其他，如 ArrayList的add、addAll、set方法等、TextUtils的isEmpty方法、Collections的sort 方法、Toast的show方法、Base64、String的getBytes、 isEmpty方法、StringBuilder, StringBuffer、常见加密库相关的hook（自吐算法）等

hook代码生成方法：

可以通过objection生成指定类的hook代码 

android hooking generate class com.XXXXXX 

jadx也可以生成frida代码



**3、动态调试**：

log插桩、JEB. IDA, unidbg

**4、H5调试**

安卓手机开启adb调试，电脑edge浏览器访问edge://inspect，手机访问h5页面，浏览器点击inspect打开 devtools即可调试（chrome浏览器访问chrome://inspect）

mac safari 调试 iphone 下 h5/webview

windows 调试 iphone

- https://github.com/google/ios-webkit-debug-proxy 

edge://inspect

### 小程序

开启小程序调试：

- https://github.com/JaveleyQAQ/WeChatOpenDevTools-Python

### 公众号

微信H5调试：手机开启adb调试，微信访问  `http://debugxweb.qq.com/?inspector=true` 

电脑访问：edge://inspect

其他H5调试方法：

```sh
npm install -g spy-debugger
```

- https://github.com/wuchangming/spy-debugger



## 二、控制台

## 三、Python服务中转

## 四、JSRPC

## 五、CDP



---

> 作者: Xavier  
> URL: http://localhost:1313/posts/20250710-%E5%89%8D%E7%AB%AFjs%E5%8A%A0%E8%A7%A3%E5%AF%86%E5%AF%B9%E6%8A%97/  

