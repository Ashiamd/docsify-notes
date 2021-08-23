# CAS学习杂记

> 理论文章
>
> - [单点登录（SSO）看这一篇就够了 - 简书 (jianshu.com)](https://www.jianshu.com/p/75edcc05acfd)
>- [cas 单点登录 登出流程说明_这是一个懒人的博客-CSDN博客_cas单点退出流程](https://blog.csdn.net/qq_30062125/article/details/84983764)
> - [单点登录CAS实现中，感觉只要TGC就足够了，干嘛还要ST的_百度知道 (baidu.com)](https://zhidao.baidu.com/question/427773428303757572.html)
>
>   1. 单点登录的过程中，第一步应用服务器将请求重定向到认证服务器，用户输入账号密码认证成功后，只是在浏览器和认证服务器之间建立了信任(TGC)，但是浏览器和应用服务器之间并没有建立信任。
>
>   2. ST是CAS认证中心认证成功后返回给浏览器，浏览器带着它去访问应用服务器，应用服务器再凭它去认证中心验证你这个用户是否合法。只有这样，浏览器和应用服务器才能建立信任的会话。
>  3. 而TGC的作用主要是用于实现单点登录，就是当浏览器要访问应用服务器2时，应用服务器2也会重定向到认证服务器，但是此时由于TGC的存在，认证服务器信任了该浏览器，就不需要用户再输入账号密码了，直接返回给浏览器ST，重复2中的步骤。
> - [CAS票据之ST与TGT过期策略详细说明_Java精选-CSDN博客](https://blog.csdn.net/afreon/article/details/53183157) <= 结合下面的大流程图，很清晰了。
> - [CAS实现单点登录SSO执行原理探究(终于明白了)_javaloveiphone的专栏-CSDN博客_cas sso原理](https://blog.csdn.net/javaloveiphone/article/details/52439613)  <=  极力推荐，超高赞文章！（多图流、完整登录退出流程）
>
> 实战文章
>
> - [基于CAS实现单点登录（SSO）：工作原理_时光在路上-CSDN博客](https://blog.csdn.net/tch918/article/details/19930037)
>
> - [基于CAS的SSO搭建详细图文_胡云台的博客-CSDN博客](https://blog.csdn.net/qq_36879870/article/details/88544468)
>
> - [基于CAS的SSO(单点登录)实例 - 梦玄庭 - 博客园 (cnblogs.com)](https://www.cnblogs.com/java-meng/p/7269990.html)
>
> - [单点登录_HealerJean梦想博客-CSDN博客](https://blog.csdn.net/u012954706/category_7482523.html)
>
> - [实战springboot+CAS单点登录系统-B站视频](https://www.bilibili.com/video/BV1xy4y1r7BU)
>
>   下面1.CAS流程 ~ 3. 运行CAS-server根据该视频学习，后面看client的教程一般，就看个大概没往下了。 
>
> - [SpringBoot 简单实现仿CAS单点登录系统_ljk126wy的博客-CSDN博客](https://blog.csdn.net/ljk126wy/article/details/90640608)    <=.   可参考的文章

# 1. CAS流程

> - [单点登录（SSO）看这一篇就够了 - 简书 (jianshu.com)](https://www.jianshu.com/p/75edcc05acfd)
> - [cas 单点登录 登出流程说明_这是一个懒人的博客-CSDN博客_cas单点退出流程](https://blog.csdn.net/qq_30062125/article/details/84983764)

![img](https://upload-images.jianshu.io/upload_images/12540413-041b3228c5e865e8.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

---

- TGT：Ticket Granted Ticket（票根，可以签发ST）
- TGC：Ticket Granted Cookie（cookie中CASTGC的value），存在Cookie中，可以通过他找到TGT。
- ST：Service Ticket，是TGT生成的，是每个应用的票据，就是流程中的ticket

单点登录：

![img](https://img-blog.csdnimg.cn/2018121310572451.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzMwMDYyMTI1,size_16,color_FFFFFF,t_70)

 单点退出（也可以是先访问应用系统，看具体怎么实现）：

![img](https://img-blog.csdnimg.cn/20181213104925514.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzMwMDYyMTI1,size_16,color_FFFFFF,t_70)

 

# 2. 搭建Tomcat启用HTTPS

 CAS实现的SSO，需要HTTPS保证数据安全。

> + **jdk1.8.0_261**
> + **apache-tomcat-9.0.52**
> + **cas-server-webapp-tomcat-5.3.14.war**

## 2.1 生成密钥和证书

> [Keytool命令详解_老猿说说专栏-CSDN博客_keytool](https://blog.csdn.net/zlfing/article/details/77648430)
>
> [Windows下如何把安全证书导入到JDK的cacerts证书库_liruiqing的专栏-CSDN博客](https://blog.csdn.net/liruiqing/article/details/80416740)
>
> Keytool 是一个Java 数据证书的管理工具，Keytool 将密钥（key）和证书（certificates）存在一个称为keystore的文件中。
>
> 在keystore里，包含两种数据： 
>
> + 密钥实体（Key entity）——密钥（secret key）又或者是私钥和配对公钥（采用非对称加密） 
> + 可信任的证书实体（trusted certificate entries）——只包含公钥
>
> ailas(别名)每个keystore都关联一个独一无二的alias，这个alias通常不区分大小写
>
> -keystore 指定密钥库的名称(产生的各类信息将不在.keystore文件中)
>
> -keyalg 指定密钥的[算法](http://lib.csdn.net/base/datastructure) (如 RSA DSA（如果不指定默认采用DSA）)
>
> -v 显示密钥库中的证书详细信息
>
> -list 显示密钥库中的证书信息   keytool -list -v -keystore 指定keystore -storepass 密码
>
> -storepass  指定密钥库的密码(获取keystore信息所需的密码)
>
> -export   将别名指定的证书导出到文件 keytool -export -alias 需要导出的别名 -keystore 指定keystore -file 指定导出的证书位置及证书名称 -storepass 密码
>
> -file 参数指定导出到文件的文件名
>
> +  查看JDK的cacerts 中的证书列表：
>
> `keytool -list -v -keystore "/Library/Java/JavaVirtualMachines/jdk1.8.0_261.jdk/Contents/Home/lib/security/cacerts" -storepass changeit`
>
> + 删除的cacerts 中指定名称的证书：
>
> `keytool -delete -alias 证书别名 -keystore "/Library/Java/JavaVirtualMachines/jdk1.8.0_261.jdk/Contents/Home/lib/security/cacerts" -storepass changeit`

1. keystore生成

   `keytool -genkey -v -alias 别名 -keyalg RSA -keystore ./密钥库名.keystore`

2. keystore查看

    `keytool -list -v -keystore ./密钥库名.keystore`

3. 从密钥库导出证书

   `keytool -export -alias 别名 -file ./证书名.crt -keystore ./密钥库名.keystore`

4. 查看证书

   `keytool -printcert -file 证书名.crt`

5. 证书导入到JDK证书库

   `keytool -import -trustcacerts -alias 指定导入证书的别名 -file ./证书名.crt -keystore "/Library/Java/JavaVirtualMachines/jdk1.8.0_261.jdk/Contents/Home/lib/security/cacerts" -storepass changeit`

> **之后实验完之后，记得把这个我们自己新建的cert证书从JDK证书库中删除！！**
>
> `keytool -delete -alias 证书别名 -keystore "/Library/Java/JavaVirtualMachines/jdk1.8.0_261.jdk/Contents/Home/lib/security/cacerts" -storepass changeit`

## 2.2 配置Tomcat支持HTTPS

> [启动tomcat: Permission denied错误_Pompeii的专栏-CSDN博客](https://blog.csdn.net/Pompeii/article/details/40262785)
>
> [tomcat9.0配置https_程序员修炼之路的博客-CSDN博客](https://blog.csdn.net/qq_34756221/article/details/103957671)

1. 修改tomcat9安装目录下`conf/server.xml`

   ```xml
   <!-- myconfig -->
       <Connector port="8443" protocol="org.apache.coyote.http11.Http11AprProtocol"
                  maxThreads="150" SSLEnabled="true" 
                  scheme="https" secure="true" clientAuth="false" sslProtocol="TLS"
                  keystoreFile="/Users/ashiamd/mydocs/docs/study/ITstudy/SSO/CAS/cas-test/keystore/ashiamd.keystore"
                  keystorePass="keystore文件的密码" />
   ```

2. 启动tomcat

   ```shell
   $ bin/startup.sh
   
   Using CATALINA_BASE:   /Users/ashiamd/mydocs/dev-tools/tomcat/apache-tomcat-10.0.10
   Using CATALINA_HOME:   /Users/ashiamd/mydocs/dev-tools/tomcat/apache-tomcat-10.0.10
   Using CATALINA_TMPDIR: /Users/ashiamd/mydocs/dev-tools/tomcat/apache-tomcat-10.0.10/temp
   Using JRE_HOME:        /Library/Java/JavaVirtualMachines/adoptopenjdk-11-openj9.jdk/Contents/Home
   Using CLASSPATH:       /Users/ashiamd/mydocs/dev-tools/tomcat/apache-tomcat-10.0.10/bin/bootstrap.jar:/Users/ashiamd/mydocs/dev-tools/tomcat/apache-tomcat-10.0.10/bin/tomcat-juli.jar
   Using CATALINA_OPTS:   
   Tomcat started.
   ```

   > 如果Permission denied
   >
   > 则执行`chmod u+x *.sh`

3. 访问本地tomcat

   *（google内核的浏览器可能没有继续浏览本地签证的页面的选项，换safari浏览器就好了）*

   ```shell
   $ curl http://localhost:8080/
   
   $ curl https://localhost:8443/                      
   curl: (60) SSL certificate problem: self signed certificate
   More details here: https://curl.haxx.se/docs/sslcerts.html
   
   curl failed to verify the legitimacy of the server and therefore could not
   establish a secure connection to it. To learn more about this situation and
   how to fix it, please visit the web page mentioned above.
   
   $ curl --insecure https://localhost:8443
   ```

# 3. 运行CAS-server

1. 下载CAS-server的war包

   > [Central Repository: org/apereo/cas/cas-server-webapp-tomcat (maven.org)](https://repo1.maven.org/maven2/org/apereo/cas/cas-server-webapp-tomcat/) 	<=	从视频教程里找到的war包下载地址，藏得好深。。。

2. 将war包移动到tomcat的webapps目录下
3. 解压war包，将解压后的文件夹重命名为cas
4. 启动tomcat
5. 访问`https://localhost:8443/cas`

6. war包下的`WEB-INF/classes/application.properties`查看账号密码

   ```properties
   ##
   # CAS Authentication Credentials
   #
   cas.authn.accept.users=casuser::Mellon
   ```

7. 修改`WEB-INF/classes/log4j2.xml`的日志路径

   ```xml
   <?xml version="1.0" encoding="UTF-8" ?>
   <!-- Specify the refresh internal in seconds. -->
   <Configuration monitorInterval="5" packages="org.apereo.cas.logging">
       <Properties>
           <Property name="baseDir">自定义日志路径（文件夹）</Property>
       </Properties>
     ...
   ```

8. 配置数据库、MD5加密等

   > 我主要就为了体验部署一下，所以这些就不弄了。



