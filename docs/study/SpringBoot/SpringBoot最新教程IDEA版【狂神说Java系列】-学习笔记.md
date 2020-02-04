# SpringBoot最新教程IDEA版【狂神说Java系列】-学习笔记

> 视频链接 https://www.bilibili.com/video/av75233634

## 1. 这阶段该如何学习

### 微服务阶段

javaSE：OOP

mysql：持久化

html+css+js+jquery+框架：视图，框架不熟悉，css不好；

javaweb：独立开发MVC三层架构的网站，原始

ssm：框架，简化了我们的开发流程，配置也开始较为复杂

war：tomcat运行

spring再简化：SpringBoot - jar：内嵌tomcat、微服务框架

服务越来越多：springcloud

### SpringBoot学习

是什么

配置如何编写 yaml

自动装配原理：重要，谈资

集成web开发：业务的核心

集成数据库 Druid

分布式开发：Dubbo+ZooKeepr

Swagger：接口文档

任务调度

SpringSecurity：Shiro

### SpringCloud学习

微服务

springcloud入门

Restful

Eureka

Ribbon

Feign

Hystrix

Zuul路由网关

SpringCloud config

## 2. 什么是SpringBoot

> [干货满满！10分钟看懂Docker和K8S](https://my.oschina.net/jamesview/blog/2994112)
>
> [MVC、MVP及MVVM之间的关系](https://www.cnblogs.com/shenyf/p/9532342.html)
>
> [SpringBoot：快速入门](https://blog.kuangstudy.com/index.php/archives/630/)
>
> [SpringBoot：初识SpringBoot](https://www.cnblogs.com/hellokuangshen/p/11255695.html)

## 3. 什么是微服务架构

MVC三层框架 MVVM 微服务架构

业务：service：userService: ===> 模块

springmvc ， controller ==> 提供接口



http：RPC

软实力：聊天+举行+谈吐+见解

## 4. 第一个springboot程序

+ web依赖：tomcat、dispatcherServlet、xml...

```xml
<dependencies>
    <!--		web依赖：tomcat、dispatcherServlet、xml-->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>

    <!--		spring-boot-starter-，所有的springboot依赖都是这个开头的-->
    
    <!--		单元测试-->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
        <exclusions>
            <exclusion>
                <groupId>org.junit.vintage</groupId>
                <artifactId>junit-vintage-engine</artifactId>
            </exclusion>
        </exclusions>
    </dependency>
</dependencies>

<build>
    <!--		打jar包插件-->
    <plugins>
        <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
        </plugin>
    </plugins>
</build>
```

## 5. IDEA快速创建即彩蛋

```xml
<!--        启动器-->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter</artifactId>
</dependency>
```

.properties修改

```properties
server.port: 8081 #修改启动的端口号为8081
```

banner.txt

```none
修改启动的提示图标，默认SpringBoot 的字符图片
```

## 6. Springboot自动装配原理

> [前端页面插件](https://www.bootschool.net/)

### 自动配置

pom.xml

+ spring-boot-dependencies：此核心依赖在父依赖spring-boot-starter-parent中
+ 引入一些SpringBoot依赖的时候，之所以不用指定版本，因为版本仓库里面已经有了

启动器

+ ```xml
  <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter</artifactId>
  </dependency>
  ```

+ 说白了就是SpringBoot的启动场景

+ 比如spring-boot-starter-web，它会帮我们自动导入web环境所有的依赖

+ springboot会将所有的功能场景，都变成一个个的启动器

### 主程序

```java
//标注这个类是一个springboot的应用；启动类的下的所有资源被导入
@SpringBootApplication
public class SpringbootTestApplication {
   public static void main(String[] args) {
      SpringApplication.run(SpringbootTestApplication.class, args);
   }
}
```

### @SpringBootApplication包装的注解

```java
@SpringBootConfiguration//springboot的配置
	@Configuration//spring配置类
		@Component//说明这也是一个spring的组件
@EnableAutoConfiguration//自动配置
	@AutoConfigurationPackage//自动配置包
		@Import({Registrar.class})//自动配置，`包注册`
		public @interface AutoConfigurationPackage {
		}
	@Import({AutoConfigurationImportSelector.class})//自动配置导入选择
		List<String> configurations = this.getCandidateConfigurations(annotationMetadata, attributes);// 获取所有的配置
@ComponentScan //扫描包
```

#### 获取候选的配置

```java
//public class AutoConfigurationImportSelector
protected List<String> getCandidateConfigurations(AnnotationMetadata metadata, AnnotationAttributes attributes) {
        List<String> configurations = SpringFactoriesLoader.loadFactoryNames(this.getSpringFactoriesLoaderFactoryClass(), this.getBeanClassLoader());
        Assert.notEmpty(configurations, "No auto configuration classes found in META-INF/spring.factories. If you are using a custom packaging, make sure that file is correct.");
        return configurations;
    }
```

```java
Properties properties = PropertiesLoaderUtils.loadProperties(resource);
//所有资源加载到配子类中
```

### 结论

springboot所有自动配置都是在启动的时候扫描并加载：`spring.factories`所有的自动配置类都在这里面，但是不一定生效，要判断条件是否成立，只要导入了对应的start，就有对应的启动器了，有了启动器，我们自动装配就会生效，然后就配置成功

1. springboot在启动的时候，从类路径下/META-INF/`spring.factories`获取指定的值
2. 将这些自动配置的类导入容器，自动配置就会生效，帮我们自动配置
3. 以前我们需要配置的东西，springboot帮我们做了
4. 整合javaEE，解决方案和自动配置的东西都在spring-boot-autoconfigure-xx.xx.xx.RELEASE.jar包下
5. 它会把所有需要导入的组件，以类名的方式返回，这些组件就会被添加到容器
6. 容器中也会存在非常多的xxxAutoConfiguration的文件(@Bean)，就是这些类给容器中导入了这个场景需要的所有组件；并自动配置，@Configuration，JavaConfig
7. 有了自动配置类，省去了我们手动编写配置文件的工作

## 7. 了解下主启动类如何启动

​	JavaConfig	@Configuration	@Bean

​	Docker： 进程

关于SpringBoot，谈谈你的理解:

+ 自动装配
+ run()

全面接管SpringMVC配置

## 8. yaml语法讲解

> [SpringBoot 全局配置文件(Properties与YAML)详解和@ConfigurationProperties与@Vuale使用](https://blog.csdn.net/qq_42402854/article/details/89884283)

### yaml

+ 键值对 key: value（value前面必须有空格）
+ 对象
+ 数组 可用
+ map 可用{}

### properties

+ 键值对 key=value

## 9. 给属性赋值的几种方式

> [@Component 和 @Bean 的区别](https://blog.csdn.net/ztx114/article/details/82665544)
>
> [@bean和@component的理解](https://blog.csdn.net/daobuxinzi/article/details/100546815)
>
> [SpringBoot中yaml配置对象](https://www.cnblogs.com/zhuxiaojie/p/6062014.html)
>
> [SpringBoot：配置文件及自动配置原理](https://www.cnblogs.com/hellokuangshen/p/11259029.html)
>
> [SpringBoot-configuration-docs](https://docs.spring.io/spring-boot/docs/2.1.6.RELEASE/reference/html/configuration-metadata.html#configuration-metadata-annotation-processor)

## 10. JSR303校验

> [JSR-303](https://www.jianshu.com/p/554533f88370)

```java
import javax.validation.constraints.Email;
//点击constraints查看其他validation源码
```

## 11. 多环境配置及配置文件位置

> [Spring Boot Reference Guide](https://docs.spring.io/spring-boot/docs/2.1.6.RELEASE/reference/htmlsingle/#boot-features-external-config)
>
> [SpringBoot + Maven实现多环境动态切换yml配置及配置文件拆分](https://blog.csdn.net/Colton_Null/article/details/82145467)

### 配置文件生肖路径

下面文字摘自官方文档[Spring Boot Reference Guide](https://docs.spring.io/spring-boot/docs/2.1.6.RELEASE/reference/htmlsingle/#boot-features-external-config)

Config locations are searched in reverse order. By default, the configured locations are `classpath:/,classpath:/config/,file:./,file:./config/`. The resulting search order is the following:

1. `file:./config/`
2. `file:./`
3. `classpath:/config/`
4. `classpath:/`

### yml多环境配置

+ 单文件内可以用\-\-\-划分多个环境的配置，即单文件实现test、dev、release等多环境配置

+ 使用spring.profiles.active=dev表示启用dev环境配置（这里指的是properties写法，yml就格式不一样而已）

## 12. 自动配置原理再理解

查看源码大致顺序

+ @SpringBootApplication
+ @EnableAutoConfiguration
+ @Import({AutoConfigurationImportSelector.class})
+ List\<String\> getCandidateConfigurations
+ List\<String\> configurations = SpringFactoriesLoader.loadFactoryNames(this.getSpringFactoriesLoaderFactoryClass(), this.getBeanClassLoader());
+ List\<String\> loadFactoryNames
+ spring-boot-autoconfigure-xx.xx.xx.RELEASE.jar/META-INF/spring.factories

按照上述顺序，最后从.factories文件读取配置，但是配置不一定生效，需要被导入实际的包依赖时，检测到对应的配置存在才会生效。

源码中常见的`@ConditionalOnXXX`根据不同条件判断当前配置或者类是否生效。

**规律：在我们这些配置文件中能配置的内容，都存在对应的xxxProperties类，类使用@ConfigurationProperties注解绑定对应的.properties文件的配置。springboot自动装配，然后从xxxAutoConfiguration获取默认值，而其从xxxProperties类取默认值**

![img](https://img2018.cnblogs.com/blog/1418974/201907/1418974-20190729005838066-64765389.png)

**那么多的自动配置类，必须在一定的条件下才能生效；也就是说，我们加载了这么多的配置类，但不是所有的都生效了。**

我们怎么知道哪些自动配置类生效；**我们可以通过启用 debug=true属性；来让控制台打印自动配置报告，这样我们就可以很方便的知道哪些自动配置类生效；**

```properties
#开启springboot的调试类
debug=true
```

## 13. web开发探究

> [Spring Boot实战：模板引擎](https://www.cnblogs.com/paddix/p/8905531.html)
>
> [springboot中Thymeleaf和Freemarker模板引擎的区别](https://blog.csdn.net/weixin_43943548/article/details/102978919)

### 自动装配

springboot到底帮我们配置了什么？我们能不能进行修改？能修改哪些东西？能不能扩展？

+ xxxAutoConfiguration：向容器中自动配置组件
+ xxxProperties：自动配置类，装配配置文件中自定义的一些内容

### 要解决的问题

+ 导入静态资源
+ 首页
+ jsp，模板引擎Thymeleaf
+ 装配扩展SpringMVC
+ 增删改查
+ 拦截器
+ 国际化

## 14. 静态资源导入探究

```java
//WebMvcAutoConfiguration.WebMvcAutoConfigurationAdapter
public void addResourceHandlers(ResourceHandlerRegistry registry) {
    if (!this.resourceProperties.isAddMappings()) {
        logger.debug("Default resource handling disabled");
    } else {
        Duration cachePeriod = this.resourceProperties.getCache().getPeriod();
        CacheControl cacheControl = this.resourceProperties.getCache().getCachecontrol().toHttpCacheControl();
        if (!registry.hasMappingForPattern("/webjars/**")) {
            this.customizeResourceHandlerRegistration(registry.addResourceHandler(new String[]{"/webjars/**"}).addResourceLocations(new String[]{"classpath:/META-INF/resources/webjars/"}).setCachePeriod(this.getSeconds(cachePeriod)).setCacheControl(cacheControl));
        }

        String staticPathPattern = this.mvcProperties.getStaticPathPattern();
        if (!registry.hasMappingForPattern(staticPathPattern)) {
            this.customizeResourceHandlerRegistration(registry.addResourceHandler(new String[]{staticPathPattern}).addResourceLocations(WebMvcAutoConfiguration.getResourceLocations(this.resourceProperties.getStaticLocations())).setCachePeriod(this.getSeconds(cachePeriod)).setCacheControl(cacheControl));
        }

    }
}
```

其中`WebMvcAutoConfiguration.getResourceLocations(this.resourceProperties.getStaticLocations()))`的`getStaticLocations`在ResourceProperties类中。

```java
//ResourceProperties类
private static final String[] CLASSPATH_RESOURCE_LOCATIONS = new String[]{"classpath:/META-INF/resources/", "classpath:/resources/", "classpath:/static/", "classpath:/public/"};
    
```

上述目录下的文件都可以被当作静态文件访问*classpath就是./src/main/resources目录*

其中springboot的./src/main/resources目录下优先级如下：

resources > static > public 

+ 一般public放公共资源（比如大家都用的js文件）
+ static放静态资源（图片等）
+ resources下放一些upload上来的文件

### 总结

1. 在Springboot，我们可以使用以下方式处理静态资源
   + webjars	`localhost:8080/webjars/`
   + public，static，/**，resources  `localhost:8080/`

2. 优先级：resources > static（springboot项目默认创建） > public 

如果在.properties或者.yml、.yaml配置了spring.mvc.static-path-pattern=/static/**，会使得访问静态由原本的localhost:8080/静态文件--变成-->localhost:8080/static/静态文件

## 15. 首页和图标定制

### 首页

依旧需要查看`WebMvcAutoConfiguration`源码

在其嵌套类`EnableWebMvcConfiguration`中的`getWelcomePage`可以看到代码先判断是否配置欢迎页面，没有则返回默认的欢迎页面index.html

```java
//WebMvcAutoConfiguration.EnableWebMvcConfiguration
@Bean
public WelcomePageHandlerMapping welcomePageHandlerMapping(ApplicationContext applicationContext, FormattingConversionService mvcConversionService, ResourceUrlProvider mvcResourceUrlProvider) {
    WelcomePageHandlerMapping welcomePageHandlerMapping = new WelcomePageHandlerMapping(new TemplateAvailabilityProviders(applicationContext), applicationContext, this.getWelcomePage(), this.mvcProperties.getStaticPathPattern());
    welcomePageHandlerMapping.setInterceptors(this.getInterceptors(mvcConversionService, mvcResourceUrlProvider));
    return welcomePageHandlerMapping;
}

private Optional<Resource> getWelcomePage() {
    String[] locations = WebMvcAutoConfiguration.getResourceLocations(this.resourceProperties.getStaticLocations());
    return Arrays.stream(locations).map(this::getIndexHtml).filter(this::isReadable).findFirst();
}

private Resource getIndexHtml(String location) {
    return this.resourceLoader.getResource(location + "index.html");
}
```

### templates目录

templates目录相当于原本web项目的WEB-INF目录（只能通过controlller访问）

其页面的显示需要模板引擎依赖，如thymeLeaf

 ### 图标

旧版Springboot会默认设置页面图标，可以通过`spring.mvc.favicon.enabled=false`来取消默认图标

现在修改图标，也依旧可以直接在静态资源路径存放`favicon.ico`作为图标

## 16. thymeleaf模板引擎

> [SpringBoot：Web开发](https://www.cnblogs.com/hellokuangshen/p/11310178.html)
>
> [thymeleaf官方文档](https://www.thymeleaf.org/doc/tutorials/3.0/usingthymeleaf.html)

查看`ThymeleafProperties`源代码

```java
public class ThymeleafProperties {
    private static final Charset DEFAULT_ENCODING;
    public static final String DEFAULT_PREFIX = "classpath:/templates/";
    public static final String DEFAULT_SUFFIX = ".html";
    private boolean checkTemplate = true;
    private boolean checkTemplateLocation = true;
    private String prefix = "classpath:/templates/";
    private String suffix = ".html";

	//...
}
```

下面是ThymeLeaf官方文档的部分代码

```java
Simple expressions:
Variable Expressions: ${...}
Selection Variable Expressions: *{...}
Message Expressions: #{...}
Link URL Expressions: @{...}
Fragment Expressions: ~{...}
Literals
Text literals: 'one text', 'Another one!',…
Number literals: 0, 34, 3.0, 12.3,…
Boolean literals: true, false
Null literal: null
Literal tokens: one, sometext, main,…
Text operations:
String concatenation: +
Literal substitutions: |The name is ${name}|
Arithmetic operations:
Binary operators: +, -, *, /, %
Minus sign (unary operator): -
Boolean operations:
Binary operators: and, or
Boolean negation (unary operator): !, not
Comparisons and equality:
Comparators: >, <, >=, <= (gt, lt, ge, le)
Equality operators: ==, != (eq, ne)
Conditional operators:
If-then: (if) ? (then)
If-then-else: (if) ? (then) : (else)
Default: (value) ?: (defaultvalue)
Special tokens:
No-Operation: _
All these features can be combined and nested:
```

## 17. ThymeLeaf语法

所有html元素都可以被thymeleaf替换接管：	th：元素名

+ th:each 遍历

+ ```html
  <!--    <h3 th:each="user:${users}">[[${user}]]</h3> 效果相同，但是建议使用下面那种-->
  <h3 th:each="user:${users}" th:text="${user}"></h3>
  ```

+ 题外话，前端建议使用三元表达式而不是if-else

```java
//在templates目录下的所有，只能通过controller来跳转
//这个需要模板引擎的支持！thymeLeaf
@Controller
public class HelloController {

    @GetMapping("/index")
    public String getIndex(){
        return "index";
    }

    @GetMapping("/test")
    public String test(Model model){
        model.addAttribute("msg","<h1>hello,spring-boot</h1>");
        model.addAttribute("users", Arrays.asList("qinjiang","kuangshen"));
        return "test";
    }
}

// templates/test.html
<!doctype html>
<html lang="en" xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8">
    <meta name="viewport"
          content="width=device-width, user-scalable=no, initial-scale=1.0, maximum-scale=1.0, minimum-scale=1.0">
    <meta http-equiv="X-UA-Compatible" content="ie=edge">
    <title>Document</title>
</head>
<body>

<!--    所有的html元素都可以被thymeleaf替换接管： th: 元素名-->
<div th:text="${msg}"></div>
<div th:utext="${msg}"></div>
<hr>

<!--    <h3 th:each="user:${users}">[[${user}]]</h3> 效果相同，但是建议使用下面那种-->
<h3 th:each="user:${users}" th:text="${user}"></h3>
</body>
</html>
```

## 18. MVC配置原理

查看`ContentNegotiatingViewResolver`源代码->`getCandidateViews`

```java
//  如果，你想diy一些定制化的功能，只要写这个组件，然后把它交给springboot，springboot就会帮我们自动装配
//  扩展 springmvc    dispatchservlet
public class MyMvcConfig implements WebMvcConfigurer {

    // ViewResolver 实现了视图解析器接口的类，我们就可以把它看作视图解析器
    @Bean
    public ViewResolver myViewResolver(){
        return new MyViewResolver();
    }

    //  自定义了一个自己的视图解析器MyViewResolver
    public static class MyViewResolver implements ViewResolver {

        @Override
        public View resolveViewName(String s, Locale locale) throws Exception {
            return null;
        }
    }
}
```

+ 实现了视图解析器接口ViewResolver 的类，我们就可以把它看作视图解析器

## 19. 扩展SpringMVC

+ 自定义的配置日器格式化

查看源码`WebMvcAutoConfiguration.WebMvcAutoConfigurationAdapter`的注解`@Import({WebMvcAutoConfiguration.EnableWebMvcConfiguration.class})`而`EnableWebMvcConfiguration`继承`DelegatingWebMvcConfiguration`，`DelegatingWebMvcConfiguration`使用@Autowired自动注入了容器的所有`WebMvcConfigurer`，自动获取。

而`WebMvcAutoConfiguration`使用了注解`@ConditionalOnMissingBean({WebMvcConfigurationSupport.class})`,只有在这个Bean不存在的时候才会使用MVC自动配置，而注解`@EnableWebMvc`包含了`DelegatingWebMvcConfiguration`，其继承`WebMvcConfigurationSupport`。所以只要使用了`@EnableWebMvc`,MVC自动装配就失效了

```yaml
# 自定义的配置日期格式化:
# spring.mvc.date.format=
```

```java
// 如果我们要扩展springmvc，官方建议我们这样去做
@Configuration
@EnableWebMvc   // 这玩意就是导入了一个类 DelegatingWebMvcConfiguration：从容器中获取所有的webmvcconfig
public class MyMvcConfig implements WebMvcConfigurer {

    //视图跳转
    @Override
    public void addViewControllers(ViewControllerRegistry registry) {
        registry.addViewController("/kuang").setViewName("test");
    }

}
```

在SpringBoot中，有非常多的xxx Configuration帮助我们进行**扩展配置**，只要看见了这个东西，我们就要注意了

## 20. 员工管理系统：准备工作

> [@Data注解 与 lombok](https://www.jianshu.com/p/c1ee7e4247bf)

## 21. 员工管理系统：首页实现

> [ThymeLeaf官方文档](https://www.thymeleaf.org/doc/tutorials/2.1/usingthymeleaf.html)
>
> [Springboot 添加server.servlet.context-path相关使用总结](https://blog.csdn.net/qq_38322527/article/details/101691785)
>
> [spring boot配置文件中 spring.mvc.static-path-pattern 配置项](https://www.cnblogs.com/yql1986/p/9219137.html)

```properties
# 应用上下文路径，原本的thymeleaf不用修改，不过url访问时需要添加/test
server.servlet.context-path=/test
```

首页配置：注意点，所有页面的静态资源都需要使用thymeleaf接管；@{}

## 22. 员工管理系统：国际化

> [什么是JavaConfig](https://blog.csdn.net/albenxie/article/details/82633775)

```properties
# application.properties 我们的配置文件的真实位置(国际化，login.properties有对应的几套语言版本),对应的thymeleaf使用#{login.tip}
spring.messages.basename=i18n.login

#login.properties
login.btn=登录 
login.password=密码
login.remember=记住我
login.tip=请登录
login.username=用户名

#login_en_US.properties
login.btn=Sign in
login.password=Password
login.remember=Remember me
login.tip=Please sign in
login.username=Username

#login_zh_CN.properties
login.btn=登录
login.password=密码
login.remember=记住我
login.tip=请登录
login.username=用户名
```

`WebMvcAutoConfiguration`的`public LocaleResolver localeResolver()`如果自己配置了地区解析器，那就使用本地配置的，否则使用默认的。即可以像之前自定义视图解析器一样，自定义这个(在java目录下与启动类同级的config目录下)。可以借鉴`LocaleResolver`的实现类`AcceptHeaderLocaleResolver`

操作：

自定义国际化组件`MyLocaleResolver`实现`LocaleResolver`，然后在实现`WebMvcConfigurer`的`MyMvcConfig`使用`@Bean`注册该国际化组件

总结：

1. 首页配置：
   1. 注意点，所有页面的静态资源都需要使用thymeleaf接管；
   2. url：`@{}`(thymeLeaf语法要求)
2. 页面国际化：
   1. 我们需要配置i18n文件
   2. 我们如果需要在项目中进行按钮自动化切换，我们需要自定义一个组件`LocaleResolver`
   3. 记得将自己写的组件配置到spring容器`@Bean`
   4. `#{}`(thymeLeaf语法要求)

## 23. 员工管理系统：登录功能实现

```html
<!--			如果msg的值为空，则不显示消息-->
<p style="color: red" th:text="${msg}" th:if="${not #strings.isEmpty(msg)}"></p>
```

```java
@Controller
public class LoginController {

    @RequestMapping("/user/login")
    public String login(
            @RequestParam("username") String username,
            @RequestParam("password") String password,
            Model model) {

        // 具体的业务：
        if(!StringUtils.isEmpty(username) && "123456".equals(password)){
            return "redirect:/main.html";
        }else {
            // 告诉用户，你登录失败了
            model.addAttribute("msg","用户名或者密码错误");
            return "index";
        }
    }
}

// -------------------------

// 如果我们要扩展springmvc，官方建议我们这样去做(启动类同级的config目录下)
@Configuration
//@EnableWebMvc   // 这玩意就是导入了一个类 DelegatingWebMvcConfiguration：从容器中获取所有的webmvcconfig
public class MyMvcConfig implements WebMvcConfigurer {


    @Override
    public void addViewControllers(ViewControllerRegistry registry) {
        registry.addViewController("/").setViewName("index");
        registry.addViewController("/index.html").setViewName("index");
        registry.addViewController("/main.html").setViewName("dashboard");
    }

    // 自定义的国际化组件生效
    @Bean
    public LocaleResolver localeResolver(){
        return new MyLocaleResolver();
    }
}
```

## 24. 员工管理系统：登录拦截器

```java
@Controller
public class LoginController {

    @RequestMapping("/user/login")
    public String login(
        @RequestParam("username") String username,
        @RequestParam("password") String password,
        Model model,
        HttpSession session) {

        // 具体的业务：
        if(!StringUtils.isEmpty(username) && "123456".equals(password)){
            session.setAttribute("loginUser", username);
            return "redirect:/main.html";
        }else {
            // 告诉用户，你登录失败了
            model.addAttribute("msg","用户名或者密码错误");
            return "index";
        }
    }
}

// ---------------------
// 启动类同级的config目录下
public class LoginHandlerInterceptor implements HandlerInterceptor {
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {

        //登录成功之后，应该有用户的session
        Object loginUser = request.getSession().getAttribute("loginUser");

        if (loginUser == null) { // 没有登录
            request.setAttribute("msg", "没有权限，请先登录");
            request.getRequestDispatcher("/index.html").forward(request, response);
            return false;
        } else {
            return true;
        }
    }
}

// ---------------------
// 启动类同级的config目录下
public class MyMvcConfig implements WebMvcConfigurer {
    //...
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(new LoginHandlerInterceptor())
            .addPathPatterns("/**")
            .excludePathPatterns("/", "/user/login", "/index.html", "/css/**", "/img/**", "/js/**");
    }
}
```

## 25. 员工管理系统：展示员工列表

1. 首页配置：
   1. 注意点，所有页面的静态资源都需要使用thymeleaf接管；
   2. url：`@{}`(thymeLeaf语法要求)
2. 页面国际化：
   1. 我们需要配置i18n文件
   2. 我们如果需要在项目中进行按钮自动化切换，我们需要自定义一个组件`LocaleResolver`
   3. 记得将自己写的组件配置到spring容器`@Bean`
   4. `#{}`(thymeLeaf语法要求)
3. 登录+拦截器
4. 员工列表展示
   1. 提取公共页面
      1. `th:fragment="sidebar"`
      2. `th:replace="~{commons/commons::topbar}"`
      3. 如果要传递参数，可以直接使用()传参，接收判断即可
   2. 列表循环展示

## 26. 员工管理系统：增加员工实现

1. 首页配置：
   1. 注意点，所有页面的静态资源都需要使用thymeleaf接管；
   2. url：`@{}`(thymeLeaf语法要求)
2. 页面国际化：
   1. 我们需要配置i18n文件
   2. 我们如果需要在项目中进行按钮自动化切换，我们需要自定义一个组件`LocaleResolver`
   3. 记得将自己写的组件配置到spring容器`@Bean`
   4. `#{}`(thymeLeaf语法要求)
3. 登录+拦截器
4. 员工列表展示
   1. 提取公共页面
      1. `th:fragment="sidebar"`
      2. `th:replace="~{commons/commons::topbar}"`
      3. 如果要传递参数，可以直接使用()传参，接收判断即可
   2. 列表循环展示
5. 添加员工
   1. 按钮提交
   2. 跳转到添加页面
   3. 添加员工成功
   4. 返回首页

## 27. 员工管理系统：修改员工信息

## 28. 员工管理系统：删除及404处理

1. 首页配置：
   1. 注意点，所有页面的静态资源都需要使用thymeleaf接管；
   2. url：`@{}`(thymeLeaf语法要求)
2. 页面国际化：
   1. 我们需要配置i18n文件
   2. 我们如果需要在项目中进行按钮自动化切换，我们需要自定义一个组件`LocaleResolver`
   3. 记得将自己写的组件配置到spring容器`@Bean`
   4. `#{}`(thymeLeaf语法要求)
3. 登录+拦截器
4. 员工列表展示
   1. 提取公共页面
      1. `th:fragment="sidebar"`
      2. `th:replace="~{commons/commons::topbar}"`
      3. 如果要传递参数，可以直接使用()传参，接收判断即可
   2. 列表循环展示
5. 添加员工
   1. 按钮提交
   2. 跳转到添加页面
   3. 添加员工成功
   4. 返回首页
6. CRUD搞定
7. 404（在resources/templates/error目录下新建404.html）

## 29. 聊聊该如何写一个网站

前端：

+ 模板：修改别人的成品
+ 框架：组件，自己动手组合拼接。BootStrap（12）、LayUI（24？）、semantic-ui（16）...
  + 栅格系统

......

1. 前端搞定：页面张什么样子，数据
2. 设计数据库（数据库设计难点）
3. 前端让它能够自动运行，独立化工程
4. 数据接口如何对接：json，对象all in one
5. 前后端联调测试

+ 有一套自己熟悉的后台模板：工作必要，例如[x-admin](http://x.xuebingsi.com/)
+ 前端界面：至少自己能够通过前端框架，组合出来一个网站页面
  + index
  + about
  + blog
  + post
  + user
+ 让这个网站能够独立运行

## 30. 回顾及这周安排

+ SpringBoot是什么？
+ 微服务
+ HelloWorld~
+ 探究源码~自动装配原理~
+ 配置yaml
+ 多文档环境切换
+ 静态资源映射
+ ThymeLeaf th:xx
+ SpringBoot如何扩展 MVC  javaConfig
+ 如何修改SpringBoot的默认配置~
+ CRUD
+ 国际化
+ 拦截器
+ 定制首页，错误页~

安排：

+ JDBC
+ **Mybatis：重点**
+ **Druid：重点**
+ **Shiro：安全：重点**
+ **Spring Security：安全：重点**
+ 异步任务~，邮件发送，定时任务
+ Swagger
+ Dubbo+ZooKeeper

## 31. 整合JDBC使用

> [SpringBoot：Mybatis + Druid 数据访问](https://www.cnblogs.com/hellokuangshen/p/11331338.html)

### 简介

​	对于数据访问层，无论是 SQL(关系型数据库) 还是 NOSQL(非关系型数据库)，Spring Boot 底层都是采用 **Spring Data** 的方式进行统一处理。

​	Spring Boot 底层都是采用 Spring Data 的方式进行统一处理各种数据库，Spring Data 也是 Spring 中与 Spring Boot、Spring Cloud 等齐名的知名项目。

### 测试使用

由于SpringBoot的自动注入特性，导入MySQL和JDBC依赖后，自动注册Database组件，可以直接注入使用。

```java
@SpringBootTest
class Springboot04DataApplicationTests {

    @Autowired
    DataSource dataSource;

    @Test
    void contextLoads() {
        // 查看一下默认的数据源 class com.zaxxer.hikari.HikariDataSource
        System.out.println(dataSource.getClass());
    }

}
```

```java
// JDBC连接
@RestController
public class JDBCController {

    @Autowired
    JdbcTemplate jdbcTemplate;

    //查询数据库的所有信息
    //没有实体类，数据库中的东西，怎么获取？ Map
    @GetMapping("/userList")
    public List<Map<String,Object>> userList(){
        String sql = "select * from user";
        List<Map<String, Object>> list_maps = jdbcTemplate.queryForList(sql);
        return list_maps;
    }
}
```

## 32. 整合Druid数据源

```yaml
spring:
datasource:
username: root
password: 123456
#?serverTimezone=UTC解决时区的报错
url: jdbc:mysql://localhost:3306/mybatis?serverTimezone=UTC&useUnicode=true&characterEncoding=utf-8
driver-class-name: com.mysql.jdbc.Driver
type: com.alibaba.druid.pool.DruidDataSource

#Spring Boot 默认是不注入这些属性值的，需要自己绑定
#druid 数据源专有配置
initialSize: 5
minIdle: 5
maxActive: 20
maxWait: 60000
timeBetweenEvictionRunsMillis: 60000
minEvictableIdleTimeMillis: 300000
validationQuery: SELECT 1 FROM DUAL
testWhileIdle: true
testOnBorrow: false
testOnReturn: false
poolPreparedStatements: true

#配置监控统计拦截的filters，stat:监控统计、log4j：日志记录、wall：防御sql注入
#如果允许时报错  java.lang.ClassNotFoundException: org.apache.log4j.Priority
#则导入 log4j 依赖即可，Maven 地址： https://mvnrepository.com/artifact/log4j/log4j
filters: stat,wall,log4j
maxPoolPreparedStatementPerConnectionSize: 20
useGlobalDataSourceStat: true
connectionProperties: druid.stat.mergeSql=true;druid.stat.slowSqlMillis=500
```

```java
//启动类同级的config目录下
@Configuration
public class DruidConfig {

    @ConfigurationProperties(prefix = "spring.datasource")
    @Bean
    public DataSource druidDataSource(){
        return new DruidDataSource();
    }

    //后台监控:相当于  web.xml,ServletRegistrationBean
    //因为SpringBoot内置了servlet容器，所以没有web.xml，替代方法：ServletRegistrationBean
    @Bean
    public ServletRegistrationBean statViewServlet(){
        ServletRegistrationBean<StatViewServlet> bean = new ServletRegistrationBean<>(new StatViewServlet(),"/druid/*");

        //后台需要有人登录，账号密码配置
        HashMap<String, String> initParameters = new HashMap<>();

        //增加配置
        initParameters.put("loginUsername","admin");//登录key 是固定的 loginUsername loginPassword
        initParameters.put("loginPassword","123456");

        //允许谁可以访问
        initParameters.put("allow","");//设置为空则所有人都可以访问

        //禁止谁能访问 initParameters.put("testman","192.168.11.123");

        bean.setInitParameters(initParameters);//设置初始化参数
        return bean;
    }

    //filter
    public FilterRegistrationBean webStatFilter(){
        FilterRegistrationBean bean = new FilterRegistrationBean();
        bean.setFilter(new WebStatFilter());
        //可以过滤哪些请求呢？
        Map<String, String> initParameters = new HashMap<>();

        //这些东西不进行统计
        initParameters.put("exclusions","*.js,*.css,/druid/*");
        bean.setInitParameters(initParameters);
        return bean;
    }
}
```

SpringBoot内置了Servlet，依然可以通过Bean的方式去注册和使用相关的特性

## 33. 整合Mybatis框架

整合包

mybatis-spring-boot-starter



1. 导入包
2. 配置文件
3. mybatis配置
4. 编写sql
5. service层调用dao
6. controller调用service层

## 34. SpringSecurity环境搭建

> [安全框架 Shiro 和 Spring Security 如何选择？](https://blog.csdn.net/qq_42914528/article/details/101442524)
>
> [外观模式（Facade模式）详解](http://c.biancheng.net/view/1369.html)
>
> [Mybatis的一级缓存和二级缓存的理解和区别](https://www.jianshu.com/p/fdddea36eb22)
>
> [安全框架Shiro和SpringSecurity的比较](https://www.cnblogs.com/zoli/p/11236799.html)
>
> [安全框架Shiro和SpringSecurity的比较](https://www.cnblogs.com/zoli/p/11236799.html)

在web开发中，安全第一位！过滤器，拦截器~

功能性需求：否

做网站：安全应该在时候考虑？设计之初

+ 漏洞，隐私泄露~
+ 架构一旦确定~



shiro、SpringSecurity：很像~除了类不一样，名字不一样；

认证，授权（vip1，vip2，vip3）



+ 功能权限
+ 访问权限
+ 菜单权限
+ ... 拦截器，过滤器：大量的原生代码~ 冗余



MVC--Spring--SpringBoot--框架思想

## 35. 用户认证和授权

### 简介

Spring Security是针对sρring项目的安全框架,也是 Spring Boot底层安全模块默认的技术选型,他可以实现强大的web安全控制,对于安全控制,我们仅需要引入 spring-boot-starter-securit!y模块,进行少量的配置,即可实现强大的安全管理

记住几个类：

+ WebSecurityConfigurerAdapter：自定义 Security策略
+ AuthenticationManagerBuilder：自定义认证策略
+ @EnableWebSecurity：开启WebSecurity模式

Spring Security的两主要目标就是"认证"和"授权"(访问控制)

"认证"（Authentication)

"授权"(Authorization)

这个概念是通用的，而不是只在Spring Security中存在

```java
//启动类同级的config目录下
// AOP 拦截器
@EnableWebSecurity
public class SecurityConfig extends WebSecurityConfigurerAdapter {

    //链式编程
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        // 首页所有人可以访问，功能页只有对应有权限的人才能访问
        // 请求授权的规则
        http.authorizeRequests()
            .antMatchers("/").permitAll()
            .antMatchers("/level1/**").hasRole("vip1")
            .antMatchers("/level2/**").hasRole("vip2")
            .antMatchers("/level3/**").hasRole("vip3");

        // 没有权限默认会到登录页面,需要 开启登陆的页面
        // login
        http.formLogin();
    }

    //认证，Springboot 2.1.x可以直接使用
    //密码编码：PasswordEncoder
    //在Spring Security 5.0+ 新增了很多的加密方法
    @Override
    protected void configure(AuthenticationManagerBuilder auth) throws Exception {
        // 这些数据正常应该从数据库中读取
        auth.inMemoryAuthentication().passwordEncoder(new BCryptPasswordEncoder())
            .withUser("ash").password(new BCryptPasswordEncoder().encode("123456")).roles("vip2","vip3")
            .and()
            .withUser("root").password(new BCryptPasswordEncoder().encode("123456")).roles("vip1","vip2","vip3")
            .and()
            .withUser("guest").password(new BCryptPasswordEncoder().encode("123456")).roles("vip1");
    }
}
```

## 36. 注销及权限控制

```html
//thymeleaf标签可以使用sec:authorize="hasRole('vip1')"来限制只有拥有"vip1"角色权力的人才能访问、点击等
//<div sec:authorize="!isAuthenticated()"></div>
```

```java
// AOP 拦截器
@EnableWebSecurity
public class SecurityConfig extends WebSecurityConfigurerAdapter {

    //链式编程
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        // 首页所有人可以访问，功能页只有对应有权限的人才能访问
        // 请求授权的规则
        http.authorizeRequests()
                .antMatchers("/").permitAll()
                .antMatchers("/level1/**").hasRole("vip1")
                .antMatchers("/level2/**").hasRole("vip2")
                .antMatchers("/level3/**").hasRole("vip3");

        // 没有权限默认会到登录页面,需要 开启登陆的页面
        // login
        http.formLogin();

        //防止网站工具：get，post

        http.csrf().disable();//关闭 csrf功能，登录失败的可能原因
        //注销，开启了注销功能,跳到首页
        http.logout().logoutSuccessUrl("/");
    }

    //认证，Springboot 2.1.x可以直接使用
    //密码编码：PasswordEncoder
    //在Spring Security 5.0+ 新增了很多的加密方法
    @Override
    protected void configure(AuthenticationManagerBuilder auth) throws Exception {
        // 这些数据正常应该从数据库中读取
        auth.inMemoryAuthentication().passwordEncoder(new BCryptPasswordEncoder())
                .withUser("ash").password(new BCryptPasswordEncoder().encode("123456")).roles("vip2","vip3")
                .and()
                .withUser("root").password(new BCryptPasswordEncoder().encode("123456")).roles("vip1","vip2","vip3")
                .and()
                .withUser("guest").password(new BCryptPasswordEncoder().encode("123456")).roles("vip1");
    }
}
```

## 37. 记住我及首页定制

```java
// AOP 拦截器
@EnableWebSecurity
public class SecurityConfig extends WebSecurityConfigurerAdapter {

    //链式编程
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        // 首页所有人可以访问，功能页只有对应有权限的人才能访问
        // 请求授权的规则
        http.authorizeRequests()
                .antMatchers("/").permitAll()
                .antMatchers("/level1/**").hasRole("vip1")
                .antMatchers("/level2/**").hasRole("vip2")
                .antMatchers("/level3/**").hasRole("vip3");

        // 没有权限默认会到登录页面,需要 开启登陆的页面
        // login
        // 定制登陆页 loginPage("/toLogin")
        http.formLogin().loginPage("/toLogin").usernameParameter("user").passwordParameter("pwd").loginProcessingUrl("/login");

        //防止网站工具：get，post

        http.csrf().disable();//关闭 csrf功能，登录失败的可能原因
        //注销，开启了注销功能,跳到首页
        http.logout().logoutSuccessUrl("/");

        //开启记住我功能 cookie,默认保存两周，自定义接收前端的参数
        http.rememberMe().rememberMeParameter("remember");
    }

    //认证，Springboot 2.1.x可以直接使用
    //密码编码：PasswordEncoder
    //在Spring Security 5.0+ 新增了很多的加密方法
    @Override
    protected void configure(AuthenticationManagerBuilder auth) throws Exception {
        // 这些数据正常应该从数据库中读取
        auth.inMemoryAuthentication().passwordEncoder(new BCryptPasswordEncoder())
                .withUser("ash").password(new BCryptPasswordEncoder().encode("123456")).roles("vip2","vip3")
                .and()
                .withUser("root").password(new BCryptPasswordEncoder().encode("123456")).roles("vip1","vip2","vip3")
                .and()
                .withUser("guest").password(new BCryptPasswordEncoder().encode("123456")).roles("vip1");
    }
}
```

## 38. Shiro快速开始

### 什么是Shiro?

+ Apache Shiro是一个Java的安全（权限）框架
+ Shiro可以非常容易地开发出足够好的应用，其不仅可以用在JavaSE环境，也可以用在JavaEE环境
+ Shiro可以完成，认证、授权、加密、会话管理、Web集成、缓存等。

## 39. Shiro的Subject分析

1. 导入依赖
2. 配置文件
3. HelloWorld

Spring Security~都有

```java
Subject currentUser SecurityUtils.getSubject();
Session session = currentUser.getSession();
currentUser.isAuthenticated();
currentUser.getPrincipal();
currentUser.hasRole("schwartz");
currentUser.isPermitted("lightsaber:wield");
currentUser.logout();
```

## 40. SpringBoot整合Shiro环境搭建

## 41. Shiro实现登录拦截

```java
@Configuration
public class ShiroConfig {

    //ShiroFilterFactoryBean:3
    @Bean
    public ShiroFilterFactoryBean getShiroFilterFactoryBean(@Qualifier("securityManager")DefaultWebSecurityManager defaultWebSecurityManager){
        ShiroFilterFactoryBean bean = new ShiroFilterFactoryBean();
        //设置安全管理器
        bean.setSecurityManager(defaultWebSecurityManager);

        //添加shiro的内置过滤器
        /*
            anon：无需认证就可以访问
            authc：必须认证了才能访问
            user：必须拥有 记住我 功能才能用
            perms：拥有对某个资源的权限才能访问
            role：拥有某个角色权限才能访问
         */

        Map<String, String> filterMap = new LinkedHashMap<>();

        //拦截
        //        filterMap.put("/user/add","authc");
        //        filterMap.put("/user/update","authc");
        filterMap.put("/user/*","authc");

        bean.setFilterChainDefinitionMap(filterMap);

        //设置登录的请求
        bean.setLoginUrl("/toLogin");

        return bean;
    }

    //DefaultWebSecurityManager 2
    @Bean(name = "securityManager")
    public DefaultWebSecurityManager getDefaultWebSecurityManager(@Qualifier("userRealm") UserRealm userRealm){
        DefaultWebSecurityManager securityManager = new DefaultWebSecurityManager();
        //关联UserRealm
        securityManager.setRealm(userRealm);
        return securityManager;
    }

    // 创建realm对象，需要自定义类：1
    @Bean
    public UserRealm userRealm(){
        return new UserRealm();
    }
}
```

## 42. Shiro实现用户认证

```java
// 固定套路
@RequestMapping("/login")
public String login(String username, String password, Model model) {
    // 获取当前用户
    Subject subject = SecurityUtils.getSubject();
    // 封装用户的登录数据
    UsernamePasswordToken token = new UsernamePasswordToken(username, password);

    try{
        subject.login(token);//执行登录方法，如果没有异常就说明OK了
        return "index";
    }catch (UnknownAccountException e){//用户名不存在
        model.addAttribute("msg","用户名错误");
        return "login";
    }catch(IncorrectCredentialsException e){//密码不存在
        model.addAttribute("msg","密码错误");
        return "login";
    }

}
```

```java
// 启动类同级的config目录下
// 自定义的UsetrRealm extends AuthorizingRealm
public class UserRealm extends AuthorizingRealm {

    //授权
    @Override
    protected AuthorizationInfo doGetAuthorizationInfo(PrincipalCollection principalCollection) {
        System.out.println("执行了=>授权doGetAuthorizationInfo");
        return null;
    }

    //认证
    @Override
    protected AuthenticationInfo doGetAuthenticationInfo(AuthenticationToken token) throws AuthenticationException {
        System.out.println("执行了=>认证doGetAuthorizationInfo");

        //用户名，密码~   数据中取
        String name = "root";
        String password = "123456";

        UsernamePasswordToken userToken = (UsernamePasswordToken) token;

        if(!userToken.getUsername().equals(name)){
            return null; //会抛出 UnknownAccountException 异常
        }

        // 密码认证，shiro做~
        return new SimpleAuthenticationInfo("",password,"");
    }
}
```

## 43. Shiro整合Mybatis

```properties
mybatis.type-aliases-package=com.ash.pojo
mybatis.mapper-locations=classpath:mapper/*.xml
```

```xml
<!-- https://mvnrepository.com/artifact/org.mybatis.spring.boot/mybatis-spring-boot-starter -->
<dependency>
    <groupId>org.mybatis.spring.boot</groupId>
    <artifactId>mybatis-spring-boot-starter</artifactId>
    <version>2.1.1</version>
</dependency>

<!-- https://mvnrepository.com/artifact/com.alibaba/druid -->
<dependency>
    <groupId>com.alibaba</groupId>
    <artifactId>druid</artifactId>
    <version>1.1.21</version>
</dependency>

<!-- https://mvnrepository.com/artifact/mysql/mysql-connector-java -->
<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
    <version>8.0.11</version>
</dependency>

<!-- https://mvnrepository.com/artifact/log4j/log4j -->
<dependency>
    <groupId>log4j</groupId>
    <artifactId>log4j</artifactId>
    <version>1.2.17</version>
</dependency>

<!-- https://mvnrepository.com/artifact/org.apache.shiro/shiro-spring-boot-starter -->
<dependency>
    <groupId>org.apache.shiro</groupId>
    <artifactId>shiro-spring-boot-starter</artifactId>
    <version>1.5.0</version>
</dependency>
```

```java
// 自定义的UsetrRealm extends AuthorizingRealm
public class UserRealm extends AuthorizingRealm {

    @Autowired
    UserService userService;

    //授权
    @Override
    protected AuthorizationInfo doGetAuthorizationInfo(PrincipalCollection principalCollection) {
        System.out.println("执行了=>授权doGetAuthorizationInfo");
        return null;
    }

    //认证
    @Override
    protected AuthenticationInfo doGetAuthenticationInfo(AuthenticationToken token) throws AuthenticationException {
        System.out.println("执行了=>认证doGetAuthorizationInfo");

        //用户名，密码~   数据中取
//        String name = "root";
//        String password = "123456";
        // 连接真实数据库


        UsernamePasswordToken userToken = (UsernamePasswordToken) token;
        //连接真实的数据路
        User user = userService.queryUserByName(userToken.getUsername());

        if(user==null){//没有这个人
            return null;//UnknownAccountException
        }

//        if(!userToken.getUsername().equals(name)){
//            return null; //会抛出 UnknownAccountException 异常
//        }


        // 可以加密: MD5    MD5盐值加密
        // 密码认证，shiro做~ 密码加密了
        return new SimpleAuthenticationInfo("",user.getPwd(),"");
    }
}
```

`UserRealm`->`AuthorizingRealm`->`AuthenticatingRealm`->`getCredentialsMatcher`->`CredentialsMatcher`

## 44. Shiro授权实现

```java
@Configuration
public class ShiroConfig {

    //ShiroFilterFactoryBean:3
    @Bean
    public ShiroFilterFactoryBean getShiroFilterFactoryBean(@Qualifier("securityManager")DefaultWebSecurityManager defaultWebSecurityManager){
        ShiroFilterFactoryBean bean = new ShiroFilterFactoryBean();
        //设置安全管理器
        bean.setSecurityManager(defaultWebSecurityManager);

        //添加shiro的内置过滤器
        /*
            anon：无需认证就可以访问
            authc：必须认证了才能访问
            user：必须拥有 记住我 功能才能用
            perms：拥有对某个资源的权限才能访问
            role：拥有某个角色权限才能访问
         */

        //拦截
        Map<String, String> filterMap = new LinkedHashMap<>();
//        filterMap.put("/user/add","authc");
//        filterMap.put("/user/update","authc");

        //授权,正常情况下,没有授权会跳转到未授权页面
        filterMap.put("/user/add","perms[user:add]");
        filterMap.put("/user/update","perms[user:update]");

        filterMap.put("/user/*","authc");

        bean.setFilterChainDefinitionMap(filterMap);

        //设置登录的请求
        bean.setLoginUrl("/toLogin");
        //未授权页面
        bean.setUnauthorizedUrl("/noauth");

        return bean;
    }

    //DefaultWebSecurityManager 2
    @Bean(name = "securityManager")
    public DefaultWebSecurityManager getDefaultWebSecurityManager(@Qualifier("userRealm") UserRealm userRealm){
        DefaultWebSecurityManager securityManager = new DefaultWebSecurityManager();
        //关联UserRealm
        securityManager.setRealm(userRealm);
        return securityManager;
    }

    // 创建realm对象，需要自定义类：1
    @Bean
    public UserRealm userRealm(){
        return new UserRealm();
    }
}
```

```java
// 自定义的UsetrRealm extends AuthorizingRealm
public class UserRealm extends AuthorizingRealm {

    @Autowired
    UserService userService;

    //授权
    @Override
    protected AuthorizationInfo doGetAuthorizationInfo(PrincipalCollection principalCollection) {
        System.out.println("执行了=>授权doGetAuthorizationInfo");
        //SimpleAuthorizationInfo
        SimpleAuthorizationInfo info = new SimpleAuthorizationInfo();

//        info.addStringPermission("user:add");

        // 拿到当前登录的这个对象
        Subject subject = SecurityUtils.getSubject();
        User currentUser = (User) subject.getPrincipal();

        // 设置当前用户的权限
        info.addStringPermission(currentUser.getPerms());

//        return null;
        return info;
    }

    //认证
    @Override
    protected AuthenticationInfo doGetAuthenticationInfo(AuthenticationToken token) throws AuthenticationException {
        System.out.println("执行了=>认证doGetAuthorizationInfo");

        //用户名，密码~   数据中取
//        String name = "root";
//        String password = "123456";
        // 连接真实数据库


        UsernamePasswordToken userToken = (UsernamePasswordToken) token;
        //连接真实的数据路
        User user = userService.queryUserByName(userToken.getUsername());

        if(user==null){//没有这个人
            return null;//UnknownAccountException
        }

//        if(!userToken.getUsername().equals(name)){
//            return null; //会抛出 UnknownAccountException 异常
//        }


        // 可以加密: MD5    MD5盐值加密
        // 密码认证，shiro做~ 密码加密了
        return new SimpleAuthenticationInfo(user,user.getPwd(),"");
    }
}
```

## 45. Shiro整合Thymeleaf

```xml
<!-- https://mvnrepository.com/artifact/com.github.theborakompanioni/thymeleaf-extras-shiro -->
<dependency>
    <groupId>com.github.theborakompanioni</groupId>
    <artifactId>thymeleaf-extras-shiro</artifactId>
    <version>2.0.0</version>
</dependency>
```

```java
@Configuration
public class ShiroConfig {

    //ShiroFilterFactoryBean:3
    @Bean
    public ShiroFilterFactoryBean getShiroFilterFactoryBean(@Qualifier("securityManager")DefaultWebSecurityManager defaultWebSecurityManager){
        ShiroFilterFactoryBean bean = new ShiroFilterFactoryBean();
        //设置安全管理器
        bean.setSecurityManager(defaultWebSecurityManager);

        //添加shiro的内置过滤器
        /*
            anon：无需认证就可以访问
            authc：必须认证了才能访问
            user：必须拥有 记住我 功能才能用
            perms：拥有对某个资源的权限才能访问
            role：拥有某个角色权限才能访问
         */

        //拦截
        Map<String, String> filterMap = new LinkedHashMap<>();
//        filterMap.put("/user/add","authc");
//        filterMap.put("/user/update","authc");

        //授权,正常情况下,没有授权会跳转到未授权页面
        filterMap.put("/user/add","perms[user:add]");
        filterMap.put("/user/update","perms[user:update]");

        filterMap.put("/user/*","authc");

        bean.setFilterChainDefinitionMap(filterMap);

        //设置登录的请求
        bean.setLoginUrl("/toLogin");
        //未授权页面
        bean.setUnauthorizedUrl("/noauth");

        return bean;
    }

    //DefaultWebSecurityManager 2
    @Bean(name = "securityManager")
    public DefaultWebSecurityManager getDefaultWebSecurityManager(@Qualifier("userRealm") UserRealm userRealm){
        DefaultWebSecurityManager securityManager = new DefaultWebSecurityManager();
        //关联UserRealm
        securityManager.setRealm(userRealm);
        return securityManager;
    }

    // 创建realm对象，需要自定义类：1
    @Bean
    public UserRealm userRealm(){
        return new UserRealm();
    }

    //整合ShiroDialect:用來整合shiro thymeleaf
    @Bean
    public ShiroDialect getShiroDialect(){
        return new ShiroDialect();
    }
}


//--------------------
//--------------------
//--------------------
// 自定义的UsetrRealm extends AuthorizingRealm
public class UserRealm extends AuthorizingRealm {

    @Autowired
    UserService userService;

    //授权
    @Override
    protected AuthorizationInfo doGetAuthorizationInfo(PrincipalCollection principalCollection) {
        System.out.println("执行了=>授权doGetAuthorizationInfo");
        //SimpleAuthorizationInfo
        SimpleAuthorizationInfo info = new SimpleAuthorizationInfo();

//        info.addStringPermission("user:add");

        // 拿到当前登录的这个对象
        Subject subject = SecurityUtils.getSubject();
        User currentUser = (User) subject.getPrincipal();

        // 设置当前用户的权限
        info.addStringPermission(currentUser.getPerms());

//        return null;
        return info;
    }

    //认证
    @Override
    protected AuthenticationInfo doGetAuthenticationInfo(AuthenticationToken token) throws AuthenticationException {
        System.out.println("执行了=>认证doGetAuthorizationInfo");

        //用户名，密码~   数据中取
//        String name = "root";
//        String password = "123456";
        // 连接真实数据库


        UsernamePasswordToken userToken = (UsernamePasswordToken) token;
        //连接真实的数据路
        User user = userService.queryUserByName(userToken.getUsername());

        if(user==null){//没有这个人
            return null;//UnknownAccountException
        }

        Subject currentSubject = SecurityUtils.getSubject();
        Session session = currentSubject.getSession();
        session.setAttribute("loginUser", user);

//        if(!userToken.getUsername().equals(name)){
//            return null; //会抛出 UnknownAccountException 异常
//        }


        // 可以加密: MD5    MD5盐值加密
        // 密码认证，shiro做~ 密码加密了
        return new SimpleAuthenticationInfo(user,user.getPwd(),"");
    }
}
```

```html
<!doctype html>
<html lang="en" xmlns:th="http://www.thymeleaf.org"
      xmlns:shiro="http://www.thymeleaf.org/thymeleaf-extras-shiro">
<head>
    <meta charset="UTF-8">
    <meta name="viewport"
          content="width=device-width, user-scalable=no, initial-scale=1.0, maximum-scale=1.0, minimum-scale=1.0">
    <meta http-equiv="X-UA-Compatible" content="ie=edge">
    <title>首页</title>
</head>
<body>
<h1>首页</h1>

<!--从session中判断值-->
<div th:if="${session.loginUser==null}">
    <a th:href="@{/toLogin}">登录</a>
</div>

<p th:text="${msg}"></p>
<hr>

<div shiro:hasPermission="user:add">
    <a th:href="@{/user/add}">add</a>
</div>

<div shiro:hasPermission="user:update">
    <a th:href="@{/user/update}">update</a>
</div>

</body>
</html>	
```

## 46. 鸡汤分析开源项目

## 47. Swagger介绍及集成

### 学习目标

+ 了解Swagger的作用和概念
+ 了解前后端分离
+ 在SpringBoot中集成Swagger

### Swagger简介

#### 前后端分离

Vue + SpringBoot

后端时代：前端只用管理静态页面；html==>后端。模板引擎 JSP => 后端是主力

前后端分离式时代：

+ 后端：后端控制层，服务层，数据访问层【后端团队】
+ 前端：前端控制层，视图层【前端团队】
  + 伪造后端数据，json。已经存在了，不需要后端，前端工程依旧能够跑起来
+ 前后端如何交互？===>API
+ 前后端相对独立，松耦合
+ 前后但甚至可以部署在不同的服务器上

解决方案：

+ 首先指定schema[计划的提纲]，实时更新最新APl，降低集成的风险;
+ 早些年：指定word计划文档
+ 前后端分离：
  + 前端测试后端接口: postman
  + 后端提供接口，需要实时更新最新的消息及改动!

#### Swagger

+ 号称世界上最流行的Api框架
+ Restful Api文档在线自动生成工具=> ==Api文档与API定义同步更新==
+ 直接运行,可以在线测试AP接口;
+ 支持多种语言:(Java,Php…)

