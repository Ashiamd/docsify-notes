# SpringBoot视频教程-蚂蚁课堂

> 惯例视频链接： http://www.mayikt.com/course/video/3708 

## 第一章 SpringBoot课程体系介绍

### 第四节 SpringBoot与SpringMVC关系

spring-boot-starter-web整合SpringMVC、Spring

### 第五节 SpringCloud与SpringBoot区别

1. SpringCloud是一套比较流行的微服务解决方案框架(非RPC远程调用框架)，

   常用组件:

   Euerka 服务注册

   Feign 客户端实现rpc远程调用

   Zuul网关

   Ribbon 本地负载均衡器

   SpringCloud Config

   Hystrix 服务保护框架

2. SpringCloud依赖SpringBoot组件，使用SpringMVC编写Http协议接口，同时SpringCloud是一套完整的微服务解决框架

   > RPC介绍文章
   >
   > [浅谈RPC调用](https://www.cnblogs.com/linlinismine/p/9205676.html)
   >
   > [如何给老婆解释什么是RPC]( https://www.jianshu.com/p/2accc2840a1b)

## 第二章 SpringBoot启动方式

### 第七节 使用@EnableAutoConfiguration启动

1. @RestController注解(Spring的)

   包装了@Controller和@ResponseBody，即返回json格式的方法

2. @EnableAutoConfiguration注解

    `@EnableAutoConfiguration`可以帮助SpringBoot应用将所有符合条件的`@Configuration`配置都加载到当前SpringBoot创建并使用的IoC容器。 

   > [SpringBoot之@EnableAutoConfiguration注解]( https://blog.csdn.net/zxc123e/article/details/80222967 )
   >
   > 顺便复习一下IOD、DI相关知识
   >
   > [Spring bean中的properties元素内的name 和 ref都代表什么意思啊？](https://www.cnblogs.com/lemon-flm/p/7217353.html)
   >
   > [面试被问烂的 Spring IOC(求求你别再问了]( https://www.jianshu.com/p/17b66e6390fd )

### 第八节 使用@ComponentScan指定扫描包启动

1. @EnableAutoConfiguration只扫描当前类范围的包

   可以用@ComponentScan指定扫描指定包范围(Spring的)

### 第九节 @SpringBootApplication启动方式

1. @SpringBootApplication实际就是(主要)@Configuraion+@EnableAutoConfiguration+@ComponentScan

   其中@Configuration作用可以等同于编写类.xml配置文件，即通过java类编写对应的.xml配置文件；
   
   @EnableAutoConfiguration就是前面提到的加载配置文件的（且扫描当前类及以下的包）
   
   @ComponentScan指定扫描包（默认该类**所在的包**下所有配置类）

## 第三章 SpringBoot整合Web视图层

### 第十节 SpringBoot静态资源访问控制

1. 静态资源访问，放在resources目录的static子目录下自动识别，可以直接url访问，如果static里面再创建子目录，需要在url中加上目录名，比如:

   resources/static/a.js -> localhost:8080/a.js

   resources/static/imgs/a.png -> localhost:8080/imgs/a.png

### 第十一节 SpringBoot整合Freemarker

1. freemarker网页静态化的一种手段，SEO优化的手段之一

### 第十二节 SpringBoot整合JSP注意事项

1. 创建SpringBoot整合JSP，一定要为war类型，否则会找不到页面

2. 创建maven工程project配置离线加载，不然过慢：

   创建时添加参数：archetypeCatalog=internal

### 第十三节 SpringBoot整合JSP项目实现

1. yml配置文件需要放在resources目录下，否则不生效

## 第四章 SpringBoot整合多数据源

### 第十四节 SpringBoot整合JdbcTemplate

1. pom引入相应依赖jdbc和mysql驱动包

2. yml和properties的选择，SpringBoot通常使用yml

   properties优先级比yml高；

   yml结构清晰、简化配置

3. SpringBoot2如果yml中配置多数据源，多个datasource下的url必须改为jdbc-url

4. @Autowired 自动依赖注入，复习一下；@Component注册到spring容器bean

### 第十五节 SpringBoot整合Mybatis框架

1. @SpringBootApplication 扫包不包含Mybatis，需要自己用@MapperScan

### 第十六节 SpringBoot多数据源使用分包拆数据源思路

1. 多数据源如何定位自己的数据源：
   1. 分包名；
   2. 注解形式；@dataSource（不推荐使用）

### 第十七节 SpringBoot整合多数据源代码实现

1. 创建一个Config.java类，使用@Configuration和@MapperScan分别把该类实现为xml配置和扫描指定位置的包
2. 多数据源配置有BUG，yml配置文件的数据源原本**“url”需改成“jdbc-url”**（亲测，确实如此）
3. 可能会在Config类中用到@Primary或@Qualifier，不然可能会报错（至少我是报错了）

