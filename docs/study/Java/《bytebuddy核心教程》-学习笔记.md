# 《bytebuddy核心教程》-学习笔记

> + 视频教程: [bytebuddy核心教程_哔哩哔哩_bilibili](https://www.bilibili.com/video/BV1G24y1a7bd)
>
> + ByteBuddy github项目: [raphw/byte-buddy: Runtime code generation for the Java virtual machine. (github.com)](https://github.com/raphw/byte-buddy)
>
> + ByteBuddy官方教程: [Byte Buddy - runtime code generation for the Java virtual machine](http://bytebuddy.net/#/tutorial-cn)
>
> + 个人学习记录demo代码的gituub仓库: [Ashiamd/ash_bytebuddy_study](https://github.com/Ashiamd/ash_bytebuddy_study/tree/main)
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

### 2.6.1 注意点

方法委托，可简单理解将目标方法的方法体逻辑修改为调用指定的某个辅助类方法。

+ `.intercept(MethodDelegation.to(Class<?> type))`：将被拦截的方法委托给指定的增强类，增强类中需要定义和目标方法一致的方法签名，然后多一个static访问标识
+ `.intercept(MethodDelegation.to(Object target))`：将被拦截的方法委托给指定的增强类实例，增强类可以指定和目标类一致的方法签名，或通过`@RuntimeType`指示 Byte Buddy 终止严格类型检查以支持运行时类型转换。

其中委托给相同签名的静态方法/实例方法相对容易理解，委托给自定义方法时，该视频主要介绍几个使用到的方法参数注解：

+ `@This Object targetObj`：表示被拦截的目标对象, 只有拦截实例方法时可用
+ `@Origin Method targetMethod`：表示被拦截的目标方法, 只有拦截实例方法或静态方法时可用
+ `@AllArguments Object[] targetMethodArgs`：目标方法的参数
+ `@Super Object targetSuperObj`：表示被拦截的目标对象, 只有拦截实例方法时可用 (可用来调用目标类的super方法)。若明确知道具体的超类(父类类型)，这里`Object`可以替代为具体超类(父类)
+ `@SuperCall Callable<?> zuper`：用于调用目标方法

**其中调用目标方法时，通过`Object result = zuper.call()`。不能直接通过反射的`Object result = targetMethod.invoke(targetObj,targetMethodArgs)`进行原方法调用。因为后者会导致无限递归进入当前增强方法逻辑。**

> 其他具体细节和相关介绍，可参考[官方教程]([Byte Buddy - runtime code generation for the Java virtual machine](http://bytebuddy.net/#/tutorial-cn))的"委托方法调用"章节。尤其是各种注解的介绍，官方教程更加完善一些，但是相对比较晦涩难懂一点。

### 2.6.2 示例代码

#### 2.6.2.1 委托方法给相同方法签名的静态方法

接收委托的类，定义和需要修改/增强的目标类中的指定方法的方法签名(方法描述符)一致的方法，仅多static访问修饰符

```java
/**
 * 用于修改/增强 {@link SomethingClass#selectUserName(Long)} 方法
 *
 * @author : Ashiamd email: ashiamd@foxmail.com
 * @date : 2023/12/2 9:21 PM
 */
public class SomethingInterceptor01 {
    /**
     * 修改/增强 {@link SomethingClass#selectUserName(Long)} 方法 <br/>
     * 注意这里除了增加static访问修饰符，其他方法描述符信息和原方法(被修改/增强的目标方法)一致
     */
    public static String selectUserName(Long userId) {
        // 原方法逻辑 return String.valueOf(userId);
        return "SomethingInterceptor01.selectUserName, userId: " + userId;
    }
}
```

将原方法调用委托给静态方法

```java
public class ByteBuddyCreateClassTest {
  /**
    * (14) 将拦截的方法委托给相同方法签名的静态方法进行修改/增强
    */
  @Test
  public void test14() throws NoSuchMethodException, InvocationTargetException, InstantiationException, IllegalAccessException, IOException {
    DynamicType.Unloaded<SomethingClass> subClassUnloaded = new ByteBuddy().subclass(SomethingClass.class)
      .method(ElementMatchers.named("selectUserName"))
      // 将 selectUserName 方法委托给 SomethingInterceptor01 中的 相同方法签名(方法描述符)的静态方法 进行修改/增强
      .intercept(MethodDelegation.to(SomethingInterceptor01.class))
      .name("com.example.AshiamdTest14")
      .make();
    // 前置 saveIn则在 subClassUnloaded.load(getClass().getClassLoader()) 报错 java.lang.IllegalStateException: Class already loaded: class com.example.AshiamdTest14
    // subClassUnloaded.saveIn(DemoTools.currentClassPathFile());
    // 加载类
    String returnStr = subClassUnloaded.load(getClass().getClassLoader())
      .getLoaded()
      // 实例化并调用 selectUserName 方法验证是否被修改/增强
      .getConstructor()
      .newInstance()
      .selectUserName(1L);
    Assert.assertEquals("SomethingInterceptor01.selectUserName, userId: 1", returnStr);
    // subClassUnloaded.saveIn(DemoTools.currentClassPathFile());
  }
}
```

通过`javap -p -c {com.example.AshiamdTest14.class的文件绝对路径}`得到字节码如下

```shell
public class com.example.AshiamdTest14 extends org.example.SomethingClass {
  public java.lang.String selectUserName(java.lang.Long);
    Code:
       0: aload_1
       1: invokestatic  #10                 // Method org/example/SomethingInterceptor01.selectUserName:(Ljava/lang/Long;)Ljava/lang/String;
       4: areturn

  public com.example.AshiamdTest14();
    Code:
       0: aload_0
       1: invokespecial #14                 // Method org/example/SomethingClass."<init>":()V
       4: return
}
```

可以看出来，原本的`selectUserName`方法体内的逻辑，直接被替换成调用`SomethingInterceptor01.selectUserName`静态方法

#### 2.6.2.2 委托方法给相同方法签名的实例方法

接收方法委托的类，这里和`SomethingInterceptor01.selectUserName`方法主要不同点在于，这里定义的是实例方法，而不是静态方法；共同点即方法签名和原方法保持一致。

```java
package org.example;

/**
 * 用于修改/增强 {@link SomethingClass#selectUserName(Long)} 方法
 *
 * @author : Ashiamd email: ashiamd@foxmail.com
 * @date : 2023/12/2 9:21 PM
 */
public class SomethingInterceptor02 {

    /**
     * 修改/增强 {@link SomethingClass#selectUserName(Long)} 方法 <br/>
     * 这里和 {@link SomethingInterceptor01#selectUserName(Long)} 主要不同点在于没有 static修饰, 是实例方法
     */
    public String selectUserName(Long userId) {
        // 原方法逻辑 return String.valueOf(userId);
        return "SomethingInterceptor02.selectUserName, userId: " + userId;
    }
}
```

委托方法给实例方法

```java
public class ByteBuddyCreateClassTest {
  /**
    * (15) 将拦截的方法委托给相同方法签名的实例方法进行修改/增强
    */
  @Test
  public void test15() throws NoSuchMethodException, InvocationTargetException, InstantiationException, IllegalAccessException, IOException {
    DynamicType.Unloaded<SomethingClass> subClassUnloaded = new ByteBuddy().subclass(SomethingClass.class)
      .method(ElementMatchers.named("selectUserName"))
      // 将 selectUserName 方法委托给 SomethingInterceptor02 中的 相同方法签名(方法描述符)的实例方法 进行修改/增强
      .intercept(MethodDelegation.to(new SomethingInterceptor02()))
      .name("com.example.AshiamdTest15")
      .make();
    // 前置 saveIn则在 subClassUnloaded.load(getClass().getClassLoader()) 报错 java.lang.IllegalStateException: Class already loaded: class com.example.AshiamdTest14
    // subClassUnloaded.saveIn(DemoTools.currentClassPathFile());
    // 加载类
    String returnStr = subClassUnloaded.load(getClass().getClassLoader())
      .getLoaded()
      // 实例化并调用 selectUserName 方法验证是否被修改/增强
      .getConstructor()
      .newInstance()
      .selectUserName(2L);
    Assert.assertEquals("SomethingInterceptor02.selectUserName, userId: 2", returnStr);
    // subClassUnloaded.saveIn(DemoTools.currentClassPathFile());
  }
}
```

通过`javap -p -c -v {com.example.AshiamdTest15.class的文件绝对路径}`得到字节码如下

```shell
Classfile /Users/ashiamd/mydocs/docs/study/javadocument/javadocument/IDEA_project/ash_bytebuddy_study/bytebuddy_test/target/test-classes/com/example/AshiamdTest15.class
  Last modified Dec 2, 2023; size 366 bytes
  SHA-256 checksum 065b7bd6f6610c615a637374b06edd64875e1e2bacc5a84b7c2b7cd4b57d4140
public class com.example.AshiamdTest15 extends org.example.SomethingClass
  minor version: 0
  major version: 65
  flags: (0x0021) ACC_PUBLIC, ACC_SUPER
  this_class: #2                          // com/example/AshiamdTest15
  super_class: #4                         // org/example/SomethingClass
  interfaces: 0, fields: 1, methods: 2, attributes: 0
Constant pool:
   #1 = Utf8               com/example/AshiamdTest15
   #2 = Class              #1             // com/example/AshiamdTest15
   #3 = Utf8               org/example/SomethingClass
   #4 = Class              #3             // org/example/SomethingClass
   #5 = Utf8               delegate$2h9gn60
   #6 = Utf8               Lorg/example/SomethingInterceptor02;
   #7 = Utf8               selectUserName
   #8 = Utf8               (Ljava/lang/Long;)Ljava/lang/String;
   #9 = NameAndType        #5:#6          // delegate$2h9gn60:Lorg/example/SomethingInterceptor02;
  #10 = Fieldref           #2.#9          // com/example/AshiamdTest15.delegate$2h9gn60:Lorg/example/SomethingInterceptor02;
  #11 = Utf8               org/example/SomethingInterceptor02
  #12 = Class              #11            // org/example/SomethingInterceptor02
  #13 = NameAndType        #7:#8          // selectUserName:(Ljava/lang/Long;)Ljava/lang/String;
  #14 = Methodref          #12.#13        // org/example/SomethingInterceptor02.selectUserName:(Ljava/lang/Long;)Ljava/lang/String;
  #15 = Utf8               <init>
  #16 = Utf8               ()V
  #17 = NameAndType        #15:#16        // "<init>":()V
  #18 = Methodref          #4.#17         // org/example/SomethingClass."<init>":()V
  #19 = Utf8               Code
{
  public static volatile org.example.SomethingInterceptor02 delegate$2h9gn60;
    descriptor: Lorg/example/SomethingInterceptor02;
    flags: (0x1049) ACC_PUBLIC, ACC_STATIC, ACC_VOLATILE, ACC_SYNTHETIC

  public java.lang.String selectUserName(java.lang.Long);
    descriptor: (Ljava/lang/Long;)Ljava/lang/String;
    flags: (0x0001) ACC_PUBLIC
    Code:
      stack=2, locals=2, args_size=2
         0: getstatic     #10                 // Field delegate$2h9gn60:Lorg/example/SomethingInterceptor02;
         3: aload_1
         4: invokevirtual #14                 // Method org/example/SomethingInterceptor02.selectUserName:(Ljava/lang/Long;)Ljava/lang/String;
         7: areturn

  public com.example.AshiamdTest15();
    descriptor: ()V
    flags: (0x0001) ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: invokespecial #18                 // Method org/example/SomethingClass."<init>":()V
         4: return
}
```

注意：

1. 这里Byte Buddy还给子类添加了成员变量`public static volatile org.example.SomethingInterceptor02 delegate$2h9gn60;`。然后通过该成员变量再调用实例方法`SomethingInterceptor02.selectUserName`。

2. 这里`delegate$2h9gn60`被标注为`ACC_SYNTHETIC`，标明是动态生成的，而非来自源代码定义。

3. 在不细看Byte Buddy代码实现的情况下，可以简单推理这里`delegate$2h9gn60`变量的值来源于我们`.intercept(MethodDelegation.to(new SomethingInterceptor02()))`传递进去的实例`new SomethingInterceptor02()`。

> [Chapter 4. The class File Format (oracle.com)](https://docs.oracle.com/javase/specs/jvms/se21/html/jvms-4.html#jvms-4.1)
>
> The `ACC_SYNTHETIC` flag indicates that this field was generated by a compiler and does not appear in source code.

#### 2.6.2.3 委托方法给自定义方法

这次接收委托的类，其中定义的方法不再需要和目标类的原方法名保持方法签名一致

```java
package org.example;

import net.bytebuddy.implementation.bind.annotation.*;

import java.lang.reflect.Method;
import java.util.Arrays;
import java.util.concurrent.Callable;

/**
 * 用于修改/增强 {@link SomethingClass#selectUserName(Long)} 方法
 *
 * @author : Ashiamd email: ashiamd@foxmail.com
 * @date : 2023/12/2 9:21 PM
 */
public class SomethingInterceptor03 {

  /**
     * 修改/增强 {@link SomethingClass#selectUserName(Long)} 方法 <br/>
     * 和 {@link SomethingInterceptor01#selectUserName(Long)} 以及 {@link SomethingInterceptor02#selectUserName(Long)} 大不相同，
     * 不需要和原目标方法保持相同的方法签名 <br/>
     * 为了克服需要一致方法签名的限制，Byte Buddy 允许给方法和方法参数添加@RuntimeType注解， 它指示 Byte Buddy 终止严格类型检查以支持运行时类型转换 <br/>
     */
  @RuntimeType
  public Object otherMethodName(
    // 表示被拦截的目标对象, 只有拦截实例方法时可用
    @This Object targetObj,
    // 表示被拦截的目标方法, 只有拦截实例方法或静态方法时可用
    @Origin Method targetMethod,
    // 目标方法的参数
    @AllArguments Object[] targetMethodArgs,
    // 表示被拦截的目标对象, 只有拦截实例方法时可用 (可用来调用目标类的super方法)
    @Super Object targetSuperObj,
    // 若确定超类(父类), 也可以用具体超类(父类)接收
    // @Super SomethingClass targetSuperObj,
    // 用于调用目标方法
    @SuperCall Callable<?> zuper
  ) {
    // 原方法逻辑 return String.valueOf(userId);
    // targetObj = com.example.AshiamdTest16@79e4c792
    System.out.println("targetObj = " + targetObj);
    // targetMethod.getName() = selectUserName
    System.out.println("targetMethod.getName() = " + targetMethod.getName());
    // Arrays.toString(targetMethodArgs) = [3]
    System.out.println("Arrays.toString(targetMethodArgs) = " + Arrays.toString(targetMethodArgs));
    // targetSuperObj = com.example.AshiamdTest16@79e4c792
    System.out.println("targetSuperObj = " + targetSuperObj);
    Object result = null;
    try {
      // 调用目标方法
      result = zuper.call();
      // 直接通过反射的方式调用原方法, 会导致无限递归进入当前增强的逻辑
      // result = targetMethod.invoke(targetObj,targetMethodArgs);
    } catch (Exception e) {
      e.printStackTrace();
    }
    return result;
  }
}
```

使用自定义方法接收委托的方法

```java
public class ByteBuddyCreateClassTest {
  /**
    * (16) 将拦截的方法委托给自定义方法
    */
  @Test
  public void test16() throws NoSuchMethodException, InvocationTargetException, InstantiationException, IllegalAccessException, IOException {
    DynamicType.Unloaded<SomethingClass> subClassUnloaded = new ByteBuddy().subclass(SomethingClass.class)
      .method(ElementMatchers.named("selectUserName"))
      // 将 selectUserName 方法委托给 SomethingInterceptor03 进行修改/增强
      .intercept(MethodDelegation.to(new SomethingInterceptor03()))
      .name("com.example.AshiamdTest16")
      .make();
    // 前置 saveIn则在 subClassUnloaded.load(getClass().getClassLoader()) 报错 java.lang.IllegalStateException: Class already loaded: class com.example.AshiamdTest14
    // subClassUnloaded.saveIn(DemoTools.currentClassPathFile());
    // 加载类
    String returnStr = subClassUnloaded.load(getClass().getClassLoader())
      .getLoaded()
      // 实例化并调用 selectUserName 方法验证是否被修改/增强
      .getConstructor()
      .newInstance()
      .selectUserName(3L);
    // returnStr = 3
    System.out.println("returnStr = " + returnStr);
    // subClassUnloaded.saveIn(DemoTools.currentClassPathFile());
  }
}
```

运行得到输出如下：

```shell
targetObj = com.example.AshiamdTest16@79e4c792
targetMethod.getName() = selectUserName
Arrays.toString(targetMethodArgs) = [3]
targetSuperObj = com.example.AshiamdTest16@79e4c792
returnStr = 3
```

通过`javap -p -c -v {com.example.AshiamdTest16.class的文件绝对路径}`得到字节码如下

```shell
Classfile /Users/ashiamd/mydocs/docs/study/javadocument/javadocument/IDEA_project/ash_bytebuddy_study/bytebuddy_test/target/test-classes/com/example/AshiamdTest16.class
  Last modified Dec 2, 2023; size 1612 bytes
  SHA-256 checksum 49cf511bf549c0affeb88a10341b7d129a687ac48f7eacb263c3bd070caf40bc
public class com.example.AshiamdTest16 extends org.example.SomethingClass
  minor version: 0
  major version: 65
  flags: (0x0021) ACC_PUBLIC, ACC_SUPER
  this_class: #2                          // com/example/AshiamdTest16
  super_class: #4                         // org/example/SomethingClass
  interfaces: 0, fields: 2, methods: 8, attributes: 0
Constant pool:
   #1 = Utf8               com/example/AshiamdTest16
   #2 = Class              #1             // com/example/AshiamdTest16
   #3 = Utf8               org/example/SomethingClass
   #4 = Class              #3             // org/example/SomethingClass
   #5 = Utf8               delegate$2h9gn60
   #6 = Utf8               Lorg/example/SomethingInterceptor03;
   #7 = Utf8               selectUserName
   #8 = Utf8               (Ljava/lang/Long;)Ljava/lang/String;
   #9 = NameAndType        #5:#6          // delegate$2h9gn60:Lorg/example/SomethingInterceptor03;
  #10 = Fieldref           #2.#9          // com/example/AshiamdTest16.delegate$2h9gn60:Lorg/example/SomethingInterceptor03;
  #11 = Utf8               cachedValue$EqnJA1Hc$vvmmnn1
  #12 = Utf8               Ljava/lang/reflect/Method;
  #13 = NameAndType        #11:#12        // cachedValue$EqnJA1Hc$vvmmnn1:Ljava/lang/reflect/Method;
  #14 = Fieldref           #2.#13         // com/example/AshiamdTest16.cachedValue$EqnJA1Hc$vvmmnn1:Ljava/lang/reflect/Method;
  #15 = Utf8               java/lang/Object
  #16 = Class              #15            // java/lang/Object
  #17 = Utf8               com/example/AshiamdTest16$auxiliary$8ABGsyBN
  #18 = Class              #17            // com/example/AshiamdTest16$auxiliary$8ABGsyBN
  #19 = Utf8               <init>
  #20 = Utf8               ()V
  #21 = NameAndType        #19:#20        // "<init>":()V
  #22 = Methodref          #18.#21        // com/example/AshiamdTest16$auxiliary$8ABGsyBN."<init>":()V
  #23 = Utf8               target
  #24 = Utf8               Lcom/example/AshiamdTest16;
  #25 = NameAndType        #23:#24        // target:Lcom/example/AshiamdTest16;
  #26 = Fieldref           #18.#25        // com/example/AshiamdTest16$auxiliary$8ABGsyBN.target:Lcom/example/AshiamdTest16;
  #27 = Utf8               com/example/AshiamdTest16$auxiliary$aXiraJW5
  #28 = Class              #27            // com/example/AshiamdTest16$auxiliary$aXiraJW5
  #29 = Utf8               (Lcom/example/AshiamdTest16;Ljava/lang/Long;)V
  #30 = NameAndType        #19:#29        // "<init>":(Lcom/example/AshiamdTest16;Ljava/lang/Long;)V
  #31 = Methodref          #28.#30        // com/example/AshiamdTest16$auxiliary$aXiraJW5."<init>":(Lcom/example/AshiamdTest16;Ljava/lang/Long;)V
  #32 = Utf8               org/example/SomethingInterceptor03
  #33 = Class              #32            // org/example/SomethingInterceptor03
  #34 = Utf8               otherMethodName
  #35 = Utf8               (Ljava/lang/Object;Ljava/lang/reflect/Method;[Ljava/lang/Object;Ljava/lang/Object;Ljava/util/concurrent/Callable;)Ljava/lang/Object;
  #36 = NameAndType        #34:#35        // otherMethodName:(Ljava/lang/Object;Ljava/lang/reflect/Method;[Ljava/lang/Object;Ljava/lang/Object;Ljava/util/concurrent/Callable;)Ljava/lang/Object;
  #37 = Methodref          #33.#36        // org/example/SomethingInterceptor03.otherMethodName:(Ljava/lang/Object;Ljava/lang/reflect/Method;[Ljava/lang/Object;Ljava/lang/Object;Ljava/util/concurrent/Callable;)Ljava/lang/Object;
  #38 = Utf8               java/lang/String
  #39 = Class              #38            // java/lang/String
  #40 = Methodref          #4.#21         // org/example/SomethingClass."<init>":()V
  #41 = Utf8               <clinit>
  #42 = String             #7             // selectUserName
  #43 = Utf8               java/lang/Class
  #44 = Class              #43            // java/lang/Class
  #45 = Utf8               java/lang/Long
  #46 = Class              #45            // java/lang/Long
  #47 = Utf8               getMethod
  #48 = Utf8               (Ljava/lang/String;[Ljava/lang/Class;)Ljava/lang/reflect/Method;
  #49 = NameAndType        #47:#48        // getMethod:(Ljava/lang/String;[Ljava/lang/Class;)Ljava/lang/reflect/Method;
  #50 = Methodref          #44.#49        // java/lang/Class.getMethod:(Ljava/lang/String;[Ljava/lang/Class;)Ljava/lang/reflect/Method;
  #51 = Utf8               toString$accessor$EqnJA1Hc
  #52 = Utf8               ()Ljava/lang/String;
  #53 = Utf8               toString
  #54 = NameAndType        #53:#52        // toString:()Ljava/lang/String;
  #55 = Methodref          #4.#54         // org/example/SomethingClass.toString:()Ljava/lang/String;
  #56 = Utf8               clone$accessor$EqnJA1Hc
  #57 = Utf8               ()Ljava/lang/Object;
  #58 = Utf8               java/lang/CloneNotSupportedException
  #59 = Class              #58            // java/lang/CloneNotSupportedException
  #60 = Utf8               clone
  #61 = NameAndType        #60:#57        // clone:()Ljava/lang/Object;
  #62 = Methodref          #4.#61         // org/example/SomethingClass.clone:()Ljava/lang/Object;
  #63 = Utf8               selectUserName$accessor$EqnJA1Hc
  #64 = NameAndType        #7:#8          // selectUserName:(Ljava/lang/Long;)Ljava/lang/String;
  #65 = Methodref          #4.#64         // org/example/SomethingClass.selectUserName:(Ljava/lang/Long;)Ljava/lang/String;
  #66 = Utf8               equals$accessor$EqnJA1Hc
  #67 = Utf8               (Ljava/lang/Object;)Z
  #68 = Utf8               equals
  #69 = NameAndType        #68:#67        // equals:(Ljava/lang/Object;)Z
  #70 = Methodref          #4.#69         // org/example/SomethingClass.equals:(Ljava/lang/Object;)Z
  #71 = Utf8               hashCode$accessor$EqnJA1Hc
  #72 = Utf8               ()I
  #73 = Utf8               hashCode
  #74 = NameAndType        #73:#72        // hashCode:()I
  #75 = Methodref          #4.#74         // org/example/SomethingClass.hashCode:()I
  #76 = Utf8               Code
  #77 = Utf8               Exceptions
{
  public static volatile org.example.SomethingInterceptor03 delegate$2h9gn60;
    descriptor: Lorg/example/SomethingInterceptor03;
    flags: (0x1049) ACC_PUBLIC, ACC_STATIC, ACC_VOLATILE, ACC_SYNTHETIC

  private static final java.lang.reflect.Method cachedValue$EqnJA1Hc$vvmmnn1;
    descriptor: Ljava/lang/reflect/Method;
    flags: (0x101a) ACC_PRIVATE, ACC_STATIC, ACC_FINAL, ACC_SYNTHETIC

  public java.lang.String selectUserName(java.lang.Long);
    descriptor: (Ljava/lang/Long;)Ljava/lang/String;
    flags: (0x0001) ACC_PUBLIC
    Code:
      stack=9, locals=2, args_size=2
         0: getstatic     #10                 // Field delegate$2h9gn60:Lorg/example/SomethingInterceptor03;
         3: aload_0
         4: getstatic     #14                 // Field cachedValue$EqnJA1Hc$vvmmnn1:Ljava/lang/reflect/Method;
         7: iconst_1
         8: anewarray     #16                 // class java/lang/Object
        11: dup
        12: iconst_0
        13: aload_1
        14: aastore
        15: new           #18                 // class com/example/AshiamdTest16$auxiliary$8ABGsyBN
        18: dup
        19: invokespecial #22                 // Method com/example/AshiamdTest16$auxiliary$8ABGsyBN."<init>":()V
        22: dup
        23: aload_0
        24: putfield      #26                 // Field com/example/AshiamdTest16$auxiliary$8ABGsyBN.target:Lcom/example/AshiamdTest16;
        27: new           #28                 // class com/example/AshiamdTest16$auxiliary$aXiraJW5
        30: dup
        31: aload_0
        32: aload_1
        33: invokespecial #31                 // Method com/example/AshiamdTest16$auxiliary$aXiraJW5."<init>":(Lcom/example/AshiamdTest16;Ljava/lang/Long;)V
        36: invokevirtual #37                 // Method org/example/SomethingInterceptor03.otherMethodName:(Ljava/lang/Object;Ljava/lang/reflect/Method;[Ljava/lang/Object;Ljava/lang/Object;Ljava/util/concurrent/Callable;)Ljava/lang/Object;
        39: checkcast     #39                 // class java/lang/String
        42: areturn

  public com.example.AshiamdTest16();
    descriptor: ()V
    flags: (0x0001) ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: invokespecial #40                 // Method org/example/SomethingClass."<init>":()V
         4: return

  static {};
    descriptor: ()V
    flags: (0x0008) ACC_STATIC
    Code:
      stack=6, locals=0, args_size=0
         0: ldc           #4                  // class org/example/SomethingClass
         2: ldc           #42                 // String selectUserName
         4: iconst_1
         5: anewarray     #44                 // class java/lang/Class
         8: dup
         9: iconst_0
        10: ldc           #46                 // class java/lang/Long
        12: aastore
        13: invokevirtual #50                 // Method java/lang/Class.getMethod:(Ljava/lang/String;[Ljava/lang/Class;)Ljava/lang/reflect/Method;
        16: putstatic     #14                 // Field cachedValue$EqnJA1Hc$vvmmnn1:Ljava/lang/reflect/Method;
        19: return

  final java.lang.String toString$accessor$EqnJA1Hc();
    descriptor: ()Ljava/lang/String;
    flags: (0x1010) ACC_FINAL, ACC_SYNTHETIC
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: invokespecial #55                 // Method org/example/SomethingClass.toString:()Ljava/lang/String;
         4: areturn

  final java.lang.Object clone$accessor$EqnJA1Hc() throws java.lang.CloneNotSupportedException;
    descriptor: ()Ljava/lang/Object;
    flags: (0x1010) ACC_FINAL, ACC_SYNTHETIC
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: invokespecial #62                 // Method org/example/SomethingClass.clone:()Ljava/lang/Object;
         4: areturn
    Exceptions:
      throws java.lang.CloneNotSupportedException

  final java.lang.String selectUserName$accessor$EqnJA1Hc(java.lang.Long);
    descriptor: (Ljava/lang/Long;)Ljava/lang/String;
    flags: (0x1010) ACC_FINAL, ACC_SYNTHETIC
    Code:
      stack=2, locals=2, args_size=2
         0: aload_0
         1: aload_1
         2: invokespecial #65                 // Method org/example/SomethingClass.selectUserName:(Ljava/lang/Long;)Ljava/lang/String;
         5: areturn

  final boolean equals$accessor$EqnJA1Hc(java.lang.Object);
    descriptor: (Ljava/lang/Object;)Z
    flags: (0x1010) ACC_FINAL, ACC_SYNTHETIC
    Code:
      stack=2, locals=2, args_size=2
         0: aload_0
         1: aload_1
         2: invokespecial #70                 // Method org/example/SomethingClass.equals:(Ljava/lang/Object;)Z
         5: ireturn

  final int hashCode$accessor$EqnJA1Hc();
    descriptor: ()I
    flags: (0x1010) ACC_FINAL, ACC_SYNTHETIC
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: invokespecial #75                 // Method org/example/SomethingClass.hashCode:()I
         4: ireturn
}
```

1. 这里使用自定义类接收委托，和`SomethingInterceptor02`同样是以实例委托给`SomethingInterceptor03`实例进行实际方法修改/增强，所以也是生成了一个`SomethingInterceptor03`成员变量，然后通过该成员变量调用增强方法`SomethingInterceptor03.otherMethod`
2. 从字节码可以看出来，比起委托给相同方法签名的静态方法/实例方法，这里委托给灵活性更高的自定义方法，需要额外生成更多的`ACC_SYNTHETIC`方法，并且还多了许多其他逻辑，这里不是重点，暂不细究
3. 实际除了`AshiamdTest16`外，还另外生成了2个辅助类(AuxiliaryType)的class文件，可以本地试试，这里暂不展开介绍。

## 2.7 动态修改入参

### 2.7.1 注意点

+ **`@Morph`：和`@SuperCall`功能基本一致，主要区别在于`@Morph`支持传入参数**。

+ **使用`@Morph`时，需要在拦截方法注册代理类/实例前，指定install注册配合`@Morph`使用的函数式接口，其入参必须为`Object[]`类型，并且返回值必须为`Object`类型**。

  ```java
  .intercept(MethodDelegation
                   .withDefaultConfiguration()
             			 // 向Byte Buddy 注册 用于中转目标方法入参和返回值的 函数式接口
                   .withBinders(Morph.Binder.install(MyCallable.class))
                   .to(new SomethingInterceptor04()))
  ```

> java源代码中`@Mopth`的文档注释如下：
>
> ```java
> /**
>  * This annotation instructs Byte Buddy to inject a proxy class that calls a method's super method with
>  * explicit arguments. For this, the {@link Morph.Binder}
>  * needs to be installed for an interface type that takes an argument of the array type {@link java.lang.Object} and
>  * returns a non-array type of {@link java.lang.Object}. This is an alternative to using the
>  * {@link net.bytebuddy.implementation.bind.annotation.SuperCall} or
>  * {@link net.bytebuddy.implementation.bind.annotation.DefaultCall} annotations which call a super
>  * method using the same arguments as the intercepted method was invoked with.
>  *
>  * @see net.bytebuddy.implementation.MethodDelegation
>  * @see net.bytebuddy.implementation.bind.annotation.TargetMethodAnnotationDrivenBinder
>  */
> @Documented
> @Retention(RetentionPolicy.RUNTIME)
> @Target(ElementType.PARAMETER)
> public @interface Morph {
>   ...
> }
> ```

### 2.7.2 示例代码

用来中转目标方法的入参和返回值的接口，后续注册到Byte Buddy中，才能实现借助`@Morph`修改目标方法入参的效果

```java
package org.example;

/**
 * 用于后续接收目标方法的参数, 以及中转返回值的函数式接口 <br/>
 * 入参必须是 Object[], 返回值必须是 Object
 *
 * @author : Ashiamd email: ashiamd@foxmail.com
 * @date : 2023/12/3 8:38 AM
 */
@FunctionalInterface
public interface MyCallable {
    // java.lang.IllegalArgumentException: public abstract java.lang.String org.example.MyCallable.apply(java.lang.Object[]) does not return an Object-type
    // String apply(Object[] args);

    // java: incompatible types: java.lang.Object[] cannot be converted to java.lang.Long[]
    // Object apply(Long[] args);

    Object apply(Object[] args);
}
```

用于增强目标方法的类

```java
package org.example;

import net.bytebuddy.implementation.bind.annotation.*;

import java.lang.reflect.Method;
import java.util.Arrays;
import java.util.concurrent.Callable;

/**
 * 用于修改/增强 {@link SomethingClass#selectUserName(Long)} 方法
 *
 * @author : Ashiamd email: ashiamd@foxmail.com
 * @date : 2023/12/2 9:21 PM
 */
public class SomethingInterceptor04 {

  /**
     * 修改/增强 {@link SomethingClass#selectUserName(Long)} 方法 <br/>
     * <a href="http://bytebuddy.net/#/tutorial-cn" target="_blank">Byte Buddy官方教程文档</a>
     * <p>
     *  {@code @Morph}: 这个注解的工作方式与{@code @SuperCall}注解非常相似。然而，使用这个注解允许指定用于调用超类方法参数。
     *  注意， 仅当你需要调用具有与原始调用不同参数的超类方法时，才应该使用此注解，因为使用@Morph注解需要对所有参数装箱和拆箱。
     *  如果过你想调用一个特定的超类方法， 请考虑使用@Super注解来创建类型安全的代理。在这个注解被使用之前，需要显式地安装和注册，类似于@Pipe注解。
     * </p>
     */
  @RuntimeType
  public Object otherMethodName(
    // 目标方法的参数
    @AllArguments Object[] targetMethodArgs,
    // @SuperCall Callable<?> zuper
    // 用于调用目标方法 (这里使用@Morph, 而不是@SuperCall, 才能修改入参)
    @Morph MyCallable zuper
  ) {
    // 原方法逻辑 return String.valueOf(userId);
    Object result = null;
    try {
      // 修改参数
      if(null != targetMethodArgs && targetMethodArgs.length > 0) {
        targetMethodArgs[0] = (long) targetMethodArgs[0] + 1;
      }
      // @SuperCall 不接受参数 result = zuper.call();
      // 调用目标方法
      result = zuper.apply(targetMethodArgs);
    } catch (Exception e) {
      e.printStackTrace();
    }
    return result;
  }
}
```

生成增强目标方法的类

```java
public class ByteBuddyCreateClassTest {
  /**
    * (17) 通过@Morph动态修改方法入参
    */
  @Test
  public void test17() throws NoSuchMethodException, InvocationTargetException, InstantiationException, IllegalAccessException, IOException {
    DynamicType.Unloaded<SomethingClass> subClassUnloaded = new ByteBuddy().subclass(SomethingClass.class)
      .method(ElementMatchers.named("selectUserName"))
      .intercept(MethodDelegation
                 .withDefaultConfiguration()
                 // 向Byte Buddy 注册 用于中转目标方法入参和返回值的 函数式接口
                 .withBinders(Morph.Binder.install(MyCallable.class))
                 .to(new SomethingInterceptor04()))
      .name("com.example.AshiamdTest17")
      .make();
    String returnStr = subClassUnloaded.load(getClass().getClassLoader())
      .getLoaded()
      // 实例化并调用 selectUserName 方法验证是否被修改/增强
      .getConstructor()
      .newInstance()
      .selectUserName(3L);
    // 符合预期，第一个参数被修改+1
    Assert.assertEquals("4", returnStr);
    subClassUnloaded.saveIn(DemoTools.currentClassPathFile());
  }
}
```

通过`javap -p -c {com.example.AshiamdTest17.class的文件绝对路径}`得到字节码如下

```shell
public class com.example.AshiamdTest17 extends org.example.SomethingClass {
  public static volatile org.example.SomethingInterceptor04 delegate$0ebbdv1;

  public java.lang.String selectUserName(java.lang.Long);
    Code:
       0: getstatic     #10                 // Field delegate$0ebbdv1:Lorg/example/SomethingInterceptor04;
       3: iconst_1
       4: anewarray     #12                 // class java/lang/Object
       7: dup
       8: iconst_0
       9: aload_1
      10: aastore
      11: new           #14                 // class com/example/AshiamdTest17$auxiliary$rULEYsoa
      14: dup
      15: aload_0
      16: invokespecial #18                 // Method com/example/AshiamdTest17$auxiliary$rULEYsoa."<init>":(Lcom/example/AshiamdTest17;)V
      19: invokevirtual #24                 // Method org/example/SomethingInterceptor04.otherMethodName:([Ljava/lang/Object;Lorg/example/MyCallable;)Ljava/lang/Object;
      22: checkcast     #26                 // class java/lang/String
      25: areturn

  public com.example.AshiamdTest17();
    Code:
       0: aload_0
       1: invokespecial #29                 // Method org/example/SomethingClass."<init>":()V
       4: return

  final java.lang.String selectUserName$accessor$JPV1Twdw(java.lang.Long);
    Code:
       0: aload_0
       1: aload_1
       2: invokespecial #32                 // Method org/example/SomethingClass.selectUserName:(Ljava/lang/Long;)Ljava/lang/String;
       5: areturn
}
```

实际还生成了一个辅助类(AuxiliaryType)，`AshiamdTest17$auxiliary$rULEYsoa.class`，对应的字节码如下：

```shell
class com.example.AshiamdTest17$auxiliary$rULEYsoa implements org.example.MyCallable {
  private final com.example.AshiamdTest17 target;

  public java.lang.Object apply(java.lang.Object[]);
    Code:
       0: aload_0
       1: getfield      #12                 // Field target:Lcom/example/AshiamdTest17;
       4: aload_1
       5: iconst_0
       6: aaload
       7: checkcast     #14                 // class java/lang/Long
      10: invokevirtual #20                 // Method com/example/AshiamdTest17.selectUserName$accessor$JPV1Twdw:(Ljava/lang/Long;)Ljava/lang/String;
      13: areturn

  com.example.AshiamdTest17$auxiliary$rULEYsoa(com.example.AshiamdTest17);
    Code:
       0: aload_0
       1: invokespecial #25                 // Method java/lang/Object."<init>":()V
       4: aload_0
       5: aload_1
       6: putfield      #12                 // Field target:Lcom/example/AshiamdTest17;
       9: return
}
```

可以看到生成的辅助类实现了我们定义的函数式接口`MyCallable`，可以看出来该辅助类的主要作用即中转目标方法的入参和返回值，本身没有其他太多逻辑。

## 2.8 对构造方法进行插桩

### 2.8.1 注意点

+ `.constructor(ElementMatchers.any())`: 表示拦截目标类的任意构造方法
+ `.intercept(SuperMethodCall.INSTANCE.andThen(Composable implementation)`: 表示在实例构造方法逻辑执行结束后再执行拦截器中定义的增强逻辑
+ `@This`: 被拦截的目标对象this引用，构造方法也是实例方法，同样有this引用可以使用

### 2.8.2 示例代码

给需要增强的类上新增构造方法，方便后续掩饰构造方法插桩效果

```java
/**
 * 具有一些方法的类
 *
 * @author : Ashiamd email: ashiamd@foxmail.com
 * @date : 2023/11/28 4:55 PM
 */
public class SomethingClass {

  public SomethingClass() {
    System.out.println("SomethingClass()");
  }
  // 省略其他方法
}
```

新建用于增强构造器方法的拦截器类，里面描述构造方法直接结束后，后续执行的逻辑

```java
package org.example;

import net.bytebuddy.implementation.bind.annotation.RuntimeType;
import net.bytebuddy.implementation.bind.annotation.This;

/**
 * 用于增强 {@link SomethingClass#SomethingClass()} 构造方法
 *
 * @author : Ashiamd email: ashiamd@foxmail.com
 * @date : 2023/12/3 11:30 AM
 */
public class SomethingInterceptor05 {

  @RuntimeType
  public void constructEnhance(
    //  表示被拦截的目标对象, 在构造方法中同样是可用的(也是实例方法)
    @This Object targetObj) {
    // constructEnhance() , com.example.AshiamdTest18@10163d6
    System.out.println("constructEnhance() , " + targetObj);
  }
}
```

生成增强类，运行查看标准输出

```java
public class ByteBuddyCreateClassTest {
  /**
     * (18) 对构造方法插桩
     */
  @Test
  public void test18() throws NoSuchMethodException, InvocationTargetException, InstantiationException, IllegalAccessException, IOException {
    DynamicType.Unloaded<SomethingClass> subClassUnloaded = new ByteBuddy().subclass(SomethingClass.class)
      // 对任何构造方法都进行插桩
      .constructor(ElementMatchers.any())
      // 表示在被拦截的构造方法原方法逻辑执行完后，再委托给拦截器
      .intercept(SuperMethodCall.INSTANCE.andThen(
        MethodDelegation.to(new SomethingInterceptor05())))
      .name("com.example.AshiamdTest18")
      .make();
    subClassUnloaded.load(getClass().getClassLoader())
      .getLoaded()
      // 实例化并调用 selectUserName 方法验证是否被修改/增强
      .getConstructor()
      .newInstance();
    // subClassUnloaded.saveIn(DemoTools.currentClassPathFile());
  }
}
```

输出如下：

```shell
SomethingClass()
constructEnhance() , com.example.AshiamdTest18@10163d6
```

通过`javap -p -c {com.example.AshiamdTest18.class的文件绝对路径}`得到字节码如下

```shell
public class com.example.AshiamdTest18 extends org.example.SomethingClass {
  public static volatile org.example.SomethingInterceptor05 delegate$n4v9vh1;

  public com.example.AshiamdTest18();
    Code:
       0: aload_0
       1: invokespecial #10                 // Method org/example/SomethingClass."<init>":()V
       4: getstatic     #12                 // Field delegate$n4v9vh1:Lorg/example/SomethingInterceptor05;
       7: aload_0
       8: invokevirtual #18                 // Method org/example/SomethingInterceptor05.constructEnhance:(Ljava/lang/Object;)V
      11: return
}
```

从字节码也可以看出来，先是调用超类构造方法`org/example/SomethingClass."<init>":()V`，然后才是调用增强方法`org/example/SomethingInterceptor05.constructEnhance:(Ljava/lang/Object;)V`。

## 2.9 对静态方法进行插桩

### 2.9.1 注意点

+ 增强静态方法时，通过`@This`和`@Super`获取不到目标对象
+ 增强静态方法时，通过`@Origin Class<?> clazz`可获取静态方法所处的Class对象

### 2.9.2 示例代码

给目标类增加static静态方法，后续演示增强静态方法

```java
package org.example;

/**
 * 具有一些方法的类
 *
 * @author : Ashiamd email: ashiamd@foxmail.com
 * @date : 2023/11/28 4:55 PM
 */
public class SomethingClass {
   // ... 省略其他方法
    public static void sayWhat(String whatToSay) {
        System.out.println("what to Say, say: " + whatToSay);
    }
}
```

定义拦截器的增强逻辑

```java
package org.example;

import net.bytebuddy.implementation.bind.annotation.*;

import java.lang.reflect.Method;
import java.util.Arrays;
import java.util.concurrent.Callable;

/**
 * 用于修改/增强 {@link SomethingClass#sayWhat(String)} 静态方法
 *
 * @author : Ashiamd email: ashiamd@foxmail.com
 * @date : 2023/12/3 7:30 PM
 */
public class SomethingInterceptor06 {

  @RuntimeType
  public void sayWhatEnhance(
    // 静态方法对应的类class对象
    @Origin Class<?> clazz,
    // 静态方法不可访问 @This Object targetObj,
    @Origin Method targetMethod,
    @AllArguments Object[] targetMethodArgs,
    // 静态方法不可访问 @Super Object targetSuperObj,
    @SuperCall Callable<?> zuper) {
    // 原方法逻辑 System.out.println("what to Say, say: " + whatToSay);
    System.out.println("clazz = " + clazz);
    System.out.println("targetMethod.getName() = " + targetMethod.getName());
    System.out.println("Arrays.toString(targetMethodArgs) = " + Arrays.toString(targetMethodArgs));
    try {
      System.out.println("before sayWhat");
      // 调用目标方法
      zuper.call();
      System.out.println("after sayWhat");
    } catch (Exception e) {
      e.printStackTrace();
    }
  }
}
```

生成增强类

```java
public class ByteBuddyCreateClassTest {
  /**
    * (19) 对静态方法插桩
    */
  @Test
  public void test19() throws InvocationTargetException, IllegalAccessException, IOException, NoSuchMethodException {
    DynamicType.Unloaded<SomethingClass> sayWhatUnload = new ByteBuddy().rebase(SomethingClass.class)
      // 拦截 名为 "sayWhat" 的静态方法
      .method(ElementMatchers.named("sayWhat").and(ModifierReviewable.OfByteCodeElement::isStatic))
      // 拦截后的修改/增强逻辑
      .intercept(MethodDelegation.to(new SomethingInterceptor06()))
      .name("com.example.AshiamdTest19")
      .make();
    // 调用类静态方法, 验证是否执行了增强逻辑
    Class<? extends SomethingClass> loadedClazz = sayWhatUnload.load(getClass().getClassLoader())
      .getLoaded();
    Method sayWhatMethod = loadedClazz.getMethod("sayWhat", String.class);
    sayWhatMethod.invoke(null, "hello world");
    // sayWhatUnload.saveIn(DemoTools.currentClassPathFile());
  }
}
```

运行后，标准输出如下

```shell
clazz = class com.example.AshiamdTest19
targetMethod.getName() = sayWhat
Arrays.toString(targetMethodArgs) = [hello world]
before sayWhat
what to Say, say: hello world
after sayWhat
```

## 2.10 @SuperCall, rebase, redefine, subclass

### 2.9.1 注意点

+ `@SuperCall`仅在原方法仍存在的场合能够正常使用，比如`subclass`超类方法仍为目标方法，而`rebase`则是会重命名目标方法并保留原方法体逻辑；但`redefine`直接替换掉目标方法，所以`@SuperCall`不可用
+ `rebase`和`redefine`都可以修改目标类静态方法，但是若想在原静态方法逻辑基础上增加其他增强逻辑，那么只有`rebase`能通过`@SuperCall`或`@Morph`调用到原方法逻辑；`redefine`不保留原目标方法逻辑

### 2.9.2 示例代码

这里使用的示例代码和"2.9.2 示例代码"一致，主要是用于说明前面"2.9 对静态方法进行插桩"时为什么只能用rebase，而不能用subclass；以及使用rebase后，整个增强的大致调用流程。

+ `subclass`：以目标类子类的形式，重写父类方法完成修改/增强。子类不能重写静态方法，所以增强目标类的静态方法时，不能用`subclass`
+ `redefine`：因为redefine不保留目标类原方法，所以`SomethingInterceptor06`中的`sayWhatEnhance`方法获取不到`@SuperCall Callable<?> zuper`参数，若注解掉zuper相关的代码，发现能正常运行，但是目标方法相当于直接被替换成我们的逻辑，达不到保留原方法逻辑并增强的目的。
+ `rebase`：原方法会被重命名并保留原逻辑，所以能够在通过`@SuperCall Callable<?> zuper`保留执行原方法逻辑执行的情况下，继续执行我们自定义的修改/增强逻辑

使用`rebase`生成了两个class，一个为`AshaimdTest19.class`，一个为辅助类`AshiamdTest19$auxiliary$souxJETk.class`。

查看`AshaimdTest19.class`反编译结果，结合其字节码进行分析：

```java
public class AshiamdTest19 {
 //... 省略其他代码
  public static void sayWhat(String var0) {
    // 参数分别对应 SomethingInterceptor06 的 sayWhatEnhance方法定义的参数
    // @Origin Class<?> clazz, @Origin Method targetMethod, @AllArguments Object[] targetMethodArgs, @SuperCall Callable<?> zuper
        delegate$e6tl5f0.sayWhatEnhance(AshiamdTest19.class, cachedValue$E9ljqfKp$01vs1t0, new Object[]{var0}, new AshiamdTest19$auxiliary$souxJETk(var0));
    }
}
```

+ `delegate$e6tl5f0`：通过字节码查看，得知是`SomethingInterceptor06`实例引用，即这里调用拦截器类的`sayWhatEnhance`实例方法。

+ `cachedValue$E9ljqfKp$01vs1t0`：通过查看字节码，得知对应`AshiamdTest19`类的`sayWhat`静态方法
+ `new Object[]{var0}`：原方法参数
+ `new AshiamdTest19$auxiliary$souxJETk(var0)`：`AshiamdTest19$auxiliary$souxJETk`对应ByteBuddy生成的辅助类实例，`var0`则是原方法参数

接下来看看Byte Buddy生成的辅助类

```java
class AshiamdTest19$auxiliary$souxJETk implements Runnable, Callable {
  private String argument0;

  public Object call() throws Exception {
    AshiamdTest19.sayWhat$original$dWjxaaDU$accessor$E9ljqfKp(this.argument0);
    return null;
  }

  public void run() {
    AshiamdTest19.sayWhat$original$dWjxaaDU$accessor$E9ljqfKp(this.argument0);
  }

  AshiamdTest19$auxiliary$souxJETk(String var1) {
    this.argument0 = var1;
  }
}
```

这里结合生成的`AshiamdTest19`代码，可知辅助类`AshiamdTest19$auxiliary$souxJETk`作用就是对应拦截器类里面的`zuper.call();`逻辑，其中转原方法参数，调用原方法逻辑

+ `AshiamdTest19.sayWhat$original$dWjxaaDU$accessor$E9ljqfKp`：rebase后，保留的被重命名的原方法

整理一下逻辑，即：

Byte Buddy通过我们指定的代码增强`SomethingClass.sayWhat`方法后，执行逻辑大致可描述为：

1. 生成拦截器类`AshiamdTest19`和`AshiamdTest19$auxiliary$souxJETk`

2. 调用`AshiamdTest19`的`sayWhat`方法时，如下流程

   1. `AshiamdTest19`的`sayWhat`对应拦截器类的`@Origin Method targetMethod`
   2. 目标方法`SomethingClass.sayWhat`逻辑则被重命名为`AshiamdTest19.sayWhat$original$dWjxaaDU$accessor$E9ljqfKp`，对应拦截器类的`@SuperCall Callable<?> zuper`

   3. 执行`AshiamdTest19.sayWhat`即执行拦截器类的`SomethingInterceptor06.sayWhatEnhance`实例方法。

## 2.11 rebase, redefine默认生成类名

`subclass`, `rebase`, `redefine`各自的默认命名策略如下：

+ `.subclass(目标类.class)`：
  + 超类为jdk自带类: `net.bytebuddy.renamed.{超类名}$ByteBuddy${随机字符串}`
  + 超类非jdk自带类 `{超类名}$ByteBuddy${随机字符串}`
+ `.rebase(目标类.class)`：和目标类的类名一致（效果上即覆盖原本的目标类class文件）
+ `.redefine(目标类.class)`：和目标类的类名一致（效果上即覆盖原本的目标类class文件）

这里就不写示例代码了，实验的方式很简单，即把自己指定的类名`.name(yyy.zzz.Xxxx)`去掉，即根据默认命名策略生成类名

## 2.12 bytebuddy的类加载器

### 2.12.1 注意点

+ `DynamicType.Unloaded<SomethingClass>实例.load(getClass().getClassLoader()).getLoaded()`等同于`DynamicType.Unloaded<SomethingClass>实例.load(getClass().getClassLoader(), ClassLoadingStrategy.Default.WRAPPER).getLoaded()`

  Byte Buddy默认使用`WRAPPER`类加载策略，该策略会优先根据类加载的双亲委派机制委派父类加载器加载指定类，若类成功被父类加载器加载，此处仍通过`.load`加载类就报错。（直观上就是将生成的类的`.class`文件保存到本地后，继续执行`.load`方法会抛异常`java.lang.IllegalStateException: Class already loaded`）

+ **若使用`CHILD_FIRST`类加载策略，那么打破双亲委派机制，优先在当前类加载器加载类**（直观上就是将生成的类的`.class`文件保存到本地后，继续执行`.load`方法不会报错，`.class`类由ByteBuddy的ByteArrayClassLoader正常加载）。具体代码可见`net.bytebuddy.dynamic.loading.ByteArrayClassLoader.ChildFirst#loadClass`

下面摘出`net.bytebuddy.dynamic.loading.ByteArrayClassLoader.ChildFirst#loadClass`源代码

```java
/**
     * Loads the class with the specified <a href="#binary-name">binary name</a>.  The
     * default implementation of this method searches for classes in the
     * following order:
     *
     * <ol>
     *
     *   <li><p> Invoke {@link #findLoadedClass(String)} to check if the class
     *   has already been loaded.  </p></li>
     *
     *   <li><p> Invoke the {@link #loadClass(String) loadClass} method
     *   on the parent class loader.  If the parent is {@code null} the class
     *   loader built into the virtual machine is used, instead.  </p></li>
     *
     *   <li><p> Invoke the {@link #findClass(String)} method to find the
     *   class.  </p></li>
     *
     * </ol>
     *
     * <p> If the class was found using the above steps, and the
     * {@code resolve} flag is true, this method will then invoke the {@link
     * #resolveClass(Class)} method on the resulting {@code Class} object.
     *
     * <p> Subclasses of {@code ClassLoader} are encouraged to override {@link
     * #findClass(String)}, rather than this method.  </p>
     *
     * <p> Unless overridden, this method synchronizes on the result of
     * {@link #getClassLoadingLock getClassLoadingLock} method
     * during the entire class loading process.
     *
     * @param   name
     *          The <a href="#binary-name">binary name</a> of the class
     *
     * @param   resolve
     *          If {@code true} then resolve the class
     *
     * @return  The resulting {@code Class} object
     *
     * @throws  ClassNotFoundException
     *          If the class could not be found
     */
protected Class<?> loadClass(String name, boolean resolve) throws ClassNotFoundException {
  synchronized (SYNCHRONIZATION_STRATEGY.initialize().getClassLoadingLock(this, name)) {
    Class<?> type = findLoadedClass(name);
    if (type != null) {
      return type;
    }
    try {
      type = findClass(name);
      if (resolve) {
        resolveClass(type);
      }
      return type;
    } catch (ClassNotFoundException exception) {
      // If an unknown class is loaded, this implementation causes the findClass method of this instance
      // to be triggered twice. This is however of minor importance because this would result in a
      // ClassNotFoundException what does not alter the outcome.
      return super.loadClass(name, resolve);
    }
  }
}
```

---

> 其他关于类加载的介绍，可以查阅[Byte Buddy官方教程文档](http://bytebuddy.net/#/tutorial-cn)的"类加载"章节，下面内容摘自官方教程文档

​	目前为止，我们只是创建了一个动态类型，但是我们并没有使用它。Byte Buddy 创建的类型是通过`DynamicType.Unloaded`的一个实例来表示的。通过名称可以猜到，这些类不会加载到JVM。 相反，Byte Buddy 创建的类以[Java 类文件格式](http://docs.oracle.com/javase/specs/jvms/se8/html/jvms-4.html)的二进制结构表示。 这样的话，你可以决定用生成的类来做什么。例如，你或许想从构建脚本运行 Byte Buddy，该脚本仅在部署前生成类以增强 Java 应用。 对于这个目的，`DynamicType.Unloaded`类允许提取动态类型的字节数组。为了方便， 该类型还额外提供了`saveIn(File)`方法，该方法允许你将一个类保存到给定的文件夹。此外， 它允许你通过`inject(File)`方法将类注入到已存在的 *jar* 文件。

​	虽然直接访问一个类的二进制结构是直截了当的，但不幸的是加载一个类更复杂。**在 Java 里，所有的类都用`ClassLoader(类加载器)`加载。 这种类加载器的一个示例是启动类加载器，它负责加载 Java 类库里的类。另一方面，系统类加载器负责加载 Java 应用程序类路径里的类。 显然，这些预先存在的类加载器都不知道我们创建的任何动态类。为了解决这个问题，我们需要找其他的可能性用于加载运行时生成的类**。 Byte Buddy 通过开箱即用的不同方法提供解决方案：

- 我们仅仅创建一个新的`ClassLoader`，它被明确地告知存在一个特定的动态创建的类。 因为 Java 类加载器是按层级组织的，我们定义的这个类加载器是程序里已经存在的类加载器的孩子。这样， 程序里的所有类对于新`类加载器`加载的动态类型都是可见的。
- 通常，Java 类加载器在尝试直接加载给定名称的类之前会询问他的父`类加载器`。这意味着，在父类加载器知道有相同名称的类时， 子类加载器通常不会加载类。为此，**Byte Buddy 提供了孩子优先创建的类加载器，它在询问父类加载器之前会尝试自己加载类**。 除此之外，这种方法类似于刚才上面提及的方法。注意，这种方法不会覆盖父类加载器加载的类，而是隐藏其他类型。
- 最后，我们可以用反射将一个类注入到已存在的`类加载器`。通常，类加载器会被要求通过类名称来提供一个给定的类。 用反射我们可以扭转这个规则，调用受保护的方法将一个新类注入类加载器，而类加载器实际上不知道如何定位这个动态类。

不幸的是，上面的方法都有其缺点：

- **如果我们创建一个新的`ClassLoader`，这个类加载器会定义一个新的命名空间。 这样可能会通过两个不同的类加载器加载两个有相同名称的类。这两个类永远不会被JVM视为相等，即时这两个类是相同的类实现**。 这个相等规则也适用于 Java 包。这意味着，如果不是用相同的类加载器加载， `example.Foo`类无法访问`example.Bar`类的包私有方法。此外， 如果`example.Bar`继承`example.Foo`，任何被覆写的包私有方法都将变为无效，但会委托给原始实现。
- 每当加载一个类时，一旦引用另一种类型的代码段被解析，它的类加载器将查找该类中引用的所有类型。该查找会委托给同一个类加载器。 想象一下这种场景：我们动态的创建了`example.Foo`和`example.Bar`两个类， 如果我们将`example.Foo`注入一个已经存在的类加载器，这个类加载器可能会尝试定位查找`example.Bar`。 然而，这个查找会失败，因为后一个类是动态创建的，而且对于刚才注入`example.Foo`类的类加载器来说是不可达的。 因此反射的方法不能用于在类加载期间生效的带有循环依赖的类。**幸运的是，大多数JVM的实现在第一次使用时都会延迟解析引用类， 这就是类注入通常在没有这些限制的时候正常工作的原因。此外，实际上，由 Byte Buddy 创建的类通常不会受这样的循环影响**。

​	你可能会任务遇到循环依赖的机会是无关紧要的，因为一次只创建一个动态类。然而，动态类型的创建可能会触发辅助类型的创建。 这些类型由 Byte Buddy 自动创建，以提供对正在创建的动态类型的访问。我们将在下面的章节学习辅助类型，现在不要担心这些。 但是，正因为如此，我们推荐你尽可能通过创建一个特定的`ClassLoader`来加载动态类， 而不是将他们注入到一个已存在的类加载器。

​	创建一个`DynamicType.Unloaded`后，这个类型可以用`ClassLoadingStrategy`加载。 如果没有提供这个策略，Byte Buddy 会基于提供的类加载器推测出一种策略，并且仅为启动类加载器创建一个新的类加载器， 该类加载器不能用反射的方式注入任何类。否则为默认设置。

​	Byte Buddy 提供了几种开箱即用的类加载策略， 每一种都遵循上述概念中的其中一个。**这些策略都在`ClassLoadingStrategy.Default`中定义，其中， `WRAPPER`策略会创建一个新的，经过包装的`ClassLoader`， `CHILD_FIRST`策略会创建一个类似的具有孩子优先语义的类加载器，`INJECTION`策略会用反射注入一个动态类型**。

​	 `WRAPPER`和`CHILD_FIRST`策略也可以在所谓的*manifest(清单)*版本中使用，即使在类加载后， 也会保留类的二进制格式。这些可替代的版本使类加载器加载的类的二进制表示可以通过`ClassLoader::getResourceAsStream`方法访问。 但是，请注意，这需要这些类加载器保留一个类的完整的二进制表示的引用，这会占用 JVM 堆上的空间。因此， 如果你打算实际访问类的二进制格式，你应该只使用清单版本。由于`INJECTION`策略通过反射实现， 而且不可能改变方法ClassLoader::getResourceAsStream的语义，因此它自然在清单版本中不可用。

​	让我们看一下这样的类加载：

```java
Class<?> type = new ByteBuddy()
  .subclass(Object.class)
  .make()
  .load(getClass().getClassLoader(), ClassLoadingStrategy.Default.WRAPPER)
  .getLoaded();
```

​	在上面的示例中，我们创建并加载了一个类。像我们之前提到的，我们用`WRAPPER`加载策略加载类， 它适用于大多数场景。最后，`getLoaded`方法返回了一个现在已经加载的 Java `Class(类)`的实例， 这个实例代表着动态类。

​	注意，当加载类时，预定义的类加载策略是通过应用执行上下文的`ProtectionDomain`来执行的。或者， 所有默认的策略通过调用`withProtectionDomain`方法来提供明确地保护域规范。 当使用安全管理器或使用签名jar包中定义的类时，定义一个明确地保护域是非常重要的。

### 2.12.2 示例代码

1. 默认类加载策略`WRAPPER`，不保存`.class`文件到本地，重复加载类

   ```java
   public class ByteBuddyCreateClassTest {
     /**
       * (20) 默认类加载策略`WRAPPER`, 不保存`.class`文件到本地, 重复加载类
       */
     @Test
     public void test20() {
       DynamicType.Unloaded<SomethingClass> sayWhatUnload = new ByteBuddy().rebase(SomethingClass.class)
         .method(ElementMatchers.named("sayWhat").and(ModifierReviewable.OfByteCodeElement::isStatic))
         .intercept(MethodDelegation.to(new SomethingInterceptor06()))
         .name("com.example.AshiamdTest20")
         .make();
       Class<? extends SomethingClass> loaded01 = sayWhatUnload.load(getClass().getClassLoader()).getLoaded();
       Class<? extends SomethingClass> loaded02 = sayWhatUnload.load(getClass().getClassLoader()).getLoaded();
       Assert.assertNotEquals(loaded01, loaded02);
       // loaded01 = class com.example.AshiamdTest20
       System.out.println("loaded01 = " + loaded01);
       // loaded02 = class com.example.AshiamdTest20
       System.out.println("loaded02 = " + loaded02);
       // loaded01.hashCode() = 589273327
       System.out.println("loaded01.hashCode() = " + loaded01.hashCode());
       // loaded02.hashCode() = 609656250
       System.out.println("loaded02.hashCode() = " + loaded02.hashCode());
     }
   }
   ```

   可以看到，每次加载出来的`Class<? extends SomethingClass>`指向堆中的Class对象实际是不同的

2. 默认类加载策略`WRAPPER`，保存`.class`文件到本地，之后加载类

   ```java
   public class ByteBuddyCreateClassTest {
     /**
       * (21) 默认类加载策略`WRAPPER`,保存`.class`文件到本地, 之后加载类
       */
     @Test
     public void test21() throws IOException {
       DynamicType.Unloaded<SomethingClass> sayWhatUnload = new ByteBuddy().rebase(SomethingClass.class)
         .method(ElementMatchers.named("sayWhat").and(ModifierReviewable.OfByteCodeElement::isStatic))
         .intercept(MethodDelegation.to(new SomethingInterceptor06()))
         .name("com.example.AshiamdTest21")
         .make();
       sayWhatUnload.saveIn(DemoTools.currentClassPathFile());
       Assert.assertThrows(IllegalStateException.class,
                           // 会抛出 java.lang.IllegalStateException: Class already loaded: class com.example.AshiamdTest21
                           () -> sayWhatUnload.load(getClass().getClassLoader()).getLoaded());
     }
   }
   ```

   根据调试对比，可以发现在没有`sayWhatUnload.saveIn(DemoTools.currentClassPathFile());`这行代码时，内部执行逻辑到`net.bytebuddy.dynamic.loading.ByteArrayClassLoader#load(java.lang.ClassLoader, java.util.Map<net.bytebuddy.description.type.TypeDescription,byte[]>, java.security.ProtectionDomain, net.bytebuddy.dynamic.loading.ByteArrayClassLoader.PersistenceHandler, net.bytebuddy.dynamic.loading.PackageDefinitionStrategy, boolean, boolean)`方法时，`type.getClassLoader() != classLoader`为false，这里两边都是`ByteArrayClassLoader`。

   若将`.class`文件保存到本地后，会发现`type.getClassLoader() != classLoader`为true，左边为`AppClassLoader`。

   ```java
   /**
        * Loads a given set of class descriptions and their binary representations.
        *
        * @param classLoader               The parent class loader.
        * @param types                     The unloaded types to be loaded.
        * @param protectionDomain          The protection domain to apply where {@code null} references an implicit protection domain.
        * @param persistenceHandler        The persistence handler of the created class loader.
        * @param packageDefinitionStrategy The package definer to be queried for package definitions.
        * @param forbidExisting            {@code true} if the class loading should throw an exception if a class was already loaded by a parent class loader.
        * @param sealed                    {@code true} if the class loader should be sealed.
        * @return A map of the given type descriptions pointing to their loaded representations.
        */
   @SuppressFBWarnings(value = "DP_CREATE_CLASSLOADER_INSIDE_DO_PRIVILEGED", justification = "Assuring privilege is explicit user responsibility.")
   public static Map<TypeDescription, Class<?>> load(@MaybeNull ClassLoader classLoader,
                                                     Map<TypeDescription, byte[]> types,
                                                     @MaybeNull ProtectionDomain protectionDomain,
                                                     PersistenceHandler persistenceHandler,
                                                     PackageDefinitionStrategy packageDefinitionStrategy,
                                                     boolean forbidExisting,
                                                     boolean sealed) {
     Map<String, byte[]> typesByName = new HashMap<String, byte[]>();
     for (Map.Entry<TypeDescription, byte[]> entry : types.entrySet()) {
       typesByName.put(entry.getKey().getName(), entry.getValue());
     }
     classLoader = new ByteArrayClassLoader(classLoader,
                                            sealed,
                                            typesByName,
                                            protectionDomain,
                                            persistenceHandler,
                                            packageDefinitionStrategy,
                                            ClassFilePostProcessor.NoOp.INSTANCE);
     Map<TypeDescription, Class<?>> result = new LinkedHashMap<TypeDescription, Class<?>>();
     for (TypeDescription typeDescription : types.keySet()) {
       try {
         Class<?> type = Class.forName(typeDescription.getName(), false, classLoader);
         if (!GraalImageCode.getCurrent().isNativeImageExecution() 
             && forbidExisting 
             // 将类文件保存到本地后，type被AppClassLoader加载; 否则被ByteArrayClassLoader加载
             // classLoader 在这里都是 ByteArrayClassLoader
             && type.getClassLoader() != classLoader) {
           throw new IllegalStateException("Class already loaded: " + type);
         }
         result.put(typeDescription, type);
       } catch (ClassNotFoundException exception) {
         throw new IllegalStateException("Cannot load class " + typeDescription, exception);
       }
     }
     return result;
   }
   ```

3. 类加载策略`CHILD_FIRST`，保存`.class`文件到本地，之后重复加载类

   ```java
   public class ByteBuddyCreateClassTest {
     /**
       * (22) 类加载策略`CHILD_FIRST`，保存`.class`文件到本地，之后重复加载类
       */
     @Test
     public void test22() throws IOException {
       DynamicType.Unloaded<SomethingClass> sayWhatUnload = new ByteBuddy().rebase(SomethingClass.class)
         .method(ElementMatchers.named("sayWhat").and(ModifierReviewable.OfByteCodeElement::isStatic))
         .intercept(MethodDelegation.to(new SomethingInterceptor06()))
         .name("com.example.AshiamdTest22")
         .make();
       sayWhatUnload.saveIn(DemoTools.currentClassPathFile());
       Class<? extends SomethingClass> loaded01 = sayWhatUnload.load(getClass().getClassLoader(),
                                                                     ClassLoadingStrategy.Default.CHILD_FIRST).getLoaded();
       Class<? extends SomethingClass> loaded02 = sayWhatUnload.load(getClass().getClassLoader(),
                                                                     ClassLoadingStrategy.Default.CHILD_FIRST).getLoaded();
       Assert.assertNotEquals(loaded01, loaded02);
       // loaded01 = class com.example.AshiamdTest22
       System.out.println("loaded01 = " + loaded01);
       // loaded02 = class com.example.AshiamdTest22
       System.out.println("loaded02 = " + loaded02);
       // loaded01.hashCode() = 1293680734
       System.out.println("loaded01.hashCode() = " + loaded01.hashCode());
       // loaded02.hashCode() = 611520720
       System.out.println("loaded02.hashCode() = " + loaded02.hashCode());
     }
   }
   ```

   可看出来，使用`CHILD_FIRST`类加载策略时，即使`sayWhatUnload.saveIn(DemoTools.currentClassPathFile());`保存类文件到本地，也不会报错。因为该策略优先使用当前类加载器加载类，但是重复加载时，同样生成不同的Class对象

4. redefine后，配合`CHILD_FIRST`重新加载类

   ```java
   public class ByteBuddyCreateClassTest {
     /**
       * (23) redefine后，配合`CHILD_FIRST`加载类
       */
     @Test
     public void test23() throws IOException, NoSuchMethodException, InvocationTargetException, InstantiationException, IllegalAccessException {
       DynamicType.Unloaded<NothingClass> redefine = new ByteBuddy().redefine(NothingClass.class)
         .defineMethod("returnBlankString", String.class, Modifier.PUBLIC | Modifier.STATIC)
         .withParameters(String.class, Integer.class)
         .intercept(FixedValue.value(""))
         .make();
   
       // redefine.saveIn(DemoTools.currentClassPathFile());
   
       Class<? extends NothingClass> loaded01 = redefine.load(getClass().getClassLoader(), ClassLoadingStrategy.Default.CHILD_FIRST).getLoaded();
       // loaded01 = class org.example.NothingClass
       System.out.println("loaded01 = " + loaded01);
       // loaded01.equals(NothingClass.class) = false
       System.out.println("loaded01.equals(NothingClass.class) = " + loaded01.equals(NothingClass.class));
       // loaded01.getClassLoader() = net.bytebuddy.dynamic.loading.ByteArrayClassLoader$ChildFirst@23348b5d
       System.out.println("loaded01.getClassLoader() = " + loaded01.getClassLoader());
       // NothingClass.class.getClassLoader() = jdk.internal.loader.ClassLoaders$AppClassLoader@4e0e2f2a
       System.out.println("NothingClass.class.getClassLoader() = " + NothingClass.class.getClassLoader());
       // loaded01.getDeclaredConstructor().newInstance() instanceof NothingClass = false
       System.out.println("loaded01.getDeclaredConstructor().newInstance() instanceof NothingClass = " +
                          (loaded01.getDeclaredConstructor().newInstance() instanceof NothingClass));
   
       Class<? extends NothingClass> loaded02 = redefine.load(getClass().getClassLoader(), ClassLoadingStrategy.Default.CHILD_FIRST).getLoaded();
       // loaded02 = class org.example.NothingClass
       System.out.println("loaded02 = " + loaded02);
       // loaded01.equals(loaded02) = false
       System.out.println("loaded01.equals(loaded02) = " + loaded01.equals(loaded02));
       // loaded01.hashCode() = 1725008249
       System.out.println("loaded01.hashCode() = " + loaded01.hashCode());
       // loaded02.hashCode() = 1620890840
       System.out.println("loaded02.hashCode() = " + loaded02.hashCode());
     }
   }
   ```

   可以看出来这里代码中的NothingClass还是指被redefine之前，由AppClassLoader加载的原类。

## 2.13 自定义类的加载路径

### 2.13.1 注意点

+ `ClassFileLocator`：类定位器，用来定位类文件所在的路径，支持jar包所在路径，`.class`文件所在路径，类加载器等。
  + `ClassFileLocator.ForJarFile.of(File file)`：jar包所在路径
  + `ClassFileLocator.ForFolder(File file)`：`.class`文件所在路径
  + `ClassFileLocator.ForClassLoader.ofXxxLoader()`：类加载器
  + 一般使用时都需要带上`ClassFileLocator.ForClassLoader.ofSystemLoader()`，才能保证jdk自带类能够正常被扫描识别到，否则会抛出异常(`net.bytebuddy.pool.TypePool$Resolution$NoSuchTypeException: Cannot resolve type description for java.lang.Object`)。
+ `ClassFileLocator.Compound`：本身也是类定位器，用于整合多个`ClassFileLocator`。
+ `TypePool`：类型池，一般配合`ClassFileLocator.Compound`使用，用于从指定的多个类定位器内获取类描述对象
  + 调用`typePool.describe("全限制类名").resolve()`获取`TypeDescription`类描述对象，**`resolve()`不会触发类加载。**
+ `TypeDescription`：类描述对象，用于描述java类，后续`subclass`, `rebase`, `redefine`时用于指定需要修改/增改的类。

---

> 其他介绍可见官方教程文档的"重新加载类"和"使用未加载的类"章节，下面内容摘至官方教程文档：

​	使用 Java 的 HotSwap 功能有一个巨大的缺陷，**HotSwap的当前实现要求重定义的类在重定义前后应用相同的类模式。 这意味着当重新加载类时，不允许添加方法或字段**。我们已经讨论过 Byte Buddy 为任何变基的类定义了原始方法的副本， 因此类的变基不适用于`ClassReloadingStrategy`。此外，类重定义不适用于具有显式的类初始化程序的方法(类中的静态块)的类， 因为该初始化程序也需要复制到额外的方法中。不幸的是， OpenJDK已经退出了[扩展HotSwap的功能](http://openjdk.java.net/jeps/159)， 因此，无法使用HotSwap的功能解决此限制。同时，Byte Buddy 的HotSwap支持可用于某些看起来有用的极端情况。 否则，当(例如，从构建脚本)增强存在的类时，变基和重定义可能是一个便利的功能。

​	意识到HotSwap功能的局限性后，人们可能会认为`变基`和`重定义`指令的唯一有意义的应用是在构建期间。 通过应用构建时的处理，人们可以断言一个已经处理过的类在它的初始类简单地加载之前没有被加载，因为这个类加载是在不同的JVM实例中完成的。 然而，**Byte Buddy 同样有能力处理尚未加载的类。为此，Byte Buddy 抽象了 Java 的反射 API，例如， 一个`Class`实例在内部由一个`TypeDescription`表示**。事实上， Byte Buddy 只知道如何处理由实现了`TypeDescription`接口的适配器提供的`Class`。 这种抽象的最大好处是类的信息不需要由`类加载器`提供，而是可以由其他的源提供。

​	**Byte Buddy 使用`TypePool(类型池)`，提供了一种标准的方式来获取类的`TypeDescription(类描述)`**。当然， 这个池的默认实现也提供了。`TypePool.Default`的实现解析类的二进制格式并将其表示为需要的`TypeDescription`。 类似于`类加载器`为加载好的类维护一个缓存，该缓存也是可定制的。此外，**它通常从`类加载器`中检索类的二进制格式， 但不指示它加载此类**。

​	JVM仅在第一次使用时加载一个类。因此，我们可以安全的重定义一个类，例如：

```java
package foo;
class Bar { }
```

​	在运行任何其他的代码之前，程序启动时：

```java
class MyApplication {
  public static void main(String[] args) {
    TypePool typePool = TypePool.Default.ofSystemLoader();
    Class bar = new ByteBuddy()
      .redefine(typePool.describe("foo.Bar").resolve(), // do not use 'Bar.class'
                ClassFileLocator.ForClassLoader.ofSystemLoader())
      .defineField("qux", String.class) // we learn more about defining fields later
      .make()
      .load(ClassLoader.getSystemClassLoader(), ClassLoadingStrategy.Default.INJECTION)
      .getLoaded();
    assertThat(bar.getDeclaredField("qux"), notNullValue());
  }
}
```

​	通过第一次在断言中使用类之前显式地重定义类，我们先于JVM内置的类加载。这样，重新定义的类`foo.Bar`就被加载了， 并且贯穿整个应用的运行时。然而，请**注意，当我们用`TypePool`来提供类型描述时，我们不会通过一个类的字面量(literal)来引用该类。 如果我们用了`foo.bar`的字面量，JVM 将在我们有机会重定义该类之前加载它，我们的重定义尝试将无效**。此外，请注意， 当处理未加载的类时，我们还需要指定一个`ClassFileLocator(类文件定位器)`，它允许定位类的类文件。在上面的示例中， 我们简单地创建了一个类文件定位器，它扫描了正在运行的应用的类路径以查找foo.Bar这样的文件。

### 2.13.2 示例代码

```java
public class ByteBuddyCreateClassTest {
  /**
     * (24) 从指定 “jar包”, “文件目录”, “系统类加载器” 加载指定类
     */
  @Test
  public void test24() throws IOException {
    // 1. 指定需要扫描的jar包路径
    ClassFileLocator jarPathLocator = ClassFileLocator.ForJarFile.of(new File("/Users/ashiamd/mydocs/dev-tools/apache-maven-3.9.3/repository/commons-io/commons-io/2.15.0/commons-io-2.15.0.jar"));
    // 2. 指定需要扫描的.class文件所在路径
    ClassFileLocator.ForFolder classPathLocator = new ClassFileLocator.ForFolder(new File("/Users/ashiamd/mydocs/docs/study/javadocument/javadocument/IDEA_project/ash_bytebuddy_study/bytebuddy_test/target/test-classes"));
    // 3. 从系统类加载器中扫描类 (不加则找不到jdk自身的类)
    ClassFileLocator classLoaderLocator = ClassFileLocator.ForClassLoader.ofSystemLoader();
    // 整合 多个 自定义的类扫描路径
    ClassFileLocator.Compound locatorCompound = new ClassFileLocator.Compound(jarPathLocator, classPathLocator, classLoaderLocator);
    // locatorCompound 去掉 classLoaderLocator 后, 后续net.bytebuddy.ByteBuddy.redefine(ByteBuddy.java:886)往下调用时,
    // 报错 net.bytebuddy.pool.TypePool$Resolution$NoSuchTypeException: Cannot resolve type description for java.lang.Object
    // ClassFileLocator.Compound locatorCompound = new ClassFileLocator.Compound(jarPathLocator, classPathLocator);
    // 类型池, 提供根据 全限制类名 从指定 类路径扫描范围内 获取 类描述对象 的方法
    TypePool typePool = TypePool.Default.of(locatorCompound);
    // 4. 从前面指定的扫描类范围中, 获取 “commons-io-2.15.0.jar” 内 FileUtils 类描述对象, resolve()不会触发类加载
    TypeDescription jarPathTypeDescription = typePool.describe("org.apache.commons.io.FileUtils").resolve();
    // 5. 获取 target下测试类路径的NothingClass类
    TypeDescription classPathTypeDescription = typePool.describe("org.example.NothingClass").resolve();

    // 6-1 redefine 指定 jar包内的 FileUtils 类, 并将生成的.class文件保存到本地
    new ByteBuddy().redefine(jarPathTypeDescription, locatorCompound)
      .method(ElementMatchers.named("current"))
      .intercept(FixedValue.nullValue())
      .make()
      .saveIn(DemoTools.currentClassPathFile());

    // 6-2 redefine 指定.class文件路径内的 NothingClass类, 并将生成的.class文件保存到本地
    new ByteBuddy().redefine(classPathTypeDescription, locatorCompound)
      .defineMethod("justVoid", void.class, Modifier.PUBLIC)
      .intercept(FixedValue.value(void.class))
      .make()
      .saveIn(DemoTools.currentClassPathFile());
  }
}
```

运行后查看target目录下的类文件，可验证逻辑生效。`org.apache.commons.io.FileUtils#current()`的方法提逻辑已经变成返回null，而`org.example.Nothing`也新增了一个名为`justVoid`的方法

## 2.14 清空方法体

### 2.14.1 注意点

+ `ElementMatchers.isDeclaredBy(Class<?> type))`：拦截仅由目标类声明的方法，通常用于排除超类方法
+ `StubMethod.INSTANCE`：Byte Buddy默认的拦截器方法实现之一，会根据被拦截的目标方法的返回值类型返回对应的默认值
  1. The value 0 for all numeric type.
  2. The null character for the char type.
  3. false for the boolean type.
  4. Nothing for void types.
  5. A null reference for any reference types. Note that this includes primitive wrapper types.
+ 当使用`ElementMatchers.any()`时，仅`subclass`包含构造方法，`rebase`和`redefine`不包含构造方法
+ 使用`ElementMatchers.any().and(ElementMatchers.isDeclaredBy(目标类.class))`时，仅`subclass`支持修改生成类名，`rebase`和`redefine`若修改类名则拦截后的修改/增强逻辑无效。

### 2.14.2 示例代码

1. 清空指定类所有方法的方法体（包含超类方法）

   ```java
   public class ByteBuddyCreateClassTest {
     /**
       * (25) 清空指定类的所有方法的方法体(包含超类方法)
       */
     @Test
     public void test25() throws IOException {
       DynamicType.Unloaded<SomethingClass> allMethodIncludeSuper = new ByteBuddy().redefine(SomethingClass.class)
         // 拦截所有方法(包括超类方法)
         .method(ElementMatchers.any())
         // 根据方法返回值类型, 返回对应类型的默认值
         .intercept(StubMethod.INSTANCE)
         .name("com.example.AshiamdTest25")
         .make();
       allMethodIncludeSuper.saveIn(DemoTools.currentClassPathFile());
     }
   }
   ```

   生成的类，反编译结果如下：

   <u>可以注意到构造方法的方法体没有被清空，当使用`rebase`或`redefine`就会如此，仅`subclass`会清空构造方法的方法体</u>

   ```java
   //
   // Source code recreated from a .class file by IntelliJ IDEA
   // (powered by FernFlower decompiler)
   //
   
   package com.example;
   
   public class AshiamdTest25 {
     public AshiamdTest25() {
       System.out.println("SomethingClass()");
     }
   
     public String selectUserName(Long var1) {
       return null;
     }
   
     public void print() {
     }
   
     public int getAge() {
       return 0;
     }
   
     public static void sayWhat(String var0) {
     }
   
     public boolean equals(Object var1) {
       return false;
     }
   
     public String toString() {
       return null;
     }
   
     public int hashCode() {
       return 0;
     }
   
     protected Object clone() throws CloneNotSupportedException {
       return null;
     }
   }
   
   ```

2. 清空指定类所有方法的方法体

   使用`subclass`时，生成类的构造方法体逻辑也被清空

   ```java
   public class ByteBuddyCreateClassTest {
     /**
       * (26) 清空指定类的当前类声明的所有方法的方法体(不包含超类方法)
       */
     @Test
     public void test26() throws IOException, ClassNotFoundException {
       DynamicType.Unloaded<SomethingClass> allMethod = new ByteBuddy().subclass(SomethingClass.class)
         // 拦截所有目标类声明的方法(不包括超类方法)
         .method(ElementMatchers.any().and(ElementMatchers.isDeclaredBy(SomethingClass.class)))
         // 根据方法返回值类型, 返回对应类型的默认值
         .intercept(StubMethod.INSTANCE)
         // 若这里使用rebase或redefine, 则需要去掉.name(“全限制类名”), 覆盖原类后才能使清空方法体的逻辑生效
         .name("com.example.AshiamdTest26")
         .make();
       allMethod.saveIn(DemoTools.currentClassPathFile());
     }
   }
   ```

   生成的类反编译结果如下：

   和上面不同的是，这里使用`subclass`，同时也清空了当前类构造方法的方法体

   ```java
   //
   // Source code recreated from a .class file by IntelliJ IDEA
   // (powered by FernFlower decompiler)
   //
   
   package com.example;
   
   import org.example.SomethingClass;
   
   public class AshiamdTest26 extends SomethingClass {
     public void print() {
     }
   
     public String selectUserName(Long var1) {
       return null;
     }
   
     public int getAge() {
       return 0;
     }
   
     public AshiamdTest26() {
     }
   }
   ```

# 三、java agent

## 3.1 原生jdk实现

### 3.1.1 注意点

+ `premain`方法在main之前执行
+ `Instrumentation#addTransformer(ClassFileTransformer transformer)`：注册字节码转换器，这里在premain方法内注册，保证在main方法执行前就完成字节码转换
+ 字节码中类名以`/`间隔，而不是`.`间隔

> 关于java agent，网上也有很多相关文章，这里不多做介绍，这里简单链接一些文章：
>
> [一文讲透Java Agent是什么玩意？能干啥？怎么用？ - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/636603910)
>
> [Java探针(javaagent) - 简书 (jianshu.com)](https://www.jianshu.com/p/cc88c8b0181b)
>
> [初探Java安全之JavaAgent - SecIN社区 - 博客园 (cnblogs.com)](https://www.cnblogs.com/SecIN/archive/2022/11/22/16915321.html)
>
> [java.lang.instrument (Java SE 21 & JDK 21) (oracle.com)](https://docs.oracle.com/en/java/javase/21/docs/api/java.instrument/java/lang/instrument/package-summary.html)

### 3.1.2 示例代码

新建一个module为`jdk-agent`，这里图方便，把后续java agent使用的premain类也放在同一个module内。需要注意maven的`pom.xml`配置，如下：

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>
  <parent>
    <groupId>org.example</groupId>
    <artifactId>ash_bytebuddy_study</artifactId>
    <version>1.0-SNAPSHOT</version>
  </parent>

  <artifactId>jdk-agent</artifactId>
  <packaging>jar</packaging>

  <name>jdk-agent</name>
  <url>http://maven.apache.org</url>

  <properties>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
  </properties>

  <dependencies>
    <!-- Byte Buddy -->
    <!-- https://mvnrepository.com/artifact/net.bytebuddy/byte-buddy -->
    <dependency>
      <groupId>net.bytebuddy</groupId>
      <artifactId>byte-buddy</artifactId>
    </dependency>
  </dependencies>

  <build>
    <plugins>
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-assembly-plugin</artifactId>
        <version>3.6.0</version>
        <configuration>
          <descriptorRefs>
            <descriptorRef>jar-with-dependencies</descriptorRef>
          </descriptorRefs>
          <archive>
            <manifest>
              <mainClass>org.example.MainClass</mainClass>
              <!-- 自动添加META-INF/MANIFEST.MF文件 -->
              <addClasspath>true</addClasspath>
              <!-- 将依赖的存放位置添加到 MANIFEST.MF 中-->
              <classpathPrefix>../lib/</classpathPrefix>
            </manifest>
            <manifestEntries>
              <!-- MANIFEST.MF 配置项 -->
              <Premain-Class>org.example.PreMainClass</Premain-Class>
              <Can-Redefine-Classes>true</Can-Redefine-Classes>
              <Can-Retransform-Classes>true</Can-Retransform-Classes>
              <Can-Set-Native-Method-Prefix>true</Can-Set-Native-Method-Prefix>
            </manifestEntries>
          </archive>
        </configuration>
        <executions>
          <execution>
            <id>make-assembly</id>
            <!-- 绑定到package生命周期 -->
            <phase>package</phase>
            <goals>
              <!-- 只运行一次 -->
              <goal>single</goal>
            </goals>
          </execution>
        </executions>
      </plugin>
    </plugins>
  </build>
</project>
```

先定义一个后续修改/增强的目标类`org.example.Something`

```java
package org.example;

/**
 * @author : Ashiamd email: ashiamd@foxmail.com
 * @date : 2023/12/10 12:04 PM
 */
public class Something {
  public static String returnHello() {
    return "Hello";
  }
}
```

其次，我们定义一个用于字节码转换的转换器类`org.example.ByteBuddyTransformer`，后续注册到`Instrumentation`实例，在premain方法执行时会执行我们的字节码修改/增强逻辑。

这里定义的修改/增强逻辑，即修改`org.example.Something#returnHello`的返回值，从"Hello"变成"Hi"

```java
package org.example;

import net.bytebuddy.ByteBuddy;
import net.bytebuddy.description.type.TypeDescription;
import net.bytebuddy.dynamic.ClassFileLocator;
import net.bytebuddy.implementation.FixedValue;
import net.bytebuddy.matcher.ElementMatchers;
import net.bytebuddy.pool.TypePool;

import java.lang.instrument.ClassFileTransformer;
import java.lang.instrument.IllegalClassFormatException;
import java.security.ProtectionDomain;

/**
 * @author : Ashiamd email: ashiamd@foxmail.com
 * @date : 2023/12/10 12:17 PM
 */
public class ByteBuddyTransformer implements ClassFileTransformer {

  /**
     * 在 某个类的字节码 被加载到JVM之前 都会先进入该方法. 如果对字节码进行修改则返回修改后的字节码, 否则直接返回null即可
     *
     * @param loader              the defining loader of the class to be transformed,
     *                            may be {@code null} if the bootstrap loader
     * @param className           the name of the class in the internal form of fully
     *                            qualified class and interface names as defined in
     *                            <i>The Java Virtual Machine Specification</i>.
     *                            For example, <code>"java/util/List"</code>.
     * @param classBeingRedefined if this is triggered by a redefine or retransform,
     *                            the class being redefined or retransformed;
     *                            if this is a class load, {@code null}
     * @param protectionDomain    the protection domain of the class being defined or redefined
     * @param classfileBuffer     the input byte buffer in class file format - must not be modified
     * @return a well-formed class file buffer (the result of the transform), or null if no transform is performed
     * @throws IllegalClassFormatException
     */
  @Override
  public byte[] transform(ClassLoader loader,
                          String className,
                          Class<?> classBeingRedefined,
                          ProtectionDomain protectionDomain,
                          byte[] classfileBuffer) throws IllegalClassFormatException {
    byte[] result = null;
    // 字节码中类名是使用/间隔, 而不是.
    if ("org/example/Something".equals(className)) {
      System.out.println("进行字节码修改");
      // 对Something.java进行字节码修改/增强 (这里修改字节码可以用任何字节码操作工具,asm, javassist, cglib, bytebuddy等)
      final String targetClassName = className.replace('/', '.');
      ClassFileLocator classLoaderLoader = ClassFileLocator.ForClassLoader.ofSystemLoader();
      ClassFileLocator.Compound loaderCompound = new ClassFileLocator.Compound(classLoaderLoader);
      TypePool typePool = TypePool.Default.of(loaderCompound);
      TypeDescription targetTypeDescription = typePool.describe(targetClassName).resolve();
      result = new ByteBuddy().redefine(targetTypeDescription, loaderCompound)
        .method(ElementMatchers.named("returnHello"))
        .intercept(FixedValue.value("Hi"))
        .make()
        .getBytes();
    }
    return result;
  }
}
```

定义premain入口对应的java类`org.example.PreMainClass`，premain方法会在main方法之前执行

```java
package org.example;

import java.lang.instrument.Instrumentation;

/**
 * 一般premain都是在另一个单独工程/module内编写然后打包, 这里图方便直接和需要插桩增强的{@link MainClass}放在同一个module内打包
 *
 * @author : Ashiamd email: ashiamd@foxmail.com
 * @date : 2023/12/10 12:02 PM
 */
public class PreMainClass {

  /**
     * java agent启动的方式之一(另一个是agentmain), premain方法在main方法之前先执行, 是插桩入口
     *
     * @param arg             javaagent指定的参数
     * @param instrumentation jdk自带的工具类
     */
  public static void premain(String arg, Instrumentation instrumentation) {
    System.out.println("premain, arg = " + arg);
    // 注册我们编写的 字节码转化器
    instrumentation.addTransformer(new ByteBuddyTransformer());
  }
}
```

最后写一个简易的main方法入口类`org.example.MainClass`，多线程输出`org.example.Something#returnHello`的返回值

```java
package org.example;

import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

/**
 * 简单的创建4个线程, 打印 {@link Something#returnHello()} 的返回值
 *
 * @author : Ashiamd email: ashiamd@foxmail.com
 * @date : 2023/12/10 12:02 PM
 */
public class MainClass {
  public static void main(String[] args) {
    ExecutorService executorService = Executors.newFixedThreadPool(4);
    for (int count = 0; count < 4; ++count) {
      executorService.submit(() -> {
        for (int i = 100; i > 0; --i) {
          System.out.println(Thread.currentThread().getName() + ", Something.returnHello():" + Something.returnHello());
          try {
            Thread.sleep(1000);
          } catch (InterruptedException e) {
            throw new RuntimeException(e);
          }
        }
      });
    }
  }
}
```

之后在项目所在路径执行`mvn clean package`完成module打包，得到`jdk-agent-1.0-SNAPSHOT-jar-with-dependencies.jar`。

执行指令运行java程序，`java -javaagent:jar包绝对路径=k1=v1,k2=v2 -jar jar包绝对路径`，标准输出如下：

```shell
premain, arg = k1=v1,k2=v2
进行字节码修改
pool-1-thread-2, Something.returnHello():Hi
pool-1-thread-1, Something.returnHello():Hi
pool-1-thread-4, Something.returnHello():Hi
pool-1-thread-3, Something.returnHello():Hi
pool-1-thread-3, Something.returnHello():Hi
pool-1-thread-4, Something.returnHello():Hi
pool-1-thread-1, Something.returnHello():Hi
pool-1-thread-2, Something.returnHello():Hi
// ...
```

## 3.2 byte buddy实现agent实战

byte buddy在jdk的java agent基础上进行了封装，更加简单易用。

### 3.2.1 拦截实例方法

#### 3.2.1.1 注意点

+ `AgentBuilder`：对java agent常见的类转换等逻辑进行包装的构造器类，通常在`premain`方法入口中使用
  + An agent builder provides a convenience API for defining a Java agent . By default, this transformation is applied by rebasing the type if not specified otherwise by setting a AgentBuilder.TypeStrategy.
+ `AgentBuilder.Transformer`：对被拦截的类进行修改/增强的转换器类，这里面主要指定拦截的方法和具体拦截后的增强逻辑
  + A transformer allows to apply modifications to a DynamicType. Such a modification is then applied to any instrumented type that was matched by the preceding matcher.
+ `AgentBuilder.Listener`：监听器类，在`instrumentation`过程中执行该类中的hook方法(里面所有类都是hook回调方法，在特定环节被调用，比如某个类被transform后，被ignored后，等等)
  + A listener that is informed about events that occur during an instrumentation process.

> 其他相关介绍，可见官方教程文档的"创建Java代理"章节，下面内容摘自官方教程文档

​	当一个应用增长得更大，且变得更模块化时，在指定的程序位置应用这样的转换当然是一个繁琐的强制约束。而且， 确实有一个更好的办法来*按需要*应用类的重定义。 用[Java 代理](https://docs.oracle.com/javase/8/docs/api/java/lang/instrument/package-summary.html)， 它可以拦截 Java 应用中进行的任何类加载活动。Java 代理被实现为一个简单的带有入口点的 jar 文件，其入口点在 jar 文件的 manifest(清单) 文件中指定， 像链接中描述的一样。在 Byte Buddy 中，通过使用`AgentBuilder`，代理的实现是相当直接的。 假定我们之前定义了一个名为`ToString`的注解，通过实现代理的`premain`方法， 对所有带该注解的类实现`toString`方法是很容易的。如下所示：

```java
class ToStringAgent {
  public static void premain(String arguments, Instrumentation instrumentation) {
    new AgentBuilder.Default()
      .type(isAnnotatedWith(ToString.class))
      .transform(new AgentBuilder.Transformer() {
        @Override
        public DynamicType.Builder transform(DynamicType.Builder builder,
                                             TypeDescription typeDescription,
                                             ClassLoader classloader) {
          return builder.method(named("toString"))
            .intercept(FixedValue.value("transformed"));
        }
      }).installOn(instrumentation);
  }
}
```

​	作为应用上述`AgentBuilder.Transformer`的结果，添加注解的类的所有`toString`方法将返回 `transformed`。 我们将在接下来的章节中学习`DynamicType.Builder`，所以现在不要担心这个类。上面的代码当然是一个微不足道且毫无意义的应用。 然而，正确使用这个概念，对于实现面向切面编程会提供一个非常强大的工具。

​	**请注意，在使用代理时也可以检测由启动类加载器加载的类。但是，这需要一些准备。首先，启动类加载器由`null`值表示， 这会导致无法通过反射在该类加载器加载类**。然而，有时需要加载辅助类到检测类的类加载器中以支持类的实现。<u>为了向启动类加载器中加载类， Byte Buddy 可以创建 jar 文件并且将这些文件添加到启动类的加载路径中。然而，为了使这个成为可能，需要将这些类保存到磁盘上。 使用`enableBootstrapInjection`命令可以指定存放这些类的文件夹，该命令也采用`Instrumentation`接口以附加这些类。 请注意，被检测类使用的所有用户类也需要放在启动类加载器类加载路径上，这样会使使用`Instrumentation`接口成为可能</u>。

#### 3.2.1.2 示例代码

新建`instance-method-agent`module，对应的`pom.xml`文件如下：

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>
  <parent>
    <groupId>org.example</groupId>
    <artifactId>ash_bytebuddy_study</artifactId>
    <version>1.0-SNAPSHOT</version>
  </parent>

  <artifactId>instance-method-agent</artifactId>
  <packaging>jar</packaging>

  <name>instance-method-agent</name>
  <url>http://maven.apache.org</url>

  <properties>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
  </properties>

  <dependencies>
    <!-- Byte Buddy -->
    <!-- https://mvnrepository.com/artifact/net.bytebuddy/byte-buddy -->
    <dependency>
      <groupId>net.bytebuddy</groupId>
      <artifactId>byte-buddy</artifactId>
    </dependency>
  </dependencies>

  <build>
    <plugins>
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-assembly-plugin</artifactId>
        <version>3.6.0</version>
        <configuration>
          <descriptorRefs>
            <descriptorRef>jar-with-dependencies</descriptorRef>
          </descriptorRefs>
          <archive>
            <manifest>
              <mainClass>org.example.InstanceMainClass</mainClass>
              <!-- 自动添加META-INF/MANIFEST.MF文件 -->
              <addClasspath>true</addClasspath>
              <!-- 将依赖的存放位置添加到 MANIFEST.MF 中-->
              <classpathPrefix>../lib/</classpathPrefix>
            </manifest>
            <manifestEntries>
              <!-- MANIFEST.MF 配置项 -->
              <Premain-Class>org.example.InstancePreMainClass</Premain-Class>
              <Can-Redefine-Classes>true</Can-Redefine-Classes>
              <Can-Retransform-Classes>true</Can-Retransform-Classes>
              <Can-Set-Native-Method-Prefix>true</Can-Set-Native-Method-Prefix>
            </manifestEntries>
          </archive>
        </configuration>
        <executions>
          <execution>
            <id>make-assembly</id>
            <!-- 绑定到package生命周期 -->
            <phase>package</phase>
            <goals>
              <!-- 只运行一次 -->
              <goal>single</goal>
            </goals>
          </execution>
        </executions>
      </plugin>
    </plugins>
  </build>
</project>
```

这里写的类相对较多，先总体讲一下每个类的作用，下面直接按顺序把每个类的代码贴上，最后贴上运行结果

```shell
src
└── main
    └── java
        └── org
            └── example
                ├── AshCallLog.java
                ├── AshOtherAnno.java
                ├── InstanceInterceptor.java
                ├── InstanceListener.java
                ├── InstanceMainClass.java
                ├── InstancePreMainClass.java
                ├── InstanceTransformer.java
                ├── Something01.java
                └── Something02.java
```

+ `AshCallLog.java`：注解类，用于标识某个类中的具体某个方法后续需要通过字节码增强，打印额外方法执行信息。
+ `AshOtherAnno.java`：注解类，无实际用处，只是用来和`AshCallLog.java`形成对比。即使用当前注解的方法，并不会因此被我们后续拦截增强
+ `InstanceInterceptor.java`：该示例代码中，拦截实例方法并增强逻辑的拦截器类，内部逻辑只是单纯的输出被调用的方法名，方法参数，返回值，以及方法执行耗时
+ `InstanceListener.java`：监听器类，当`Instrumentation`实例执行Byte Buddy封装的agent逻辑时，会执行其内部的多个hook回调方法
+ `InstanceMainClass.java`：main方法入口类，简单的调用`Something01`和`Something02`的方法
+ `InstancePreMainClass.java`：premain方法入口类，使用Byte Buddy封装的agent逻辑进行transform
+ `InstanceTransformer.java`：类转化器，用于注册需要拦截的方法，以及指定拦截后的增强逻辑
+ `Something01.java`：被增强的目标类1，实际增强其内部使用了`@AshCallLog`注解的方法
+ `Something02.java`：被增强的目标类2，实际增强其内部使用了`@AshCallLog`注解的方法

---

+ ``AshCallLog.java`：注解类，用于标识某个类中的具体某个方法后续需要通过字节码增强，打印额外方法执行信息。`

```java
package org.example;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

/**
 * 标识 需要记录方法调用信息 的方法, 后续 插桩逻辑会log输出 被调用的方法名, 方法参数, 方法执行耗时等信息
 *
 * @author : Ashiamd email: ashiamd@foxmail.com
 * @date : 2023/12/10 5:05 PM
 */
@Retention(RetentionPolicy.RUNTIME)
@Target(value = {ElementType.TYPE, ElementType.METHOD})
public @interface AshCallLog {
}
```

+ `AshOtherAnno.java`：注解类，无实际用处，只是用来和`AshCallLog.java`形成对比。即使用当前注解的方法，并不会因此被我们后续拦截增强

```java
package org.example;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

/**
 * 模拟其他 注解, 无实际作用
 *
 * @author : Ashiamd email: ashiamd@foxmail.com
 * @date : 2023/12/10 5:06 PM
 */
@Retention(RetentionPolicy.RUNTIME)
@Target(value = {ElementType.TYPE, ElementType.METHOD})
public @interface AshOtherAnno {
}
```

+ `InstanceInterceptor.java`：该示例代码中，拦截实例方法并增强逻辑的拦截器类，内部逻辑只是单纯的输出被调用的方法名，方法参数，返回值，以及方法执行耗时

```java
package org.example;

import net.bytebuddy.implementation.bind.annotation.AllArguments;
import net.bytebuddy.implementation.bind.annotation.Origin;
import net.bytebuddy.implementation.bind.annotation.RuntimeType;
import net.bytebuddy.implementation.bind.annotation.SuperCall;

import java.lang.reflect.Method;
import java.util.Arrays;
import java.util.concurrent.Callable;

/**
 * 实例方法 拦截器, 这里配合 {@link AshCallLog} 注解使用,
 * 输出被调用的方法的方法名, 参数, 返回值, 执行时间
 *
 * @author : Ashiamd email: ashiamd@foxmail.com
 * @date : 2023/12/10 8:30 PM
 */
public class InstanceInterceptor {

  @RuntimeType
  public Object instanceMethodIntercept(
    @Origin Method targetMethod,
    @AllArguments Object[] targetMethodArgs,
    @SuperCall Callable<?> zuper) {
    System.out.println("「增强逻辑」targetMethod.getName() = " + targetMethod.getName());
    System.out.println("「增强逻辑」Arrays.toString(targetMethodArgs) = " + Arrays.toString(targetMethodArgs));
    long callStartTime = System.currentTimeMillis();
    Object returnValue = null;
    try {
      returnValue = zuper.call();
      System.out.println("「增强逻辑」returnValue = " + returnValue);
    } catch (Exception e) {
      throw new RuntimeException(e);
    } finally {
      System.out.println("「增强逻辑」callTime: " + (System.currentTimeMillis() - callStartTime));
    }
    return returnValue;
  }
}
```

+ `InstanceListener.java`：监听器类，当`Instrumentation`实例执行Byte Buddy封装的agent逻辑时，会执行其内部的多个hook回调方法

```java
package org.example;

import net.bytebuddy.agent.builder.AgentBuilder;
import net.bytebuddy.description.type.TypeDescription;
import net.bytebuddy.dynamic.DynamicType;
import net.bytebuddy.utility.JavaModule;

/**
 * 注册监听器, 在 类进行transform过程中, 会回调下面这些hook方法<br/>
 * A listener that is informed about events that occur during an instrumentation process.
 *
 * @author : Ashiamd email: ashiamd@foxmail.com
 * @date : 2023/12/10 8:42 PM
 */
public class InstanceListener implements AgentBuilder.Listener {
  /**
     * 当某个类将要被加载时, 就会回调该方法
     *
     * @param typeName    The binary name of the instrumented type.
     * @param classLoader The class loader which is loading this type or {@code null} if loaded by the boots loader.
     * @param module      The instrumented type's module or {@code null} if the current VM does not support modules.
     * @param loaded      {@code true} if the type is already loaded.
     */
  @Override
  public void onDiscovery(String typeName, ClassLoader classLoader, JavaModule module, boolean loaded) {
    System.out.println("【onDiscovery】typeName: " + typeName);
  }

  /**
     * 对某个类完成了transform之后会回调
     *
     * @param typeDescription The type that is being transformed.
     * @param classLoader     The class loader which is loading this type or {@code null} if loaded by the boots loader.
     * @param module          The transformed type's module or {@code null} if the current VM does not support modules.
     * @param loaded          {@code true} if the type is already loaded.
     * @param dynamicType     The dynamic type that was created.
     */
  @Override
  public void onTransformation(TypeDescription typeDescription, ClassLoader classLoader, JavaModule module, boolean loaded, DynamicType dynamicType) {
    System.out.println("【onTransformation】typeDescription: " + typeDescription);
  }

  /**
     * 当某个类将要被加载, 但配置了被byte buddy忽略(或本身没有配置被拦截), 则执行该方法
     *
     * @param typeDescription The type being ignored for transformation.
     * @param classLoader     The class loader which is loading this type or {@code null} if loaded by the boots loader.
     * @param module          The ignored type's module or {@code null} if the current VM does not support modules.
     * @param loaded          {@code true} if the type is already loaded.
     */
  @Override
  public void onIgnored(TypeDescription typeDescription, ClassLoader classLoader, JavaModule module, boolean loaded) {
    System.out.println("【onIgnored】 typeDescription: " + typeDescription);
  }

  /**
     * 当 Byte Buddy 在 transform 过程中 发生异常, 则执行该方法
     *
     * @param typeName    The binary name of the instrumented type.
     * @param classLoader The class loader which is loading this type or {@code null} if loaded by the boots loader.
     * @param module      The instrumented type's module or {@code null} if the current VM does not support modules.
     * @param loaded      {@code true} if the type is already loaded.
     * @param throwable   The occurred error.
     */
  @Override
  public void onError(String typeName, ClassLoader classLoader, JavaModule module, boolean loaded, Throwable throwable) {
    System.out.println("【onError】 typeName: " + typeName);
  }

  /**
     * 某个类处理结束后(transform/ignore/error都算), 回调该方法
     *
     * @param typeName    The binary name of the instrumented type.
     * @param classLoader The class loader which is loading this type or {@code null} if loaded by the boots loader.
     * @param module      The instrumented type's module or {@code null} if the current VM does not support modules.
     * @param loaded      {@code true} if the type is already loaded.
     */
  @Override
  public void onComplete(String typeName, ClassLoader classLoader, JavaModule module, boolean loaded) {
    System.out.println("【onComplet】 typeName: " + typeName);
  }
}
```

+ `InstanceMainClass.java`：main方法入口类，简单的调用`Something01`和`Something02`的方法

```java
package org.example;

/**
 * 入口类, 观察 java agent修改字节码的效果
 *
 * @author : Ashiamd email: ashiamd@foxmail.com
 * @date : 2023/12/10 4:57 PM
 */
public class InstanceMainClass {
  public static void main(String[] args) {
    System.out.println("执行 main");
    Something01 something01 = new Something01();
    System.out.println("main, something01.returnSeven(1, 2) = " + something01.returnSeven(1, 2));
    System.out.println("main, something01.returnZero(1, 2) = " + something01.returnZero(1, 2));

    Something02 something02 = new Something02();
    System.out.println("main, something02.returnHello(\"hi\", \"hello\") = " + something02.returnHello("hi", "hello"));
    System.out.println("main, something02.returnHi(\"hahaha\") = " + something02.returnHi("hahaha"));
    System.out.println("main, something02.returnArgs(\"arg1\", \"arg2\") = " + something02.returnArgs("arg1", "arg2"));
  }
}
```

+ `InstancePreMainClass.java`：premain方法入口类，使用Byte Buddy封装的agent逻辑进行transform

```java
package org.example;

import net.bytebuddy.agent.builder.AgentBuilder;
import net.bytebuddy.matcher.ElementMatchers;

import java.lang.instrument.Instrumentation;

import static net.bytebuddy.matcher.ElementMatchers.*;

/**
 * premain 入口类, 用于java agent进行插桩
 *
 * @author : Ashiamd email: ashiamd@foxmail.com
 * @date : 2023/12/10 5:21 PM
 */
public class InstancePreMainClass {
  /**
     * java agent 入口, premain在main方法之前执行
     */
  public static void premain(String arg, Instrumentation instrumentation) {
    System.out.println("执行 premain");
    // 使用 Byte Buddy 包装的 agent常见处理逻辑(指定要拦截的对象, 以及拦截后的处理逻辑, 任何字节码操作工具都基本这个流程)
    AgentBuilder agentBuilder = new AgentBuilder.Default()
      // 忽略(不拦截)的类, 这里忽略 java自带类和byte buddy的类
      .ignore(ElementMatchers.nameStartsWith("java.")
              .or(ElementMatchers.nameStartsWith("javax."))
              .or(ElementMatchers.nameStartsWith("jdk."))
              .or(ElementMatchers.nameStartsWith("sun."))
              // 忽略byte buddy的类
              .or(ElementMatchers.nameStartsWith("net.bytebuddy.")))
      // 拦截的类
      .type(isAnnotatedWith(nameStartsWith("org.example.Ash").and(nameEndsWith("Log"))))
      // 拦截的方法, 以及指定修改/增强的逻辑
      .transform(new InstanceTransformer())
      // 注册 回调方法监听器
      .with(new InstanceListener());
    // 注册到 Instrumentation
    agentBuilder.installOn(instrumentation);
  }
}
```

+ `InstanceTransformer.java`：类转化器，用于注册需要拦截的方法，以及指定拦截后的增强逻辑

```java
package org.example;

import net.bytebuddy.ByteBuddy;
import net.bytebuddy.agent.builder.AgentBuilder;
import net.bytebuddy.description.type.TypeDescription;
import net.bytebuddy.dynamic.DynamicType;
import net.bytebuddy.implementation.MethodDelegation;
import net.bytebuddy.utility.JavaModule;

import java.security.ProtectionDomain;

import static net.bytebuddy.matcher.ElementMatchers.*;

/**
 * 针对 实例 方法进行 修改/增强的 类转换器 <br/>
 * 某个类被 {@link AgentBuilder#type(ElementMatcher)}匹配后, 将要被类加载时, 进入transform方法
 *
 * @author : Ashiamd email: ashiamd@foxmail.com
 * @date : 2023/12/10 6:06 PM
 */
public class InstanceTransformer implements AgentBuilder.Transformer {
  /**
     * transform 转化逻辑
     *
     * @param builder          之前Byte Buddy 增强修改类时的中间产物Builder(比如 {@link ByteBuddy#subclass(Class)}的返回值就是) The dynamic builder to transform.
     * @param typeDescription  将被加载的类对应的类信息 The description of the type currently being instrumented.
     * @param classLoader      The class loader of the instrumented class. Might be {@code null} to represent the bootstrap class loader.
     * @param module           The class's module or {@code null} if the current VM does not support modules.
     * @param protectionDomain The protection domain of the transformed type.
     * @return : A transformed version of the supplied builder.
     */
  @Override
  public DynamicType.Builder<?> transform(DynamicType.Builder<?> builder,
                                          TypeDescription typeDescription,
                                          ClassLoader classLoader,
                                          JavaModule module,
                                          ProtectionDomain protectionDomain) {
    return builder.method(not(isStatic()).and(isAnnotatedWith(nameStartsWith("org.example.Ash").and(nameEndsWith("Log")))))
      .intercept(MethodDelegation.to(new InstanceInterceptor()));
  }
}

```

+ `Something01.java`：被增强的目标类1，实际增强其内部使用了`@AshCallLog`注解的方法

```java
package org.example;

/**
 * 后续 进行 插桩增强的目标类 01
 *
 * @author : Ashiamd email: ashiamd@foxmail.com
 * @date : 2023/12/10 5:01 PM
 */
@AshCallLog
@AshOtherAnno
public class Something01 {
  public int returnZero(int number01, int number02) {
    return 0;
  }

  @AshCallLog
  public int returnSeven(int... numbers) {
    return 7;
  }
}

```

+ `Something02.java`：被增强的目标类2，实际增强其内部使用了`@AshCallLog`注解的方法

```java
package org.example;

/**
 * 后续 进行 插桩增强的目标类 01
 *
 * @author : Ashiamd email: ashiamd@foxmail.com
 * @date : 2023/12/10 5:01 PM
 */
@AshCallLog
@AshOtherAnno
public class Something01 {
  public int returnZero(int number01, int number02) {
    return 0;
  }

  @AshCallLog
  public int returnSeven(int... numbers) {
    return 7;
  }
}
```

之后在项目所在路径执行`mvn clean package`完成module打包，得到`instance-method-agent-1.0-SNAPSHOT-jar-with-dependencies.jar`。

执行指令运行java程序，`java -javaagent:jar包绝对路径=k1=v1,k2=v2 -jar jar包绝对路径`，标准输出如下：

```shell
执行 premain
【onDiscovery】typeName: jdk.internal.vm.PostVMInitHook
【onIgnored】 typeDescription: class jdk.internal.vm.PostVMInitHook
【onComplet】 typeName: jdk.internal.vm.PostVMInitHook
【onDiscovery】typeName: jdk.internal.vm.PostVMInitHook$1
【onIgnored】 typeDescription: class jdk.internal.vm.PostVMInitHook$1
【onComplet】 typeName: jdk.internal.vm.PostVMInitHook$1
【onDiscovery】typeName: sun.launcher.LauncherHelper
【onIgnored】 typeDescription: class sun.launcher.LauncherHelper
【onComplet】 typeName: sun.launcher.LauncherHelper
【onDiscovery】typeName: org.example.InstanceMainClass
【onIgnored】 typeDescription: class org.example.InstanceMainClass
【onComplet】 typeName: org.example.InstanceMainClass
【onDiscovery】typeName: jdk.internal.misc.MainMethodFinder
【onIgnored】 typeDescription: class jdk.internal.misc.MainMethodFinder
【onComplet】 typeName: jdk.internal.misc.MainMethodFinder
执行 main
【onDiscovery】typeName: org.example.Something01
【onTransformation】typeDescription: class org.example.Something01
【onComplet】 typeName: org.example.Something01
【onDiscovery】typeName: org.example.Something01$auxiliary$9zuoZ9to
【onIgnored】 typeDescription: class org.example.Something01$auxiliary$9zuoZ9to
【onComplet】 typeName: org.example.Something01$auxiliary$9zuoZ9to
「增强逻辑」targetMethod.getName() = returnSeven
「增强逻辑」Arrays.toString(targetMethodArgs) = [[I@563e4951]
「增强逻辑」returnValue = 7
【onDiscovery】typeName: java.lang.invoke.BoundMethodHandle$Species_LJ
【onIgnored】 typeDescription: class java.lang.invoke.BoundMethodHandle$Species_LJ
【onComplet】 typeName: java.lang.invoke.BoundMethodHandle$Species_LJ
「增强逻辑」callTime: 0
main, something01.returnSeven(1, 2) = 7
main, something01.returnZero(1, 2) = 0
【onDiscovery】typeName: org.example.Something02
【onTransformation】typeDescription: class org.example.Something02
【onComplet】 typeName: org.example.Something02
【onDiscovery】typeName: org.example.Something02$auxiliary$OqhCeZyv
【onIgnored】 typeDescription: class org.example.Something02$auxiliary$OqhCeZyv
【onComplet】 typeName: org.example.Something02$auxiliary$OqhCeZyv
main, something02.returnHello("hi", "hello") = Hello
main, something02.returnHi("hahaha") = Hi
「增强逻辑」targetMethod.getName() = returnArgs
「增强逻辑」Arrays.toString(targetMethodArgs) = [[Ljava.lang.String;@3e6f3f28]
「增强逻辑」returnValue = [arg1, arg2]
「增强逻辑」callTime: 0
main, something02.returnArgs("arg1", "arg2") = [arg1, arg2]
【onDiscovery】typeName: java.util.IdentityHashMap$IdentityHashMapIterator
【onIgnored】 typeDescription: class java.util.IdentityHashMap$IdentityHashMapIterator
【onComplet】 typeName: java.util.IdentityHashMap$IdentityHashMapIterator
【onDiscovery】typeName: java.util.IdentityHashMap$KeyIterator
【onIgnored】 typeDescription: class java.util.IdentityHashMap$KeyIterator
【onComplet】 typeName: java.util.IdentityHashMap$KeyIterator
【onDiscovery】typeName: java.lang.Shutdown
【onIgnored】 typeDescription: class java.lang.Shutdown
【onComplet】 typeName: java.lang.Shutdown
【onDiscovery】typeName: java.lang.Shutdown$Lock
【onIgnored】 typeDescription: class java.lang.Shutdown$Lock
【onComplet】 typeName: java.lang.Shutdown$Lock
```

### 3.2.2 拦截静态方法

#### 3.2.2.1 注意点

+ `ElementMatcher`：元素匹配器，本次示例代码实现了自定义类元素匹配器

```java
public static ElementMatcher<? super TypeDescription> getMatcher() {
  // 相同效果, 类名匹配 return ElementMatchers.named("org.example.StaticUtils");
  return new ElementMatcher.Junction.AbstractBase<NamedElement>() {
    @Override
    public boolean matches(NamedElement target) {
      // 当类名匹配时, 拦截
      return "org.example.StaticUtils".equals(target.getActualName());
    }
  };
}
```

#### 3.2.2.2 示例代码

新建`static-method-agent`模块，对应的`pom.xml`文件如下：

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>
  <parent>
    <groupId>org.example</groupId>
    <artifactId>ash_bytebuddy_study</artifactId>
    <version>1.0-SNAPSHOT</version>
  </parent>

  <artifactId>static-method-agent</artifactId>
  <packaging>jar</packaging>

  <name>static-method-agent</name>
  <url>http://maven.apache.org</url>

  <properties>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
  </properties>

  <dependencies>
    <!-- Byte Buddy -->
    <!-- https://mvnrepository.com/artifact/net.bytebuddy/byte-buddy -->
    <dependency>
      <groupId>net.bytebuddy</groupId>
      <artifactId>byte-buddy</artifactId>
    </dependency>
  </dependencies>

  <build>
    <plugins>
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-assembly-plugin</artifactId>
        <version>3.6.0</version>
        <configuration>
          <descriptorRefs>
            <descriptorRef>jar-with-dependencies</descriptorRef>
          </descriptorRefs>
          <archive>
            <manifest>
              <mainClass>org.example.StaticMain</mainClass>
              <!-- 自动添加META-INF/MANIFEST.MF文件 -->
              <addClasspath>true</addClasspath>
              <!-- 将依赖的存放位置添加到 MANIFEST.MF 中-->
              <classpathPrefix>../lib/</classpathPrefix>
            </manifest>
            <manifestEntries>
              <!-- MANIFEST.MF 配置项 -->
              <Premain-Class>org.example.StaticPremain</Premain-Class>
              <Can-Redefine-Classes>true</Can-Redefine-Classes>
              <Can-Retransform-Classes>true</Can-Retransform-Classes>
              <Can-Set-Native-Method-Prefix>true</Can-Set-Native-Method-Prefix>
            </manifestEntries>
          </archive>
        </configuration>
        <executions>
          <execution>
            <id>make-assembly</id>
            <!-- 绑定到package生命周期 -->
            <phase>package</phase>
            <goals>
              <!-- 只运行一次 -->
              <goal>single</goal>
            </goals>
          </execution>
        </executions>
      </plugin>
    </plugins>
  </build>
</project>

```

src目录结构如下：

```shell
src
└── main
    └── java
        └── org
            └── example
                ├── StaticInterceptor.java
                ├── StaticMain.java
                ├── StaticPremain.java
                ├── StaticTransformer.java
                └── StaticUtils.java
```

+ `StaticInterceptor.java`

  ```java
  package org.example;
  
  import net.bytebuddy.implementation.bind.annotation.AllArguments;
  import net.bytebuddy.implementation.bind.annotation.Origin;
  import net.bytebuddy.implementation.bind.annotation.RuntimeType;
  import net.bytebuddy.implementation.bind.annotation.SuperCall;
  
  import java.lang.reflect.Method;
  import java.util.concurrent.Callable;
  
  /**
   * 静态方法拦截
   *
   * @author : Ashiamd email: ashiamd@foxmail.com
   * @date : 2023/12/15 10:03 PM
   */
  public class StaticInterceptor {
  
    @RuntimeType
    public Object staticMethodIntercept(
      @Origin Class<?> clazz,
      @Origin Method targetMethod,
      @AllArguments Object[] targetMethodArgs,
      @SuperCall Callable<?> zuper) {
      System.out.println("「增强逻辑」targetMethod.getName() = " + targetMethod.getName());
      long callStartTime = System.currentTimeMillis();
      Object returnValue = null;
      try {
        returnValue = zuper.call();
      } catch (Exception e) {
        throw new RuntimeException(e);
      } finally {
        System.out.println("「增强逻辑」callTime: " + (System.currentTimeMillis() - callStartTime));
      }
      return returnValue;
    }
  }
  ```

+ `StaticMain.java`

  ```java
  package org.example;
  
  /**
   * main 入口, 观察 java agent修改字节码的效果
   *
   * @author : Ashiamd email: ashiamd@foxmail.com
   * @date : 2023/12/15 9:56 PM
   */
  public class StaticMain {
    public static void main(String[] args) {
      System.out.println("执行 main");
      System.out.println(StaticUtils.hi("Ashiamd"));
      System.out.println(StaticUtils.hi("IABTD"));
    }
  }
  ```

+ `StaticPremain.java`

  ```java
  package org.example;
  
  import net.bytebuddy.agent.builder.AgentBuilder;
  import net.bytebuddy.description.NamedElement;
  import net.bytebuddy.description.type.TypeDescription;
  import net.bytebuddy.matcher.ElementMatcher;
  
  import java.lang.instrument.Instrumentation;
  
  /**
   * premain 入口, 进行java agent 插桩
   *
   * @author : Ashiamd email: ashiamd@foxmail.com
   * @date : 2023/12/15 9:56 PM
   */
  public class StaticPremain {
  
    /**
       * java agent 入口, premain 在main方法之前执行
       */
    public static void premain(String arg, Instrumentation instrumentation) {
      System.out.println("执行 premain");
      new AgentBuilder.Default()
        // 使用我们自定义的匹配器指定拦截的类
        .type(getMatcher())
        .transform(new StaticTransformer())
        .installOn(instrumentation);
    }
  
    public static ElementMatcher<? super TypeDescription> getMatcher() {
      // 相同效果, 类名匹配 return ElementMatchers.named("org.example.StaticUtils");
      return new ElementMatcher.Junction.AbstractBase<NamedElement>() {
        @Override
        public boolean matches(NamedElement target) {
          // 当类名匹配时, 拦截
          return "org.example.StaticUtils".equals(target.getActualName());
        }
      };
    }
  }
  ```

+ `StaticTransformer.java`

  ```java
  package org.example;
  
  import net.bytebuddy.agent.builder.AgentBuilder;
  import net.bytebuddy.description.type.TypeDescription;
  import net.bytebuddy.dynamic.DynamicType;
  import net.bytebuddy.implementation.MethodDelegation;
  import net.bytebuddy.matcher.ElementMatcher;
  import net.bytebuddy.matcher.ElementMatchers;
  import net.bytebuddy.utility.JavaModule;
  
  import java.security.ProtectionDomain;
  
  /**
   * 针对静态方法进行增强<br/>
   * 某个类被 {@link AgentBuilder#type(ElementMatcher)}匹配, 将要被类加载时, 进入transform方法
   *
   * @author : Ashiamd email: ashiamd@foxmail.com
   * @date : 2023/12/15 10:04 PM
   */
  public class StaticTransformer implements AgentBuilder.Transformer {
    @Override
    public DynamicType.Builder<?> transform(DynamicType.Builder<?> builder, TypeDescription typeDescription, ClassLoader classLoader, JavaModule module, ProtectionDomain protectionDomain) {
      return builder.method(ElementMatchers.isStatic())
        .intercept(MethodDelegation.to(new StaticInterceptor()));
    }
  }
  
  ```

+ `StaticUtils.java`

  ```java
  package org.example;
  
  /**
   * 本次Demo准备修改/增强的目标类
   *
   * @author : Ashiamd email: ashiamd@foxmail.com
   * @date : 2023/12/15 10:03 PM
   */
  public class StaticUtils {
    public static String hi(String name) {
      return "Hi, " + name;
    }
  }
  ```

之后在项目所在路径执行`mvn clean package`完成module打包，得到`static-method-agent-1.0-SNAPSHOT-jar-with-dependencies.jar`。

执行指令运行java程序，`java -javaagent:jar包绝对路径 -jar jar包绝对路径`，标准输出如下：

```shell
执行 premain
执行 main
「增强逻辑」targetMethod.getName() = hi
「增强逻辑」callTime: 0
Hi, Ashiamd
「增强逻辑」targetMethod.getName() = hi
「增强逻辑」callTime: 0
Hi, IABTD
```

### 3.2.3 拦截构造方法

#### 3.2.3.1 注意点

#### 3.2.3.2 示例代码



## 3.3 总结

