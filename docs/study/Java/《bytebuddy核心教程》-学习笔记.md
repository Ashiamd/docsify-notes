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
public class ByteBuddyCreateClassTest {
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
}
```

## 2.3 动态增强的三种方式

### 2.3.1 注意点

修改/增强现有类主要有3种方法，subclass(创建子类)，rebase(变基)，redefine（重定义）。

+ `.subclass(目标类.class)`：继承目标类，以子类的形式重写超类方法，达到增强效果
+ `.rebase(目标类.class)`：变基，原方法变为private，并且方法名增加`&origanl&{随机字符串}`后缀，目标方法体替换为指定逻辑
+ `.redefine(目标类.class)`：重定义，原方法体逻辑直接替换为指定逻辑

---

根据官方教程文档，对变基截取如下说明：

```java
class Foo {
  String bar() { return "bar"; }
}
```

当对类型变基时，Byte Buddy 会保留所有被变基类的方法实现。Byte Buddy 会用兼容的签名复制所有方法的实现为一个私有的重命名过的方法， 而不像*类重定义*时丢弃覆写的方法。用这种方式的话，不存在方法实现的丢失，而且变基的方法可以通过调用这些重命名的方法， 继续调用原始的方法。这样，上面的`Foo`类可能会变基为这样

```java
class Foo {
  String bar() { return "foo" + bar$original(); }
  private String bar$original() { return "bar"; }
}
```

其中`bar`方法原来返回的"bar"保存在另一个方法中，因此仍然可以访问。当对一个类变基时， Byte Buddy 会处理所有方法，就像你定义了一个子类一样。例如，如果你尝试调用变基的方法的超类方法实现， 你将会调用变基的方法。但相反，它最终会扁平化这个假设的超类为上面显示的变基的类。

### 2.3.2 示例代码

修改/增强的目标类`SomethingClass`

```java
public class SomethingClass {
  public String selectUserName(Long userId) {
    return String.valueOf(userId);
  }

  public void print() {
    System.out.println("print something");
  }

  public int getAge() {
    return Integer.MAX_VALUE;
  }
}
```

#### 2.3.2.1 subclass(子类) 

```java
public class ByteBuddyCreateClassTest {
  /**
     * (9) 通过subclass继承类, 重写父类方法
     */
  @Test
  public void test09() throws IOException {
    DynamicType.Unloaded<SomethingClass> subClass = new ByteBuddy().subclass(SomethingClass.class)
      .method(ElementMatchers.named("selectUserName")
              // 注意实际字节码Local Variable 0 位置为this引用, 但是这里说的参数位置index只需要关注方法声明时的参数顺序, 无需关注隐性参数this引用
              .and(ElementMatchers.takesArgument(0, Long.class))
              // .and(ElementMatchers.returns(Objects.class)) 匹配不到
              .and(ElementMatchers.returns(String.class))
             )
      .intercept(FixedValue.value("ashiamd"))
      .name("com.example.AshiamdTest09")
      .make();
    // subClass.saveIn(DemoTools.currentClassPathFile());
  }
}
```

通过`javap -p -c {com.example.AshiamdTest09.class的文件绝对路径}`得到字节码如下

```shell
public class com.example.AshiamdTest09 extends org.example.SomethingClass {
  public java.lang.String selectUserName(java.lang.Long);
    Code:
       0: ldc           #8                  // String ashiamd
       2: areturn

  public com.example.AshiamdTest09();
    Code:
       0: aload_0
       1: invokespecial #12                 // Method org/example/SomethingClass."<init>":()V
       4: return
}
```

可以看到`selectUserName`方法已经被重写，原本返回值由`String.valueOf(userId)`变为"ashiamd"。

#### 2.3.2.2 rebase(变基)

```java
public class ByteBuddyCreateClassTest {
  /**
     * (10) rebase变基, 原方法保留变为private且被改名(增加$original${随机字符串}后缀), 原方法名内逻辑替换成我们指定的逻辑
     */
  @Test
  public void test10() throws IOException {
    DynamicType.Unloaded<SomethingClass> rebase = new ByteBuddy()
      .rebase(SomethingClass.class)
      .method(ElementMatchers.named("selectUserName")
              // 注意实际字节码Local Variable 0 位置为this引用, 但是这里说的参数位置index只需要关注方法声明时的参数顺序, 无需关注隐性参数this引用
              .and(ElementMatchers.takesArgument(0, Long.class))
              // .and(ElementMatchers.returns(Objects.class)) 匹配不到
              .and(ElementMatchers.returns(String.class))
             )
      .intercept(FixedValue.value("ashiamd"))
      .method(ElementMatchers.named("getAge"))
      .intercept(FixedValue.value(0))
      .name("com.example.AshiamdTest10")
      .make();
    rebase.saveIn(DemoTools.currentClassPathFile());
  }
}
```

通过`javap -p -c {com.example.AshiamdTest10.class的文件绝对路径}`得到字节码如下

```shell
public class com.example.AshiamdTest10 {
  public com.example.AshiamdTest10();
    Code:
       0: aload_0
       1: invokespecial #1                  // Method java/lang/Object."<init>":()V
       4: return

