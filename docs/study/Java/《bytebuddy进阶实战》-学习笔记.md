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

## 

