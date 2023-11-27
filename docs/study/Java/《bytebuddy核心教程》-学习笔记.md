# 《bytebuddy核心教程》-学习笔记

> + 视频教程: [bytebuddy核心教程_哔哩哔哩_bilibili](https://www.bilibili.com/video/BV1G24y1a7bd)
>
> + ByteBuddy github项目: [raphw/byte-buddy: Runtime code generation for the Java virtual machine. (github.com)](https://github.com/raphw/byte-buddy)
>
> + ByteBuddy官方教程: [Byte Buddy - runtime code generation for the Java virtual machine](http://bytebuddy.net/#/tutorial-cn)
>
> + 个人学习记录的demo代码仓库: [Ashiamd/ash_bytebuddy_study](https://github.com/Ashiamd/ash_bytebuddy_study/tree/main)
>
> ---
>
> 下面的图片全部来自网络博客/文章
>
> 下面学习笔记主要由视频内容, 官方教程, 网络文章, 个人简述组成。

# 一、简介

> [raphw/byte-buddy: Runtime code generation for the Java virtual machine. (github.com)](https://github.com/raphw/byte-buddy)

​	ByteBuddy是基于[ASM (ow2.io)](https://asm.ow2.io/)实现的字节码操作类库。比起ASM，ByteBuddy的API更加简单易用。开发者无需了解[class file format](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-4.html)知识，也可通过ByteBuddy完成字节码编辑。

+ ByteBuddy使用java5实现，并且支持生成JDK6及以上版本的字节码(由于jdk6和jdk7使用未加密的HTTP类库, 作者建议至少使用jdk8版本)

+ 和其他字节码操作类库一样，ByteBuddy支持生成类和修改现存类
+ 与与静态编译器类似，需要在快速生成代码和生成快速的代码之间作出平衡，ByteBuddy主要关注以最少的运行时间生成代码

> [Byte Buddy - runtime code generation for the Java virtual machine](http://bytebuddy.net/#/tutorial-cn)

| JIT优化后的平均ns纳秒耗时(标准差) | 基线          | Byte Buddy                             | cglib              | Javassist          | Java proxy         |
| --------------------------------- | ------------- | -------------------------------------- | ------------------ | ------------------ | ------------------ |
| 普通类创建                        | 0.003 (0.001) | 142.772 (1.390)                        | 515.174 (26.753)   | 193.733 (4.430)    | 70.712 (0.645)     |
| 接口实现                          | 0.004 (0.001) | 1'126.364 (10.328)                     | 960.527 (11.788)   | 1'070.766 (59.865) | 1'060.766 (12.231) |
| stub方法调用                      | 0.002 (0.001) | 0.002 (0.001)                          | 0.003 (0.001)      | 0.011 (0.001)      | 0.008 (0.001)      |
| 类扩展                            | 0.004 (0.001) | 885.983 *5'408.329* (7.901) *(52.437)* | 1'632.730 (52.737) | 683.478 (6.735)    | –                  |
| super method invocation           | 0.004 (0.001) | 0.004 *0.004* (0.001) *(0.001)*        | 0.021 (0.001)      | 0.025 (0.001)      | –                  |

​	上表通过一些测试，对比各种场景下，不同字节码生成的耗时。对比其他同类字节码生成类库，Byte Buddy在生成字节码方面整体耗时还是可观的，并且生成后的字节码运行时耗时和基线十分相近。

+ [Java 代理](http://docs.oracle.com/javase/8/docs/api/java/lang/reflect/Proxy.html)

  Java 类库自带的一个代理工具包，它允许创建实现了一组给定接口的类。这个内置的代理很方便，但是受到的限制非常多。 例如，上面提到的安全框架不能以这种方式实现，因为我们想要扩展类而不是接口。

+ [cglib](https://github.com/cglib/cglib)

  该*代码生成库*是在 Java 开始的最初几年实现的，不幸的是，它没有跟上 Java 平台的发展。尽管如此，cglib仍然是一个相当强大的库， 但它是否积极发展变得很模糊。出于这个原因，许多用户已不再使用它。

  (cglib目前已不再维护，并且github中也推荐开发者转向使用Byte Buddy)

+ [ Javassist](https://github.com/jboss-javassist/javassist)

  该库带有一个编译器，该编译器采用包含 Java 源码的字符串，这些字符串在应用程序运行时被翻译成 Java 字节码。 这是非常雄心勃勃的，原则上是一个好主意，因为 Java 源代码显然是描述 Java 类的非常的好方法。但是， Javassist 编译器在功能上无法与 javac 编译器相比，并且在动态组合字符串以实现更复杂的逻辑时容易出错。此外， Javassist 带有一个代理库，它类似于 Java 的代理程序，但允许扩展类并且不限于接口。然而， Javassist 代理工具的范围在其API和功能方面同样受限限制。

  (2023-11-26看javassist在github上一次更新在一年前，而ByteBuddy在3天前还有更新)

# 二、常用API

## 2.1 生成一个类

### 2.1.1 注意点

1. Byte Buddy默认命名策略(NamingStrategy)，生成的类名
   1. 超类为jdk自带类: `net.bytebuddy.renamed.{超类名}$ByteBuddy${随机字符串}`
   2. 超类非jdk自带类 `{超类名}$ByteBuddy${随机字符串}`

2. 如果自定义命名策略，官方建议使用Byte Buddy内置的`NamingStrategy.SuffixingRandom`
3. Byte Buddy本身有对生成的字节码进行校验的逻辑，可通过`.with(TypeValidation.of(false))`关闭
4. `.subclass(XXX.class)` 指定超类(父类)
5. `.name("packagename.ClassName")` 指定类名

### 2.1.2 示例代码

> [ash_bytebuddy_study/bytebuddy_test/src/test/java/org/example/ByteBuddyCreateClassTest.java at main · Ashiamd/ash_bytebuddy_study (github.com)](https://github.com/Ashiamd/ash_bytebuddy_study/blob/main/bytebuddy_test/src/test/java/org/example/ByteBuddyCreateClassTest.java)

```java
package org.example;

import net.bytebuddy.ByteBuddy;
import net.bytebuddy.NamingStrategy;
import net.bytebuddy.dynamic.DynamicType;
import net.bytebuddy.dynamic.scaffold.TypeValidation;
import org.junit.Assert;
import org.junit.Test;

import java.io.IOException;
import java.util.ArrayList;

/**
 * <p>
 * 使用 Byte Buddy 生成类字节码, 用例介绍:
 *   <ol>
 *     <li>{@link ByteBuddyCreateClassTest#test01()}: 生成Object(jdk自带类)的子类, 不指定任何特定参数</li>
 *     <li>{@link ByteBuddyCreateClassTest#test02()}: 生成非jdk自带类的子类, 不指定任何特定参数</li>
 *     <li>{@link ByteBuddyCreateClassTest#test03()}: 指定父类为ArrayList, 使用官方教程建议的命名策略NamingStrategy.SuffixingRandom</li>
 *     <li>{@link ByteBuddyCreateClassTest#test04()}: 父类非jdk自带类, 指定命名策略和具体类名</li>
 *     <li>{@link ByteBuddyCreateClassTest#test05()}: 尝试指定不合法的类名, 由于Byte Buddy本身带有字节码校验逻辑, 会提前报错</li>
 *     <li>{@link ByteBuddyCreateClassTest#test06()}: 指定不合法类名, 关闭Byte Buddy自带的字节码校验逻辑(该校验虽耗费性能, 但一般对项目影响不大, 也不建议关闭)</li>
 *     <li>{@link ByteBuddyCreateClassTest#test07()}: 将生成的字节码, 注入一个jar包中</li>
 *   </ol>
 * </p>
 *
 * @author : Ashiamd email: ashiamd@foxmail.com
 * @date : 2023/11/26 11:53 PM
 */
public class ByteBuddyCreateClassTest {

  /**
     * <p>(1) 不指定任何特别的参数, 只声明为Object的子类 </p>
     * <p>
     * <a href="http://bytebuddy.net/#/tutorial-cn">官方教程</a>已经说明了没有显示命名会发生什么: <br/>
     * <pre>
     * 如果没有显式的命名会发生什么？ Byte Buddy 遵循约定优于配置原则， 并且提供我们发现的便利的默认值。
     * 至于类的名称，Byte Buddy 的默认配置提供了一个NamingStrategy（命名策略）， 它可以根据动态类的超类名称随机生成一个名称。
     * 此外，定义的类名中的包和超类相同的话，直接父类的包私有方法对动态类就是可见的。 例如，如果你子类化一个名为example.Foo的类，
     * 生成的类的名称就像example.Foo$$ByteBuddy$$1376491271，其中数字序列是随机的。
     * 这条规则的一个例外情况是：子类化的类型来自Object所在的包java.lang。 Java 的安全模型不允许自定义的类在这个命名空间。
     * 因此，在默认的命名策略中，这种类型以net.bytebuddy.renamed前缀命名。
     * </pre>
     * </p>
     * <p>
     * 根据官方教程可以看出来, 生成的新类默认命名策略即:
     *  <ol>
     *    <li>父类是jdk自带类: {超类名}$ByteBuddy${随机字符串}</li>
     *    <li>父类非jdk自带类: net.bytebuddy.renamed{超类名}$ByteBuddy${随机字符串}</li>
     *  </ol>
     * </p>
     */
  @Test
  public void test01() throws IOException {
    // 1. 创建Object的子类(Object是所有java类的父类)
    DynamicType.Unloaded<Object> objectSubClass = new ByteBuddy()
      // 表示当前新生成的类为 Object 的子类
      .subclass(Object.class).make();
    // 2. 将生成的字节码保存到 本地 (由于没有直接指定类名, 每次运行时生成不同的类, 类名不同)
    // 我本地第一次运行: net.bytebuddy.renamed.java.lang.Object$ByteBuddy$YbDNW0Kx
    // 我本地第二次运行: net.bytebuddy.renamed.java.lang.Object$ByteBuddy$FrN82cJg
    // objectSubClass.saveIn(DemoTools.currentClassPathFile());
  }

  /**
     * (2) 指定父类为非jdk自带类, 不指定命名策略和其他参数
     */
  @Test
  public void test02() throws IOException {
    // 1. 创建 非jdk自带类 的子类
    DynamicType.Unloaded<NothingClass> noJdkSubClass = new ByteBuddy()
      // 表示当前新生成的类为 NothingClass 的子类
      .subclass(NothingClass.class)
      .make();
    // 2. 将生成的字节码保存到 本地 (由于没有直接指定类名, 每次运行时生成不同的类, 类名不同)
    // 我本地第一次运行: org.example.NothingClass$ByteBuddy$f7zBKYwS
    // 我本地第二次运行: org.example.NothingClass$ByteBuddy$FHZdoEVm
    // noJdkSubClass.saveIn(DemoTools.currentClassPathFile());
  }

  /**
     * (3) 指定父类为ArrayList(jdk自带类), 使用官方教程建议的Byte Buddy自带的命名策略 (NamingStrategy.SuffixingRandom)
     */
  @Test
  public void test03() throws IOException {
    // 1. 创建 ArrayList(jdk自带类) 的子类
    DynamicType.Unloaded<ArrayList> arrayListSubClass = new ByteBuddy()
      // 使用官方教程建议的Byte Buddy自带的命名策略 (NamingStrategy.SuffixingRandom)
      .with(new NamingStrategy.SuffixingRandom("ashiamd"))
      // 表示当前新生成的类为 ArrayList 的子类
      .subclass(ArrayList.class)
      .make();
    // 2. 将生成的字节码保存到 本地 (由于没有直接指定类名, 每次运行时生成不同的类, 类名不同)
    // 我本地第一次运行: net.bytebuddy.renamed.java.util.ArrayList$ashiamd$UZCeJHeg
    // 我本地第二次运行: net.bytebuddy.renamed.java.util.ArrayList$ashiamd$HYNKU9cF
    // arrayListSubClass.saveIn(DemoTools.currentClassPathFile());
  }

  /**
     * (4) 父类非jdk自带类, 指定命名策略和具体类名
     */
  @Test
  public void test04() throws IOException {
    // 1. 创建 NothingClass 的子类
    DynamicType.Unloaded<NothingClass> nothingClassSubClass = new ByteBuddy()
      // 使用官方教程建议的Byte Buddy自带的命名策略 (NamingStrategy.SuffixingRandom)
      .with(new NamingStrategy.SuffixingRandom("ashiamd"))
      // 表示当前新生成的类为 NothingClass 的子类
      .subclass(NothingClass.class)
      // 指定类名
      .name("com.example.AshiamdTest04")
      .make();
    // 2. 将生成的字节码保存到 本地, 每次运行结果一致
    // 第N次运行: com.example.AshiamdTest04
    // nothingClassSubClass.saveIn(DemoTools.currentClassPathFile());
  }

  /**
     * (5) 尝试指定不合法的类名, 由于Byte Buddy本身带有字节码校验逻辑, 会提前报错
     */
  @Test
  public void test05() {
    try {
      // 1. 创建 NothingClass 的子类
      DynamicType.Unloaded<NothingClass> nothingClassSubClass = new ByteBuddy()
        // 使用官方教程建议的Byte Buddy自带的命名策略 (NamingStrategy.SuffixingRandom)
        .with(new NamingStrategy.SuffixingRandom("ashiamd"))
        // 表示当前新生成的类为 NothingClass 的子类
        .subclass(NothingClass.class)
        // 指定类名 (不合法, 不能以数字开头)
        .name("com.example.1111AshiamdTest05")
        .make();
    } catch (Exception e) {
      // java.lang.IllegalStateException: Illegal type name: com.example.1111AshiamdTest05 for class com.example.1111AshiamdTest04
      Assert.assertTrue(e instanceof IllegalStateException);
    }
  }

  /**
     * (6) 指定不合法类名, 关闭Byte Buddy自带的字节码校验逻辑(该校验虽耗费性能, 但一般对项目影响不大, 也不建议关闭)
     */
  @Test
  public void test06() throws IOException {
    // 1. 创建 NothingClass 的子类
    DynamicType.Unloaded<NothingClass> nothingClassSubClass = new ByteBuddy()
      // 关闭Byte Buddy的默认字节码校验逻辑
      .with(TypeValidation.of(false))
      // 使用官方教程建议的Byte Buddy自带的命名策略 (NamingStrategy.SuffixingRandom)
      .with(new NamingStrategy.SuffixingRandom("ashiamd"))
      // 表示当前新生成的类为 NothingClass 的子类
      .subclass(NothingClass.class)
      // 指定类名 (不合法, 不能以数字开头)
      .name("com.example.321AshiamdTest06")
      .make();
    // 2. 将生成的字节码保存到 本地, 生成的字节码实际非法
    // 第N次运行: com.example.321AshiamdTest06
    // nothingClassSubClass.saveIn(DemoTools.currentClassPathFile());
  }

  /**
     * (7) 将生成的字节码, 注入一个jar包中 <br/>
     * 这里本地将 simple_jar 模块打包成 simple_jar-1.0-SNAPSHOT-jar-with-dependencies.jar
     */
  @Test
  public void test07() throws IOException {
    // 1. 创建 NothingClass 的子类
    DynamicType.Unloaded<NothingClass> nothingClassSubClass = new ByteBuddy()
      // 关闭Byte Buddy的默认字节码校验逻辑
      .with(TypeValidation.of(false))
      // 使用官方教程建议的Byte Buddy自带的命名策略 (NamingStrategy.SuffixingRandom)
      .with(new NamingStrategy.SuffixingRandom("ashiamd"))
      // 表示当前新生成的类为 NothingClass 的子类
      .subclass(NothingClass.class)
      // 指定类名
      .name("com.example.AshiamdTest07")
      .make();
    // 2. 将生成的字节码 注入到 simple_jar-1.0-SNAPSHOT-jar-with-dependencies.jar 中
    // 获取当前工作目录路径 (也就是当前 bytebuddy_test 目录路径)
    String currentModulePath = System.getProperty("user.dir");
    // 获取 simple_jar 模块目录路径
    String simpleJarModulePath = currentModulePath.replace("bytebuddy_test", "simple_jar");
    // 需本地提前将simple_jar 通过 mvn package 打包
    // File jarFile = new File( simpleJarModulePath + "/target/simple_jar-1.0-SNAPSHOT.jar");
    // 本地打开jar可以看到新生成的class文件也在其中
    // nothingClassSubClass.inject(jarFile);
  }
}
```

## 2.2 对实例方法进行插桩

### 2.2.1 注意点

> [程序插桩_百度百科 (baidu.com)](https://baike.baidu.com/item/程序插桩/242087?fr=ge_ala)

java开发中说的插桩(stub)通常指对字节码进行修改(增强)。

埋点可通过插桩或其他形式实现，比如常见的代码逻辑调用次数、耗时监控打点，Android安卓应用用户操作行为打点上报等。

+ `.method(XXX)`指定后续需要修改/增强的方法
+ `.intercept(XXX)`对方法进行修改/增强
+ `DynamicType.Unloaded`表示未加载到JVM中的字节码实例
+ `DynamicType.Loaded`表示已经加载到JVM中的字节码实例
+ 无特别配置参数的情况下，通过Byte Buddy动态生成的类，实际由`net.bytebuddy.dynamic.loading.ByteArrayClassLoader`加载
+ 其他注意点，见官方教程文档的"类加载"章节，这里暂不展开

### 2.2.2 示例代码

```java
/**
  * (8) 对实例方法插桩(stub), 修改原本的toString方法逻辑
  */
@Test
public void test08() throws NoSuchMethodException, InvocationTargetException, InstantiationException, IllegalAccessException, IOException {
  // 1. 声明一个未加载到ClassLoader中的 Byte Buddy 对象
  DynamicType.Unloaded<NothingClass> nothingClassUnloaded = new ByteBuddy()
    // 指定 超类 NothingClass
    .subclass(NothingClass.class)
    // 指定要拦截(插桩)的方法
    .method(ElementMatchers.named("toString"))
    // 指定拦截(插桩)后的逻辑, 这里设置直接返回指定值
    .intercept(FixedValue.value("just nothing."))
    .name("com.example.AshiamdTest08")
    .make();
  // 2. 将类通过 AppClassLoader 加载到 JVM 中
  ClassLoader currentClassLoader = getClass().getClassLoader();
  Assert.assertEquals("app", currentClassLoader.getName());
  Assert.assertEquals("jdk.internal.loader.ClassLoaders$AppClassLoader",
                      currentClassLoader.getClass().getName());
  DynamicType.Loaded<NothingClass> loadedType = nothingClassUnloaded.load(currentClassLoader);
  // 3. 反射调用 toString方法, 验证方法内逻辑被我们修改
  Class<? extends NothingClass> loadedClazz = loadedType.getLoaded();
  Assert.assertEquals("net.bytebuddy.dynamic.loading.ByteArrayClassLoader",
                      loadedClazz.getClassLoader().getClass().getName());
  NothingClass subNothingObj = loadedClazz.getDeclaredConstructor().newInstance();
  Assert.assertEquals("just nothing.", subNothingObj.toString());
  // 4. 将字节码写入本地
  loadedType.saveIn(DemoTools.currentClassPathFile());
}
```

## 2.3 插入新方法