  public java.lang.String selectUserName(java.lang.Long);
    Code:
       0: ldc           #50                 // String ashiamd
       2: areturn

  private java.lang.String selectUserName$original$iTRnD3qL(java.lang.Long);
    Code:
       0: aload_1
       1: invokestatic  #7                  // Method java/lang/String.valueOf:(Ljava/lang/Object;)Ljava/lang/String;
       4: areturn

  public void print();
    Code:
       0: getstatic     #13                 // Field java/lang/System.out:Ljava/io/PrintStream;
       3: ldc           #19                 // String print something
       5: invokevirtual #21                 // Method java/io/PrintStream.println:(Ljava/lang/String;)V
       8: return

  public int getAge();
    Code:
       0: iconst_0
       1: ireturn

  private int getAge$original$iTRnD3qL();
    Code:
       0: ldc           #29                 // int 2147483647
       2: ireturn
}
```

可以看到，`selectUserName()`和`getAge()`的方法内逻辑已经被我们修改，而原本的方法变成了`private`方法，并且方法名增加后缀`$original${随机字符串}`

#### 2.3.2.3 redefine(重定义)

```java
public class ByteBuddyCreateClassTest {
  /**
     * (11) redefine重定义, 重写指定的方法, 原方法逻辑不保留(被我们指定的逻辑覆盖掉)
     */
  @Test
  public void test11() throws IOException {
    DynamicType.Unloaded<SomethingClass> redefine = new ByteBuddy()
      .redefine(SomethingClass.class)
      .method(ElementMatchers.named("print")
              // 不匹配 .and(ElementMatchers.returns(NullType.class))
              // 不匹配 .and(ElementMatchers.returnsGeneric(Void.class))
              // 不匹配 .and(ElementMatchers.returns(TypeDescription.ForLoadedType.of(Void.class)))
              // 不匹配 .and(ElementMatchers.returns(Void.class))
              // 匹配 .and(ElementMatchers.returns(TypeDescription.VOID))
              // 匹配 .and(ElementMatchers.returns(void.class))
             )
      .intercept(FixedValue.value(TypeDescription.ForLoadedType.of(Void.class)))
      .name("com.example.AshiamdTest11")
      .make();
    // redefine.saveIn(DemoTools.currentClassPathFile());
  }
}
```

通过`javap -p -c {com.example.AshiamdTest11.class的文件绝对路径}`得到字节码如下

```shell
public class com.example.AshiamdTest11 {
  public com.example.AshiamdTest11();
    Code:
       0: aload_0
       1: invokespecial #1                  // Method java/lang/Object."<init>":()V
       4: return

  public java.lang.String selectUserName(java.lang.Long);
    Code:
       0: aload_1
       1: invokestatic  #7                  // Method java/lang/String.valueOf:(Ljava/lang/Object;)Ljava/lang/String;
       4: areturn

  public void print();
    Code:
       0: ldc           #50                 // class java/lang/Void
       2: pop
       3: return

