# JavaScript杂记

> 这个MD笔记，主要用来记录一些零散的看到和JavaScript相关的不错文章。
>
> 许久没再看这些最最最最基础同样也是很重要的概念了，有些内容已经记不清楚了

# 1. JavaScript组成

> [BOM和DOM详解](https://www.cnblogs.com/wangxiang9528/p/9855358.html)	<=	下面内容是对该文章的整理

## 1.1 JavaScript的组成-概述

​	JavaScript由三部分构成，ECMAScript，DOM和BOM，根据宿主（浏览器）的不同，具体的表现形式也不尽相同，IE和其他的浏览器风格迥异。

1. DOM 是W3C 的标准； [所有浏览器公共遵守的标准]
2. BOM 是各个浏览器厂商根据 DOM在各自浏览器上的实现;[表现为不同浏览器定义有差别,实现方式不同]
3. **window 是BOM 对象，而非 js 对象**；

## 1.2 DOM、BOM - 简述

+ **DOM（文档对象模型）是 HTML 和 XML 的应用程序接口（API）**。

+ **BOM（浏览器对象模型）主要处理浏览器窗口和框架**，不过<u>通常浏览器特定的 JavaScript 扩展都被看做 BOM 的一部分</u>。这些扩展包括：

  + 弹出新的浏览器窗口
  + 移动、关闭浏览器窗口以及调整窗口大小
  + 提供 Web 浏览器详细信息的定位对象
  + 提供用户屏幕分辨率详细信息的屏幕对象
  + 对 cookie 的支持

  IE 扩展了 BOM，加入了 ActiveXObject 类，可以通过 JavaScript 实例化 ActiveX 对象

## 1.3 BOM、DOM、window、document之间的关系

​	JavaScript是通过访问BOM（Browser Object Model）对象来访问、控制、修改客户端(浏览器)，<u>由于**BOM的window包含了document**，window对象的属性和方法是直接可以使用而且被感知的，因此可以直接使用window对象的document属性，通过document属性就可以访问、检索、修改XHTML文档内容与结构</u>。

​	**document对象是DOM（Document Object Model）模型的根节点**。

​	<u>**可以说，BOM包含了DOM(对象)，浏览器提供出来给予访问的是BOM对象，从BOM对象再访问到DOM对象，从而js可以操作浏览器以及浏览器读取到的文档**</u>。

---

Window对象包含属性：

+ document
+ location
+ navigator
+ screen
+ history
+ frames

Document根节点包含子节点

+ forms
+ location
+ anchors
+ images
+ links

**从window.document已然可以看出，DOM的最根本的对象是BOM的window对象的子对象**。

*ps-区别：**DOM描述了处理网页内容的方法和接口，BOM描述了与浏览器进行交互的方法和接口**。*

## 1.3 认识DOM

先看段HTML代码

```html
<!DOCTYPE html PUBLIC "-//W3C//DTD XHTML 1.0 Transitional//EN" "http://www.w3.org/TR/xhtml1/DTD/xhtml1-transitional.dtd">
<html xmlns="http://www.w3.org/1999/xhtml" xml:lang="en">
  <head> 
    <meta http-equiv="Content-Type" content="text/html;charset=UTF-8" /> 
    <title>DOM</title> 
  </head> 
  <body> 
    <h2><a href="http://www.baidu.com">javascript DOM</a></h2> 
    <p>对HTML元素进行操作，可添加、改变或移除css样式等</p> 
    <ul> 
      <li>Javascript</li> 
      <li>DOM</li> 
      <li>CSS</li> 
    </ul>  
  </body>
</html>
```

将HTML代码分解为DOM节点层次图：

![img](https://p.ssl.qhimg.com/t017a40571dccf818c3.jpg)

HTML文档可以说是由**节点**构成的集合，**DOM节点**有：

1. **元素节点**：上图中\<html\>、\<body\>、\<p\>等都是元素节点，即标签。
2. **文本节点**：向用户展示的内容，如\<li\>...\</li\>中的JavaScript、DOM、CSS等文本。

3. **属性节点**：元素属性，如\<a\>标签的链接属性`href="http://www.baidu.com"`。

### 1.3.1 DOM节点

#### 1.3.1.1 DOM节点属性

+ nodeName返回一个字符串，其内容是节点的名字
+ nodeType返回一个整数，这个数值代表给定节点的类型
+ nodeValue返回给定节点的当前值

#### 1.3.1.2 DOM节点树

​	遍历DOM节点树childNodes时返回一个数组，这个数组由给定元素的子节点构成。

+ firstChild返回第一个子节点
+ lastChild返回最后一个子节点
+ parentNode返回一个给定节点的父节点
+ nextSibling返回给定节点的下一个子节点
+ previousSibling返回给定节点的上一个子节点

#### 1.3.1.3 DOM节点操作

+ `creatElement(element)`创建一个新的元素节点
+ `creatTextNode()`创建一个包含给定文本的新文本节点
+ `appendChild()`在指定节点的最后一个节点列表后添加一个新的子节
+ `insertBefore()`将一个给定节点插入到一个给定元素节点的给定子节点的前面
+ `removeChild()`从一个给定元素中删除子节点
+ `replaceChild()`把一个给定父元素里的一个子节点替换为另外一个节点

​	**DOM通过创建树来表示文档，描述了处理网页内容的方法和接口，从而使开发者对文档的内容和结构具有空前的控制力，用DOM API可以轻松地删除、添加和替换节点**。

#### 1.3.1.4 DOM节点访问

+ `var oHtml = document.documentElement;`	

  返回存在于 XML 以及 HTML 文档中的文档根节点，oHtml包含了一个表示\<html /\>的HTMLElement对象 

+ `document.body`

  对 HTML 页面的特殊扩展，提供了对 \<body\> 标签的直接访问

+ `document.getElementById("ID")`

  通过指定的 ID 来返回元素，`getElementById()` 无法工作在 XML 中，IE6还会返回name为指定ID的元素

+ `document.getElementByName("name")`

  获取所有name特性等于指定值的元素，不过在IE6和Opera7.5上还会返回id为给定名称的元素且仅检查\<input/\>和\<img/\>

+ `var x = document.getElementsByTagName("p");`

  **使用指定的标签名返回所有的元素列表NodeList**，索引号从0开始。<u>当参数是一个星号的时候，IE6并不返回所有的元素，必须用document.all来替代</u>

#### 1.3.1.5 Node节点的特性和方法

> [dom中Node和Element有什么区别？ 请高手详细说明，在下不胜感激！](https://zhidao.baidu.com/question/1500016267709286299.html?qbl=relate_question_0&word=dom%BA%CDnode)

+ firstChild

  ​	Node，指向在childNodes列表中的第一个节点

+ lastChild

  ​	Node，指向在childNodes列表中的最后一个节点

+ parentNode

  ​	Node，指向父节点

+ **ownerDocument**

  ​	**Document，指向这个节点所属的文档**

+ childNodes

  ​	NodeList，所有子节点的列表

+ previousSibling

  ​	Node，指向前一个兄弟节点（如果这个节点就是第一个节点，那么该值为 null）

+ nextSibling

  ​	Node，指向后一个兄弟节点（如果这个节点就是最后一个节点，那么该值为null）

+ `hasChildNodes()`

  ​	Boolean，当childNodes包含一个或多个节点时，返回真值

### 1.3.2 DOM事件

> [终于弄懂了事件冒泡和事件捕获！](https://blog.csdn.net/chenjuan1993/article/details/81347590)	<=	有更具体的图和例子，建议阅读

DOM同时两种事件模型：冒泡型事件和捕获型事件。

+  **冒泡型事件：事件按照从最特定的事件目标到最不特定的事件目标的顺序触发**

  ```html
  <html>
   <head></head>
   <body onclick="handleClick()"> 
    <div onclick="handleClick()">
     Click Me
    </div>  
   </body>
  </html>
  ```

  触发的顺序是：div、body、html(IE 6.0和Mozilla 1.0)、document、window(Mozilla 1.0) 

+ **捕获型事件：与冒泡事件相反的过程，事件从最不精确的对象开始触发，然后到最精确** 

  触发顺序和上述的相反

​	<u>DOM事件模型最独特的性质是，文本节点也触发事件</u>(在IE中不会)。

#### 1.3.2.1 事件处理函数/监听函数

+ 在JavaScript中：`var oDiv = document.getElementById("div1");`

  onclick只能用小写，默认为冒泡型事件

+ 在HTML中：`<div onclick="javascript: alert("Clicked!")"></div>`

  onclick大小写任意

#### 1.3.2.2 IE事件处理程序attachEvent()和detachEvent()

​	在IE中，每个元素和window对象都有两个方法：`attachEvent()`和`detachEvent()`，这两个方法接受两个相同的参数，事件处理程序名称和事件处理程序函数，如：

+ `[object].attachEvent("name_of_event_handler","function_to_attach")`
+ `[object].detachEvent("name_of_event_handler","function_to_remove")`

```javascript
var fnClick = function(){ alert("Clicked!");
oDiv.attachEvent("onclick", fnClick); //添加事件处理函数
oDiv.attachEvent("onclick", fnClickAnother); // 可以添加多个事件处理函数
oDiv.detachEvent("onclick", fnClick); //移除事件处理函数
```

Ps: **在直接使用`attachEvent()`方法的情况下，事件处理程序会在全局作用域中运行，因此this等于window**。

#### 1.3.2.3 跨浏览器的事件处理程序

> [EventUtil对象详解](https://blog.csdn.net/featureme/article/details/66972365)

`addHandler()`和`removeHandler()`

​	`addHandler()`方法属于一个叫`EventUntil()`的对象，这两个方法均接受三个相同的参数，要操作的元素，事件名称和事件处理程序函数。

#### 1.3.2.4 事件类型

+ 鼠标事件：click、dbclick、mousedown、mouseup、mouseover、mouseout、mousemove
+ 键盘事件：keydown、keypress、keyup
+ HTML事件：load、unload、abort、error、select、change、submit、reset、 resize、scroll、focus、blur

#### 1.3.2.5 事件处理器

​	**执行JavaScript 代码的程序在事件发生时会对事件做出响应。为了响应一个特定事件而被执行的代码称为事件处理器**。

​	在HTML标签中使用事件处理器的语法是：

```html
<HTML标签 事件处理器="JavaScript代码''>
```

#### 1.3.2.6 事件处理程序

​	**事件就是用户或浏览器自身执行的某种动作**。比如click,mouseup,keydown,mouseover等都是事件的名字。而<u>**响应某个事件的函数就叫事件处理程序（事件监听器）**，事件处理程序以on开头，因此click的事件处理程序就是onclick</u>。

![img](https://p.ssl.qhimg.com/t01c02c8f06206c5b14.png)

![img](https://p.ssl.qhimg.com/t01be3974fa86338f8e.png)

#### 1.3.2.7 DOM 0级事件处理程序

​	**DOM 0级事件处理程序：把一个函数赋值给一个事件的处理程序属性**

```html
<html>
  <body>
    <input type="button" value="按钮2" id="ben2"/>
  </body>
  <script>
  	var btn2=document.getElementById('btn2'); // 获得btn2按钮对象
    btn2.onclick //给btn2添加onclick属性，属性又触发了一个事件处理程序
    btn2.onclick=function(){ } //添加匿名函数
    btn2.onclick=null //删除onclick属性
  </script>
</html>
```

#### 1.3.2.8 阻止事件冒泡

阻止冒泡有以下方法：

+ `e.cancelBubble=true; `
+ `e.stopPropagation(); `
+ `return false;`

#### 1.3.2.9 innerText、innerHTML、outerHTML、outerText

> [JS中innerHTML、outerHTML、innerText 、outerText、value的区别与联系？jQuery中的text()、html()和val() ？](https://www.cnblogs.com/jongsuk0214/p/6930876.html)

+ innerText：表示起始标签和结束标签之间的文本

+ innerHTML：表示元素的所有元素和文本的HTML代码

  如：`<div><b>Hello</b> world</div>`的innerText为"Hello world"，innerHTML为"\<b\>Hello\</b\> world" 

+ outerText：与前者的区别是替换的是整个目标节点，返回和innerText一样的内容

+ outerHTML: 与前者的区别是替换的是整个目标节点，返回元素完整的HTML代码，包括元素本身

一张图了解OUTHTML和innerText、innerHTML：

![img](https://p.ssl.qhimg.com/t0188f422f3ee960d59.gif)

#### 1.3.2.10 DOM 2级事件处理程序

`addEventListener()`和`removeEventListener()`

​	**在DOM中，`addEventListener()`和`removeEventListener()`用来分配和移除事件处理函数**。

​	与IE不同的是，这些方法需要三个参数：

+ 事件名称
+ 要分配的函数
+ **处理函数是用于冒泡阶段(false)还是捕获阶段(true)，默认为冒泡阶段false** 

`[object].addEventListener("name_of_event",fnhander,bcapture)`

`[object].removeEventListener("name_of_event",fnhander,bcapture)`

```javascript
var fnClick = function(){ alert("Clicked!"); }
oDiv.addEventListener("onclick", fnClick, false); //添加事件处理函数
oDiv.addEventListener("onclick", fnClickAnother, false); // 与IE一样，可以添加多个事件处理函数
oDiv.removeEventListener("onclick", fnClick, false); //移除事件处理函数
```

 	**如果使用`addEventListener()`将事件处理函数加入到捕获阶段，则必须在`removeEventListener()`中指明是捕获阶段，才能正确地将这个事件处理函数删除**。

+ `oDiv.onclick = fnClick; oDiv.onclick = fnClickAnother;`

  **使用直接赋值，后续的事件处理函数会覆盖前面的处理函数**

+ `oDiv.onclick = fnClick; oDiv.addEventListener("onclick", fnClickAnother, false); `

  会按顺序进行调用，不会覆盖

### 1.3.3 DOM基本操作思维导图

![img](https://p.ssl.qhimg.com/t0174118af570f2e5b1.gif)

*ps：更详细的XML DOM - Element 对象的属性和方法请访问w3cshool*

## 1.4 BOM

### 1.4.1 BOM概述

​	**BOM的核心是window，而window对象又具有双重角色，它既是通过js访问浏览器窗口的一个接口，又是一个Global（全局）对象**。<u>这意味着在网页中定义的任何对象，变量和函数，都以window作为其global对象。</u>

#### 1.4.1.1 window

+ `window.close();` 

  关闭窗口 

+ `window.alert("message");` 

  弹出一个具有OK按钮的系统消息框，显示指定的文本

+ `window.confirm("Are you sure?");`

  弹出一个具有OK和Cancel按钮的询问对话框，返回一个布尔值

+ `window.prompt("What's your name?", "Default");`

  提示用户输入信息，接受两个参数，即要显示给用户的文本和文本框中的默认值，将文本框中的值作为函数值返回

+ `window.status`

  可以使状态栏的文本暂时改变

+ `window.defaultStatus`

  默认的状态栏信息，可在用户离开当前页面前一直改变文本

+ `window.setTimeout("alert('xxx')", 1000);`

  设置在指定的毫秒数后执行指定的代码，接受2个参数，要执行的代码和等待的毫秒数

+ `window.clearTimeout("ID");`

  取消还未执行的暂停，将暂停ID传递给它

+ `window.setInterval(function, 1000);`

  无限次地每隔指定的时间段重复一次指定的代码，参数同setTimeout()一样 

+ `window.clearInterval("ID");`

  取消时间间隔，将间隔ID传递给它

+ `window.history.go(-1);`

  访问浏览器窗口的历史，负数为后退，正数为前进

+ `window.history.back();`

  同上 

+ `window.history.forward();` 

  同上

+ `window.history.length`

  可以查看历史中的页面数

----

windows对象方法

![img](https://p.ssl.qhimg.com/t01dfaedc0f147a414d.jpg)

window对象思维导图

![img](https://p.ssl.qhimg.com/t016c684c2c50e6468a.gif)

#### 1.4.1.2 document对象

> [HTML \<embed\> 标签](https://www.w3school.com.cn/tags/tag_embed.asp)

​	**document对象：实际上是window对象的属性，document == window.document为true，是唯一一个既属于BOM又属于DOM的对象。**

+ `document.lastModified`

  **获取最后一次修改页面的日期的字符串表示**

+ `document.referrer`

  **用于跟踪用户从哪里链接过来的**

+ `document.title`

  获取当前页面的标题，可读写

+ `document.URL`

  **获取当前页面的URL，可读写**

+ `document.anchors[0]`或`document.anchors["anchName"]`

  访问页面中第一个/所有的**锚**

+ `document.forms[0]`或`document.forms["formName"]`

  访问页面中第一个/所有的表单

+ `document.images[0]`或`document.images["imgName"]`

  访问页面中第一个/所有的图像 

+ `document.links [0]`或`document.links["linkName"]`

  访问页面中第一个/所有的链接

+ `document.applets [0]`或`document.applets["appletName"]`

  访问页面中第一个/所有的Applet 

+ `document.embeds[0]`或`document.embeds["embedName"]`

  **访问页面中第一个/所有的嵌入式对象**

+ `document.write(); `或`document.writeln();`

  将字符串插入到调用它们的位置

#### 1.4.1.3 location对象

​	**location对象：表示载入窗口的URL**，<u>也可用window.location引用它</u>

+ location.href

  **当前载入页面的完整URL**，如`http://www.somewhere.com/pictures/index.htm`

+ location.portocol

  **URL中使用的协议，即双斜杠之前的部分，如http**

+ location.host

  服务器的名字，`如www.wrox.com`

+ location.hostname

  通常等于host，有时会省略前面的`www`

+ location.port

  URL声明的请求的端口，默认情况下，大多数URL没有端口信息，如8080

+ location.pathname

  **URL中主机名后的部分**，如`/pictures/index.htm`

+ location.search

  **执行GET请求的URL中的问号后的部分，又称查询字符串**，如`?param=xxxx`

+ location.hash

  如果URL包含#，返回该符号之后的内容，如#anchor1

+ `location.assign("http:www.baidu.com");`

  **同location.href，新地址都会被加到浏览器的历史栈中**

+ `location.replace("http:www.baidu.com");`

  **同`assign()`，但新地址不会被加到浏览器的历史栈中，不能通过back和forward访问**

+ `location.reload(true | false);`

  **重新载入当前页面，为false时从浏览器缓存中重载，为true时从服务器端重载，默认为false**

#### 1.4.1.4 navigator对象

​	**navigator对象，包含大量有关Web浏览器的信息，在检测浏览器及操作系统上非常有用，也可用window.navigator引用它**。

+  navigator.appCodeName

  浏览器代码名的字符串表示

+ navigator.appName

  官方浏览器名的字符串表示

+ navigator.appVersion

  浏览器版本信息的字符串表示

+ navigator.cookieEnabled

  如果启用cookie返回true，否则返回false

+ navigator.javaEnabled

  如果启用java返回true，否则返回false

+ navigator.platform

  **浏览器所在计算机平台的字符串表示**

+ navigator.plugins

  **安装在浏览器中的插件数组**

+ navigator.taintEnabled

  如果启用了数据污点返回true，否则返回false

+ navigator.userAgent

  用户代理头的字符串表示

#### 1.4.1.5 screen对象

​	**screen对象，用于获取某些关于用户屏幕的信息，也可用window.screen引用它。**

+ screen.width/height

  屏幕的宽度与高度，以像素计

+ screen.availWidth/availHeight

  窗口可以使用的屏幕的宽度和高度，以像素计

+ screen.colorDepth

  用户表示颜色的位数，大多数系统采用32位

+ `window.moveTo(0, 0);`和`window.resizeTo(screen.availWidth, screen.availHeight);`

  填充用户的屏幕

## 1.4.2 BOM和DOM之间的关系

![img](https://p.ssl.qhimg.com/t0187afc90d98985e41.jpg)



