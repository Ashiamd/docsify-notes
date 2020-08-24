# OAuth学习杂记

## 1. 网文汇集

+ Token、JWT

> [SpringBoot集成JWT实现token验证](https://www.jianshu.com/p/e88d3f8151db)
>
> [存储JWT令牌的最佳MYSQL或Maria DB数据类型是什么？](http://www.voidcn.com/article/p-avmcopfc-bwm.html)
>
> [Spring Security用户名密码登陆和token授权登陆两种实现](https://blog.csdn.net/markfengfeng/article/details/100171744)
>
> [spring-security实现的token授权](https://www.cnblogs.com/lori/p/10538592.html)
>
> [基于 Token 的身份验证：JSON Web Token(JWT)](https://www.jianshu.com/p/2036987a22fb)
>
> [退出登录使JWT失效的解决方案](https://www.jianshu.com/p/2f8b6591a09d)
>
> [JWT简要说明](https://www.cnblogs.com/hellxz/p/12041701.html)
>
> [JWT签名算法中HS256和RS256有什么区别](https://www.jianshu.com/p/cba0dfe4ad4a)
>
> [jwt与token+redis，哪种方案更好用？](https://www.zhihu.com/question/274566992)

+ Outh2

> [微信网页授权,微信登录,oauth2](https://www.cnblogs.com/boundless-sky/p/6056664.html)
>
> [SpringBoot 整合 oauth2（三）实现 token 认证](https://www.jianshu.com/p/19059060036b)
>
> [OAuth2介绍与使用](https://www.jianshu.com/p/4f5fcddb4106)
>
> [OAuth 2.0 的一个简单解释](https://www.ruanyifeng.com/blog/2019/04/oauth_design.html)
>
> [OAuth 2.0 的四种方式](https://www.ruanyifeng.com/blog/2019/04/oauth-grant-types.html)
>
> [关于 OAuth2.0 安全性你应该要知道的一些事](https://www.chrisyue.com/security-issue-about-oauth-2-0-you-should-know.html)
>
> [基于oauth2.0实现应用的第三方登录](https://www.cnblogs.com/yunche/p/10695430.html)
>
> [OAuth2.0认证和授权机制讲解](https://www.tianmaying.com/tutorial/oAuth-login)
>
> [性能优化（Oauth2应用场景）](https://my.oschina.net/u/3728166/blog/2996622)
>
> [Spring Security OAuth专题学习-密码模式JWT实现](https://blog.csdn.net/icarusliu/article/details/87968090)
>
> [spring security oauth2 自动刷新续签token (refresh token)](https://blog.csdn.net/m0_37834471/article/details/83213002)
>
> [Spring Security OAuth2 Demo —— 授权码模式 (Authorization Code)](https://www.cnblogs.com/hellxz/p/oauth2_oauthcode_pattern.html)
>
> [Spring Security OAuth2 Demo —— 隐式授权模式（Implicit）](https://www.cnblogs.com/hellxz/p/oauth2_impilit_pattern.html)
>
> [Spring Security OAuth2 Demo —— 密码模式（Password）](https://www.cnblogs.com/hellxz/p/12041495.html)
>
> [Spring Security OAuth2 Demo —— 客户端模式（ClientCredentials）](https://www.cnblogs.com/hellxz/p/12041588.html)
>
> [使用JWT作为Spring Security OAuth2的token存储](https://www.cnblogs.com/hellxz/p/12044340.html)
>
> [理解OAuth2.0认证与客户端授权码模式详解](https://segmentfault.com/a/1190000010540911)

+ SSO

> [keycloak学习](https://www.cnblogs.com/weschen/p/9530044.html)

+ 对比

> [jwt、oauth2和oidc等认证授权技术的理解](https://www.cnblogs.com/tuzongxun/p/11677307.html)
>
> [认证 (authentication) 和授权 (authorization) 的区别](https://www.cnblogs.com/jinhengyu/p/10257792.html)
>
> [JWT和Oauth2的区别和联系](https://www.jianshu.com/p/1870f456b334)





## 2. 普通token登录验证和OAuth2.0应用场景(个人简述,之后有空再详写)

> 之前2月底特地看了很多关于token机制和Oauth2.0的文章。由于3月初到现在每天都很忙，所以一直没时间整理。今天其实也还是没时间。不过现大概写一些东西。后面有空在补上吧。之前忙着发现杯的作品提交材料，最近则是忙着Netty开发。

在正式介绍Token之前，先大致回顾一下Cookie和Session。（下面介绍跨度比较大，涉及内容不止Cookie和Session）
### 1. Cookie
​	Cookie一般是指**客户端**浏览器的Cookie，别的地方有没有Cookie我就不清楚了。Cookie可以用于保存一些**非隐私**的公用状态数据。
​	比如早期的前端页面开发中，由于刷新页面，JS代码的变量就会失效，必须通过别的形式使一些变量能够持久化。这时候可以`考虑把变量存在Cookie中，使其持久化`。当然，这也不是什么最优的方式。

​	*类似的形式，把参数保存在URL中，保存在公用的JS文件中（题外话，如果是Vue的话，在一些路由操作，多页面需要公用的状态参数，可以考虑使用VUEX）*
​	总之，Cookie就是一种保存**非私密**状态数据的方案，作用于客户端，往往只浏览器。由于Cookie不安全，不建议存储私密数据。现在很多网站开发，也尽量避免使用Cookie了。

### 2. Session

​	Session同样也是保存状态的一种方式。不过Session存在于**服务器**。由于管理状态的是服务器，那么相对Cookie就安全许多了。比如你把你的账号密码都存在服务器中，反正服务器也不会随便把你的账号密码发给其他人。但是Cookie是在你的浏览器的，假如存了账号密码，指不定就被人盗走了。

​	**Session在单机环境比较实用**，在集群or分布式环境就比较麻烦了。*这里先扯一会Cookie吧，你用不同浏览器访问你的QQ邮箱啥的，每次都需要登录，因为是不同浏览器，Cookie在浏览器中存储，那么不同浏览器当然Cookie不共享了（不同浏览器对HTML、JS、CSS的实现和兼容也是有区别，IE出来挨打，哈哈）。*

​	为什么说Session在集群or分布式环境比较不方便呢？这里再扯一会别的东西。啥是集群，啥是分布式？说的土一点，集群就是同一个应用/服务，复制粘贴多个相同的去运行。分布式就是不同服务放到不同服务器上。那为什么闲着慌有人要搞集群or分布式？？集群的话，你可以假设你其中一个挂了，那么只要集群中其他的相同服务还在，用户就可以正常访问了。分布式的话，如果是提供相同服务的分布式，一般是为了容灾处理，集群能够保证程序级别的容灾（比如一个java程序挂了，同一台服务器集群的java程序还或者就ok了），而分布式你一台服务器挂了，我这台服务器还在就ok了。不过一般分布式不只是为了容灾，分布式比较灵活，通常把不同服务包装成一个中台放到不同服务器上运行，这些服务器加起来形成一个大服务，这样你可以把这几台服务器叫作分布式服务了。

​	回到正题，Session在集群中，虽然集群提供相同的服务，但是毕竟是不同的程序，不同程序的类、方法、变量怎么可能是同一个？要是你使用Session保存用户的状态（账号、密码啥的隐私数据），你第一次浏览器访问 “张三.com”，你登录了一次，服务器A中的服务1-java程序记录了你的登录信息，你刷新了一下，访问到了服务器A的服务2-java程序，又要求你重新登录一遍，因为它没有你的信息，只有服务1-java程序有。作为用户，你崩溃了吗？赶紧投诉电话打起来。这里你可能会困惑，为什么你访问“张三.com”会访问到不同的应用，不是一个域名对应一个服务器吗？域名是标识了一个服务器没错，但是一个服务内的多个应用不都是这个服务器的，只要后台使用Ribbon、Nginx等负载均衡、代理机制，就能使得用户请求被分发到不同的应用程序上。

​	集群中，Session保存状态可能出现以下情况：

（1）用户 -> 浏览器 -> 用户登录请求 -> 服务器A ->  应用1 -> Session1记录登录信息 -> 用户登录成功，不用再登录啦，恭喜你。

（2）用户，哈哈，不用登录了 -> 浏览器 -> 服务器A -> 负载均衡，转发请求 -> 应用2，Session2找不到用户信息 -> 抱歉你谁我不知道，你登录一下。 -> 用户，投诉走起。

​	Session要在集群中同步也不是不行，可以通过**第三者同步Session**。比如你再开一个服务5，它就专门存Session，其他几个服务1，2，3，4啥的，每次都从服务5那里取Session，也可以干脆把用户登录状态都存在数据库，每次都从数据库读取记录，读取到特定数据就表示用户已经登录。这里就提供一点思路，如果真要同步集群的Session，可以考虑成熟的开源框架Spring Session啥的。

### 3. Token

+ 前面提到Cookie，一般存储非隐私数据，且保存在客户端。

+ Session，可以存储隐私数据，保存在服务器端（集群、分布式烦琐）。

这里，讲token。首先，token也是用于保存一些数据的，它一般用于用户的**认证/鉴权**。

啥是认证，啥又是鉴权？？？

+ **认证**，个人理解就是`保证用户是这个用户`。也就是现在操作微博的人是一个叫“张三”的，他的密码是“隔壁老王”，这个用户的信息，我后台数据库有存储过，这个人确实存在。那么他可以访问“张三”的微博，可以写说说。
+ **鉴权**，个人理解就是`保证这个用于有权限`。也就是B站直播的普通直播观看用户“张三”，和UP主的房管“李四”的区别。张三只能发发弹幕喷UP主，但是“李四”看了可来气了，反手就是禁言"李四"一年。这里李四用户权限比张三高。（ps：实际中，鉴权不限于用户之间，也可以是服务之间。比如现在有3台服务器A、B、C，每个服务器上有不同的服务，服务器A有A-1，A-2，A-3，服务器B有B-1、B-2......，A-1可以访问B-1，但是没有权限访问B-2，这也是一种鉴权。不过这个我觉得主要是OAuth2.0的应用范畴了。）

知道认证/鉴权后，我们回到Token。token凭啥本事做登录鉴权？？我们在设计token时，一般往里面存用户状态信息，过期时间等参数。比如下面的数据结构

```none
Token

{

	String	equipId;//标识设备
    String username;//标识用户
    .... 其他信息
	TimeStamp TTL;// timeToLive,存活时间
}
```

使用上面token的方式如下流程：

（1）用户登录->输入账号密码->HTTPS（HTTP无状态，HTTPS多了SSL，较安全）-> 后台服务器。

（2）后台服务器-> 数据库验证是否存在该用户->存在，则**颁发token**->用户

（3）用户->收到token，之后每次请求都需要携带token。->header添加字段携带token

（4）后台->收到header有token的请求，检验token的信息，这个token没过期，该username的用户不需要再登录，可以继续访问。

上面只是简化版的说法，实际可以更复杂，比较忙我就不细讲了。比如颁发token的时候，最好对数据进行**对称加密**（防止被别人直接看到私密数据），然后进行**数字签名**（防止数据被伪造，HTTPS也是主要干的这个，但是HTTPS不只是防伪造，实际上也算是能加密信息了。）再发给用户。了解HTTPS需要知道对称加密、非对称加密、CA（够写一篇文章了，有空再说吧）。

​	使用了token，如果是浏览器，可以把token存放在cookie中，然后浏览器发出请求的时候，要么发送cookie给服务器，要么把token加到header发送给服务器。**服务器不需要特定存储token**，只要验证里面的时间戳属性是否过期就行。

​	这里，你可能有以下疑问：

1. token的信息真的真的不用存在服务器中吗？不怕伪造吗
2. 如果token的信息服务器“不存储”，那么怎么验证token的。
3. 假如token的部分信息存在服务器的话，那么用户恶意用脚本注册上千次，是不是服务器就要存上千次信息了？？
4. token被盗用怎么办?

​	这里一个一个解答：

1. 前面提到服务器不需要特定存token，指的是不需要像Session那样针对某个和自己连接的客户端创建一个专门存储用户信息的对象（对象指的就是Session）。

   至于伪造问题，如果你token传输用的HTTPS，而且你为了双重保险，token还是经过了非对称加密（数字签名，HTTPS的话多一步CA认证->包装成数字证书），那么你的数据还能被伪造，只能说黑客技高一筹。

2. 如果是简单的服务，各种安全性啥的没什么考虑，确实token不需要特地存储，只要用户登录操作需要后台颁发token时，从数据库验证用户账号密码正确就可以颁发token给用户了。况且如果用了非对称加密，本身这个token的解密密钥只有服务器有，token也难以被伪造。

3. 这个问题的话，我个人觉得光是靠token就不大行了。解决方案可以是使用Redis数据库（之所以用Redis，一方面快，还有就是Redis单线程，任务处理就好像阻塞队列一样，不会出现多线程同步问题。）。

   具体可以这样设计。使用username作为key，value 为Set类型，Set只存放3-5个值，每个值为username.equipId，每次颁发Token的时候，先查看这个Set的大小，要是>=3,则直接使用里面的equipId为用户签发token，而不另外新建。

   另外，可以使用username.equipId为key，val为其他私密数据（比如token的密钥等信息）的String类型存放 `用户的过期时间`。一般网站不都有7天内免登录吗？可以设置这个String7天过期，前面提到的Set也是7天过期。这样就可以做到7天内用户免登录+用户最多3-5个设备能够颁发token（具体别的细节不说）。token为了安全性，最好失效时间短一点，比如1小时or半小时失效，那么只要这个7天的String类型变量还在，就允许token刷新重新颁发（或者延用上次的equipId，看你实际业务）。

4. 被盗用，那你做出标准的祈祷姿势，希望黑客无法解密对称加密，哈哈。不过如果token的存活时间短，黑客可能只能拥有你的token一个小时。被盗用，黑客可以直接用你的身份免登录（当然后台其实还是可以有对策的，这里不细讲了。）

​	总之，使用了token，虽然需要每次都传输token，但是相对cookie安全，相对session不存在分布式/集群同步问题（因为验证都是通过数据库，不同服务可能连接的同一个数据库）。

### OAuth2.0

​	OAuth2.0的概念我这里不说了，文章不错的有很多，这里不细贴了。要是感兴趣又懒得找文章的，可以去我[github的笔记]([https://ashiamd.github.io/docsify-notes/#/study/%E7%BD%91%E7%BB%9C%E5%AE%89%E5%85%A8/OAuth/OAuth%E5%AD%A6%E4%B9%A0%E6%9D%82%E8%AE%B0](https://ashiamd.github.io/docsify-notes/#/study/网络安全/OAuth/OAuth学习杂记))看看。其实这篇文章我本地MD笔记也有，以后再上传到github的笔记上。

​	这里不细讲OAuth2.0的使用啥的。我就说说个人理解的OAuth2.0的使用场景和Token应用场景。

+ Token：前面也能感觉到token被盗用很蛋疼。比如黑客拿到后，要是他也能续签token，那你岂不是7天内（假如后台免密登录7天）账号都被黑客控制了？
+ OAuth2.0在认证/授权中，有个很厉害的东西，叫作`授权码模式`,这个要说也是一篇文章，不细讲。你就只要知道里面有个**core授权码**，它就厉害在可消费，也就是一个token只能对应一个core。假如黑客拿你的token去续签，他获得了一个新core。后面你要续签了，你获得的core和黑客获取的不一样，这个token失效。这时候你可能就发现异常了，于是修改了密码（一般修改密码，后台需要让token都失效，实现形式很多，也可以是一篇文章，不细讲了。）

​	OAuth2.0如果要以之前简单token的方式用于登录认证，那么**OAuth2.0可以比简单token更适合分布式**。前面简单token机制，需要每个服务器的服务都编写相同的token验证，非常低效（当然不是真的所有实现都这样，只不过简单实现往往就是如此）。而OAuth2.0往往用于独立的认证/鉴权服务，就好像前面说的Session同步需要多个服务去第三方获取Session。这里OAuth2.0把认证/鉴权独立出来，其他服务要认证/鉴权全找它就好了。

​	还有，OAuth2.0使用其实很频繁，比如你网站用微信登录，github登录等**第三方登录，走的就是OAuth2.0**.

### 简单Token和OAuth2.0使用场景对比

​	如果是多台服务器分布式服务，且服务多样化，比如有3个以上的服务，那么建议使用OAuth2.0，把认证/鉴权的工作独立出来（这个认证/鉴权也可以是服务和服务之间的，而且**个人觉得，OAuth2.0一般也就是用于服务和服务之间**）。

​	如果是只有单种服务，或者只有2种服务，可以不考虑抽出OAuth2.0，用简单的token机制就ok了。不过，**个人觉得，简单token机制往往用于客户端和服务之间**。

​	总结：

+ 分布式多服务（3个以上），考虑OAuth2.0
+ 第三方登录（第三方也可以是自己的服务），不用想了，OAuth2.0
+ 单个或2个服务，不需要服务器之间认证/鉴权，只要客户端与服务器，简单Token

最后说句题外话，token实现可以考虑使用JWT，一个成熟的token实现方式。