  public int getAge();
    Code:
       0: ldc           #29                 // int 2147483647
       2: ireturn
}
```

可以看到`print()`方法内的逻辑已经被我们修改，并且不像rebase操作会保留原方法。

## 2.4 插入新方法

### 2.4.1 注意点

+ `.defineMethod(方法名, 方法返回值类型, 方法访问描述符)`: 定义新增的方法
+ `.withParameters(Type...)`: 定义新增的方法对应的形参类型列表
+ `.intercept(XXX)`: 和修改/增强现有方法一样，对前面的方法对象的方法体进行修改

### 2.4.2 示例代码

```java
public class ByteBuddyCreateClassTest {
  /**
    * (12) redefine基础上, 增加新方法
    */
  @Test
  public void test12() throws IOException {
    DynamicType.Unloaded<NothingClass> redefine = new ByteBuddy().redefine(NothingClass.class)
      // 定义方法的 方法名, 方法返回值类型, 方法访问修饰符
      .defineMethod("returnBlankString", String.class, Modifier.PUBLIC | Modifier.STATIC)
      // 定义方法的形参
      .withParameters(String.class, Integer.class)
      // 定义方法体内逻辑
      .intercept(FixedValue.value(""))
      .name("com.example.AshiamdTest12")
      .make();
    // redefine.saveIn(DemoTools.currentClassPathFile());
  }
}
```

通过`javap -p -c {com.example.AshiamdTest12.class的文件绝对路径}`得到字节码如下

```shell
public class com.example.AshiamdTest12 {
  public com.example.AshiamdTest12();
    Code:
       0: aload_0
       1: invokespecial #1                  // Method java/lang/Object."<init>":()V
       4: return

  public static java.lang.String returnBlankString(java.lang.String, java.lang.Integer);
    Code:
       0: ldc           #22                 // String
       2: areturn
}
```

可以看到新增了一个`private static`修饰的`returnBlankString(String, Integer)`方法，里面的逻辑就是直接返回空字符串""

## 2.5 插入新属性

### 2.5.1 注意点

+ `.defineField(String name, Type type, int modifier)`: 定义成员变量
+ `.implement(Type interfaceType)`: 指定实现的接口类
+ `.intercept(FieldAccessor.ofField("成员变量名")` 或`.intercept(FieldAccessor.ofBeanProperty())`在实现的接口为Bean规范接口时，都能生成成员变量对应的getter和setter方法

>  视频使用`intercept(FieldAccessor.ofField("成员变量名")`，而官方教程的"访问字段"章节使用`.intercept(FieldAccessor.ofBeanProperty())`来生成getter和setter方法

### 2.5.2 示例代码

后续生成getter, setter方法需要依赖的接口类定义

```java
/**
 * 简单的Bean接口(getter, setter)
 *
 * @author : Ashiamd email: ashiamd@foxmail.com
 * @date : 2023/12/2 4:35 PM
 */
public interface IAgeBean {
    int getAge();
    void setAge(int age);
}
```

给生成的子类新增"age"成员变量，并且按照Bean规范生成getter和setter方法

```java
public class ByteBuddyCreateClassTest {
  /**
    * (13) 增加新成员变量, 以及生成对应的getter, setter方法
    */
  @Test
  public void test13() throws IOException {
    DynamicType.Unloaded<NothingClass> ageBean = new ByteBuddy().subclass(NothingClass.class)
      // 定义新增的字段 name, type, 访问描述符
      .defineField("age", int.class, Modifier.PRIVATE)
      // 指定类实现指定接口(接口内定义我们需要的getter和setter方法)
      .implement(IAgeBean.class)
      // 指定实现接口的逻辑
      // ok .intercept(FieldAccessor.ofField("age"))
      .intercept(FieldAccessor.ofBeanProperty())
      .name("com.example.AshiamdTest13")
      .make();
    ageBean.saveIn(DemoTools.currentClassPathFile());
  }
}
```

使用`.intercept(FieldAccessor.ofField("age"))`和使用`.intercept(FieldAccessor.ofBeanProperty())`在这里效果是一样的，视频教程使用前者，官方文档中使用后者。

通过`javap -p -c {com.example.AshiamdTest13.class的文件绝对路径}`得到字节码如下

```shell
public class com.example.AshiamdTest13 extends org.example.NothingClass implements org.example.IAgeBean {
  private int age;

  public int getAge();
    Code:
       0: aload_0
       1: getfield      #12                 // Field age:I
       4: ireturn

  public void setAge(int);
    Code:
       0: aload_0
       1: iload_1
       2: putfield      #12                 // Field age:I
       5: return

  public com.example.AshiamdTest13();
    Code:
       0: aload_0
       1: invokespecial #18                 // Method org/example/NothingClass."<init>":()V
       4: return
}
```

## 2.6 方法委托

