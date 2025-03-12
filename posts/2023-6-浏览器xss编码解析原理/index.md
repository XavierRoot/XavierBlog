# 浏览器XSS编码解析原理


## 一、小测验

本文介绍浏览器加载过程以及常用的XSS编码解析原理。首发于T00ls论坛，在博客中再发一遍留档。

首先做个小测试，下面的XSS payload能否触发，题目选自[大佬博客](https://www.attacker-domain.com/2013/04/deep-dive-into-browser-parsing-and-xss.html)。

### 问题

基础题：

```html
1.URL编码&#34;javascript:alert(1)&#34;
&lt;a href=&#34;%6a%61%76%61%73%63%72%69%70%74:%61%6c%65%72%74%28%31%29&#34;&gt;&lt;/a&gt;

2.实体编码&#34;javascript&#34;，URL编码&#34;alert(2)&#34;
&lt;a href=&#34;&amp;#x6a;&amp;#x61;&amp;#x76;&amp;#x61;&amp;#x73;&amp;#x63;&amp;#x72;&amp;#x69;&amp;#x70;&amp;#x74;:%61   %6c%65%72%74%28%32%29&#34;&gt;

3.URL编码&#34;:&#34;
&lt;a href=&#34;javascript%3aalert(3)&#34;&gt;&lt;/a&gt;

4.实体编码&lt; &gt;
&lt;div&gt;&amp;#60;img src=x onerror=alert(4)&amp;#62;&lt;/div&gt;

5.实体编码&lt; &gt;
&lt;textarea&gt;&amp;#60;script&amp;#62;alert(5)&amp;#60;/script&amp;#62;&lt;/textarea&gt;

6. &lt;textarea&gt;&lt;script&gt;alert(6)&lt;/script&gt;&lt;/textarea&gt;
```

高级题：

```html
7.实体编码&#39;
&lt;button onclick=&#34;confirm(&#39;7&amp;#39;);&#34;&gt;Button&lt;/button&gt;

8.Unicode编码&#39;
&lt;button onclick=&#34;confirm(&#39;8\u0027);&#34;&gt;Button&lt;/button&gt;

9.实体编码alert(9);
&lt;script&gt;&amp;#97;&amp;#108;&amp;#101;&amp;#114;&amp;#116&amp;#40;&amp;#57;&amp;#41;&amp;#59&lt;/script&gt;

10.Unicode编码alert
&lt;script&gt;\u0061\u006c\u0065\u0072\u0074(10);&lt;/script&gt;

11.unicode编码alert(11)
&lt;script&gt;\u0061\u006c\u0065\u0072\u0074\u0028\u0031\u0031\u0029&lt;/script&gt;

12.Unicode编码alert和12
&lt;script&gt;\u0061\u006c\u0065\u0072\u0074(\u0031\u0032)&lt;/script&gt;

13.Unicode编码&#39;
&lt;script&gt;alert(&#39;13\u0027)&lt;/script&gt;

14.Unicode编码换行符
&lt;script&gt;alert(&#39;14\u000a&#39;)&lt;/script&gt;
```

附加题：

```html
&lt;a
	  href=&#34;&amp;#x6a;&amp;#x61;&amp;#x76;&amp;#x61;&amp;#x73;&amp;#x63;&amp;#x72;&amp;#x69;&amp;#x70;&amp;#x74;&amp;#x3a;&amp;#x25;&amp;#x35;&amp;#x63;&amp;#x25;&amp;#x37;&amp;#x35;&amp;#x25;&amp;#x33;&amp;#x30;&amp;#x25;&amp;#x33;&amp;#x30;&amp;#x25;&amp;#x33;&amp;#x36;&amp;#x25;&amp;#x33;&amp;#x31;&amp;#x25;&amp;#x35;&amp;#x63;&amp;#x25;&amp;#x37;&amp;#x35;&amp;#x25;&amp;#x33;&amp;#x30;&amp;#x25;&amp;#x33;&amp;#x30;&amp;#x25;&amp;#x33;&amp;#x36;&amp;#x25;&amp;#x36;&amp;#x33;&amp;#x25;&amp;#x35;&amp;#x63;&amp;#x25;&amp;#x37;&amp;#x35;&amp;#x25;&amp;#x33;&amp;#x30;&amp;#x25;&amp;#x33;&amp;#x30;&amp;#x25;&amp;#x33;&amp;#x36;&amp;#x25;&amp;#x33;&amp;#x35;&amp;#x25;&amp;#x35;&amp;#x63;&amp;#x25;&amp;#x37;&amp;#x35;&amp;#x25;&amp;#x33;&amp;#x30;&amp;#x25;&amp;#x33;&amp;#x30;&amp;#x25;&amp;#x33;&amp;#x37;&amp;#x25;&amp;#x33;&amp;#x32;&amp;#x25;&amp;#x35;&amp;#x63;&amp;#x25;&amp;#x37;&amp;#x35;&amp;#x25;&amp;#x33;&amp;#x30;&amp;#x25;&amp;#x33;&amp;#x30;&amp;#x25;&amp;#x33;&amp;#x37;&amp;#x25;&amp;#x33;&amp;#x34;&amp;#x28;&amp;#x31;&amp;#x35;&amp;#x29;&#34;&gt;&lt;/a&gt;
```



### 答案

&gt; The answers to these questions are [here](http://test.attacker-domain.com/browserparsing/answers.txt) and the testing page is [here](http://test.attacker-domain.com/browserparsing/tests.html). 

具体内容可以去大佬博客看，我这就写个结果，Y表示触发，N表示不触发

1N，2Y，3N，4N，5N，6N

7Y，8N，9N，10Y，11N，12N，13N，14Y

15Y

如果大部分问题都回答正确，且在这过程中没有感到困惑，那么就不需要再阅读这篇文章了。

本文将讲解浏览器工作的一些原理和解析过程，希望能有些帮助。

## 二、浏览器工作原理

浏览器的主要组件，包含用户界面、浏览器引擎、呈现引擎、网络、用户界面后端、JavaScript解释器、数据存储。

1.  用户界面： 包括地址栏、前进/后退按钮、书签菜单等。除了浏览器主窗口显示的您请求的页面外，其他显示的各个部分都属于用户界面。
2.  浏览器引擎：在用户界面和呈现引擎之间传送指令。
3.  呈现引擎：负责显示请求的内容。如果请求的内容是 HTML，它就负责解析 HTML 和 CSS 内容，并将解析后的内容显示在屏幕上。
4.  网络：用于网络调用，比如 HTTP 请求。其接口与平台无关，并为所有平台提供底层实现。
5.  用户界面后端：用于绘制基本的窗口小部件，比如组合框和窗口。其公开了与平台无关的通用接口，而在底层使用操作系统的用户界面方法。
6.  JavaScript 解释器：用于解析和执行 JavaScript 代码。
7.  数据存储：这是持久层。浏览器需要在硬盘上保存各种数据，例如 Cookie。新的 HTML 规范 (HTML5) 定义了“网络数据库”，这是一个完整（但是轻便）的浏览器内数据库。

## 三、网页加载过程

浏览器加载解析渲染呈现的整个过程称为关键渲染路径（Critical Rendering Path），指的是浏览器从请求 HTML，CSS，JavaScript 文件开始，到将它们最终以像素输出到屏幕上这一过程。包括以下几个部分：

构建DOM树，构建CSSOM，构建Render，Layout和Paint，这些过程中存在这一定的交叉。

下面简述浏览器加载解析渲染的整个过程：

1、浏览器通过URL进行域名解析，向服务器发起请求，接受文件

2、**构建DOM树**：浏览器获取一个HTML页面时，会自上而下加载，并逐行解析HTML标签，解析器根据HTML文档开始构建**DOM树**。

	**HTML文档解析过程**：
	
	1）浏览器接收到显示字节内容的html文件；
	
	2）浏览器将字节内容转换为字符文件；
	
	3）浏览器将字符文件解析成许多Tokens（符号标签）；
	
	4）将Token解析为节点对象Objects；
	
	5）将节点对象Objects连接起来形成DOM树；

3、**构建CSSOM树**：浏览器在HTML文档解析过程中，遇到外部CSS文件，浏览器会另外发起请求去获取CSS文件并进行解析构建CSSOM树，并不会阻塞网页加载过程，不影响解析HTML文件。

	**解析CSS过程**：
	
	步骤与HTML文档解析类似：
	
	1）浏览器接收到字节内容的CSS文件；
	
	2）浏览器将字节内容转换为字符文件；
	
	3）浏览器将字符文件解析成许多Tokens（符号标签）；
	
	4）将Token解析为节点对象Objects；
	
	5）将节点对象Objects连接起来构建成CSSOM树（CSS Rule Tree）；

4、加载过程中，遇到图片资源，也是另发起一个请求获取图片资源，也不会影响HTML文档的加载。

5、**JavaScript**：在加载过程中，当遇到外链的JS脚本文件或`&lt;script&gt;`等标签内的JS代码时，会**暂停**HTML标签的解析，挂起HTML文档加载解析渲染的线程，加载JS脚本文件，使用JavaScript引擎对JS进行解析，通过DOM API 和CSSOM API来操作 DOM Tree和 CSS Rule Tree。等到JS文件加载、解析完毕，才会继续解析HTML，恢复HTML文档的渲染线程。

	**解析JS代码过程**：

  1）浏览器会请求js代码，这时HTML文件的解析会停下来；

  2）但是CSS文件的解析不会停止，所以会继续构造出CSSOM树；

  3）在构建CSSOM树时，返回的js文件并不会执行，在CSSOM树构建完成，才会运行JS文件。

  4）JS文件执行结束后，HTML继续解析并构建出DOM树，进行下一步。

6、**构建渲染树**：在DOM树和CSSOM树构建完成后，浏览器会结合DOM Tree和CSS Rule Tree 构建**呈现树/渲染树**（Rendering Tree）；

7、布局阶段：获取渲染树结构、节点位置和大小等；

8、绘制元素：根据渲染树和布局绘制所有节点的可视属性。

9、合并渲染层：把以上绘制的所有图层（类似于PhotoShop中的“图层”）合并,最终输出一张图片

当用户在浏览网页时进行交互或通过 js 脚本改变页面结构时，以上的部分操作有可能重复运行，此过程称为 Repaint 或 Reflow。

&gt; **Repaint**：当元素改变时，不会影响元素在页面当中的位置（比如 background-color, border-color, visibility），浏览器仅仅会应用新的样式重绘此元素，此过程称为 Repaint。
&gt;
&gt; Reflow：当元素改变时，将会影响文档内容、结构或元素位置，此过程称为 Reflow。（ HTML 使用的是 flow based layout ，即流式布局，所以，如果某元件的几何尺寸发生了变化，需要重新布局，也就叫 Reflow。）



## 四、浏览器解析

解析 HTML 文档涉及三个主要过程：HTML 解析器、URL 解析器和 JavaScript 解析器。每个解析器负责解码和解析其文档，并且在相应的解析器规范中明确指定了它们应该如何执行此操作。

### 4.1 HTML解析

上文说过浏览器将收到的HTML文件的字节内容转换为字符文件，然后将字符文件解析成许多Tokens（符号标签），在这一过程用到的就是HTML解析器。

参考[HTML标记规范](https://html.spec.whatwg.org/multipage/parsing.html)，我们只需关注如何完成文档解码和何时创建新token。

首先了解一下HTML解析器的工作原理：

- HTML解析器是一个状态机，它自上而下对HTML资源从进行解析，当遇到一个`&lt;`符号（后面没有`/`）时，就会进入标签开始状态（Tag Open State）；
- 然后进入标签名称状态（Tag name state）搜寻标签，img可以被识别为正确的标签，img1则不会识别。
- *后面可能还有其他很多种状态，如：&#34;[Before attribute name state](https://html.spec.whatwg.org/multipage/parsing.html#before-attribute-name-state)&#34;、“[Attribute name state](https://html.spec.whatwg.org/multipage/parsing.html#attribute-name-state)”等，我们先不提，有兴趣可以去看HTML标记规范。*
- 当最后在读到最近的一个`&gt;`时，结束标签状态进入数据状态（Data State），发出当前标签token。

当解析器处于“数据状态”时，它会继续进行HTML解析处理并在发现完整标记时发出token。

**HTML字符实体解码**

在解析过程中遇到HTML字符实体编码，HTML解析器处于以下三种状态时，能够对HTML字符实体进行解码：

- “数据状态”（&#34;Character reference in data state&#34;；
- “RCDATA状态”（&#34;Character reference in RCDATA state&#34;）
- “属性值状态”（&#34;Character reference in attribute value state&#34;）。

这些状态下，HTML 实体将从其“`&amp;#..;`”形式中解码出来，变成相应的字符插入到数据缓冲区。

也就是说，HTML字符实体编码只能在数据状态（标签外部和标签的text段），RCDATA状态（RCDATA元素内）、属性值状态（标签内的属性值位置）才能被解析。

#### 1.数据状态

&gt; #### 13.2.5.1 Data state
&gt;
&gt; Consume the [next input character](https://html.spec.whatwg.org/multipage/parsing.html#next-input-character):
&gt;
&gt; - U&#43;0026 AMPERSAND (&amp;)
&gt;
&gt;   Set the [return state](https://html.spec.whatwg.org/multipage/parsing.html#return-state) to the [data state](https://html.spec.whatwg.org/multipage/parsing.html#data-state). Switch to the [character reference state](https://html.spec.whatwg.org/multipage/parsing.html#character-reference-state).
&gt;
&gt; - U&#43;003C LESS-THAN SIGN (&lt;)
&gt;
&gt;   Switch to the [tag open state](https://html.spec.whatwg.org/multipage/parsing.html#tag-open-state).
&gt;
&gt; - U&#43;0000 NULL
&gt;
&gt;   This is an [unexpected-null-character](https://html.spec.whatwg.org/multipage/parsing.html#parse-error-unexpected-null-character) [parse error](https://html.spec.whatwg.org/multipage/parsing.html#parse-errors). Emit the [current input character](https://html.spec.whatwg.org/multipage/parsing.html#current-input-character) as a character token.
&gt;
&gt; - EOF
&gt;
&gt;   Emit an end-of-file token.
&gt;
&gt; - Anything else
&gt;
&gt;   Emit the [current input character](https://html.spec.whatwg.org/multipage/parsing.html#current-input-character) as a character token.

当在数据状态，即标签外部，或者标签内的text段中，匹配到&amp;符号时，会切换到字符引用状态（character reference state）。

例如问题4，“&lt;”和“&gt;”字符在输入流中被编码为“`&amp;#60;`”和“`&amp;#62;`”，当解析器处于“数据状态”时，它们将被解析。

```html
4.实体编码&lt; &gt;
&lt;div&gt;&amp;#60;img src=x onerror=alert(4)&amp;#62;&lt;/div&gt;
```

那么当HTML实体被解析为`&lt; &gt;`时，它们中的脚本能否被执行呢？

答案是不能，因为当解析器在解析实体编码后，不会转换成“标签打开状态”，因此不会创建新的标签。

这就是我们为什么建议用字符实体编码来处理不受信任的用户输入，以确保它们作为数据进行解析。

再看，我们对h1的h进行HTML实体编码，它能被解析形成标签吗？

```html
&lt;div&gt;
  &lt;&amp;#104;1&gt;123&lt;/h1&gt;
&lt;/div&gt;
```

答案是不能，HTML解析器首先匹配到div标签，进入数据状态，接着搜索到&lt;，进入标签名称状态，但并没有匹配到正确的标签，所以回到了数据状态，对`&amp;#104;`进行解码，最后返回的结果就是文本状态下的`&lt;h1&gt;123`。



#### 2.RCDATA状态

什么是RCDATA？我们需要知道HTML中共有5种元素：空元素、原始文本元素、RCDATA元素、外来元素以及常规元素。RCDATA状态就是RCDATA元素标签内的状态。

**Void 元素**：area, base, br, col, embed, hr, img, input, keygen, link, menuitem, meta, param, source, track, wbr。Void元素没有结束标签，因此不能在开始标签和结束标签直接放任何内容。

**Raw text 元素**：script, style。Raw text元素能含有text文本内容。

**RCDATA 元素**：textarea, title。RCDATA 元素可以有文本和字符引用

**Foreign 元素**：MathML 命名空间和 SVG 命名空间的元素。外部元素可以有文本、字符引用、CDATA 部分、其他元素和注释。

**Normal 元素**：除了以上4种元素以外的其他所有允许的 HTML 元素都是普通元素。普通元素可以有文本、字符引用、其他元素和注释。

&gt; ##### 13.2.5.2 RCDATA state
&gt;
&gt; Consume the [next input character](https://html.spec.whatwg.org/multipage/parsing.html#next-input-character):
&gt;
&gt; - U&#43;0026 AMPERSAND (&amp;)
&gt;
&gt;   Set the [return state](https://html.spec.whatwg.org/multipage/parsing.html#return-state) to the [RCDATA state](https://html.spec.whatwg.org/multipage/parsing.html#rcdata-state). Switch to the [character reference state](https://html.spec.whatwg.org/multipage/parsing.html#character-reference-state).
&gt;
&gt; - U&#43;003C LESS-THAN SIGN (&lt;)
&gt;
&gt;   Switch to the [RCDATA less-than sign state](https://html.spec.whatwg.org/multipage/parsing.html#rcdata-less-than-sign-state).
&gt;
&gt; - U&#43;0000 NULL
&gt;
&gt;   This is an [unexpected-null-character](https://html.spec.whatwg.org/multipage/parsing.html#parse-error-unexpected-null-character) [parse error](https://html.spec.whatwg.org/multipage/parsing.html#parse-errors). Emit a U&#43;FFFD REPLACEMENT CHARACTER character token.
&gt;
&gt; - EOF
&gt;
&gt;   Emit an end-of-file token.
&gt;
&gt; - Anything else
&gt;
&gt;   Emit the [current input character](https://html.spec.whatwg.org/multipage/parsing.html#current-input-character) as a character token.

之前我们说“RCDATA 状态”下可以对HTML字符引用进行解码。这意味着“textarea”和“title”中的字符引用将被 HTML 解析器解码。

同样，在解码这些字符引用时没有进入“标签打开状态”，这就是问题 5 中没有脚本执行的原因。

```html
5.实体编码&lt; &gt;
&lt;textarea&gt;&amp;#60;script&amp;#62;alert(5)&amp;#60;/script&amp;#62;&lt;/textarea&gt;
```

此外，RCDATA还有一个特别之处。当浏览器解析到 RCDATA 元素时，它会进入“RCDATA 状态”。在这种状态下，如果遇到&#39;&lt;&#39;字符，就会切换到“RCDATA小于号状态”（[RCDATA less-than sign state](https://html.spec.whatwg.org/multipage/parsing.html#rcdata-less-than-sign-state)），如果后面紧跟的不是相应的RCDATA元素闭合标签，则会回到“RCDATA状态”。

这意味着在 RCDATA 元素（例如` &lt;textarea&gt;`、`&lt;title&gt;`）的标签内部，唯一能够被识别为标签的字符是 ”`&lt;/textarea&gt;`”闭合标签或“`&lt;/title&gt;`”闭合标签，具体取决于前面打开的标签。

因此，即便在“textarea”或“title”中创建额外的标签，也无法在其中执行脚本。这就解释了为什么问题6中脚本不执行。

```html
&lt;textarea&gt;&lt;script&gt;alert(6)&lt;/script&gt;&lt;/textarea&gt;
```

&gt; 关于“CDATA”元素的简单说明。CDATA 中包含的任何内容都不会导致解析器创建新的开放标记，并且它以“]]&gt;”序列结束。
&gt;
&gt; 因此，如果用户输入想要跳出 CDATA 上下文，必须使用没有任何编码的精确“]]&gt;”序列，否则它不会跳出上下文。

所以当我们遇到`&lt;textarea&gt;`或`&lt;title&gt;`标签下的XSS插入点时，首先要做的就是闭合前面的标签，这样后续的payload才可能被执行，如：

```html
&lt;/title&gt;&lt;script&gt;alert(1)&lt;/script&gt;
&lt;/textarea&gt;&lt;script&gt;alert(2)&lt;/script&gt;
```



##### Foreign 元素

细心的同学会发现上文我们提到Foreign元素也可以有字符引用。

&gt; **Foreign 元素**：MathML 命名空间和 SVG 命名空间的元素。外部元素可以有文本、字符引用、CDATA 部分、其他元素和注释。

这类情况，也是属于数据状态解析HTML字符引用，看示例：

```html
编码&lt;img src onerror=alert(1)&gt;
&lt;math&gt;&amp;lt;&amp;#105;&amp;#109;&amp;#103;&amp;#32;&amp;#115;&amp;#114;&amp;#99;&amp;#32;&amp;#111;&amp;#110;&amp;#101;&amp;#114;&amp;#114;&amp;#111;&amp;#114;&amp;equals;&amp;#97;&amp;#108;&amp;#101;&amp;#114;&amp;#116;&amp;lpar;&amp;#49;&amp;rpar;&amp;gt;&lt;/math&gt;
&lt;svg&gt;&amp;lt;&amp;#105;&amp;#109;&amp;#103;&amp;#32;&amp;#115;&amp;#114;&amp;#99;&amp;#32;&amp;#111;&amp;#110;&amp;#101;&amp;#114;&amp;#114;&amp;#111;&amp;#114;&amp;equals;&amp;#97;&amp;#108;&amp;#101;&amp;#114;&amp;#116;&amp;lpar;&amp;#49;&amp;rpar;&amp;gt;&lt;/svg&gt;
```

上述示例可以被解码，但与之前提过的一样，无法建立img标签，因此无法执行XSS。

我们可以看几个可以执行的SVG和MathML命名空间的XSS：

```html
&lt;svg&gt;&lt;script&gt;alert&amp;#40;1&amp;#41;&lt;/script&gt;
&lt;svg&gt;&lt;script&gt;&amp;#97;&amp;#108;&amp;#101;&amp;#114;&amp;#116;&amp;#40;1&amp;#41;&lt;/script&gt;

&lt;math&gt;&lt;img src onerror=alert&amp;#40;2&amp;#41;&gt;&lt;/math&gt;
```

可以看到，还是在`script`标签内的数据状态和`img`标签的属性值状态处解析后进入了JavaScript解析器，从而触发XSS。

#### 3.属性值状态

“属性值状态下的字符引用”（&#34;Character reference in attribute value state&#34;），它可以细分为无引号、单引号、双引号3种状态下的HTML实体引用。

看原文：

&gt; ##### 13.2.5.36 Attribute value (double-quoted) state
&gt;
&gt; Consume the [next input character](https://html.spec.whatwg.org/multipage/parsing.html#next-input-character):
&gt;
&gt; - U&#43;0022 QUOTATION MARK (&#34;)
&gt;
&gt;   Switch to the [after attribute value (quoted) state](https://html.spec.whatwg.org/multipage/parsing.html#after-attribute-value-(quoted)-state).
&gt;
&gt; - U&#43;0026 AMPERSAND (&amp;)
&gt;
&gt;   Set the [return state](https://html.spec.whatwg.org/multipage/parsing.html#return-state) to the [attribute value (double-quoted) state](https://html.spec.whatwg.org/multipage/parsing.html#attribute-value-(double-quoted)-state). Switch to the [character reference state](https://html.spec.whatwg.org/multipage/parsing.html#character-reference-state).
&gt;
&gt; - U&#43;0000 NULL
&gt;
&gt;   This is an [unexpected-null-character](https://html.spec.whatwg.org/multipage/parsing.html#parse-error-unexpected-null-character) [parse error](https://html.spec.whatwg.org/multipage/parsing.html#parse-errors). Append a U&#43;FFFD REPLACEMENT CHARACTER character to the current attribute&#39;s value.
&gt;
&gt; - EOF
&gt;
&gt;   This is an [eof-in-tag](https://html.spec.whatwg.org/multipage/parsing.html#parse-error-eof-in-tag) [parse error](https://html.spec.whatwg.org/multipage/parsing.html#parse-errors). Emit an end-of-file token.
&gt;
&gt; - Anything else
&gt;
&gt;   Append the [current input character](https://html.spec.whatwg.org/multipage/parsing.html#current-input-character) to the current attribute&#39;s value.
&gt;
&gt; ##### 13.2.5.37 Attribute value (single-quoted) state
&gt;
&gt; Consume the [next input character](https://html.spec.whatwg.org/multipage/parsing.html#next-input-character):
&gt;
&gt; - U&#43;0027 APOSTROPHE (&#39;)
&gt;
&gt;   Switch to the [after attribute value (quoted) state](https://html.spec.whatwg.org/multipage/parsing.html#after-attribute-value-(quoted)-state).
&gt;
&gt; - U&#43;0026 AMPERSAND (&amp;)
&gt;
&gt;   Set the [return state](https://html.spec.whatwg.org/multipage/parsing.html#return-state) to the [attribute value (single-quoted) state](https://html.spec.whatwg.org/multipage/parsing.html#attribute-value-(single-quoted)-state). Switch to the [character reference state](https://html.spec.whatwg.org/multipage/parsing.html#character-reference-state).
&gt;
&gt; - U&#43;0000 NULL
&gt;
&gt;   This is an [unexpected-null-character](https://html.spec.whatwg.org/multipage/parsing.html#parse-error-unexpected-null-character) [parse error](https://html.spec.whatwg.org/multipage/parsing.html#parse-errors). Append a U&#43;FFFD REPLACEMENT CHARACTER character to the current attribute&#39;s value.
&gt;
&gt; - EOF
&gt;
&gt;   This is an [eof-in-tag](https://html.spec.whatwg.org/multipage/parsing.html#parse-error-eof-in-tag) [parse error](https://html.spec.whatwg.org/multipage/parsing.html#parse-errors). Emit an end-of-file token.
&gt;
&gt; - Anything else
&gt;
&gt;   Append the [current input character](https://html.spec.whatwg.org/multipage/parsing.html#current-input-character) to the current attribute&#39;s value.
&gt;
&gt; ##### 13.2.5.38 Attribute value (unquoted) state
&gt;
&gt; Consume the [next input character](https://html.spec.whatwg.org/multipage/parsing.html#next-input-character):
&gt;
&gt; - U&#43;0009 CHARACTER TABULATION (tab)
&gt;
&gt; - U&#43;000A LINE FEED (LF)
&gt;
&gt; - U&#43;000C FORM FEED (FF)
&gt;
&gt; - U&#43;0020 SPACE
&gt;
&gt;   Switch to the [before attribute name state](https://html.spec.whatwg.org/multipage/parsing.html#before-attribute-name-state).
&gt;
&gt; - U&#43;0026 AMPERSAND (&amp;)
&gt;
&gt;   Set the [return state](https://html.spec.whatwg.org/multipage/parsing.html#return-state) to the [attribute value (unquoted) state](https://html.spec.whatwg.org/multipage/parsing.html#attribute-value-(unquoted)-state). Switch to the [character reference state](https://html.spec.whatwg.org/multipage/parsing.html#character-reference-state).
&gt;
&gt; - U&#43;003E GREATER-THAN SIGN (&gt;)
&gt;
&gt;   Switch to the [data state](https://html.spec.whatwg.org/multipage/parsing.html#data-state). Emit the current tag token.
&gt;
&gt; - U&#43;0000 NULL
&gt;
&gt;   This is an [unexpected-null-character](https://html.spec.whatwg.org/multipage/parsing.html#parse-error-unexpected-null-character) [parse error](https://html.spec.whatwg.org/multipage/parsing.html#parse-errors). Append a U&#43;FFFD REPLACEMENT CHARACTER character to the current attribute&#39;s value.
&gt;
&gt; - U&#43;0022 QUOTATION MARK (&#34;)
&gt;
&gt; - U&#43;0027 APOSTROPHE (&#39;)
&gt;
&gt; - U&#43;003C LESS-THAN SIGN (&lt;)
&gt;
&gt; - U&#43;003D EQUALS SIGN (=)
&gt;
&gt; - U&#43;0060 GRAVE ACCENT (`)
&gt;
&gt;   This is an [unexpected-character-in-unquoted-attribute-value](https://html.spec.whatwg.org/multipage/parsing.html#parse-error-unexpected-character-in-unquoted-attribute-value) [parse error](https://html.spec.whatwg.org/multipage/parsing.html#parse-errors). Treat it as per the &#34;anything else&#34; entry below.
&gt;
&gt; - EOF
&gt;
&gt;   This is an [eof-in-tag](https://html.spec.whatwg.org/multipage/parsing.html#parse-error-eof-in-tag) [parse error](https://html.spec.whatwg.org/multipage/parsing.html#parse-errors). Emit an end-of-file token.
&gt;
&gt; - Anything else
&gt;
&gt;   Append the [current input character](https://html.spec.whatwg.org/multipage/parsing.html#current-input-character) to the current attribute&#39;s value.

例如下面字符实体编码，都将被解码还原：

```html
实体编码&#34;javascript&#34;
&lt;h1 id=&amp;#x6a;&amp;#x61;&amp;#x76;&amp;#x61;&amp;#x73;&amp;#x63;&amp;#x72;&amp;#x69;&amp;#x70;&amp;#x74;&gt;123&lt;/h1&gt;
&lt;h1 id=&#39;&amp;#x6a;&amp;#x61;&amp;#x76;&amp;#x61;&amp;#x73;&amp;#x63;&amp;#x72;&amp;#x69;&amp;#x70;&amp;#x74;&#39;&gt;456&lt;/h1&gt;
&lt;h1 id=&#34;&amp;#x6a;&amp;#x61;&amp;#x76;&amp;#x61;&amp;#x73;&amp;#x63;&amp;#x72;&amp;#x69;&amp;#x70;&amp;#x74;&#34;&gt;789&lt;/h1&gt;
```

在属性值下的HTML实体编码，是平时XSS编码的一种常用方式，如下述payload：

```html
编码alert(/xss/)
&lt;svg/onload=&amp;#x0061;&amp;#x006c;&amp;#x0065;&amp;#x0072;&amp;#x0074;&amp;#x0028;&amp;#x002f;&amp;#x0078;&amp;#x0073;&amp;#x0073;&amp;#x002f;&amp;#x0029;&gt;
  
&lt;a/href=javascript&amp;colon;alert&amp;grave;1&amp;grave;&gt;123&lt;/a&gt;
&lt;a href=&#34;javascri&amp;#x0070;t:alert(1)&#34;&gt;link&lt;/a&gt;
&lt;img src onerror=alert&amp;#40;2&amp;#41;&gt;
```



### 4.2 URL解析

URL 解析器也可以视为状态机，输入流中的字符可以将其定向到不同的状态。[URL解析器规范在这里](https://url.spec.whatwg.org/)。

从安全或 XSS 转义的角度来看，有几件事很有趣。

首先，URL 格式必须是 ASCII 字母字符。 (U&#43;0041-U&#43;005A || U&#43;0061-U&#43;007A)，否则状态会转变为“无格式”状态。

例如，如果对协议格式进行任何形式的编码，URL 解析器都将无法识别为协议。问题 1 中的脚本不执行就是因为URL 解析器没有将url 编码的“javascript”识别为协议。

```html
1.URL编码&#34;javascript:alert(1)&#34;
&lt;a href=&#34;%6a%61%76%61%73%63%72%69%70%74:%61%6c%65%72%74%28%31%29&#34;&gt;&lt;/a&gt;
3.URL编码&#34;:&#34;
&lt;a href=&#34;javascript%3aalert(3)&#34;&gt;&lt;/a&gt;
```

同样的理论也适用于 &#39;:&#39; 字符，如果对其进行编码将无法识别。这就是问题 3 中的脚本不会执行的原因。

那么，当使用字符实体对方案 (javascript) 进行编码时，为什么问题 2 中的脚本会执行？

```html
&lt;a href=&#34;&amp;#x6a;&amp;#x61;&amp;#x76;&amp;#x61;&amp;#x73;&amp;#x63;&amp;#x72;&amp;#x69;&amp;#x70;&amp;#x74;:%61   %6c%65%72%74%28%32%29&#34;&gt;
```

这里就要用到刚在HTML 解析部分讨论过的内容，有一种状态称为“属性值状态”，在这种状态下，字符引用将被解码并替换为解码后的版本。

```html
&lt;a href=&#34;javascript:%61   %6c%65%72%74%28%32%29&#34;&gt;
```

这里实际发生的是HTML解析器解析文档、创建标签tokens并解码 href 属性中的HTML字符实体。

当 HTML 解析器完成后，URL 解析器开始解析 href 值中的链接，此时“javascript”已经解码，并被 URL 解析器识别为JavaScript协议。

然后 URL 解析器继续对链接的后续部分进行 URL 解码。因为是“javascript”协议，所以后续使用JavaScript 解析器来解析执行，这就是问题 2 中的脚本被执行的原因。

:warning:*注意：URL编码是使用UTF-8编码方案对每个字符进行编码。如果使用另一种方案对URL进行编码，则 URL 解析器可能无法识别，这可能会变得失效。*

url解析器识别相应的协议，我们上面提到JavaScript伪协议之外，通过http协议加载外部JS文件，也是一种方式。

在对外部链接的处理上，还可以设计一些URI的变形，不过这些不在本文进行赘述。

### 4.3 JavaScript解析

JavaScript 解析不同于 HTML 解析，因为JS语言本身是上下文无关的语言(context free language)，，可以通过上下文无关语法(context free grammar)来弄清楚 JavaScript 是如何被解析的。

ECMAScript-262 的规范在[这里](http://www.ecma-international.org/publications/standards/Ecma-262.htm)，一个单独的HTML解析语法文件在[这里](https://html.spec.whatwg.org/multipage/parsing.html#parsing)。

JS的脚本处理模型是按照源码处理-函数解析-代码执行这个顺序进行，不管是外部JS还是`&lt;script&gt;` 标签内，或者 HTML标签属性中的，对JS编码的解码都是相同的。

#### 1. 执行上下文

当一段 JavaScript 代码在运行的时候，它实际上是运行在执行上下文中。而下列三种代码都会创建相应的执行上下文：

- **全局执行上下文**：它是为运行代码主体而创建的执行上下文，也就是说它是为那些存在于函数之外的任何代码而创建的。
- **函数执行上下文**：每个函数会在执行的时候创建自己的执行上下文。
- **Eval 函数执行上下文**：使用 eval() 函数也会创建一个新的执行上下文。

这部分内容不是重点，就不多说了。

#### 2.字符解码

我们主要关注关于安全性、字符解码方式以及在某些上下文中转义是否有效这些问题。

首先，让我们稍微回到 HTML 解析中的&#34;Raw text&#34;元素，因为它与 JavaScript 解析有关。

所有“script”块都属于原始文本元素，“script”块有一个有趣的属性：字符引用实体在其中不被解析和解码。这意味着问题 9 中的脚本将不会执行。

```html
9.实体编码alert(9);
&lt;script&gt;&amp;#97;&amp;#108;&amp;#101;&amp;#114;&amp;#116&amp;#40;&amp;#57;&amp;#41;&amp;#59&lt;/script&gt;
```

那么像 \uXXXX（例如 \u0000、\u000A）这样的Unicode 转义序列字符编码的 JavaScript 会被解析和执行吗？答案是需要看情况，这取决于编码序列所处的位置。

根据上下文，有三种情况可以使用转义序列：字符串、标识符名称或控制字符。

**字符串**：当字符串中存在 Unicode 转义序列时，它们只会被解释为常规字符，而不是单双引号（&#39; &#34;）或可以结束字符串的行终止符。这在 ECMAScript 规范中有明确说明。因此，Unicode 转义序列永远不会脱离 JavaScript 中的字符串上下文，因为它们总是被解释为字符串文字。

&gt; *ECMA-262 edition13.0, Rev 11, Clause 1
&gt; &#34;ECMAScript differs from the Java programming language in the behaviour of Unicode escape sequences. In a Java program, if the Unicode escape sequence \u000A, for example, occurs within a single-line comment, it is interpreted as a line terminator (Unicode character 000A is line feed) and therefore the next Unicode character is not part of the comment. Similarly, if the Unicode escape sequence \u000A occurs within a string literal in a Java program, it is likewise interpreted as a line terminator, which is not allowed within a string literal—one must write \n instead of\u000A to cause a line feed to be part of the string value of a string literal. In an ECMAScript program, a Unicode escape sequence occurring within a comment is never interpreted and therefore cannot contribute to termination of the comment. Similarly, a Unicode escape sequence occurring within a string literal in an ECMAScript program always contributes a Unicode character to the literal and is never interpreted as a line terminator or as a quote mark that might terminate the string literal.*&#34;

翻译：

&gt; ECMA-262 版本13.0，第11章，第1条款
&gt;
&gt; ECMAScript 与 Java 编程语言的区别在于 Unicode 转义序列的行为。
&gt;
&gt; 在Java中，如果Unicode转义序列\u000A，出现在单行注释中，它被解释为行终止符（Unicode 字符 000A 是换行符），因此下一个 Unicode 字符将不是注释的一部分。同样，如果\u000A 出现在 Java 程序的字符串文字中，它同样被解释为行终止符，这在字符串文字中是不允许的——必须写 \n 而不是 \u000A 以使换行符成为字符串文字的一部分。
&gt;
&gt; 在 ECMAScript 程序中，注释中出现的 Unicode 转义序列永远不会被解释，因此不会导致注释的终止。类似地，在 ECMAScript 程序的字符串文字中出现的 Unicode 转义序列总是为文字提供一个 Unicode 字符，并且永远不会被解释为行终止符或可能字符串终止的引号。

**标识符名称**：当标识符名称中存在 Unicode 转义序列时，它们将被解码并解释为标识符的名称，例如函数名称、属性名称等。

这就是为什么问题 10 中的脚本是可执行的。

```html
&lt;script&gt;\u0061\u006c\u0065\u0072\u0074(10);&lt;/script&gt;
```

如果我们深入研究规范，它也清楚地说明如下。

&gt; ECMA-262 edition13.0, Rev 12, Clause 6.1
&gt;
&gt; *&#34;Unicode escape sequences are permitted in an IdentifierName, where they contribute a single Unicode code point to the IdentifierName. The code point is expressed by the CodePoint of the UnicodeEscapeSequence (see 12.8.4). The \ preceding the UnicodeEscapeSequence and the u and { } code units, if they appear, do not contribute code points to the IdentifierName. A UnicodeEscapeSequence cannot be used to put a code point into an IdentifierName that would otherwise be illegal. &#34;*

翻译：

&gt; 标识符名称中也允许使用 Unicode 转义序列，它们将作为标识符中的一个字符，由Unicode 转义序列的 CV 计算转换（见 7.8.4）。Unicode 转义序列前的 \ 不会转换为标识符名称中的一个字符。 Unicode 转义序列不能用于将字符放入标识符名称中，否则将是非法的。
&gt;
&gt; 标识符名称中允许使用 Unicode 转义序列，它们将作为标识符中的Unicode代码点。代码点由 Unicode 转义序列的 CodePoint 表示（见 12.8.4）。Unicode 转义序列之前的 \ 以及 u 和 { } 代码单元（如果有）不会作为标识符名称中的代码点。 Unicode 转义序列不能用于将代码点放入标识符名称，否则将是非法的。

**控制字符**：当 Unicode 转义序列表示控制字符时，如单引号、双引号、括号等，它们不会被解释为控制字符，只会被解码和解析为标识符名称或字符串文字。查看 ECMAScript 语法，就会发现没有任何可以充当控制字符的 Unicode 转义序列。例如，如果解析器正在解析函数调用语句，则括号必须是“(”和“)”，而不是像 \u0028 和 \u0029 这样的字符。

也就是说，Unicode 转义序列只有在标识符名称的上下文中才会被解释为字符串，这是唯一可以注入并利用的地方。

回顾问题 11不起作用，因为 &#39;(11)&#39; 没有被正确解释而且 &#39;alert(11)&#39; 不是有效的标识符名称。

```html
11.unicode编码alert(11)
&lt;script&gt;\u0061\u006c\u0065\u0072\u0074\u0028\u0031\u0031\u0029&lt;/script&gt;
```

问题 12 ， &#39;\u0031\u0032&#39; 不会被解释为字符串，因为它们必须以单引号或双引号开头，或者是 ASCII 数字。

```html
12.Unicode编码alert和12
&lt;script&gt;\u0061\u006c\u0065\u0072\u0074(\u0031\u0032)&lt;/script&gt;
```

问题 13 ，因为 &#39;\u0027&#39; 仅被解释为单引号文字，并没有起到控制字符的效果，导致字符串不完整。

```html
13.Unicode编码&#39;
&lt;script&gt;alert(&#39;13\u0027)&lt;/script&gt;
```

问题 14 有效，因为 &#39;\u000a&#39; 被解释为换行符，不会被当做行终止符，没有破坏 JavaScript 语法。

```html
14.Unicode编码换行符
&lt;script&gt;alert(&#39;14\u000a&#39;)&lt;/script&gt;
```

根据上述描述，以下编码都可以执行：

```html
&lt;script&gt; \u0061lert(&#34;Hello&#34;); &lt;/script&gt;
&lt;script&gt; alert(&#34;\u0048ello&#34;); &lt;/script&gt;
&lt;script&gt; alert(&#39;Hello\u000a&#39;);&lt;/script&gt;
```



## 五、浏览器解码顺序

在讨论了浏览器 HTML、URL 和 JavaScript 解析之后，所有这些如何一起发挥作用？接下来将非常简要地回顾浏览器如何解析文档。

- 当浏览器从网络堆栈获取内容时，HTML 解析器被触发并开始标记文档。这是完成HTML字符引用解码的步骤。标记化完成后，构建 DOM 树。

- JavaScript 解析器开始解析内联脚本和`script`块中的脚本。这是对 Unicode 转义序列和 Hex 转义序列进行解码的步骤。

- 同时，如果浏览器遇到需要 URL 的上下文，URL 解析器也会启动以解码 URL 内容。这是完成 URL 解码的步骤。根据 URL 的位置，URL 解析器可能会在 JavaScript 解析器进入之前或之后解析内容。

考虑以下两个示例。

```
Example A: &lt;a href=&#34;UserInput&#34;&gt;&lt;/a&gt;
Example B: &lt;a href=# onclick=&#34;window.open(&#39;UserInput&#39;)&#34;&gt;&lt;/a&gt;
```

在示例 A 中，HTML 解析器将首先启动并对用户输入执行字符引用解码。然后 URL 解析器开始对 href 中的值进行 URL 解码。最后，如果 URL scheme 是 javascript，则 JavaScript 解析器来执行 Unicode 转义序列和 Hex 转义序列解码。之后，脚本被执行。所以一共有三轮解码，依次是HTML、URL、JavaScript。

在示例 B 中，HTML 解析器也首先启动。但是之后，JavaScript 解析器开始解析 onclick 处理程序中的值，因为 onclick 处理程序中需要脚本。当 JavaScript 被解析和执行时，它看到它正在执行“window.open()”，其中参数应该是一个 URL。此时，URL 解析器开始对用户输入进行 URL 解码，并将结果传回 JavaScript 引擎。所以一共有三轮解码，依次是HTML、JavaScript、URL。

有没有可能超过三轮解码？考虑下面的例子

```
Example C: &lt;a href=&#34;javascript:window.open(&#39;UserInput&#39;)&#34;&gt;
```

示例 C 与 A 类似，但在 JavaScript 中存在“window.open”调用的意义上也有所不同。因此，将对 UserInput 进行额外的 URL 解码。一般情况下会进行四次解码，依次为HTML、URL、JavaScript、URL。

此时，您应该具备解决文章开头所有问题的必要知识。

## 六、总结

总而言之，为了了解如何使 XSS payload脱离攻击者的上下文，或者如何使应用程序服务器正确编码用户输入，就必须真正了解浏览器解析器的工作原理以及组件（HTML、URL 和 JavaScript 解析器）如何协同工作。

同样，如果想要绕过防御检测，使payload能够被正确解析执行，也必须要了解浏览器的工作原理和解析器的协调处理，只有在这种情况下，才能像浏览器一样正确地编码攻击载荷。



## 参考文章：

- [Deep dive into browser parsing and XSS payload encoding](https://www.attacker-domain.com/2013/04/deep-dive-into-browser-parsing-and-xss.html)
- [HTML标记规范](https://html.spec.whatwg.org/multipage/parsing.html)
- [URL规范](https://url.spec.whatwg.org/)
- [HTML 语法](https://www.w3.org/html/ig/zh/wiki/HTML5/syntax)

- [浏览器解析HTML，渲染页面的过程](https://blog.csdn.net/xawjz/article/details/120515355)
- [深入理解浏览器解析渲染 HTML](https://zhuanlan.zhihu.com/p/445810614 )
- [浏览器工作原理与XSS-HTML编码](https://www.jianshu.com/p/c0dc4bbab8e8)
- [理解 JavaScript 的执行上下文这篇就够了！](https://juejin.cn/post/6954966248233009182)


---

> 作者: Xavier  
> URL: http://localhost:1313/posts/2023-6-%E6%B5%8F%E8%A7%88%E5%99%A8xss%E7%BC%96%E7%A0%81%E8%A7%A3%E6%9E%90%E5%8E%9F%E7%90%86/  

