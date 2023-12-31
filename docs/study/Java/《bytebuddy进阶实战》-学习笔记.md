# 《bytebuddy进阶实战》-学习笔记

> + 视频教程: [bytebuddy进阶实战-skywalking agent可插拔式架构实现_哔哩哔哩_bilibili](https://www.bilibili.com/video/BV1Jv4y1a7Kw)
>
> + ByteBuddy github项目: [raphw/byte-buddy: Runtime code generation for the Java virtual machine. (github.com)](https://github.com/raphw/byte-buddy)
>
> + ByteBuddy官方教程: [Byte Buddy - runtime code generation for the Java virtual machine](http://bytebuddy.net/#/tutorial-cn)
>
> + SkyWalking官网: [Apache SkyWalking](https://skywalking.apache.org/)
>
> + ByteBuddy个人学习中记录demo代码的github仓库: [Ashiamd/ash_bytebuddy_study](https://github.com/Ashiamd/ash_bytebuddy_study/tree/main)
>
> + SkyWalking Agent个人学习中记录demo代码的github仓库: [Ashiamd/ash_skywalking_agent_study](https://github.com/Ashiamd/ash_skywalking_agent_study)
>
> ---
>
> 下面的图片全部来自网络博客/文章
>
> 下面学习笔记主要由视频内容, 官方教程, 网络文章, 个人简述组成。

# 1. 网络文章收集

> [深入理解 Skywalking Agent - 简书 (jianshu.com)](https://www.jianshu.com/p/61d2edbe2275) <= 特别棒的文章，推荐阅读

# 2. SkyWalking简述

**SkyWalking**: an APM(application performance monitor) system, especially designed for microservices, cloud native and container-based architectures.

![img](https://upload-images.jianshu.io/upload_images/23610677-1bd4fe5f5ea9b253.png?imageMogr2/auto-orient/strip|imageView2/2/w/943/format/webp)

​	SkyWalking主要可以划分为4部分：

+ Probes: agent项目（sniffer）
+ Platform backend：oap
+ Storage：oap使用的存储
+ UI: 界面

​	大致流程即：

（1）agent(sniff)探针在服务端程序中执行SkyWalking的插件的拦截器逻辑，完成监测数据获取

（2）agent(sniff)探针通过gRPC(默认)等形式将获取的监测数据传输到OAP系统

（3）OAP系统将数据进行处理/整合后，存储到Storage中(ES, MySQL, H2等)

（4）用户可通过UI界面快捷查询数据，了解请求调用链路，调用耗时等信息

---

> [Overview | Apache SkyWalking](https://skywalking.apache.org/docs/main/v9.7.0/en/concepts-and-designs/overview/)

![img](https://skywalking.apache.org/images/home/architecture_2160x720.png?t=20220617)

- **Probe**s collect telemetry data, including metrics, traces, logs and events in various formats(SkyWalking, Zipkin, OpenTelemetry, Prometheus, Zabbix, etc.)
- **Platform backend** supports data aggregation, analysis and streaming process covers traces, metrics, logs and events. Work as Aggregator Role, Receiver Role or both.
- **Storage** houses SkyWalking data through an open/plugable interface. You can choose an existing implementation, such as ElasticSearch, H2, MySQL, TiDB, BanyanDB, or implement your own.
- **UI** is a highly customizable web based interface allowing SkyWalking end users to visualize and manage SkyWalking data.

# 3. SkyWalking Agent

> [深入理解 Skywalking Agent - 简书 (jianshu.com)](https://www.jianshu.com/p/61d2edbe2275) <= 推荐先阅读，很详细

+ 基础介绍

  SkyWalking Agent，是SkyWalking中的组件之一(`skywalking-agent.jar`)，在Java中通过Java Agent实现。

+ 使用简述

  （1）通过`java -javaagent:skywalking-agent.jar包的绝对路径 -jar 目标服务jar包绝对路径`指令，在目标服务main方法执行前，会先执行`skywalking-agent.jar`的`org.apache.skywalking.apm.agent.SkyWalkingAgent#premain`方法

  （2）premian方法执行时，加载与`skywalking-agent.jar`同层级文件目录的`./activations`和`./plusgins`目录内的所有插件jar包，根据jar文件内的`skywalking-plugin.def`配置文件作相关解析工作

  （3）解析工作完成后，将所有插件实现的拦截器逻辑，通过JVM工具类`Instrumentation`提供的redefine/retransform能力，修改目标类的`.class`内容，将拦截器内指定的增强逻辑附加到被拦截的类的原方法实现中

  （4）目标服务的main方法执行，此时被拦截的多个类内部方法逻辑已经被增强，比如某个方法执行前后额外通过gRPC将方法耗时记录并发送到OAP。简单理解的话，类似Spring中常用的AOP切面技术，这里相当于字节码层面完成切面增强

+ 实现思考

  Byte Buddy可以指定一个拦截器对指定拦截的类中指定的方法进行增强。SkyWalking Java Agent使用ByteBuddy实现，从0到1实现时，避不开以下几个问题需要考虑：

  （1）每个插件都需要有各自的拦截器逻辑，如何让Byte Buddy使用我们指定的多个拦截器

  （2）多个拦截器怎么区分各自需要拦截的类和方法

  （3）如何在premain中使用Byte Buddy时加载多个插件的拦截器逻辑

# 4. SkyWalking Agent Demo实现

## 4.1 学习目标

1. 下面跟着视频教程完成Java版本的SkyWalking Agent实现。因为视频主要是对Byte Buddy实际应用的扩展讲解，所以下面demo编写时，也主要围绕和Byte Buddy有关的部分，如：

+ 通过Byte Buddy完成不同插件不同类拦截范围的需求
+ 通过Byte Buddy完成不同插件不同方法拦截范围的需求
+ 通过Byte Buddy完成不同插件不同增强逻辑的需求

2. 学习SkyWalking Agent Demo实现中的一些代码设计技巧

   不同的工具，框架等，往往根据其使用场景，都有特别的设计或实现思想，值得学习和效仿

## 4.2 阶段一: Byte Buddy Agent回顾

### 4.2.1 目标

1. 完成项目初始化搭建
2. 回顾Byte Buddy Agent的使用

### 4.2.2 代码实现

1. 搭建简易的SpringBoot项目
2. 分别编写MySQL和SpringMVC的Byte Buddy Agent代码
3. 指定多个javagent启动参数，观察MySQL和SpringMVC的agent代码是否生效

### 4.2.3 小结

1. 使用多个`-javaagent:{agent的jar包绝对路径}`指定使用多个javaagent

### 4.2.4 思考

1. 如何记录多个拦截器各自拦截的范围，transform拦截的方法范围，以及使用的interceptor？

   可以提供抽象类/接口，并要求插件需要在实现中说明需要拦截的类的范围，拦截的方法范围，以及对应的拦截器逻辑

2. 如何仅指定一次`-javaagent`参数就加载多个agent插件？

   提供一个加载指定目录下所有插件的agent包

3. 加载插件的agent包，如何得知每个插件的入口类？

   可以约定要求每个插件在jar包内以配置文件记录自身的插件入口类

4. 插件和插件中拦截到的目标类使用什么类加载器？

   插件和插件拦截到的类都是用自定义类加载器，加载器的父类(双亲委托原则)则为ByteBuddy Agent加载目标类时使用的类加载器。不同类加载器之间互相隔离，这里插件和插件拦截的类使用相同的类加载器，保证插件能够访问到被拦截的类

## 4.3 阶段二: 可插拔插件加载思路分析

### 4.3.1 目标

像SkyWalking一样，只提供一个统一的agent.jar包作为`-javaagent:`的参数，且后续能够通过该jar包加载指定插件目录下的所有插件jar包。

### 4.3.2 代码实现

#### 4.3.2.1 目录结构

```pseudocode
apm-sniffer
├── apm-agent
│   ├── pom.xml
│   └── src/main/java/org.example.SkyWalkingAgent.java
├── apm-agent-core
│   ├── pom.xml
│   └── src
├── apm-plugins
│   ├── pom.xml
│   └── src
└── pom.xml
```

+ `apm-agent`：统一的java agent包
+ `apm-agent-core`：agent的核心逻辑，包括插件的顶级抽象类/接口，拦截器的顶级抽象类/接口等，插件实现需要依赖该模块
+ `apm-plugins`：插件实现的模块集合，所有插件实现在该模块下创建子模块，并且根据`apm-agent-core`内规定的抽象类/接口进行实现

#### 4.3.2.2 Agent实现和思考

+ 问题思考

1. 如何实现单一java agent入口类

   仅在`apm-agent`模块提供一个`premain`方法入口，其他插件未来都在`apm-plugins`内实现

2. 如何整合多个插件需要拦截的类范围

   `AgentBuilder#type`方法中，通过`.or(Xxx)`链接多个插件需要拦截的类范围

3. 如何整合多个插件需要增强的方法范围

   `DynamicType.Builder#method`方法中，通过`.or(Xxx)`链接多个插件需要增强的方法范围

4. 如何整个多个插件各自增强的方法对应的拦截器

   `DynamicType.Builder.MethodDefinition.ImplementationDefinition#intercept`可以通过`andThen`指定多个拦截器

5. 如何让被拦截的类中增强的方法走正确的拦截器逻辑？

   拦截器InterceptorA对应类拦截范围ClassA下的增强方法范围MethodA，所以需要有一种机制维护三者之间的关系

   需要记录的映射：

   + 拦截类范围 => 增强的方法范围
   + 拦截类范围 => 拦截器

   （如果没有维护这些映射关系，那么多个拦截器互相干扰，比如意外增强了其他类中的同名方法）

+ 代码实现

`org.example.SkyWalkingAgent.java`

```java
package org.example;

import net.bytebuddy.agent.builder.AgentBuilder;
import net.bytebuddy.description.type.TypeDescription;
import net.bytebuddy.dynamic.DynamicType;
import net.bytebuddy.implementation.MethodDelegation;
import net.bytebuddy.utility.JavaModule;

import java.lang.instrument.Instrumentation;
import java.security.ProtectionDomain;

import static net.bytebuddy.matcher.ElementMatchers.*;

/**
 * 模仿SkyWalking 的 统一 Java Agent 入口类
 *
 * @author : Ashiamd email: ashiamd@foxmail.com
 * @date : 2023/12/30 3:08 PM
 */
public class SkyWalkingAgent {
  public static void premain(String args, Instrumentation instrumentation) {
    AgentBuilder agentBuilder = new AgentBuilder.Default()
      .type(
      /*
                        // springmvc
                        isAnnotatedWith(named("org.springframework.stereotype.Controller")
                                .or(named("org.springframework.web.bind.annotation.RestController")))
                        // mysql
                        .or(named("com.mysql.cj.jdbc.ClientPreparedStatement")
                                .or(named("com.mysql.cj.jdbc.ServerPreparedStatement")))
                        // 其他插件 类拦截配置
                        */
    )
      .transform()
      .installOn(instrumentation);
  }

  private static class Transformer implements AgentBuilder.Transformer {

    @Override
    public DynamicType.Builder<?> transform(DynamicType.Builder<?> builder,
                                            TypeDescription typeDescription,
                                            // 加载 typeDescription 的类加载器
                                            ClassLoader classLoader,
                                            JavaModule javaModule,
                                            ProtectionDomain protectionDomain) {
      return builder.method(
        /*
                    // springmvc
                    not(isStatic())
                            .and(isAnnotatedWith(nameStartsWith("org.springframework.web.bind.annotation")
                                    .and((nameEndsWith("Mapping")))))
                    // mysql
                    .or(named("execute").or(named("executeUpdate").or(named("executeQuery"))))
                    */
      ).intercept(
        // springMvc 需要使用 SpringMVC插件的Interceptor
        // mysql 需要使用 mysql 插件的 Interceptor
        MethodDelegation.to());
    }
  }
}
```

## 4.4 阶段三: 插件抽象

### 4.4.1 目标

根据前面可插拔插件加载可知，我们需要有一种机制(抽象)能够将拦截的类与被增强的方法，以及增强方法的拦截器关联起来。

### 4.4.2 代码实现和思考

#### 4.4.2.1 插件规范

`AbstractClassEnhancePluginDefine`：所有的插件要求实现该类

1. 需要插件表明自己增强的类范围

   这里要求实现`protected abstract ClassMatch enhanceClass();`，`ClassMatch`是一个接口，表示类匹配器。

2. 需要插件表明增强类中哪些方法，以及对应的增强逻辑(拦截器逻辑)

   这里对应Byte Buddy支持的三种方法拦截，实例方法，构造方法，静态方法

   + `InstanceMethodsInterceptorPoint`：实例方法拦截点，内部声明方法匹配起和对应的拦截器类的全限制类名（注意，是类名，而不是`Class`对象）
   + `ConstructorMethodsInterceptorPoint`：构造方法拦截点
   + `StaticMethodsInterceptorPoint`：静态方法拦截点

```java
package org.example.core;

import net.bytebuddy.agent.builder.AgentBuilder;
import net.bytebuddy.description.type.TypeDescription;
import net.bytebuddy.matcher.ElementMatcher;
import org.example.core.interceptor.ConstructorMethodsInterceptorPoint;
import org.example.core.interceptor.InstanceMethodsInterceptorPoint;
import org.example.core.interceptor.StaticMethodsInterceptorPoint;
import org.example.core.match.ClassMatch;

/**
 * 所有插件的顶级父类
 *
 * @author : Ashiamd email: ashiamd@foxmail.com
 * @date : 2023/12/30 6:10 PM
 */
public abstract class AbstractClassEnhancePluginDefine {
  /**
     * 表示当前插件要拦截/增强的类范围 <br/>
     * 相当于对应 {@link AgentBuilder#type(ElementMatcher)} 的 参数 {@code ElementMatcher<? super TypeDescription>}
     */
  protected abstract ClassMatch enhanceClass();

  /**
     * 实例方法的拦截点 <br/>
     * ps: 类下面多个方法可能使用不同的拦截器
     */
  protected abstract InstanceMethodsInterceptorPoint[] getInstanceMethodsInterceptorPoints();

  /**
     * 构造方法的拦截点 <br/>
     */
  protected abstract ConstructorMethodsInterceptorPoint[] getConstructorMethodsInterceptorPoints();

  /**
     * 静态方法的拦截点 <br/>
     * ps: 类下面多个方法可能使用不同的拦截器
     */
  protected abstract StaticMethodsInterceptorPoint[] getStaticMethodsInterceptorPoints();
}
```

思考：

1. 这里` protected abstract ClassMatch enhanceClass();`是不是可以直接改成要求传递`ElementMatcher.Junction<? extends TypeDescription>`，即`AgentBuilder.Default#type(ElementMatcher)` 的参数？
   + 个人认为是可以的，但是这里用`ClassMath`接口，使得项目自身可以预先实现几个常用的类型匹配器，减少开发者的开发成本。比如可以封装常用的通用类名进行匹配的匹配器，或者通过注解匹配的类型匹配器。
2. 实例方法、构造方法、静态方法拦截点抽象方法，是否可以合成一个，然后通过新增一个type字段判断？
   + 个人认为是可以的，并且如果三个拦截点使用同一个类实现，通过type字段区分，也可以减少项目中的实现类。
   + 但是如果多个扩展点合并到一个类实现，未来如果三个扩展点有各自的不同点，则单类实现中可能就会存在方法A仅在实例方法扩展点有效，方法B仅在静态方法扩展点有效之类的情况。
   + 实际上在`apache-skywalking-java-agent`源码中，实例方法拦截点定义和静态方法拦截点一致，然后和构造方法拦截点不同，前者多个`boolean isOverrideArgs();`方法定义
   + 实际上方法类型在java中基本也可认为只区分实例方法、构造方法、静态方法，这里忽略静态代码块代码，所以拦截点方法就3个，分成3个具体类更易维护和理解。

#### 4.4.2.2 拦截点定义

+ `InstanceMethodsInterceptorPoint`：实例方法拦截点
+ `StaticMethodsInterceptorPoint`：静态方法拦截点
+ `ConstructorMethodsInterceptorPoint`：构造方法拦截点

这里三个方法拦截点，这里简单实现，代码基本一样，这里只看`ConstructorMethodsInterceptorPoint`

插件规范要求每个插件说明拦截的类范围，以及使用的多个拦截点，拦截点需要有拦截的方法范围和拦截器

1. 说明拦截的方法范围

   `ElementMatcher<MethodDescription> getConstructorMatcher()`要求声明拦截的方法范围

2. 说明使用的拦截器

   这里要求传递拦截器类名，而不是直接传递`Class`对象。`String getConstructorInterceptor();`

```java
package org.example.core.interceptor;

import net.bytebuddy.description.method.MethodDescription;
import net.bytebuddy.dynamic.DynamicType;
import net.bytebuddy.matcher.ElementMatcher;

/**
 * 构造方法拦截点
 *
 * @author : Ashiamd email: ashiamd@foxmail.com
 * @date : 2023/12/30 6:26 PM
 */
public interface ConstructorMethodsInterceptorPoint {
  /**
     * 表示要拦截的方法范围
     * {@link DynamicType.Builder#method(ElementMatcher)} 的匹配器参数
     */
  ElementMatcher<MethodDescription> getConstructorMatcher();

  /**
     * 获取被增强方法对应的拦截器
     */
  String getConstructorInterceptor();
}
```

思考：

1. 为什么要求传递拦截器类名，而不是拦截器类`Class`对象？

   如果只是为了后续找拦截器类，那么传递类名或者传递`Class`对象，个人认为都是可以的。

   个人猜测SkyWalking使用类名而不是要求传递`Class<?>`的原因在于，传递`Class<?>`可能存在类被提前加载的情况（虽然我暂时没找到什么情况会如此），而传递类名则稳定不会有该问题，保证后续使用统一的类加载加载指定的拦截器类。假设使用`Class<?>`并且被意外提前类加载了，则无法保证使用的类加载器一定是我们要的。

2. 为什么只要求传递一个拦截器类名，而不是传递多个？

   实际上`MethodDefinition.ImplementationDefinition#intercept`方法可以有如下的代码

   ```java
   DynamicType.Builder.MethodDefinition.ReceiverTypeDefinition<?> interceptor = builder.method(named("execute").or(named("executeUpdate").or(named("executeQuery"))))
                   .intercept(MethodDelegation.to(new Interceptor1())
                           .andThen(MethodDelegation.to(new Interceptor2())));
   ```

   即通过`.andThen`链接多个拦截器。

   + 实际插件开发，通常只需要一个拦截器即可，多个拦截器的内部逻辑本身也可以集成到同一个拦截器中。
   + 如果支持传递多个拦截器类名，那么可能还需要考虑每个拦截器类名对应哪些拦截方法范围，那么会导致整体设计更加复杂，并且使用多个拦截器本身并不是一个很有必要的需求。

#### 4.4.2.3 类匹配器定义

+ `ClassMatch`：所有类匹配器的顶级接口，本身空实现
+ `NameMatch`：单个全限制类名匹配的匹配器
+ `IndirectMatch`：所有非`NameMatch`的匹配器的顶级接口
  + `MultiClassNameMatch`：匹配多个类名的匹配器（多个全限制类名之间是or的关系）
  + `ClassAnnotationNameMatch`：匹配同时带有多个指定注解的类匹配器（多个注解之间是and的关系）

这里就`IndirectMatch`的实现类要复杂点，就拿`ClassAnnotationNameMatch`举例：

+ 间接类匹配器需要直接说明如何匹配

  这里`IndirectMatch`要求实现`public ElementMatcher.Junction<? extends TypeDescription> buildJunction();`方法，即说明匹配哪些方法范围

```java
package org.example.core.match;

import net.bytebuddy.description.type.TypeDescription;
import net.bytebuddy.matcher.ElementMatcher;

import java.util.Arrays;
import java.util.List;

import static net.bytebuddy.matcher.ElementMatchers.isAnnotatedWith;
import static net.bytebuddy.matcher.ElementMatchers.named;

/**
 * 同时带有指定的多个注解的匹配器
 * @author : Ashiamd email: ashiamd@foxmail.com
 * @date : 2023/12/30 10:58 PM
 */
public class ClassAnnotationNameMatch implements IndirectMatch{
  /**
     * 需要匹配的注解集合 (所有都满足的才会视为匹配成功)
     */
  private List<String> needMatchAnnotations;

  private ClassAnnotationNameMatch(String... annotations) {
    if (null == annotations || 0 == annotations.length) {
      throw new IllegalArgumentException("annotations can't be empty");
    }
    this.needMatchAnnotations = Arrays.asList(annotations);
  }

  /**
     * 多个注解之间，要求是and的关系
     */
  @Override
  public ElementMatcher.Junction<? extends TypeDescription> buildJunction() {
    ElementMatcher.Junction<? extends TypeDescription> junction = null;
    for (String annotationName : needMatchAnnotations) {
      if(null == junction) {
        junction = isAnnotatedWith(named(annotationName));
      } else {
        junction = junction.and(isAnnotatedWith(named(annotationName)));
      }
    }
    return junction;
  }

  public static IndirectMatch byClassAnnotationMatch(String... annotationNames) {
    return new ClassAnnotationNameMatch(annotationNames);
  }
}
```

思考：

1. 这里构造方法为私有的，并且以静态方法的形式对外提供实例，有啥好处？
   + 私有构造方法，则对外可以屏蔽匹配器内部逻辑细节
   + 只讨论这里类匹配器对外暴露的方式，使用共有构造方法对外暴露，或者使用静态方法，本身都是一样的效果。

#### 4.4.2.4 插件实现

所有的插件都需要继承前面"4.4.2.1 插件规范"定义的抽象类`AbstractClassEnhancePluginDefine`。

这里只挑Mysql插件举例：

+ 插件需要说明拦截的类范围

  这里通过`MultiClassNameMatch.byMultiClassMatch`声明需要拦截的多个全限制类名

+ 插件需要声明拦截的方法范围和对应的拦截器

  这里只拦截mysql的实例方法，所以构造方法拦截点和静态方法拦截点可以直接返回null或空数组。到这里为止，暂时没有实现具体的拦截器逻辑，只是声明了拦截器类的全限制类名

```java
package org.example.plugins.mysql;

import net.bytebuddy.description.method.MethodDescription;
import net.bytebuddy.matcher.ElementMatcher;
import org.example.core.AbstractClassEnhancePluginDefine;
import org.example.core.interceptor.ConstructorMethodsInterceptorPoint;
import org.example.core.interceptor.InstanceMethodsInterceptorPoint;
import org.example.core.interceptor.StaticMethodsInterceptorPoint;
import org.example.core.match.ClassMatch;
import org.example.core.match.MultiClassNameMatch;

import static net.bytebuddy.matcher.ElementMatchers.named;

/**
 * 定义MySQL插件
 *
 * @author : Ashiamd email: ashiamd@foxmail.com
 * @date : 2023/12/30 9:34 PM
 */
public class MysqlInstrumentation extends AbstractClassEnhancePluginDefine {
  @Override
  protected ClassMatch enhanceClass() {
    return MultiClassNameMatch.byMultiClassMatch(
      "com.mysql.cj.jdbc.ClientPreparedStatement",
      "com.mysql.cj.jdbc.ServerPreparedStatement");
  }

  @Override
  protected InstanceMethodsInterceptorPoint[] getInstanceMethodsInterceptorPoints() {
    // 之前单独实现的MySQL agent，我们只拦截实例方法，这里也只拦截实例方法
    return new InstanceMethodsInterceptorPoint[]{
      new InstanceMethodsInterceptorPoint() {
        @Override
        public ElementMatcher<MethodDescription> getMethodsMatcher() {
          // 拦截的方法范围
          return named("execute").or(named("executeUpdate").or(named("executeQuery")));
        }

        @Override
        public String getMethodsInterceptor() {
          // 这里拦截器使用类名，而不是要求传递Class类对象
          return "org.example.plugins.mysql.interceptor.MysqlInterceptor";
        }
      }
    };
  }

  @Override
  protected ConstructorMethodsInterceptorPoint[] getConstructorMethodsInterceptorPoints() {
    //        return new ConstructorMethodsInterceptorPoint[0];
    return null;
  }

  @Override
  protected StaticMethodsInterceptorPoint[] getStaticMethodsInterceptorPoints() {
    // 或者 return null, 反正 MySQL 插件，我们这里只拦截实例方法
    return new StaticMethodsInterceptorPoint[0];
  }
}
```

## 4.5 阶段四: 拦截器逻辑

### 4.5.1 目标

完善插件的拦截器逻辑，具体包括拦截的方法范围，拦截后的拦截器增强逻辑

### 4.5.2 代码实现和思考

#### 4.5.2.1 拦截器内部逻辑抽象

SkyWalking对拦截器内的具体逻辑进行抽象，只需要开发者根据拦截方法的类型实现拦截器接口方法。

+ `InstanceMethodAroundInterceptor`：实例方法拦截器接口，所有插件中的实例方法拦截器需要实现该接口
+ `ConstructorInterceptor`：构造方法拦截器接口，所有插件中的构造方法拦截器需要实现该接口
+ `StaticMethodAroundInterceptor`：静态方法拦截器接口，所有插件中的静态方法拦截器需要实现该接口

这里拿`InstanceMethodAroundInterceptor`代码举例：

+ `beforeMethod`：在被增强的实例方法之前执行

+ `afterMethod`：相当于被增强的实例方法的finally中执行
+ `handleEx`：被增强的方法出现异常时执行

```java
package org.example.core.interceptor.enhance;

import java.lang.reflect.Method;

/**
 * 实例方法(不包括 构造方法) 的拦截器 都需要实现当前接口
 * @author : Ashiamd email: ashiamd@foxmail.com
 * @date : 2023/12/31 6:37 PM
 */
public interface InstanceMethodAroundInterceptor {
  /**
     * 前置增强逻辑
     */
  void beforeMethod(EnhancedInstance obj, Method method, Object[] allArgs, Class<?>[] parameterTypes);

  /**
     * finally 后置增强逻辑 (不管原方法是否异常都会执行)
     * @return 方法返回值
     */
  Object afterMethod(EnhancedInstance obj, Method method, Object[] allArgs, Class<?>[] parameterTypes, Object returnValue);

  /**
     * 异常处理
     */
  void handleEx(EnhancedInstance obj, Method method, Object[] allArgs, Class<?>[] parameterTypes, Throwable t);
}
```

思考：

1. 拦截器内部逻辑分类？

   本身开发Byte Buddy拦截器时，对实例方法、构造方法、静态方法的拦截器处理逻辑不同，所以分成3类很合理

2. 拦截器内部逻辑抽象的方式？

   从接口定义不难看出来，很像常见的AOP切面开发。个人觉得这是一种经验总结后得到的开发模式。即大部分人在进行方法增强时，一般也就是在方法之前、方法执行后，以及出现异常时，执行我们自己的拦截器逻辑。

#### 4.5.2.2 拦截器模版

使用ByteBuddy指定拦截的方法范围后，需要指定使用的拦截器，而拦截器的实现基本都是套路化了，SkyWalking也是为构造方法、实例方法、静态方法提供了具体的拦截器模版实现。

+ `ConstructorInter`：构造方法拦截器模版
+ `InstanceMethodsInter`：实例方法拦截器模版
+ `StaticMethodsInter`：静态方法拦截器模版

这里以`StaticMethodsInter`举例：

+ `intercept`方法，ByteBuddy执行拦截的目标方法时，会调用该拦截逻辑。可以看出来里面用到了插件具体实现的拦截器逻辑，而整体的调用链路基本都是套路，所以直接模版化实现

```java
package org.example.core.interceptor.enhance;

import lombok.extern.slf4j.Slf4j;
import net.bytebuddy.implementation.bind.annotation.AllArguments;
import net.bytebuddy.implementation.bind.annotation.Origin;
import net.bytebuddy.implementation.bind.annotation.RuntimeType;
import net.bytebuddy.implementation.bind.annotation.SuperCall;

import java.lang.reflect.Method;
import java.util.concurrent.Callable;

/**
 * 通用的静态方法拦截器
 *
 * @author : Ashiamd email: ashiamd@foxmail.com
 * @date : 2023/12/31 6:27 PM
 */
@Slf4j
public class StaticMethodsInter {
  /**
     * 拦截器在各个位置执行某些逻辑，封装在此成员变量中
     */
  private StaticMethodAroundInterceptor interceptor;

  public StaticMethodsInter(String methodsInterceptorName, ClassLoader classLoader) {
  }


  /**
     * 具体的拦截器逻辑， 整体预留的填充逻辑和AOP切面编程类似
     */
  @RuntimeType
  public Object intercept(
    // 被拦截的目标类
    @Origin Class<?> clazz,
    // 被拦截的目标方法
    @Origin Method method,
    // 方被拦截的方法的法参数
    @AllArguments Object[] allArgs,
    // 调用原被拦截的目标方法
    @SuperCall Callable<?> zuper) throws Throwable {
    // 1. 前置增强
    try {
      interceptor.beforeMethod(clazz, method, allArgs, method.getParameterTypes());
    } catch (Throwable e) {
      log.error("Static Method Interceptor: {}, enhance method: {}, before method failed, e: ", interceptor.getClass().getName(), method.getName(), e);
    }
    Object returnValue = null;
    try {
      returnValue = zuper.call();
    } catch (Throwable e) {
      // 2. 异常处理
      try {
        interceptor.handleEx(clazz, method, allArgs, method.getParameterTypes(), e);
      } catch (Throwable innerError) {
        log.error("Static Method Interceptor: {}, enhance method: {}, handle Exception failed, e: ", interceptor.getClass().getName(), method.getName(), e);
      }
      // 继续上抛异常, 不影响原方法执行逻辑
      throw e;
    } finally {
      // 3. 后置增强
      try {
        returnValue = interceptor.afterMethod(clazz, method, allArgs, method.getParameterTypes(), returnValue);
      } catch (Throwable e) {
        log.error("Static Method Interceptor: {}, enhance method: {}, after method failed, e: ", interceptor.getClass().getName(), method.getName(), e);
      }
    }
    return returnValue;
  }
}
```

#### 4.5.2.3 插件规范

前面的 "4.4.2.1 插件规范"中，只要求插件声明自己需要拦截的类范围，以及拦截点（拦截的方法范围和拦截器类名），但是并没有定义如何执行增强逻辑。

这里对插件规范进一步完善，规范定义了插件进行类增强的执行流程。

+ `AbstractClassEnhancePluginDefine`：所有插件的顶级父类
+ `ClassEnhancePluginDefine`：所有插件的父类，指定具体的transform逻辑（拦截的方法范围，以及拦截器逻辑指定）

1. `AbstractClassEnhancePluginDefine`

+ `CONTEXT_ATTR_NAME`：增强实例方法/构造方法时，给目标类新增的成员变量的变量名，用于拦截器`beforeMethod`, `afterMethod`, `handleEx`之间传递中间值
+ `define`：定义方法的增强逻辑（包括绑定增强的方法范围，以及使用的拦截器）
+ `enhance`：具体的增强逻辑，包括增强构造方法、实例方法、静态方法
  + `enhanceInstance`：增强实例方法和构造方法，包括绑定方法拦截范围和拦截器指定
  + `enhanceClass`：增强静态方法，包括绑定方法拦截范围和拦截器指定

```java
package org.example.core;

import lombok.extern.slf4j.Slf4j;
import net.bytebuddy.agent.builder.AgentBuilder;
import net.bytebuddy.description.type.TypeDescription;
import net.bytebuddy.dynamic.DynamicType;
import net.bytebuddy.matcher.ElementMatcher;
import org.example.core.interceptor.ConstructorMethodsInterceptorPoint;
import org.example.core.interceptor.InstanceMethodsInterceptorPoint;
import org.example.core.interceptor.StaticMethodsInterceptorPoint;
import org.example.core.match.ClassMatch;

/**
 * 所有插件的顶级父类
 *
 * @author : Ashiamd email: ashiamd@foxmail.com
 * @date : 2023/12/30 6:10 PM
 */
@Slf4j
public abstract class AbstractClassEnhancePluginDefine {

  /**
     * 为匹配到的字节码(类)新增的成员变量的名称
     */
  public static final String CONTEXT_ATTR_NAME = "_$EnhancedClassField_ws";

  /**
     * 表示当前插件要拦截/增强的类范围 <br/>
     * 相当于对应 {@link AgentBuilder#type(ElementMatcher)} 的 参数 {@code ElementMatcher<? super TypeDescription>}
     */
  protected abstract ClassMatch enhanceClass();

  /**
     * 实例方法的拦截点 <br/>
     * ps: 类下面多个方法可能使用不同的拦截器
     */
  protected abstract InstanceMethodsInterceptorPoint[] getInstanceMethodsInterceptorPoints();

  /**
     * 构造方法的拦截点 <br/>
     */
  protected abstract ConstructorMethodsInterceptorPoint[] getConstructorMethodsInterceptorPoints();

  /**
     * 静态方法的拦截点 <br/>
     * ps: 类下面多个方法可能使用不同的拦截器
     */
  protected abstract StaticMethodsInterceptorPoint[] getStaticMethodsInterceptorPoints();

  /**
     * 增强类的主入口，绑定 当前类中哪些方法被拦截+具体的拦截器逻辑 到 builder(后续链式调用会把所有插件的配置都绑定上)
     */
  public DynamicType.Builder<?> define(TypeDescription typeDescription,
                                       DynamicType.Builder<?> builder,
                                       ClassLoader classLoader,
                                       EnhanceContext enhanceContext) {
    log.info("类: {}, 被插件: {} 增强 -- start", typeDescription, this.getClass().getName());
    DynamicType.Builder<?> newBuilder = this.enhance(typeDescription, builder, classLoader, enhanceContext);
    // 表示 当前类已经被增强过了
    enhanceContext.initializationStageCompleted();
    log.info("类: {}, 被插件: {} 增强 -- end", typeDescription, this.getClass().getName());
    return newBuilder;
  }

  /**
     * 具体的增强逻辑
     */
  private DynamicType.Builder<?> enhance(TypeDescription typeDescription,
                                         DynamicType.Builder<?> newBuilder,
                                         ClassLoader classLoader,
                                         EnhanceContext enhanceContext) {
    // 1. 静态方法增强
    newBuilder = this.enhanceClass(typeDescription, newBuilder, classLoader);
    // 2. 实例方法增强(包括构造方法)
    newBuilder = this.enhanceInstance(typeDescription, newBuilder, classLoader, enhanceContext);
    return newBuilder;
  }

  /**
     * 实例方法增强(包括构造方法)
     */
  protected abstract DynamicType.Builder<?> enhanceInstance(TypeDescription typeDescription,
                                                            DynamicType.Builder<?> newBuilder,
                                                            ClassLoader classLoader,
                                                            EnhanceContext context);

  /**
     * 静态方法增强
     */
  protected abstract DynamicType.Builder<?> enhanceClass(TypeDescription typeDescription,
                                                         DynamicType.Builder<?> newBuilder,
                                                         ClassLoader classLoader);
}
```

2. `ClassEnhancePluginDefine`

+ `enhanceInstance`：遍历实例方法拦截点和构造方法拦截点，通过Byte Buddy绑定拦截的方法范围和对应的拦截器，并且头一次增强时新增用于传递中间值的成员变量。
+ `enhanceClass`：遍历静态方法拦截点，绑定拦截的方法范围和对应的拦截器

```java
package org.example.core.interceptor.enhance;

import lombok.extern.slf4j.Slf4j;
import net.bytebuddy.description.method.MethodDescription;
import net.bytebuddy.description.type.TypeDescription;
import net.bytebuddy.dynamic.DynamicType;
import net.bytebuddy.implementation.FieldAccessor;
import net.bytebuddy.implementation.MethodDelegation;
import net.bytebuddy.implementation.SuperMethodCall;
import net.bytebuddy.jar.asm.Opcodes;
import net.bytebuddy.matcher.ElementMatcher;
import org.example.core.AbstractClassEnhancePluginDefine;
import org.example.core.EnhanceContext;
import org.example.core.interceptor.ConstructorMethodsInterceptorPoint;
import org.example.core.interceptor.InstanceMethodsInterceptorPoint;
import org.example.core.interceptor.StaticMethodsInterceptorPoint;

import static net.bytebuddy.matcher.ElementMatchers.isStatic;
import static net.bytebuddy.matcher.ElementMatchers.not;

/**
 * 所有插件直接or间接继承当前类, 当前类完成具体的增强逻辑(transform的方法拦截范围指定，以及拦截器指定)
 * <p>
 * 当前类的作用相当于:
 * {@code DynamicType.Builder<?> newBuilder = builder.method(xx).intercept(MethodDelegation.to(xx))}
 * </p>
 *
 * @author : Ashiamd email: ashiamd@foxmail.com
 * @date : 2023/12/31 5:57 PM
 */
@Slf4j
public abstract class ClassEnhancePluginDefine extends AbstractClassEnhancePluginDefine {
  @Override
  protected DynamicType.Builder<?> enhanceInstance(TypeDescription typeDescription, DynamicType.Builder<?> newBuilder, ClassLoader classLoader, EnhanceContext context) {
    ConstructorMethodsInterceptorPoint[] constructorPoints = getConstructorMethodsInterceptorPoints();
    InstanceMethodsInterceptorPoint[] instanceMethodPoints = getInstanceMethodsInterceptorPoints();
    // 构造方法拦截点是否存在
    boolean existedConstructorPoint = null != constructorPoints && constructorPoints.length > 0;
    // 实例方法拦截点是否存在
    boolean existedInstanceMethodPoint = null != instanceMethodPoints && instanceMethodPoints.length > 0;
    if (!existedConstructorPoint && !existedInstanceMethodPoint) {
      // 都不存在，则拦截的类范围内，没有需要拦截器增强的方法
      return newBuilder;
    }

    // 如果是头一次增强当前类, 则新增用于传递中间值的成员变量 (如果还没实现EnhanceContext接口，且没有新增过字段，则新增)
    if(!typeDescription.isAssignableFrom(EnhanceContext.class)
       && !context.isObjectExtended()) {
      newBuilder
        // 新增成员变量，用于拦截器逻辑传递中间值
        .defineField(CONTEXT_ATTR_NAME,Object.class, Opcodes.ACC_PRIVATE | Opcodes.ACC_VOLATILE)
        // 实现 getter, setter接口
        .implement(EnhancedInstance.class)
        .intercept(FieldAccessor.ofField(CONTEXT_ATTR_NAME));
      // 当前类已经拓展过(新增过成员变量)
      context.objectExtendedCompleted();
    }

    // 1.构造方法增强
    if(existedConstructorPoint) {
      String typeName = typeDescription.getTypeName();
      for (ConstructorMethodsInterceptorPoint constructorPoint : constructorPoints) {
        String constructorInterceptorName = constructorPoint.getConstructorInterceptor();
        if (null == constructorInterceptorName || "".equals(constructorInterceptorName.trim())) {
          log.error("插件: {} 没有指定目标类: {} 的构造方法对应的拦截器", this.getClass().getName(), typeName);
        }
        ElementMatcher<MethodDescription> constructorMatcher = constructorPoint.getConstructorMatcher();
        newBuilder = newBuilder.constructor(constructorMatcher)
          .intercept(SuperMethodCall.INSTANCE.andThen(
            // 构造方法直接结束后调用
            MethodDelegation.withDefaultConfiguration()
            .to(new ConstructorInter(constructorInterceptorName, classLoader))
          ));
      }
    }
    // 2. 实例方法增强
    if(existedInstanceMethodPoint) {
      String typeName = typeDescription.getTypeName();
      for (InstanceMethodsInterceptorPoint instancePoint : instanceMethodPoints) {
        String instanceInterceptorName = instancePoint.getMethodsInterceptor();
        if (null == instanceInterceptorName || "".equals(instanceInterceptorName.trim())) {
          log.error("插件: {} 没有指定目标类: {} 的实例方法对应的拦截器", this.getClass().getName(), typeName);
        }
        ElementMatcher<MethodDescription> methodsMatcher = instancePoint.getMethodsMatcher();
        newBuilder = newBuilder.method(not(isStatic()).and(methodsMatcher))
          .intercept(MethodDelegation.withDefaultConfiguration()
                     .to(new InstanceMethodsInter(instanceInterceptorName, classLoader)));
      }
    }
    return newBuilder;
  }

  @Override
  protected DynamicType.Builder<?> enhanceClass(TypeDescription typeDescription, DynamicType.Builder<?> newBuilder, ClassLoader classLoader) {
    StaticMethodsInterceptorPoint[] staticPoint = getStaticMethodsInterceptorPoints();
    if (null == staticPoint || staticPoint.length == 0) {
      // 当前插件没有指定 静态方法的增强逻辑(方法范围+拦截器), 则直接返回链式builder
      return newBuilder;
    }
    String typeName = typeDescription.getTypeName();
    for (StaticMethodsInterceptorPoint staticMethodsInterceptorPoint : staticPoint) {
      String methodsInterceptorName = staticMethodsInterceptorPoint.getMethodsInterceptor();
      if (null == methodsInterceptorName || "".equals(methodsInterceptorName.trim())) {
        log.error("插件: {} 没有指定目标类: {} 的静态方法对应的拦截器", this.getClass().getName(), typeName);
      }
      ElementMatcher<MethodDescription> methodsMatcher = staticMethodsInterceptorPoint.getMethodsMatcher();
      newBuilder = newBuilder.method(isStatic().and(methodsMatcher))
        .intercept(MethodDelegation.withDefaultConfiguration()
                   .to(new StaticMethodsInter(methodsInterceptorName, classLoader)));
    }
    return newBuilder;
  }
}
```

思考：

1. 代码实现方式

   通过将平时开发的经验总结，得出Byte Buddy方法增强的代码套路，并且提供相应的开发模版，简化后续开发者插件开发成本。平时项目开发时，如果有类似这种执行流程都是一致的，只不过某些特定动作不同，则可以考虑将执行流程模版化实现，只需要对其中几个步骤提供具体的实现类即可。

#### 4.5.2.4 插件信息汇总

`PluginFinder`用于搜寻指定目录下的插件文件，然后解析并汇总维护每个插件拦截的类范围，拦截的方法范围，以及方法对应的拦截器。

+ `nameMatchDefine`：成员变量，用于维护 全限制类名匹配的插件
+ `signatureMatchDefine`：成员变量，用于维护 除了全限制类名匹配以外的插件
+ `PluginFinder(List<AbstractClassEnhancePluginDefine> plugins)`：传入插件集合，解析并分类为`nameMatchDefine`或`signatureMatchDefine`
+ `ElementMatcher<? super TypeDescription> buildMatch() `：构造`AgentBuilder#type(ElementMatcher)`的参数，即汇总并返回所有插件需要拦截的类范围（并集）
+ `List<AbstractClassEnhancePluginDefine> find(TypeDescription typeDescription)`：获取拦截的目标类对应的插件集合（用于后续构造transform需要的方法拦截范围、拦截器参数）

```java
package org.example.core;

import net.bytebuddy.agent.builder.AgentBuilder;
import net.bytebuddy.description.NamedElement;
import net.bytebuddy.description.type.TypeDescription;
import net.bytebuddy.matcher.ElementMatcher;
import org.example.core.match.ClassMatch;
import org.example.core.match.IndirectMatch;
import org.example.core.match.NameMatch;

import java.util.*;

import static net.bytebuddy.matcher.ElementMatchers.*;

/**
 * 用于查找/dist/plugins下的插件
 *
 * @author : Ashiamd email: ashiamd@foxmail.com
 * @date : 2023/12/31 1:28 PM
 */
public class PluginFinder {

  /**
     * 用于存储 {@link org.example.core.match.NameMatch} 类型的插件 <br/>
     * key: 匹配的全限制类名, value: 插件plugin集合
     */
  private final Map<String, LinkedList<AbstractClassEnhancePluginDefine>> nameMatchDefine = new HashMap<>();

  /**
     * 存储 {@link org.example.core.match.IndirectMatch} 类型的插件 <br/>
     */
  private final List<AbstractClassEnhancePluginDefine> signatureMatchDefine = new ArrayList<>();

  /**
     * 对/dist/plugins 目录下的插件进行分类
     *
     * @param plugins 插件集合
     */
  public PluginFinder(List<AbstractClassEnhancePluginDefine> plugins) {
    for (AbstractClassEnhancePluginDefine plugin : plugins) {
      ClassMatch classMatch = plugin.enhanceClass();
      if (null == classMatch) {
        continue;
      }
      // 1. 全限制类名精准匹配
      if (classMatch instanceof NameMatch) {
        NameMatch nameMatch = (NameMatch) classMatch;
        nameMatchDefine.computeIfAbsent(nameMatch.getClassName(), item -> new LinkedList<>()).add(plugin);
      } else {
        // 2. 间接匹配 (本身目前的 ClassMatch实现中, 就两大类: 类名匹配, 间接匹配)
        signatureMatchDefine.add(plugin);
      }
    }
  }

  /**
     * 构造 {@link AgentBuilder#type(ElementMatcher)} 的参数, 即需要拦截的类范围 <br/>
     * 多个插件插件的类范围用or链接，表示取并集，所有插件总计需要拦截的类范围
     */
  public ElementMatcher<? super TypeDescription> buildMatch() {
    // 1. 先判断 全限制类名 匹配
    ElementMatcher.Junction<? super TypeDescription> junction = new ElementMatcher.Junction.AbstractBase<NamedElement>() {
      @Override
      public boolean matches(NamedElement target) {
        // 某个类首次被加载时回调当前逻辑
        return nameMatchDefine.containsKey(target.getActualName());
      }
    };
    // 2. 接着判断是否命中 间接匹配 (这里只增强 非 接口类)
    junction.and(not(isInterface()));
    for (AbstractClassEnhancePluginDefine pluginDefine : signatureMatchDefine) {
      ClassMatch classMatch = pluginDefine.enhanceClass();
      if (classMatch instanceof IndirectMatch) {
        // 实际上运行到这里，signatureMatchDefine中的一定时IndirectMatch
        IndirectMatch indirectMatch = (IndirectMatch) classMatch;
        // 用or链接，即取所有插件 需要拦截的类范围 并集
        junction = junction.or(indirectMatch.buildJunction());
      }
    }
    return junction;
  }

  /**
     * 从pluginFinder维护的 类=>插件 关系(插件内有 类 => 拦截点(方法范围, 拦截器)) 中搜索 当前类对应的插件集合
     *
     * @param typeDescription 被匹配到的类 (某个插件声明拦截的类)
     * @return 拦截当前指定类的插件集合
     */
  public List<AbstractClassEnhancePluginDefine> find(TypeDescription typeDescription) {
    List<AbstractClassEnhancePluginDefine> matechedPluginList = new LinkedList<>();
    String className = typeDescription.getActualName();
    // 1. 判断 全限制类名匹配的插件
    if(nameMatchDefine.containsKey(className)){
      matechedPluginList.addAll(nameMatchDefine.get(className));
    }
    // 2. 判断 间接匹配的插件 (需要插件提供 判断是否匹配的方法)
    for(AbstractClassEnhancePluginDefine indirectMatchPlugin : signatureMatchDefine) {
      IndirectMatch indirectMatch = (IndirectMatch) indirectMatchPlugin;
      if(indirectMatch.isMatch(typeDescription)) {
        matechedPluginList.add(indirectMatchPlugin);
      }
    }
    return matechedPluginList;
  }
}

```

#### 4.5.2.5 Agent指定转化器

在有了`PluginFinder`后，可以从中获取插件的信息，构造转换器中需要的方法拦截范围和拦截器。

对原本的Agent（`SkyWalkingAgent`）进一步修改：

```java
package org.example;

import lombok.extern.slf4j.Slf4j;
import net.bytebuddy.ByteBuddy;
import net.bytebuddy.agent.builder.AgentBuilder;
import net.bytebuddy.description.type.TypeDescription;
import net.bytebuddy.dynamic.DynamicType;
import net.bytebuddy.dynamic.scaffold.TypeValidation;
import net.bytebuddy.utility.JavaModule;
import org.example.core.AbstractClassEnhancePluginDefine;
import org.example.core.EnhanceContext;
import org.example.core.PluginFinder;

import java.lang.instrument.Instrumentation;
import java.security.ProtectionDomain;
import java.util.List;

/**
 * 模仿SkyWalking 的 统一 Java Agent 入口类
 *
 * @author : Ashiamd email: ashiamd@foxmail.com
 * @date : 2023/12/30 3:08 PM
 */
@Slf4j
public class SkyWalkingAgent {
  public static void premain(String args, Instrumentation instrumentation) {

    PluginFinder pluginFinder = null;
    try {
      pluginFinder = new PluginFinder(null);
    } catch (Exception e) {
      log.error("Agent初始化失败, e: ", e);
    }

    ByteBuddy byteBuddy = new ByteBuddy().with(TypeValidation.of(true));
    AgentBuilder agentBuilder = new AgentBuilder.Default(byteBuddy);

    agentBuilder
      // 1. 因为 pluginFinder负责加载插件，所以维护了所有插件需要拦截的类范围
      .type(pluginFinder.buildMatch())
      // 2. 从 所有插件中提取需要拦截的方法和拦截器
      .transform(new Transformer(pluginFinder))
      .installOn(instrumentation);
  }

  private static class Transformer implements AgentBuilder.Transformer {
    /**
         * 这里Transformer 需要知道 类对应的方法拦截范围，以及拦截器，而pluginFinder在加载插件时正好汇总了这些信息
         */
    private PluginFinder pluginFinder;

    public Transformer(PluginFinder pluginFinder) {
      this.pluginFinder = pluginFinder;
    }

    @Override
    public DynamicType.Builder<?> transform(DynamicType.Builder<?> builder,
                                            TypeDescription typeDescription,
                                            // 加载 typeDescription 的类加载器
                                            ClassLoader classLoader,
                                            JavaModule javaModule,
                                            ProtectionDomain protectionDomain) {
      log.info("typeDescription: {}, prepare to transform.", typeDescription.getActualName());
      // 1. 因为 PluginFinder 存储了 类与拦截方法范围, 拦截器之间的关系，所以从pluginFinder找每个插件拦截当前类时需要的方法范围和拦截器
      List<AbstractClassEnhancePluginDefine> pluginDefineList = pluginFinder.find(typeDescription);
      if (pluginDefineList.isEmpty()) {
        // 当前拦截的类 实际没有对应的 插件，则直接返回builder (下次进入当前方法就是下一个待判断的类了)
        log.debug("pluginFinder拦截了指定类,但是没有对应的插件(正常不该出现该情况), typeDescription: {}", typeDescription.getActualName());
        return builder;
      }
      DynamicType.Builder<?> newBuilder = builder;
      EnhanceContext enhanceContext = new EnhanceContext();
      for (AbstractClassEnhancePluginDefine pluginDefine : pluginDefineList) {
        // 2. 每个插件都有指定的拦截方法范围和拦截器逻辑，遍历(构造)每个插件需要的拦截方法范围和拦截器逻辑
        DynamicType.Builder<?> possibleBuilder = pluginDefine.define(typeDescription, newBuilder, classLoader, enhanceContext);
        // 这里相当于平时Byte Buddy代码指定了 A方法拦截范围+a拦截器，然后后面继续链式调用 B方法拦截范围+b拦截器
        newBuilder =  possibleBuilder == null ? newBuilder : possibleBuilder;
      }
      if(enhanceContext.isEnhanced()) {
        log.debug("class: {} has been enhanced", typeDescription.getActualName());
      }
      // 链式调用后，已经绑定了拦截typeDescription的插件集合的各自方法拦截范围+拦截器
      return newBuilder;
    }
  }
}
```

思考：

1. 这里transformer，每个插件都遍历其拦截的方法范围，并且指定拦截器。（`pluginDefine.define`），内部如果是实例方法或构造方法增强，会给类增强成员变量。那么多个插件增强同一个类时共用这一个变量，会不会出现问题，比如A插件的值被B覆盖之类？

   假设A和B插件都是对同一个类的同一个实例方法增强，由于Byte Buddy指定的拦截器逻辑是链式调用，并且SkyWalking要求插件实现类AOP风格的增强逻辑（`beforeMethod`, `afterMethod`, `handleEx`），成员变量值只在执行类AOP的增强方法之间传递。假设链式调用中A在B之前，那么轮到B执行时，A的完整执行结果丢给B继续使用。执行流程大致如下：

   ```pse
   1. 链中 A => B => ...其他拦截器
   
   2. A增强逻辑执行时（假设都有异常）
   
   A.beforeMethod
   被拦截的方法
   A.handleEx
   A.afterMethod
   
   3. 轮到B增强逻辑执行时
   
   B.beforeMethod
   (
   A.beforeMethod
   被拦截的方法
   A.handleEx（会继续上抛异常）
   A.afterMethod
   抛出异常（被拦截的方法的异常）
   )
   B.handleEx
   B.afterMethod（被拦截的方法的异常）
   ```

   从上面流程可以看出来，A和B实际互不影响（A和B自己的逻辑都有try-catch），所以A和B就算公用同一个成员变量，也没任何问题。

## 4.6 阶段五: 插件和拦截器类加载

### 4.6.1 目标

将指定的插件目录(`dist/plugins`)下的所有的插件和拦截器加载到Agent中，使得增强逻辑生效

### 4.6.2 代码实现和思考

#### 4.6.2.1 配置声明插件

前面虽然已经实现了插件和拦截器逻辑，但并没有实现把插件类和拦截器类实际加载到JVM中的逻辑。

这里SkyWalking通过要求插件jar包内声明特定的配置文件`skywalking-plugin.def`来告知Agent该加载哪些插件实现类。

+ `skywalking-plugin.def`：配置文件，`key=value`，key表示拦截器名（可重复），value表示拦截器全限制类名
+ `PluginDefine`：表示`skywalking-plugin.def`配置文件中的一行数据
+ `PluginCfg`：表示`skywalking-plugin.def`配置对象，汇总所有插件的配置文件的所有配置行`PluginDefine`
+ `PluginResourceResolver`：解析插件目录下所有插件jar包中配置文件`skywalking-plugin.def文件`的URL

---

+ `PluginDefine`

```java
package org.example.core;

import org.apache.commons.lang3.StringUtils;

/**
 * 表示  skywalking-plugin.def 配置文件中的一行记录
 * @author : Ashiamd email: ashiamd@foxmail.com
 * @date : 2024/1/1 3:36 AM
 */
public class PluginDefine {
  /**
     * 插件名，一个插件名可以对应多个类，配置中的key
     */
  private String name;

  /**
     * 插件的全限制类名
     */
  private String defineClass;

  public String getDefineClass() {
    return defineClass;
  }

  private PluginDefine(String name, String defineClass) {
    this.name = name;
    this.defineClass = defineClass;
  }

  /**
     * @param define 表示 skywalking-plugin.def 配置文件中的一行记录
     * @return PluginDefine 实例对象
     */
  public static PluginDefine build(String define) {
    if (StringUtils.isEmpty(define)) {
      throw new RuntimeException(define);
    }

    String[] pluginDefine = define.split("=");
    if (pluginDefine.length != 2) {
      throw new RuntimeException(define);
    }

    String pluginName = pluginDefine[0];
    String defineClass = pluginDefine[1];
    return new PluginDefine(pluginName, defineClass);
  }
}

```



+ `PluginCfg`

```java
package org.example.core;

import lombok.extern.slf4j.Slf4j;

import java.io.BufferedReader;
import java.io.IOException;
import java.io.InputStream;
import java.io.InputStreamReader;
import java.util.ArrayList;
import java.util.List;

/**
 * 表示 skywalking-plugin.def 配置
 *
 * @author : Ashiamd email: ashiamd@foxmail.com
 * @date : 2024/1/1 3:34 AM
 */
@Slf4j
public enum PluginCfg {
  /**
     * 单例
     */
  INSTANCE;

  /**
     * 所有插件的skywalking-plugin.def文件解析出来的PluginDefine实例 （也就是多行配置记录key=value）
     */
  private List<PluginDefine> pluginClassList = new ArrayList<>();

  /**
     * 转换skywalking-plugin.def文件的内容为PluginDefine
     */
  void load(InputStream input) throws IOException {
    try (BufferedReader reader = new BufferedReader(new InputStreamReader(input))) {
      String pluginDefine;
      // 读取多行配置，构造 配置行对象
      while ((pluginDefine = reader.readLine()) != null) {
        try {
          if (pluginDefine.trim().isEmpty() || pluginDefine.startsWith("#")) {
            continue;
          }
          PluginDefine plugin = PluginDefine.build(pluginDefine);
          pluginClassList.add(plugin);
        } catch (Exception e) {
          log.error("Failed to format plugin({}) define, e :", pluginDefine, e);
        }
      }
    }
  }

  public List<PluginDefine> getPluginClassList() {
    return pluginClassList;
  }
}
```



+ `PluginResourceResolver`

```java
package org.example.core;

import lombok.extern.slf4j.Slf4j;
import org.example.core.loader.AgentClassLoader;

import java.net.URL;
import java.util.ArrayList;
import java.util.Enumeration;
import java.util.List;

/**
 * 解析 插件目录下所有 插件jar包中配置文件skywalking-plugin.def文件的URL
 *
 * @author : Ashiamd email: ashiamd@foxmail.com
 * @date : 2024/1/1 2:59 AM
 */
@Slf4j
public class PluginResourceResolver {
  /**
     * 获取插件目录(/plugins)下所有jar包中的skywalking-plugin.def配置文件的URL
     */
  public List<URL> getResources() {
    List<URL> cfgUrlPaths = new ArrayList<>();
    try {
      Enumeration<URL> urls = AgentClassLoader.getDefaultLoader().getResources("skywalking-plugin.def");
      while (urls.hasMoreElements()) {
        URL pluginDefineDefUrl = urls.nextElement();
        cfgUrlPaths.add(pluginDefineDefUrl);
        log.info("find skywalking plugin define file url: {}", pluginDefineDefUrl);
      }
      return cfgUrlPaths;
    } catch (Exception e) {
      log.error("read resource error", e);
    }
    return null;
  }
}
```



思考：

1. jar包对Agent提供插件类位置的方式

   通过在jar内提供配置文件来说明插件类的全限制类名。

   这种通过约定配置来完成数据读取的方式，在项目开发中很常见，比如SpringBoot的`application.yml`配置

#### 4.6.2.2 插件类加载

Agent通过配置得知插件类的全限制类名后，还需要能够从jar文件中加载插件类，

+ `AgentPackagePath`：用于获取agent.jar（Agent的jar包）所在的目录
+ `AgentClassLoader`：自定义类加载器，用于加载插件类和拦截器类
+ `PluginBoostrap`：读取所有插件jar，使用自定义拦截器`AgentClassLoader`加载插件jar中的插件类。这里必须使用自定义类加载器，因为自带的类加载器无法从我们自定义的路径加载类

---

+ `AgentPackagePath`

```java
package org.example.core.booster;

import lombok.extern.slf4j.Slf4j;
import org.apache.commons.lang3.StringUtils;

import java.io.File;
import java.net.URL;

/**
 * 表示 agent.jar 所在的路径
 *
 * @author : Ashiamd email: ashiamd@foxmail.com
 * @date : 2024/1/1 2:01 AM
 */
@Slf4j
public class AgentPackagePath {

  /**
     * apm-agent.jar 所在路径对应的File对象
     */
  private static File AGENT_PACKAGE_PATH;

  public static File getPath() {
    return null == AGENT_PACKAGE_PATH ? AGENT_PACKAGE_PATH = findPath() : AGENT_PACKAGE_PATH;
  }

  /**
     * 获取 apm-agent.jar 所在路径对应的File对象
     */
  private static File findPath() {
    // 类名 => 路径名
    String classResourcePath = AgentPackagePath.class.getName().replaceAll("\\.", "/") + ".class";
    // file:Class文件路径.class
    // jar:file:jar文件路径.jar!/Class文件路径.class (这里因为打包，所以一定是这种情况)
    URL resource = ClassLoader.getSystemClassLoader().getResource(classResourcePath);
    if (resource != null) {
      String urlStr = resource.toString();
      // 如果是jar包中的类文件路径，则含有"!"
      boolean isInJar = urlStr.indexOf('!') > -1;
      // 因为agent会打包成jar，所以其实一定是jar里面的class类
      if (isInJar) {
        urlStr = StringUtils.substringBetween(urlStr, "file:", "!");
        File agentJarFile = null;
        try {
          agentJarFile = new File(urlStr);
        } catch (Exception e) {
          log.error("agent jar not find by url : {}, exception:", urlStr, e);
        }
        if (agentJarFile.exists()) {
          // 返回jar所在的目录对应的File对象
          return agentJarFile.getParentFile();
        }
      }
    }
    log.error("agent jar not find ");
    throw new RuntimeException("agent jar not find");
  }
}
```



+ `AgentClassLoader`

```java
package org.example.core.loader;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.apache.commons.io.IOUtils;
import org.apache.commons.lang3.ArrayUtils;
import org.example.core.AbstractClassEnhancePluginDefine;
import org.example.core.PluginBoostrap;
import org.example.core.booster.AgentPackagePath;

import java.io.File;
import java.io.IOException;
import java.net.MalformedURLException;
import java.net.URL;
import java.util.*;
import java.util.concurrent.locks.ReentrantLock;
import java.util.jar.JarEntry;
import java.util.jar.JarFile;

/**
 * 自定义类加载器，用于加载插件和插件内定义的拦截器
 *
 * @author : Ashiamd email: ashiamd@foxmail.com
 * @date : 2024/1/1 1:16 AM
 */
@Slf4j
public class AgentClassLoader extends ClassLoader {
  /**
     * 用于加载 插件jar中 {@link AbstractClassEnhancePluginDefine} 插件定义类 (jar中拦截器类由拦截的目标类对应的类加载器加载)
     */
  private static AgentClassLoader DEFAULT_LOADER;

  /**
     * 自定义类加载器 加载类的路径
     */
  private List<File> classpath;
  /**
     * 所有的插件jar包
     */
  private List<Jar> allJars;
  /**
     * 避免jar并发加载，加锁
     */
  private ReentrantLock jarScanLock = new ReentrantLock();

  public AgentClassLoader(ClassLoader parentClassLoader) {
    super(parentClassLoader);
    // 获取 agent.jar 的目录
    File agentJarDir = AgentPackagePath.getPath();
    classpath = new ArrayList<>();
    // 指定从 agent.jar 同级别目录下的子目录/plugins 加载类
    classpath.add(new File(agentJarDir, "plugins"));
  }

  public static void intDefaultLoader() {
    if (DEFAULT_LOADER == null) {
      DEFAULT_LOADER = new AgentClassLoader(PluginBoostrap.class.getClassLoader());
    }
  }

  public static AgentClassLoader getDefaultLoader() {
    return DEFAULT_LOADER;
  }

  /**
     * 双亲委派 loadClass -> 找不到 则调用当前findClass(自定义类加载逻辑) -> 最后通过defineClass获取Class对象 <br/>
     * 这里的逻辑即从 指定的插件目录加载所有的jar文件后，寻找jar中指定的 插件类 并返回其字节码
     */
  @Override
  protected Class<?> findClass(String className) throws ClassNotFoundException {
    List<Jar> allJars = getAllJars();
    // . 转为 /， 获取类对应的文件路径
    String path = className.replace(".", "/").concat(".class");
    for (Jar jar : allJars) {
      JarEntry jarEntry = jar.jarFile.getJarEntry(path);
      if (jarEntry == null) {
        continue;
      }
      try {
        URL url = new URL("jar:file:" + jar.sourceFile.getAbsolutePath() + "!/" + path);
        byte[] classBytes = IOUtils.toByteArray(url);
        return defineClass(className, classBytes, 0, classBytes.length);
      } catch (Exception e) {
        log.error("can't find class: {}, exception: ", className, e);
      }
    }
    throw new ClassNotFoundException("can't find class: " + className);
  }

  /**
     * 解析 指定类所在的 URL (从众多插件jar中寻找)
     */
  @Override
  public URL getResource(String name) {
    List<Jar> allJars = getAllJars();
    for (Jar jar : allJars) {
      JarEntry jarEntry = jar.jarFile.getJarEntry(name);
      if (jarEntry == null) {
        continue;
      }
      try {
        return new URL("jar:file:" + jar.sourceFile.getAbsolutePath() + "!/" + name);
      } catch (MalformedURLException e) {
        log.error("getResource failed, e: ", e);
      }
    }
    return null;
  }

  @Override
  public Enumeration<URL> getResources(String name) throws IOException {
    List<URL> allResources = new ArrayList<>();
    List<Jar> allJars = getAllJars();
    log.info("allJars is not empty: {}", null!=allJars && !allJars.isEmpty());
    for (Jar jar : allJars) {
      JarEntry jarEntry = jar.jarFile.getJarEntry(name);
      if (jarEntry == null) {
        continue;
      }
      allResources.add(new URL("jar:file:" + jar.sourceFile.getAbsolutePath() + "!/" + name));
    }
    Iterator<URL> iterator = allResources.iterator();
    return new Enumeration<URL>() {
      @Override
      public boolean hasMoreElements() {
        return iterator.hasNext();
      }

      @Override
      public URL nextElement() {
        return iterator.next();
      }
    };
  }

  private List<Jar> getAllJars() {
    // 确保仅加载一次jar
    if (allJars == null) {
      jarScanLock.lock();
      try {
        if (allJars == null) {
          allJars = doGetJars();
        }
      } finally {
        jarScanLock.unlock();
      }
    }
    return allJars;
  }

  private List<Jar> doGetJars() {
    List<Jar> list = new LinkedList<>();
    for (File path : classpath) {
      if (path.exists() && path.isDirectory()) {
        String[] jarFileNames = path.list((dir, name) -> name.endsWith(".jar"));
        if (ArrayUtils.isEmpty(jarFileNames)) {
          continue;
        }
        for (String jarFileName : jarFileNames) {
          try {
            File jarSourceFile = new File(path, jarFileName);
            Jar jar = new Jar(new JarFile(jarSourceFile), jarSourceFile);
            list.add(jar);
            log.info("jar: {} loaded", jarSourceFile.getAbsolutePath());
          } catch (Exception e) {
            log.error("jar: {} load failed, e: ", jarFileName, e);
          }

        }
      }
    }
    return list;
  }

  @RequiredArgsConstructor
  private static class Jar {
    /**
         * jar文件对应的jarFile对象
         */
    private final JarFile jarFile;
    /**
         * jar文件对象
         */
    private final File sourceFile;
  }
}
```



+ `PluginBoostrap`

```java
package org.example.core;

import lombok.extern.slf4j.Slf4j;
import org.example.core.loader.AgentClassLoader;

import java.net.URL;
import java.util.ArrayList;
import java.util.List;

/**
 * 加载jar文件中的 插件类
 *
 * @author : Ashiamd email: ashiamd@foxmail.com
 * @date : 2024/1/1 1:18 AM
 */
@Slf4j
public class PluginBoostrap {
  /**
     * 加载所有生效的插件(即指定目录下的所有插件jar文件)
     * <ol>
     *   <li>获取指定存放插件jar文件的路径</li>
     *   <li>使用自定义类加载器进行加载(默认的类加载器不会从我们指定的路径加载class)</li>
     * </ol>
     */
  public List<AbstractClassEnhancePluginDefine> loadPlugins() {
    AgentClassLoader.intDefaultLoader();
    PluginResourceResolver resourceResolver = new PluginResourceResolver();
    // 1. 获取插件jar中配置文件skywalking-plugin.def的 URL集合
    List<URL> resources = resourceResolver.getResources();
    if (null == resources || resources.isEmpty()) {
      log.warn("don't find any skywalking-plugin.def");
      return new ArrayList<>();
    }
    // 2. 存储所有插件jar中的 skywalking-plugin.def 的配置行记录
    for (URL resource : resources) {
      try {
        PluginCfg.INSTANCE.load(resource.openStream());
      } catch (Exception e) {
        log.error("plugin def file {} init fail", resource, e);
      }
    }
    // 3. 根据配置行中记录的全限制类名加载 插件类
    List<PluginDefine> pluginClassList = PluginCfg.INSTANCE.getPluginClassList();
    List<AbstractClassEnhancePluginDefine> plugins = new ArrayList<>();
    for (PluginDefine pluginDefine : pluginClassList) {
      try {
        // 注意这里使用自定义的类加载器 AgentClassLoader，因为只有这个类加载器知道从我们定义的路径寻找class文件
        AbstractClassEnhancePluginDefine plugin = (AbstractClassEnhancePluginDefine) Class.forName(pluginDefine.getDefineClass(),
                                                                                                   true, AgentClassLoader.getDefaultLoader()).newInstance();
        plugins.add(plugin);
      } catch (Exception e) {
        log.error("load class {} failed, e: ", pluginDefine.getDefineClass(), e);
      }
    }
    return plugins;
  }
}
```



思考：

1. 为什么插件类需要使用自定义类加载器加载？

   因为自带的类加载器无法自动从我们指定的路径加载类，所以需要使用自定义类加载器。并且需要重写以下方法：

   + `URL getResource(String name)`：返回全限制类名对应的文件的URL，因为我们使用`/dist/plugins`作为类加载路径，所以需要重写该方法，从该路径中获取类文件URL。自带的类加载器本身不知道我们自定义的类加载路径
   + `Enumeration<URL> getResources(String name)`：返回全限制类名对应的URL集合，需要重写的原因和`URL getResource(String name)`一致。
   + `protected Class<?> findClass(String className)`：返回全限制类名对应的`Class<?>`对象，同样因为需要从自定义类加载路径的插件jar中获取类信息，所以需要重写方法（自带类加载器无法得知我们的类自定义加载路径）

#### 4.6.2.3 拦截器类加载

为了保证拦截器类能够访问到被拦截的目标类，所以要求拦截器类使用的类加载器需要是被拦截类的类加载器或其子类。

+ `InterceptorInstanceLoader`：负责加载插件jar中的拦截器类，本身非类加载器，但是会使用被拦截类的类加载器作为双亲委派的父类加载器以创建一个`AgentClassLoader`类加载器对象，然后用自定义类加载器`AgentClassLoader`加载拦截器类。

```java
package org.example.core.loader;

import org.example.core.interceptor.enhance.ConstructorInterceptor;
import org.example.core.interceptor.enhance.InstanceMethodAroundInterceptor;
import org.example.core.interceptor.enhance.StaticMethodAroundInterceptor;

/**
 * 插件内的 拦截器 的加载器
 * <p>
 * 拦截器包括:
 *     <ol>
 *       <li>构造方法拦截器: {@link ConstructorInterceptor}</li>
 *       <li>实例方法拦截器: {@link InstanceMethodAroundInterceptor}</li>
 *       <li>静态方法拦截器: {@link StaticMethodAroundInterceptor}</li>
 *     </ol>
 * </p>
 *
 * @author : Ashiamd email: ashiamd@foxmail.com
 * @date : 2024/1/1 4:05 AM
 */
public class InterceptorInstanceLoader {
  /**
     * @param interceptorClassName 插件中拦截器的全限制类名
     * @param targetClassLoader    要想在插件拦截器中能够访问到被拦截的类,
     *                             需要拦截器和被拦截的类使用同一个类加载器，或拦截器的类加载器是被拦截类的类加载器的子类
     * @return ConstructorInterceptor 或 InstanceMethodsAroundInterceptor 或 StaticMethodsAroundInterceptor 的实例
     */
  public static <T> T load(String interceptorClassName, ClassLoader targetClassLoader)
    throws ClassNotFoundException, IllegalAccessException, InstantiationException {
    if (targetClassLoader == null) {
      targetClassLoader = InterceptorInstanceLoader.class.getClassLoader();
    }
    // 新建类加载器，设置父类为 被拦截类的类加载器，保证 拦截器后续能访问到被拦截类
    AgentClassLoader classLoader = new AgentClassLoader(targetClassLoader);
    return (T) Class.forName(interceptorClassName, true, classLoader).newInstance();
  }
}
```



思考：

1. 为什么拦截器类还需要使用被拦截的类的类加载器或其子类来进行类加载？

   因为在HotSpot JVM虚拟机中，不同类加载器加载的类是彼此独立且不能互相访问的。

   + 类A可以访问类B，要求类A的和类B的类加载器相同，或者类A的类加载器C1为类B的类加载C2的子类

## 4.7 总结

